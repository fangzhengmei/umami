# 目标转化代码最终边界校准

## 一、Cohort 参数完整链路与 eventType 间接约束

### 1.1 Cohort 参数构造到 SQL 生成的完整链路

```
前端传入 cohort 参数
    ↓
getQueryFilters(...) (src/lib/request.ts:134-160)
    ↓
调用 getWebsiteSegment() 获取 cohort 配置
    ↓
构造 cohortFilters，每个 filter 名称添加 cohort_ 前缀
    ↓
添加 cohort_action 过滤器（类型为 cohort_{action.type}）
    ↓
转换为 filters 对象（filtersArrayToObject）
    ↓
parseFilters(...) (prisma.ts:232-246 / clickhouse.ts:224-236)
    ↓
分离 cohort 过滤器（以 cohort_ 开头）
    ↓
getCohortQuery(...) (prisma.ts:152-173 / clickhouse.ts:142-161)
    ↓
调用 getFilterQuery(..., { isCohort: true })
    ↓
生成 JOIN 子查询 SQL
    ↓
最终 SQL 中插入 ${cohortQuery}
```

### 1.2 Cohort 参数构造细节

**代码位置**：`src/lib/request.ts:134-160`

```typescript
if (params.cohort) {
  const cohortParams = (await getWebsiteSegment(websiteId, params.cohort))
    ?.parameters as Record<string, any>;

  const { startDate, endDate } = parseDateRange(cohortParams.dateRange);

  // 步骤1：普通 cohort 过滤器添加 cohort_ 前缀
  const cohortFilters = cohortParams.filters.map(({ name, ...props }) => ({
    ...props,
    name: `cohort_${name}`,  // 如 'browser' → 'cohort_browser'
  }));

  // 步骤2：cohort action 过滤器添加 cohort_ 前缀
  cohortFilters.push({
    name: `cohort_${cohortParams.action.type}`,  // 如 'eventType' → 'cohort_eventType'
    operator: OPERATORS.equals,
    value: cohortParams.action.value,
  });

  // 步骤3：转换为对象格式
  Object.assign(filters, {
    ...filtersArrayToObject(cohortFilters),  // 如 { cohort_browser: 'eq.Chrome' }
    cohort_startDate: startDate,
    cohort_endDate: endDate,
    ...(cohortParams.match && {
      cohort_match: cohortParams.match,
      cohort_actionName: `cohort_${cohortParams.action.type}`,  // 如 'cohort_eventType'
    }),
  });
}
```

**关键参数**：
- `cohort_actionName`：标记哪个过滤器是 cohort action，用于 `isAlwaysAnd` 判断
- `cohort_match`：匹配模式，'all' 表示所有条件都满足（AND），'any' 表示任意条件满足（OR）

### 1.3 isAlwaysAnd 逻辑详解

**代码位置**：
- Prisma：`src/lib/prisma.ts:122`
- ClickHouse：`src/lib/clickhouse.ts:113`

```typescript
const isAlwaysAnd = name === 'eventType' || (isCohort && name === cohortActionName);
```

**两个触发条件**：

| 条件 | 场景 | 效果 |
|------|------|------|
| `name === 'eventType'` | 普通查询中的 eventType 过滤 | 强制 AND 连接，不受 match 模式影响 |
| `isCohort && name === cohortActionName` | cohort 查询中的 action 过滤器 | 强制 AND 连接，不受 cohort_match 模式影响 |

**为什么需要 isAlwaysAnd？**

1. **逻辑必要性**：eventType（目标类型）是基础过滤条件，必须始终满足，不能被 OR 逻辑干扰
2. **Cohort action 语义**：cohort action 定义了"用户必须完成的动作"，这是 cohort 的核心定义，必须始终满足
3. **避免逻辑错误**：如果 cohort_match = 'any'，其他条件可以用 OR 连接，但 action 条件必须用 AND 连接

### 1.4 Cohort action 命中 eventType 的场景分析

当 `cohortParams.action.type === 'eventType'` 时，会发生以下连锁反应：

#### 场景参数示例
```javascript
cohortParams = {
  action: {
    type: 'eventType',  // action 类型是 eventType
    value: 1            // pageView
  },
  match: 'any',         // 其他条件用 OR 连接
  filters: [
    { name: 'browser', operator: 'eq', value: 'Chrome' },
    { name: 'country', operator: 'eq', value: 'US' }
  ]
};
```

