# Umami Cohort 统计流程深度分析

## 一、概述

Umami 的 Cohort（队列分析）功能是基于 Segment 体系实现的，通过对用户群体进行分组追踪，分析用户在不同时间周期内的留存和行为模式。Cohort 本质上是一种特殊类型的 Segment（`type: 'cohort'`），存储在 `segment` 表中。

## 二、核心数据结构

### 2.1 Segment 表结构

Cohort 数据存储在 Prisma 的 `segment` 表中，核心字段如下：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 唯一标识 |
| websiteId | UUID | 关联网站 ID |
| type | String | 类型：`'segment'` 或 `'cohort'` |
| name | String | 队列名称 |
| parameters | JSON | 队列配置参数 |
| createdAt | DateTime | 创建时间 |
| updatedAt | DateTime | 更新时间 |

**代码依据**：`src/queries/prisma/segment.ts:51-53` 中的 `createSegment` 函数，使用 `Prisma.SegmentUncheckedCreateInput` 类型。

### 2.2 Cohort Parameters 结构

在 `src/lib/schema.ts:303-321` 中定义了 Cohort 的参数结构：

```typescript
// segmentParamSchema 定义
{
  filters: Array<{
    name: string;           // 过滤字段名（对应 FILTER_COLUMNS 的 key）
    operator: Operator;     // 操作符：eq/neq/c/dnc/re/nre 等
    value: string;          // 过滤值
  }>;
  match: 'all' | 'any';     // 过滤条件匹配方式（all=AND, any=OR）
  dateRange: string;        // 统计窗口，如 '30day'、'7day'
  action: {
    type: string;           // 动作类型：'path' 或 'event'
    value: string;          // 动作值：路径或事件名
  };
}
```

**代码依据**：`src/lib/schema.ts:303-321` 的 `segmentParamSchema`。

## 三、编辑与保存流程

### 3.1 前端编辑表单

编辑入口位于 `src/app/(main)/websites/[websiteId]/cohorts/CohortEditForm.tsx`，表单包含以下核心字段：

1. **名称 (name)**：队列的显示名称，必填，最大 200 字符
2. **动作 (action)**：定义用户需要执行的目标动作
   - 类型选择：通过 `ActionSelect` 选择 `'path'` 或 `'event'`
   - 值输入：通过 `LookupField` 根据类型动态选择路径或事件
3. **日期范围 (dateRange)**：通过 `DateFilter` 选择统计窗口（如 24 小时、7 天、30 天、90 天等）
4. **过滤器 (filters)**：通过 `FieldFilters` 组件添加多维过滤条件
   - 可过滤字段：排除了 `'path'` 和 `'event'`（已在 action 中定义）
   - 支持字段：os、browser、device、country、region、city、language、utmSource 等（见 `FILTER_COLUMNS`）
   - 匹配方式：`'all'`（全部满足）或 `'any'`（任一满足）

**代码依据**：`src/app/(main)/websites/[websiteId]/cohorts/CohortEditForm.tsx:73-143` 的表单渲染逻辑。

### 3.2 前端提交逻辑

通过 `useUpdateQuery` Hook 实现提交：

```typescript
// src/app/(main)/websites/[websiteId]/cohorts/CohortEditForm.tsx:42-67
const { mutateAsync, error, isPending, touch, toast } = useUpdateQuery(
  `/websites/${websiteId}/segments${cohortId ? `/${cohortId}` : ''}`,
  { type: 'cohort' },
);

const handleSubmit = async (formData: any) => {
  await mutateAsync(
    {
      ...formData,
      parameters: {
        ...formData.parameters,
        match: currentMatch !== 'all' ? currentMatch : undefined,  // 'all' 不保存
      },
    },
    {
      onSuccess: async () => {
        toast(t(messages.saved));
        touch('cohorts');  // 触发缓存失效
        onSave?.();
        onClose?.();
      },
    },
  );
};
```

