# Umami 跨时区聚合的桶对齐机制

## 一、全局数据流概览

```
客户端采集 → 服务端落库(UTC) → 前端选时区 → API 传参(startAt/endAt/timezone) → SQL 桶对齐 → 时序填充 → 图表渲染
```

整条链路可以拆为三段：**原始事件落桶**、**显示时区切换**、**统计窗口跨日衔接**。

---

## 二、原始事件落桶（写入阶段）

### 2.1 事件采集：原始时间戳以 UTC 落库

事件采集入口为 [`route.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/app/api/send/route.ts#L140-L141)：

```ts
const createdAt = timestamp ? new Date(timestamp * 1000) : new Date();
```

- 若客户端传了 `timestamp`（秒级 UNIX），则乘 1000 转 `Date`；否则取服务端当前时间。
- **关键点**：`createdAt` 始终是一个绝对时间（JS Date 内部为 UTC 毫秒），**不携带任何时区语义**。

### 2.2 写入 ClickHouse：`DateTime('UTC')`

[`saveEvent.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/queries/sql/events/saveEvent.ts#L245) 将 `createdAt` 通过 `getUTCString()` 格式化后写入：

```ts
created_at: getUTCString(createdAt),
```

[`getUTCString()`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/clickhouse.ts#L50-L52)：

```ts
function getUTCString(date?: Date | string | number) {
  return formatInTimeZone(date || new Date(), 'UTC', 'yyyy-MM-dd HH:mm:ss');
}
```

ClickHouse 表定义 [`schema.sql`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/db/clickhouse/schema.sql#L48)：

```sql
created_at DateTime('UTC')
```

- `DateTime('UTC')` 表示该列存储的是 UTC 时间，ClickHouse 在解析时不会做隐式时区转换。
- **因此，原始事件以 UTC 秒精度落入 `website_event` 表，写入时无桶概念。**

### 2.3 预聚合物化视图：按 UTC 小时桶

[`website_event_stats_hourly_mv`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/db/clickhouse/schema.sql#L145-L239) 物化视图完成小时级预聚合：

```sql
toStartOfHour(created_at) timestamp   -- 第223行
```

- `toStartOfHour` 是 ClickHouse 内置函数，对 UTC 时间截断到整小时。
- 预聚合表的 `created_at` 同样是 `Datetime('UTC')`，桶边界对齐到 **UTC 整小时**。
- **预聚合桶是 UTC 对齐的，与用户选择的显示时区无关。**

### 2.4 查询时的桶选择策略

统计查询（如 [`getPageviewStats`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/queries/sql/pageviews/getPageviewStats.ts#L46-L101)）根据条件选择原始表或预聚合表：

```ts
if (EVENT_COLUMNS.some(item => Object.keys(filters).includes(item)) || unit === 'minute') {
  // 查 website_event 原始表（分钟级桶需要原始数据）
} else {
  // 查 website_event_stats_hourly 预聚合表（性能更优）
}
```

- **涉及 EVENT_COLUMNS 过滤或 unit=minute 时**：走原始表，可做到分钟桶。
- **其他场景**：走 `website_event_stats_hourly`，聚合 `sum(views)` 计数。

---

## 三、显示时区切换（查询阶段）

### 3.1 前端时区选择与存储

时区状态存储在 Zustand 全局 store [`app.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/store/app.ts#L14-L17)：

```ts
timezone: getItem(TIMEZONE_CONFIG) || getTimezone(),
```

- 初始化优先读取 localStorage (`umami.timezone`)；否则取浏览器时区 `Intl.DateTimeFormat().resolvedOptions().timeZone`。
- 用户通过 [`TimezoneSetting`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/app/(main)/settings/preferences/TimezoneSetting.tsx) 修改后，调用 [`saveTimezone`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useTimezone.ts#L15-L18) 写入 localStorage 并更新 store。

### 3.2 时区参数从客户端到 API 的传递

[`useDateParameters`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useDateParameters.ts) 是关键的"日期参数组装器"：

```ts
export function useDateParameters() {
  const { dateRange: { startDate, endDate, unit } } = useDateRange();
  const { timezone, localToUtc, canonicalizeTimezone } = useTimezone();

  return {
    startAt: +localToUtc(startDate),
    endAt: +localToUtc(endDate),
    startDate: localToUtc(startDate).toISOString(),
    endDate: localToUtc(endDate).toISOString(),
    unit,
    timezone: canonicalizeTimezone(timezone),
  };
}
```

**核心转换**：
1. `useDateRange()` 计算出用户所选时区下的 `startDate`/`endDate`（此时是"用户时区的本地时间"）。
2. `localToUtc()` 将"用户时区本地时间"转为 UTC 时间戳（`zonedTimeToUtc(date, timezone)`）。
3. `startAt`/`endAt` 是 UTC 毫秒时间戳，传给 API。
4. **`timezone` 字符串也一并传给 API**——这是后续 SQL 桶对齐的依据。

### 3.3 时区参数的校验与规范化

API 端通过 [`timezoneParam`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/schema.ts#L5-L10) 校验：

```ts
export const timezoneParam = z
  .string()
  .refine((value: string) => isValidTimezone(value), { message: 'Invalid timezone' })
  .transform((value: string) => normalizeTimezone(value));
```

- `isValidTimezone` 用 `Intl.DateTimeFormat` 验证时区字符串合法性。
- `normalizeTimezone` 将旧别名映射到标准名（如 `Asia/Calcutta` → `Asia/Kolkata`）。
- [`TIMEZONE_LEGACY`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/constants.ts#L698-L717) 提供完整的旧→新映射表。

### 3.4 服务端提取时区并注入 SQL

[`getRequestDateRange`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/request.ts#L63-L77)：

```ts
export function getRequestDateRange(query: Record<string, string>) {
  const { startAt, endAt, unit, timezone } = query;
  const startDate = new Date(+startAt);
  const endDate = new Date(+endAt);
  return { startDate, endDate, timezone, unit: ... };
}
```

时区字符串随 `QueryFilters` 一路传递到 SQL 生成函数。

---

## 四、SQL 层桶对齐（核心机制）

### 4.1 PostgreSQL（Prisma）的桶对齐

[`getDateSQL`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/prisma.ts#L50-L56)：

```ts
function getDateSQL(field: string, unit: string, timezone?: string): string {
  if (timezone && timezone !== 'utc') {
    return `to_char(date_trunc('${unit}', ${field} at time zone '${timezone}'), '${DATE_FORMATS[unit]}')`;
  }
  return `to_char(date_trunc('${unit}', ${field}), '${DATE_FORMATS_UTC[unit]}')`;
}
```

**PostgreSQL 桶对齐机制**：
- `created_at at time zone 'Asia/Shanghai'`：将 UTC 时间戳转为指定时区的本地时间。
- `date_trunc('day', ...)`：对本地时间截断到指定精度的整点。
- `to_char(..., format)`：格式化为字符串作为桶标签。

**示例**：`timezone = 'Asia/Shanghai'`，`unit = 'day'`
- UTC `2024-01-01 20:00:00` → 上海时间 `2024-01-02 04:00:00` → 截断为 `2024-01-02` → 该事件归入 1 月 2 日的桶。
- UTC `2024-01-01 15:30:00` → 上海时间 `2024-01-01 23:30:00` → 截断为 `2024-01-01` → 该事件归入 1 月 1 日的桶。

**UTC 模式下的格式差异**：

| 条件 | 格式模板 | 输出示例 |
|------|---------|---------|
| timezone ≠ 'utc' | `YYYY-MM-DD HH24:00:00` (DATE_FORMATS) | `2024-01-02 00:00:00` |
| timezone = 'utc' 或无 | `YYYY-MM-DD"T"HH24:00:00"Z"` (DATE_FORMATS_UTC) | `2024-01-01T20:00:00Z` |

UTC 模式在格式中嵌入了 `T` 和 `Z` 标记，便于前端区分。

### 4.2 ClickHouse 的桶对齐

[`getDateSQL`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/clickhouse.ts#L62-L67)：

```ts
function getDateSQL(field: string, unit: string, timezone?: string) {
  if (timezone) {
    return `toDateTime(date_trunc('${unit}', ${field}, '${timezone}'), '${timezone}')`;
  }
  return `toDateTime(date_trunc('${unit}', ${field}))`;
}
```

**ClickHouse 桶对齐机制**：
- `date_trunc('day', created_at, 'Asia/Shanghai')`：ClickHouse 的 `date_trunc` 支持第三参数时区名，先将 UTC 时间转到目标时区再截断。
- 外层 `toDateTime(..., 'Asia/Shanghai')`：将截断结果标记为目标时区，确保返回的时间值是正确的。

**示例**：`timezone = 'Asia/Shanghai'`，`unit = 'hour'`
- UTC `2024-01-01 20:30:00` → 上海 `2024-01-02 04:30:00` → 截断为 `2024-01-02 04:00:00`（上海时区）。

### 4.3 ClickHouse 日期格式化（用于周流量等）

[`getDateStringSQL`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/clickhouse.ts#L54-L60)：

```ts
function getDateStringSQL(data: any, unit: string = 'utc', timezone?: string) {
  if (timezone) {
    return `formatDateTime(${data}, '${CLICKHOUSE_DATE_FORMATS[unit]}', '${timezone}')`;
  }
  return `formatDateTime(${data}, '${CLICKHOUSE_DATE_FORMATS[unit]}')`;
}
```

`formatDateTime` 的第三参数是 ClickHouse 特有的时区参数，在格式化时进行时区转换。

### 4.4 周流量热力图的特殊桶

[`getWeeklyTraffic`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/queries/sql/getWeeklyTraffic.ts) 使用独立的桶逻辑：

- PostgreSQL：`getDateWeeklySQL` → `extract(dow from (${field} at time zone '${timezone}'))` 提取星期几 + 小时。
- ClickHouse：`formatDateTime(toDateTime(created_at, '${timezone}'), '%w:%H')`。

桶键为 `星期:小时`（如 `1:14` = 周一14点），最终 `formatResults` 填充为 7×24 的二维数组。

---

## 五、统计窗口跨日衔接

### 5.1 前端日期范围计算

[`parseDateRange`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/date.ts#L140-L220) 根据用户选的快捷范围（如 `1day`、`1month`）在"用户时区本地时间"上做截断：

```ts
const now = timezone ? utcToZonedTime(date, timezone) : date;
// ...
case 'day':
  return {
    startDate: num ? subDays(startOfDay(now), num) : startOfDay(now),
    endDate: endOfDay(now),
    ...
  };
```

- `utcToZonedTime(date, timezone)` 将当前 UTC 时间转为用户时区的本地时间。
- `startOfDay(now)` 在该本地时间上取当天 00:00:00。
- 这确保了 **"今天"的边界由用户时区决定**，而非 UTC。

### 5.2 日期边界转 UTC 传给 API

[`useDateParameters`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useDateParameters.ts) 中：

```ts
startAt: +localToUtc(startDate),   // 用户时区 00:00 → UTC 时间戳
endAt:   +localToUtc(endDate),      // 用户时区 23:59:59 → UTC 时间戳
```

**示例**：用户时区 `Asia/Shanghai` (UTC+8)，选择"今天"
- `startDate` = 上海 2024-01-02 00:00:00 → `localToUtc` → UTC 2024-01-01 16:00:00
- `endDate` = 上海 2024-01-02 23:59:59 → `localToUtc` → UTC 2024-01-02 15:59:59
- API 收到的 `startAt`/`endAt` 已是 UTC 范围，确保 SQL `BETWEEN` 能正确框住数据。

### 5.3 SQL 时间范围过滤与桶截断的协同

以 ClickHouse 为例，统计查询的 SQL 模板为：

```sql
WHERE created_at BETWEEN {startDate:DateTime64} AND {endDate:DateTime64}
GROUP BY ${getDateSQL('created_at', unit, timezone)}
```

- `WHERE` 用的是 **UTC 时间范围**（startAt/endAt），从物理存储中过滤出正确的事件。
- `GROUP BY` 用的是 **时区对齐的桶**（`date_trunc('day', created_at, 'Asia/Shanghai')`），确保桶边界与用户时区的日期对齐。

两者协同的效果：
- 即使 `startAt` 对应的 UTC 时间跨了"UTC 意义上的两天"，只要用户时区的日期桶对齐正确，桶内的事件一定是该用户时区"同一天"的。

### 5.4 ClickHouse 的时区时间范围过滤

在 ClickHouse 的 [`getDateQuery`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/clickhouse.ts#L182-L200) 中还有一个可选的时区转换：

```ts
if (timezone) {
  return `and created_at between toTimezone({startDate:DateTime64},{timezone:String})
    and toTimezone({endDate:DateTime64},{timezone:String})`;
}
```

- `toTimezone` 将传入的 UTC 时间戳转为指定时区的 DateTime。
- 但由于 `created_at` 本身是 `DateTime('UTC')`，ClickHouse 在做 `BETWEEN` 比较时会自动做时区转换，所以这实际上是**等价的**。

### 5.5 前端时序填充（generateTimeSeries）

[`generateTimeSeries`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/date.ts#L349-L377) 负责补齐空桶：

```ts
export function generateTimeSeries(data, minDate, maxDate, unit, locale) {
  const add = DATE_FUNCTIONS[unit].add;
  const start = DATE_FUNCTIONS[unit].start;
  const fmt = DATE_FORMATS[unit];

  let current = start(minDate);
  const end = start(maxDate);
  const timeseries = [];

  while (isBefore(current, end) || isEqual(current, end)) {
    timeseries.push(formatDate(current, fmt, locale));
    current = add(current, 1);
  }
  // ...
}
```

- `minDate`/`maxDate` 是前端 `useDateRange` 计算出的"用户时区本地时间"。
- `start(minDate)` 对齐到桶起点（如 `startOfDay`）。
- 逐个 +1 unit 生成完整时间轴。
- 查询返回的桶标签（如 `2024-01-02 00:00:00`）通过 `formatDate` 格式化后与时间轴匹配。

**这意味着**：即使某个桶没有数据（SQL 不返回该行），前端也能补出 `y: null` 的空桶，保证时间轴连续不断。

### 5.6 实时图表的特殊处理

[`RealtimeChart`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/metrics/RealtimeChart.tsx) 使用 [`formatSeriesTimezone`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useTimezone.ts#L31-L61) 做前端时区转换：

```ts
const formatSeriesTimezone = (data: any, column: string, timezone: string) => {
  return data.map(item => {
    const date = new Date(item[column]);
    const format = new Intl.DateTimeFormat('en-US', {
      timeZone: timezone,
      hour12: false,
      year: 'numeric', month: '2-digit', day: '2-digit',
      hour: '2-digit', minute: '2-digit', second: '2-digit',
    });
    // ...
    return { ...item, [column]: `${year}-${month}-${day} ${hour}:${minute}:${second}` };
  });
};
```

- 实时图表不走 SQL 桶对齐，而是在前端用 `Intl.DateTimeFormat` 按时区格式化时间戳。
- 这是因为实时数据本身就是分钟粒度的时间点，不需要 SQL 聚合。

---

## 六、留存报告中的跨日衔接

[`getRetention`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/queries/sql/reports/getRetention.ts) 使用时区桶做"天差"计算：

**PostgreSQL**：
```sql
-- 首次出现日期（时区桶）
min(date_trunc('day', created_at at time zone 'Asia/Shanghai')) as cohort_date
-- 天数差
(created_at at time zone 'Asia/Shanghai')::date - cohort_items.cohort_date::date as day_number
```

**ClickHouse**：
```sql
-- 首次出现日期（时区桶）
min(toDateTime(date_trunc('day', created_at, 'Asia/Shanghai'), 'Asia/Shanghai')) as cohort_date
-- 天数差（除以86400秒）
toInt32((toDateTime(date_trunc('day', created_at, 'Asia/Shanghai'), 'Asia/Shanghai') - cohort_items.cohort_date) / 86400) as day_number
```

- 留存报告中"第 N 天回访"的计算完全基于时区对齐的日期桶。
- 如果不传 timezone，则默认用 UTC 截断，可能导致不同时区用户的"一天"边界偏移。

---

## 七、总结：桶对齐机制的核心原则

| 阶段 | 时区处理 | 关键代码位置 |
|------|---------|------------|
| **写入** | 事件始终以 UTC 落库，不涉及时区 | [`saveEvent.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/queries/sql/events/saveEvent.ts#L245) `getUTCString(createdAt)` |
| **预聚合** | 物化视图按 UTC 整小时桶 | [`schema.sql`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/db/clickhouse/schema.sql#L223) `toStartOfHour(created_at)` |
| **查询-桶截断** | `date_trunc(unit, field, timezone)` 在指定时区上截断 | [`clickhouse.ts getDateSQL`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/clickhouse.ts#L62-L67)，[`prisma.ts getDateSQL`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/prisma.ts#L50-L56) |
| **查询-范围过滤** | 前端将用户时区的日期边界转为 UTC 时间戳，SQL 用 UTC BETWEEN | [`useDateParameters`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useDateParameters.ts#L10-L11) `localToUtc` |
| **前端填充** | `generateTimeSeries` 在用户时区本地时间上递增填充空桶 | [`date.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/date.ts#L349-L377) |
| **实时** | 前端 `Intl.DateTimeFormat` 做时区格式化，不走 SQL 桶 | [`useTimezone.ts formatSeriesTimezone`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useTimezone.ts#L31-L61) |

### 核心设计要点

1. **写入无时区**：原始事件只存 UTC 时间戳，不做任何桶对齐或时区转换，保留最大灵活性。
2. **查询时对齐**：桶对齐完全在查询时通过 SQL `date_trunc(field, unit, timezone)` 完成，属于"读时计算"模式。
3. **预聚合与桶解耦**：`website_event_stats_hourly` 按 UTC 小时桶预聚合，查询时再按用户时区 `date_trunc` 做二次截断（如小时桶可直接映射，天桶则跨多个 UTC 小时桶合并）。
4. **前端负责边界**：用户选择的"日期范围"在前端转换为 UTC 时间戳，确保 SQL 的 `BETWEEN` 与时区桶截断协同一致。
5. **空桶前端补齐**：SQL 只返回有数据的桶，空桶由 `generateTimeSeries` 在前端填充，保证时间轴连续。