#### 构造过程
1. `cohort_actionName = 'cohort_eventType'`
2. cohortFilters 包含：
   - `{ name: 'cohort_browser', ... }`
   - `{ name: 'cohort_country', ... }`
   - `{ name: 'cohort_eventType', ... }`

#### Cohort 子查询 WHERE 条件生成
```sql
where website_event.website_id = {{websiteId}}
  and website_event.created_at between {{cohort_startDate}} and {{cohort_endDate}}
  
  -- OR 连接的其他条件
  and (
    browser = ANY({{cohort_browser}})
    or country = ANY({{cohort_country}})
  )
  
  -- cohort action (eventType) 强制 AND 连接，不受 match='any' 影响
  and event_type = ANY({{cohort_eventType}})  -- event_type = 1 (pageView)
```

#### 对 total 统计的间接约束

**核心机制**：
- cohortQuery 是一个 `JOIN (select distinct session_id ...) as cohort on cohort.session_id = website_event.session_id`
- 这个 JOIN 同时作用于 num 查询和 total 查询
- cohort 子查询中的 eventType 过滤（`event_type = 1`）限制了 cohort 子查询返回的 session_id 范围
- 因此 total 查询虽然排除了主查询中的 eventType 过滤，但通过 cohort JOIN 被**间接约束**

**SQL 结构示意**（Prisma 版本）：

```sql
-- num 查询
select 
  count(distinct website_event.session_id) as num,
  (
    -- total 子查询
    select count(distinct website_event.session_id)
    from website_event
    ${cohortQuery}           -- ← 这里继承了 cohortQuery，包含 JOIN
    ${joinSessionQuery}
    where website_event.website_id = {{websiteId::uuid}}
      ${dateQuery}
      ${excludeEventTypeFilterQuery}  -- ← 这里排除了主查询的 eventType
  ) as total
from website_event
${cohortQuery}               -- ← 这里也继承了 cohortQuery
${joinSessionQuery}
where website_event.website_id = {{websiteId::uuid}}
  and url_path = {{value}}
  ${dateQuery}
  ${filterQuery}             -- ← 这里包含主查询的 eventType
```

**cohortQuery 的内容**：
```sql
join
  (select distinct website_event.session_id
  from website_event
  join session on ...
  where website_event.website_id = {{websiteId}}
    and website_event.created_at between {{cohort_startDate}} and {{cohort_endDate}}
    and (browser = ... or country = ...)                -- OR 连接
    and event_type = ANY({{cohort_eventType}})          -- ← eventType 强制 AND
  ) cohort
  on cohort.session_id = website_event.session_id
```

**间接约束效果**：
- total 子查询虽然排除了主查询的 eventType 过滤（`excludeEventTypeFilterQuery`）
- 但通过 `cohortQuery` JOIN，total 子查询只能统计 `cohort_eventType = 1` 范围内的 session
- 因此 total 统计**间接被约束**为只包含触发了 pageView 事件的访客

### 1.5 常规规则 vs Cohort action 命中 eventType 例外

| 维度 | 常规规则（无 cohort） | 例外规则（cohort action 是 eventType） |
|------|----------------------|---------------------------------------|
| total 统计的 eventType 约束 | 通过 `excludeEventTypeFilterQuery` 排除，无 eventType 约束 | 通过 cohortQuery JOIN 间接约束，实际有 eventType 约束 |
| total 统计的访客范围 | 所有类型事件的访客 | 仅 cohort action 定义的 eventType 范围内的访客 |
| eventType 过滤位置 | 主查询 filterQuery 中，total 子查询排除 | cohort 子查询 WHERE 中，total 子查询通过 JOIN 继承 |
| 转化率计算影响 | 分母（total）大，转化率相对低 | 分母（total）被 cohort 约束，转化率相对高 |
| 语义 | "所有访客中完成目标的比例" | "cohort 定义范围内的访客中完成目标的比例" |

**关键理解**：
- `excludeEventTypeFilterQuery` 只排除**主查询** filterQuery 中的 eventType
- 它**不排除** cohortQuery 中的 eventType 过滤（因为 cohortQuery 是独立的 JOIN 子查询）
- 当 cohort action 是 eventType 时，total 统计实际上被间接约束了

---

## 二、Prisma 与 ClickHouse 过滤实现语义层级差异

### 2.1 同名筛选条件的语义层级对比

