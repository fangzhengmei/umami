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

### 1.5 分群（Segment）存储模型

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
2. 提取普通筛选参数（含带后缀参数）
3. **合并分群（Segment）筛选条件**
4. **合并队列（Cohort）筛选条件**

```typescript
export async function getQueryFilters(
  params: Record<string, any>,
  websiteId?: string,
): Promise<QueryFilters> {
  const dateRange = getRequestDateRange(params);
  const filters = getRequestFilters(params);

  let match = params?.match;

  if (websiteId) {
    // 合并分群筛选条件
    if (params.segment) {
      const segmentParams = (await getWebsiteSegment(websiteId, params.segment))
        ?.parameters as Record<string, any>;

      Object.assign(filters, filtersArrayToObject(segmentParams.filters));

      if (segmentParams.match) {
        match = segmentParams.match;
      }
    }

    // 合并队列筛选条件（前缀 cohort_）
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

### 场景：创建分群「中国 Chrome 用户」

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

## 七、关键文件索引

| 模块 | 文件路径 | 核心功能 |
|------|----------|----------|
| 类型定义 | `src/lib/types.ts` | Filter, QueryFilters, Operator 等类型 |
| 常量定义 | `src/lib/constants.ts` | OPERATORS, FILTER_COLUMNS, 字段分组 |
| 参数编解码 | `src/lib/params.ts` | filtersArrayToObject, filtersObjectToArray, parseFilterValue, isSearchOperator |
| 请求解析 | `src/lib/request.ts` | parseRequest（后缀回填）, getQueryFilters（分群/队列合并）, getRequestFilters |
| PG 查询转换 | `src/lib/prisma.ts` | mapFilter, getFilterQuery, parseFilters, getQueryParams |
| CH 查询转换 | `src/lib/clickhouse.ts` | 同上，ClickHouse 版本 |
| 字段元数据 | `src/components/hooks/useFields.ts` | 字段分组、标签定义 |
| 操作符标签 | `src/components/hooks/useOperatorLabels.ts` | 操作符枚举 → 显示文本映射 |
| 筛选 Hook | `src/components/hooks/useFilters.ts` | 操作符类型映射（typeFilters）、URL → Filter[] |
| 参数提取 | `src/components/hooks/useFilterParameters.ts` | 从 URL 提取筛选参数 |
| 字段筛选 UI | `src/components/input/FieldFilters.tsx` | 筛选条件列表 |
| 单条筛选 UI | `src/components/common/FilterRecord.tsx` | 单条筛选条件行（含操作符下拉逻辑） |
| 筛选编辑弹窗 | `src/components/input/FilterEditForm.tsx` | 完整筛选编辑器 |
| 分群编辑 | `src/app/(main)/websites/[websiteId]/segments/SegmentEditForm.tsx` | 分群保存表单 |
| Schema 验证 | `src/lib/schema.ts` | segmentParamSchema, operatorParam, filterParams |
