# Umami 自定义事件属性：读写聚合 & ClickHouse 表/索引/查询路径精确对照

> 本文档所有源码引用使用**仓库根目录相对路径**，便于随仓库迁移。行号标注为定位参考。

---

## 一、ClickHouse 表清单与索引策略（精确 DDL）

所有表 DDL 定义于 `db/clickhouse/schema.sql`。

### 1.1 事件主表：`umami.website_event`

**文件位置**：[db/clickhouse/schema.sql](db/clickhouse/schema.sql)（约第 2–55 行）

```sql
ENGINE = MergeTree
PARTITION BY toYYYYMM(created_at)
ORDER BY (toStartOfHour(created_at), website_id, session_id, visit_id, created_at)
PRIMARY KEY (toStartOfHour(created_at), website_id, session_id, visit_id)
SETTINGS index_granularity = 8192;
```

**排序键 = 稀疏索引（primary key）= 物理组织顺序**（ClickHouse 无 BTree 索引，靠 ORDER BY + mark 文件做粒度化跳转）：

| 位次 | 键列 | 覆盖的典型查询前缀 |
|---|---|---|
| 1 | `toStartOfHour(created_at)` | 任何时间范围（先按整小时粗筛 mark） |
| 2 | `website_id` | 所有查询必带，强命中 |
| 3 | `session_id` | 会话级 JOIN / cohort 留存 |
| 4 | `visit_id` | 访问级去重 / 明细 |
| 5 | `created_at` | 末位时间范围下推（mark 内二次裁剪） |

**投影索引（Projections）**：

| 投影名 | 重排键 | 命中场景 |
|---|---|---|
| `website_event_url_path_projection` | `toStartOfDay(created_at), website_id, url_path, created_at` | 按页面路径聚合报表 |
| `website_event_referrer_domain_projection` | `toStartOfDay(created_at), website_id, referrer_domain, created_at` | 按来源域名聚合报表 |

### 1.2 事件属性表：`umami.event_data`（属性明细表）

**文件位置**：[db/clickhouse/schema.sql](db/clickhouse/schema.sql)（约第 57–74 行）

```sql
ENGINE = MergeTree
ORDER BY (website_id, event_id, data_key, created_at)
SETTINGS index_granularity = 8192;
```

> ⚠️ **注意**：此表**不分区**（无 PARTITION BY），也未显式声明 PRIMARY KEY，因此 ORDER BY 同时承担物理排序 + 稀疏索引双重角色。

#### 排序键的物理布局直观理解

MergeTree 数据按排序键逐级分组，物理存储顺序如下：

```
website_id = W1                     ← 第 1 层（等值 → mark 可跳）
  ├─ event_id = E1                  ← 第 2 层（等值 → mark 可跳）
  │    ├─ data_key = "plan"         ← 第 3 层（仅在同一 event_id 内有序）
  │    │    └─ created_at = t1
  │    ├─ data_key = "price"
  │    │    └─ created_at = t1
  │    └─ data_key = "currency"
  │         └─ created_at = t1
  ├─ event_id = E2
  │    ├─ data_key = "plan"
  │    └─ ...
  └─ event_id = E3
       └─ ...
```

#### 连续前缀匹配规则（关键！）

ClickHouse MergeTree 的稀疏索引（mark）只能按**排序键的连续前缀**跳转——**跳列不生效**。

| 查询 WHERE 条件 | 实际命中的排序键前缀 | 说明 |
|---|---|---|
| `website_id = ?` | 第 1 列（完全命中） | 整站范围 mark 跳跃，最粗粒度 |
| `website_id = ? AND event_id = ?` | 第 1+2 列（完全命中） | 单事件范围，mark 定位极精准，**最佳场景** |
| `website_id = ? AND created_at BETWEEN ? AND ?` | 仅第 1 列（created_at 被 2、3 列隔开） | ❌ created_at 在第 4 位，跳 2、3 列后无法直接用于 mark 跳跃。**hasData 子查询正是此模式**——只能按 website_id 粗定位后扫全站数据再过滤时间 |
| `website_id = ? AND data_key = ?` | 仅第 1 列（data_key 被跳过） | ❌ **常见误解**：data_key 是第 3 列，被 event_id 隔开，不能直接走前缀；需扫完 website_id 下全部 event_id 再过滤 |
| `website_id = ? AND event_id = ? AND data_key = ?` | 第 1+2+3 列（完全命中） | 单事件 + 单属性，mark 极度精准 |
| `website_id = ? AND event_id = ? AND data_key = ? AND created_at BETWEEN ?` | 第 1+2+3 列完全命中，第 4 列在 mark 内下推 | 理论最优，但同 event 的 created_at 基本相同，收益有限 |

**结论**：`event_data` 的排序键设计**只为两类查询服务**：
1. **按 website_id 全量扫**（大多数属性聚合查询的常见形态）——仅利用第 1 列做租户隔离。
2. **按 website_id + event_id 查单事件的所有属性**（事件详情展开）——前两列精准命中，mark 跳跃零成本。

`data_key` 和 `created_at` 只有在其**前面所有列都是等值条件**时才能发挥索引作用。

### 1.3 会话属性表：`umami.session_data`

**文件位置**：[db/clickhouse/schema.sql](db/clickhouse/schema.sql)（约第 76–91 行）

