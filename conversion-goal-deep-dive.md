# 目标转化代码深度解析

## 一、total 统计的过滤条件继承与排除

### 1.1 完整的过滤条件拆解

在 `getGoal.ts` 的 SQL 查询中，`num` 和 `total` 两个统计值的过滤条件存在细微但关键的差异。让我们从参数传入开始，完整追踪每个条件的去向。

**参数传入** (`getGoal.ts:40-47`)：
```typescript
const { filterQuery, dateQuery, joinSessionQuery, cohortQuery, queryParams } = parseFilters({
  ...filters,           // 外部传入的过滤条件（浏览器、国家、来源等）
  websiteId,
  value: paramValue,    // 目标匹配值（如 '/checkout/*'）
  startDate,
  endDate,
  eventType,            // 根据目标类型设置：1=pageView, 2=customEvent
});
```

**parseFilters 返回的 5 个关键组件**：

| 组件 | 含义 |
|------|------|
| `filterQuery` | 维度过滤条件（event_type、browser、country、referrer 等） |
| `dateQuery` | 日期范围过滤（`created_at between ...`） |
| `joinSessionQuery` | 是否需要 JOIN session 表 |
| `cohortQuery` | 群组/队列过滤（JOIN 子查询） |
| `queryParams` | 参数绑定对象（防 SQL 注入） |

### 1.2 num（转化数）的完整过滤条件

**外层查询** (`getGoal.ts:67-73`)：
```sql
from website_event
${cohortQuery}           -- 继承：群组过滤
${joinSessionQuery}      -- 继承：session 表 JOIN
where website_event.website_id = {{websiteId::uuid}}
  and ${column} ${operator} {{value}}  -- 特有：目标匹配条件
  ${dateQuery}           -- 继承：日期过滤
  ${filterQuery}         -- 继承：完整的过滤条件（含 event_type）
```

### 1.3 total（总访客数）的过滤条件

**子查询** (`getGoal.ts:58-65`)：
```sql
(
  select count(distinct website_event.session_id)
  from website_event
  ${cohortQuery}                     -- 继承：群组过滤
  ${joinSessionQuery}                -- 继承：session 表 JOIN
  where website_event.website_id = {{websiteId::uuid}}
    ${dateQuery}                     -- 继承：日期过滤
    ${excludeEventTypeFilterQuery}   -- 部分继承：过滤条件，但排除 event_type
) as total
```

### 1.4 excludeEventTypeFilterQuery 的实现细节

**代码** (`getGoal.ts:49-53`)：
```typescript
const excludeEventTypeFilterQuery = filterQuery
  .split('\n')                     // 按行拆分
  .filter(filter => !filter.includes('event_type'))  // 排除包含 event_type 的行
  .join('\n')
  .trim();
```

**为什么要排除 event_type？**

假设我们有一个 `path` 类型目标（匹配 `/checkout/*`），此时 `eventType = 1`（pageView）。

如果 `total` 也加上 `event_type = 1` 的限制：
- total 只会统计触发了 pageView 事件的访客
- 但有些访客可能只触发了 customEvent（如点击按钮），没有触发 pageView
- 这些访客会被排除在 total 之外，导致转化率计算偏高

**正确逻辑**：
- `num`：需要 `event_type` 过滤，因为目标类型决定了统计哪些事件
- `total`：不需要 `event_type` 过滤，因为总访客应该是所有类型事件的访客

### 1.5 过滤条件继承关系表

| 过滤条件 | num（转化数） | total（总访客数） | 备注 |
|---------|--------------|-----------------|------|
| `website_id` | ✅ 继承 | ✅ 继承 | 网站 ID，始终需要 |
| `startDate/endDate` | ✅ 继承 | ✅ 继承 | 日期范围，通过 `${dateQuery}` |
| `cohortQuery` | ✅ 继承 | ✅ 继承 | 群组过滤，通过 `${cohortQuery}` |
| `joinSessionQuery` | ✅ 继承 | ✅ 继承 | session 表 JOIN，根据过滤字段自动判断 |
| `event_type` | ✅ 继承 | ❌ 排除 | 通过 `excludeEventTypeFilterQuery` 排除 |
| 其他维度（browser、country、referrer 等） | ✅ 继承 | ✅ 继承 | 通过 `excludeEventTypeFilterQuery` 保留 |
| 目标匹配条件（`url_path`/`event_name`） | ✅ 特有 | ❌ 无 | 只在 num 查询中，通过 `${column} ${operator} {{value}}` |