| 筛选条件 | Prisma (PostgreSQL) 实现 | ClickHouse 实现 | 语义层级差异 |
|---------|-------------------------|-----------------|-------------|
| **表名前缀** | 通过 `mapFilter` 自动添加表名前缀 | 不添加表名前缀（宽表设计） | 重大差异：Prisma 是多表 JOIN，ClickHouse 是单宽表 |
| **字段归属** | `SESSION_COLUMNS` 字段归属 session 表，其他归属 website_event 表 | 所有字段都在 website_event 表 | 架构差异导致的语义不同 |
| **eventType 数据类型** | 无特殊类型处理（PostgreSQL 自动类型转换） | 明确指定 `UInt32` 类型 | ClickHouse 是强类型，需要显式指定 |
| **IN/NOT IN vs ANY/ALL** | `= ANY(...)` / `!= ALL(...)` | `IN (...)` / `NOT IN (...)` | SQL 语法差异，语义相同 |
| **LIKE vs positionCaseInsensitive** | `ilike`（PostgreSQL 内置） | `positionCaseInsensitive(...) > 0` | ClickHouse 没有 ilike，用函数实现 |
| **joinSessionQuery** | 有，自动判断是否需要 JOIN session 表 | 无，宽表设计不需要 JOIN | 架构差异 |
| **isAlwaysAnd 逻辑** | 相同的判断逻辑 | 相同的判断逻辑 | 语义一致 |
| **OR 分组逻辑** | `and (cond1 or cond2)` | `and (cond1 or cond2)` | 语义一致 |
| **referrer 额外过滤** | `referrer_domain != regexp_replace(hostname, ...)` | `referrer_domain != hostname` | 语义基本一致，实现略有不同 |

### 2.2 表名前缀的自动判断（Prisma 特有）

**代码位置**：`src/lib/prisma.ts:84-88`

```typescript
function mapFilter(column: string, operator: string, name: string, type: string = '', paramName?: string) {
  // ...
  
  if (name.startsWith('cohort_')) {
    name = name.slice('cohort_'.length);  // 移除 cohort_ 前缀后再判断
  }

  const table = SESSION_COLUMNS.includes(name) ? 'session' : 'website_event';
  
  switch (operator) {
    case OPERATORS.equals:
      return `${table}.${column} = ANY(${value})`;  // 如 session.browser = ANY(...)
    // ...
  }
}
```

**SESSION_COLUMNS**（`src/lib/constants.ts:55-65`）：
```typescript
export const SESSION_COLUMNS = [
  'browser', 'os', 'device', 'screen', 'language',
  'country', 'city', 'region', 'distinctId',
];
```

**字段归属判断**：

| 字段名 | 归属表（Prisma） | 归属表（ClickHouse） |
|--------|-----------------|---------------------|
| browser | session | website_event（宽表） |
| os | session | website_event（宽表） |
| device | session | website_event（宽表） |
| country | session | website_event（宽表） |
| url_path | website_event | website_event |
| event_name | website_event | website_event |
| event_type | website_event | website_event |
| referrer_domain | website_event | website_event |

### 2.3 eventType 数据类型差异

**Prisma 版本**（`src/lib/prisma.ts:74-106`）：
```typescript
function mapFilter(column: string, operator: string, name: string, type: string = '', paramName?: string) {
  const value = `{{${param}${type ? `::${type}` : ''}}}`;  // type 默认为空
  // ...
  // eventType 不特殊处理，依赖 PostgreSQL 自动类型转换
  return `${table}.${column} = ANY(${value})`;
}
```

**ClickHouse 版本**（`src/lib/clickhouse.ts:115-118`）：
```typescript
if (isAlwaysAnd) {
  andClauses.push(
    `and ${mapFilter(column, operator, name, name === 'eventType' ? 'UInt32' : 'String', paramName)}`,
  );
}
```

**mapFilter 中的类型处理**（`src/lib/clickhouse.ts:73-99`）：
```typescript
function mapFilter(column: string, operator: string, name: string, type: string = 'String', paramName?: string) {
  const value = `{${param}:${type}}`;  // eventType → {eventType:UInt32}
  // ...
  return `${column} IN {${param}:Array(${type})}`;  // event_type IN {eventType:Array(UInt32)}
}
```

**差异原因**：
- PostgreSQL 是弱类型，字符串可以自动转换为整数
- ClickHouse 是强类型，必须显式指定参数类型，否则会报类型不匹配错误

### 2.4 可跨库通用的结论

以下结论在 Prisma 和 ClickHouse 两条路径中**语义完全一致**：

✅ **通用结论 1**：total 统计通过 `excludeEventTypeFilterQuery` 排除主查询 filterQuery 中的 eventType 过滤

✅ **通用结论 2**：`isAlwaysAnd` 逻辑完全相同——`eventType` 和 `cohort action` 始终使用 AND 连接，不受 match 模式影响