```sql
ENGINE = ReplacingMergeTree
ORDER BY (website_id, session_id, data_key)
SETTINGS index_granularity = 8192;
```

用于 `identify()` 调用的 user/session 级属性（不在事件报表范围内，此处仅作索引对照）。

### 1.4 小时级物化聚合表：`umami.website_event_stats_hourly`

**文件位置**：[db/clickhouse/schema.sql](db/clickhouse/schema.sql)（约第 94–143 行）

```sql
ENGINE = AggregatingMergeTree
PARTITION BY toYYYYMM(created_at)
ORDER BY (website_id, event_type, toStartOfHour(created_at), cityHash64(visit_id), visit_id)
SAMPLE BY cityHash64(visit_id);
```

**写入来源**：由 `website_event_stats_hourly_mv` 物化视图基于 `website_event` 实时写入，见 [db/clickhouse/schema.sql](db/clickhouse/schema.sql)（约第 145–239 行）。

**关键**：此表**不含自定义事件属性**，仅聚合 event_name 数组（`SimpleAggregateFunction(groupArrayArray, Array(String))`）；属性聚合只能回源 `event_data` + `website_event` JOIN。

### 1.5 收益宽表：`umami.website_revenue`

**文件位置**：[db/clickhouse/schema.sql](db/clickhouse/schema.sql)（约第 257–289 行）

```sql
ENGINE = MergeTree
PARTITION BY toYYYYMM(created_at)
ORDER BY (website_id, session_id, created_at)
```

**来源**：`website_revenue_mv` 物化视图自动从 `event_data` 抽取 **data_key 含 'revenue' + data_key 含 'currency'** 的两条行合成一行，无需应用层写入。

---

## 二、写入路径完整链路（属性采集 → CH 落盘）

### 2.1 端到端时序

```
浏览器端
 └─ src/tracker/index.js track(name, data) 函数
    └─ POST /api/send  body: { type:'event', payload:{ name, data, ... } }
服务端采集路由
 └─ src/app/api/send/route.ts
     │  ① event_type = name ? customEvent(2) : pageView(1)
     │  ② payload.data  →  eventData 参数
     └─ saveEvent(args)  →  src/queries/sql/events/saveEvent.ts
        ├─ 写 website_event（获取 eventId=uuid()）
        ├─ 若 eventData 非空 → saveEventData()
        │     └─ src/queries/sql/events/saveEventData.ts
        │        ├─ flattenJSON(eventData)  →  [{key,value,dataType}...]
        │        ├─ kafka.enabled  →  kafka.sendMessage('event_data', ...)
        │        └─ 否则 clickhouse.insert('event_data', ...)
        └─ 若 eventData.revenue>0 && currency → saveRevenue()
```

### 2.2 JSON 扁平化 + 类型分类

核心代码位于 [src/lib/data.ts](src/lib/data.ts)。

- `flattenJSON`：嵌套对象用 `.` 拼接键（`{a:{b:1}} → a.b`）；数组整体 JSON.stringify；日期字符串通过 `DATETIME_REGEX` 正则识别。
- `createKey`：将 typeof 值映射到 `DATA_TYPE` 枚举（见 2.3），写入时数值统一 `toFixed(4)`、日期统一 ISO。

### 2.3 属性类型枚举

定义于 [src/lib/constants.ts](src/lib/constants.ts)：

| 常量 | 编码 | DB 列选择 | 示例 |
|---|---|---|---|
| `DATA_TYPE.string` | 1 | `string_value` | `"pro"` |
| `DATA_TYPE.number` | 2 | `number_value` + `string_value`(toFixed4) | `99.9900` |
| `DATA_TYPE.boolean` | 3 | `string_value`("true"/"false") | `"true"` |
| `DATA_TYPE.date` | 4 | `date_value` + `string_value`(ISO) | `2025-01-01T00:00:00.000Z` |
| `DATA_TYPE.array` | 5 | `string_value`(JSON) | `"[\"a\",\"b\"]"` |

### 2.4 ClickHouse event_data 行写入结构

每行 = 一个属性（即一个事件 N 属性写 N 行），写入时即冗余 `event_name / session_id / url_path` 到 event_data 表——这是 ClickHouse 模式下的"宽表"设计，避免属性聚合时频繁 JOIN website_event。

```typescript
{
  website_id, session_id, event_id, url_path, event_name,  // 冗余列
  data_key,       // 扁平化后属性名，如 "user.address.city"
  data_type,      // 1~5 枚举
  string_value,   // 所有类型均有（统一可检索表达）
  number_value,   // 仅 DATA_TYPE.number 非空 (Decimal22(4))
  date_value,     // 仅 DATA_TYPE.date 非空 (DateTime UTC)
  created_at,     // 与 website_event.created_at 一致
}
```

写入逻辑见 [src/queries/sql/events/saveEventData.ts](src/queries/sql/events/saveEventData.ts) 中 `clickhouseQuery` 函数。

---

## 三、报表查询路径 ↔ ClickHouse 表 ↔ 排序键命中 对照表

> 下表对每个报表场景精确标注：实际访问哪张 CH 表、命中排序键的哪几列前缀。

### 3.1 事件页总览（4 指标卡）

