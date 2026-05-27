# Umami 数据保留策略分析

## 1. 策略表达：明细与聚合的分层架构

### 1.1 双轨数据存储设计

Umami 在 ClickHouse 存储层采用了**明细数据 + 聚合数据**的双轨设计，这是实现"保留窗口取舍"的基础架构：

**明细数据表** (`website_event`)
- 存储每条原始事件的完整信息（URL、浏览器、设备、地域等）
- 引擎：`MergeTree`，按 `toYYYYMM(created_at)` 按月分区
- 特点：数据量大、字段完整、查询灵活但速度较慢

**聚合表** (`website_event_stats_hourly`)
- 按小时粒度预聚合的统计数据
- 引擎：`AggregatingMergeTree`，支持增量聚合
- 特点：数据量小、查询速度快、但维度受限

### 1.2 物化视图的自动聚合机制

通过 Materialized View 实现从明细到聚合的实时同步：

```sql
-- db/clickhouse/schema.sql:145-239
CREATE MATERIALIZED VIEW umami.website_event_stats_hourly_mv
TO umami.website_event_stats_hourly
AS
SELECT ... FROM umami.website_event
GROUP BY website_id, session_id, visit_id, ..., toStartOfHour(created_at)
```

**聚合维度**：website_id + session_id + visit_id + 小时 + 事件类型

**聚合指标**：
- `views`: 页面浏览量求和（`sumIf(1, event_type NOT IN (2, 5))`）
- `min_time`/`max_time`: 访问起止时间
- `entry_url`/`exit_url`: 入口/出口页面（argMin/argMax）
- 各维度字段的数组去重存储（`SimpleAggregateFunction(groupArrayArray, ...)`）

**关键设计**：
- 聚合表使用 `AggregatingMergeTree` 引擎，支持后台自动合并
- 物化视图是**只追加（INSERT-ONLY）** 模式：只在新数据写入时触发
- 删除明细数据时，物化视图**不会反向删除**已聚合的数据
- 这确保了"删明细、保聚合"的取舍策略天然成立

## 2. 触发清理的身份与执行机制

### 2.1 清理动作的触发方

**代码中不存在内置的定时清理任务**。清理动作由以下方触发：

| 触发方 | 机制 | 代码位置 |
|--------|------|---------|
| **运维管理员** | 手动执行 `ALTER TABLE ... DROP PARTITION` | 无代码，运维操作 |
| **ClickHouse TTL** | 表级 TTL 表达式自动过期 | 需运维配置（代码中未设置） |
| **用户重置** | `resetWebsite` API 触发 Prisma 层删除 | `src/queries/prisma/website.ts:133-186` |

**重要发现**：Umami 代码库中**没有定义 TTL 或定时清理逻辑**。这是一个设计取舍：
- 应用层专注于数据写入和查询
- 数据生命周期管理交由 DBA/运维层面
- 通过 ClickHouse 的 `MODIFY TTL` 语句在部署时配置

### 2.2 批次执行顺序：如何确保只删明细不影响聚合

**ClickHouse 分区删除的独立性**：

```
数据写入流程：
  website_event (明细)  →  MV自动触发  →  website_event_stats_hourly (聚合)
         ↓                                        ↓
    按月分区存储                              按月分区存储
```

**关键机制**：

1. **物化视图的单向传播**
   - 新数据写入 `website_event` 时，自动插入 `website_event_stats_hourly`
   - 删除 `website_event` 分区时，**不会触发**物化视图的反向操作
   - 聚合表数据保持不变

2. **分区独立管理**
   ```sql
   -- 删除明细分区（不影响聚合表）
   ALTER TABLE umami.website_event DROP PARTITION '2024-01';
   
   -- 聚合表分区独立存在
   -- umami.website_event_stats_hourly 的 2024-01 分区仍然保留
   ```

3. **数据完整性保护**
   - 聚合表一旦生成，即成为独立数据源
   - 常规统计查询直接从聚合表读取，不依赖明细
   - 明细删除后，聚合数据的历史统计依然完整

### 2.3 Prisma 层的删除顺序（PostgreSQL）

对于 PostgreSQL 后端的删除，`resetWebsite` 函数遵循**从子表到主表**的顺序：