### 1.6 joinSessionQuery 的自动判断

**代码** (`prisma.ts:232-246`)：
```typescript
function parseFilters(filters: Record<string, any>, options?: QueryOptions) {
  const joinSession = Object.keys(filters).find(key => {
    const baseName = key.replace(/\d+$/, '');
    return ['referrer', ...SESSION_COLUMNS].includes(baseName);
  });

  return {
    joinSessionQuery:
      options?.joinSession || joinSession
        ? `inner join session on website_event.session_id = session.session_id ...`
        : '',
    // ...
  };
}
```

**SESSION_COLUMNS** (`constants.ts:55-65`)：
```typescript
export const SESSION_COLUMNS = [
  'browser', 'os', 'device', 'screen', 'language',
  'country', 'city', 'region', 'distinctId',
];
```

**自动判断逻辑**：
- 如果过滤条件包含 `referrer` 或任何 SESSION_COLUMNS 中的字段 → 自动 JOIN session 表
- 否则 → 不需要 JOIN

**继承关系**：
- `joinSessionQuery` 在 num 和 total 查询中都会被继承
- 因为如果用户按"浏览器=Chrome"过滤，那么转化数和总访客数都应该只统计 Chrome 用户

---

## 二、Kafka 分流在写入链路的具体位置与 runQuery 职责边界

### 2.1 写入链路的完整调用栈

```
POST /api/send (route.ts)
    ↓
saveEvent(args) (saveEvent.ts:65-70)
    ↓
runQuery({ PRISMA, CLICKHOUSE })  ← 第一层选择：数据库类型
    │
    ├─ PRISMA → relationalQuery() → 直接写入 PostgreSQL
    │
    └─ CLICKHOUSE → clickhouseQuery()
                        ↓
                        ├─ 构造 message 对象
                        │
                        ├─ kafka.enabled?  ← 第二层选择：是否走 Kafka
                        │     ├─ true → sendMessage('event', message) → Kafka 主题
                        │     └─ false → insert('website_event', [message]) → 直接写 ClickHouse
                        │
                        └─ eventData 存在时 → saveEventData()
                                            ↓
                                            runQuery({ PRISMA, CLICKHOUSE })
                                                ↓
                                                clickhouseQuery()
                                                    ↓
                                                    kafka.enabled?
                                                        ├─ true → sendMessage('event_data', ...)
                                                        └─ false → insert('event_data', ...)
```

### 2.2 Kafka 分流的具体位置

**关键点**：Kafka 分流**不在 `runQuery` 层**，而是在 `clickhouseQuery` 函数**内部**。

**代码证据 1：saveEvent 的 runQuery 调用** (`saveEvent.ts:65-70`)：
```typescript
export async function saveEvent(args: SaveEventArgs) {
  return runQuery({
    [PRISMA]: () => relationalQuery(args),
    [CLICKHOUSE]: () => clickhouseQuery(args),
    // 注意：这里没有 KAFKA 选项！
  });
}
```

**代码证据 2：clickhouseQuery 内部的 Kafka 分流** (`saveEvent.ts:259-263`)：
```typescript
async function clickhouseQuery({ ... }: SaveEventArgs) {
  const { insert, getUTCString } = clickhouse;
  const { sendMessage } = kafka;
  // ... 构造 message ...

  if (kafka.enabled) {
    // Kafka 启用时走 Kafka
    await sendMessage('event', message);
  } else {
    // Kafka 未启用时直接写入 ClickHouse
    await insert('website_event', [message]);
  }
  // ...
}
```

**代码证据 3：saveEventData 同样的模式** (`saveEventData.ts:74-78`)：
```typescript
async function clickhouseQuery(data: SaveEventDataArgs) {
  // ... 构造 messages ...
  
  if (kafka.enabled) {
    await sendMessage('event_data', messages);
  } else {
    await insert('event_data', messages);
  }
}
```

### 2.3 runQuery 的真实职责

