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

## 3. 查询回退到明细层的场景梳理

### 3.1 核心查询的路由逻辑

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

### 3.2 回退触发明细表的完整场景

#### A. 基础统计类查询

| 查询函数 | 文件路径 | 回退条件 | 影响 |
|---------|---------|---------|------|
| `getPageviewStats` | `src/queries/sql/pageviews/getPageviewStats.ts:59-99` | `unit === 'minute'` 或使用 EVENT_COLUMNS 过滤 | 页面浏览趋势图 |
| `getWebsiteStats` | `src/queries/sql/getWebsiteStats.ts:86-135` | 使用 EVENT_COLUMNS 过滤 | 核心指标（PV/UV/跳出率） |
| `getWeeklyTraffic` | `src/queries/sql/getWeeklyTraffic.ts:56-86` | 使用 EVENT_COLUMNS 过滤 | 周流量热力图 |
| `getEventStats` | `src/queries/sql/events/getEventStats.ts:103-137` | 有 filterQuery 或 cohortQuery | 事件统计图表 |

#### B. 会话类查询

| 查询函数 | 文件路径 | 回退条件 | 影响 |
|---------|---------|---------|------|
| `getWebsiteSessions` | `src/queries/sql/sessions/getWebsiteSessions.ts:53-147` | 使用 EVENT_COLUMNS 过滤或分页查询 | 会话列表 |
| `getSessionStats` | `src/queries/sql/sessions/getSessionStats.ts:30-88` | 使用 EVENT_COLUMNS 过滤 | 会话统计 |
| `getSessionMetrics` | `src/queries/sql/sessions/getSessionMetrics.ts:54-120` | 使用 EVENT_COLUMNS 过滤 | 会话指标 |
| `getWebsiteSessionStats` | `src/queries/sql/sessions/getWebsiteSessionStats.ts:44-88` | 使用 EVENT_COLUMNS 过滤 | 网站会话统计 |

#### C. 高级报表类查询（**始终使用明细表**）

| 查询函数 | 文件路径 | 是否有聚合表版本 | 影响 |
|---------|---------|----------------|------|
| `getRetention` | `src/queries/sql/reports/getRetention.ts:121-172` | ❌ 无 | 留存率报表**完全不可用** |
| `getFunnel` | `src/queries/sql/reports/getFunnel.ts:325-348` | ❌ 无 | 漏斗报表**完全不可用** |
| `getJourney` | `src/queries/sql/reports/getJourney.ts` | ❌ 无 | 路径分析**完全不可用** |
| `getGoal` | `src/queries/sql/reports/getGoal.ts` | ❌ 无 | 目标追踪**完全不可用** |
| `getBreakdown` | `src/queries/sql/reports/getBreakdown.ts` | ❌ 无 | 细分报表**完全不可用** |
| `getAttribution` | `src/queries/sql/reports/getAttribution.ts` | ❌ 无 | 归因分析**完全不可用** |
| `getUTM` | `src/queries/sql/reports/getUTM.ts` | ❌ 无 | UTM 报表**完全不可用** |
| `getPerformance` | `src/queries/sql/reports/getPerformance.ts` | ❌ 无 | 性能报表**完全不可用** |
| `getRevenue` / `getRevenueStats` / `getRevenueMetrics` | `src/queries/sql/reports/getRevenue*.ts` | ❌ 无 | 收入报表**完全不可用** |

#### D. 明细数据类查询（**始终使用明细表**）

| 查询函数 | 文件路径 | 影响 |
|---------|---------|------|
| `getSessionActivity` | `src/queries/sql/sessions/getSessionActivity.ts` | 会话活动时间线 |
| `getSessionDataProperties` / `getSessionDataValues` | `src/queries/sql/sessions/getSessionData*.ts` | 会话属性数据 |
| `getEventDataEvents` / `getEventDataStats` / `getEventDataUsage` 等 | `src/queries/sql/events/getEventData*.ts` | 事件数据详情 |
| `getPageviewMetrics` / `getPageviewExpandedMetrics` | `src/queries/sql/pageviews/getPageviewMetrics*.ts` | 页面浏览指标 |
| `getActiveVisitors` | `src/queries/sql/getActiveVisitors.ts` | 实时活跃访客 |
| `getValues` | `src/queries/sql/getValues.ts` | 字段去重值列表 |
| `getSessionReplays` | `src/queries/sql/replays/getSessionReplays.ts` | 会话回放列表 |

### 3.3 EVENT_COLUMNS 过滤字段详解

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

## 4. 统计数据缺口的影响评估

### 4.1 明细删除后的功能影响矩阵

