# 目标报表查询链路边界代码证据分析

## 一、Prisma (PostgreSQL) 查询路径逐段分析

### 1.1 完整调用链路

```
POST /api/reports/goal (route.ts:7-26)
    ↓
getGoal(websiteId, parameters, filters) (getGoal.ts:14-21)
    ↓
runQuery({ PRISMA, CLICKHOUSE }) (db.ts:22-36)
    ↓
relationalQuery(websiteId, parameters, filters) (getGoal.ts:23-77)
```

### 1.2 relationalQuery 逐段代码标注

**代码位置**：`src/queries/sql/reports/getGoal.ts:23-77`

#### 第 28-31 行：参数初始化
```typescript
const { startDate, endDate, type, value } = parameters;
const { rawQuery, parseFilters } = prisma;
const eventType = type === 'path' ? EVENT_TYPE.pageView : EVENT_TYPE.customEvent;  // 1=pageView, 2=customEvent
const column = type === 'path' ? 'url_path' : 'event_name';
```
- **证据**：`getGoal.ts:30-31`
- **说明**：根据目标类型设置 eventType 和匹配字段

#### 第 33-38 行：通配符处理
```typescript
let operator = '=';
let paramValue = value;
if (value.startsWith('*') || value.endsWith('*')) {
  operator = 'like';
  paramValue = value.replace(/^\*|\*$/g, '%');
}
```
- **证据**：`getGoal.ts:33-38`
- **说明**：将 `*` 通配符转换为 SQL `LIKE` 语法

#### 第 40-47 行：调用 parseFilters 生成过滤组件
```typescript
const { filterQuery, dateQuery, joinSessionQuery, cohortQuery, queryParams } = parseFilters({
  ...filters,           // 外部过滤条件
  websiteId,            // 网站ID
  value: paramValue,    // 目标匹配值（已转换通配符）
  startDate,            // 开始日期
  endDate,              // 结束日期
  eventType,            // 事件类型（1或2）
});
```
- **证据**：`getGoal.ts:40-47`
- **说明**：parseFilters 返回 5 个关键组件

#### 第 49-53 行：构造 excludeEventTypeFilterQuery
```typescript
const excludeEventTypeFilterQuery = filterQuery
  .split('\n')                      // 按行拆分
  .filter(filter => !filter.includes('event_type'))  // 排除包含 event_type 的行
  .join('\n')
  .trim();
```
- **证据**：`getGoal.ts:49-53`
- **说明**：从 filterQuery 中移除 event_type 相关的过滤条件行

#### 第 55-76 行：执行 SQL 查询

**num（转化数）查询条件**（第 67-73 行）：
```sql
from website_event
${cohortQuery}           -- 继承：群组过滤 JOIN
${joinSessionQuery}      -- 继承：session 表 JOIN（自动判断）
where website_event.website_id = {{websiteId::uuid}}
  and ${column} ${operator} {{value}}  -- 特有：目标匹配条件
  ${dateQuery}           -- 继承：日期范围
  ${filterQuery}         -- 继承：完整过滤条件（含 event_type）
```
- **证据**：`getGoal.ts:67-73`

**total（总访客数）查询条件**（第 58-65 行）：
```sql
(
  select count(distinct website_event.session_id)
  from website_event
  ${cohortQuery}                     -- 继承：群组过滤 JOIN
  ${joinSessionQuery}                -- 继承：session 表 JOIN（自动判断）
  where website_event.website_id = {{websiteId::uuid}}
    ${dateQuery}                     -- 继承：日期范围
    ${excludeEventTypeFilterQuery}   -- 部分继承：过滤条件，但排除 event_type
) as total
```
- **证据**：`getGoal.ts:58-65`

### 1.3 Prisma 路径 total 过滤条件继承表