| 层级 | 模块 | 文件路径 | CH 主表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端 Hook | `useEventStatsQuery` | [src/components/hooks/queries/useEventStatsQuery.ts](src/components/hooks/queries/useEventStatsQuery.ts) | - | - |
| API 路由 | GET `/websites/:id/events/stats` | [src/app/api/websites/[websiteId]/events/stats/route.ts](src/app/api/websites/%5BwebsiteId%5D/events/stats/route.ts) | - | - |
| 查询函数 | `getWebsiteEventStats` | [src/queries/sql/events/getWebsiteEventStats.ts](src/queries/sql/events/getWebsiteEventStats.ts) | **website_event** | 第 1+2 列：`toStartOfHour(created_at), website_id` |

CH 方言 SQL 核心（见 clickhouseQuery 分支）：
```sql
SELECT sum(c) events, uniq(session_id) visitors, uniq(visit_id) visits,
       count(distinct event_name) uniqueEvents
FROM (SELECT session_id, visit_id, event_name, count(*) c
      FROM website_event
      WHERE website_id = ? AND created_at BETWEEN ? AND ? AND event_type = 2
      GROUP BY session_id, visit_id, event_name) t
```

### 3.2 事件趋势图（Chart Tab）

| 层级 | 模块 | 文件路径 | CH 主表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<EventsChart>` | [src/components/metrics/EventsChart.tsx](src/components/metrics/EventsChart.tsx) | - | - |
| 查询函数 | `getEventStats` | [src/queries/sql/events/getEventStats.ts](src/queries/sql/events/getEventStats.ts) | **条件分支：**<br>① 无过滤/无 cohort → **website_event_stats_hourly**（物化表快路径）<br>② 有过滤/cohort → **website_event**（回源明细） | ① hourly: `website_id, event_type, toStartOfHour` 前 3 列<br>② we: `toStartOfHour(created_at), website_id` 前 2 列 |

**关键加速路径**：若 `parseFilters` 产出的 `filterQuery` 和 `cohortQuery` 都为空，直接用**小时物化表** `arrayJoin(event_name)` 展开，避免扫明细。详见 `clickhouseQuery` 函数。

### 3.3 MetricsTable（事件类型 Top N 排行）

| 层级 | 模块 | 文件路径 | CH 主表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<MetricsTable type="event">` | [src/components/metrics/MetricsTable.tsx](src/components/metrics/MetricsTable.tsx) | - | - |
| 前端 Hook | `useWebsiteMetricsQuery` (type=event) | [src/components/hooks/queries/useWebsiteMetricsQuery.ts](src/components/hooks/queries/useWebsiteMetricsQuery.ts) | - | - |
| 查询函数 | `getEventMetrics` (column=event_name) | [src/queries/sql/events/getEventMetrics.ts](src/queries/sql/events/getEventMetrics.ts) | **website_event** | 第 1+2 列：`toStartOfHour(created_at), website_id` |

SQL 核心：
```sql
SELECT event_name x, count(*) y FROM website_event
WHERE website_id=? AND created_at BETWEEN ? AND ? AND event_type=2 AND ...
GROUP BY x ORDER BY y DESC LIMIT 500
```

### 3.4 Activity Tab（事件明细分页 + hasData 标志）

| 层级 | 模块 | 文件路径 | CH 主表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<EventsDataTable>` → `<EventsTable>` | [src/app/(main)/websites/[websiteId]/events/EventsDataTable.tsx](src/app/(main)/websites/%5BwebsiteId%5D/events/EventsDataTable.tsx) | - | - |
| 前端 Hook | `useWebsiteEventsQuery` (view=events → eventType=2) | [src/components/hooks/queries/useWebsiteEventsQuery.ts](src/components/hooks/queries/useWebsiteEventsQuery.ts) | - | - |
| API 路由 | GET `/websites/:id/events` | [src/app/api/websites/[websiteId]/events/route.ts](src/app/api/websites/%5BwebsiteId%5D/events/route.ts) | - | - |
| 查询函数 | `getWebsiteEvents` | [src/queries/sql/events/getWebsiteEvents.ts](src/queries/sql/events/getWebsiteEvents.ts) | **website_event**（主查）<br>+ **event_data**（hasData 子查询） | we：第 1+2 列命中<br>ed：**仅第 1 列** `website_id`（见下方详细分析） |

hasData 子查询结构（CH 方言，[getWebsiteEvents.ts:103-106](src/queries/sql/events/getWebsiteEvents.ts)）：
```sql
event_id IN (
  select event_id
  from event_data
  where website_id = {websiteId:UUID}
    and created_at between {startDate} and {endDate}   -- 第 4 位排序键，非连续前缀
) as hasData
```

#### ❗ 关键分析：这不是单事件属性查询

此子查询的语义是"找出该站点在时间范围内**有哪些 event_id 拥有属性行**"——`event_id` 是子查询的 **SELECT 输出列**，不是 WHERE 过滤条件。子查询的 WHERE 中实际只有两个条件：

| WHERE 条件 | 在排序键中的位置 | 是否构成连续前缀 |
|---|---|---|
| `website_id = ?` | 第 1 列 | ✅ 命中，mark 可跳跃 |
| `created_at BETWEEN ? AND ?` | 第 4 列（被 event_id、data_key 隔开） | ❌ 不构成连续前缀，mark 无法利用 |

**实际执行路径**：
1. 按 `website_id = ?` 粗定位 → 命中排序键第 1 列，mark 跳跃到该站数据起始位置
2. 扫描该站**全部** event_data 行 → `created_at` 过滤只能在数据读取时做，无法通过 mark 提前裁剪
3. 返回满足时间范围的所有 `event_id` 构成集合
4. 外层 `event_id IN (...)` 做匹配

**为什么 `event_id` 不是过滤条件**：子查询需要找出所有有属性行的事件 ID，它是结果集而非过滤输入。只有当子查询写成 `WHERE website_id = ? AND event_id = ?`（查特定事件的属性）时，第 1+2 列才能连续命中——但这不是 hasData 的语义。

### 3.5 Properties Tab：事件×属性组合列表

| 层级 | 模块 | 文件路径 | CH 主表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<EventProperties>` 首次挂载 | [src/app/(main)/websites/[websiteId]/events/EventProperties.tsx](src/app/(main)/websites/%5BwebsiteId%5D/events/EventProperties.tsx) | - | - |
| 前端 Hook | `useEventDataPropertiesQuery` | [src/components/hooks/queries/useEventDataPropertiesQuery.ts](src/components/hooks/queries/useEventDataPropertiesQuery.ts) | - | - |
| API 路由 | GET `/websites/:id/event-data/properties` | [src/app/api/websites/[websiteId]/event-data/properties/route.ts](src/app/api/websites/%5BwebsiteId%5D/event-data/properties/route.ts) | - | - |
| 查询函数 | `getEventDataProperties` | [src/queries/sql/events/getEventDataProperties.ts](src/queries/sql/events/getEventDataProperties.ts) | **event_data**（主查）<br>ANY LEFT JOIN **website_event**（type=2 子查询） | ed：**仅第 1 列** `website_id` 命中（created_at 在第 4 位被 event_id、data_key 隔开，不能直接走前缀；event_name 来自 event_data 宽表冗余列）<br>we：第 1+2 列命中 |

