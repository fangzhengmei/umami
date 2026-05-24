# 事件分群构造器：UI → 查询编排完整流程分析

本文档从代码实现角度梳理 Umami 中事件分群（Segment）构造器的完整工作流程，包括筛选条件的描述模型、UI 组件与数据模型的映射关系，以及前端参数到后端查询的编排过程。

---

## 一、核心数据模型

### 1.1 筛选条件（Filter）数据结构

**文件：** `src/lib/types.ts:33-41`

```typescript
export interface Filter {
  name: string;           // 字段基础名（如 path, country, browser，去掉数字后缀）
  operator: Operator;     // 操作符（枚举值）
  value: string | string[]; // 筛选值
  type?: string;          // 数据类型（string, number, boolean, date, array, uuid）
  column?: string;        // 数据库列名（映射自 FILTER_COLUMNS）
  prefix?: string;        // 表前缀
  paramName?: string;     // 原始参数名（带数字后缀，用于 SQL 参数绑定去重）
}
```

### 1.2 操作符（Operator）枚举

**文件：** `src/lib/constants.ts:137-154`

| 常量名 | 枚举值 | 含义 | 适用类型 |
|--------|--------|------|----------|
| `OPERATORS.equals` | `eq` | 等于 | string, number, uuid |
| `OPERATORS.notEquals` | `neq` | 不等于 | string, number |
| `OPERATORS.set` | `s` | 已设置 | - |
| `OPERATORS.notSet` | `ns` | 未设置 | - |
| `OPERATORS.contains` | `c` | 包含 | string, array |
| `OPERATORS.doesNotContain` | `dnc` | 不包含 | string, array |
| `OPERATORS.regex` | `re` | 正则匹配 | string |
| `OPERATORS.notRegex` | `nre` | 正则不匹配 | string |
| `OPERATORS.true` | `t` | 为真 | boolean |
| `OPERATORS.false` | `f` | 为假 | boolean |
| `OPERATORS.greaterThan` | `gt` | 大于 | number |
| `OPERATORS.lessThan` | `lt` | 小于 | number |
| `OPERATORS.greaterThanEquals` | `gte` | 大于等于 | number |
| `OPERATORS.lessThanEquals` | `lte` | 小于等于 | number |
| `OPERATORS.before` | `bf` | 之前 | date |
| `OPERATORS.after` | `af` | 之后 | date |

### 1.3 字段分组（Field Groups）

**文件：** `src/components/hooks/useFields.ts:15-129`

字段按语义分组，在 UI 中以分类菜单展示：

| 分组 | 字段 |
|------|------|
| `url` | path, query, title |
| `sources` | referrer |
| `location` | country, region, city |
| `environment` | browser, os, device |
| `utm` | utmSource, utmMedium, utmCampaign, utmContent, utmTerm |
| `other` | hostname, distinctId, tag, event |

### 1.4 字段 → 数据库列映射

**文件：** `src/lib/constants.ts:72-97`

`FILTER_COLUMNS` 常量定义了前端字段名到数据库列名的映射：

```typescript
export const FILTER_COLUMNS = {
  path: 'url_path',
  referrer: 'referrer_domain',
  hostname: 'hostname',
  distinctId: 'distinct_id',
  title: 'page_title',
  query: 'url_query',
  os: 'os',
  browser: 'browser',
  device: 'device',
  country: 'country',
  region: 'region',
  city: 'city',
  language: 'language',
  event: 'event_name',
  tag: 'tag',
  eventType: 'event_type',
  utmSource: 'utm_source',
  utmMedium: 'utm_medium',
  utmCampaign: 'utm_campaign',
  utmContent: 'utm_content',
  utmTerm: 'utm_term',
};
```

### 1.5 参数分类：过滤参数 vs 控制参数

**⚠️ 关键概念区分：** `FILTER_COLUMNS`（过滤参数）与 `filterParams`（Schema 校验字段）是两个不同的集合。

#### 过滤参数（Filter Parameters）
- **定义来源：** `FILTER_COLUMNS`（`src/lib/constants.ts:72-97`）
- **提取函数：** `getRequestFilters()`（`src/lib/request.ts:79-90`）
- **判定逻辑：** `if (baseName in FILTER_COLUMNS)` 才会进入 `filters` 条件集合
- **作用：** 最终会被转换为 SQL WHERE 条件

| 过滤参数 | 数据库列 |
|---------|---------|
| path, referrer, hostname, distinctId, title, query | website_event 表字段 |
| os, browser, device, country, region, city, language | session 表字段（SESSION_COLUMNS） |
| event, tag, eventType | website_event 表字段 |
| utmSource, utmMedium, utmCampaign, utmContent, utmTerm | website_event 表字段 |

#### 控制参数（Control Parameters）
- **定义来源：** `filterParams`（`src/lib/schema.ts:45-71`）- 用于 Zod Schema 校验
- **特点：** NOT IN `FILTER_COLUMNS`，不会被 `getRequestFilters()` 提取
- **作用：** 控制查询行为，不直接转为 SQL WHERE 条件

| 控制参数 | 处理方式 | 是否进入 filters 集合 |
|---------|---------|---------------------|
| `segment` | `params.segment` 读取，用于查库加载分群 | ❌ 从不进入 |
| `cohort` | `params.cohort` 读取，用于查库加载队列 | ❌ 从不进入 |
| `match` | `params.match` 读取，控制 AND/OR 逻辑 | ❌ 从不进入 |
| `excludeBounce` | 特殊处理：`params.excludeBounce` 为真时，通过 `Object.assign(filters, { excludeBounce: true })` 加入 | ⚠️ 条件性加入（作为布尔标志） |
| `startAt`, `endAt`, `timezone`, `unit` | `getRequestDateRange()` 单独处理 | ❌ 作为 dateRange 单独返回 |
| `page`, `pageSize`, `orderBy`, `sortDescending`, `search`, `compare` | 直接从 `params` 读取 | ❌ 作为独立字段返回 |

#### 特殊参数处理

| 参数 | 分类 | 特殊处理 |
|-----|------|---------|
| `eventType` | ✅ 过滤参数 | 在 `FILTER_COLUMNS` 中，但是在 `getFilterQuery()` 中强制 AND 连接 |
| `excludeBounce` | ⚠️ 混合 | 不在 `FILTER_COLUMNS`，但通过 `Object.assign(filters, { excludeBounce: true })` 加入 `filters`，用于触发 `getExcludeBounceQuery()` 生成 JOIN 子查询 |
| `cohort_*` 前缀字段 | ⚠️ 混合 | 队列筛选条件加 `cohort_` 前缀后加入 `filters`，在 `parseFilters()` 中分离处理 |

**核心判定代码（`src/lib/request.ts:79-90`）：**
```typescript
export function getRequestFilters(query: Record<string, any>) {
  const result: Record<string, any> = {};
  for (const key of Object.keys(query)) {
    const baseName = key.replace(/\d+$/, '');
    if (baseName in FILTER_COLUMNS) {  // ⭐ 只有 FILTER_COLUMNS 中的字段才会被提取
      result[key] = query[key];
    }
  }
  return result;
}
```