**代码依据**：
- `src/components/hooks/queries/useUpdateQuery.ts:6-15`：`useUpdateQuery` 内部使用 `post` 方法
- 新建和更新使用**同一个 Hook**，通过 URL 路径是否包含 `cohortId` 区分

### 3.3 新建与更新接口分工

#### 新建接口：`POST /api/websites/{websiteId}/segments`

**文件**：`src/app/api/websites/[websiteId]/segments/route.ts:38-70`

```typescript
// Schema 校验（严格）
const schema = z.object({
  type: segmentTypeParam,           // 'segment' | 'cohort'
  name: z.string().max(200),
  parameters: segmentParamSchema,   // ⚠️ 严格校验 parameters 结构
});

// 权限检查
if (!(await canUpdateWebsite(auth, websiteId))) {
  return unauthorized();
}

// 自动生成 UUID 后创建
const result = await createSegment({
  id: uuid(),
  websiteId,
  type,
  name,
  parameters,
} as any);
```

**特点**：
- 使用 `segmentParamSchema` 严格校验 `parameters` 的结构
- `parameters` 字段本身**必填**（schema 无 `.optional()`），但内部 `filters`、`match`、`dateRange`、`action` 均为 `.optional()`
- 允许传入空对象 `parameters: {}`，但不能省略 `parameters` 字段
- 自动生成 UUID
- 权限：`canUpdateWebsite`

#### 更新接口：`POST /api/websites/{websiteId}/segments/{segmentId}`

**文件**：`src/app/api/websites/[websiteId]/segments/[segmentId]/route.ts:33-69`

```typescript
// Schema 校验（宽松）
const schema = z.object({
  type: segmentTypeParam,
  name: z.string().max(200),
  parameters: anyObjectParam,       // ⚠️ 任意 JSON 对象，不校验结构
});

// ⚠️ 先鉴权，后检查存在性（安全设计：避免越权探测资源是否存在）
if (!(await canUpdateWebsite(auth, websiteId))) {
  return unauthorized();
}

const segment = await getWebsiteSegment(websiteId, segmentId);
if (!segment) {
  return notFound();
}

// 更新
const result = await updateSegment(segmentId, {
  type,
  name,
  parameters,
} as any);
```

**特点**：
- 使用 `anyObjectParam` 宽松校验 `parameters`（`z.record(z.string(), z.any())`）
- **先鉴权后查存在性**：防止未授权用户通过 404/403 状态码差异探测 segment 是否存在
- 使用 `POST` 方法而非 `PUT/PATCH`
- 权限：`canUpdateWebsite`

#### 新建 vs 更新 对比表

| 维度 | 新建接口 | 更新接口 |
|------|----------|----------|
| URL | `POST /segments` | `POST /segments/{id}` |
| Schema 校验 | `segmentParamSchema`（严格） | `anyObjectParam`（宽松） |
| 存在性检查 | 不需要 | 必须检查 |
| ID 生成 | 服务端自动生成 UUID | 使用 URL 中的 ID |
| 权限 | `canUpdateWebsite` | `canUpdateWebsite` |

**代码依据**：
- 新建：`src/app/api/websites/[websiteId]/segments/route.ts:38-70`
- 更新：`src/app/api/websites/[websiteId]/segments/[segmentId]/route.ts:33-69`

### 3.4 删除接口

**文件**：`src/app/api/websites/[websiteId]/segments/[segmentId]/route.ts:71-96`

```typescript
export async function DELETE(request, { params }) {
  // 权限检查
  if (!(await canDeleteWebsite(auth, websiteId))) {
    return unauthorized();
  }
  
  // 存在性检查
  const segment = await getWebsiteSegment(websiteId, segmentId);
  if (!segment) {
    return notFound();
  }
  
  await deleteSegment(segmentId);
  return ok();
}
```

**特点**：
- 权限：`canDeleteWebsite`（比更新权限更高）
- 必须先检查 segment 是否存在