**`runQuery` 的完整逻辑** (`db.ts:22-36`)：
```typescript
export async function runQuery(queries: any) {
  // 优先级 1：如果配置了 CLICKHOUSE_URL
  if (process.env.CLICKHOUSE_URL) {
    // 检查是否有 Kafka 实现（注意：几乎没有查询函数提供这个选项）
    if (queries[KAFKA]) {
      return queries[KAFKA]();
    }
    // 使用 ClickHouse 查询
    return queries[CLICKHOUSE]();
  }

  // 优先级 2：检查 DATABASE_URL 类型
  const db = getDatabaseType();
  if (db === POSTGRESQL) {
    return queries[PRISMA]();
  }
}
```

**runQuery 的职责边界**：
- ✅ 选择使用 PostgreSQL 还是 ClickHouse
- ✅ 作为数据库抽象层，隔离上层业务代码与具体数据库
- ❌ **不负责** Kafka 分流决策（这是 clickhouseQuery 内部的事）
- ❌ **不负责** 写入链路的消息队列选择

### 2.4 Kafka 启用条件

**代码** (`kafka.ts:14`)：
```typescript
const enabled = Boolean(process.env.KAFKA_URL && process.env.KAFKA_BROKER);
```

**需要同时配置两个环境变量**：
- `KAFKA_URL`：Kafka 连接 URL（含认证信息）
- `KAFKA_BROKER`：Kafka Broker 地址列表

### 2.5 runQuery 与 Kafka 的界限总结

| 层级 | 决策逻辑 | 影响范围 |
|------|----------|----------|
| **runQuery 层** | 根据 `CLICKHOUSE_URL` 或 `DATABASE_URL` 选择数据库 | 所有查询和写入操作 |
| **clickhouseQuery 内部** | 根据 `kafka.enabled` 选择写入方式 | 仅写入链路（saveEvent、saveEventData 等） |

**架构意图**：
- `runQuery`：纯数据库路由，不关心消息队列
- Kafka：ClickHouse 写入时的可选缓冲层，用于高并发场景削峰
- PostgreSQL 写入：不经过 Kafka，直接写入

### 2.6 完整架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        runQuery 决策层                               │
│  职责：根据环境变量选择 PRISMA / CLICKHOUSE 数据库实现                │
└─────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      数据库实现层                                    │
│                                                                     │
│  ┌──────────────────────────┐   ┌───────────────────────────────┐  │
│  │  PRISMA (PostgreSQL)    │   │  CLICKHOUSE                   │  │
│  │  直接写入 PostgreSQL    │   │  内部包含 Kafka 分流逻辑       │  │
│  │  无 Kafka 选项          │   │  ┌─────────────────────────┐   │  │
│  │                         │   │  │ kafka.enabled?          │   │  │
│  │                         │   │  │   ├─ true → Kafka 主题   │   │  │
│  │                         │   │  │   └─ false → ClickHouse │   │  │
│  │                         │   │  └─────────────────────────┘   │  │
│  └──────────────────────────┘   └───────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      实际数据存储                                    │
│                                                                     │
│  PostgreSQL  ← 直接写入 ←┐                                          │
│                          │                                          │
│  ClickHouse  ← 直接写入 ←┤← Kafka 消费者 ← Kafka 主题 ← sendMessage  │
│                          │                                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.7 关键误区澄清

❌ **错误理解**：Kafka 是与 PostgreSQL、ClickHouse 同级的数据库选项，由 `runQuery` 统一调度。

✅ **正确理解**：
1. `runQuery` 只在 PostgreSQL 和 ClickHouse 之间选择
2. Kafka 是 ClickHouse 写入路径上的**可选缓冲层**，不是独立的数据库选项
3. Kafka 分流决策发生在 `clickhouseQuery` 函数内部，由 `kafka.enabled` 标志控制
4. PostgreSQL 写入路径永远不经过 Kafka

---

## 三、事件命中数与转化计数在过滤条件继承上的差异

### 3.1 概念再澄清

在深入分析之前，先明确两个容易混淆的概念：

| 概念 | 定义 | SQL 示例 |
|------|------|----------|
| **事件命中数** | 满足目标匹配条件的原始事件记录条数 | `count(*)` |
| **转化计数** | 满足目标匹配条件的独立会话数（去重后） | `count(distinct session_id)` |

**数值关系**：事件命中数 ≥ 转化计数

### 3.2 Umami 中实际返回的是哪个？