**⚠️ 重要结论 1：** `segment` 参数**从不**进入 `filters` 条件集合。它是一个控制参数，仅用于从数据库加载对应的分群定义，然后分群的 `filters` 数组才会被转换并合并到查询条件中。

**⚠️ 重要结论 2（命名混淆点）：** 代码中存在三个不同层次的 `filters` 对象，容易混淆：

| 层次 | 来源 | 包含内容 | `segment` / `cohort` |
|------|------|---------|---------------------|
| **1. 原始 params** | URL query / POST body | 所有参数（过滤 + 控制） | ✅ 包含 |
| **2. 狭义 filters** | `getRequestFilters(params)` | 仅 `FILTER_COLUMNS` 中的过滤参数 | ❌ 不包含（通过 `if (baseName in FILTER_COLUMNS)` 过滤） |
| **3. 广义 filters** | `getQueryFilters()` 返回值 | `{ ...dateRange, ...狭义filters, match, excludeBounce, page, pageSize, ... }` | ❌ 不包含（`segment` / `cohort` 仅用于查库，未加入返回值） |

**`match` 参数的特殊访问路径：**
```
getQueryFilters() 返回 { ..., match }  →  广义filters.match
  ↓
parseFilters(广义filters) → getFilterQuery(广义filters)
  ↓
getFilterQuery(filters) 中访问 filters.match （虽然 match 不在 FILTER_COLUMNS）
```

**`excludeBounce` 参数的特殊访问路径：**
```
params.excludeBounce （控制参数）
  ↓
getQueryFilters() 中条件性加入 filters:
  if (params.excludeBounce) Object.assign(filters, { excludeBounce: true })
  ↓
parseFilters(广义filters) → getExcludeBounceQuery(filters)
  ↓
if (filters.excludeBounce !== true) 返回空，否则生成 JOIN 子查询
```

### 1.6 分群（Segment）存储模型

**文件：** `prisma/schema.prisma`（Segment 表）

分群保存时，筛选条件以 JSON 格式存储在 `parameters` 字段中：

```json
{
  "filters": [
    { "name": "country", "operator": "eq", "value": "CN" },
    { "name": "browser", "operator": "eq", "value": "Chrome" }
  ],
  "match": "all"  // 或 "any"
}
```

**Schema 定义：** `src/lib/schema.ts:303-321`

```typescript
export const segmentParamSchema = z.object({
  filters: z.array(
    z.object({
      name: z.string(),
      operator: operatorParam,
      value: z.string(),
    }),
  ).optional(),
  match: z.enum(['all', 'any']).optional(),
  dateRange: z.string().optional(),
  action: z.object({
    type: z.string(),
    value: z.string(),
  }).optional(),
});
```

---

## 二、UI 组件层 → 数据模型映射

### 2.1 组件层级结构

```
FilterEditForm (src/components/input/FilterEditForm.tsx)
├─ Tabs: 字段 / 分群 / 队列
├─ FieldFilters (src/components/input/FieldFilters.tsx)
│  ├─ 左侧字段分组列表（List/ListSection）
│  └─ 右侧筛选条件列表
│     └─ FilterRecord (src/components/common/FilterRecord.tsx) × N
│        ├─ 字段标签（Label）
│        ├─ 操作符选择（Select）
│        └─ 值输入（MultiSelect 或 TextField）
└─ 匹配模式选择（Match: all / any）
```

### 2.2 FieldFilters 组件

**文件：** `src/components/input/FieldFilters.tsx`

**核心状态管理：**
- `value: Filter[]` - 当前筛选条件数组
- `match: 'all' | 'any'` - 多条件匹配模式

**关键交互：**
- `handleAdd(name)` - 添加新筛选条件，默认 `operator: 'eq'`, `value: ''`
- `handleChange(index, value)` - 更新筛选值
- `handleSelect(index, operator)` - 更新操作符
- `handleRemove(index)` - 删除筛选条件

### 2.3 FilterRecord 组件

**文件：** `src/components/common/FilterRecord.tsx`

单个筛选条件行的 UI 组件，包含三部分：

1. **字段标签**：通过 `useFields()` 查找 `name` 对应的显示标签
2. **操作符下拉**：⚠️ **硬编码仅显示 string 类型操作符**（`FilterRecord.tsx:82-89`）
3. **值输入**：
   - 搜索类操作符（`c`, `dnc`, `re`, `nre`）→ `TextField` 自由输入
   - 其他操作符 → `MultiSelect` 从后端枚举值中选择，调用 `useWebsiteValuesQuery` 动态加载可选值

#### ⚠️ 操作符选择逻辑的理解纠正

**之前的理解偏差**：操作符下拉会根据字段类型动态过滤可用操作符（使用 `useFilters().typeFilters`）

**实际代码实现**（`FilterRecord.tsx:82-89`）：

```typescript
<Select value={operator} onChange={handleSelectOperator}>
  {operators
    .filter(({ type }) => type === 'string')  // 硬编码只显示 string 类型操作符！
    .map(({ name, label }: any) => (
      <ListItem key={name} id={name}>
        {label}
      </ListItem>
    ))}
</Select>
```

**真相**：当前实现中，**所有字段的操作符下拉都只显示 string 类型的 6 个操作符**（`eq`, `neq`, `c`, `dnc`, `re`, `nre`）。`useFilters.ts` 中虽然定义了 `typeFilters` 和 `getFilters(type)` 函数（按类型返回对应操作符），但这些在 UI 中**并未被实际使用**。

**`typeFilters` 定义**（`src/components/hooks/useFilters.ts:36-57`）：

```typescript
const typeFilters = {
  string: [eq, neq, c, dnc, re, nre],      // 当前实际使用的
  array: [c, dnc],
  boolean: [t, f],
  number: [eq, neq, gt, lt, gte, lte],     // 定义了但 UI 未使用
  date: [bf, af],                           // 定义了但 UI 未使用
  uuid: [eq],                               // 定义了但 UI 未使用
};
```

### 2.4 操作符标签映射

**文件：** `src/components/hooks/useOperatorLabels.ts`

`useOperatorLabels()` 返回操作符枚举值到国际化显示文本的映射：

```typescript
{
  eq: '是',
  neq: '不是',
  c: '包含',
  dnc: '不包含',
  re: '正则匹配',
  nre: '正则不匹配',
  // ...其他操作符的标签
}
```

### 2.5 数据流向（UI 状态 → URL 参数）

**文件：** `src/components/input/FilterEditForm.tsx:60-68`

点击「应用」按钮时，通过 `filtersArrayToObject` 将 Filter 数组转为 URL 参数对象：

```typescript
onChange?.({
  filters: currentFilters.filter(f => f.value),
  segment: currentSegment,
  cohort: currentCohort,
  match: currentMatch !== 'all' ? currentMatch : undefined,
});
```

**数组 → URL 参数转换：** `src/lib/params.ts:78-90`