SQL 结构（CH 方言）：
```sql
SELECT event_name, data_key propertyName, count(*) total
FROM event_data
ANY LEFT JOIN (SELECT * FROM website_event
                WHERE website_id=? AND created_at BETWEEN ? AND ? AND event_type=2) we
  ON we.event_id = event_data.event_id AND we.session_id = event_data.session_id AND we.website_id = event_data.website_id
WHERE event_data.website_id=? AND event_data.created_at BETWEEN ? AND ?
GROUP BY event_name, data_key ORDER BY total DESC LIMIT 500
```

→ **注意**：ANY LEFT JOIN 右表的 `event_type=2` **不等于过滤左表**。它只是确保右表只包含事件记录，但不会过滤掉 event_data 中那些 event_id 属于 pageView（event_type=1）的属性行——只是那些行的 website_event 字段为 NULL。实际过滤 event_type=2 的是**event_data 自身冗余的 event_name 不为空**（pageView 无 event_name），以及外层 GROUP BY 时 event_name 为空的行被自然忽略。详见 3.9 节。

### 3.6 Properties Tab：指定属性的 Top 值分布（饼图 + 表格）

| 层级 | 模块 | 文件路径 | CH 主表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<EventValues>`（EventProperties 子组件） | [src/app/(main)/websites/[websiteId]/events/EventProperties.tsx](src/app/(main)/websites/%5BwebsiteId%5D/events/EventProperties.tsx) | - | - |
| 前端 Hook | `useEventDataValuesQuery` | [src/components/hooks/queries/useEventDataValuesQuery.ts](src/components/hooks/queries/useEventDataValuesQuery.ts) | - | - |
| API 路由 | GET `/websites/:id/event-data/values` | [src/app/api/websites/[websiteId]/event-data/values/route.ts](src/app/api/websites/%5BwebsiteId%5D/event-data/values/route.ts) | - | - |
| 查询函数 | `getEventDataValues` | [src/queries/sql/events/getEventDataValues.ts](src/queries/sql/events/getEventDataValues.ts) | **event_data**（主查）<br>ANY LEFT JOIN **website_event**（type=2 子查询） | ❗ **修正说明**：仅第 1 列 `website_id` 命中前缀。`data_key = ?` 是第 3 列，被第 2 列 event_id 隔开，**不构成连续前缀**，无法用于 mark 跳跃。实际执行路径：先按 website_id 粗定位 → 扫该 website_id 下的全量数据 → 用 data_key 和 created_at 过滤后聚合。 |

**类型归一化 + 分组 SQL**（CH 方言）：
```sql
SELECT
  multiIf(data_type=2, replaceAll(string_value, '.0000', ''),
          data_type=4, toString(date_trunc('hour', date_value)),
          string_value) AS value,
  count(*) AS total
FROM event_data
ANY LEFT JOIN (SELECT * FROM website_event
                WHERE website_id=? AND created_at BETWEEN ? AND ? AND event_type=2) website_event
  ON website_event.event_id = event_data.event_id
  AND website_event.session_id = event_data.session_id
  AND website_event.website_id = event_data.website_id