## 四、统计窗口机制

### 4.1 日期范围解析

统计窗口通过 `dateRange` 字符串定义（如 `'30day'`、`'7day'`），在 `src/lib/date.ts` 的 `parseDateRange` 函数中解析为具体的起止日期。

### 4.2 双时间维度设计

Cohort 有两个独立的时间维度，可以灵活组合：

1. **Cohort 定义窗口**：保存在 `parameters.dateRange` 中，用于筛选进入队列的用户
   - 例如：最近 30 天内访问过 `/pricing` 页面的用户

2. **报表查询窗口**：用户在报表页面选择的时间范围，用于分析该时间段内的用户行为
   - 例如：分析这些用户在最近 7 天内的留存情况

**示例组合**：
- Cohort 定义窗口：最近 30 天（筛选用户群）
- 报表查询窗口：最近 7 天（分析行为）
- 结果：显示过去 30 天内每天加入的用户，在后续 7 天内的留存率

## 五、筛选器 URL 参数到报表查询的完整链路

### 5.1 链路总览

```
URL 查询参数
    ↓
useNavigation() → query
    ↓
useFilterParameters() → filters
    ↓
useResultQuery() → 组装请求体
    ↓
POST /api/reports/{type}
    ↓
getQueryFilters() → 解析并注入 cohort 参数
    ↓
parseFilters() → 生成 SQL 片段
    ↓
getRetention() → 执行统计查询
    ↓
返回结果 → 前端渲染
```

### 5.2 步骤 1：URL 参数提取

**文件**：`src/components/hooks/useFilterParameters.ts:5-29`

```typescript
export function useFilterParameters() {
  const { query } = useNavigation();  // 从 URL 读取所有查询参数

  return useMemo(() => {
    const filterParams: Record<string, any> = {};

    // 提取所有在 FILTER_COLUMNS 中定义的过滤参数
    for (const key of Object.keys(query)) {
      const baseName = key.replace(/\d+$/, '');  // 处理 browser1, os2 等带数字后缀的参数
      if (FILTER_COLUMNS[baseName]) {
        filterParams[key] = query[key];
      }
    }

    return {
      ...filterParams,
      search: query.search,
      segment: query.segment,
      cohort: query.cohort,           // ⚠️ Cohort ID 从 URL 获取
      excludeBounce: query.excludeBounce,
      match: query.match,
      page: query.page,
      pageSize: query.pageSize,
    };
  }, [query]);
}
```

**支持的 URL 参数格式**：
- 基础过滤：`?os=eq.Windows&country=eq.US`
- 多值同字段：`?os=eq.Windows&os1=eq.macOS`
- Cohort 过滤：`?cohort=xxxx-xxxx-xxxx`

**代码依据**：`src/components/hooks/useFilterParameters.ts:5-29`

### 5.3 步骤 2：报表查询组装

**文件**：`src/components/hooks/queries/useResultQuery.ts:6-46`

```typescript
export function useResultQuery(type, params, options) {
  const { websiteId, ...parameters } = params;
  const { post, useQuery } = useApi();
  const { startDate, endDate, timezone, unit } = useDateParameters();  // 从 URL/状态读取日期
  const filters = useFilterParameters();                               // 从 URL 读取筛选器

  return useQuery({
    queryKey: ['reports', { type, websiteId, startDate, endDate, ...filters }],
    queryFn: () =>
      post(`/reports/${type}`, {
        websiteId,
        type,
        filters,           // 筛选器参数透传给后端
        parameters: {
          startDate,
          endDate,
          timezone,
          unit,
          ...parameters,
        },
      }),
    enabled: !!type,
    ...options,
  });
}
```

**代码依据**：`src/components/hooks/queries/useResultQuery.ts:6-46`

### 5.4 步骤 3：后端参数解析

**文件**：`src/lib/request.ts:111-178`

`getQueryFilters` 函数负责解析前端传来的筛选器参数，并处理 Cohort 注入：

