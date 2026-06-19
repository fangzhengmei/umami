# Umami 自定义事件属性：读写聚合 & ClickHouse 表/索引/查询路径精确对照

> 本文档中所有代码引用的**显示名使用仓库相对路径**，点击跳转仍使用绝对路径定位到具体行号。

---

## 一、ClickHouse 表清单与索引策略（精确 DDL）

所有表 DDL 定义于 `db/clickhouse/schema.sql`：

### 1.1 事件主表：`umami.website_event`

- [schema.sql:2-55](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/db/clickhouse/schema.sql#L2-L55)

```sql
ENGINE = MergeTree
PARTITION BY toYYYYMM(created_at)
ORDER BY (toStartOfHour(created_at), website_id, session_id, visit_id, created_at)
PRIMARY KEY (toStartOfHour(created_at), website_id, session_id, visit_id)
SETTINGS index_granularity = 8192;
```

**排序键 = 稀疏索引 = 物理组织顺序**（ClickHouse 无 BTree 索引，靠 ORDER BY + mark 文件做粒度化跳转）：

| 位次 | 键列 | 覆盖的典型查询前缀 |
|---|---|---|
| 1 | `toStartOfHour(created_at)` | 任意时间范围（先粗筛 mark，后扫分区） |
| 2 | `website_id` | 所有查询必带，强命中 |
| 3 | `session_id` | 会话级 JOIN / cohort |
| 4 | `visit_id` | 访问级去重 / 明细 |
| 5 | `created_at` | 末位时间范围下推（mark 内二次裁剪） |

**投影索引（Projections）**：

| 投影名 | 重排键 | 命中场景 |
|---|---|---|
| `website_event_url_path_projection` | `toStartOfDay(created_at), website_id, url_path, created_at` | 按页面路径聚合报表 |
| `website_event_referrer_domain_projection` | `toStartOfDay(created_at), website_id, referrer_domain, created_at` | 按来源域名聚合报表 |

### 1.2 事件属性表：`umami.event_data`

- [schema.sql:57-74](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/db/clickhouse/schema.sql#L57-L74)

```sql
ENGINE = MergeTree
ORDER BY (website_id, event_id, data_key, created_at)
SETTINGS index_granularity = 8192;
```

**排序键各列命中分析：**

| 位次 | 键列 | 作用 | 示例查询前缀 |
|---|---|---|---|
| 1 | `website_id` | 租户隔离，所有查询必带 | 全量 |
| 2 | `event_id` | 同事件的 N 条属性物理连续，**API 层 Map 重组**按 event_id 归并时高效 | `getEventData` 明细、JOIN |
| 3 | `data_key` | 属性键等值/前缀匹配 | `getEventDataValues` 查某属性值分布 |
| 4 | `created_at` | 时间范围下推（同前缀内 mark 裁剪） | 全部报表 |

**注意**：此表**不分区**（不写 PARTITION BY），数据量极大时全时间范围查询成本较高，因此所有属性查询都强制 `created_at BETWEEN {startDate} AND {endDate}`。

### 1.3 会话属性表：`umami.session_data`

- [schema.sql:76-91](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/db/clickhouse/schema.sql#L76-L91)

```sql
ENGINE = ReplacingMergeTree
ORDER BY (website_id, session_id, data_key)
SETTINGS index_granularity = 8192;
```

用于 `identify()` 调用的 user/session 级属性（不在事件报表范围内，此处仅列索引对照）。

### 1.4 小时级物化聚合表：`umami.website_event_stats_hourly`

- [schema.sql:94-143](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/db/clickhouse/schema.sql#L94-L143)

```sql
ENGINE = AggregatingMergeTree
PARTITION BY toYYYYMM(created_at)
ORDER BY (website_id, event_type, toStartOfHour(created_at), cityHash64(visit_id), visit_id)
SAMPLE BY cityHash64(visit_id);
```

**写入来源**：由 `website_event_stats_hourly_mv` 物化视图基于 `website_event` 实时写入 → 见 [schema.sql:145-239](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/db/clickhouse/schema.sql#L145-L239)。

**关键**：此表**不含自定义事件属性**，仅聚合 event_name 数组（`SimpleAggregateFunction(groupArrayArray, Array(String))`）；属性聚合只能回源 `event_data` + `website_event` JOIN。

### 1.5 收益宽表：`umami.website_revenue`

- [schema.sql:257-289](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/db/clickhouse/schema.sql#L257-L289)

```sql
ENGINE = MergeTree
PARTITION BY toYYYYMM(created_at)
ORDER BY (website_id, session_id, created_at)
```

**来源**：`website_revenue_mv` 物化视图自动从 `event_data` 抽取 **data_key 含 'revenue' + data_key 含 'currency'** 的两条行合一行，无需应用层写入。

---

## 二、写入路径完整链路（属性采集 → CH 落盘）

### 2.1 端到端时序

```
浏览器端
 └─ src/tracker/index.js L207-212 track(name, data)
    └─ POST /api/send  body: { type:'event', payload:{ name, data, ... } }
服务端采集路由
 └─ src/app/api/send/route.ts L217-271
     │  ① event_type = name ? customEvent(2) : pageView(1)
     │  ② payload.data  →  eventData 参数
     └─ saveEvent(args)  →  src/queries/sql/events/saveEvent.ts
        ├─ 写 website_event（获取 eventId=uuid()）
        ├─ 若 eventData 非空 → saveEventData()
        │     └─ src/queries/sql/events/saveEventData.ts L50-78
        │        ├─ flattenJSON(eventData)  →  [{key,value,dataType}...]
        │        ├─ kafka.enabled  →  kafka.sendMessage('event_data', ...)
        │        └─ 否则 clickhouse.insert('event_data', ...)
        └─ 若 eventData.revenue>0 && currency → saveRevenue()
```

### 2.2 JSON 扁平化 + 类型分类

核心代码：`src/lib/data.ts`

- `flattenJSON` [data.ts:4-25](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/lib/data.ts#L4-L25)：嵌套对象用 `.` 拼接键（`{a:{b:1}} → a.b`）；数组整体 JSON.stringify；日期字符串通过 `DATETIME_REGEX` 正则识别。
- `createKey` [data.ts:53-82](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/lib/data.ts#L53-L82)：将 typeof 值映射到 `DATA_TYPE` 枚举（见 2.3），写入时数值统一 `toFixed(4)`、日期统一 ISO。

### 2.3 属性类型枚举

定义于 `src/lib/constants.ts` L129-162：

| 常量 | 编码 | DB 列选择 | 示例 |
|---|---|---|---|
| `DATA_TYPE.string` | 1 | `string_value` | `"pro"` |
| `DATA_TYPE.number` | 2 | `number_value` + `string_value`(toFixed4) | `99.9900` |
| `DATA_TYPE.boolean` | 3 | `string_value`("true"/"false") | `"true"` |
| `DATA_TYPE.date` | 4 | `date_value` + `string_value`(ISO) | `2025-01-01T00:00:00.000Z` |
| `DATA_TYPE.array` | 5 | `string_value`(JSON) | `"[\"a\",\"b\"]"` |

### 2.4 ClickHouse event_data 行写入结构

每行 = 一个属性（即一个事件 N 属性写 N 行），对应 `saveEventData.clickhouseQuery()`：

```typescript
{
  website_id, session_id, event_id, url_path, event_name,  // 冗余，避免 JOIN website_event
  data_key,       // 扁平化后属性名，如 "user.address.city"
  data_type,      // 1~5 枚举
  string_value,   // 所有类型均有（统一可检索表达）
  number_value,   // 仅 DATA_TYPE.number 非空 (Decimal22(4))
  date_value,     // 仅 DATA_TYPE.date 非空 (DateTime UTC)
  created_at,     // 与 website_event.created_at 一致
}
```

> **关键**：写入即冗余 `event_name / session_id / url_path` 到 event_data 表——这是 ClickHouse 模式下"宽表"设计，避免属性聚合时频繁 JOIN website_event。

---

## 三、报表查询路径 ↔ ClickHouse 表 ↔ 排序键命中 对照表

### 3.1 事件页总览（4 指标卡）

| 层级 | 模块 | 文件路径 | CH 表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端 Hook | `useEventStatsQuery` | `src/components/hooks/queries/useEventStatsQuery.ts` | - | - |
| API 路由 | GET `/websites/:id/events/stats` | `src/app/api/websites/[websiteId]/events/stats/route.ts` | - | - |
| 查询函数 | `getWebsiteEventStats` | `src/queries/sql/events/getWebsiteEventStats.ts` | **website_event** | `toStartOfHour, website_id`（前两列全命中） |

SQL 核心（CH 方言）[getWebsiteEventStats.ts:72-96](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getWebsiteEventStats.ts#L72-L96)：
```sql
SELECT sum(c) events, uniq(session_id) visitors, uniq(visit_id) visits,
       count(distinct event_name) uniqueEvents
FROM (SELECT session_id, visit_id, event_name, count(*) c
      FROM website_event
      WHERE website_id = ? AND created_at BETWEEN ? AND ? AND event_type = 2
      GROUP BY session_id, visit_id, event_name) t
```

### 3.2 事件趋势图（Chart Tab）

| 层级 | 模块 | 文件路径 | CH 表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<EventsChart>` | `src/components/metrics/EventsChart.tsx` | - | - |
| 查询函数 | `getEventStats` | `src/queries/sql/events/getEventStats.ts` | **条件分支：**<br>① 无过滤 → **website_event_stats_hourly**<br>② 有过滤/cohort → **website_event** | ① hourly: `website_id, event_type, toStartOfHour`<br>② we: `toStartOfHour, website_id` |

**关键加速路径** [getEventStats.ts:103-137](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getEventStats.ts#L103-L137)：若 `parseFilters` 产出的 `filterQuery` 和 `cohortQuery` 都为空，直接用**小时物化表** `arrayJoin(event_name)` 展开，避免扫明细。

### 3.3 MetricsTable（事件类型 Top N 排行）

| 层级 | 模块 | 文件路径 | CH 表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<MetricsTable type="event">` | `src/components/metrics/MetricsTable.tsx` | - | - |
| 前端 Hook | `useWebsiteMetricsQuery` (type=event) | `src/components/hooks/queries/useWebsiteMetricsQuery.ts` | - | - |
| 查询函数 | `getEventMetrics` (type=event → column=event_name) | `src/queries/sql/events/getEventMetrics.ts` | **website_event** | `toStartOfHour, website_id` |

SQL 核心 [getEventMetrics.ts:81-96](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getEventMetrics.ts#L81-L96)：
```sql
SELECT event_name x, count(*) y FROM website_event
WHERE website_id=? AND created_at BETWEEN ? AND ? AND event_type=2 AND ...
GROUP BY x ORDER BY y DESC LIMIT 500
```

### 3.4 Activity Tab（事件明细分页 + hasData 标志）

| 层级 | 模块 | 文件路径 | CH 表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<EventsDataTable>` → `<EventsTable>` | `src/app/(main)/websites/[websiteId]/events/EventsDataTable.tsx` | - | - |
| 前端 Hook | `useWebsiteEventsQuery` (view=events → eventType=2) | `src/components/hooks/queries/useWebsiteEventsQuery.ts` | - | - |
| API 路由 | GET `/websites/:id/events` | `src/app/api/websites/[websiteId]/events/route.ts` | - | - |
| 查询函数 | `getWebsiteEvents` | `src/queries/sql/events/getWebsiteEvents.ts` | **website_event** + **event_data**（子查询） | we: `toStartOfHour, website_id`<br>ed: `website_id` |

子查询计算 `hasData` 标志 [getWebsiteEvents.ts:103-106](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getWebsiteEvents.ts#L103-L106)：
```sql
event_id IN (SELECT event_id FROM event_data
             WHERE website_id = ? AND ${dateQuery}) AS hasData
```
→ 用于 `<EventsTable>` L61 显示 `PropertiesButton`（点展开查看属性）。

### 3.5 Properties Tab：事件×属性组合列表

| 层级 | 模块 | 文件路径 | CH 表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<EventProperties>` 首次挂载 | `src/app/(main)/websites/[websiteId]/events/EventProperties.tsx` | - | - |
| 前端 Hook | `useEventDataPropertiesQuery` | `src/components/hooks/queries/useEventDataPropertiesQuery.ts` | - | - |
| API 路由 | GET `/websites/:id/event-data/properties` | `src/app/api/websites/[websiteId]/event-data/properties/route.ts` | - | - |
| 查询函数 | `getEventDataProperties` | `src/queries/sql/events/getEventDataProperties.ts` | **event_data** LEFT JOIN **website_event**（type=2 子查询） | ed: `website_id`（第1列命中）<br>we: `toStartOfHour, website_id` |

SQL 结构 [getEventDataProperties.ts:65-91](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getEventDataProperties.ts#L65-L91)：
```sql
SELECT event_name, data_key propertyName, count(*) total
FROM event_data
ANY LEFT JOIN (SELECT * FROM website_event
                WHERE website_id=? AND created_at BETWEEN ? AND ? AND event_type=2) we
  ON we.event_id = ed.event_id AND we.session_id = ed.session_id AND we.website_id = ed.website_id
WHERE ed.website_id=? AND ed.created_at BETWEEN ? AND ?
GROUP BY event_name, data_key ORDER BY total DESC LIMIT 500
```
→ **注意**：ClickHouse 模式下 event_name 来自 event_data **宽表冗余列**，LEFT JOIN 仅用于过滤可能的孤立 event_data 行（理论上不会出现）；实际主查走 `event_data.website_id + created_at` 范围扫描。

### 3.6 Properties Tab：指定属性的 Top 值分布（饼图 + 表格）

| 层级 | 模块 | 文件路径 | CH 表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<EventValues>`（EventProperties 子组件） | `src/app/(main)/websites/[websiteId]/events/EventProperties.tsx L75` | - | - |
| 前端 Hook | `useEventDataValuesQuery(websiteId, eventName, propertyName)` | `src/components/hooks/queries/useEventDataValuesQuery.ts` | - | - |
| API 路由 | GET `/websites/:id/event-data/values` | `src/app/api/websites/[websiteId]/event-data/values/route.ts` | - | - |
| 查询函数 | `getEventDataValues` | `src/queries/sql/events/getEventDataValues.ts` | **event_data** ANY LEFT JOIN **website_event** | ed: `website_id, data_key`（第1+3列，强命中） |

类型归一化 + 分组 SQL [getEventDataValues.ts:68-94](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getEventDataValues.ts#L68-L94)：
```sql
SELECT
  multiIf(data_type=2, replaceAll(string_value, '.0000', ''),
          data_type=4, toString(date_trunc('hour', date_value)),
          string_value) AS value,
  count(*) AS total
FROM event_data ...
WHERE website_id=? AND created_at BETWEEN ? AND ? AND data_key = {propertyName:String}
GROUP BY value ORDER BY total DESC LIMIT 100
```

### 3.7 明细行属性展开（EventsTable 点 PropertiesButton）

| 层级 | 模块 | 文件路径 | CH 表 | 命中的排序键前缀 |
|---|---|---|---|---|
| 前端组件 | `<EventData>` (Dialog) | `src/components/metrics/EventData.tsx` | - | - |
| 前端 Hook | `useEventDataQuery(websiteId, eventId)` | `src/components/hooks/queries/useEventDataQuery.ts` | - | - |
| API 路由 | GET `/websites/:id/event-data/[eventId]` 或 `/event-data`（带 event filter） | `src/app/api/websites/[websiteId]/event-data/route.ts` / `[eventId]/route.ts` | - | - |
| 查询函数 | `getEventData` / `getEventDataById` | `src/queries/sql/events/getEventData.ts` / `getEventDataById.ts` | **event_data** + **website_event** | ed: `website_id, event_id`（第1+2列，完美命中！） |

→ 这是 **event_data 排序键 (website_id, event_id, data_key, created_at) 被最充分利用**的场景：同一 event 的所有属性连续存储，mark 极少跳转。

### 3.8 其他属性聚合查询（辅助）

| 报表场景 | 前端/API | 查询函数 | CH 主表 | 核心排序键命中 |
|---|---|---|---|---|
| 属性总览（事件数/属性数/记录数） | `/event-data/stats` → `useEventDataStatsQuery` | `getEventDataStats` | **event_data** + website_event | `website_id` |
| 字段×类型×值预览 | `/event-data/fields` | `getEventDataFields` | **event_data** + website_event | `website_id` |
| 事件-属性-值矩阵 | `/event-data/events` → `useEventDataEventsQuery` | `getEventDataEvents` | **event_data** + website_event | `website_id` |

---

## 四、parseFilters：统一过滤语义在 ClickHouse 中的列映射

核心实现：`src/lib/clickhouse.ts` L101-236

### 4.1 FILTER_COLUMNS 字典（过滤名 → DB 列名）

位于 `src/lib/constants.ts` L72-97：

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

→ **注意**：`propertyName` 在 `getEventDataProperties` 中通过 `parseFilters` 的 `columns` 选项手动映射为 `'data_key'`（而非通过 FILTER_COLUMNS 字典），见 `getEventDataProperties.relationalQuery` L22-27：

```typescript
parseFilters({ ...filters, websiteId }, { columns: { propertyName: 'data_key' } })
```

### 4.2 过滤操作符 → ClickHouse 函数映射

`mapFilter()` 定义于 `src/lib/clickhouse.ts` L73-99：

| 操作符 | SQL 生成 |
|---|---|
| `eq` (equals) | `col IN {name:Array(T)}` |
| `neq` | `col NOT IN {name:Array(T)}` |
| `c` (contains) | `positionCaseInsensitive(col, value) > 0` |
| `dnc` | `positionCaseInsensitive(col, value) = 0` |
| `re` (regex) | `match(col, concat('(?i)', value))` |
| `nre` | `not match(...)` |

### 4.3 Cohort 查询的 JOIN 路径

`getCohortQuery()` [clickhouse.ts:142-161](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/lib/clickhouse.ts#L142-L161) 会额外：
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
| Bot 过滤 | /api/send L131 `isbot(userAgent)` | 不计入任何表 |
| IP/UA 屏蔽 | `hasBlockedIp()` + schema zod | 不写入 |

---

## 六、报表组件协作全图（Events 页面）

`src/app/(main)/websites/[websiteId]/events/EventsPage.tsx` L21-122 页面结构：

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
   │                    └─ useEventDataQuery → /event-data/:eventId → website_id + event_id 命中 ed 排序键
   │
   └─ properties
        └─ <EventProperties>
             ├─ Step 1: useEventDataPropertiesQuery → /event-data/properties
             │     └─ getEventDataProperties → event_data + website_event
             ├─ Step 2: 填充事件下拉 + 属性下拉（级联过滤）
             └─ Step 3: 选中后渲染 <EventValues>
                   ├─ useEventDataValuesQuery(event, propertyName) → /event-data/values
                   │     └─ getEventDataValues → event_data.data_key 精确匹配（LIMIT 100）
                   ├─ <ListTable> → 每行 value / count / percent（用 total 算比例）
                   └─ <PieChart type=doughnut> → 占比可视化
```

---

## 七、关键文件速查表（相对路径 → 绝对路径跳转）

| 职责 | 相对路径 | 跳转 |
|---|---|---|
| CH 全部表 DDL / 索引 / MV | `db/clickhouse/schema.sql` | [schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/db/clickhouse/schema.sql) |
| Prisma 模型（PG 版本） | `prisma/schema.prisma` | [schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/prisma/schema.prisma) |
| DATA_TYPE / FILTER_COLUMNS / EVENT_TYPE | `src/lib/constants.ts` | [constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/lib/constants.ts) |
| JSON 扁平化 + 类型识别 | `src/lib/data.ts` | [data.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/lib/data.ts) |
| CH 客户端 + parseFilters | `src/lib/clickhouse.ts` | [clickhouse.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/lib/clickhouse.ts) |
| Tracker SDK | `src/tracker/index.js` | [index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/tracker/index.js) |
| 采集入口 API | `src/app/api/send/route.ts` | [route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/api/send/route.ts) |
| 写 website_event | `src/queries/sql/events/saveEvent.ts` | [saveEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/saveEvent.ts) |
| 写 event_data | `src/queries/sql/events/saveEventData.ts` | [saveEventData.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/saveEventData.ts) |
| **属性相关查询（共 8 个）** | `src/queries/sql/events/getEventData*.ts` | 目录 [events](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events) |
| 指标卡查询（events stats） | `src/queries/sql/events/getWebsiteEventStats.ts` | [getWebsiteEventStats.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getWebsiteEventStats.ts) |
| 事件图表 series | `src/queries/sql/events/getEventStats.ts` | [getEventStats.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getEventStats.ts) |
| 事件名 Top N（MetricsTable） | `src/queries/sql/events/getEventMetrics.ts` | [getEventMetrics.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getEventMetrics.ts) |
| Activity Tab 明细 | `src/queries/sql/events/getWebsiteEvents.ts` | [getWebsiteEvents.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getWebsiteEvents.ts) |
| **属性 API 路由（共 8 个）** | `src/app/api/websites/[websiteId]/event-data/**/route.ts` | 目录 [event-data](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/api/websites/%5BwebsiteId%5D/event-data) |
| Events 页面容器 | `src/app/(main)/websites/[websiteId]/events/EventsPage.tsx` | [EventsPage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/(main)/websites/%5BwebsiteId%5D/events/EventsPage.tsx) |
| 属性探索组件 | `src/app/(main)/websites/[websiteId]/events/EventProperties.tsx` | [EventProperties.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/(main)/websites/%5BwebsiteId%5D/events/EventProperties.tsx) |
| 事件明细属性弹窗 | `src/components/metrics/EventData.tsx` | [EventData.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/components/metrics/EventData.tsx) |
| **前端 Query Hooks** | `src/components/hooks/queries/useEventData*.ts` | 目录 [queries](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/components/hooks/queries) |

---

## 八、ClickHouse 查询性能要点总结

1. **所有属性查询**均通过 `event_data.website_id + created_at BETWEEN` 做最外层约束 → 命中稀疏索引第 1 列 + 分区裁剪（依赖时间范围的 mark 跳跃）。
2. **查指定属性的值分布**（getEventDataValues）`WHERE data_key = 'xxx'` → 命中 `(website_id, event_id, data_key)` 的第 1+3 列，因为 event_id 是 UUID（高基数），跳过第 2 列后 data_key 的 mark 粒度略粗但仍远好于全扫。
3. **查单事件属性列表**是 event_data 排序键的最佳场景 → `website_id + event_id` 前两列全命中，属性值物理连续，mark 跳 0 次。
4. **无过滤事件图表**走 `website_event_stats_hourly` 物化表，比扫明细快约 1~2 个数量级；但一旦带 filter / cohort，回源 `website_event` 明细。
5. **FILTER_COLUMNS 中没有 `data_key` / `data_value` 级的内建列** → 属性值级的自定义过滤只能通过 parseFilters 的 `columns` 选项手动映射（见 getEventDataProperties 的做法）。