```typescript
export function filtersArrayToObject(filters: Filter[]) {
  const nameCounts: Record<string, number> = {};
  return filters.reduce((obj, filter: Filter) => {
    const { name, operator, value } = filter;
    const count = nameCounts[name] ?? 0;
    const key = count === 0 ? name : `${name}${count}`;  // 同名字段加数字后缀
    nameCounts[name] = count + 1;

    obj[key] = `${operator}.${Array.isArray(value) ? value.join(',') : value}`;

    return obj;
  }, {});
}
```

**示例转换：**
```typescript
// 输入（Filter[]）
[
  { name: 'country', operator: 'eq', value: ['CN', 'US'] },
  { name: 'browser', operator: 'c', value: 'Chrome' },
  { name: 'country', operator: 'neq', value: 'JP' }
]

// 输出（URL params）
{
  country: 'eq.CN,US',    // 第一个 country 无后缀
  browser: 'c.Chrome',
  country1: 'neq.JP'      // 第二个 country 加后缀 1
}
```

---

## 三、同名筛选条件数字后缀完整链路

### 3.1 问题背景

Zod Schema 中只定义了基础字段名（如 `country`, `browser`），未定义带数字后缀的字段（如 `country1`, `country2`）。因此在请求校验时，带后缀的参数会被 Zod 过滤掉，需要特殊处理。

### 3.2 完整链路说明

#### 阶段 1：前端生成带后缀参数

```
Filter[] 数组（含同名字段）
  ↓ filtersArrayToObject() [src/lib/params.ts:78-90]
URL 参数对象：{ country: 'eq.CN', country1: 'neq.US' }
  ↓
HTTP 请求：?country=eq.CN&country1=neq.US
```

#### 阶段 2：请求校验与后缀回填

**文件：** `src/lib/request.ts:23-42`

```typescript
if (schema) {
  const isGet = request.method === 'GET';
  const rawQuery = query;                    // 1. 保存原始 query（含后缀）
  const result = schema.safeParse(isGet ? query : body);  // 2. Zod 校验，会剥离 country1

  if (!result.success) {
    error = () => badRequest(z.treeifyError(result.error));
  } else if (isGet) {
    query = result.data;                     // 3. 此时 query 中已无 country1

    // ⭐ 关键：回填被 Zod 剥离的带后缀参数
    for (const key of Object.keys(rawQuery)) {
      if (/\d+$/.test(key) && !(key in query)) {  // 检测数字后缀且不在校验结果中
        query[key] = rawQuery[key];          // 4. 回填：{ country: 'eq.CN', country1: 'neq.US' }
      }
    }
  }
}
```

#### 阶段 3：筛选参数提取

**文件：** `src/lib/request.ts:79-90`

```typescript
export function getRequestFilters(query: Record<string, any>) {
  const result: Record<string, any> = {};

  for (const key of Object.keys(query)) {
    const baseName = key.replace(/\d+$/, '');  // 去掉数字后缀匹配字段
    if (baseName in FILTER_COLUMNS) {
      result[key] = query[key];               // 保留原始 key（带后缀）
    }
  }

  return result;  // { country: 'eq.CN', country1: 'neq.US' }
}
```

#### 阶段 4：参数 → Filter 对象数组转换

**文件：** `src/lib/params.ts:42-76`

```typescript
export function filtersObjectToArray(filters: QueryFilters, options: QueryOptions = {}): Filter[] {
  return Object.keys(filters).reduce((arr, key) => {
    const filter = filters[key];

    const baseName = key.replace(/\d+$/, '');       // 'country1' → 'country'
    const paramName = key !== baseName ? key : undefined;  // 'country1' 作为 paramName 保留

    const { operator, value } = parseFilterValue(filter);

    return arr.concat({
      name: baseName,        // 基础名：'country'（用于查表名、列名映射）
      paramName,             // 原始参数名：'country1'（用于 SQL 参数绑定）
      column: FILTER_COLUMNS[baseName],  // 'country'
      operator,
      value,
      prefix: options?.prefix,
    });
  }, []);
}
```

转换结果：
```typescript
[
  { name: 'country', paramName: undefined, column: 'country', operator: 'eq', value: ['CN'] },
  { name: 'country', paramName: 'country1', column: 'country', operator: 'neq', value: 'US' }
]
```

#### 阶段 5：SQL 参数绑定（避免参数名冲突）

**文件：** `src/lib/prisma.ts:208-230`

```typescript
function getQueryParams(filters: Record<string, any>) {
  return {
    ...filters,
    ...filtersObjectToArray(filters).reduce((obj, { name, column, operator, value, paramName }) => {
      const key = paramName ?? name;  // ⭐ 使用 paramName 避免参数名冲突

      if (operator === 'c' || operator === 'dnc') {
        obj[key] = `%${value}%`;
      } else if (operator === 'eq' || operator === 'neq') {
        obj[key] = Array.isArray(value) ? value : [value];
      } else {
        obj[key] = value;
      }

      return obj;
    }, {}),
  };
}
```

最终参数对象：
```typescript
{
  country: ['CN'],      // 对应第一个 country 条件
  country1: ['US'],     // 对应第二个 country 条件（paramName 避免了冲突）
  // ...其他参数
}
```

#### 阶段 6：SQL 条件生成

**文件：** `src/lib/prisma.ts:74-106`

```typescript
function mapFilter(column: string, operator: string, name: string, type: string = '', paramName?: string) {
  const param = paramName ?? name;  // ⭐ 使用 paramName 作为参数占位符名
  const value = `{{${param}${type ? `::${type}` : ''}}}`;

  const table = SESSION_COLUMNS.includes(name) ? 'session' : 'website_event';

  switch (operator) {
    case 'eq':
      return `${table}.${column} = ANY(${value})`;
    case 'neq':
      return `${table}.${column} != ALL(${value})`;
    // ...
  }
}
```

生成的 SQL 条件片段：
```sql
AND session.country = ANY({{country}})
AND session.country != ALL({{country1}})
```

#### 完整链路示意图

```
UI 添加 2 个 country 筛选条件
  ↓
Filter[]: [{name:'country',op:'eq',val:'CN'}, {name:'country',op:'neq',val:'US'}]
  ↓ filtersArrayToObject()
URL: ?country=eq.CN&country1=neq.US
  ↓ parseRequest()
  1. Zod 校验 → { country: 'eq.CN' } （country1 被剥离）
  2. 后缀回填 → { country: 'eq.CN', country1: 'neq.US' }
  ↓ getRequestFilters()
filters: { country: 'eq.CN', country1: 'neq.US' }
  ↓ filtersObjectToArray()
Filter[]: [
  { name:'country', paramName:undefined, column:'country', op:'eq', val:['CN'] },
  { name:'country', paramName:'country1', column:'country', op:'neq', val:['US'] }
]
  ↓ getQueryParams() + mapFilter()
SQL: AND session.country = ANY({country}) AND session.country != ALL({country1})
```

---

## 3.5 分群筛选与页面临时筛选的合并规则

### 3.5.1 合并顺序与覆盖优先级

**核心代码：** `src/lib/request.ts:116,123-127,134-151`