```typescript
// src/queries/prisma/website.ts:133-186
await tx.sessionReplaySaved.deleteMany({ where: { websiteId } });  // 1. 保存的回放
await tx.sessionReplay.deleteMany({ where: { websiteId } });       // 2. 会话回放
await tx.revenue.deleteMany({ where: { websiteId } });             // 3. 收入数据
await tx.eventData.deleteMany({ where: { websiteId } });           // 4. 事件数据
await tx.sessionData.deleteMany({ where: { websiteId } });         // 5. 会话数据
await tx.websiteEvent.deleteMany({ where: { websiteId } });        // 6. 网站事件
await tx.session.deleteMany({ where: { websiteId } });             // 7. 会话
```

**注意**：此函数只删除 PostgreSQL 中的数据。如果启用了 ClickHouse，ClickHouse 中的数据需要通过 `DROP PARTITION` 单独清理。

## 3. 数据边界：保留窗口与有效范围

### 3.1 双轨数据的保留窗口定义

设：
- `T_agg_start`：聚合表最早数据日期
- `T_agg_end`：聚合表最新数据日期
- `T_raw_start`：明细表最早数据日期
- `T_raw_end`：明细表最新数据日期
- `T_eventdata_start`：事件参数表最早数据日期
- `T_sessiondata_start`：会话参数表最早数据日期
- `T_replay_start`：会话回放表最早数据日期
- `T_query_start`：查询起始日期
- `T_query_end`：查询结束日期

**典型分层保留配置示例**：
```
聚合表保留 3 年    (T_agg_start = 3年前)
明细表保留 90 天   (T_raw_start = 90天前)
事件参数保留 90 天 (T_eventdata_start = 90天前)
会话参数保留 90 天 (T_sessiondata_start = 90天前)
回放数据保留 14 天 (T_replay_start = 14天前)

时间轴：
  ──┼──────────────────────────────────────────┼──────────────────┼──────┼──
    T_agg_start (3年前)                        T_raw_start (90天前)       T_replay_start  Today
    │                                            │                        │              │
    └──────────── 聚合数据完整范围 ──────────────┘                        │              │
                                                 └── 明细/事件/会话数据范围 ─┘              │
                                                                          └── 回放数据范围 ──┘
```

### 3.2 查询结果的有效性分段

根据查询日期范围与保留窗口的相对位置，查询结果分为三种情况：

| 查询区间 | 数据来源 | 结果状态 |
|---------|---------|---------|
| `[T_query_start, T_query_end]` **完全在** `[T_raw_start, T_raw_end]` 内 | 明细表 | ✅ 数据完整 |
| `[T_query_start, T_query_end]` **完全在** `[T_agg_start, T_raw_start)` 内 | 聚合表（当查询支持聚合时） | ⚠️ 维度受限（部分过滤不可用） |
| `[T_query_start, T_raw_start)` + `[T_raw_start, T_query_end]` | 分段：聚合 + 明细 | ⚠️ 混合模式（注意口径一致） |
| `[T_query_start, T_query_end]` **完全早于** `T_agg_start` | 无数据 | ❌ 数据已过期 |

### 3.3 高级报表的有效数据范围

**关键修正**：高级报表不是"完全失效"，而是在**明细表保留窗口内**数据完整，超出部分无数据。

以**留存率报表** (`getRetention`) 为例：

```
假设：明细保留 90 天，查询 01-01 至 03-31 的留存率

时间轴（按天）：
  01-01  01-15  02-01  02-15  03-01  03-15  03-31
    ├──────┼──────┼──────┼──────┼──────┼──────┤
    │             │                           │
    │             └── T_raw_start (90天前)    │
    │                                         │
    └────────── 查询日期范围 ─────────────────┘

实际有效数据：
  01-01  01-15  02-01  02-15  03-01  03-15  03-31
    ├──────┼──────┼──────┼──────┼──────┼──────┤
    │ 无数据 │      └────── 有数据 ───────────┘
    └───────── 明细已删除 ──────────────────────
```

**结果表现**：
- 2月1日之前的 cohort（用户群）：留存率为 0 或空
- 2月1日之后的 cohort：留存率数据完整
- 报表整体显示为"前半部分空白，后半部分有数据"，而非完全空白

## 4. 查询回退到明细层的场景梳理

### 4.1 核心查询的路由逻辑

**聚合表优先，条件触发回退**，这是贯穿所有查询的通用模式：

```typescript
// 典型模式（以 getPageviewStats 为例）
if (EVENT_COLUMNS.some(item => Object.keys(filters).includes(item)) || unit === 'minute') {
  // 回退到明细：有特殊过滤 或 分钟级粒度
  sql = `SELECT ... FROM website_event ...`;
} else {
  // 使用聚合表：常规统计
  sql = `SELECT ... FROM website_event_stats_hourly ...`;
}
```

