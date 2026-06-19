# Umami 自定义事件属性：读写与聚合协作分析

## 一、总体架构概览

Umami 事件属性系统采用**双引擎存储**（关系型 PostgreSQL / ClickHouse）+ **属性行转列扁平化存储**的设计。完整数据链路如下：

```
浏览器 Tracker (JS SDK)
    │
    ▼  POST /api/send
采集路由 (route.ts) → saveEvent() → saveEventData()
    │                                   │
    │                                   ├── Prisma (PostgreSQL) 写入 event_data 表
    │                                   └── ClickHouse (可选 Kafka 缓冲) 写入 event_data 表
    ▼
前端报表组件 ← REST API ← 聚合查询 SQL (双引擎适配)
```

---

## 二、属性类型系统

### 2.1 类型常量定义

定义于 [constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/lib/constants.ts#L129-L162)：

```typescript
export const DATA_TYPE = {
  string: 1,   // 字符串
  number: 2,   // 数字
  boolean: 3,  // 布尔
  date: 4,     // 日期
  array: 5,    // 数组/对象
} as const;
```

### 2.2 多列存储策略

在 `EventData` 模型中，针对不同类型使用独立列存储（EAV 模式的变体）：

| 模型字段 | 数据库列 | 类型 | 说明 |
|---|---|---|---|
| `dataKey` | `data_key` | `VarChar(500)` | 属性键（支持嵌套扁平化，如 `user.profile.age`） |
| `stringValue` | `string_value` | `VarChar(500)` | 字符串类型的值 |
| `numberValue` | `number_value` | `Decimal(19,4)` / `Decimal(22,4)` | 数值类型的值 |
| `dateValue` | `date_value` | `Timestamptz(6)` / `DateTime` | 日期类型的值 |
| `dataType` | `data_type` | `Integer` / `UInt32` | 标识该属性的类型枚举 |

定义于 [schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/prisma/schema.prisma#L152-L172) 和 [schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/db/clickhouse/schema.sql#L57-L74)。

### 2.3 类型检测与值转换

核心逻辑在 [data.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/lib/data.ts#L53-L82)：

```typescript
function createKey(key, value, acc) {
  const type = getDataType(value);
  switch (type) {
    case 'number':
      dataType = DATA_TYPE.number; break;
    case 'string':
      dataType = DATA_TYPE.string; break;
    case 'boolean':
      dataType = DATA_TYPE.boolean;
      value = value ? 'true' : 'false'; break;
    case 'date':  // 通过 DATETIME_REGEX 识别 ISO 日期字符串
      dataType = DATA_TYPE.date; break;
    case 'object': // 数组（对象已被递归扁平化）
      dataType = DATA_TYPE.array;
      value = JSON.stringify(value); break;
  }
}
```

`getStringValue` 函数保证所有类型都能在 `stringValue` 中有统一可检索的字符串表达：

```typescript
export function getStringValue(value: string, dataType: number) {
  if (dataType === DATA_TYPE.number) return parseFloat(value).toFixed(4);
  if (dataType === DATA_TYPE.date) return new Date(value).toISOString();
  return value;
}
```

---

## 三、写入流程（采集 → 扁平化 → 存储）

### 3.1 客户端采集

定义于 [tracker/index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/tracker/index.js#L207-L212)：

```javascript
// 两种触发方式：
// 1. 声明式：data-umami-event + data-umami-event-* 属性
// 2. 编程式：umami.track('signup', { plan: 'pro', price: 99 })
const track = (name, data) => {
  if (typeof name === 'string') return send({ ...getPayload(), name, data });
  // ...
};
```

采集后通过 `POST /api/send` 发送，body 结构为 `{ type: 'event', payload: { name, data, ... } }`。

### 3.2 采集入口路由

[route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/api/send/route.ts#L177-L271) 处理：

1. 校验 website/link/pixel 三元互斥
2. Bot 检测与 IP 屏蔽
3. 生成 sessionId/visitId（加盐哈希）
4. 解析 URL / UTM / referrer 参数
5. 判断 eventType（pageView=1, customEvent=2, ...）
6. 若 `payload.data` 存在，作为 `eventData` 传入 `saveEvent()`

### 3.3 saveEvent → saveEventData 两级写入

[saveEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/saveEvent.ts#L65-L276)：

- 第一步：创建 `WebsiteEvent` 主记录（获取 eventId）
- 第二步：若 `eventData` 非空，调用 `saveEventData()` 批量写入属性
- 第三步：若属性含 `revenue` + `currency`，额外写入 `Revenue` 表用于收益统计

### 3.4 JSON 扁平化（核心）

[saveEventData.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/saveEventData.ts#L27-L48) 调用 `flattenJSON`：

```typescript
const jsonKeys = flattenJSON(eventData);
const flattenedData = jsonKeys.map(a => ({
  id: uuid(),
  websiteEventId: eventId,
  websiteId,
  dataKey: a.key,           // 扁平化后 key，如 "address.city"
  stringValue: getStringValue(a.value, a.dataType),
  numberValue: a.dataType === DATA_TYPE.number ? a.value : null,
  dateValue: a.dataType === DATA_TYPE.date ? new Date(a.value) : null,
  dataType: a.dataType,
  createdAt,
}));
await prisma.client.eventData.createMany({ data: flattenedData });
```

`flattenJSON` 递归逻辑见 [data.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/lib/data.ts#L4-L25)：
- 嵌套对象用点号拼接：`{ user: { age: 25 } }` → `key = "user.age"`
- 数组保留为整体：`{ tags: ["a","b"] }` → `dataType = array, value = JSON.stringify(...)`
- 日期字符串（正则匹配）从 string 升级为 date 类型

### 3.5 ClickHouse 与 Kafka 通道

同文件的 `clickhouseQuery()` 中，event_data 表额外携带 `session_id`、`url_path`、`event_name`，形成宽表减少 JOIN：

```typescript
const messages = jsonKeys.map(({ key, value, dataType }) => ({
  website_id, session_id, event_id, url_path, event_name,
  data_key: key,
  data_type: dataType,
  string_value: getStringValue(value, dataType),
  number_value: dataType === DATA_TYPE.number ? value : null,
  date_value: dataType === DATA_TYPE.date ? getUTCString(value) : null,
  created_at: getUTCString(createdAt),
}));
if (kafka.enabled) sendMessage('event_data', messages);
else insert('event_data', messages);
```

ClickHouse 还通过物化视图 `website_revenue_mv` 自动从 `event_data` 中抽取 `revenue`/`currency` 构建收益宽表（[schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/db/clickhouse/schema.sql#L273-L289)）。

---

## 四、基数控制策略

### 4.1 字段长度硬截断

写入路径中多处 `substring(0, MAX_LENGTH)`：

| 字段 | 常量 | 长度 | 位置 |
|---|---|---|---|
| `eventName` | `EVENT_NAME_LENGTH` | 50 | [saveEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/saveEvent.ts#L131) |
| `dataKey` | - | 500 (DB 列约束) | schema.prisma |
| `stringValue` | - | 500 (DB 列约束) | schema.prisma |
| `urlPath` / `referrerDomain` | `URL_LENGTH` | 500 | schema.prisma |
| `pageTitle` | `PAGE_TITLE_LENGTH` | 500 | schema.prisma |

### 4.2 查询端 LIMIT 限制（防基数爆炸）

所有聚合查询内置 LIMIT，防止前端一次性拉取过多唯一值：

| 查询函数 | LIMIT | 用途 |
|---|---|---|
| `getEventDataProperties` | 500 | 事件名 × 属性名 组合列表 |
| `getEventDataFields` | 100 | 字段 × 值 分布 |
| `getEventDataValues` | 100 | 指定属性的 Top 值分布 |
| `getEventDataEvents` | 500 | 事件 × 属性 聚合表 |
| `getEventData` | `DEFAULT_PAGE_SIZE` (默认 50) | 分页明细 |

### 4.3 值归一化

数值类型统一 `toFixed(4)`，日期类型统一 ISO 字符串或按 hour 截断聚合，降低不必要的精度导致的高基数。

见 [getEventDataValues.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getEventDataValues.ts#L35-L39)：

```sql
case
  when data_type = 2 then replace(string_value, '.0000', '')
  when data_type = 4 then date_trunc('hour', date_value)
  else string_value
end as "value"
```

---

## 五、聚合查询体系

### 5.1 查询分层与 API 路由

| 路由 | 底层函数 | 返回结构 | 作用 |
|---|---|---|---|
| `GET /event-data/stats` | `getEventDataStats` | `{events, properties, records}` | 总览指标 |
| `GET /event-data/properties` | `getEventDataProperties` | `{eventName, propertyName, total}[]` | 事件-属性组合统计 |
| `GET /event-data/fields` | `getEventDataFields` | `{propertyName, dataType, value, total}[]` | 属性值分布预览 |
| `GET /event-data/values` | `getEventDataValues` | `{value, total}[]` | 指定属性的 Top 值 |
| `GET /event-data/events` | `getEventDataEvents` | `{eventName, propertyName, dataType, propertyValue?, total}[]` | 事件-属性-值聚合 |
| `GET /event-data` | `getEventData` | `{data:[{eventId,eventName,eventProperties[]}], count, page}` | 明细分页 |
| `GET /event-data/[eventId]` | `getEventDataById` | 单事件详情 | 单条事件详情 |

### 5.2 双引擎适配模式

所有查询通过 `runQuery({ [PRISMA], [CLICKHOUSE] })` 根据部署配置自动路由，分别对应 PostgreSQL 原生 SQL 与 ClickHouse SQL：

**关键差异：**
- **JOIN 方式**：PostgreSQL 使用 `JOIN website_event ON website_event_id = event_id`；ClickHouse 使用 `any left join (子查询)` 且额外匹配 `session_id`、`website_id` 以提高 MergeTree 键命中。
- **日期函数**：PostgreSQL 用 `getDateSQL()`（`date_trunc`/`to_char`）；ClickHouse 原生 `date_trunc`、`multiIf`。
- **参数占位符**：PostgreSQL 用 `{{param::uuid}}`；ClickHouse 用 `{param:UUID}`。

### 5.3 核心聚合模式

#### ① 属性-值分布（getEventDataValues）

[getEventDataValues.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getEventDataValues.ts#L32-L57)：

```sql
SELECT
  CASE -- 类型归一化
    WHEN data_type = 2 THEN REPLACE(string_value, '.0000', '')
    WHEN data_type = 4 THEN DATE_TRUNC('hour', date_value)::TEXT
    ELSE string_value
  END AS "value",
  COUNT(*) AS "total"
FROM event_data
JOIN website_event ON ...
WHERE website_id = $1 AND created_at BETWEEN $2 AND $3
  AND data_key = {propertyName}
GROUP BY value
ORDER BY 2 DESC
LIMIT 100
```

#### ② 事件 × 属性组合（getEventDataProperties）

[getEventDataProperties.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getEventDataProperties.ts#L29-L50)：

```sql
SELECT
  website_event.event_name AS "eventName",
  event_data.data_key      AS "propertyName",
  COUNT(*)                 AS "total"
FROM event_data
JOIN website_event ON ...
GROUP BY website_event.event_name, event_data.data_key
ORDER BY 3 DESC
LIMIT 500
```

#### ③ 总览三维统计（getEventDataStats）

[getEventDataStats.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/getEventDataStats.ts#L28-L53)：

```sql
SELECT
  COUNT(DISTINCT t.website_event_id) AS "events",     -- 事件条数
  COUNT(DISTINCT t.data_key)         AS "properties", -- 去重属性数
  SUM(t.total)                       AS "records"     -- 总属性记录数
FROM (
  SELECT website_event_id, data_key, COUNT(*) AS total
  FROM event_data JOIN website_event ...
  GROUP BY website_event_id, data_key
) AS t
```

#### ④ 分页明细 + 重组（/event-data 路由）

后端先以"行模式"查询，再在 API 层按 `eventId` 重组回嵌套结构：

[event-data/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/api/websites/%5BwebsiteId%5D/event-data/route.ts#L35-L49)：

```typescript
const eventMap = new Map<string, { eventId, eventName, eventProperties }>();
for (const { eventId, eventName, ...props } of rows) {
  let entry = eventMap.get(eventId);
  if (!entry) {
    entry = { websiteId, eventId, eventName, eventProperties: [] };
    eventMap.set(eventId, entry);
  }
  entry.eventProperties.push(props);
}
return { data: [...eventMap.values()], count, page, pageSize };
```

### 5.4 过滤系统协作：parseFilters

每个查询通过 `parseFilters(filters, { columns })` 注入通用过滤条件（时间范围、事件名、会话属性、cohort、URL 等），将 `filters` 对象展开为：
- `filterQuery`：WHERE 子句（基于 propertyName/value 的操作符匹配）
- `cohortQuery`：用于留存 cohort 的 JOIN 子句
- `joinSessionQuery`：自动 JOIN session 表做设备/地域过滤
- `queryParams`：参数化绑定数组

这使得所有聚合查询都能复用同一套过滤语义。

---

## 六、索引策略

### 6.1 PostgreSQL（Prisma）

`event_data` 表的索引定义于 [schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/prisma/schema.prisma#L166-L171)：

```prisma
@@index([createdAt])
@@index([websiteId])
@@index([websiteEventId])
@@index([websiteId, createdAt])
@@index([websiteId, createdAt, dataKey])  -- 核心组合索引
```

**索引设计意图：**
1. `(websiteId, createdAt, dataKey)`：最常用查询路径（按站点 × 时间范围 × 属性键筛选），满足最左前缀：
   - 仅 websiteId → 命中
   - websiteId + createdAt 范围 → 命中
   - websiteId + createdAt + dataKey 等值 → 高度选择性
2. `(websiteEventId)`：反向从事件查属性列表、以及 JOIN 高效查找
3. `(websiteId, createdAt)`：时间范围总览统计

`website_event` 对应索引：`(websiteId, createdAt, eventName)`，支撑 JOIN 时按事件名过滤。

### 6.2 ClickHouse

[schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/db/clickhouse/schema.sql#L57-L74)：

```sql
CREATE TABLE umami.event_data
ENGINE = MergeTree
ORDER BY (website_id, event_id, data_key, created_at)
SETTINGS index_granularity = 8192;
```

**排序键设计意图：**
- `website_id` 前缀：租户隔离，查询必带
- `event_id` 第二级：同一事件的属性连续存储，重组高效
- `data_key` 第三级：按属性名范围/Rewind 快速定位
- `created_at` 末级：时间顺序排列，TTL 删除友好

**配套物化视图加速：**
1. `website_event_stats_hourly_mv`：按小时聚合 visit 级别维度，支撑主看板图表（不含自定义属性）
2. `website_revenue_mv`：实时从 event_data 抽取含 revenue/currency 的记录

**投影索引（Projections）：**
```sql
ALTER TABLE umami.website_event
ADD PROJECTION website_event_url_path_projection (
  SELECT * ORDER BY toStartOfDay(created_at), website_id, url_path, created_at
);
```
为高频维度（如 `url_path`、`referrer_domain`）构建列存投影，按不同排序键物理重排，对应报表直接扫投影。

---

## 七、报表展示协作

### 7.1 Events 页面三栏结构

[EventsPage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/(main)/websites/%5BwebsiteId%5D/events/EventsPage.tsx#L90-L118)：

```
Tabs:
  ├── "chart"    → EventsChart + MetricsTable<type=event>
  ├── "activity" → EventsDataTable (分页明细)
  └── "properties" → EventProperties (属性探索)
```

顶部 MetricsBar：调用 `useEventStatsQuery` 拉取 visitors / visits / events / uniqueEvents 四大指标卡片。

### 7.2 属性探索组件：EventProperties

[EventProperties.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/(main)/websites/%5BwebsiteId%5D/events/EventProperties.tsx#L13-L73) 采用**级联筛选**：

```
Step 1: useEventDataPropertiesQuery → 事件×属性组合列表
         ↓ 提取去重事件名填充
  [事件下拉框: select eventName]
         ↓ onChange 过滤同事件属性
Step 2: [属性下拉框: select propertyName]
         ↓ 选中后调用
Step 3: useEventDataValuesQuery(eventName, propertyName)
         │
         ├── ListTable → value / count / percent 明细
         └── PieChart  → 占比可视化
```

内部 `EventValues` 子组件使用 `useMemo`：
- `propertySum`：对 total 求和用于百分比
- `chartData`：喂给 PieChart 的 doughnut 格式
- `tableData`：含 `percent: 100 * (total / propertySum)`

### 7.3 明细视图：EventsDataTable

[EventsDataTable.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/(main)/websites/%5BwebsiteId%5D/events/EventsDataTable.tsx#L14-L46)：

- FilterButtons 切换视图：`all` / `views` / `events`
- `useWebsiteEventsQuery` 拉取分页数据
- 外层 `DataGrid` 组件提供分页、搜索、渲染
- `EventsTable` 接收 `data` 按行渲染，若行内含 eventProperties 则展开为子网格

### 7.4 MetricsTable（通用指标表）

"chart" 标签中的 `MetricsTable type=event` 按事件名统计 `count`，调用 `getEventMetrics`（底层与 getEventDataProperties 同 JOIN 路径但仅聚合到 event_name 级别）。

### 7.5 前后端数据流完整时序

```
用户切换到 Properties Tab
  │
  ▼
EventProperties 挂载 → useEventDataPropertiesQuery(websiteId)
  │  GET /event-data/properties?startAt&endAt
  ▼
  getEventDataProperties
    ├── JOIN website_event + event_data
    ├── GROUP BY event_name, data_key
    └── ORDER BY total DESC LIMIT 500
  │
  ▼ 渲染事件下拉框 + 属性下拉框
用户选中 eventName + propertyName
  │
  ▼ useEventDataValuesQuery(websiteId, eventName, propertyName)
  │  GET /event-data/values?event=&propertyName=
  ▼
  getEventDataValues
    ├── data_key = propertyName WHERE
    ├── 类型归一化 (value CASE)
    ├── GROUP BY value ORDER BY total DESC LIMIT 100
    └── 可叠加 parseFilters 的 cohort / 地域 / URL 过滤
  │
  ▼
ListTable 渲染值分布 + PieChart 可视化占比
```

---

## 八、设计亮点与权衡

| 决策 | 收益 | 代价 |
|---|---|---|
| **EAV 扁平化（行存储）** | 任意深度属性、无需改表；支持对任意 key 做 GROUP BY | 一个 N 属性事件写入 N 行；重组需按 eventId 聚合 |
| **多列值（string/number/date）** | 数值可 SUM/AVG、日期可范围查询、字符串可 LIKE，性能优于 JSON 提取 | 写入时需类型推断，错误推断需修正时成本高 |
| **点号扁平化嵌套键** | 兼容 JS 对象语义，报表端直接展示 `user.age` | 含点号的键名需转义；深嵌套长度逼近 500 字符 |
| **双引擎 runQuery** | 同一 API 兼容中小站点 PostgreSQL 与大站 ClickHouse | 每次查询需维护两份 SQL 方言实现 |
| **LIMIT 100/500 防基数爆炸** | UI 永远只展示 Top N，稳定响应时间 | 长尾值不可见，需定制查询 |
| **ClickHouse 宽表反范式** | 写入时冗余 event_name/session_id/url_path，查询避免 JOIN | 存储放大；更新成本极高（MergeTree 非更新型） |

---

## 九、关键文件速查

| 职责 | 文件 |
|---|---|
| 类型常量与枚举 | [constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/lib/constants.ts#L129-L162) |
| JSON 扁平化与类型推断 | [data.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/lib/data.ts) |
| Tracker SDK | [tracker/index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/tracker/index.js) |
| 采集 API 入口 | [/api/send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/api/send/route.ts) |
| saveEvent（事件主写入） | [saveEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/saveEvent.ts) |
| saveEventData（属性写入） | [saveEventData.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/queries/sql/events/saveEventData.ts) |
| Prisma 数据模型 | [schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/prisma/schema.prisma#L99-L172) |
| ClickHouse DDL + MV | [schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/db/clickhouse/schema.sql) |
| 属性聚合查询组 | `src/queries/sql/events/getEventData*.ts` 共 8 个文件 |
| 属性 API 路由组 | `src/app/api/websites/[websiteId]/event-data/**/route.ts` 共 8 个路由 |
| 前端属性探索组件 | [EventProperties.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/(main)/websites/%5BwebsiteId%5D/events/EventProperties.tsx) |
| 前端事件页容器 | [EventsPage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/51-umami/src/app/(main)/websites/%5BwebsiteId%5D/events/EventsPage.tsx) |