✅ **通用结论 3**：cohort action 命中 eventType 时，total 统计通过 cohortQuery JOIN 被间接约束

✅ **通用结论 4**：事件命中数与转化计数的过滤条件完全相同，差异仅在聚合方式

✅ **通用结论 5**：num 和 total 都继承 cohortQuery、dateQuery、除 eventType 外的维度过滤

✅ **通用结论 6**：OR 分组逻辑相同——非 isAlwaysAnd 条件在 `match='any'` 时用 OR 连接

✅ **通用结论 7**：referrer 过滤会额外添加"排除自引用"的条件

✅ **通用结论 8**：cohort_ 前缀在生成 SQL 条件前会被移除（用于正确判断字段归属）

### 2.5 仅适用于单一路径的结论

以下结论**只适用于特定数据库路径**，不能跨库通用：

⚠️ **Prisma 特有**：
- `joinSessionQuery` 自动判断是否需要 JOIN session 表
- 字段通过表名前缀（`session.` 或 `website_event.`）区分归属
- 使用 `= ANY(...)` 和 `!= ALL(...)` 语法
- 使用 `ilike` 进行大小写不敏感匹配
- 使用 `regexp_replace` 处理 hostname

⚠️ **ClickHouse 特有**：
- 不需要 JOIN session 表（宽表设计，所有字段都在 website_event）
- 没有表名前缀（单表查询）
- `eventType` 必须显式指定 `UInt32` 类型
- 使用 `IN (...)` 和 `NOT IN (...)` 语法
- 使用 `positionCaseInsensitive(...) > 0` 实现大小写不敏感匹配
- 使用 `match()` 函数实现正则匹配
- 直接比较 `referrer_domain != hostname`

### 2.6 跨库语义差异的深层原因

| 差异点 | Prisma (PostgreSQL) | ClickHouse | 设计意图 |
|--------|---------------------|------------|----------|
| 表结构 | 规范化设计（多表） | 宽表设计（单表） | PostgreSQL 适合事务性操作，ClickHouse 适合分析查询 |
| 类型系统 | 弱类型，自动转换 | 强类型，显式声明 | ClickHouse 追求极致性能，强类型减少运行时开销 |
| JOIN 策略 | 自动判断是否 JOIN | 从不 JOIN | ClickHouse JOIN 性能代价高，宽表设计避免 JOIN |
| 函数选择 | 使用 PostgreSQL 内置函数 | 使用 ClickHouse 专用函数 | 每个数据库的函数库不同 |

---

## 三、最终可确认结论清单

### ✅ C1: Cohort action 命中 eventType 时 total 被间接约束
- **证据**：`request.ts:145-149`（cohort action 构造）、`prisma.ts:122` / `clickhouse.ts:113`（isAlwaysAnd）、`getGoal.ts:62,68`（cohortQuery 同时用于 num 和 total）
- **适用范围**：跨库通用

### ✅ C2: excludeEventTypeFilterQuery 只排除主查询的 eventType，不排除 cohort 中的 eventType
- **证据**：`getGoal.ts:49-53`（字符串过滤只作用于 filterQuery，不作用于 cohortQuery）
- **适用范围**：跨库通用

### ✅ C3: isAlwaysAnd 有两个独立触发条件（eventType 和 cohort action）
- **证据**：`prisma.ts:122` / `clickhouse.ts:113`
- **适用范围**：跨库通用

### ✅ C4: Prisma 自动添加表名前缀，ClickHouse 不添加
- **证据**：`prisma.ts:84-88`（table 判断逻辑）、`clickhouse.ts:73-99`（无 table 变量）
- **适用范围**：分别适用于各自路径

### ✅ C5: ClickHouse 的 eventType 必须显式指定 UInt32 类型
- **证据**：`clickhouse.ts:117`（`name === 'eventType' ? 'UInt32' : 'String'`）
- **适用范围**：仅 ClickHouse 路径

### ✅ C6: 事件命中数与转化计数的过滤条件完全相同
- **证据**：两者共享相同的 WHERE 条件，差异仅在 SELECT 聚合方式
- **适用范围**：跨库通用

### ✅ C7: 两个路径的 isAlwaysAnd 和 OR 分组逻辑完全相同
- **证据**：`prisma.ts:108-150` 和 `clickhouse.ts:101-140` 的控制流结构完全一致
- **适用范围**：跨库通用（逻辑语义相同）