```typescript
export async function getQueryFilters(params, websiteId) {
  // 1. 解析日期范围
  const dateRange = getRequestDateRange(params);
  // 2. 提取普通筛选器
  const filters = getRequestFilters(params);
  
  let match = params?.match;

  if (websiteId) {
    await setWebsiteDate(websiteId, dateRange);

    // 处理 Segment
    if (params.segment) {
      const segmentParams = (await getWebsiteSegment(websiteId, params.segment))?.parameters;
      Object.assign(filters, filtersArrayToObject(segmentParams.filters));
      if (segmentParams.match) match = segmentParams.match;
    }

    // ⚠️ 处理 Cohort（核心逻辑）
    if (params.cohort) {
      // 3.1 从数据库读取 Cohort 配置
      const cohortParams = (await getWebsiteSegment(websiteId, params.cohort))?.parameters;
      
      // 3.2 解析 Cohort 的日期范围
      const { startDate, endDate } = parseDateRange(cohortParams.dateRange);
      
      // 3.3 转换过滤器：添加 'cohort_' 前缀，避免与主查询过滤器冲突
      const cohortFilters = cohortParams.filters.map(({ name, ...props }) => ({
        ...props,
        name: `cohort_${name}`,
      }));
      
      // 3.4 将 action 也转换为过滤条件
      cohortFilters.push({
        name: `cohort_${cohortParams.action.type}`,
        operator: OPERATORS.equals,
        value: cohortParams.action.value,
      });
      
      // 3.5 注入到查询参数中
      Object.assign(filters, {
        ...filtersArrayToObject(cohortFilters),  // 转换为 URL 参数字符串格式
        cohort_startDate: startDate,
        cohort_endDate: endDate,
        ...(cohortParams.match && {
          cohort_match: cohortParams.match,
          cohort_actionName: `cohort_${cohortParams.action.type}`,
        }),
      });
    }

    if (params.excludeBounce) {
      Object.assign(filters, { excludeBounce: true });
    }
  }

  return {
    ...dateRange,
    ...filters,
    match,
    // ... 分页排序参数
  };
}
```

**代码依据**：`src/lib/request.ts:111-178`

### 5.4 步骤 4：过滤器数组与对象转换

**文件**：`src/lib/params.ts:42-90`

```typescript
// filtersArrayToObject: 将数组格式的过滤器转换为 URL 参数格式
export function filtersArrayToObject(filters: Filter[]) {
  const nameCounts: Record<string, number> = {};
  return filters.reduce((obj, filter) => {
    const { name, operator, value } = filter;
    const count = nameCounts[name] ?? 0;
    const key = count === 0 ? name : `${name}${count}`;  // 处理同字段多值
    nameCounts[name] = count + 1;
    
    // 格式：{ os: 'eq.Windows', os1: 'eq.macOS' }
    obj[key] = `${operator}.${Array.isArray(value) ? value.join(',') : value}`;
    return obj;
  }, {});
}

// filtersObjectToArray: 反向转换，用于 SQL 生成
export function filtersObjectToArray(filters, options) {
  return Object.keys(filters).reduce((arr, key) => {
    const baseName = key.replace(/\d+$/, '');
    // ... 解析 operator 和 value
    return arr.concat({ name: baseName, column, operator, value });
  }, []);
}
```

**代码依据**：`src/lib/params.ts:42-90`

## 六、查询聚合流程

### 6.1 SQL 生成 - Cohort 子查询

**文件**：`src/lib/prisma.ts:152-173`（PostgreSQL 版本）