```typescript
const filters = getRequestFilters(params);                         // 1. 先提取页面临时筛选（ad-hoc）
                                                                    //    ⭐ params 包含 segment/cohort 等控制参数
                                                                    //    ⭐ filters 只包含 FILTER_COLUMNS 中的过滤参数

if (params.segment) {                                               // 2. 从 params.segment（控制参数）读取分群 ID
  const segmentParams = (await getWebsiteSegment(websiteId, params.segment))?.parameters;
  Object.assign(filters, filtersArrayToObject(segmentParams.filters));  // 3. 分群筛选 MERGE INTO 临时筛选
  if (segmentParams.match) match = segmentParams.match;               // 4. 分群 match 覆盖临时 match
}

if (params.cohort) {                                                // 5. 从 params.cohort（控制参数）读取队列 ID
  // ... 队列处理 ...
  Object.assign(filters, { ...filtersArrayToObject(cohortFilters) }); // 6. 队列筛选最后合并（加前缀，无冲突）
}
```

**⚠️ 重要澄清：**
- `params.segment` 是**控制参数**，用于从数据库加载分群定义
- `segmentParams.filters` 才是分群的**实际筛选条件数组**，经 `filtersArrayToObject()` 转换后合并入 `filters`
- `segment` 字符串本身**从不**进入 `filters` 条件集合

**关键机制：** `Object.assign(target, source)` —— **source 的同 key 属性会覆盖 target**

| 合并顺序 | 来源 | 优先级 | 同 key 处理 |
|----------|------|--------|-------------|
| 第 1 层 | 页面临时筛选（URL params） | 低（被覆盖） | 作为合并基底 |
| 第 2 层 | 分群筛选（Segment DB） | 高（覆盖） | ⚠️ **同 key 完全覆盖临时筛选** |
| 第 3 层 | 队列筛选（Cohort DB） | 最高 | 加 `cohort_` 前缀，无冲突 |

### 3.5.2 match 参数的覆盖规则

**代码：** `src/lib/request.ts:118,129-131`

```typescript
let match = params?.match;                    // 初始为 URL 中的 match

if (segmentParams.match) {
  match = segmentParams.match;                // 分群有 match 则覆盖
}
```

| 临时筛选 match | 分群 match | 最终 match |
|---------------|-----------|-----------|
| `all` (默认) | `any` | `any`（分群覆盖） |
| `any` | `all` | `all`（分群覆盖） |
| `any` | undefined | `any`（分群无 match，保留临时） |
| undefined | `any` | `any`（分群覆盖） |

**重要：** 合并后所有筛选条件（临时 + 分群）共享同一个 `match` 逻辑，由分群的 `match` 决定（如果分群设置了的话）。

### 3.5.3 同名字段冲突的具体场景分析

#### 场景 1：单条件完全覆盖

**页面临时筛选：** `country=eq.CN`  
**分群筛选：** `[{name: 'country', op: 'neq', value: 'US'}]`

```
步骤 1: getRequestFilters(params)
  → filters = { country: 'eq.CN' }

步骤 2: filtersArrayToObject(segment.filters)
  → { country: 'neq.US' }  （segment 只有 1 个 country，从 0 开始编号）

步骤 3: Object.assign(filters, { country: 'neq.US' })
  → filters = { country: 'neq.US' }  ⚠️ 临时筛选的 country 被完全覆盖！
```

**最终 SQL：** `session.country != ALL({{country}})` —— 只有分群条件生效

---

#### 场景 2：多条件部分覆盖（数字后缀错位）

**页面临时筛选：** `country=eq.CN&country1=neq.US`（2 个 country 条件）  
**分群筛选：** `[{name: 'country', op: 'eq', value: 'JP'}]`（只有 1 个 country 条件）

```
步骤 1: filters = { country: 'eq.CN', country1: 'neq.US' }

步骤 2: segment 转换 → { country: 'eq.JP' }  （segment 只有 1 个，后缀从 0 开始）

步骤 3: Object.assign(filters, { country: 'eq.JP' })
  → filters = { 
       country: 'eq.JP',     // ⚠️ 被分群覆盖
       country1: 'neq.US'    // ✅ 保留自临时筛选（segment 没有 country1）
     }
```

**最终 Filter[]：**
```typescript
[
  { name: 'country', paramName: undefined, op: 'eq', value: ['JP'] },   // 来自分群
  { name: 'country', paramName: 'country1', op: 'neq', value: ['US'] }  // 来自临时筛选
]
```

**最终 SQL（match=all）：**
```sql
AND session.country = ANY({{country}})     -- JP（分群）
AND session.country != ALL({{country1}})   -- US（临时筛选）
```

**⚠️ 微妙之处：** `country1` 保留是因为分群的 `filtersArrayToObject` 只生成 `country`（无后缀），不生成 `country1`。如果分群也有 2 个 country 条件，那么 `country1` 也会被覆盖。

---

#### 场景 3：多条件完全覆盖（数字后缀对齐）

**页面临时筛选：** `country=eq.CN&country1=neq.US`  
**分群筛选：** `[{name:'country',op:'eq',val:'JP'}, {name:'country',op:'neq',val:'KR'}]`

```
步骤 1: filters = { country: 'eq.CN', country1: 'neq.US' }

步骤 2: segment 转换 → { country: 'eq.JP', country1: 'neq.KR' }  （2 个条件，后缀 0 和 1）

步骤 3: Object.assign(filters, { country: 'eq.JP', country1: 'neq.KR' })
  → filters = { 
       country: 'eq.JP',      // ⚠️ 覆盖
       country1: 'neq.KR'     // ⚠️ 覆盖
     }
```

**结果：** 临时筛选的 2 个 country 条件被完全清除，只有分群的条件生效。

---

#### 场景 4：不同字段无冲突合并

**页面临时筛选：** `browser=eq.Chrome`  
**分群筛选：** `[{name: 'country', op: 'eq', value: 'CN'}]`

```
步骤 1: filters = { browser: 'eq.Chrome' }
步骤 2: segment → { country: 'eq.CN' }
步骤 3: Object.assign → { browser: 'eq.Chrome', country: 'eq.CN' }  ✅ 两者都保留
```

**最终 SQL（match=all）：**
```sql
AND session.browser = ANY({{browser}})   -- 来自临时筛选
AND session.country = ANY({{country}})   -- 来自分群
```

### 3.5.4 合并后对 SQL 参数绑定的影响

合并发生在**扁平对象层**（`filters: Record<string, any>`），因此数字后缀在 key 中保留，后续 `filtersObjectToArray` 和 SQL 生成不受影响：

```
合并后的 filters 对象：
{ country: 'eq.JP', country1: 'neq.US', browser: 'eq.Chrome' }
  ↓ filtersObjectToArray()
[
  { name: 'country', paramName: undefined, value: ['JP'] },
  { name: 'country', paramName: 'country1', value: ['US'] },
  { name: 'browser', paramName: undefined, value: ['Chrome'] }
]
  ↓ getQueryParams()
{
  country: ['JP'],      // key = paramName ?? name → 'country'
  country1: ['US'],     // key = paramName ?? name → 'country1'
  browser: ['Chrome']   // key = 'browser'
}
  ↓ mapFilter()
SQL:
  session.country = ANY({{country}})
  session.country != ALL({{country1}})
  session.browser = ANY({{browser}})
```