WHERE event_data.website_id=? AND event_data.created_at BETWEEN ? AND ? AND event_data.data_key = {propertyName:String}
GROUP BY value ORDER BY total DESC LIMIT 100
```

→ **ANY LEFT JOIN 的作用**：右表 `event_type=2` 只筛选自定义事件，但**不影响左表数据范围**。`getEventDataValues` 查询还有 `parseFilters` 产生的 `filterQuery`（如事件名过滤），以及 `data_key = ?` 等值条件（由 propertyName 参数显式写入），这些才是真正过滤数据的手段。详见 3.9 节。

> 💡 **性能启示**：如果某站点属性键数量极多（几千个），`data_key = ?` 的过滤只能在扫数据时做，开销与该站总属性行数成正比。若需要高频按 data_key 聚合，可以考虑加一个 `(website_id, data_key, created_at)` 的 PROJECTION 或换一张按 data_key 排序的物化表。

### 3.7 明细行属性展开（EventsTable 点 PropertiesButton）

| 层级 | 模块 | 文件路径 | CH 主表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<EventData>` (Dialog) | [src/components/metrics/EventData.tsx](src/components/metrics/EventData.tsx) | - | - |
| 前端 Hook | `useEventDataQuery` / `useEventDataByIdQuery` | [src/components/hooks/queries/useEventDataQuery.ts](src/components/hooks/queries/useEventDataQuery.ts) | - | - |
| API 路由 | GET `/websites/:id/event-data/[eventId]` | 见 event-data 目录 | - | - |
| 查询函数 | `getEventDataById` | [src/queries/sql/events/getEventDataById.ts](src/queries/sql/events/getEventDataById.ts) | **event_data** | ✅ 第 1+2 列：`website_id, event_id` 完全命中——**event_data 排序键的最佳使用场景**。同一事件的 N 条属性物理连续存储，mark 零跳跃。 |

SQL 极简（CH 方言）：
```sql
SELECT ... FROM event_data
WHERE website_id = {websiteId:UUID} AND event_id = {eventId:UUID}
```

### 3.8 其他属性聚合查询（辅助）

| 报表场景 | 前端/API | 查询函数 | 文件路径 | CH 主表 | 核心排序键命中 |
|---|---|---|---|---|---|
| 属性总览（事件数/属性数/记录数） | `/event-data/stats` → `useEventDataStatsQuery` | `getEventDataStats` | [src/queries/sql/events/getEventDataStats.ts](src/queries/sql/events/getEventDataStats.ts) | **event_data**（主查）<br>ANY LEFT JOIN **website_event**（type=2 子查询） | 仅第 1 列 `website_id` |
| 字段×类型×值预览 | `/event-data/fields` | `getEventDataFields` | [src/queries/sql/events/getEventDataFields.ts](src/queries/sql/events/getEventDataFields.ts) | **event_data**（主查）<br>ANY LEFT JOIN **website_event**（type=2 子查询） | 仅第 1 列 `website_id` |
| 事件-属性-值矩阵 | `/event-data/events` → `useEventDataEventsQuery` | `getEventDataEvents` | [src/queries/sql/events/getEventDataEvents.ts](src/queries/sql/events/getEventDataEvents.ts) | **event_data**（主查）<br>ANY LEFT JOIN **website_event**（type=2 子查询） | 仅第 1 列 `website_id` |

### 3.9 ANY LEFT JOIN 的协作机制解析

本节专门分析 ClickHouse 模式下所有属性聚合查询中 `ANY LEFT JOIN website_event` 的真实作用，厘清三个常见误区。

#### 3.9.1 误区一：右表 event_type=2 不等于过滤左表 event_data

**典型 SQL 模式**（所有 5 个属性聚合查询完全相同）：
```sql
FROM event_data
ANY LEFT JOIN (
  SELECT * FROM website_event
  WHERE website_id = ?
    AND created_at BETWEEN ? AND ?
    AND event_type = 2          -- 只取自定义事件
) website_event
  ON website_event.event_id   = event_data.event_id
  AND website_event.session_id = event_data.session_id
  AND website_event.website_id = event_data.website_id
${cohortQuery}
WHERE event_data.website_id = ?
  AND event_data.created_at BETWEEN ? AND ?
```

**关键分析**：
- 右表子查询加了 `event_type=2`，只能限制**右表**只包含自定义事件，不能过滤**左表** event_data。
- 左表 event_data 中那些 event_id 属于 pageView（event_type=1）的属性行，仍会被扫描出来，只是 JOIN 后对应的 website_event 字段全部为 NULL。
- **真正过滤掉 pageView 属性的，是 event_data 表自身的冗余列**：event_data 写入时就冗余了 `event_name` 列（只有自定义事件才有 event_name，pageView 的 event_name 为空字符串）。因此：
  - `getEventDataProperties` 的 `GROUP BY event_name` 会将 event_name 为空的行分到一组（通常被 LIMIT 500 截断掉）
  - `getEventDataValues` 的 `filterQuery` 通常包含 `event_name = ?`（从前端事件下拉框传入）
  - 所有属性聚合查询的最终结果只包含自定义事件，是**冗余列 + 过滤条件**共同作用的结果，不是 ANY LEFT JOIN 的作用。

#### 3.9.2 误区二：ANY LEFT JOIN 是为了获取事件字段

实际上 event_data 表在写入时已经冗余了 `event_name`、`session_id`、`url_path`、`website_id`、`event_id`、`created_at` 等多个 website_event 的字段（见 saveEventData.ts 的 clickhouseQuery 函数）。