| 过滤条件 | num（转化数） | total（总访客数） | 代码行号 |
|---------|--------------|-----------------|----------|
| `website_id` | ✅ 继承 | ✅ 继承 | 63, 70 |
| 目标匹配（`url_path`/`event_name`） | ✅ 特有 | ❌ 无 | 71 |
| `dateQuery`（日期范围） | ✅ 继承 | ✅ 继承 | 64, 72 |
| `filterQuery`（含 event_type） | ✅ 继承 | ❌ 排除 | 65, 73 |
| `excludeEventTypeFilterQuery`（除 event_type 外） | ❌ 无 | ✅ 继承 | 65 |
| `cohortQuery`（群组过滤） | ✅ 继承 | ✅ 继承 | 61, 68 |
| `joinSessionQuery`（session JOIN） | ✅ 继承 | ✅ 继承 | 62, 69 |

### 1.4 joinSessionQuery 的自动判断逻辑

**代码位置**：`src/lib/prisma.ts:232-246`

```typescript
function parseFilters(filters: Record<string, any>, options?: QueryOptions) {
  const joinSession = Object.keys(filters).find(key => {
    const baseName = key.replace(/\d+$/, '');
    return ['referrer', ...SESSION_COLUMNS].includes(baseName);
  });

  return {
    joinSessionQuery:
      options?.joinSession || joinSession
        ? `inner join session on website_event.session_id = session.session_id and website_event.website_id = session.website_id`
        : '',
    // ...
  };
}
```

**SESSION_COLUMNS**（`src/lib/constants.ts:55-65`）：
```typescript
export const SESSION_COLUMNS = [
  'browser', 'os', 'device', 'screen', 'language',
  'country', 'city', 'region', 'distinctId',
];
```

**触发 JOIN 的条件**：
- 过滤条件包含 `referrer`
- 或过滤条件包含任何 SESSION_COLUMNS 字段（browser、os、device、screen、language、country、city、region、distinctId）

---

## 二、ClickHouse 查询路径逐段分析

### 2.1 关键架构差异

**核心发现**：ClickHouse 的 `website_event` 表是**宽表设计**，所有 session 字段（browser、os、country 等）直接存储在 website_event 表中，**不需要 JOIN session 表**。

**证据**：`db/clickhouse/schema.sql:2-55`
```sql
CREATE TABLE umami.website_event
(
    website_id UUID,
    session_id UUID,
    visit_id UUID,
    event_id UUID,
    -- sessions 字段直接存储在 website_event 表中
    hostname LowCardinality(String),
    browser LowCardinality(String),
    os LowCardinality(String),
    device LowCardinality(String),
    screen LowCardinality(String),
    language LowCardinality(String),
    country LowCardinality(String),
    region LowCardinality(String),
    city String,
    -- ... 其他字段
)
```

### 2.2 clickhouseQuery 逐段代码标注

**代码位置**：`src/queries/sql/reports/getGoal.ts:79-131`

#### 第 84-94 行：参数初始化（与 Prisma 相同）
```typescript
const { startDate, endDate, type, value } = parameters;
const { rawQuery, parseFilters } = clickhouse;
const eventType = type === 'path' ? EVENT_TYPE.pageView : EVENT_TYPE.customEvent;
const column = type === 'path' ? 'url_path' : 'event_name';

let operator = '=';
let paramValue = value;
if (value.startsWith('*') || value.endsWith('*')) {
  operator = 'like';
  paramValue = value.replace(/^\*|\*$/g, '%');
}
```
- **证据**：`getGoal.ts:84-94`

#### 第 96-103 行：调用 parseFilters（注意返回值差异）
```typescript
const { filterQuery, dateQuery, cohortQuery, queryParams } = parseFilters({
  ...filters,
  websiteId,
  value: paramValue,
  startDate,
  endDate,
  eventType,
});
```
- **证据**：`getGoal.ts:96-103`
- **关键差异**：ClickHouse 版本的 parseFilters **不返回** `joinSessionQuery`

#### 第 105-109 行：构造 excludeEventTypeFilterQuery（与 Prisma 相同）
```typescript
const excludeEventTypeFilterQuery = filterQuery
  .split('\n')
  .filter(filter => !filter.includes('event_type'))
  .join('\n')
  .trim();
```
- **证据**：`getGoal.ts:105-109`

