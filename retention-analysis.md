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
- `T_query_start`：查询起始日期
- `T_query_end`：查询结束日期

**典型分层保留配置示例**：
```
聚合表保留 3 年  (T_agg_start = 3年前)
明细表保留 90 天 (T_raw_start = 90天前)

时间轴：
  ──┼──────────────────────────────────────────┼──────────────────┼──
    T_agg_start (3年前)                        T_raw_start (90天前)  Today
    │                                            │                    │
    └──────────── 聚合数据完整范围 ──────────────┘                    │
                                                 └── 明细数据范围 ───┘
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

| 查询函数 | 文件路径 | 回退条件 | 数据边界影响 |
|---------|---------|---------|------------|
| `getWebsiteSessions` | `src/queries/sql/sessions/getWebsiteSessions.ts:53-147` | 使用 EVENT_COLUMNS 过滤或分页查询 | 列表查询在 `T_raw_start` 前无数据 |
| `getSessionStats` | `src/queries/sql/sessions/getSessionStats.ts:30-88` | 使用 EVENT_COLUMNS 过滤 | 带过滤查询在 `T_raw_start` 前无数据 |
| `getSessionMetrics` | `src/queries/sql/sessions/getSessionMetrics.ts:54-120` | 使用 EVENT_COLUMNS 过滤 | 带过滤查询在 `T_raw_start` 前无数据 |
| `getWebsiteSessionStats` | `src/queries/sql/sessions/getWebsiteSessionStats.ts:44-88` | 使用 EVENT_COLUMNS 过滤 | 带过滤查询在 `T_raw_start` 前无数据 |

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
| `getSessionDataProperties` / `getSessionDataValues` | `src/queries/sql/sessions/getSessionData*.ts` | `T_raw_start` 前的会话属性无数据 |
| `getEventDataEvents` / `getEventDataStats` / `getEventDataUsage` 等 | `src/queries/sql/events/getEventData*.ts` | `T_raw_start` 前的事件数据无数据 |
| `getPageviewMetrics` / `getPageviewExpandedMetrics` | `src/queries/sql/pageviews/getPageviewMetrics*.ts` | `T_raw_start` 前的页面指标无数据 |
| `getActiveVisitors` | `src/queries/sql/getActiveVisitors.ts` | 通常查询实时窗口，不受影响 |
| `getValues` | `src/queries/sql/getValues.ts` | `T_raw_start` 前的字段值无数据 |
| `getSessionReplays` | `src/queries/sql/replays/getSessionReplays.ts` | `T_raw_start` 前的回放数据无数据 |

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

## 5. 统计空窗校准：聚合与明细的范围不匹配处理

### 5.1 空窗产生的场景

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

### 5.2 日期范围校准的代码实现

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

### 5.3 空窗校准建议

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

### 5.4 数据缺口的用户感知

**用户可能看到的异常**：
1. 报表页面"前半部分空白，后半部分有数据"（非完全空白）
2. 筛选特定 URL/事件后，历史数据突然减少
3. 时间范围选择器显示的最早日期（从聚合表）与实际可用数据（从明细表）不一致
4. 导出的数据在某一日期后突然有数据

**建议的 UI 提示**（当前代码未实现）：
- 在报表页面提示："历史数据仅保留 N 天，更早数据不可用"
- 在日期范围选择器中标注可用数据范围
- 数据图表中用虚线或灰色标记数据不完整的区域

## 6. 功能影响矩阵（修正版）

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
| | Revenue 收入 | ⚠️ 时间窗口内有效 | `T_raw_start` 前收入数据缺失 |
| **实时** | 实时访客数 | ✅ 正常 | 短时间窗口查询，通常在保留范围内 |
| | 实时活动日志 | ⚠️ 局部缺失 | `T_raw_start` 前日志无数据 |
| **事件** | 事件统计（无过滤） | ✅ 完整 | 聚合表回退成功 |
| | 事件统计（有过滤） | ⚠️ 局部缺失 | `T_raw_start` 前数据缺失 |
| | 事件数据详情 | ⚠️ 局部缺失 | `T_raw_start` 前数据缺失 |
| **会话** | 会话列表（无过滤） | ✅ 完整 | 聚合表回退成功 |
| | 会话列表（有 URL 过滤） | ⚠️ 局部缺失 | `T_raw_start` 前数据缺失 |
| | 会话详情/活动时间线 | ⚠️ 局部缺失 | `T_raw_start` 前数据缺失 |
| **回放** | 会话回放 | ⚠️ 时间窗口内有效 | 回放表独立 TTL，通常更短 |
| **导出** | 数据导出 | ⚠️ 局部缺失 | `T_raw_start` 前数据无法导出 |

## 7. 代码中不显眼的取舍点

### 7.1 隐式的分层保留设计

代码中没有显式的 `retentionDays` 配置，而是通过**查询路由**间接实现：

- 聚合表查询路径：`src/queries/sql/*` 中 `clickhouseQuery` 函数的 else 分支
- 明细表查询路径：同文件的 if 分支
- 两者在代码中相邻但职责分离
- 运维可以独立配置两张表的 TTL，实现分层保留

### 7.2 聚合表的取舍边界

**聚合表不存储以下信息**（导致报表功能依赖明细表）：

| 缺失能力 | 原因 | 影响的功能 |
|---------|------|-----------|
| 原始 URL 路径 | 存储为去重数组，无法精确过滤 | Breakdown、Funnel、Journey |
| 事件参数 | 未聚合 event_data 表 | Goal、自定义事件分析 |
| 会话时间线 | 只有起止时间，无中间步骤 | Session Activity |
| 分钟级粒度 | 最细为小时级 | 实时趋势图、精细报表 |
| 事件间顺序 | 无法重建用户行为路径 | Journey、Funnel |

### 7.3 Prisma vs ClickHouse 的双写架构

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

## 8. 数据保留的实践建议

### 8.1 ClickHouse 部署的保留配置

对于生产环境部署，建议配置：

```sql
-- 明细数据保留 90 天
ALTER TABLE umami.website_event 
MODIFY TTL created_at + INTERVAL 90 DAY;

-- 聚合数据保留 3 年
ALTER TABLE umami.website_event_stats_hourly 
MODIFY TTL created_at + INTERVAL 3 YEAR;

-- 回放数据保留 14 天（数据量大）
ALTER TABLE umami.session_replay 
MODIFY TTL created_at + INTERVAL 14 DAY;

-- 事件数据保留 90 天
ALTER TABLE umami.event_data 
MODIFY TTL created_at + INTERVAL 90 DAY;
```

### 8.2 保留窗口与功能的权衡

| 业务场景 | 建议明细保留 | 建议聚合保留 | 功能影响 |
|---------|------------|------------|---------|
| 个人网站/博客 | 30 天 | 1 年 | 高级报表仅近 30 天数据可用 |
| 中小企业官网 | 90 天 | 3 年 | 高级报表仅近 90 天数据可用 |
| 电商/营销站点 | 180 天 | 5 年 | 高级报表仅近 180 天数据可用 |
| 数据合规要求严格 | 按法规 | 按法规 | 完全合规优先 |

### 8.3 删除后的统计一致性

删除明细数据后需要注意：

1. **报表功能受影响**：
   - Breakdown 报告（按字段细分）在 `T_raw_start` 前无数据
   - Retention、Funnel 等高级报告在 `T_raw_start` 前的 cohort 数据缺失

2. **导出功能受限**：
   - 事件导出只能导出保留窗口内的数据

3. **实时数据**：
   - 实时查询（30 分钟内）通常仍在内存/缓存中，不受影响

4. **日期范围展示校准**：
   - 前端应从明细表单独获取 `T_raw_start` 用于限制用户选择
   - 避免用户选择早于保留窗口的日期导致困惑

## 9. 关键代码位置速查

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| 聚合表 Schema | `db/clickhouse/schema.sql` | 94-143 |
| 物化视图定义 | `db/clickhouse/schema.sql` | 145-239 |
| 页面浏览统计路由 | `src/queries/sql/pageviews/getPageviewStats.ts` | 59-99 |
| 网站统计路由 | `src/queries/sql/getWebsiteStats.ts` | 86-135 |
| 周流量统计路由 | `src/queries/sql/getWeeklyTraffic.ts` | 56-86 |
| 事件统计路由 | `src/queries/sql/events/getEventStats.ts` | 103-137 |
| 留存率报表（用明细） | `src/queries/sql/reports/getRetention.ts` | 121-172 |
| 漏斗报表（用明细） | `src/queries/sql/reports/getFunnel.ts` | 325-348 |
| 日期范围查询（聚合表） | `src/queries/sql/getWebsiteDateRange.ts` | 42-54 |
| 网站重置（删除） | `src/queries/prisma/website.ts` | 133-186 |
| 数据库路由 | `src/lib/db.ts` | 22-36 |
| EVENT_COLUMNS 定义 | `src/lib/constants.ts` | 37-53 |