### 4.2 回退触发明细表的完整场景

#### A. 基础统计类查询

| 查询函数 | 文件路径 | 回退条件 | 数据边界影响 |
|---------|---------|---------|------------|
| `getPageviewStats` | `src/queries/sql/pageviews/getPageviewStats.ts:59-99` | `unit === 'minute'` 或使用 EVENT_COLUMNS 过滤 | 分钟级/过滤查询在 `T_raw_start` 前无数据 |
| `getWebsiteStats` | `src/queries/sql/getWebsiteStats.ts:86-135` | 使用 EVENT_COLUMNS 过滤 | 带过滤查询在 `T_raw_start` 前无数据 |
| `getWeeklyTraffic` | `src/queries/sql/getWeeklyTraffic.ts:56-86` | 使用 EVENT_COLUMNS 过滤 | 带过滤查询在 `T_raw_start` 前无数据 |
| `getEventStats` | `src/queries/sql/events/getEventStats.ts:103-137` | 有 filterQuery 或 cohortQuery | 带过滤查询在 `T_raw_start` 前无数据 |

#### B. 会话类查询

| 查询函数 | 文件路径 | 回退条件（修正版） | 数据边界影响 |
|---------|---------|------------------|------------|
| `getWebsiteSessions` | `src/queries/sql/sessions/getWebsiteSessions.ts:98-156` | **使用 EVENT_COLUMNS 过滤时回退**，否则使用聚合表 | 带 EVENT_COLUMNS 过滤的查询在 `T_raw_start` 前无数据；无过滤时数据完整 |
| `getSessionStats` | `src/queries/sql/sessions/getSessionStats.ts:30-88` | 使用 EVENT_COLUMNS 过滤 | 带过滤查询在 `T_raw_start` 前无数据 |
| `getSessionMetrics` | `src/queries/sql/sessions/getSessionMetrics.ts:54-120` | 使用 EVENT_COLUMNS 过滤 | 带过滤查询在 `T_raw_start` 前无数据 |
| `getWebsiteSessionStats` | `src/queries/sql/sessions/getWebsiteSessionStats.ts:44-88` | 使用 EVENT_COLUMNS 过滤 | 带过滤查询在 `T_raw_start` 前无数据 |

**会话列表查询回退条件修正说明**：
- 原描述"使用 EVENT_COLUMNS 过滤或分页查询"不准确
- 实际代码中：`getWebsiteSessions.ts:98` 只判断 `EVENT_COLUMNS.some(...)`
- 分页查询本身不触发回退，只有使用 EVENT_COLUMNS 字段过滤时才回退
- 无过滤的会话列表使用 `website_event_stats_hourly` 聚合表，数据完整

#### C. 高级报表类查询（**始终使用明细表**）

| 查询函数 | 文件路径 | 聚合表版本 | 数据边界影响 |
|---------|---------|----------|------------|
| `getRetention` | `src/queries/sql/reports/getRetention.ts:121-172` | ❌ 无 | `T_raw_start` 前的 cohort 数据缺失 |
| `getFunnel` | `src/queries/sql/reports/getFunnel.ts:325-348` | ❌ 无 | `T_raw_start` 前的漏斗步骤数据缺失 |
| `getJourney` | `src/queries/sql/reports/getJourney.ts` | ❌ 无 | `T_raw_start` 前的路径分析数据缺失 |
| `getGoal` | `src/queries/sql/reports/getGoal.ts` | ❌ 无 | `T_raw_start` 前的目标达成数据缺失 |
| `getBreakdown` | `src/queries/sql/reports/getBreakdown.ts` | ❌ 无 | `T_raw_start` 前的细分数据缺失 |
| `getAttribution` | `src/queries/sql/reports/getAttribution.ts` | ❌ 无 | `T_raw_start` 前的归因数据缺失 |
| `getUTM` | `src/queries/sql/reports/getUTM.ts` | ❌ 无 | `T_raw_start` 前的 UTM 分析数据缺失 |
| `getPerformance` | `src/queries/sql/reports/getPerformance.ts` | ❌ 无 | `T_raw_start` 前的性能指标缺失 |
| `getRevenue` / `getRevenueStats` / `getRevenueMetrics` | `src/queries/sql/reports/getRevenue*.ts` | ❌ 无 | `T_raw_start` 前的收入数据缺失 |

#### D. 明细数据类查询（**始终使用明细表**）