**ANY LEFT JOIN 的真实目的有两个**：

1. **配合 cohort 查询**：`${cohortQuery}` 是一个 `INNER JOIN (SELECT DISTINCT session_id ...) AS cohort ON cohort.cohort_session_id = website_event.session_id`。它依赖 ANY LEFT JOIN 引入的 `website_event.session_id` 列来做留存过滤。**这才是 ANY LEFT JOIN 存在的首要原因**。

2. **配合过滤条件（filterQuery）中只存在于 website_event 的列**：parseFilters 生成的 filterQuery 不带表前缀，直接附加在 WHERE 末尾。对于那些只存在于 website_event 的列（`os`、`browser`、`device`、`country`、`region`、`city`、`language`、`url_path`、`referrer_domain`、`hostname` 等），必须通过 ANY LEFT JOIN 引入这些列才能生效。

#### 3.9.3 误区三：filterQuery 的列名冲突与解析

`getFilterQuery`（[src/lib/clickhouse.ts](src/lib/clickhouse.ts)）生成的过滤条件不带任何表前缀，列名解析规则如下：

| 列名来源 | 示例列名 | 解析行为 | 作用对象 |
|---|---|---|---|
| 两表共有 | `event_name`、`session_id`、`website_id`、`event_id`、`created_at` | ClickHouse 按列名匹配（两表同值，无歧义） | 可作用于任一表，实际按查询计划优化 |
| 仅 website_event | `os`、`browser`、`country`、`url_path`、`referrer_domain` | 必须通过 ANY LEFT JOIN 引入 | 作用于 JOIN 后的 website_event 列 |
| 仅 event_data | `data_key`、`data_type`、`string_value`、`number_value`、`date_value` | 直接作用于 event_data（若 JOIN 未引入则语法错误） | 作用于 event_data |

**特殊映射**：`getEventDataProperties` 通过 `parseFilters(..., { columns: { propertyName: 'data_key' } })` 手动将前端参数 `propertyName` 映射为 `data_key` 列，这是因为 `FILTER_COLUMNS` 字典中没有 `propertyName` → `data_key` 的内建映射（见 [src/lib/params.ts](src/lib/params.ts) 的 `filtersObjectToArray` 函数，优先使用 `options.columns` 映射）。

#### 3.9.4 各属性聚合查询中 ANY LEFT JOIN 的实际贡献

| 查询函数 | ANY LEFT JOIN 是否必要 | 真实贡献 |
|---|---|---|
| `getEventDataProperties` | ✅ 是 | 1) cohort 留存过滤；2) 设备/地域/URL 等 website_event 独有列的过滤；3) 防止孤立 event_data 行（理论上不存在） |
| `getEventDataValues` | ✅ 是 | 1) cohort 留存过滤；2) 设备/地域/URL 等过滤；3) filterQuery 中 `event_name` 列若从 website_event 取也可，但 event_data 已有冗余 |
| `getEventDataStats` | ✅ 是 | 1) cohort 留存过滤；2) 设备/地域/URL 等过滤 |
| `getEventDataFields` | ✅ 是 | 1) cohort 留存过滤；2) 设备/地域/URL 等过滤 |
| `getEventDataEvents` | ✅ 是 | 1) cohort 留存过滤；2) 设备/地域/URL 等过滤 |
| `getEventData`（明细分页） | ✅ 是 | 1) cohort 留存过滤；2) 设备/地域/URL 等过滤；3) 获取事件详情字段 |
| `getEventDataById`（单事件详情） | ❌ 否 | **此查询没有 ANY LEFT JOIN**！直接查 event_data，因为 event_data 已冗余 event_name 等所有需要的字段 |

> 💡 架构设计启示：`event_data` 宽表冗余是 ClickHouse 模式下的核心性能优化，将高频使用的 `event_name`、`session_id`、`url_path` 冗余到 event_data 中，使得大多数查询可以避免 JOIN。ANY LEFT JOIN 的存在只是为了兼容**两类不常见的过滤需求**：cohort 留存分析、以及基于 website_event 独有列（如设备、地域）的过滤。若无需这两类过滤，理论上可以完全去掉 ANY LEFT JOIN。

---

## 四、parseFilters：统一过滤语义在 ClickHouse 中的列映射

核心实现位于 [src/lib/clickhouse.ts](src/lib/clickhouse.ts) 和 [src/lib/params.ts](src/lib/params.ts)。

### 4.1 FILTER_COLUMNS 字典（过滤名 → DB 列名）

定义于 [src/lib/constants.ts](src/lib/constants.ts)：

```typescript
FILTER_COLUMNS = {
  path:'url_path', entry:'url_path', exit:'url_path',
  referrer:'referrer_domain', domain:'referrer_domain', hostname:'hostname',
  distinctId:'distinct_id', title:'page_title', query:'url_query',
  os, browser, device, country, region, city, language,
  event:'event_name',          // ← 事件名过滤
  tag, eventType:'event_type',
  utmSource, utmMedium, utmCampaign, utmContent, utmTerm
}
```

→ **注意**：`propertyName` 在 `getEventDataProperties` 中通过 `parseFilters` 的 `columns` 选项手动映射为 `'data_key'`（而非通过 FILTER_COLUMNS 字典）。`filtersObjectToArray` 函数的解析优先级为：`options.columns?.[name]` → `FILTER_COLUMNS[name]`。