**✅ 参数绑定正确性保证：** 由于 `paramName` 保留了原始 key（带数字后缀），即使 `name` 字段相同，SQL 参数占位符也不会冲突。

### 3.5.5 队列（Cohort）的特殊处理

**代码：** `src/lib/request.ts:140-143`

```typescript
const cohortFilters = cohortParams.filters.map(({ name, ...props }) => ({
  ...props,
  name: `cohort_${name}`,  // ⭐ 添加 cohort_ 前缀，彻底避免命名冲突
}));
```

队列筛选永远不会与临时筛选或分群筛选冲突，因为所有字段名都加上了 `cohort_` 前缀：
- `country` → `cohort_country`
- `browser` → `cohort_browser`

同时队列还有独立的 `cohort_match` 和 `cohort_actionName` 参数。

---

## 3.6 GET vs POST 入口下的筛选参数处理差异

### 3.6.1 两种请求入口概览

| 入口类型 | 典型场景 | 参数来源 | 路径示例 |
|----------|---------|---------|---------|
| **GET** | 统计面板、实时数据 | URL query string | `/api/websites/{id}/stats?country=eq.CN&country1=neq.US` |
| **POST** | 报表查询（漏斗、细分、留存等） | Request Body（`filters` 字段） | `POST /api/reports/funnel`  body: `{ filters: {...} }` |

### 3.6.2 Schema 校验与数字后缀处理对比

**核心差异代码：** `src/lib/request.ts:23-41`

```typescript
if (schema) {
  const isGet = request.method === 'GET';
  const rawQuery = query;
  const result = schema.safeParse(isGet ? query : body);  // GET 解析 query，POST 解析 body

  if (!result.success) {
    error = () => badRequest(z.treeifyError(result.error));
  } else if (isGet) {
    query = result.data;

    // ⭐ 仅 GET：重新添加被 Zod 剥离的带后缀参数
    for (const key of Object.keys(rawQuery)) {
      if (/\d+$/.test(key) && !(key in query)) {
        query[key] = rawQuery[key];
      }
    }
  } else {
    body = result.data;  // ❌ POST：没有后缀重新添加逻辑！
  }
}
```

### 3.6.3 GET 请求完整链路（数字后缀保留）

**URL：** `?country=eq.CN&country1=neq.US&segment=xxx`
**分群「Chrome 用户」：** `[{ name: 'browser', op: 'eq', value: 'Chrome' }]`

```
步骤 1: parseRequest(schema)
  rawQuery = { country: 'eq.CN', country1: 'neq.US', segment: 'xxx' }
  schema.safeParse(rawQuery)
    → Zod 校验：filterParams 中只有 'country'，没有 'country1'
    → result.data = { country: 'eq.CN', segment: 'xxx' }  （country1 被剥离）
  query = result.data

步骤 2: GET 分支后缀回填（request.ts:33-38）
  for (const key of Object.keys(rawQuery)) {
    if (/\d+$/.test(key) && !(key in query)) {  // key='country1' 匹配
      query[key] = rawQuery[key];  // 回填 country1
    }
  }
  → query = { country: 'eq.CN', country1: 'neq.US', segment: 'xxx' }  ✅ 完整保留

步骤 3: getQueryFilters(query, websiteId)
  params = query = { country: 'eq.CN', country1: 'neq.US', segment: 'xxx' }
  filters = getRequestFilters(params)
    → { country: 'eq.CN', country1: 'neq.US' }  ✅ segment 不在 FILTER_COLUMNS，不进入 filters

步骤 4: 加载并合并分群筛选（request.ts:123-127）
  if (params.segment) {  // ⭐ 从 params.segment 读取分群 ID（不是从 filters）
    segmentParams = getWebsiteSegment(websiteId, params.segment).parameters
    → { filters: [{ name: 'browser', op: 'eq', value: 'Chrome' }], match: 'all' }
  }
  Object.assign(filters, filtersArrayToObject(segmentParams.filters))
  → { country: 'eq.CN', country1: 'neq.US', browser: 'eq.Chrome' }  ✅ segment 从未进入 filters

步骤 5: 最终 Filter[]（3 个条件）
  [
    { name: 'country', op: 'eq', value: ['CN'] },
    { name: 'country', paramName: 'country1', op: 'neq', value: ['US'] },
    { name: 'browser', op: 'eq', value: ['Chrome'] }
  ]
```

### 3.6.4 POST 请求完整链路（数字后缀丢失）

**前端流程：**
1. URL：`?country=eq.CN&country1=neq.US&segment=xxx`
2. `useFilterParameters()` 从 URL 提取 → `{ country: 'eq.CN', country1: 'neq.US', segment: 'xxx' }`
3. `useResultQuery()` 发送 POST 请求：
   ```typescript
   post('/reports/funnel', {
     websiteId,
     type: 'funnel',
     filters,  // 包含 country1 和 segment
     parameters: { ... }
   })
   ```

**后端流程：**
```
步骤 1: parseRequest(reportResultSchema)
  body = {
    websiteId: 'xxx',
    type: 'funnel',
    filters: { country: 'eq.CN', country1: 'neq.US', segment: 'xxx' },
    parameters: { ... }
  }
  schema.safeParse(body)
    → reportResultSchema.filters = z.object({ ...filterParams })
    → filterParams 中只有 'country'，没有 'country1'
    → result.data.filters = { country: 'eq.CN', segment: 'xxx' }  （country1 被剥离！）
  body = result.data  ❌ POST 分支没有后缀回填逻辑

步骤 2: getQueryFilters(body.filters, websiteId)
  params = body.filters = { country: 'eq.CN', segment: 'xxx' }
  filters = getRequestFilters(params)
    → { country: 'eq.CN' }  ✅ segment 不在 FILTER_COLUMNS，不进入 filters（country1 已丢失）

步骤 3: 加载并合并分群筛选
  if (params.segment) {
    segmentParams = getWebsiteSegment(websiteId, params.segment).parameters
  }
  Object.assign(filters, filtersArrayToObject(segmentParams.filters))
  → { country: 'eq.CN', browser: 'eq.Chrome' }

步骤 4: 最终 Filter[]（只有 2 个条件！country1 丢失）
  [
    { name: 'country', op: 'eq', value: ['CN'] },
    { name: 'browser', op: 'eq', value: ['Chrome'] }
  ]
```

### 3.6.5 差异对比总结表