| 查询函数 | 文件路径 | 数据边界影响 |
|---------|---------|------------|
| `getSessionActivity` | `src/queries/sql/sessions/getSessionActivity.ts` | `T_raw_start` 前的会话活动无数据 |
| `getSessionDataProperties` / `getSessionDataValues` | `src/queries/sql/sessions/getSessionData*.ts` | `T_sessiondata_start` 前的会话属性无数据 |
| `getEventDataEvents` / `getEventDataStats` / `getEventDataUsage` 等 | `src/queries/sql/events/getEventData*.ts` | `T_eventdata_start` 前的事件数据无数据 |
| `getPageviewMetrics` / `getPageviewExpandedMetrics` | `src/queries/sql/pageviews/getPageviewMetrics*.ts` | `T_raw_start` 前的页面指标无数据 |
| `getActiveVisitors` | `src/queries/sql/getActiveVisitors.ts` | 通常查询实时窗口，不受影响 |
| `getValues` | `src/queries/sql/getValues.ts` | `T_raw_start` 前的字段值无数据 |
| `getSessionReplays` | `src/queries/sql/replays/getSessionReplays.ts` | `T_replay_start` 前的回放数据无数据 |

### 4.3 EVENT_COLUMNS 过滤字段详解

当查询中使用以下字段时，会**强制回退到明细表**：

```typescript
// src/lib/constants.ts:37-53
export const EVENT_COLUMNS = [
  'path',       // url_path - 页面路径
  'entry',      // url_path - 入口页面
  'exit',       // url_path - 出口页面
  'referrer',   // referrer_domain - 来源域名
  'domain',     // referrer_domain - 来源域名
  'title',      // page_title - 页面标题
  'query',      // url_query - URL 参数
  'event',      // event_name - 事件名
  'tag',        // tag - 标签
  'hostname',   // hostname - 主机名
  'utmSource',  // utm_source - UTM 来源
  'utmMedium',  // utm_medium - UTM 媒介
  'utmCampaign',// utm_campaign - UTM 活动
  'utmContent', // utm_content - UTM 内容
  'utmTerm',    // utm_term - UTM 关键词
];
```

**设计原因**：聚合表中这些字段使用 `SimpleAggregateFunction(groupArrayArray, ...)` 存储为数组，无法直接过滤。

## 5. 实时数据链路与缓存分析

### 5.1 实时数据链路架构

```
前端实时页面 → getRealtimeData API
                  │
                  ├─→ getRealtimeActivity (最近 100 条事件，明细表)
                  ├─→ getPageviewStats (可回退到聚合表)
                  └─→ getSessionStats (可回退到聚合表)
```

### 5.2 Redis 缓存的作用边界

**关键结论**：Redis **不缓存统计数据**，仅缓存配置信息和认证数据。

| 缓存类型 | 缓存键 | TTL | 说明 |
|---------|--------|-----|------|
| 网站配置 | `website:${websiteId}` | 86400秒 | 网站基本信息、域名、所有者 |
| 会话配置 | `session:${sessionId}` | 动态 | 会话元数据 |
| 认证令牌 | `auth:${token}` | 动态 | 登录状态 |
| 白标签配置 | `white-label:${accountId}` | 动态 | 品牌定制配置 |
| 团队信息 | `team:${teamId}` | 动态 | 团队成员和权限 |

**代码证据**：
```typescript
// src/lib/load.ts:9-10 - 仅缓存网站配置，不缓存统计数据
if (redis.enabled) {
  website = await redis.client.fetch(`website:${websiteId}`, () => getWebsite(websiteId), 86400);
}

// src/queries/sql/getRealtimeActivity.ts:51-81 - 实时活动直接查询 ClickHouse，无缓存
async function clickhouseQuery(websiteId: string, filters: QueryFilters): Promise<{ x: number }> {
  return rawQuery(
    `select ... from website_event ... order by createdAt desc limit 100`,
    queryParams,
    FUNCTION_NAME,
  );
}
```

### 5.3 实时数据的保留窗口影响

| 实时功能 | 数据来源 | 受保留窗口影响 | 说明 |
|---------|---------|--------------|------|
| 实时访客数 | getPageviewStats / getSessionStats | ⚠️ 部分 | 无过滤时用聚合表（完整），有过滤时用明细表（受限） |
| 实时活动日志 | getRealtimeActivity → website_event | ✅ 受限 | 仅显示保留窗口内的数据，默认取最近100条 |
| 国家/URL/来源统计 | getRealtimeActivity 内存聚合 | ✅ 受限 | 基于活动日志聚合，同样受窗口限制 |