```typescript
function getCohortQuery(filters) {
  if (!filters || Object.keys(filters).length === 0) return '';

  const cohortMatch = filters.cohort_match;
  const cohortActionName = filters.cohort_actionName;

  // 生成 Cohort 过滤器的 SQL 片段（isCohort: true）
  const filterQuery = getFilterQuery(filters, { 
    isCohort: true, 
    cohortMatch, 
    cohortActionName 
  });

  return `join (
    select distinct website_event.session_id
    from website_event
    join session on session.session_id = website_event.session_id
      and session.website_id = website_event.website_id
    where website_event.website_id = {{websiteId}}
      and website_event.created_at between {{cohort_startDate}} and {{cohort_endDate}}
      ${filterQuery}  -- Cohort 过滤条件（action + filters）
  ) cohort
  on cohort.session_id = website_event.session_id`;
}
```

**代码依据**：
- PostgreSQL：`src/lib/prisma.ts:152-173`
- ClickHouse：`src/lib/clickhouse.ts:142-161`

### 6.2 SQL 生成 - 过滤器处理

**文件**：`src/lib/prisma.ts:108-150`

```typescript
function getFilterQuery(filters, options) {
  const { isCohort, cohortMatch, cohortActionName } = options;
  const isOr = isCohort ? cohortMatch === 'any' : filters.match === 'any';
  const orClauses: string[] = [];
  const andClauses: string[] = [];

  filtersObjectToArray(filters, options).forEach(({ name, column, operator }) => {
    if (isCohort) {
      // 去掉 'cohort_' 前缀，映射到实际数据库列名
      column = FILTER_COLUMNS[name.slice('cohort_'.length)];
    }

    if (column) {
      const clause = mapFilter(column, operator, name);
      
      // ⚠️ action 条件和 eventType 始终使用 AND 连接
      const isAlwaysAnd = name === 'eventType' || (isCohort && name === cohortActionName);

      if (isAlwaysAnd) {
        andClauses.push(`and ${clause}`);
      } else if (isOr) {
        orClauses.push(clause);
      } else {
        andClauses.push(`and ${clause}`);
      }
    }
  });

  // 组装 SQL
  const parts: string[] = [];
  if (orClauses.length > 0) {
    parts.push(`and (\n  ${orClauses.join('\n  or ')}\n)`);
  }
  parts.push(...andClauses);

  return parts.join('\n');
}
```

**关键逻辑**：
- `isAlwaysAnd` 条件确保 action 不会被 OR 逻辑影响
- 普通过滤器根据 `match` 参数决定使用 AND 或 OR
- Cohort 过滤器使用独立的 `cohort_match` 参数

**代码依据**：`src/lib/prisma.ts:108-150`

### 6.3 SQL 生成 - 完整查询组装

**文件**：`src/lib/prisma.ts:232-253`

```typescript
function parseFilters(filters, options) {
  // 分离 Cohort 过滤器和普通过滤器
  const cohortFilters = Object.fromEntries(
    Object.entries(filters).filter(([key]) => key.startsWith('cohort_')),
  );

  return {
    joinSessionQuery: /* ... */,
    dateQuery: getDateQuery(filters),
    filterQuery: getFilterQuery(filters, options),       // 主查询过滤器
    queryParams: getQueryParams(filters),                // 参数绑定
    cohortQuery: getCohortQuery(cohortFilters),          // Cohort 子查询
    excludeBounceQuery: getExcludeBounceQuery(filters),
  };
}
```

**代码依据**：`src/lib/prisma.ts:232-253`

## 七、报表展示 - 留存报表完整链路

### 7.1 页面入口

**文件**：`src/app/(main)/websites/[websiteId]/(reports)/retention/RetentionPage.tsx:8-21`

```typescript
export function RetentionPage({ websiteId }) {
  const { dateRange: { startDate } } = useDateRange();  // 从 URL 读取月份

  // 自动转换为当月第一天到最后一天
  const monthStartDate = startOfMonth(startDate);
  const monthEndDate = endOfMonth(startDate);

  return (
    <Column gap>
      <WebsiteControls websiteId={websiteId} allowDateFilter={false} allowMonthFilter />
      <Retention websiteId={websiteId} startDate={monthStartDate} endDate={monthEndDate} />
    </Column>
  );
}
```