| 处理环节 | GET 请求 | POST 请求 | 差异影响 |
|----------|---------|----------|---------|
| Zod 校验对象 | `query`（单层） | `body.filters`（嵌套） | 校验层级不同 |
| 后缀参数剥离 | ✅ 是 | ✅ 是 | 相同 |
| 原始数据备份 | ✅ `rawQuery = query` | ❌ 无 `rawBody` 备份 | 无法回填的根源 |
| 后缀参数回填 | ✅ 有（遍历 `rawQuery` → `query`） | ❌ 无 | **核心差异** |
| 最终 `country1` 保留 | ✅ 保留 | ❌ 丢失 | 条件数量不同 |
| `getRequestFilters()` 输入 | 含后缀 | 不含后缀 | 结果不同 |
| 分群合并基底 | 完整临时筛选 | 残缺临时筛选 | **查询结果不一致** |

### 3.6.6 对分群 + 临时筛选组合查询的一致性影响

**⚠️ 关键问题：** GET 和 POST 对同一份筛选条件（URL 相同）会产生不同的查询结果

#### 对比示例（同 URL 不同入口）

**前置条件：**
- URL：`?country=eq.CN&country1=neq.US&segment=xxx`
- Segment「Chrome 用户」：`[{ name: 'browser', op: 'eq', value: 'Chrome' }]`, `match=all`

| 入口 | 有效条件 | 最终 SQL（match=all） | 查询结果 |
|------|---------|----------------------|---------|
| **GET /stats** | `country=CN` AND `country≠US` AND `browser=Chrome` | `session.country = ANY(ARRAY['CN']) AND session.country != ALL(ARRAY['US']) AND session.browser = ANY(ARRAY['Chrome'])` | 中国（非美国）的 Chrome 用户 |
| **POST /reports/funnel** | `country=CN` AND `browser=Chrome` | `session.country = ANY(ARRAY['CN']) AND session.browser = ANY(ARRAY['Chrome'])` | 中国的 Chrome 用户（包含美国用户） |

**一致性影响：**
1. **查询结果不一致**：同一页面的 GET 统计和 POST 报表显示不同的用户数量
2. **条件静默丢失**：无任何错误提示，`country1` 条件默默消失
3. **分群覆盖逻辑受影响**：如果分群也有 `country` 条件，POST 中临时筛选的 `country` 被覆盖后，`country1` 又丢失，可能导致所有 country 相关临时筛选全部失效
4. **match 逻辑影响**：如果丢失的是 OR 组合中的关键条件，可能导致逻辑完全改变

#### 极端场景（分群 + 多同名字段）

**临时筛选：** `country=eq.CN&country1=neq.US&country2=neq.JP&match=any`  
**分群：** `[{ name: 'country', op: 'eq', value: 'KR' }]`（1 个 country 条件）

| 入口 | 合并后条件 | 最终逻辑（match=any） |
|------|-----------|----------------------|
| **GET** | `country=KR`（分群覆盖）+ `country1≠US` + `country2≠JP` | `KR OR ≠US OR ≠JP` → 几乎所有用户 |
| **POST** | `country=KR`（分群覆盖） | `KR` → 仅韩国用户 |

**结果差异：** GET 返回几乎全部数据，POST 只返回韩国用户，结果天差地别。

### 3.6.7 根本原因分析

**Schema 定义问题：** `src/lib/schema.ts:45-71`

```typescript
export const filterParams = {
  country: z.string().optional(),   // 只定义了 'country'
  browser: z.string().optional(),
  // ... 其他字段
  // ❌ 缺少动态后缀支持：没有 country1, country2, browser1 等
};

export const reportResultSchema = z.intersection(
  z.object({
    websiteId: z.uuid(),
    filters: z.object({ ...filterParams }),  // ❌ 使用静态 filterParams 校验动态 filters
  }),
  reportTypeSchema,
);
```

**回填逻辑不完整：** `src/lib/request.ts:30-41`

```typescript
if (isGet) {
  query = result.data;
  // ✅ GET: rawQuery 保存了原始 query，回填被剥离的后缀参数
  for (const key of Object.keys(rawQuery)) {
    if (/\d+$/.test(key) && !(key in query)) {
      query[key] = rawQuery[key];
    }
  }
} else {
  body = result.data;  // ❌ POST：没有保存原始 body，也没有 body.filters 的后缀回填逻辑
  // ⚠️ POST 分支甚至没有原始 body 的副本，无法进行后缀回填
}
```

**POST 无法回填的更深层原因：**
1. POST 请求的原始数据在 `body` 变量中，经过 `schema.safeParse(body)` 后直接被覆盖为 `body = result.data`
2. 没有像 GET 那样保存 `rawBody = body` 的原始副本
3. 即使有原始副本，也需要递归遍历 `body.filters` 对象进行后缀回填，而不是简单遍历 `rawQuery`
4. 回填逻辑只实现了单层 `query` 对象的遍历，没有考虑嵌套的 `body.filters` 结构

---

## 四、URL 参数 → 后端查询编排

### 4.1 参数解析流程

#### 步骤 1：前端 → 后端 API 调用

**文件：** `src/components/hooks/useFilterParameters.ts`

`useFilterParameters()` 从 URL query 中提取筛选参数：

```typescript
return useMemo(() => {
  const filterParams: Record<string, any> = {};

  for (const key of Object.keys(query)) {
    const baseName = key.replace(/\d+$/, '');  // 去掉数字后缀
    if (FILTER_COLUMNS[baseName]) {
      filterParams[key] = query[key];         // 保留原始 key（带后缀）
    }
  }

  return {
    ...filterParams,
    search: query.search,
    segment: query.segment,
    cohort: query.cohort,
    excludeBounce: query.excludeBounce,
    match: query.match,
    page: query.page,
    pageSize: query.pageSize,
  };
}, [query]);
```

#### 步骤 2：后端解析请求

**文件：** `src/lib/request.ts:111-177`

`getQueryFilters()` 是核心编排函数，负责：
1. 解析日期范围
2. 提取普通筛选参数（含带后缀参数，POST 入口下可能已丢失）
3. **合并分群（Segment）筛选条件**（同 key 覆盖，详见 3.5 节）
4. **合并队列（Cohort）筛选条件**（加前缀，无冲突）

```typescript
export async function getQueryFilters(
  params: Record<string, any>,
  websiteId?: string,
): Promise<QueryFilters> {
  const dateRange = getRequestDateRange(params);
  const filters = getRequestFilters(params);

  let match = params?.match;

  if (websiteId) {
    // 合并分群筛选条件（⚠️ 同 key 覆盖临时筛选）
    if (params.segment) {
      const segmentParams = (await getWebsiteSegment(websiteId, params.segment))
        ?.parameters as Record<string, any>;

      Object.assign(filters, filtersArrayToObject(segmentParams.filters));

      if (segmentParams.match) {
        match = segmentParams.match;
      }
    }

    // 合并队列筛选条件（前缀 cohort_，无冲突）
    if (params.cohort) {
      const cohortParams = (await getWebsiteSegment(websiteId, params.cohort))
        ?.parameters as Record<string, any>;

      const { startDate, endDate } = parseDateRange(cohortParams.dateRange);

      const cohortFilters = cohortParams.filters.map(({ name, ...props }) => ({
        ...props,
        name: `cohort_${name}`,  // 添加 cohort_ 前缀区分
      }));

      cohortFilters.push({
        name: `cohort_${cohortParams.action.type}`,
        operator: OPERATORS.equals,
        value: cohortParams.action.value,
      });

      Object.assign(filters, {
        ...filtersArrayToObject(cohortFilters),
        cohort_startDate: startDate,
        cohort_endDate: endDate,
        ...(cohortParams.match && {
          cohort_match: cohortParams.match,
          cohort_actionName: `cohort_${cohortParams.action.type}`,
        }),
      });
    }
  }

  return { ...dateRange, ...filters, match, /* ...其他参数 */ };
}
```