**实时数据行为**：
- 实时页面查询的时间窗口通常很短（如 30 分钟）
- 只要保留窗口 > 实时查询窗口，功能不受影响
- 如果保留窗口设置过短（如 < 1小时），实时面板可能显示空白

## 6. 明细附属数据的保留边界与风险

### 6.1 附属数据表一览

| 表名 | 存储内容 | 关联主表 | 独立 TTL 配置 | 保留窗口建议 |
|------|---------|---------|--------------|------------|
| `event_data` | 自定义事件参数（键值对） | website_event.event_id | ✅ 可独立设置 | 90 天 |
| `session_data` | 自定义会话属性（键值对） | website_event.session_id | ✅ 可独立设置 | 90 天 |
| `session_replay` | 会话回放视频/操作记录 | website_event.session_id | ✅ 可独立设置 | 14 天（数据量大） |
| `website_revenue` | 收入/转化数据 | website_event.session_id | ✅ 可独立设置 | 365 天 |

### 6.2 Schema 定义与分区策略

```sql
-- db/clickhouse/schema.sql:57-75
CREATE TABLE umami.event_data (
    event_id UUID,
    website_id UUID,
    session_id UUID,
    data_key String,
    data_type Int8,
    string_value Nullable(String),
    number_value Nullable(Decimal64(4)),
    date_value Nullable(DateTime64(3)),
    created_at DateTime64(3) DEFAULT now64(3),
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)  -- 按月分区，可独立设置 TTL
ORDER BY (website_id, session_id, event_id)
TTL created_at + INTERVAL 90 DAY;  -- 建议配置

CREATE TABLE umami.session_replay (
    session_id UUID,
    website_id UUID,
    data String,  -- 回放数据，通常较大
    created_at DateTime64(3) DEFAULT now64(3),
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
TTL created_at + INTERVAL 14 DAY;  -- 建议更短的保留期
```

### 6.3 附属数据的查询依赖链

**场景1：事件详情查询**
```
getEventData (事件参数)
    ↓ JOIN
website_event (明细)
    ↓
T_eventdata_start 与 T_raw_start 取较早值
    ↓
早于该日期的事件参数查询返回空
```

**场景2：会话详情查询**
```
getSessionActivity (会话活动)
    ↓
getSessionData (会话属性)
    ↓
T_sessiondata_start 需 ≥ T_raw_start 才一致
    ↓
否则可能出现"会话存在但属性缺失"的异常
```

### 6.4 保留策略不一致的风险

| 风险场景 | 现象 | 原因 | 影响程度 |
|---------|------|------|---------|
| **事件参数早于明细删除** | 事件列表存在，但点击详情后参数空白 | TTL_eventdata < TTL_raw | ⚠️ 中等 |
| **会话属性早于明细删除** | 会话列表存在，但自定义属性丢失 | TTL_sessiondata < TTL_raw | ⚠️ 中等 |
| **回放数据早于明细删除** | 会话显示"有回放"但点击后无法播放 | TTL_replay < TTL_raw | ⚠️ 中等 |
| **收入数据早于明细删除** | 转化报表与收入报表数据不一致 | TTL_revenue < TTL_raw | ⚠️ 高 |
| **聚合数据早于明细删除** | 聚合统计数据早于明细消失（极罕见） | TTL_agg < TTL_raw | ❌ 严重 |

**风险规避建议**：
```sql
-- 确保附属数据 TTL ≥ 明细表 TTL
ALTER TABLE umami.event_data MODIFY TTL created_at + INTERVAL 90 DAY;
ALTER TABLE umami.session_data MODIFY TTL created_at + INTERVAL 90 DAY;
ALTER TABLE umami.website_event MODIFY TTL created_at + INTERVAL 90 DAY;

-- 回放数据可独立设置更短
ALTER TABLE umami.session_replay MODIFY TTL created_at + INTERVAL 14 DAY;
```

### 6.5 JOIN 查询的边界行为

```sql
-- getEventData.ts:89-107 - 左连接明细表
select event_data.event_id
from event_data
any left join (
  select event_id, session_id, website_id, event_name, created_at
  from website_event
  where website_id = {websiteId:UUID}
    and created_at between {startDate:DateTime64} and {endDate:DateTime64}
) website_event
on website_event.event_id = event_data.event_id
```

**边界行为**：
- 如果 `website_event` 中该 `event_id` 已因 TTL 删除，左连接后字段为 NULL
- `event_data` 本身的数据仍可查询到（只要未过期）
- 但事件名、会话信息等关联字段会显示为空

## 7. 统计空窗校准：聚合与明细的范围不匹配处理