**代码证据** (`getGoal.ts:57,113`)：
```typescript
// PostgreSQL
select count(distinct website_event.session_id) as num, ...

// ClickHouse  
select count(distinct session_id) as num, ...
```

**结论**：Umami 目标报表返回的 `num` 是**转化计数**（去重后的会话数），不是事件命中数。

### 3.3 过滤条件继承差异

虽然 Umami 不直接返回事件命中数，但通过代码分析可以推导出两者在过滤条件继承上的差异。

**假设我们需要同时获取两个指标**，SQL 会是这样：

```sql
select 
  count(*) as event_hits,           -- 事件命中数
  count(distinct session_id) as conversions,  -- 转化计数
  (select count(distinct session_id) from ...) as total
from website_event
where ...  -- 相同的过滤条件
```

**关键发现**：
- ✅ 事件命中数和转化计数的 **WHERE 过滤条件完全相同**
- ❌ 差异只在 **SELECT 子句** 的聚合方式上

### 3.4 过滤条件在两种统计中的一致性

让我们验证每个过滤条件在两种统计中是否一致：

| 过滤条件 | 事件命中数 | 转化计数 | 一致性 |
|---------|-----------|----------|--------|
| `website_id` | ✅ 需要 | ✅ 需要 | 一致 |
| 日期范围 | ✅ 需要 | ✅ 需要 | 一致 |
| `event_type` | ✅ 需要 | ✅ 需要 | 一致 |
| 目标匹配（`url_path`/`event_name`） | ✅ 需要 | ✅ 需要 | 一致 |
| 维度过滤（browser、country 等） | ✅ 需要 | ✅ 需要 | 一致 |
| `cohortQuery` | ✅ 需要 | ✅ 需要 | 一致 |
| `joinSessionQuery` | ✅ 需要 | ✅ 需要 | 一致 |

**结论**：过滤条件在事件命中数和转化计数中**完全一致**，差异只在聚合函数。

### 3.5 为什么容易产生误解？

**误解来源 1：total 的计算差异**

`total`（总访客数）的过滤条件与 `num`（转化计数）不同（排除了 event_type 和目标匹配），这容易让人误以为事件命中数和转化计数的过滤条件也不同。

**误解来源 2：语义混淆**

- "转化"一词在日常语境中可能指"事件被触发"（事件命中）
- 但在 Umami 的代码中，"转化"特指"独立访客完成目标"（转化计数）

### 3.6 前端展示的印证

**Goal.tsx 中的展示逻辑** (`Goal.tsx:71-84`)：
```typescript
<Text title={`${data?.num} / ${data?.total}`}>
  {`${formatLongNumber(data?.num)} / ${formatLongNumber(data?.total)}`}
</Text>
// ...
<ProgressBar
  value={data?.num || 0}
  minValue={0}
  maxValue={data?.total || 1}
/>
<Text weight="bold" size="4xl">
  {data?.total ? Math.round((+data?.num / +data?.total) * 100) : '0'}%
</Text>
```

**展示内容**：
- `num / total`：转化数 / 总访客数
- 转化率：`(num / total) * 100%`
- 进度条：num 在 total 中的占比

**印证**：前端完全基于会话级别的统计（num 和 total 都是 `count(distinct session_id)`），没有展示事件命中数。

### 3.7 过滤条件继承的完整图谱

```
外部过滤条件（filters）
    │
    ├─ 传入 parseFilters()
    │     │
    │     ├─ filterQuery（含 event_type）
    │     │     │
    │     │     ├─ num 查询 → 完整继承
    │     │     └─ total 查询 → 排除 event_type
    │     │
    │     ├─ dateQuery
    │     │     ├─ num 查询 → 继承
    │     │     └─ total 查询 → 继承
    │     │
    │     ├─ cohortQuery
    │     │     ├─ num 查询 → 继承
    │     │     └─ total 查询 → 继承
    │     │
    │     └─ joinSessionQuery
    │           ├─ num 查询 → 继承
    │           └─ total 查询 → 继承
    │
    └─ 目标匹配参数（type、value）
          │
          └─ num 查询 → 特有：and url_path like '...'
                  （total 查询中没有这行）
```

---

## 四、新增修正点清单

