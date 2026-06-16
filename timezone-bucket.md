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

注意这里用的是 `localToUtc`（定义在 [`useTimezone.ts#L71-L73`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useTimezone.ts#L71-L73)）：

```ts
const localToUtc = (date) => zonedTimeToUtc(date, localTimeZone); // localTimeZone = 浏览器系统时区
```

`localToUtc` 使用的是**浏览器系统时区**，而不是用户在 Umami 中选择的显示时区。它与 `parseDateRange` 中的 `utcToZonedTime(date, timezone)`（使用用户选时区）的组合使用，**在浏览器系统时区 == 用户选的显示时区这一前提成立时**，才能得到正确的 startAt/endAt UTC 时间戳（详见第五节详细推演与行为定性）。

### 3.3 时区规范化：前端 `canonicalizeTimezone` 与服务端 `normalizeTimezone` 两条通道

时区字符串的规范化在前端和服务端各有一条独立的通道，使用的映射表**不同**。

**前端通道：`canonicalizeTimezone` + `TIMEZONE_LEGACY`**

[`useTimezone.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useTimezone.ts#L79-L81)：

```ts
const canonicalizeTimezone = (timezone: string): string => {
  return TIMEZONE_LEGACY[timezone] ?? timezone;
};
```

[`TIMEZONE_LEGACY`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/constants.ts#L698-L717) 共 **18 条**映射：

| 旧别名 | 标准名 |
|--------|-------|
| Asia/Batavia | Asia/Jakarta |
| Asia/Calcutta | Asia/Kolkata |
| Asia/Chongqing | Asia/Shanghai |
| Asia/Harbin | Asia/Shanghai |
| Asia/Jayapura | Asia/Pontianak |
| Asia/Katmandu | Asia/Kathmandu |
| Asia/Macao | Asia/Macau |
| Asia/Rangoon | Asia/Yangon |
| Asia/Saigon | Asia/Ho_Chi_Minh |
| Europe/Kiev | Europe/Kyiv |
| Europe/Zaporozhye | Europe/Kyiv |
| Etc/UTC | UTC |
| US/Arizona | America/Phoenix |
| US/Central | America/Chicago |
| US/Eastern | America/New_York |
| US/Mountain | America/Denver |
| US/Pacific | America/Los_Angeles |
| US/Samoa | Pacific/Pago_Pago |

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
| 映射表 | `TIMEZONE_LEGACY`（18 条） | `TIMEZONE_MAPPINGS`（1 条） |
| 所在文件 | `constants.ts` | `date.ts` |
| 调用时机 | 请求参数组装时 | Zod schema 校验 transform 阶段 |
| 覆盖范围 | 更广，含旧 US/*、Asia/*、Europe/* 历史别名 | 仅一条 Asia/Calcutta → Asia/Kolkata |

二者映射表条数不同，属于前后端独立维护的状态。

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
- `to_char(..., format)`：**直接在 SQL 中格式化为字符串**作为桶标签返回。

**示例**：`timezone = 'Asia/Shanghai'`，`unit = 'day'`
- UTC `2024-01-01 20:00:00` → 上海时间 `2024-01-02 04:00:00` → 截断为 `2024-01-02` → 该事件归入 1 月 2 日的桶。
- UTC `2024-01-01 15:30:00` → 上海时间 `2024-01-01 23:30:00` → 截断为 `2024-01-01` → 该事件归入 1 月 1 日的桶。

**UTC 模式下的格式差异**：

| 条件 | 格式模板 | 输出示例 |
|------|---------|---------|
| timezone ≠ 'utc' | `YYYY-MM-DD HH24:00:00` (DATE_FORMATS) | `2024-01-02 00:00:00` |
| timezone = 'utc' 或无 | `YYYY-MM-DD"T"HH24:00:00"Z"` (DATE_FORMATS_UTC) | `2024-01-01T20:00:00Z` |

UTC 模式在格式中嵌入了 `T` 和 `Z` 标记，便于前端区分。

### 4.2 ClickHouse 的桶对齐：`toDateTime` 第二参数的语义

[`getDateSQL`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/clickhouse.ts#L62-L67)：

```ts
function getDateSQL(field: string, unit: string, timezone?: string) {
  if (timezone) {
    return `toDateTime(date_trunc('${unit}', ${field}, '${timezone}'), '${timezone}')`;
  }
  return `toDateTime(date_trunc('${unit}', ${field}))`;
}
```

**ClickHouse `date_trunc` 行为**：
- `date_trunc('day', created_at, 'Asia/Shanghai')`：ClickHouse 的 `date_trunc` 支持第三参数时区名，先将 UTC 时间转到目标时区再截断。
- 返回值已经是一个 DateTime 类型，其**内部 UTC 时间戳**是正确的截断结果（例如上海 00:00:00 对应的 UTC 前一天 16:00:00）。

**外层 `toDateTime(..., timezone)` 的语义**：
- 当第一参数已经是 DateTime 类型时，`toDateTime(x, timezone)` **不改变 x 内部的 UTC 时间戳值**，也不做任何时间点的换算。
- 它的作用是**给这个 DateTime 值附加一个 `timezone` 元数据标记**（相当于在结果类型上写死 `timezone='Asia/Shanghai'`）。
- 这个标记的影响体现在 ClickHouse 序列化结果集为 JSON 时：`formatDateTime` 或隐式的 DateTime→String 转换会按该标记的时区来格式化输出字符串。
- 例如：内部 UTC = `2024-06-15T18:30:00Z`（= Kolkata 2024-06-16 00:00:00 IST）：
  - 若没有 `toDateTime(..., 'Asia/Kolkata')`，默认按列类型 `DateTime('UTC')` 序列化为 `"2024-06-15 18:30:00"`（前端会误解为用户时区 18:30）。
  - 若加了 `toDateTime(..., 'Asia/Kolkata')`，序列化为 `"2024-06-16 00:00:00"`（前端按用户时区识别正确）。

**与 PostgreSQL 的对比**：PostgreSQL 用 `to_char` 直接在 SQL 中把桶格式化为字符串并附加时区格式，不需要额外的"时区标记"步骤；ClickHouse 通过 `toDateTime` 的第二参数达到同样的目的。

### 4.3 非整数偏移时区在 hourly 视图上的 day/hour 桶归属：具体用例

预聚合表 `website_event_stats_hourly` 的桶是 **UTC 整小时**对齐的。当用户时区为整数偏移（如 `Asia/Shanghai` +8）时，UTC 整点也是用户时区整点，`date_trunc` 二次截断能正确映射。下面用 **`Asia/Kolkata` (+5:30)** 作为半小时偏移的典型用例，逐事件推演桶归属。

**场景设定**：
- 用户显示时区：`Asia/Kolkata`（IST，UTC+5:30）
- 统计范围：IST 2024-06-16 全天（= UTC 2024-06-15 18:30 ~ UTC 2024-06-16 18:29:59）

**3 个具体事件**：

| 事件 | 原始 UTC 时间 | 对应的 IST 本地时间 | 理论正确桶（原始表查） |
|------|-------------|-------------------|---------------------|
| E1 | 2024-06-16 18:10 UTC | 2024-06-16 23:40 IST | day=6/16, hour=23 |
| E2 | 2024-06-16 18:40 UTC | 2024-06-17 00:10 IST | day=6/17, hour=00 |
| E3 | 2024-06-16 20:15 UTC | 2024-06-17 01:45 IST | day=6/17, hour=01 |

**预聚合 hourly 视图中实际存储的行**：
E1 和 E2 的 UTC 时间都落在 `18:00-18:59` 这个 UTC 小时桶里，被合并为一行：

| created_at (UTC) | views |
|-----------------|-------|
| 2024-06-16 18:00 UTC | 2 （E1 + E2 合计） |
| 2024-06-16 20:00 UTC | 1 （E3） |

#### Day 桶归属推演

走 **hourly 预聚合表**，对 created_at 做 `date_trunc('day', created_at, 'Asia/Kolkata')`：

| hourly 行 | created_at | trunc 过程（Kolkata） | 归到 day 桶 | views |
|-----------|-----------|----------------------|-----------|-------|
| row-18:00 | 18:00 UTC → 23:30 IST | trunc day = 2024-06-16 00:00 IST | **6/16** | 2 |
| row-20:00 | 20:00 UTC → 01:30 IST (6/17) | trunc day = 2024-06-17 00:00 IST | **6/17** | 1 |

**hourly 视图给出的 day 桶结果**：6/16=2, 6/17=1
**理论正确的 day 桶（原始表查）**：6/16=1（E1）, 6/17=2（E2+E3）

→ **Day 桶错位！** 差了 1 条事件从 6/17 误归到 6/16，原因是 `18:00-18:59 UTC` 这个桶覆盖了 IST 的 23:30-00:29，**跨越了日界**，但预聚合行用 created_at=18:00 UTC（=IST 23:30）作为代表点 trunc，把整个桶的 views 都归到了 6/16。

#### Hour 桶归属推演

同样走 hourly 视图，做 `date_trunc('hour', created_at, 'Asia/Kolkata')`：

| hourly 行 | created_at | trunc 过程（Kolkata） | 归到 hour 桶 | views |
|-----------|-----------|----------------------|-----------|-------|
| row-18:00 | 18:00 UTC → 23:30 IST | trunc hour = IST 23:00 | **23:00** | 2 |
| row-20:00 | 20:00 UTC → 01:30 IST (6/17) | trunc hour = IST 01:00 | **01:00** | 1 |

**hourly 视图给出的 hour 桶**：IST 23:00=2, IST 01:00=1
**理论正确的 hour 桶**：IST 23:00=1（E1）, IST 00:00=1（E2）, IST 01:00=1（E3）

→ **Hour 桶错位！** IST 00:00 的桶丢失了 1 条，IST 23:00 的桶多了 1 条。

#### 各粒度影响总表

| 统计粒度 | 是否受影响 | 受影响的 UTC 小时桶 | 说明 |
|---------|-----------|------------------|------|
| `year/month/week` | 极大概率不受 | — | 1 小时的日级偏差在更大粒度可忽略 |
| `day` | **受影响** | 跨越用户时区日界的那 1 个 UTC 小时桶 | 对 +5:30 是 18:00 UTC；每天最多 1 个桶出问题 |
| `hour` | **所有 UTC 小时桶都受影响** | 全部 24 个桶 | 非整数偏移下每个 UTC 小时桶都横跨 2 个用户时区小时 |
| `minute` | 不影响 | — | 强制回源原始表，不走预聚合 |

**结论**：非整数偏移时区下，`website_event_stats_hourly` 预聚合表不仅 hour 桶会错位，**day 桶也会错位**（当天跨越日界的那一个 UTC 小时桶会造成跨 1 天的归属偏移）。受影响的事件量占比取决于日界小时桶的流量，但桶归属无法通过 SQL 的 `date_trunc` 修复——因为这是预聚合粒度本身的信息损失。只有走 EVENT_COLUMNS 过滤或 `unit=minute` 触发回源原始表时，归属才完全正确。

#### 多种非整数偏移时区的对照：Kolkata / Tehran / Kathmandu / Marquesas

`Asia/Kolkata (+5:30)` 是半小时偏移的典型代表，但现实中还有几种不同模式的非整数偏移时区。它们在 hourly 预聚合表上的错位模式各有差异：

**四个代表时区的基本属性**：

| 时区 | 标准偏移 | DST | 偏移类型 | UTC 整点对应的本地时间 |
|------|---------|-----|---------|---------------------|
| Asia/Kolkata | +5:30 | 无 | 半小时偏移 | HH:30 |
| Asia/Tehran | +3:30 / +4:30 | 有 | 半小时偏移 + DST | HH:30 / HH:30（偏移量随季节变） |
| Asia/Kathmandu | +5:45 | 无 | 45 分钟偏移 | HH:45 |
| Pacific/Marquesas | -9:30 | 无 | 负半小时偏移 | HH:30（前一天） |

**错位模式对照表**：

| 对比维度 | Kolkata (+5:30) | Tehran (+3:30/+4:30 DST) | Kathmandu (+5:45) | Marquesas (-9:30) |
|---------|----------------|------------------------|-------------------|------------------|
| **每个 UTC 桶跨越几个本地小时** | 2 个 | 2 个 | 2 个 | 2 个 |
| **桶内两小时的时间分配** | 30 分钟 / 30 分钟（均匀） | 30 分钟 / 30 分钟（均匀） | 15 分钟 / 45 分钟（偏斜） | 30 分钟 / 30 分钟（均匀） |
| **跨日界的 UTC 桶数** | 每天 1 个 | 每天 1 个（偏移量随 DST 变，跨日界的 UTC 小时也跟着变） | 每天 1 个 | 每天 1 个（方向相反） |
| **Day 桶错位方向** | 次日事件误归到当日 | 次日事件误归到当日 | 次日事件误归到当日（但只有 15 分钟） | 当日事件误归到前一日（负偏移反向） |
| **Hour 桶错位模式** | 所有 24 个桶全部顺延错 1 位 | 所有 24 个桶全部顺延错 1 位（DST 切换日模式变化） | 所有 24 个桶全部错，但错位程度不均匀（有的桶只含 15 分钟真实数据） | 所有 24 个桶全部错（负偏移方向相反） |
| **DST 额外影响** | 无 | 有：DST 切换日不仅错位模式变，还叠加 DST 本身的非整桶效应 | 无 | 无 |
| **回源原始表后是否正确** | ✅ 完全正确 | ✅ 完全正确 | ✅ 完全正确 | ✅ 完全正确 |

**Kathmandu (+5:45) 的特殊偏斜模式**：

以 UTC 18:00 桶为例：
- UTC 18:00 → Kathmandu 23:45（当天）
- UTC 19:00 → Kathmandu 00:45（次日）
- 桶内 60 分钟分布：当天 23 点占 15 分钟，次日 0 点占 45 分钟
- 整个桶归到**当天 23 点**（按起始时间 trunc）
- 错位程度：当天 23 点桶多了 15 分钟数据，次日 0 点桶少了 45 分钟数据

与 Kolkata（30/30 均分）相比，Kathmandu 的错位更加"偏斜"——有的桶只包含少量真实数据，有的桶包含大部分真实数据。但从"桶数量"的角度看，两者都是每个 UTC 桶横跨 2 个本地小时，全部 24 个桶都错位。

**Tehran 的 DST 叠加效应**：

Tehran 不仅偏移量是半小时，而且有夏令时：
- 标准时（+3:30）：跨日界的 UTC 小时是 20:30 左右
- 夏令时（+4:30）：跨日界的 UTC 小时变成 19:30 左右

一年中错位模式会变化两次（DST 开始和结束时）。DST 切换日还会叠加"一天 23/25 小时"的非整桶效应，与预聚合的 UTC 整点桶错位形成复合影响。

**Marquesas 的负偏移对称性**：

`Pacific/Marquesas (-9:30)` 是负偏移的代表，错位模式与 Kolkata 对称但方向相反：
- Kolkata（正偏移）：次日的事件误归到当日
- Marquesas（负偏移）：当日的事件误归到前一日

但错位的本质和程度完全相同，只是方向相反。

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

与主统计一样，周流量热力图也有**两条查询路径**：
- EVENT_COLUMNS 命中 → 查 `website_event` 原始表
- EVENT_COLUMNS 未命中 → 查 `website_event_stats_hourly` 预聚合表

#### 非整数偏移时区的双向错位：以 Asia/Kolkata (+5:30) 为例

走 hourly 预聚合表时，由于 `created_at` 是 UTC 整点桶，`formatDateTime` 只能按桶的起始时间来格式化，会导致 **hour 维度和 dow 维度的双重错位**。

**具体用例推演**：取 2024-06-16（周日，dow=0）/ 2024-06-17（周一，dow=1）的两个连续 UTC 小时桶。

| UTC 小时桶 | 桶起始 Kolkata 时间 | formatDateTime 结果 | 理论正确分布（桶内 60 分钟） | 归属偏差 |
|-----------|-------------------|-------------------|---------------------------|---------|
| 6/16 18:00-19:00 UTC | 23:30 IST（周日） | `0:23`（周日23点） | 周日 23:30-23:59（30分钟） + 周一 00:00-00:30（30分钟） | **跨日！** 周一 0 点的 30 分钟数据误归到周日 23 点 |
| 6/16 19:00-20:00 UTC | 00:30 IST（周一） | `1:00`（周一0点） | 周一 00:30-01:30（60分钟，不跨日） | 周一 1 点的 30 分钟数据误归到周一 0 点 |
| 6/16 20:00-21:00 UTC | 01:30 IST（周一） | `1:01`（周一1点） | 周一 01:30-02:30（60分钟） | 周一 2 点的 30 分钟数据误归到周一 1 点 |
| ... | ... | ... | ... | ... |
| 6/17 17:00-18:00 UTC | 22:30 IST（周一） | `1:22`（周一22点） | 周一 22:30-23:30（60分钟） | 周一 23 点的 30 分钟数据误归到周一 22 点 |
| 6/17 18:00-19:00 UTC | 23:30 IST（周一） | `1:23`（周一23点） | 周一 23:30-00:30（周二）（60分钟） | **跨日！** 周二 0 点的 30 分钟数据误归到周一 23 点 |

**错位模式总结**：

1. **Hour 维度：所有 24 个 UTC 小时桶全部错位**
   - 每个 UTC 整点桶在 Kolkata 时间中都落在 HH:30 的位置，横跨两个本地小时
   - 每个桶归到起始时间所在的小时，导致后 30 分钟的事件顺延错一位
   - 与主统计的 hour 桶错位模式完全一致

2. **Dow 维度：每天有 1 个 UTC 小时桶跨日界**
   - UTC 18:00 桶（对应 Kolkata 23:30-00:30）横跨两天，dow 从当天变到次日
   - 整个桶归到前一天的 23 点，导致后 30 分钟的事件 dow 也错了
   - dow 错位的比例：每天约 30 分钟的数据错了 dow（占全天的 1/48 ≈ 2.1%）

3. **整体影响**：
   - 周流量热力图的 7×24 格子中，每个格子的数值都有误差
   - 误差是"水平方向"（小时维度）和"垂直方向"（dow 维度）的双重偏移
   - 但所有桶的总和不变，只是在格子之间重新分配

**与主统计的对比**：
- 主统计 day 桶：只有 1 个小时桶跨日界，影响相对较小
- 周热力图：每个小时都错位 + 每天 1 个 dow 错位，影响更广泛
- 共同点：走原始表时都完全正确，走 hourly 预聚合时都因 UTC 整点桶与非整数偏移的不匹配而错位

---

## 五、统计窗口跨日衔接

### 5.1 前端日期范围计算

[`parseDateRange`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/date.ts#L140-L220) 根据用户选的快捷范围（如 `1day`、`1month`）做截断：

```ts
const date = new Date();
const now = timezone ? utcToZonedTime(date, timezone) : date;
// ...
case 'day':
  return {
    startDate: num ? subDays(startOfDay(now), num) : startOfDay(now),
    endDate: endOfDay(now),
    ...
  };
```

- `utcToZonedTime(date, timezone)`：date-fns-tz 函数。它不改变"真实时刻"，而是把返回的 Date 对象内部的 UTC 毫秒值"偏转"了一个量——使得在**浏览器系统时区**下调用 `.getHours()`、`.getDate()` 等 getter 时，得到的数值等于目标时区的本地时间分量。
- 后续的 `startOfDay(now)`、`addDays` 等都是 **date-fns 原生函数**，内部实现基于浏览器时区的 setter（如 `setHours(0,0,0,0)`）。

### 5.2 日期边界转 UTC 传给 API：浏览器时区与用户选时区的对齐问题

[`useDateParameters`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useDateParameters.ts) 中：

```ts
startAt: +localToUtc(startDate),
endAt:   +localToUtc(endDate),
```

其中 `localToUtc` 定义在 [`useTimezone.ts#L71-L73`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useTimezone.ts#L71-L73)：

```ts
const localToUtc = (date) => {
  return zonedTimeToUtc(date, localTimeZone); // localTimeZone = 浏览器系统时区
};
```

`zonedTimeToUtc(date, timezone)` 的含义是：把 `date` 的本地时间组件（按浏览器时区读取的年月日时分秒）当作是指定 `timezone` 的本地时间，然后计算出对应的真实 UTC 时间戳 Date 返回。

**正确性的前提：浏览器系统时区 == 用户选的显示时区**

下面用两种情况代入推演：

#### 情况 A：浏览器时区 = 用户选时区（America/Los_Angeles）

设当前真实 UTC = 2024-06-16T12:00:00Z，用户选时区 = `America/Los_Angeles`（PDT，UTC-7），快捷范围 `1day`：

```
Step 1：utcToZonedTime(date, 'America/Los_Angeles')
  真实 UTC 12:00Z → LA 本地 05:00 PDT
  返回的 Date 内部值调整为：在浏览器（LA）上读 .getHours()=5 → 内部 UTC = 2024-06-16T12:00:00Z（和原值相同）

Step 2：startOfDay(now)
  浏览器（LA）的 setter → LA 本地 2024-06-16 00:00:00 PDT
  对应 UTC = 2024-06-16T07:00:00Z
  startDate 内部值 = 2024-06-16T07:00:00Z

Step 3：localToUtc(startDate) = zonedTimeToUtc(startDate, 'America/Los_Angeles')
  取 startDate 的本地组件（LA 读）：2024-06-16 00:00
  把它当作 LA 本地时间 → UTC = 2024-06-16T07:00:00Z ✓
  startAt = 2024-06-16T07:00:00Z（= LA 06/16 00:00 正确）
```

→ **此时完全正确**。

#### 情况 B：浏览器时区 ≠ 用户选时区（LA 浏览器 + Kolkata 用户选）

设当前真实 UTC = 2024-06-16T12:00:00Z，浏览器系统时区 = `America/Los_Angeles`（PDT，-7），用户在 Umami 中选显示时区 = `Asia/Kolkata`（IST，+5:30），快捷范围 `1day`：

```
Step 1：utcToZonedTime(date, 'Asia/Kolkata')
  真实 UTC 12:00Z → Kolkata 本地 17:30 IST (6/16)
  返回的 Date 需满足：在 LA 浏览器上 getHours()=17, getMinutes()=30
  即 LA 本地 2024-06-16 17:30 → UTC = 2024-06-17T00:30:00Z
  now 内部值 = 2024-06-17T00:30:00Z

Step 2：startOfDay(now)
  浏览器（LA）的 setter：取 LA 本地的年月日 = 2024-06-16，设 00:00:00
  → LA 本地 2024-06-16 00:00:00 PDT → UTC = 2024-06-16T07:00:00Z
  startDate 内部值 = 2024-06-16T07:00:00Z

  但用户真正想要的是"Kolkata 今天 00:00 IST"对应的 UTC：
  Kolkata 2024-06-16 00:00:00 IST → UTC = 2024-06-15T18:30:00Z
  偏差 = 2024-06-16T07:00:00Z - 2024-06-15T18:30:00Z = 12h 30m
  = 用户时区偏移（+5:30） - 浏览器时区偏移（-7:00） = +12:30 ✓

Step 3：localToUtc(startDate) = zonedTimeToUtc(startDate, 'America/Los_Angeles')
  取 startDate 的本地组件（LA 读）：2024-06-16 00:00
  当作 LA 本地 → UTC = 2024-06-16T07:00:00Z（和 startDate 自身一样，没修正偏差）
  startAt = 2024-06-16T07:00:00Z（错误，应为 2024-06-15T18:30:00Z）
```

→ **此时传给 API 的 startAt/endAt 偏移了 12.5 小时**。

#### 总结：date-fns 链路的对齐规则

`parseDateRange` → `startOfDay`/`addDays` → `localToUtc` → `startAt` 的这整条 date-fns 操作链路：

1. **浏览器系统时区 = 用户选时区时**：startAt/endAt 的 UTC 值完全正确。
2. **浏览器系统时区 ≠ 用户选时区时**：startAt/endAt 的 UTC 值系统性偏移了（用户时区偏移 − 浏览器时区偏移）的量。
3. `generateTimeSeries` 的填充也用同样的 date-fns 原生函数（`startOfDay`、`addDays` 等），它生成的时间轴标签是"基于浏览器时区读取 startDate"再做格式化，因此**在字符串标签层面上和 SQL 返回的桶标签可能对不上**——除非前端也使用一致的时区格式化。
4. 注：`useTimezone` 中也提供了一个 `toUtc` 函数（`zonedTimeToUtc(date, timezone)`，基于用户选时区），但 `useDateParameters` 实际使用的是 `localToUtc`（基于浏览器时区）。

### 5.2.1 行为定性：已知 Bug 与历史溯源

**上游仓库 issue 与修复**：

上述系统性偏移是一个**已知 Bug**，在 umami 上游仓库中有完整的报告与修复记录：

- **Issue #4107**（2026 年 3 月报告，v3.0.3 版本）：*"Revenue chart, 'Last 24 hours' shows data according to PC Timezone (not Settings Timezone)"*。用户报告浏览器时区（UTC+5）与 Umami 设置时区（UTC+1）不同时，收入图表按 PC 本地时区显示数据，与头部统计（按设置时区）不一致。

- **PR #4112**（2026 年 3 月 30 日合并入 v3.1.0）：*"fix: use settings timezone in revenue chart date range"*。修复内容是在 `RevenuePage` 中调用 `useDateRange({ timezone })` 时**显式传入用户设置的 timezone 参数**（此前 `RevenuePage` 调用 `useDateRange()` 不带参数，导致 `parseDateRange` 回退到浏览器本地时区，根本没有进入带用户时区的代码路径）。

**当前代码库状态**：

本仓库（fork 自 umami，commit `c0ea3ae` "Migrate tests to Vitest"，作者 Mike Cao 2026-05-14）**已包含 PR #4112 的修复**：
- [RevenuePage.tsx L8-L11](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/app/(main)/websites/[websiteId]/(reports)/revenue/RevenuePage.tsx#L8-L11)：`const { timezone } = useTimezone(); ... useDateRange({ timezone });`
- [WebsiteChart.tsx L14-L15](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/app/(main)/websites/[websiteId]/WebsiteChart.tsx#L14-L15)：同样正确传入 timezone。

**更深层的设计限制（PR #4112 未解决）**：

PR #4112 只修复了"调用 `useDateRange` 时忘记传 timezone"这一层问题。但即使正确传入 timezone，只要**浏览器系统时区 ≠ 用户选的显示时区**，`useDateParameters` 中 `localToUtc` 仍然会造成系统性偏移（如情况 B 的 12.5 小时偏差）。这个更根本的问题在当前代码中仍然存在。

### 5.2.2 `useDateParameters` 选 `localToUtc` 而非 `toUtc` 的设计意图

`useTimezone` 同时提供了**四对**时区转换函数，分为两个正交维度：

| 函数对 | 方向 | 基准时区 | 用途 |
|-------|------|---------|------|
| `toUtc` / `fromUtc` | ↔ UTC | **用户选时区**（settings timezone） | 按用户配置的显示时区做转换 |
| `localToUtc` / `localFromUtc` | ↔ UTC | **浏览器系统时区**（local timezone） | 按浏览器运行时的本地时区做转换 |

实现上全部基于 `date-fns-tz` 的 `zonedTimeToUtc` / `utcToZonedTime`，只是传入的时区参数不同：

```ts
// useTimezone.ts 中四对函数的实现
const toUtc = (date) => zonedTimeToUtc(date, timezone);           // 用户选时区
const fromUtc = (date) => utcToZonedTime(date, timezone);        // 用户选时区
const localToUtc = (date) => zonedTimeToUtc(date, localTimeZone);   // 浏览器系统时区
const localFromUtc = (date) => utcToZonedTime(date, localTimeZone); // 浏览器系统时区
```

**`useDateParameters` 选择 `localToUtc` 的原因：与 `parseDateRange` 形成闭环**

`parseDateRange` 的工作模式是：
1. `utcToZonedTime(date, timezone)` — 用**用户选时区**做正向偏转
2. `startOfDay` / `addDays` 等 date-fns 原生函数 — 用**浏览器时区**做操作

第 2 步的操作结果（startDate/endDate）已经处于"浏览器时区的表示空间"里。要把它转回真实 UTC，必须用**同样以浏览器时区为基准**的 `localToUtc` 做反向偏转，才能形成"偏转→操作→反向偏转"的闭环。

如果改用 `toUtc`（以用户选时区为基准做反向），则：
- 正向偏转为用户时区，操作在浏览器时区，反向偏转为用户时区 → **偏转方向不匹配**，闭环断裂。
- 结果不是简单的"偏移量不同"，而是**整个日期语义都会错乱**（年月日都可能错）。

**两种选择在不同场景下的对比**：

| 场景 | 用 `localToUtc`（当前实现） | 用 `toUtc`（假设修改） |
|------|---------------------------|----------------------|
| 浏览器时区 == 用户选时区 | ✓ 正确（偏转抵消） | ✓ 同样正确（两种方式等价） |
| 浏览器时区 ≠ 用户选时区 | ⚠ 系统性时间偏移（偏移量 = 用户时区偏移 − 浏览器时区偏移），但**日期（年月日）是对的**（因为 startOfDay 操作的是偏转后的日期分量） | ✗ **日期（年月日）可能错乱**，因为用用户时区反向时，会把浏览器时区下读取的日期分量重新解释，导致"差一天"等更严重的问题 |

**结论**：在 `parseDateRange` 采用"用户时区偏转 + 浏览器时区操作"这一既定架构下，`localToUtc` 是形成闭环的**唯一合理选择**——它保证了日期分量（年月日）的正确性，仅在精确时间戳上有系统性偏移。这不是一个理想的设计，而是 date-fns 原生函数只能基于浏览器时区操作这一技术约束下的折衷。

**附带的两个格式化函数**（不参与日期计算，仅用于显示）：

- `formatTimezoneDate`：用 `formatInTimeZone(date, timezone, pattern)` 直接按用户选时区格式化字符串，**不经过 Date 对象的浏览器时区解析**，因此不受上述系统性偏移影响。用于需要正确时区显示的独立日期字段。
- `formatSeriesTimezone`：用 `Intl.DateTimeFormat({ timeZone: timezone })` 显式指定时区格式化，同样**不依赖浏览器时区**，用于实时图表等需要跨时区正确显示的场景。

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
  const timeseries: string[] = [];

  while (isBefore(current, end) || isEqual(current, end)) {
    timeseries.push(formatDate(current, fmt, locale));
    current = add(current, 1);
  }

  const lookup = new Map(data.map(({ x, y, d }) => [formatDate(x, fmt, locale), { x, y, d }]));

  return timeseries.map(t => {
    const { x, y, d } = lookup.get(t) || {};
    return { x: t, d: d ?? x, y: y ?? null };
  });
}
```

关键步骤：
1. `start(minDate)` 对齐到桶起点（`startOfDay`/`startOfHour` 等，均为 date-fns 原生函数）。
2. `add(current, 1)` 逐个 +1 unit 生成完整时间轴（`addDays`/`addHours` 等）。
3. 每一步都用 `formatDate(current, fmt, locale)` 格式化为字符串，加入时间轴。
4. SQL 返回的桶标签 `x`（如 `"2024-06-16 00:00:00"` 字符串）也通过 `formatDate(x, fmt, locale)` 再次格式化（`formatDate` 遇到字符串会 `new Date(string)`），然后用 lookup 匹配。

**匹配逻辑的细节**：
- SQL 返回的桶标签是 ClickHouse/PostgreSQL 按用户时区格式化出的本地时间字符串（如 `"2024-06-16 00:00:00"` = IST 当天开始）。
- `new Date("2024-06-16 00:00:00")`：这个字符串**没有时区标记**，浏览器按**本地时区（系统时区）**解析为 Date 对象。
- `formatDate(date, fmt)` 再按浏览器本地时区格式化回字符串。
- 因此**只有在浏览器时区 = 用户选时区时**，SQL 返回的 `"2024-06-16 00:00:00"` 在"浏览器解析 → 浏览器格式化"后才能和 generateTimeSeries 生成的轴标签完全一致。
- 当浏览器时区 ≠ 用户选时区时，lookup 会匹配失败，出现大量 `y: null` 空桶（即使 SQL 返回了有效数据）。

**DST 切换日的特别说明**：`generateTimeSeries` 使用 `date-fns` 的 `addDays`/`addHours` 在用户时区本地时间上递增。在 DST 切换日：
- 春季向前拨：`addHours` 会跳过缺失的那一小时，生成的时间轴少一个桶。
- 秋季向后拨：`addHours` 会按日历小时递增，重复的小时只出现一次。
- 这与 SQL `date_trunc` 的行为一致，日级及以上粒度不受影响。

#### DST 具体用例推演：America/New_York 春令 2024-03-10

**场景设定**：
- 浏览器系统时区 = `America/New_York`
- 用户选时区 = `America/New_York`（正确性前提满足）
- 真实 UTC = 2024-03-10T12:00:00Z（此时 NYC 已进入夏令时 EDT，UTC-4）
- 快捷范围 = `1day`，`unit = 'hour'`
- DST 切换：2024-03-10 凌晨 2:00 EST（UTC-5）→ 直接跳到 3:00 EDT（UTC-4），跳过 1 小时

**`parseDateRange` 中 `startOfDay` 推演**：

```
Step 1：utcToZonedTime(date, 'America/New_York')
  真实 UTC 12:00Z → NYC 本地 08:00 EDT (3/10，已夏令时)
  返回的 Date 内部值 = 2024-03-10T12:00:00Z（和原值相同，因为浏览器时区=用户选时区）

Step 2：startOfDay(now)
  date-fns startOfDay 内部调用 setHours(0,0,0,0)，基于浏览器时区（NYC）
  → NYC 本地 2024-03-10 00:00:00
  注意：3/10 00:00 还是 EST（UTC-5），因为 DST 切换在凌晨 2 点
  → 对应 UTC = 2024-03-10T05:00:00Z
  startDate 内部值 = 2024-03-10T05:00:00Z

Step 3：localToUtc(startDate) = zonedTimeToUtc(startDate, 'America/New_York')
  取 startDate 的本地组件（NYC 读）：2024-03-10 00:00
  把它当作 NYC 本地时间 → 00:00 EST = UTC 05:00Z
  startAt = 2024-03-10T05:00:00Z ✓（= NYC 03/10 00:00 正确）
```

**`generateTimeSeries` 中 `addHours` 推演**（unit=hour）：

```
start(minDate) = startOfHour(startDate) = startDate = 2024-03-10T05:00:00Z

addHours(current, 1) 迭代 24 次（其中 1 次跨越 DST 切换点）：

| 迭代 | current 内部 UTC | NYC 本地时间（读） | format 输出 | 说明 |
|------|----------------|-------------------|-----------|------|
| 0 | 05:00Z | 00:00 EST | "2024-03-10 00" | 正常 |
| 1 | 06:00Z | 01:00 EST | "2024-03-10 01" | 正常 |
| 2 | 07:00Z | 03:00 EDT | "2024-03-10 03" | **跳过了 02:00！** 2 点直接变 3 点 |
| 3 | 08:00Z | 04:00 EDT | "2024-03-10 04" | 恢复正常 |
| 4 | 09:00Z | 05:00 EDT | "2024-03-10 05" | 正常 |
| ... | ... | ... | ... | ... |
| 22 | 03:00Z (3/11) | 23:00 EDT (3/10) | "2024-03-10 23" | 当天最后一小时 |
```

**DST 切换日的关键现象**：
1. `startOfDay` 仍然正确，因为 00:00 在 DST 切换之前（02:00），仍为 EST。
2. `addHours` 第 2 次迭代时自动跳过缺失的 02:00 小时，直接从 01:00 跳到 03:00。
3. 最终生成 **23 个小时桶**（缺少 "2024-03-10 02"），与 SQL `date_trunc` 的行为完全一致，前后端不会出现不匹配。
4. DST 切换是由 date-fns 原生函数内部自动处理的，业务代码无需特殊判断。

#### Fall back 对称推演：America/New_York 秋令 2024-11-03

**场景设定**：
- 浏览器系统时区 = `America/New_York`
- 用户选时区 = `America/New_York`（正确性前提满足）
- 真实 UTC = 2024-11-03T12:00:00Z（此时 NYC 处于标准时间 EST，UTC-5）
- 快捷范围 = `1day`，`unit = 'hour'`
- DST 切换：2024-11-03 凌晨 2:00 EDT → 回拨到 1:00 EST，**1:00-2:00 这一小时出现两次**

**`parseDateRange` 中 `startOfDay` 推演**：

```
Step 1：utcToZonedTime(date, 'America/New_York')
  真实 UTC 12:00Z → NYC 本地 07:00 EST (11/03，已标准时)
  返回的 Date 内部值 = 2024-11-03T12:00:00Z

Step 2：startOfDay(now)
  date-fns startOfDay 内部调用 setHours(0,0,0,0)，基于浏览器时区（NYC）
  → NYC 本地 2024-11-03 00:00:00
  注意：11/03 00:00 还是 EDT（UTC-4），因为 DST 回拨在凌晨 2 点
  → 对应 UTC = 2024-11-03T04:00:00Z
  startDate 内部值 = 2024-11-03T04:00:00Z

Step 3：localToUtc(startDate) = zonedTimeToUtc(startDate, 'America/New_York')
  取 startDate 的本地组件（NYC 读）：2024-11-03 00:00
  把它当作 NYC 本地时间 → 00:00 EDT = UTC 04:00Z
  startAt = 2024-11-03T04:00:00Z ✓（= NYC 11/03 00:00 正确）
```

**`generateTimeSeries` 中 `addHours` 推演**（unit=hour）：

```
start(minDate) = startOfHour(startDate) = startDate = 2024-11-03T04:00:00Z

addHours(current, 1) 迭代，跨越 DST 回拨点：

| 迭代 | current 内部 UTC | NYC 本地时间（读） | format 输出 | 说明 |
|------|----------------|-------------------|-----------|------|
| 0 | 04:00Z | 00:00 EDT | "2024-11-03 00" | 正常 |
| 1 | 05:00Z | 01:00 EDT | "2024-11-03 01" | 第一个 01:00（夏令时） |
| 2 | 06:00Z | 01:00 EST | "2024-11-03 01" | **第二个 01:00（标准时）！** 字符串与上一行重复 |
| 3 | 07:00Z | 02:00 EST | "2024-11-03 02" | 恢复正常，进入标准时 |
| 4 | 08:00Z | 03:00 EST | "2024-11-03 03" | 正常 |
| ... | ... | ... | ... | ... |
| 24 | 04:00Z (11/04) | 23:00 EST (11/03) | "2024-11-03 23" | 当天最后一小时 |
```

**Fall back 的关键现象**：
1. `startOfDay` 仍然正确，00:00 在 DST 切换之前（02:00），仍为 EDT。
2. `addHours` 第 2 次迭代时进入重复小时，出现**两个 "01:00" 的本地时间**（先 EDT 后 EST），format 后字符串完全相同。
3. 最终 timeseries 数组中有 **25 个元素**（比普通日多 1 个），但其中两个字符串都是 `"2024-11-03 01"`。
4. **与 SQL 的差异**：SQL `date_trunc` 把这一小时内的所有事件统一归到一个 `01:00` 桶中（即一个桶包含两个物理小时的数据量）；而前端 timeseries 有两个相同的字符串键，lookup Map 中后一个会覆盖前一个，最终视觉上仍然只看到一个 01:00 桶。
5. **实际影响**：由于 SQL 返回的 01:00 桶数据量是两倍，而前端只显示一个桶，数值上是正确的；但 timeseries 数组长度比预期多 1（25 vs 24），且有重复标签——不过图表库通常会合并相同的 x 值，用户感知不明显。
6. DST 回拨也是 date-fns 原生函数内部自动处理的，业务代码无需特殊判断。

**Spring forward vs Fall back 对照表**：

| 现象 | Spring forward（3/10） | Fall back（11/03） |
|------|----------------------|-------------------|
| 本地小时数 | 23 个（缺 02:00） | 25 个物理小时，但只有 24 个不同的小时字符串（01:00 重复） |
| timeseries 长度 | 23 个元素 | 25 个元素（2 个重复字符串） |
| SQL 返回桶数 | 23 个桶 | 24 个桶（01:00 合并为一桶） |
| 前后端一致性 | 完全一致 | 字符串重复但数值正确，图表显示基本一致 |
| date-fns 处理方式 | 自动跳过 | 自动保留两个物理小时但格式化后字符串相同 |

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
    const parts = format.formatToParts(date);
    // ... 提取 parts 并拼接
    return { ...item, [column]: `${year}-${month}-${day} ${hour}:${minute}:${second}` };
  });
};
```

- 实时图表不走 SQL 桶对齐，而是在前端用 `Intl.DateTimeFormat`（**明确指定了 `timeZone` 参数**）按时区格式化时间戳。
- 这种方式不依赖浏览器时区，无论浏览器时区是什么，都能**正确**将 UTC 时间戳转为用户选时区的本地时间字符串。
- 这是因为实时数据本身就是分钟粒度的时间点，不需要 SQL 聚合，前端直接处理更方便也更正确。

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
| **时区规范化（前端）** | `canonicalizeTimezone` + `TIMEZONE_LEGACY`（18 条） | [`useTimezone.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useTimezone.ts#L79-L81) |
| **时区规范化（服务端）** | `normalizeTimezone` + `TIMEZONE_MAPPINGS`（1 条） | [`date.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/date.ts#L112-L114) |
| **前端填充** | `generateTimeSeries` + date-fns 原生函数（依赖浏览器时区） | [`date.ts`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/lib/date.ts#L349-L377) |
| **实时** | 前端 `Intl.DateTimeFormat(timeZone)` 明确指定时区，不依赖浏览器时区 | [`useTimezone.ts formatSeriesTimezone`](file:///d:/fz/0601-2/solo-dogfeeding/code/4-umami/src/components/hooks/useTimezone.ts#L31-L61) |

### 核心设计要点

1. **写入无时区**：原始事件只存 UTC 时间戳，不做任何桶对齐或时区转换，保留最大灵活性。
2. **查询时对齐**：桶对齐完全在查询时通过 SQL `date_trunc(field, unit, timezone)` 完成，属于"读时计算"模式。
3. **预聚合与桶解耦**：`website_event_stats_hourly` 按 UTC 小时桶预聚合。整数偏移时区下二次截断正确；**非整数偏移时区（如 +5:30）下，hour 桶全部错位，day 桶也会因跨日界的那一个 UTC 小时桶出现归属偏差**（可用 EVENT_COLUMNS 触发回源原始表来消除误差）。
4. **多种非整数偏移模式**：半小时偏移（Kolkata +5:30、Tehran +3:30/+4:30、Marquesas -9:30）的桶内分布为 30/30 均分；45 分钟偏移（Kathmandu +5:45）为 15/45 偏斜分布。Tehran 有 DST，一年中错位模式变化两次。负偏移（Marquesas）错位方向相反但本质相同。
5. **ClickHouse toDateTime 第二参数**：仅给 DateTime 值加时区元数据标记，用于结果序列化时按目标时区输出字符串，不改变内部 UTC 时间戳。PostgreSQL 用 `to_char` 直接格式化达到同样目的。
6. **前端日期边界的正确性前提**：`parseDateRange` + `localToUtc` 整条 date-fns 链路只有在**浏览器系统时区 = 用户选的显示时区**时才计算出正确的 startAt/endAt UTC 时间戳，否则系统性偏移（偏移量 = 用户时区偏移 − 浏览器时区偏移）。这是 date-fns 只能基于浏览器时区操作这一技术约束下的折衷设计。
7. **已知 Bug 与修复状态**："浏览器时区 ≠ 设置时区导致日期范围错误"是上游 umami 仓库的已知 Bug（issue #4107，v3.0.3）。PR #4112（v3.1.0）修复了"调用 `useDateRange` 时忘记传 timezone"的表层问题，但**浏览器时区 ≠ 用户选时区时的系统性偏移这一更根本的设计限制仍然存在**。本仓库（commit `c0ea3ae`）已包含 PR #4112 修复。
8. **`localToUtc` vs `toUtc` 的设计选择**：`useDateParameters` 选择 `localToUtc`（基于浏览器时区）而非 `toUtc`（基于用户选时区），是为了与 `parseDateRange` 中 `utcToZonedTime`（用户时区偏转）+ date-fns 原生函数（浏览器时区操作）形成"偏转-操作-反向偏转"的闭环。**在当前架构下 `localToUtc` 是唯一合理选择**——它保证了日期分量（年月日）的正确性，仅在精确时间戳上有系统性偏移；若改用 `toUtc` 会导致日期语义错乱（差一天等更严重问题）。
9. **两条时间过滤路径**：主统计查询直接写死 UTC BETWEEN；事件列表/渠道/目标等查询通过 `getDateQuery` 生成时间条件，ClickHouse 版带 `toTimezone`。
10. **两条规范化通道**：前端 `canonicalizeTimezone` 用 `TIMEZONE_LEGACY`（18 条），服务端 `normalizeTimezone` 用 `TIMEZONE_MAPPINGS`（仅 1 条），前后端独立维护。
11. **EVENT_COLUMNS 触发回源**：涉及 15 个事件级字段过滤或分钟粒度时，必须回源原始表，无法走预聚合。这也是非整数偏移时区获得准确桶归属的唯一可靠路径。
12. **周热力图双重错位**：周流量热力图走 hourly 预聚合表时，非整数偏移时区会导致 **hour 维度和 dow 维度的双重错位**——每个 UTC 小时桶横跨两个本地小时（hour 错位），且每天有 1 个桶跨日界导致 dow 也错。走原始表时完全正确。
13. **DST 透明处理**：`date_trunc` 和 date-fns `addHours`/`startOfDay` 都天然处理 DST，桶的定义按"日历日/小时"而非固定时长。
    - **Spring forward**（3/10）：`startOfDay` 正确（00:00 在切换前），`addHours` 自动跳过 02:00，生成 23 个小时桶，与 SQL 完全一致。
    - **Fall back**（11/03）：`startOfDay` 正确，`addHours` 产生两个物理 01:00 小时但 format 后字符串相同，timeseries 有 25 个元素（2 个重复标签），SQL 返回 24 个桶（重复小时合并），数值正确但图表标签有细微差异。
14. **空桶前端补齐**：SQL 只返回有数据的桶，空桶由 `generateTimeSeries` 在前端填充，但 lookup 匹配的正确性同样依赖"浏览器时区 = 用户选时区"这一前提。