### 7.1 空窗产生的场景

**场景1：查询时间跨明细保留边界**

用户选择查询最近 6 个月的数据，但明细表只保留 90 天：

```
查询范围：1月1日 ~ 6月30日（6个月）
实际可用：
  1月1日 ~ 3月31日：只有聚合数据（明细已删）
  4月1日 ~ 6月30日：明细完整

智能路由结果：
  - 无过滤、按天粒度：使用聚合表，6个月数据完整
  - 带 URL 过滤、按天粒度：4月1日前数据空白（回退明细）
  - 按分钟粒度：4月1日前数据空白（回退明细）
```

**场景2：报表日期与保留窗口不匹配**

留存率报表查询起始日期早于 `T_raw_start`：

```
留存分析：1月1日（Day 0）~ 1月31日（Day 30）
T_raw_start = 3月1日（90天前）

结果：
  Day 0 cohort（1月1日用户）：明细已删 → 无数据
  Day 30 回访数据：3月1日前的回访记录已删 → 无数据
  报表显示：留存率曲线在 3月1日前为空白
```

**场景3：附属数据与明细保留不一致**

事件参数保留 30 天，明细保留 90 天：
```
查询 60 天前的事件详情：
  - 事件在列表中存在（明细未删）
  - 但点击查看参数时显示空白（event_data 已删）
```

### 7.2 日期范围校准的代码实现

`getWebsiteDateRange` 函数在 ClickHouse 模式下**从聚合表读取日期范围**，而不是明细表：

```typescript
// src/queries/sql/getWebsiteDateRange.ts:35-54
async function clickhouseQuery(websiteId: string) {
  const result = await rawQuery(
    `
    select
      min(created_at) as startDate,
      max(created_at) as endDate
    from website_event_stats_hourly  -- 注意：从聚合表读取，而非明细表
    where website_id = {websiteId:UUID}
      and created_at >= {startDate:DateTime64}
    `,
    queryParams,
  );

  return result[0] ?? null;
}
```

**设计意图**：
- 给用户展示的"最早数据日期"是聚合表的范围（更长）
- 但实际执行明细查询时，早期数据实际不存在
- 这是造成"统计空窗"的根本原因之一

### 7.3 空窗校准建议

当前代码**未实现**跨边界查询的自动校准。以下是建议的处理策略：

**策略 A：前端日期限制**
```typescript
// 建议在前端实现：日期选择器的最小值设为 T_raw_start
const minDate = T_raw_start;  // 从后端获取明细表保留边界
const maxDate = today;
```

**策略 B：查询时分段**
```typescript
// 伪代码：跨边界查询的分段处理
function getHybridStats(queryStart, queryEnd, filters) {
  const rawStart = max(queryStart, T_raw_start);
  const aggEnd = min(queryEnd, T_raw_start);

  const aggData = queryAggregate(queryStart, aggEnd, filters);
  const rawData = queryRaw(rawStart, queryEnd, filters);

  return mergeData(aggData, rawData);
}
```

**策略 C：UI 提示**
- 在报表顶部显示："注：90天前的历史数据仅支持基础指标统计，高级分析功能可能不完整"
- 日期选择器中标注："完整数据从 XXXX-XX-XX 开始"
- 数据图表中用虚线或灰色标记数据不完整的区域

### 7.4 数据缺口的用户感知

**用户可能看到的异常**：
1. 报表页面"前半部分空白，后半部分有数据"（非完全空白）
2. 筛选特定 URL/事件后，历史数据突然减少
3. 时间范围选择器显示的最早日期（从聚合表）与实际可用数据（从明细表）不一致
4. 导出的数据在某一日期后突然有数据
5. 事件/会话详情页"参数缺失"或"回放不可用"但列表仍显示

**建议的 UI 提示**（当前代码未实现）：
- 在报表页面提示："历史数据仅保留 N 天，更早数据不可用"
- 在日期范围选择器中标注可用数据范围
- 数据图表中用虚线或灰色标记数据不完整的区域
- 详情页中对已过期的参数/回放显示"数据已归档"提示

## 8. 功能影响矩阵（修正版）