### ✅ C8: cohort_ 前缀在生成 SQL 条件前被移除
- **证据**：`prisma.ts:84-86`、`clickhouse.ts:108-110`
- **适用范围**：跨库通用

### ✅ C9: referrer 过滤会额外添加"排除自引用"条件
- **证据**：`prisma.ts:132-136`、`clickhouse.ts:125-127`
- **适用范围**：跨库通用（语义相同，实现略有差异）

---

## 四、保留假设清单

以下内容基于代码分析和合理推断，**没有直接的代码证据**可以 100% 确认，需要保留为假设：

### ⚠️ A1: Cohort action 类型可以是任意 FILTER_COLUMNS 中的字段
- **推断依据**：代码中没有限制 cohort action.type 的取值，只是通过 `FILTER_COLUMNS[name.slice('cohort_'.length)]` 进行映射
- **验证方式**：查看 cohort 配置的 UI 或 API 校验逻辑
- **置信度**：高

### ⚠️ A2: Cohort action 是 eventType 是一个有意设计的功能，不是 bug
- **推断依据**：isAlwaysAnd 逻辑明确区分了普通 eventType 和 cohort action eventType，两者都是强制 AND
- **验证方式**：查看产品文档或与开发团队确认
- **置信度**：高

### ⚠️ A3: Cohort 间接约束 total 是预期行为，不是设计疏忽
- **推断依据**：cohort 的语义就是"在特定人群中分析"，total 应该被 cohort 约束
- **验证方式**：查看产品设计文档
- **置信度**：高

### ⚠️ A4: ClickHouse 宽表设计是为了查询性能优化
- **推断依据**：ClickHouse 是列式分析数据库，宽表设计可以避免 JOIN，显著提升查询性能
- **验证方式**：查看架构设计文档或性能测试报告
- **置信度**：高

### ⚠️ A5: Prisma 的多表设计是为了数据一致性和存储空间优化
- **推断依据**：PostgreSQL 是关系型数据库，规范化设计可以减少数据冗余，保证一致性
- **验证方式**：查看数据库设计文档
- **置信度**：高

### ⚠️ A6: 两个路径的查询结果应该是等价的（相同输入产生相同输出）
- **推断依据**：代码逻辑语义基本一致，只是 SQL 语法和表结构不同
- **验证方式**：运行双写测试，对比两个数据库的查询结果
- **置信度**：中

### ⚠️ A7: excludeEventTypeFilterQuery 的字符串过滤方式是安全的
- **推断依据**：event_type 字段名是固定的，不太可能出现在其他过滤条件的 SQL 片段中
- **验证方式**：分析所有可能的 filterQuery 生成路径，确认没有误过滤风险
- **置信度**：高

---

## 五、边界校准总结

### 5.1 最容易误解的边界点

1. **"total 排除 eventType"不是绝对的**
   - 排除的只是**主查询**中的 eventType
   - 如果 cohort action 是 eventType，total 通过 JOIN 被间接约束

2. **"isAlwaysAnd 只有 eventType"是不完整的**
   - 还有第二个触发条件：cohort 查询中的 cohort action
   - 这两个条件是 OR 关系，满足任一即可

3. **"两个数据库路径逻辑相同"是表面现象**
   - 控制流结构相同，但表结构、数据类型、SQL 语法都有差异
   - 部分结论可以跨库通用，部分只能限定在单一路径

4. **"cohort 前缀只是命名约定"的理解不够深入**
   - 它不仅用于区分 cohort 过滤和普通过滤
   - 还用于 `isAlwaysAnd` 判断和字段归属判断
   - 在生成 SQL 前会被移除

### 5.2 完整的过滤条件继承图（考虑 cohort 间接约束）

```
外部参数（含 cohort）
    │
    ├─ 普通过滤条件 → parseFilters → filterQuery
    │     ├─ num 查询 → 完整继承（含 eventType）
    │     └─ total 查询 → 排除 eventType（excludeEventTypeFilterQuery）
    │
    └─ cohort 过滤条件（cohort_ 前缀）→ getCohortQuery
          ├─ cohort 子查询 WHERE 条件
          │     ├─ 普通 cohort 过滤 → OR/AND（根据 cohort_match）
          │     └─ cohort action → 强制 AND（isAlwaysAnd）
          │           ↓
          │           如果 action 是 eventType → event_type 过滤
          │
          └─ cohortQuery（JOIN 子查询）
                ├─ num 查询 → 继承（通过 JOIN 约束 session 范围）
                └─ total 查询 → 继承（通过 JOIN 间接约束 eventType 范围）
```