### ✅ 修正 8：total 统计的过滤条件继承规则
- **之前理解**：可能以为 total 只继承日期条件
- **正确理解**：total 继承日期条件、cohort 过滤、joinSessionQuery，以及除 event_type 外的所有维度过滤条件；只排除 event_type 和目标匹配条件

### ✅ 修正 9：excludeEventTypeFilterQuery 的精确行为
- **之前理解**：可能以为是简单地移除 event_type 参数
- **正确理解**：通过字符串拆分和过滤，精确移除 `filterQuery` 中包含 `event_type` 的 SQL 行，保留其他所有过滤条件行

### ✅ 修正 10：joinSessionQuery 的自动判断逻辑
- **之前理解**：可能以为 JOIN session 表是固定行为
- **正确理解**：根据过滤条件自动判断是否需要 JOIN——只有当过滤条件包含 referrer 或 SESSION_COLUMNS（browser、os、device 等）时才会 JOIN

### ✅ 修正 11：Kafka 分流的具体位置
- **之前理解**：可能以为 Kafka 分流发生在 runQuery 层，与 PostgreSQL、ClickHouse 同级
- **正确理解**：Kafka 分流发生在 `clickhouseQuery` 函数**内部**，由 `kafka.enabled` 标志控制，不是 runQuery 的职责

### ✅ 修正 12：runQuery 的真实职责边界
- **之前理解**：可能以为 runQuery 负责所有数据库/消息队列的选择
- **正确理解**：runQuery 只负责在 PostgreSQL（PRISMA）和 ClickHouse 之间选择；Kafka 是 ClickHouse 写入路径上的可选缓冲层，与 runQuery 无关

### ✅ 修正 13：saveEvent 的 runQuery 调用没有 KAFKA 选项
- **之前理解**：可能以为 saveEvent 的 runQuery 调用包含 KAFKA 选项
- **正确理解**：`saveEvent` 调用 `runQuery({ PRISMA, CLICKHOUSE })` 时**没有 KAFKA 选项**，Kafka 选择是在 clickhouseQuery 内部通过 if-else 实现的

### ✅ 修正 14：事件命中数与转化计数的过滤条件一致性
- **之前理解**：可能以为事件命中数和转化计数的过滤条件不同
- **正确理解**：两者的 WHERE 过滤条件**完全相同**，差异只在 SELECT 子句的聚合方式（`count(*)` vs `count(distinct session_id)`）

### ✅ 修正 15：Umami 目标报表不返回事件命中数
- **之前理解**：可能以为 num 是事件命中数
- **正确理解**：Umami 目标报表返回的 `num` 是**转化计数**（`count(distinct session_id)`），事件命中数（`count(*)`）在当前代码中没有被计算和返回

### ✅ 修正 16：eventData 的写入链路同样有 Kafka 分流
- **之前理解**：可能以为只有主事件写入有 Kafka 分流
- **正确理解**：`saveEventData` 的 clickhouseQuery 实现了与 `saveEvent` 完全相同的 Kafka 分流模式，发送到 `event_data` 主题

---

## 五、核心代码位置速查

| 分析点 | 文件路径 | 关键行号 |
|--------|----------|----------|
| total 过滤条件排除 event_type | `src/queries/sql/reports/getGoal.ts` | 49-53, 58-65 |
| joinSessionQuery 自动判断 | `src/lib/prisma.ts` | 232-246 |
| SESSION_COLUMNS 定义 | `src/lib/constants.ts` | 55-65 |
| FILTER_COLUMNS 定义 | `src/lib/constants.ts` | 72-97 |
| saveEvent 的 runQuery 调用 | `src/queries/sql/events/saveEvent.ts` | 65-70 |
| clickhouseQuery 内部 Kafka 分流 | `src/queries/sql/events/saveEvent.ts` | 259-263 |
| saveEventData 的 Kafka 分流 | `src/queries/sql/events/saveEventData.ts` | 74-78 |
| runQuery 数据库选择逻辑 | `src/lib/db.ts` | 22-36 |
| Kafka 启用条件 | `src/lib/kafka.ts` | 14 |
| Goal 组件 num/total 使用 | `src/app/(main)/websites/[websiteId]/(reports)/goals/Goal.tsx` | 71-84 |
| 目标匹配条件拼接 | `src/queries/sql/reports/getGoal.ts` | 33-38, 71, 125 |