| 功能模块 | 子功能 | 数据完整性状态 | 详细说明 |
|---------|--------|--------------|---------|
| **仪表盘** | 核心指标（PV/UV/访问数/跳出率） | ✅ 完整 | 使用聚合表，不受明细删除影响 |
| | 页面浏览趋势图（小时/天/月，无过滤） | ✅ 完整 | 使用聚合表 |
| | 页面浏览趋势图（按分钟） | ⚠️ 局部缺失 | `T_raw_start` 前无数据 |
| | 页面浏览趋势图（带 URL/标题过滤） | ⚠️ 局部缺失 | `T_raw_start` 前无数据 |
| | 周流量热力图（无过滤） | ✅ 完整 | 使用聚合表 |
| | 周流量热力图（有过滤） | ⚠️ 局部缺失 | `T_raw_start` 前无数据 |
| **报表** | Retention 留存率 | ⚠️ 时间窗口内有效 | `T_raw_start` 前 cohort 数据缺失 |
| | Funnel 漏斗 | ⚠️ 时间窗口内有效 | `T_raw_start` 前漏斗步骤数据缺失 |
| | Journey 路径 | ⚠️ 时间窗口内有效 | `T_raw_start` 前路径数据缺失 |
| | Goal 目标 | ⚠️ 时间窗口内有效 | `T_raw_start` 前目标达成数据缺失 |
| | Breakdown 细分 | ⚠️ 时间窗口内有效 | `T_raw_start` 前细分数据缺失 |
| | Attribution 归因 | ⚠️ 时间窗口内有效 | `T_raw_start` 前归因数据缺失 |
| | UTM 分析 | ⚠️ 时间窗口内有效 | `T_raw_start` 前 UTM 数据缺失 |
| | Performance 性能 | ⚠️ 时间窗口内有效 | `T_raw_start` 前性能指标缺失 |
| | Revenue 收入 | ⚠️ 时间窗口内有效 | `T_revenue_start` 前收入数据缺失 |
| **实时** | 实时访客数（无过滤） | ✅ 完整 | 使用聚合表 |
| | 实时访客数（有过滤） | ⚠️ 局部缺失 | `T_raw_start` 前数据缺失 |
| | 实时活动日志 | ⚠️ 时间窗口内有效 | `T_raw_start` 前日志无数据 |
| **事件** | 事件统计（无过滤） | ✅ 完整 | 聚合表回退成功 |
| | 事件统计（有过滤） | ⚠️ 局部缺失 | `T_raw_start` 前数据缺失 |
| | 事件参数详情 | ⚠️ 时间窗口内有效 | `T_eventdata_start` 前参数无数据 |
| **会话** | 会话列表（无过滤） | ✅ 完整 | 使用聚合表（已修正） |
| | 会话列表（有 URL 过滤） | ⚠️ 局部缺失 | `T_raw_start` 前数据缺失 |
| | 会话详情/活动时间线 | ⚠️ 时间窗口内有效 | `T_raw_start` 前数据缺失 |
| | 会话自定义属性 | ⚠️ 时间窗口内有效 | `T_sessiondata_start` 前属性无数据 |
| **回放** | 会话回放 | ⚠️ 时间窗口内有效 | `T_replay_start` 前回放无数据 |
| **导出** | 数据导出 | ⚠️ 局部缺失 | `T_raw_start` 前数据无法导出 |

## 9. 代码中不显眼的取舍点

### 9.1 隐式的分层保留设计

代码中没有显式的 `retentionDays` 配置，而是通过**查询路由**间接实现：

- 聚合表查询路径：`src/queries/sql/*` 中 `clickhouseQuery` 函数的 else 分支
- 明细表查询路径：同文件的 if 分支
- 两者在代码中相邻但职责分离
- 运维可以独立配置两张表的 TTL，实现分层保留

### 9.2 聚合表的取舍边界

**聚合表不存储以下信息**（导致报表功能依赖明细表）：

| 缺失能力 | 原因 | 影响的功能 |
|---------|------|-----------|
| 原始 URL 路径 | 存储为去重数组，无法精确过滤 | Breakdown、Funnel、Journey |
| 事件参数 | 未聚合 event_data 表 | Goal、自定义事件分析 |
| 会话时间线 | 只有起止时间，无中间步骤 | Session Activity |
| 分钟级粒度 | 最细为小时级 | 实时趋势图、精细报表 |
| 事件间顺序 | 无法重建用户行为路径 | Journey、Funnel |

### 9.3 Prisma vs ClickHouse 的双写架构

在 `src/lib/db.ts:22-36` 中定义了查询路由：

```typescript
export async function runQuery(queries: any) {
  if (process.env.CLICKHOUSE_URL) {
    return queries[CLICKHOUSE]();  // ClickHouse 优先
  }
  return queries[PRISMA]();       // PostgreSQL 回退
}
```