| 功能模块 | 子功能 | 明细删除后状态 | 原因 |
|---------|--------|--------------|------|
| **仪表盘** | 核心指标（PV/UV/访问数/跳出率） | ✅ 正常 | 使用聚合表 |
| | 页面浏览趋势图（按小时/天/月） | ✅ 正常 | 使用聚合表 |
| | 页面浏览趋势图（按分钟） | ❌ 缺口 | 使用明细表 |
| | 周流量热力图（无过滤） | ✅ 正常 | 使用聚合表 |
| | 周流量热力图（有 URL/标题过滤） | ❌ 缺口 | 回退明细表 |
| **报表** | Retention 留存率 | ❌ **完全失效** | 始终用明细表 |
| | Funnel 漏斗 | ❌ **完全失效** | 始终用明细表 |
| | Journey 路径 | ❌ **完全失效** | 始终用明细表 |
| | Goal 目标 | ❌ **完全失效** | 始终用明细表 |
| | Breakdown 细分 | ❌ **完全失效** | 始终用明细表 |
| | Attribution 归因 | ❌ **完全失效** | 始终用明细表 |
| | UTM 分析 | ❌ **完全失效** | 始终用明细表 |
| | Performance 性能 | ❌ **完全失效** | 始终用明细表 |
| | Revenue 收入 | ❌ **完全失效** | 始终用明细表 |
| **实时** | 实时访客数 | ✅ 正常 | 短时间窗口查询 |
| | 实时活动日志 | ❌ 缺口 | 明细表 |
| **事件** | 事件统计（无过滤） | ✅ 正常 | 聚合表回退 |
| | 事件统计（有过滤） | ❌ 缺口 | 明细表 |
| | 事件数据详情 | ❌ 缺口 | 明细表 |
| **会话** | 会话列表（无过滤） | ✅ 正常 | 聚合表回退 |
| | 会话列表（有 URL 过滤） | ❌ 缺口 | 明细表 |
| | 会话详情/活动时间线 | ❌ 缺口 | 明细表 |
| **回放** | 会话回放 | ❌ **完全失效** | 明细表 + 回放表 |
| **导出** | 数据导出 | ❌ 缺口 | 明细表 |

### 4.2 数据缺口的用户感知

**用户可能看到的异常**：
1. 报表页面显示"无数据"或空白图表
2. 筛选特定 URL/事件后数据突然减少
3. 时间范围选择器中早于保留窗口的日期灰显或无数据
4. 导出的数据缺失历史记录

**建议的 UI 提示**（当前代码未实现）：
- 在报表页面提示："历史数据仅保留 N 天，更早数据不可用"
- 在日期范围选择器中标注可用数据范围

## 5. 代码中不显眼的取舍点

### 5.1 隐式的分层保留设计

代码中没有显式的 `retentionDays` 配置，而是通过**查询路由**间接实现：

- 聚合表查询路径：`src/queries/sql/*` 中 `clickhouseQuery` 函数的 else 分支
- 明细表查询路径：同文件的 if 分支
- 两者在代码中相邻但职责分离
- 运维可以独立配置两张表的 TTL，实现分层保留

### 5.2 聚合表的取舍边界

**聚合表不存储以下信息**（导致报表功能依赖明细表）：

| 缺失能力 | 原因 | 影响的功能 |
|---------|------|-----------|
| 原始 URL 路径 | 存储为去重数组，无法精确过滤 | Breakdown、Funnel、Journey |
| 事件参数 | 未聚合 event_data 表 | Goal、自定义事件分析 |
| 会话时间线 | 只有起止时间，无中间步骤 | Session Activity |
| 分钟级粒度 | 最细为小时级 | 实时趋势图、精细报表 |
| 事件间顺序 | 无法重建用户行为路径 | Journey、Funnel |

### 5.3 Prisma vs ClickHouse 的双写架构

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

## 6. 数据保留的实践建议

### 6.1 ClickHouse 部署的保留配置

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

### 6.2 保留窗口与功能的权衡

| 业务场景 | 建议明细保留 | 建议聚合保留 | 受损功能 |
|---------|------------|------------|---------|
| 个人网站/博客 | 30 天 | 1 年 | 高级报表仅近 30 天 |
| 中小企业官网 | 90 天 | 3 年 | 高级报表仅近 90 天 |
| 电商/营销站点 | 180 天 | 5 年 | 高级报表仅近 180 天 |
| 数据合规要求严格 | 按法规 | 按法规 | 完全合规优先 |

### 6.3 删除后的统计一致性

删除明细数据后需要注意：

1. **报表功能受影响**：
   - Breakdown 报告（按字段细分）可能不完整
   - Retention、Funnel 等高级报告依赖明细数据

2. **导出功能受限**：
   - 事件导出只能导出保留窗口内的数据

3. **实时数据**：
   - 实时查询（30 分钟内）通常仍在内存/缓存中，不受影响

## 7. 关键代码位置速查

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| 聚合表 Schema | `db/clickhouse/schema.sql` | 94-143 |
| 物化视图定义 | `db/clickhouse/schema.sql` | 145-239 |
| 页面浏览统计路由 | `src/queries/sql/pageviews/getPageviewStats.ts` | 59-99 |
| 网站统计路由 | `src/queries/sql/getWebsiteStats.ts` | 86-135 |
| 周流量统计路由 | `src/queries/sql/getWeeklyTraffic.ts` | 56-86 |
| 事件统计路由 | `src/queries/sql/events/getEventStats.ts` | 103-137 |
| 留存率报表（始终用明细） | `src/queries/sql/reports/getRetention.ts` | 121-172 |
| 漏斗报表（始终用明细） | `src/queries/sql/reports/getFunnel.ts` | 325-348 |
| 网站重置（删除） | `src/queries/prisma/website.ts` | 133-186 |
| 数据库路由 | `src/lib/db.ts` | 22-36 |
| 日期范围查询 | `src/queries/sql/getWebsiteDateRange.ts` | 42-54 |
| EVENT_COLUMNS 定义 | `src/lib/constants.ts` | 37-53 |