### 4.2 参数值解析

**文件：** `src/lib/params.ts:4-27`

`parseFilterValue()` 解析 URL 参数值，提取操作符和值：

```typescript
export function parseFilterValue(param: any) {
  if (typeof param === 'string') {
    const operatorValues = Object.values(OPERATORS).join('|');
    const regex = new RegExp(`^(${operatorValues})\\.(.*)$`);

    const [, operator, value] = param.match(regex) || [];

    const resolvedOperator = operator || OPERATORS.equals;
    const resolvedValue = value ?? param;

    // eq/neq 操作符的值按逗号分隔为数组
    if (resolvedOperator === OPERATORS.equals || resolvedOperator === OPERATORS.notEquals) {
      return { operator: resolvedOperator, value: resolvedValue.split(',') };
    }

    return { operator: resolvedOperator, value: resolvedValue };
  }
  // ...
}
```

---

## 五、Filter → SQL 查询转换

### 5.1 分库适配：PostgreSQL vs ClickHouse

Umami 支持两种数据库后端，查询转换逻辑分别在：
- **PostgreSQL**: `src/lib/prisma.ts:74-150`
- **ClickHouse**: `src/lib/clickhouse.ts:73-140`

两者结构完全一致，仅 SQL 语法不同。

### 5.2 操作符 → SQL 映射（PostgreSQL）

**文件：** `src/lib/prisma.ts:74-106`

```typescript
function mapFilter(column: string, operator: string, name: string, type: string = '', paramName?: string) {
  const param = paramName ?? name;
  const value = `{{${param}${type ? `::${type}` : ''}}}`;
  
  // 确定表名：会话字段在 session 表，事件字段在 website_event 表
  const table = SESSION_COLUMNS.includes(name) ? 'session' : 'website_event';

  switch (operator) {
    case OPERATORS.equals:
      return `${table}.${column} = ANY(${value})`;
    case OPERATORS.notEquals:
      return `${table}.${column} != ALL(${value})`;
    case OPERATORS.contains:
      return `${table}.${column} ilike ${value}`;
    case OPERATORS.doesNotContain:
      return `${table}.${column} not ilike ${value}`;
    case OPERATORS.regex:
      return `${table}.${column} ~* ${value}`;
    case OPERATORS.notRegex:
      return `${table}.${column} !~* ${value}`;
    default:
      return '';
  }
}
```

### 5.3 AND/OR 逻辑编排

**文件：** `src/lib/prisma.ts:108-150`

`getFilterQuery()` 处理多条件的逻辑组合：

```typescript
function getFilterQuery(filters: Record<string, any>, options: QueryOptions = {}): string {
  const { isCohort, cohortMatch, cohortActionName } = options;
  const isOr = isCohort ? cohortMatch === 'any' : filters.match === 'any';
  const orClauses: string[] = [];
  const andClauses: string[] = [];

  filtersObjectToArray(filters, options).forEach(
    ({ name, column, operator, prefix = '', paramName }) => {
      if (column) {
        const clause = mapFilter(`${prefix}${column}`, operator, name, '', paramName);
        const isAlwaysAnd = name === 'eventType' || (isCohort && name === cohortActionName);

        // eventType 和 cohortActionName 永远用 AND 连接
        if (isAlwaysAnd) {
          andClauses.push(`and ${clause}`);
        } else if (isOr) {
          orClauses.push(clause);
        } else {
          andClauses.push(`and ${clause}`);
        }

        // referrer 特殊处理：排除自身域名
        if (name === 'referrer') {
          andClauses.push(
            `and (website_event.referrer_domain != regexp_replace(website_event.hostname, '^www.', '') or website_event.referrer_domain is null)`,
          );
        }
      }
    },
  );

  const parts: string[] = [];

  // OR 条件用括号包裹
  if (orClauses.length > 0) {
    parts.push(`and (\n  ${orClauses.join('\n  or ')}\n)`);
  }

  parts.push(...andClauses);

  return parts.join('\n');
}
```

### 5.4 查询参数值预处理

**文件：** `src/lib/prisma.ts:208-230`

`getQueryParams()` 对参数值进行预处理以适配 SQL：

```typescript
function getQueryParams(filters: Record<string, any>) {
  return {
    ...filters,
    ...filtersObjectToArray(filters).reduce((obj, { name, column, operator, value, paramName }) => {
      const key = paramName ?? name;

      if (([OPERATORS.contains, OPERATORS.doesNotContain] as Operator[]).includes(operator)) {
        obj[key] = `%${value}%`;  // 模糊查询加通配符
      } else if (([OPERATORS.equals, OPERATORS.notEquals] as Operator[]).includes(operator)) {
        obj[key] = Array.isArray(value) ? value : [value];  // 确保为数组
      } else {
        obj[key] = value;
      }

      return obj;
    }, {}),
  };
}
```

### 5.5 完整查询组装

**文件：** `src/lib/prisma.ts:232-253`

`parseFilters()` 组装所有查询组件：

```typescript
function parseFilters(filters: Record<string, any>, options?: QueryOptions) {
  // 判断是否需要 JOIN session 表
  const joinSession = Object.keys(filters).find(key => {
    const baseName = key.replace(/\d+$/, '');
    return ['referrer', ...SESSION_COLUMNS].includes(baseName);
  });

  // 分离队列筛选条件
  const cohortFilters = Object.fromEntries(
    Object.entries(filters).filter(([key]) => key.startsWith('cohort_')),
  );

  return {
    joinSessionQuery: options?.joinSession || joinSession
      ? `inner join session on website_event.session_id = session.session_id and website_event.website_id = session.website_id`
      : '',
    dateQuery: getDateQuery(filters),
    filterQuery: getFilterQuery(filters, options),
    queryParams: getQueryParams(filters),
    cohortQuery: getCohortQuery(cohortFilters),     // 队列子查询 JOIN
    excludeBounceQuery: getExcludeBounceQuery(filters),  // 排除跳出子查询 JOIN
  };
}
```

---

## 六、完整数据流示例

### 场景 1：创建分群「中国 Chrome 用户」

#### 步骤 1：UI 操作
1. 在 FilterEditForm 中添加字段 `country`，操作符 `等于`，值选择 `China`
2. 添加字段 `browser`，操作符 `等于`，值选择 `Chrome`
3. 匹配模式选择 `全部匹配 (all)`
4. 保存为分群，命名「中国 Chrome 用户」