### 4.2 过滤条件的列名解析与表归属

`getFilterQuery` 生成的过滤条件**不带任何表前缀**，直接附加在 SQL WHERE 末尾。结合 ANY LEFT JOIN 的结构，列名解析规则如下（详见 3.9.3 节）：

| 列名分类 | 示例 | 是否需要 ANY LEFT JOIN | 作用表 |
|---|---|---|---|
| 两表共有列 | `event_name`、`session_id`、`website_id`、`event_id` | 不需要 | event_data（冗余列）或 website_event |
| website_event 独有列 | `os`、`browser`、`country`、`url_path`、`referrer_domain` | **必须** | website_event（通过 ANY LEFT JOIN 引入） |
| event_data 独有列 | `data_key`、`data_type`、`string_value`、`number_value`、`date_value` | 不需要 | event_data |

**特殊映射示例**（getEventDataProperties.ts）：
```typescript
parseFilters({ ...filters, websiteId }, { columns: { propertyName: 'data_key' } })
```
→ 手动将前端参数名 `propertyName` 映射为 event_data 表的 `data_key` 列。

### 4.3 过滤操作符 → ClickHouse 函数映射

`mapFilter()` 函数定义于 [src/lib/clickhouse.ts](src/lib/clickhouse.ts)：

| 操作符 | SQL 生成 |
|---|---|
| `eq` (equals) | `col IN {name:Array(T)}` |
| `neq` | `col NOT IN {name:Array(T)}` |
| `c` (contains) | `positionCaseInsensitive(col, value) > 0` |
| `dnc` | `positionCaseInsensitive(col, value) = 0` |
| `re` (regex) | `match(col, concat('(?i)', value))` |
| `nre` | `not match(...)` |

### 4.3 Cohort 查询的 JOIN 路径

`getCohortQuery()` 会额外：
1. 从 `website_event` 中查出满足 cohort 条件的 `session_id` 集合
2. `INNER JOIN website_event.session_id = cohort_session_id` 实现留存裁剪
→ 所有 event_data 查询若带 cohort 条件，会自动继承此 JOIN（因为 JOIN 的是 website_event 子查询，event_data 通过 any left join 关联到 website_event 后再 cohort）。

---

## 五、基数控制汇总

| 控制手段 | 实施位置 | 具体值/阈值 |
|---|---|---|
| 长度截断 | saveEvent.ts, tracker schema zod | `name<=50 / data_key<=500 / string_value<=500` |
| 类型归一化 | getStringValue + getEventDataValues CASE | number `.0000`, date `hour` 粒度 |
| 聚合 LIMIT | getEventData* 查询内置 | properties=500 / values=100 / fields=100 / events=500 |
| 事件数 LIMIT | EventsChart limit | 50 |
| 分页 pageSize | DEFAULT_PAGE_SIZE 常量 | 50 |
| Bot 过滤 | /api/send 路由 `isbot(userAgent)` | 不计入任何表 |
| IP/UA 屏蔽 | `hasBlockedIp()` + schema zod | 不写入 |

---

## 六、报表组件协作全图（Events 页面）

页面入口：[src/app/(main)/websites/[websiteId]/events/EventsPage.tsx](src/app/(main)/websites/%5BwebsiteId%5D/events/EventsPage.tsx)

```
EventsPage
├─ 顶部 MetricsBar → 4 张 MetricCard
│   └─ useEventStatsQuery → /events/stats → getWebsiteEventStats → website_event
│
└─ Tabs (chart / activity / properties)
   │
   ├─ chart
   │   ├─ <EventsChart> → getEventStats → website_event_stats_hourly (快) / website_event (慢)
   │   └─ <MetricsTable type=event> → useWebsiteMetricsQuery(type=event)
   │           └─ /websites/:id/metrics → getEventMetrics → website_event (GROUP BY event_name)
   │
   ├─ activity
   │   └─ <EventsDataTable>
   │        ├─ FilterButtons (all/views/events → view param → eventType filter)
   │        ├─ useWebsiteEventsQuery → /websites/:id/events → getWebsiteEvents
   │        │     └─ website_event + event_data(hasData 子查询)
   │        └─ <EventsTable>
   │              └─ 每行 hasData ? <PropertiesButton> → <EventData Dialog>
   │                    └─ useEventDataQuery → /event-data/:eventId → website_id + event_id 命中 ed 排序键前 2 列
   │
   └─ properties
        └─ <EventProperties>
             ├─ Step 1: useEventDataPropertiesQuery → /event-data/properties
             │     └─ getEventDataProperties → event_data (主查) + ANY LEFT JOIN website_event (仅 cohort/设备/地域过滤用，event_name 取自 event_data 冗余列)
             ├─ Step 2: 填充事件下拉 + 属性下拉（级联过滤）
             └─ Step 3: 选中后渲染 <EventValues>
                   ├─ useEventDataValuesQuery(event, propertyName) → /event-data/values
                   │     └─ getEventDataValues → event_data (主查) + ANY LEFT JOIN website_event（注意：右表 event_type=2 不用于过滤左表；data_key 不构成连续前缀，需扫全量后过滤）
                   ├─ <ListTable> → 每行 value / count / percent（用 total 算比例）
                   └─ <PieChart type=doughnut> → 占比可视化
```