#### 第 111-130 行：执行 SQL 查询

**num（转化数）查询条件**（第 122-127 行）：
```sql
from website_event
${cohortQuery}           -- 继承：群组过滤 JOIN
-- 注意：没有 joinSessionQuery！ClickHouse 不需要 JOIN
where website_id = {websiteId:UUID}
  and ${column} ${operator} {value:String}  -- 特有：目标匹配条件
  ${dateQuery}           -- 继承：日期范围
  ${filterQuery}         -- 继承：完整过滤条件（含 event_type）
```
- **证据**：`getGoal.ts:122-127`

**total（总访客数）查询条件**（第 114-120 行）：
```sql
(
  select count(distinct session_id)
  from website_event
  ${cohortQuery}                     -- 继承：群组过滤 JOIN
  -- 注意：没有 joinSessionQuery！
  where website_id = {websiteId:UUID}
    ${dateQuery}                     -- 继承：日期范围
    ${excludeEventTypeFilterQuery}   -- 部分继承：过滤条件，但排除 event_type
) as total
```
- **证据**：`getGoal.ts:114-120`

### 2.3 ClickHouse parseFilters 无 joinSessionQuery 的证据

**代码位置**：`src/lib/clickhouse.ts:224-236`

```typescript
function parseFilters(filters: Record<string, any>, options?: QueryOptions) {
  const cohortFilters = Object.fromEntries(
    Object.entries(filters).filter(([key]) => key.startsWith('cohort_')),
  );

  return {
    filterQuery: getFilterQuery(filters, options),
    dateQuery: getDateQuery(filters),
    queryParams: getQueryParams(filters),
    cohortQuery: getCohortQuery(cohortFilters),
    excludeBounceQuery: getExcludeBounceQuery(filters),
    // 注意：没有 joinSessionQuery！
  };
}
```
- **证据**：`src/lib/clickhouse.ts:224-236`

**Grep 验证**：在 `src/lib/clickhouse.ts` 中搜索 `joinSessionQuery`，无匹配结果。

### 2.4 ClickHouse 路径 total 过滤条件继承表

| 过滤条件 | num（转化数） | total（总访客数） | 代码行号 |
|---------|--------------|-----------------|----------|
| `website_id` | ✅ 继承 | ✅ 继承 | 118, 124 |
| 目标匹配（`url_path`/`event_name`） | ✅ 特有 | ❌ 无 | 125 |
| `dateQuery`（日期范围） | ✅ 继承 | ✅ 继承 | 119, 126 |
| `filterQuery`（含 event_type） | ✅ 继承 | ❌ 排除 | 120, 127 |
| `excludeEventTypeFilterQuery`（除 event_type 外） | ❌ 无 | ✅ 继承 | 120 |
| `cohortQuery`（群组过滤） | ✅ 继承 | ✅ 继承 | 117, 123 |
| `joinSessionQuery`（session JOIN） | ❌ 无（架构差异） | ❌ 无（架构差异） | N/A |

---

## 三、Kafka 相关代码证据核对

### 3.1 Kafka 启用条件

**代码位置**：`src/lib/kafka.ts:14`

```typescript
const enabled = Boolean(process.env.KAFKA_URL && process.env.KAFKA_BROKER);
```

**需要同时配置**：
- `KAFKA_URL`
- `KAFKA_BROKER`

### 3.2 runQuery 中的 KAFKA 选项

**代码位置**：`src/lib/db.ts:22-36`

```typescript
export async function runQuery(queries: any) {
  if (process.env.CLICKHOUSE_URL) {
    if (queries[KAFKA]) {           // 检查是否提供了 KAFKA 实现
      return queries[KAFKA]();       // 如果有，执行 KAFKA 实现
    }
    return queries[CLICKHOUSE]();    // 否则执行 ClickHouse 实现
  }
  // ... PostgreSQL 逻辑
}
```