#### 步骤 2：数据转换
```
UI 状态（Filter[]）
  ↓
[
  { name: 'country', operator: 'eq', value: 'CN' },
  { name: 'browser', operator: 'eq', value: 'Chrome' }
]
  ↓ filtersArrayToObject()
{
  country: 'eq.CN',
  browser: 'eq.Chrome',
  match: 'all'
}
  ↓ 保存到 segment 表 parameters 字段
{
  "filters": [
    { "name": "country", "operator": "eq", "value": "CN" },
    { "name": "browser", "operator": "eq", "value": "Chrome" }
  ],
  "match": "all"
}
```

#### 步骤 3：查询时应用分群
```
URL: /api/websites/{id}/stats?segment={segmentId}
  ↓ getQueryFilters()
  1. 读取 segment 表，获取 filters
  2. filtersArrayToObject() → { country: 'eq.CN', browser: 'eq.Chrome' }
  3. 合并到查询参数
  ↓ parseFilters()
  1. filtersObjectToArray() → Filter[]
  2. mapFilter() 逐个转换为 SQL 条件
     - country = ANY(ARRAY['CN'])
     - browser = ANY(ARRAY['Chrome'])
  3. 逻辑组合：AND 连接
  ↓ 最终 SQL
SELECT ...
FROM website_event
INNER JOIN session ON ...  -- 因为 browser 是 SESSION_COLUMN
WHERE website_event.website_id = {websiteId}
  AND website_event.created_at BETWEEN ...
  AND session.country = ANY(ARRAY['CN'])
  AND session.browser = ANY(ARRAY['Chrome'])
```

---

### 场景 2：分群 + 页面临时筛选同时存在（含冲突）

#### 前置条件
- 分群「中国用户」：`country=eq.CN`，`match=any`
- 页面临时筛选：`country=neq.US&os=eq.Windows`，`match=all`

#### 请求 URL
```
/api/websites/{id}/stats?country=neq.US&os=eq.Windows&match=all&segment={segmentId}
```

#### 后端处理流程
```
params = { country: 'neq.US', os: 'eq.Windows', match: 'all', segment: '{segmentId}' }
  ⭐ segment 在 params 中（控制参数），不在 filters 中（过滤参数）

步骤 1：getRequestFilters(params)
  → filters = { country: 'neq.US', os: 'eq.Windows' }  ✅ segment 未进入 filters
  → match = params.match = 'all'

步骤 2：读取 segment 表（通过 params.segment）
  → segmentParams.filters = [{name:'country', op:'eq', value:'CN'}]
  → segmentParams.match = 'any'

步骤 3：filtersArrayToObject(segmentParams.filters)
  → { country: 'eq.CN' }

步骤 4：Object.assign(filters, { country: 'eq.CN' })
  → filters = {
       country: 'eq.CN',     // ⚠️ 分群覆盖了临时筛选的 country
       os: 'eq.Windows'      // ✅ 临时筛选的 os 保留
     }

步骤 5：match 覆盖
  → match = 'any'（分群的 match 覆盖了临时筛选的 'all'）

步骤 6：filtersObjectToArray(filters)
  → [
      { name:'country', paramName:undefined, op:'eq', value:['CN'] },  // 来自分群
      { name:'os', paramName:undefined, op:'eq', value:['Windows'] }   // 来自临时筛选
    ]

步骤 7：getQueryParams()
  → {
      country: ['CN'],      // 分群值
      os: ['Windows'],      // 临时筛选值
      match: 'any'          // 分群的 match
    }

步骤 8：mapFilter() → SQL 条件
  session.country = ANY({{country}})
  session.os = ANY({{os}})

步骤 9：getFilterQuery() 逻辑组合（match='any'）
  and (
    session.country = ANY({{country}})
    or session.os = ANY({{os}})
  )
```

#### 最终 SQL
```sql
SELECT ...
FROM website_event
INNER JOIN session ON ...
WHERE website_event.website_id = {websiteId}
  AND website_event.created_at BETWEEN ...
  AND (
    session.country = ANY(ARRAY['CN'])      -- 分群条件
    OR session.os = ANY(ARRAY['Windows'])   -- 临时筛选条件
  )
```

**⚠️ 关键结果：**
1. 临时筛选的 `country != US` 被分群的 `country = CN` 完全覆盖，不复存在
2. 临时筛选的 `os = Windows` 保留，因为分群没有 os 筛选
3. `match` 逻辑从 `all`（AND）被改为 `any`（OR），由分群决定
4. 两个条件最终以 OR 组合，而非用户在临时筛选中设置的 AND

**⚠️ GET vs POST 一致性提醒：** 以上流程假设是 GET 请求。如果是 POST 报表请求（如漏斗、细分），且临时筛选包含带数字后缀的同名字段（如 `country1`），则在 Schema 校验后后缀字段会丢失，导致查询条件与 GET 入口不一致。详见 3.6 节。

---

## 七、关键文件索引

| 模块 | 文件路径 | 核心功能 |
|------|----------|----------|
| 类型定义 | `src/lib/types.ts` | Filter, QueryFilters, Operator 等类型 |
| 常量定义 | `src/lib/constants.ts` | OPERATORS, FILTER_COLUMNS, 字段分组 |
| 参数编解码 | `src/lib/params.ts` | filtersArrayToObject, filtersObjectToArray, parseFilterValue, isSearchOperator |
| 请求解析 | `src/lib/request.ts` | parseRequest（GET 后缀回填，POST 无）, getQueryFilters（分群/队列合并）, getRequestFilters |
| PG 查询转换 | `src/lib/prisma.ts` | mapFilter, getFilterQuery, parseFilters, getQueryParams |
| CH 查询转换 | `src/lib/clickhouse.ts` | 同上，ClickHouse 版本 |
| 字段元数据 | `src/components/hooks/useFields.ts` | 字段分组、标签定义 |
| 操作符标签 | `src/components/hooks/useOperatorLabels.ts` | 操作符枚举 → 显示文本映射 |
| 筛选 Hook | `src/components/hooks/useFilters.ts` | 操作符类型映射（typeFilters）、URL → Filter[] |
| 参数提取 | `src/components/hooks/useFilterParameters.ts` | 从 URL 提取筛选参数 |
| 报表查询 | `src/components/hooks/queries/useResultQuery.ts` | POST 报表请求构建（filters 取自 URL） |
| 字段筛选 UI | `src/components/input/FieldFilters.tsx` | 筛选条件列表 |
| 单条筛选 UI | `src/components/common/FilterRecord.tsx` | 单条筛选条件行（含操作符下拉逻辑） |
| 筛选编辑弹窗 | `src/components/input/FilterEditForm.tsx` | 完整筛选编辑器 |
| 分群编辑 | `src/app/(main)/websites/[websiteId]/segments/SegmentEditForm.tsx` | 分群保存表单 |
| 报表 API | `src/app/api/reports/funnel/route.ts` | 漏斗报表 POST 入口 |
| 报表 API | `src/app/api/reports/breakdown/route.ts` | 细分报表 POST 入口 |
| Schema 验证 | `src/lib/schema.ts` | segmentParamSchema, operatorParam, filterParams, reportResultSchema |