**注意**：留存报表使用月份粒度，自动将选择的日期转换为当月范围。

**代码依据**：`src/app/(main)/websites/[websiteId]/(reports)/retention/RetentionPage.tsx:8-21`

### 7.2 报表 API 入口

**文件**：`src/app/api/reports/retention/route.ts:7-26`

```typescript
export async function POST(request: Request) {
  // 1. 验证请求体
  const { auth, body, error } = await parseRequest(request, reportResultSchema);
  
  // 2. 权限检查
  if (!(await canViewWebsite(auth, websiteId))) {
    return unauthorized();
  }

  // 3. 解析筛选器（含 Cohort 注入）
  const filters = await getQueryFilters(body.filters, websiteId);
  // 4. 应用网站日期限制（如重置日期、数据保留期限）
  const parameters = await setWebsiteDate(websiteId, body.parameters);

  // 5. 执行统计
  const data = await getRetention(websiteId, parameters, filters);

  return json(data);
}
```

**代码依据**：`src/app/api/reports/retention/route.ts:7-26`

### 7.3 留存统计 SQL

**文件**：`src/queries/sql/reports/getRetention.ts:29-101`（PostgreSQL 版本）

```sql
WITH cohort_items AS (
  -- 步骤1：按首次访问日期对用户（session）分组
  select
    min(date_trunc('day', website_event.created_at)) as cohort_date,
    website_event.session_id
  from website_event
  ${cohortQuery}  -- ⚠️ 这里注入 Cohort 子查询，筛选符合条件的 session
  ${joinSessionQuery}
  where website_event.website_id = {{websiteId}}
    and website_event.created_at between {{startDate}} and {{endDate}}
    ${filterQuery}
  group by website_event.session_id
),
user_activities AS (
  -- 步骤2：计算每个用户在不同日期的活跃度（去重）
  select distinct
    website_event.session_id,
    (date_trunc('day', created_at)::date - cohort_items.cohort_date::date) as day_number
  from website_event
  join cohort_items on website_event.session_id = cohort_items.session_id
  where website_id = {{websiteId}}
    and created_at between {{startDate}} and {{endDate}}
),
cohort_size as (
  -- 步骤3：计算每个队列的初始大小（第0天用户数）
  select cohort_date, count(*) as visitors
  from cohort_items
  group by 1
),
cohort_date as (
  -- 步骤4：计算每个队列在各天的活跃用户数
  select c.cohort_date, a.day_number, count(*) as visitors
  from user_activities a
  join cohort_items c on a.session_id = c.session_id
  group by 1, 2
)
-- 最终结果：计算留存率
select
  c.cohort_date as date,
  c.day_number as day,
  s.visitors,
  c.visitors as returnVisitors,
  c.visitors::float * 100 / s.visitors as percentage
from cohort_date c
join cohort_size s on c.cohort_date = s.cohort_date
where c.day_number <= 31  -- 只显示31天内的数据
order by 1, 2
```

**代码依据**：`src/queries/sql/reports/getRetention.ts:29-101`

### 7.4 前端数据处理与展示

**文件**：`src/app/(main)/websites/[websiteId]/(reports)/retention/Retention.tsx:28-119`

```typescript
// 将扁平数据转换为矩阵格式
const rows = data?.reduce((arr, row) => {
  if (row.day === 0) {
    return arr.concat({
      date: row.date,
      visitors: row.visitors,
      // 预填充每一天的留存数据
      records: [1, 2, 3, 4, 5, 6, 7, 14, 21, 28].map(day => 
        data.find(x => x.date === row.date && x.day === day)
      ).filter(n => n),
    });
  }
  return arr;
}, []);

// 渲染为矩阵
// 行：按 cohort_date 分组（如 2024-01-01, 2024-01-02, ...）
// 列：第1天、第2天、...、第28天
// 单元格：留存百分比
```

