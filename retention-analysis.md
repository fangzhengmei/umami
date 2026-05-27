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

### 1.2 物化视图的自动聚合

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
- `views`: 页面浏览量求和
- `min_time`/`max_time`: 访问起止时间
- `entry_url`/`exit_url`: 入口/出口页面
- 各维度字段的数组去重存储

## 2. 删除批次：分区策略与保留窗口

### 2.1 按月分区的物理删除机制

所有表都采用**按月分区**的设计：

```sql
PARTITION BY toYYYYMM(created_at)
```

涉及的表包括：
- `website_event` - 事件明细
- `website_event_stats_hourly` - 小时聚合
- `website_revenue` - 收入数据
- `session_replay` - 会话回放

**删除策略**：
- 物理删除通过 `ALTER TABLE ... DROP PARTITION` 实现
- 按月份批量删除，性能开销小
- 删除粒度：整月数据一次性清除

### 2.2 保留窗口的取舍逻辑

**代码中不显式配置 TTL，但架构设计天然支持分层保留**：

| 数据层级 | 建议保留窗口 | 用途 | 删除影响 |
|---------|-------------|------|---------|
| 明细数据 (website_event) | 短周期（如 30-90 天） | 细粒度查询、事件详情、自定义维度分析 | 无法进行分钟级查询、特定字段过滤时回退失败 |
| 聚合数据 (website_event_stats_hourly) | 长周期（如 1-3 年） | 常规统计报表、趋势分析、核心指标 | 影响常规统计功能 |
| 回放数据 (session_replay) | 极短周期（如 7-14 天） | 用户行为回放 | 功能不可用 |

## 3. 统计校准：查询时的智能路由

### 3.1 聚合表优先的查询策略

在 `getPageviewStats` 和 `getWebsiteStats` 等核心查询中实现了**智能路由**：

```typescript
// src/queries/sql/pageviews/getPageviewStats.ts:59-99
if (EVENT_COLUMNS.some(item => Object.keys(filters).includes(item)) || unit === 'minute') {
  // 使用明细数据：有特定字段过滤 或 按分钟粒度
  sql = `SELECT ... FROM website_event ...`;
} else {
  // 使用聚合表：常规统计查询
  sql = `SELECT ... FROM website_event_stats_hourly ...`;
}
```

**触发明细查询的条件**：
1. 查询粒度为 `minute`（聚合表只有小时级）
2. 使用了 `EVENT_COLUMNS` 中的字段过滤：
   - path, entry, exit, referrer, domain, title, query
   - event, tag, hostname
   - utmSource, utmMedium, utmCampaign, utmContent, utmTerm

### 3.2 统计校准的边界处理

**明细数据删除后的查询行为**：

1. **常规统计**（无特殊过滤、按小时/天/月）
   - 完全使用聚合表，不受明细删除影响
   - 统计结果完整准确

2. **细粒度查询**（分钟级）
   - 必须使用明细表
   - 删除后查询结果缺失该时段数据

3. **字段过滤查询**（如按 URL 路径过滤）
   - 必须使用明细表
   - 删除后该过滤条件下的统计数据缺失

**日期范围校准**：
```typescript
// src/queries/sql/getWebsiteDateRange.ts:42-54
// ClickHouse 模式下从聚合表获取日期范围
select min(created_at) as startDate, max(created_at) as endDate
from website_event_stats_hourly
```

## 4. 代码中不显眼的取舍点

### 4.1 隐式的分层保留设计

代码中没有显式的 `retentionDays` 配置，而是通过**查询路由**间接实现：

- 聚合表查询路径：`src/queries/sql/*` 中 `clickhouseQuery` 函数的 else 分支
- 明细表查询路径：同文件的 if 分支
- 两者在代码中相邻但职责分离

### 4.2 重置功能的全量删除

`resetWebsite` 函数（`src/queries/prisma/website.ts:133-186`）作为极端情况的处理：

```typescript
// 删除顺序：从子表到主表
await tx.sessionReplaySaved.deleteMany(...)
await tx.sessionReplay.deleteMany(...)
await tx.revenue.deleteMany(...)
await tx.eventData.deleteMany(...)
await tx.sessionData.deleteMany(...)
await tx.websiteEvent.deleteMany(...)
await tx.session.deleteMany(...)
```

**注意**：此函数只删除 PostgreSQL 中的数据，ClickHouse 数据需要单独处理。

### 4.3 Prisma vs ClickHouse 的双写架构

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

## 5. 数据保留的实践建议

### 5.1 ClickHouse 部署的保留配置

对于生产环境部署，建议配置：

```sql
-- 明细数据保留 90 天
ALTER TABLE umami.website_event 
MODIFY TTL created_at + INTERVAL 90 DAY;

-- 聚合数据保留 3 年
ALTER TABLE umami.website_event_stats_hourly 
MODIFY TTL created_at + INTERVAL 3 YEAR;

-- 回放数据保留 14 天
ALTER TABLE umami.session_replay 
MODIFY TTL created_at + INTERVAL 14 DAY;
```

### 5.2 删除后的统计一致性

删除明细数据后需要注意：

1. **报告功能受影响**：
   - Breakdown 报告（按字段细分）可能不完整
   - Retention、Funnel 等高级报告依赖明细数据

2. **导出功能受限**：
   - 事件导出只能导出保留窗口内的数据

3. **实时数据**：
   - 实时查询（30 分钟内）通常仍在内存/缓存中，不受影响

## 6. 关键代码位置速查

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| 聚合表 Schema | `db/clickhouse/schema.sql` | 94-143 |
| 物化视图定义 | `db/clickhouse/schema.sql` | 145-239 |
| 页面浏览统计路由 | `src/queries/sql/pageviews/getPageviewStats.ts` | 59-99 |
| 网站统计路由 | `src/queries/sql/getWebsiteStats.ts` | 86-135 |
| 网站重置（删除） | `src/queries/prisma/website.ts` | 133-186 |
| 数据库路由 | `src/lib/db.ts` | 22-36 |
| 日期范围查询 | `src/queries/sql/getWebsiteDateRange.ts` | 42-54 |