**关键发现**：`runQuery` 支持 KAFKA 选项，但**整个 `src/queries` 目录下没有任何查询函数提供 `[KAFKA]` 实现**。

**Grep 验证**：在 `src/queries` 目录下搜索 `[KAFKA]`，无匹配结果。

### 3.3 可直接证明：Kafka 仅在 4 个写入函数中使用

通过代码搜索，Kafka 仅在以下 4 个写入函数的 `clickhouseQuery` 内部使用：

| 写入函数 | 代码位置 | Kafka 主题 | 证据行号 |
|---------|----------|-----------|----------|
| `saveEvent` | `src/queries/sql/events/saveEvent.ts` | `event` | 259-263 |
| `saveEventData` | `src/queries/sql/events/saveEventData.ts` | `event_data` | 74-78 |
| `saveSessionData` | `src/queries/sql/sessions/saveSessionData.ts` | `session_data` | 99-103 |
| `saveRecording` | `src/queries/sql/replays/saveRecording.ts` | `session_replay` | 78-82 |

**典型实现模式**（以 saveEvent 为例）：

**代码位置**：`src/queries/sql/events/saveEvent.ts:65-70`（runQuery 调用）
```typescript
export async function saveEvent(args: SaveEventArgs) {
  return runQuery({
    [PRISMA]: () => relationalQuery(args),
    [CLICKHOUSE]: () => clickhouseQuery(args),
    // 注意：这里没有 KAFKA 选项！
  });
}
```

**代码位置**：`src/queries/sql/events/saveEvent.ts:259-263`（clickhouseQuery 内部 Kafka 分流）
```typescript
if (kafka.enabled) {
  await sendMessage('event', message);    // Kafka 启用：发送到 Kafka
} else {
  await insert('website_event', [message]);  // Kafka 未启用：直接写入 ClickHouse
}
```

### 3.4 可直接证明 vs 推断范围

#### ✅ 可直接证明（有代码证据）

1. **Kafka 启用条件**：需要同时配置 `KAFKA_URL` 和 `KAFKA_BROKER`
   - 证据：`src/lib/kafka.ts:14`

2. **Kafka 仅在写入链路使用**：查询链路完全不使用 Kafka
   - 证据：`src/queries` 目录下所有 `get*` 函数的 runQuery 调用都没有 `[KAFKA]` 选项

3. **Kafka 分流发生在 clickhouseQuery 内部**：不是通过 runQuery 的 KAFKA 选项
   - 证据：`saveEvent.ts:65-70` 显示 runQuery 调用没有 KAFKA 选项；`saveEvent.ts:259-263` 显示 Kafka 分流在 clickhouseQuery 内部通过 if-else 实现

4. **4 个写入函数使用 Kafka**：saveEvent、saveEventData、saveSessionData、saveRecording
   - 证据：Grep 搜索 `kafka` 在 `src/queries/sql` 目录下返回这 4 个文件

5. **4 个 Kafka 主题**：event、event_data、session_data、session_replay
   - 证据：各写入函数中的 `sendMessage` 调用

6. **runQuery 的 KAFKA 选项存在但未被使用**：代码中有这个分支，但没有查询/写入函数提供 KAFKA 实现
   - 证据：`src/lib/db.ts:24-25` 有 KAFKA 分支逻辑，但 `src/queries` 目录下没有 `[KAFKA]` 实现

#### ⚠️ 推断范围（无直接代码证据）

1. **Kafka 消费者实现**：将消息从 Kafka 主题写入 ClickHouse 的消费者代码不在当前代码库中
   - 推断依据：代码库中只有生产者逻辑（sendMessage），没有消费者逻辑

2. **Kafka 主题配置**：主题的分区数、副本数、保留策略等配置不在当前代码库中
   - 推断依据：代码中只使用主题名称（'event'、'event_data' 等），没有配置信息