---

## 七、关键文件速查表

所有路径均为仓库根目录相对路径。

| 职责 | 相对路径 |
|---|---|
| CH 全部表 DDL / 索引 / MV | `db/clickhouse/schema.sql` |
| Prisma 模型（PG 版本） | `prisma/schema.prisma` |
| DATA_TYPE / FILTER_COLUMNS / EVENT_TYPE | `src/lib/constants.ts` |
| JSON 扁平化 + 类型识别 | `src/lib/data.ts` |
| CH 客户端 + parseFilters + getFilterQuery | `src/lib/clickhouse.ts` |
| 过滤器值解析 + filtersObjectToArray | `src/lib/params.ts` |
| Tracker SDK | `src/tracker/index.js` |
| 采集入口 API | `src/app/api/send/route.ts` |
| 写 website_event | `src/queries/sql/events/saveEvent.ts` |
| 写 event_data（含 CH 宽表冗余） | `src/queries/sql/events/saveEventData.ts` |
| **属性相关查询（共 8 个）** | `src/queries/sql/events/getEventData*.ts` |
| 指标卡查询（events stats） | `src/queries/sql/events/getWebsiteEventStats.ts` |
| 事件图表 series | `src/queries/sql/events/getEventStats.ts` |
| 事件名 Top N（MetricsTable） | `src/queries/sql/events/getEventMetrics.ts` |
| Activity Tab 明细（含 hasData 子查询） | `src/queries/sql/events/getWebsiteEvents.ts` |
| **属性 API 路由（共 8 个）** | `src/app/api/websites/[websiteId]/event-data/**/route.ts` |
| Events 页面容器 | `src/app/(main)/websites/[websiteId]/events/EventsPage.tsx` |
| 属性探索组件 | `src/app/(main)/websites/[websiteId]/events/EventProperties.tsx` |
| 事件明细属性弹窗 | `src/components/metrics/EventData.tsx` |
| **前端 Query Hooks** | `src/components/hooks/queries/useEventData*.ts` |

---

## 八、ClickHouse 查询性能要点总结

1. **website_event 表**的排序键前两列 `(toStartOfHour(created_at), website_id)` 被绝大多数事件级查询充分利用，mark 跳跃高效。

2. **event_data 表**的排序键 `(website_id, event_id, data_key, created_at)` 只有两种查询形态能高效命中：
   - `website_id = ? AND event_id = ?`（单事件查属性，如 `getEventDataById`）——前两列完全命中，**最佳性能**。
   - `website_id = ?`（全站属性聚合）——仅第 1 列命中，其余需扫数据。

3. ❗ **常见性能误区修正**：
   - `WHERE website_id = ? AND data_key = ?` **不构成连续前缀**，因为 event_id 夹在中间。data_key 过滤只能在扫数据时做，无法通过 mark 跳跃提前裁剪。
   - `WHERE website_id = ? AND created_at BETWEEN ?` 同样**不构成连续前缀**，created_at 在第 4 位被 event_id 和 data_key 隔开。hasData 子查询（Activity Tab）正是这种模式——只能按 website_id 粗定位后全量扫描再过滤时间。
   - 若某站点属性键极多且频繁按 data_key 查询，可考虑新增 `(website_id, data_key, created_at)` 排序的 PROJECTION 或物化表。

4. **hasData 子查询（Activity Tab）不是单事件属性查询**：子查询 `SELECT event_id FROM event_data WHERE website_id=? AND created_at BETWEEN ?` 的 `event_id` 是**输出列**而非过滤条件，它的语义是"找出该站有哪些事件拥有属性行"。因此排序键只有第 1 列 `website_id` 命中，需扫全站属性行后按 created_at 过滤。

5. **ANY LEFT JOIN 的真实作用不是过滤 event_type=2**：
   - 右表 `event_type=2` 只能限制右表内容，不能过滤左表 event_data。真正过滤 pageView 的是 event_data 自身冗余的 `event_name` 列（pageView 的 event_name 为空）。
   - ANY LEFT JOIN 存在的**两个真实原因**：① 配合 cohort 留存分析（INNER JOIN cohort ON website_event.session_id）；② 配合 filterQuery 中只存在于 website_event 的列（设备、地域、URL 等）。
   - `getEventDataById` 查询**没有 ANY LEFT JOIN**，因为只需要 event_data 表中的冗余字段即可。
   - 若无需 cohort 留存分析和设备/地域过滤，理论上可以移除所有 ANY LEFT JOIN。

6. **无过滤事件图表**走 `website_event_stats_hourly` 物化表，比扫明细快约 1~2 个数量级；但一旦带 filter / cohort，回源 `website_event` 明细。

7. **FILTER_COLUMNS 中没有 data_key / data_value 级的内建列** → 属性值级的自定义过滤只能通过 parseFilters 的 `columns` 选项手动映射（见 getEventDataProperties 的做法）。

8. event_data 表**不分区**，所有查询都必须带 `created_at BETWEEN` 时间范围来减少扫描量；但由于 created_at 在排序键末位，时间过滤对 mark 跳跃帮助有限，主要靠**每 mark 内的二级裁剪**。