**展示形式**：
```
+------------+--------+--------+--------+-----+
| Cohort     | Day 1  | Day 2  | Day 3  | ... |
+------------+--------+--------+--------+-----+
| 2024-01-01 | 65.2%  | 42.8%  | 31.5%  | ... |
| 2024-01-02 | 68.1%  | 45.3%  |        | ... |
+------------+--------+--------+--------+-----+
```

**代码依据**：`src/app/(main)/websites/[websiteId]/(reports)/retention/Retention.tsx:28-119`

## 八、架构设计要点

### 8.1 双数据库支持

Umami 同时支持 PostgreSQL 和 ClickHouse，Cohort 查询在两个数据库中都有独立实现：

| 数据库 | 实现文件 | 特点 |
|--------|----------|------|
| PostgreSQL | `src/lib/prisma.ts` | 通用实现，使用 `$queryRawUnsafe` + `{{param}}` 占位符 |
| ClickHouse | `src/lib/clickhouse.ts` | 高性能实现，使用原生参数化查询 `{param:Type}` |

**查询路由**：`src/lib/db.ts` 中的 `runQuery` 根据配置自动选择数据库。

### 8.2 缓存机制

- **前端缓存**：使用 React Query，`queryKey` 包含 `websiteId`、`cohortId`、`modified` 版本戳
- **缓存失效**：`useModified('cohorts')` 提供版本戳，保存后调用 `touch('cohorts')` 使缓存失效
- **数据一致性**：所有查询共享同一个 `modified` 状态，确保列表和详情同步更新

**代码依据**：`src/components/hooks/useModified.ts`

### 8.3 权限控制

| 操作 | 权限函数 | 说明 |
|------|----------|------|
| 查看列表/详情 | `canViewWebsite` | 网站查看权限 |
| 创建/更新 | `canUpdateWebsite` | 网站编辑权限 |
| 删除 | `canDeleteWebsite` | 网站删除权限（更高权限） |

**代码依据**：`src/permissions/index.ts`

### 8.4 数据安全

- SQL 注入防护：使用参数化查询，不拼接字符串
- 权限检查在 API 层执行，不是前端
- Cohort ID 是 UUID，难以枚举
- 所有输入通过 Zod 验证

## 九、关键代码索引

| 功能 | 文件位置 |
|------|----------|
| 类型定义 | `src/lib/types.ts` |
| Schema 验证 | `src/lib/schema.ts:293-321` |
| Cohort 列表查询 | `src/components/hooks/queries/useWebsiteCohortsQuery.ts` |
| Cohort 详情查询 | `src/components/hooks/queries/useWebsiteCohortQuery.ts` |
| 编辑表单 | `src/app/(main)/websites/[websiteId]/cohorts/CohortEditForm.tsx` |
| 列表展示 | `src/app/(main)/websites/[websiteId]/cohorts/CohortsTable.tsx` |
| URL 参数提取 | `src/components/hooks/useFilterParameters.ts` |
| 报表查询 Hook | `src/components/hooks/queries/useResultQuery.ts` |
| 后端参数解析 | `src/lib/request.ts:111-178` |
| 过滤器转换 | `src/lib/params.ts` |
| Prisma Cohort SQL 生成 | `src/lib/prisma.ts:152-173` |
| ClickHouse Cohort SQL 生成 | `src/lib/clickhouse.ts:142-161` |
| 留存报表统计 SQL | `src/queries/sql/reports/getRetention.ts` |
| 留存报表展示 | `src/app/(main)/websites/[websiteId]/(reports)/retention/Retention.tsx` |
| 新建 Segment API | `src/app/api/websites/[websiteId]/segments/route.ts:38-70` |
| 更新 Segment API | `src/app/api/websites/[websiteId]/segments/[segmentId]/route.ts:33-69` |
| 留存报表 API | `src/app/api/reports/retention/route.ts` |

## 十、易错点对照清单