3. **Kafka 消费者组配置**：消费者组 ID、offset 重置策略等不在当前代码库中
   - 推断依据：代码中没有相关配置

4. **Kafka 错误处理**：消息发送失败后的重试、死信队列等逻辑
   - 推断依据：`sendMessage` 函数只有基本的 try-catch 和错误日志，没有重试逻辑

### 3.5 Kafka 完整架构图（含证明/推断标注）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              写入链路                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  POST /api/send                                                         │
│       ↓                                                                 │
│  saveEvent()  [可证明]                                                  │
│       ↓                                                                 │
│  runQuery({ PRISMA, CLICKHOUSE })  [可证明：无 KAFKA 选项]              │
│       │                                                                 │
│       ├─ PRISMA → 直接写入 PostgreSQL  [可证明]                         │
│       │                                                                 │
│       └─ CLICKHOUSE → clickhouseQuery()  [可证明]                       │
│                      ↓                                                 │
│                      kafka.enabled?  [可证明]                           │
│                        ├─ true → sendMessage('event', message)  [可证明]│
│                        │              ↓                                 │
│                        │         Kafka 主题 'event'                     │
│                        │              ↓                                 │
│                        │         Kafka 消费者 → ClickHouse  [推断]      │
│                        │                                                 │
│                        └─ false → insert('website_event', [message])    │
│                                          [可证明]                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                              查询链路                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  POST /api/reports/goal  [可证明]                                       │
│       ↓                                                                 │
│  getGoal()  [可证明]                                                    │
│       ↓                                                                 │
│  runQuery({ PRISMA, CLICKHOUSE })  [可证明：无 KAFKA 选项]              │
│       │                                                                 │
│       ├─ CLICKHOUSE_URL 已配置 → clickhouseQuery() → 查 ClickHouse      │
│       │                          [可证明]                               │
│       │                                                                 │
│       └─ DATABASE_URL 是 postgres → relationalQuery() → 查 PostgreSQL   │
│                                  [可证明]                               │
│                                                                         │
│  ✅ 可证明：查询链路完全不经过 Kafka                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 四、事件命中数与转化计数的可复核对照

### 4.1 概念定义

| 指标 | 定义 | SQL 聚合方式 |
|------|------|-------------|
| **事件命中数** | 满足目标匹配条件的原始事件记录条数 | `count(*)` |
| **转化计数** | 满足目标匹配条件的独立会话数（去重后） | `count(distinct session_id)` |

### 4.2 过滤条件继承对照表

假设我们要同时获取两个指标，SQL 结构如下：

```sql
select 
  count(*) as event_hits,           -- 事件命中数
  count(distinct session_id) as conversions,  -- 转化计数
  (select count(distinct session_id) from ...) as total
from website_event
where ...  -- 相同的过滤条件
```

**逐条件对照**：

| 过滤条件 | 事件命中数 (`count(*)`) | 转化计数 (`count(distinct session_id)`) | 是否一致 | 代码证据 |
|---------|------------------------|----------------------------------------|----------|----------|
| `website_id` | ✅ 需要 | ✅ 需要 | 一致 | 隐含在查询中 |
| 目标匹配（`url_path`/`event_name`） | ✅ 需要 | ✅ 需要 | 一致 | `getGoal.ts:71, 125` |
| `dateQuery`（日期范围） | ✅ 需要 | ✅ 需要 | 一致 | `getGoal.ts:72, 126` |
| `event_type` 过滤 | ✅ 需要 | ✅ 需要 | 一致 | 通过 `filterQuery` 继承 |
| 维度过滤（browser、country 等） | ✅ 需要 | ✅ 需要 | 一致 | 通过 `filterQuery` 继承 |
| `cohortQuery`（群组过滤） | ✅ 需要 | ✅ 需要 | 一致 | `getGoal.ts:68, 123` |
| `joinSessionQuery`（Prisma 特有） | ✅ 需要 | ✅ 需要 | 一致 | `getGoal.ts:69` |
| SELECT 聚合函数 | `count(*)` | `count(distinct session_id)` | 不一致 | `getGoal.ts:57, 113` |