这意味着：
- 启用 ClickHouse 后，聚合表成为主要查询来源
- PostgreSQL 中的明细数据可以更早清理
- 但 Prisma 模式下的重置/删除不影响 ClickHouse

### 9.4 Redis 不缓存统计数据的设计选择

**设计取舍**：
- Redis 仅用于配置和认证缓存，不用于统计数据
- 统计数据直接查询 ClickHouse（其本身就是列存数据库，查询速度快）
- 避免缓存与数据库的数据一致性问题
- 代价：实时查询每次都访问数据库，但 ClickHouse 足够快

## 10. 数据保留的实践建议

### 10.1 ClickHouse 部署的保留配置

对于生产环境部署，建议配置：

```sql
-- 明细数据保留 90 天
ALTER TABLE umami.website_event 
MODIFY TTL created_at + INTERVAL 90 DAY;

-- 聚合数据保留 3 年
ALTER TABLE umami.website_event_stats_hourly 
MODIFY TTL created_at + INTERVAL 3 YEAR;

-- 事件参数保留 90 天（与明细同步）
ALTER TABLE umami.event_data 
MODIFY TTL created_at + INTERVAL 90 DAY;

-- 会话属性保留 90 天（与明细同步）
ALTER TABLE umami.session_data 
MODIFY TTL created_at + INTERVAL 90 DAY;

-- 回放数据保留 14 天（数据量大）
ALTER TABLE umami.session_replay 
MODIFY TTL created_at + INTERVAL 14 DAY;

-- 收入数据保留 1 年
ALTER TABLE umami.website_revenue 
MODIFY TTL created_at + INTERVAL 365 DAY;
```

### 10.2 保留窗口与功能的权衡

| 业务场景 | 建议明细保留 | 建议聚合保留 | 功能影响 |
|---------|------------|------------|---------|
| 个人网站/博客 | 30 天 | 1 年 | 高级报表仅近 30 天数据可用 |
| 中小企业官网 | 90 天 | 3 年 | 高级报表仅近 90 天数据可用 |
| 电商/营销站点 | 180 天 | 5 年 | 高级报表仅近 180 天数据可用 |
| 数据合规要求严格 | 按法规 | 按法规 | 完全合规优先 |

### 10.3 删除后的统计一致性

删除明细数据后需要注意：

1. **报表功能受影响**：
   - Breakdown 报告（按字段细分）在 `T_raw_start` 前无数据
   - Retention、Funnel 等高级报告在 `T_raw_start` 前的 cohort 数据缺失

2. **导出功能受限**：
   - 事件导出只能导出保留窗口内的数据

3. **实时数据**：
   - 实时查询（30 分钟内）通常仍在保留范围内
   - Redis 不缓存统计数据，查询直接访问数据库

4. **附属数据一致性**：
   - 确保 event_data、session_data 的 TTL ≥ website_event 的 TTL
   - 避免出现"列表有数据但详情空白"的不一致现象

5. **日期范围展示校准**：
   - 前端应从明细表单独获取 `T_raw_start` 用于限制用户选择
   - 避免用户选择早于保留窗口的日期导致困惑

## 11. 关键代码位置速查

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| 聚合表 Schema | `db/clickhouse/schema.sql` | 94-143 |
| 物化视图定义 | `db/clickhouse/schema.sql` | 145-239 |
| event_data 表定义 | `db/clickhouse/schema.sql` | 57-75 |
| session_replay 表定义 | `db/clickhouse/schema.sql` | 291-322 |
| 页面浏览统计路由 | `src/queries/sql/pageviews/getPageviewStats.ts` | 59-99 |
| 会话列表查询路由（修正） | `src/queries/sql/sessions/getWebsiteSessions.ts` | 98-156 |
| 实时活动查询（无缓存） | `src/queries/sql/getRealtimeActivity.ts` | 51-81 |
| 事件参数查询 | `src/queries/sql/events/getEventData.ts` | 80-152 |
| Redis 配置缓存 | `src/lib/load.ts` | 9-26 |
| 留存率报表（用明细） | `src/queries/sql/reports/getRetention.ts` | 121-172 |
| 漏斗报表（用明细） | `src/queries/sql/reports/getFunnel.ts` | 325-348 |
| 日期范围查询（聚合表） | `src/queries/sql/getWebsiteDateRange.ts` | 42-54 |
| 网站重置（删除） | `src/queries/prisma/website.ts` | 133-186 |
| 数据库路由 | `src/lib/db.ts` | 22-36 |
| EVENT_COLUMNS 定义 | `src/lib/constants.ts` | 37-53 |