| 序号 | 错误说法 | 正确代码事实 | 证据位置 |
|------|----------|--------------|----------|
| 1 | 更新接口先检查存在性再鉴权 | **先鉴权后查存在性**：先调用 `canUpdateWebsite`，检查通过后才查询 segment 是否存在。这是安全设计，防止未授权用户通过 HTTP 状态码差异（404 vs 403）探测 segment 是否存在。 | `src/app/api/websites/[websiteId]/segments/[segmentId]/route.ts:52-60` |
| 2 | 创建接口的 `parameters` 字段可选，可省略 | **`parameters` 字段必填**：schema 定义为 `parameters: segmentParamSchema`（无 `.optional()`），但内部所有子字段（filters、match、dateRange、action）均为 `.optional()`。允许 `parameters: {}`，但不能不传该字段。 | `src/app/api/websites/[websiteId]/segments/route.ts:42-46`、`src/lib/schema.ts:303-321` |
| 3 | Cohort 过滤器和主查询过滤器使用相同的 `match` 参数 | **各自独立**：Cohort 使用 `cohort_match`，主查询使用 `match`。两者互不影响。 | `src/lib/prisma.ts:108-150`、`src/lib/request.ts:384-388` |
| 4 | Cohort 的 action 条件会受 `match: 'any'` 影响 | **action 始终用 AND 连接**：代码中 `isAlwaysAnd = name === cohortActionName`，确保 action 条件不会被 OR 逻辑影响，保证队列定义的准确性。 | `src/lib/prisma.ts:108-150`（第 497 行） |
| 5 | Cohort 定义窗口和报表查询窗口必须一致 | **两个窗口完全独立**：Cohort 定义窗口保存在 `parameters.dateRange`，报表查询窗口来自 URL 参数。可以用"过去 30 天访问过 /pricing 的用户"分析"他们在过去 7 天的留存"。 | `src/lib/request.ts:358-389` |
| 6 | 新建和更新使用不同的 HTTP 方法（POST vs PUT） | **都用 POST**：无论新建还是更新，统一使用 `POST` 方法，通过 URL 路径是否包含 ID 区分操作。 | `src/app/api/websites/[websiteId]/segments/route.ts`、`[segmentId]/route.ts` |
| 7 | 新建接口对 `parameters` 的校验和更新一样宽松 | **新建更严格**：新建用 `segmentParamSchema`（结构校验），更新用 `anyObjectParam`（任意 JSON）。 | `src/app/api/websites/[websiteId]/segments/route.ts:45` vs `[segmentId]/route.ts:40` |
| 8 | 删除权限和更新权限相同 | **删除权限更高**：删除用 `canDeleteWebsite`，更新用 `canUpdateWebsite`。 | `src/app/api/websites/[websiteId]/segments/[segmentId]/route.ts:83` vs `52` |
| 9 | Cohort 的 `match: 'all'` 会保存到数据库 | **'all' 不保存**：前端代码 `match: currentMatch !== 'all' ? currentMatch : undefined`，只有非 'all' 时才保存。 | `src/app/(main)/websites/[websiteId]/cohorts/CohortEditForm.tsx:85` |
| 10 | Cohort 过滤器直接拼接到主查询 WHERE 条件 | **用子查询 JOIN**：Cohort 条件在子查询中筛选出符合条件的 `session_id`，然后通过 INNER JOIN 限制主查询。这样 Cohort 条件只影响"哪些用户"，不影响"这些用户的哪些行为"。 | `src/lib/prisma.ts:152-173` |

---

**文档版本**：v1.2（事实校准版）
**修订历史**：
- v1.0：初版，覆盖 Cohort 全流程
- v1.1：修正接口分工差异，补充完整链路
- v1.2：事实校准
  - 修正更新接口鉴权与存在性检查顺序（先鉴权后查存在性）
  - 修正创建接口 parameters 字段必填性（字段必填，子字段可选）
  - 新增易错点对照清单（10 条常见误解 vs 代码事实）