### 4.3 数值关系验证

**定理**：事件命中数 ≥ 转化计数

**证明**：
- 事件命中数 = `count(*)` = 满足条件的行数
- 转化计数 = `count(distinct session_id)` = 满足条件的不同 session_id 的数量
- 因为一个 session_id 可能对应多行（同一用户多次触发目标），所以行数 ≥ 不同 session_id 的数量

**示例验证**：

| 场景 | 事件命中数 | 转化计数 | 关系 |
|------|-----------|----------|------|
| 1 个用户触发目标 1 次 | 1 | 1 | 相等 |
| 1 个用户触发目标 3 次 | 3 | 1 | 命中 > 转化 |
| 2 个用户各触发目标 1 次 | 2 | 2 | 相等 |
| 2 个用户各触发目标 2 次 | 4 | 2 | 命中 > 转化 |

### 4.4 Umami 实际返回值验证

**代码证据**：`src/queries/sql/reports/getGoal.ts:57, 113`

```typescript
// Prisma 版本
select count(distinct website_event.session_id) as num, ...

// ClickHouse 版本
select count(distinct session_id) as num, ...
```

**结论**：Umami 目标报表返回的 `num` 是**转化计数**（`count(distinct session_id)`），**不是**事件命中数（`count(*)`）。

**前端展示证据**：`src/app/(main)/websites/[websiteId]/(reports)/goals/Goal.tsx:71-84`

```typescript
<Text title={`${data?.num} / ${data?.total}`}>
  {`${formatLongNumber(data?.num)} / ${formatLongNumber(data?.total)}`}
</Text>
// ...
<Text weight="bold" size="4xl">
  {data?.total ? Math.round((+data?.num / +data?.total) * 100) : '0'}%
</Text>
```

**说明**：前端展示 `num / total`（转化数 / 总访客数）和转化率，完全基于会话级别的统计。

---

## 五、证据清单

### 5.1 Prisma vs ClickHouse 过滤条件差异证据

| 编号 | 证据内容 | 文件路径 | 行号 |
|------|----------|----------|------|
| E1 | Prisma parseFilters 返回 joinSessionQuery | `src/lib/prisma.ts` | 232-246 |
| E2 | ClickHouse parseFilters 不返回 joinSessionQuery | `src/lib/clickhouse.ts` | 224-236 |
| E3 | ClickHouse website_event 表包含所有 session 字段（宽表设计） | `db/clickhouse/schema.sql` | 2-55 |
| E4 | Prisma 版本 total 子查询包含 joinSessionQuery | `src/queries/sql/reports/getGoal.ts` | 62 |
| E5 | ClickHouse 版本 total 子查询不包含 joinSessionQuery | `src/queries/sql/reports/getGoal.ts` | 117 |
| E6 | SESSION_COLUMNS 定义 | `src/lib/constants.ts` | 55-65 |
| E7 | excludeEventTypeFilterQuery 实现（Prisma） | `src/queries/sql/reports/getGoal.ts` | 49-53 |
| E8 | excludeEventTypeFilterQuery 实现（ClickHouse） | `src/queries/sql/reports/getGoal.ts` | 105-109 |

### 5.2 Kafka 相关证据

| 编号 | 证据内容 | 文件路径 | 行号 |
|------|----------|----------|------|
| E9 | Kafka 启用条件 | `src/lib/kafka.ts` | 14 |
| E10 | runQuery 支持 KAFKA 选项但未被使用 | `src/lib/db.ts` | 24-25 |
| E11 | saveEvent 的 runQuery 调用无 KAFKA 选项 | `src/queries/sql/events/saveEvent.ts` | 65-70 |
| E12 | saveEvent 的 clickhouseQuery 内部 Kafka 分流 | `src/queries/sql/events/saveEvent.ts` | 259-263 |
| E13 | saveEventData 的 Kafka 分流 | `src/queries/sql/events/saveEventData.ts` | 74-78 |
| E14 | saveSessionData 的 Kafka 分流 | `src/queries/sql/sessions/saveSessionData.ts` | 99-103 |
| E15 | saveRecording 的 Kafka 分流 | `src/queries/sql/replays/saveRecording.ts` | 78-82 |
| E16 | 查询层无 KAFKA 实现（Grep 验证） | `src/queries` 目录 | N/A |

