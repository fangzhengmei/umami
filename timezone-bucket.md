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
- 预聚合粒度是小时，因此 `unit=minute` 的查询无法从预聚合表获取数据，必须回源原始表。

### 2.4 查询时的桶选择策略：EVENT_COLUMNS 与原始表的取舍

统计查询（如 [`getPageviewStats`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/queries/sql/pageviews/getPageviewStats.ts#L59-L99)）根据条件选择原始表或预聚合表：

```ts
if (EVENT_COLUMNS.some(item => Object.keys(filters).includes(item)) || unit === 'minute') {
  // 查 website_event 原始表
} else {
  // 查 website_event_stats_hourly 预聚合表
}
```

**触发回源原始表的两个条件**：

| 条件 | 原因 |
|------|------|
| `unit === 'minute'` | 预聚合表只有小时粒度，分钟级桶需要秒级原始数据 |
| 过滤参数命中 `EVENT_COLUMNS` | 预聚合表中这些字段是 `groupArray` 数组类型，无法直接 WHERE 过滤 |

**EVENT_COLUMNS 包含**：`path`、`entry`、`exit`、`referrer`、`domain`、`title`、`query`、`event`、`tag`、`hostname`、`utmSource`、`utmMedium`、`utmCampaign`、`utmContent`、`utmTerm`，共 15 个字段。这些字段全部属于 `website_event` 表的事件属性，对应 [`FILTER_COLUMNS`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/constants.ts#L72-L97) 中的 url_path、referrer_domain 等列。

**性能取舍的设计意图**：
- 无事件级过滤时：走预聚合表 + `sum(views)`，吞吐高、延迟低。
- 有事件级过滤或分钟粒度时：回源 `website_event` 原始表，精确但开销大。
- `SESSION_COLUMNS`（browser/os/device/country 等）的过滤**不会**触发回源，因为这些是维度列，在预聚合表中也是标量低基数字段。

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

### 3.3 时区规范化：前端 `canonicalizeTimezone` 与服务端 `normalizeTimezone` 两条通道

时区字符串的规范化在前端和服务端各有一条独立的通道，使用的映射表**不同**。

**前端通道：`canonicalizeTimezone` + `TIMEZONE_LEGACY`**

[`useTimezone.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useTimezone.ts#L79-L81)：

```ts
const canonicalizeTimezone = (timezone: string): string => {
  return TIMEZONE_LEGACY[timezone] ?? timezone;
};
```

[`TIMEZONE_LEGACY`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/constants.ts#L698-L717) 共 17 条映射，包括：
- `Asia/Batavia` → `Asia/Jakarta`
- `Asia/Calcutta` → `Asia/Kolkata`
- `Asia/Chongqing` / `Asia/Harbin` → `Asia/Shanghai`
- `Europe/Kiev` / `Europe/Zaporozhye` → `Europe/Kyiv`
- `Etc/UTC` → `UTC`
- `US/Arizona` / `US/Central` / `US/Eastern` / `US/Mountain` / `US/Pacific` / `US/Samoa` → 对应标准名

调用时机：`useDateParameters` 在组装请求参数时调用，确保发出的 timezone 是标准名。

**服务端通道：`normalizeTimezone` + `TIMEZONE_MAPPINGS`**

[`date.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/date.ts#L108-L114)：

```ts
const TIMEZONE_MAPPINGS: Record<string, string> = {
  'Asia/Calcutta': 'Asia/Kolkata',
};

export function normalizeTimezone(timezone: string): string {
  return TIMEZONE_MAPPINGS[timezone] || timezone;
}
```

[`timezoneParam`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/schema.ts#L5-L10) Zod schema 在 `transform` 阶段调用：

```ts
export const timezoneParam = z
  .string()
  .refine((value: string) => isValidTimezone(value), { message: 'Invalid timezone' })
  .transform((value: string) => normalizeTimezone(value));
```

**两条通道的差异**：

| 维度 | 前端 canonicalizeTimezone | 服务端 normalizeTimezone |
|------|-------------------------|-------------------------|
| 映射表 | `TIMEZONE_LEGACY`（17 条） | `TIMEZONE_MAPPINGS`（1 条） |
| 所在文件 | `constants.ts` | `date.ts` |
| 调用时机 | 请求参数组装时 | Zod schema 校验 transform 阶段 |
| 覆盖范围 | 更广，含旧 US/时区别名 | 仅一条 Asia/Calcutta 兼容性 |

服务端映射表更精简，是因为它依赖 `Intl.DateTimeFormat` 做合法性校验——大多数旧别名在现代 JS 运行时已被 `Intl` 自动识别，无需额外映射。

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

### 4.3 非整数偏移时区与 hourly 预聚合的桶错位

**问题背景**：预聚合表 `website_event_stats_hourly` 的桶是 **UTC 整小时** 对齐的。对于 UTC 偏移为整数小时的时区（如 `Asia/Shanghai` +8），UTC 整点也是用户时区的整点，`date_trunc('hour', created_at, timezone)` 二次截断能正确映射。

但对于**非整数偏移时区**（如 `Asia/Kolkata` +5:30、`Asia/Kathmandu` +5:45、`Asia/Tehran` +3:30），情况不同：

- UTC `18:00:00` → 印度标准时间 (IST) `23:30:00`，不是整点。
- 预聚合表的 UTC 小时桶（18:00–18:59 UTC）对应 IST 的 23:30–00:29，**跨越了两个 IST 小时**（23 点和 00 点）。

**对不同粒度的影响**：

| 统计粒度 | 影响 | 说明 |
|---------|------|------|
| `day` 及以上 | 无影响 | 天级截断只关心日期是否变化，小时级偏移不改变日期归属 |
| `hour` | 有错位 | 从 hourly 预聚合表读 hour 桶时，UTC 小时桶的"边界"不在用户时区整点，单个 UTC 小时桶的事件会被 `date_trunc` 分到不同的用户时区小时桶，但总和仍然正确 |
| `minute` | 不影响 | 分钟级强制回源原始表，不走预聚合 |

**结论**：非整数偏移时区下，若走 hourly 预聚合表且 `unit=hour`，桶边界在用户时区意义上是"错位"的，但**计数总和是准确的**——因为每个事件都通过 `date_trunc` 正确归桶，只是预聚合的 UTC 小时桶需要被拆分到不同的用户时区小时桶中（ClickHouse 的 `date_trunc` 会在查询时完成这件事）。

### 4.4 DST 切换日的非整桶处理

夏令时（DST）切换日会出现"一天 23 小时"或"一天 25 小时"的情况：

- **春季向前拨**（Spring Forward）：某天凌晨 2 点直接跳到 3 点，当天少一小时（23 小时）。
- **秋季向后拨**（Fall Back）：某天凌晨 2 点倒回 1 点，当天多一小时（25 小时），有一个小时重复。

**Umami 的处理方式**：

1. **桶截断**：`date_trunc('day', created_at, timezone)` 仍然能正确截断到当天 00:00。DST 切换日的"一天"物理时长不是 24 小时，但桶的定义仍然是"日历日"。

2. **小时数不对等**：
   - 春季 DST 切换日：小时桶会少一个（少了被跳过的那一小时）。
   - 秋季 DST 切换日：小时桶会多一个（重复的小时会有两个桶，或合并取决于数据库实现）。

3. **与预聚合表的交互**：DST 切换发生在用户时区的凌晨，通常对应 UTC 的非整点时刻。由于预聚合表是 UTC 小时桶，DST 切换不会造成预聚合桶的异常，只是在查询时 `date_trunc('day', ..., timezone)` 的结果会体现出"当天的小时数不同"。

4. **留存报告中的"天差"**：[`getRetention`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/queries/sql/reports/getRetention.ts) 按"天"计算回访，DST 切换不影响日差计算（因为按日历日算，不是按 24 小时算）。

### 4.5 主统计 vs 事件列表：两条 SQL 时间过滤路径

**主统计查询（getPageviewStats / getSessionStats / getEventStats）不走 `getDateQuery`**。

主统计查询的 SQL 模板中，时间范围是**直接硬写**在 WHERE 子句里的：

```sql
-- getPageviewStats 主统计
WHERE website_id = {websiteId:UUID}
  AND created_at BETWEEN {startDate:DateTime64} AND {endDate:DateTime64}
  AND event_type NOT IN (2, 5)
```

[`getDateQuery`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/clickhouse.ts#L182-L200) 是另一条独立的时间过滤路径，服务于**事件列表、渠道/UTM、目标/漏斗/旅程**等查询：

```ts
function getDateQuery(filters: Record<string, any>) {
  const { startDate, endDate, timezone } = filters;
  if (startDate) {
    if (endDate) {
      if (timezone) {
        return `and created_at between toTimezone({startDate:DateTime64},{timezone:String})
          and toTimezone({endDate:DateTime64},{timezone:String})`;
      }
      return `and created_at between {startDate:DateTime64} and {endDate:DateTime64}`;
    }
    // ...
  }
  return '';
}
```

**两条路径的差异**：

| 维度 | 主统计路径 | getDateQuery 路径 |
|------|-----------|-----------------|
| SQL 写法 | 直接写在 WHERE 中，变量名固定 | 通过 parseFilters 返回 dateQuery，变量注入 |
| 时区转换 | 直接用 UTC BETWEEN，时区转换只发生在桶截断 | ClickHouse 版会用 `toTimezone()` 做显式时区转换 |
| 服务的查询 | getPageviewStats / getSessionStats / getEventStats | getWebsiteEvents（事件列表）、getChannelMetrics（渠道）、getGoal（目标）、getJourney（旅程）、getFunnel（漏斗）、getRevenueSessions（收入会话）、getRealtimeActivity（实时活动）等 |

**注意**：PostgreSQL 版的 `getDateQuery` **不做**时区转换，与主统计路径行为一致；只有 ClickHouse 版有 `toTimezone()` 调用。由于 `created_at` 列本身是 `DateTime('UTC')`，传入的参数也是 UTC 时间戳，`toTimezone()` 转换在语义上等价于直接比较。

### 4.6 ClickHouse 日期格式化（用于周流量等）

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

### 4.7 周流量热力图的特殊桶

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

### 5.4 前端时序填充（generateTimeSeries）

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

**DST 切换日的特别说明**：`generateTimeSeries` 使用 `date-fns` 的 `addDays`/`addHours` 在用户时区本地时间上递增。在 DST 切换日：
- 春季向前拨：`addHours` 会跳过缺失的那一小时，生成的时间轴少一个桶。
- 秋季向后拨：`addHours` 会按日历小时递增，重复的小时只出现一次。
- 这与 SQL `date_trunc` 的行为可能存在细微差异，但日级及以上粒度不受影响。

### 5.5 实时图表的特殊处理

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
- DST 切换日的日差计算仍然正确（按日历日而非 24 小时）。

---

## 七、总结：桶对齐机制的核心原则

| 阶段 | 时区处理 | 关键代码位置 |
|------|---------|------------|
| **写入** | 事件始终以 UTC 落库，不涉及时区 | [`saveEvent.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/queries/sql/events/saveEvent.ts#L245) `getUTCString(createdAt)` |
| **预聚合** | 物化视图按 UTC 整小时桶 | [`schema.sql`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/db/clickhouse/schema.sql#L223) `toStartOfHour(created_at)` |
| **查询-桶截断** | `date_trunc(unit, field, timezone)` 在指定时区上截断 | [`clickhouse.ts getDateSQL`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/clickhouse.ts#L62-L67)，[`prisma.ts getDateSQL`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/prisma.ts#L50-L56) |
| **查询-范围过滤（主统计）** | 直接 UTC BETWEEN，时区转换仅在桶截断 | [`getPageviewStats.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/queries/sql/pageviews/getPageviewStats.ts#L71-L73) |
| **查询-范围过滤（列表/渠道）** | `getDateQuery`，ClickHouse 版带 `toTimezone` | [`clickhouse.ts getDateQuery`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/clickhouse.ts#L182-L200) |
| **时区规范化（前端）** | `canonicalizeTimezone` + `TIMEZONE_LEGACY`（17 条） | [`useTimezone.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useTimezone.ts#L79-L81) |
| **时区规范化（服务端）** | `normalizeTimezone` + `TIMEZONE_MAPPINGS`（1 条） | [`date.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/date.ts#L112-L114) |
| **前端填充** | `generateTimeSeries` 在用户时区本地时间上递增填充空桶 | [`date.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/date.ts#L349-L377) |
| **实时** | 前端 `Intl.DateTimeFormat` 做时区格式化，不走 SQL 桶 | [`useTimezone.ts formatSeriesTimezone`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useTimezone.ts#L31-L61) |

### 核心设计要点

1. **写入无时区**：原始事件只存 UTC 时间戳，不做任何桶对齐或时区转换，保留最大灵活性。
2. **查询时对齐**：桶对齐完全在查询时通过 SQL `date_trunc(field, unit, timezone)` 完成，属于"读时计算"模式。
3. **预聚合与桶解耦**：`website_event_stats_hourly` 按 UTC 小时桶预聚合，查询时再按用户时区 `date_trunc` 做二次截断。非整数偏移时区的小时级统计会有桶边界错位，但计数总和准确。
4. **前端负责边界**：用户选择的"日期范围"在前端转换为 UTC 时间戳，确保 SQL 的 `BETWEEN` 与时区桶截断协同一致。
5. **两条时间过滤路径**：主统计查询直接写死 UTC BETWEEN；事件列表/渠道/目标等查询通过 `getDateQuery` 生成时间条件，ClickHouse 版带 `toTimezone`。
6. **两条规范化通道**：前端 `canonicalizeTimezone` 用 `TIMEZONE_LEGACY`（17 条），服务端 `normalizeTimezone` 用 `TIMEZONE_MAPPINGS`（仅 1 条），覆盖范围不同。
7. **EVENT_COLUMNS 触发回源**：涉及事件级字段过滤或分钟粒度时，必须回源原始表，无法走预聚合。
8. **DST 透明处理**：`date_trunc` 天然处理 DST，桶的定义按"日历日/小时"而非固定时长，DST 切换日小时数不对等但语义正确。
9. **空桶前端补齐**：SQL 只返回有数据的桶，空桶由 `generateTimeSeries` 在前端填充，保证时间轴连续。