### 5.3 事件命中与转化计数证据

| 编号 | 证据内容 | 文件路径 | 行号 |
|------|----------|----------|------|
| E17 | Umami 返回 count(distinct session_id) 作为 num | `src/queries/sql/reports/getGoal.ts` | 57, 113 |
| E18 | 前端展示 num / total 和转化率 | `src/app/(main)/websites/[websiteId]/(reports)/goals/Goal.tsx` | 71-84 |
| E19 | filterQuery 生成逻辑（Prisma） | `src/lib/prisma.ts` | 108-150 |
| E20 | filterQuery 生成逻辑（ClickHouse） | `src/lib/clickhouse.ts` | 101-140 |

---

## 六、仍待确认项

| 编号 | 待确认内容 | 确认方式 | 优先级 |
|------|----------|----------|--------|
| T1 | Kafka 消费者的具体实现（如何将消息从 Kafka 写入 ClickHouse） | 查看部署配置或外部消费者代码 | 高 |
| T2 | Kafka 主题的详细配置（分区数、副本数、保留策略） | 查看 Kafka 集群配置 | 中 |
| T3 | Kafka 消费者组配置（group.id、offset 重置策略） | 查看消费者代码或配置 | 中 |
| T4 | 消息发送失败后的重试和死信队列策略 | 查看 Kafka 生产者配置和监控 | 中 |
| T5 | 宽表设计的性能考量（为什么 ClickHouse 用宽表，PostgreSQL 用 JOIN） | 查看架构设计文档或与开发团队确认 | 低 |
| T6 | ClickHouse 版本为什么没有 joinSessionQuery（是遗漏还是刻意设计） | 查看 Git 历史或与开发团队确认 | 低 |
| T7 | 是否有计划在查询层支持 Kafka（目前代码中有 KAFKA 分支但未使用） | 查看 roadmap 或与开发团队确认 | 低 |

---

## 七、关键发现总结

### 发现 1：Prisma 与 ClickHouse 的架构差异
- **Prisma (PostgreSQL)**：使用规范化设计，session 字段存储在独立的 session 表中，通过 joinSessionQuery 自动判断是否需要 JOIN
- **ClickHouse**：使用宽表设计，所有 session 字段直接存储在 website_event 表中，不需要 JOIN session 表
- **影响**：total 统计的过滤条件继承在两种数据库中略有不同（Prisma 有 joinSessionQuery，ClickHouse 没有）

### 发现 2：Kafka 的角色边界
- **可证明**：Kafka 仅在 4 个写入函数的 clickhouseQuery 内部使用，作为 ClickHouse 写入的可选缓冲层
- **可证明**：查询链路完全不经过 Kafka
- **可证明**：runQuery 的 KAFKA 选项存在但未被任何查询/写入函数使用
- **推断**：Kafka 消费者实现不在当前代码库中

### 发现 3：total 统计的精确过滤条件
- **继承**：website_id、dateQuery、cohortQuery、joinSessionQuery（仅 Prisma）、除 event_type 外的所有维度过滤
- **排除**：event_type 过滤、目标匹配条件
- **实现方式**：通过字符串拆分和过滤，精确移除 filterQuery 中包含 event_type 的行

### 发现 4：事件命中与转化计数的关系
- **过滤条件**：完全相同
- **差异**：仅在 SELECT 子句的聚合方式（`count(*)` vs `count(distinct session_id)`）
- **数值关系**：事件命中数 ≥ 转化计数
- **Umami 实际返回**：转化计数（`count(distinct session_id)`），不是事件命中数
