# Umami Cohort 统计流程深度分析

## 一、概述

Umami 的 Cohort（队列分析）功能是基于 Segment 体系实现的，通过对用户群体进行分组追踪，分析用户在不同时间周期内的留存和行为模式。Cohort 本质上是一种特殊类型的 Segment（type: 'cohort'），存储在 `segment` 表中。

## 二、核心数据结构

### 2.1 Segment 表结构

Cohort 数据存储在 Prisma 的 `segment` 表中，核心字段如下：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | UUID | 唯一标识 |
| websiteId | UUID | 关联网站 ID |
| type | String | 类型：'segment' 或 'cohort' |
| name | String | 队列名称 |
| parameters | JSON | 队列配置参数 |
| createdAt | DateTime | 创建时间 |
| updatedAt | DateTime | 更新时间 |

### 2.2 Cohort Parameters 结构

在 `src/lib/schema.ts:303-321` 中定义了 Cohort 的参数结构：

```typescript
{
  filters: Array<{
    name: string;           // 过滤字段名
    operator: Operator;     // 操作符：eq/neq/c/dnc/re/nre 等
    value: string;          // 过滤值
  }>;
  match: 'all' | 'any';     // 过滤条件匹配方式
  dateRange: string;        // 统计窗口，如 '30day'
  action: {
    type: string;           // 动作类型：'path' 或 'event'
    value: string;          // 动作值：路径或事件名
  };
}
```

## 三、编辑与保存流程

### 3.1 前端编辑表单

编辑入口位于 `src/app/(main)/websites/[websiteId]/cohorts/CohortEditForm.tsx`，表单包含以下核心字段：

1. **名称 (name)**：队列的显示名称
2. **动作 (action)**：定义用户需要执行的目标动作
   - 类型选择：通过 `ActionSelect` 选择 'path' 或 'event'
   - 值输入：通过 `LookupField` 根据类型动态选择路径或事件
3. **日期范围 (dateRange)**：通过 `DateFilter` 选择统计窗口（如 7天、30天、90天等）
4. **过滤器 (filters)**：通过 `FieldFilters` 组件添加多维过滤条件
   - 可过滤字段：排除了 'path' 和 'event'（已在 action 中定义）
   - 支持字段：os、browser、device、country、region、city、language 等
   - 匹配方式：'all'（全部满足）或 'any'（任一满足）

### 3.2 保存流程

保存流程通过 `useUpdateQuery` Hook 实现：

**提交数据结构**：
```typescript
{
  name: string;
  type: 'cohort';
  parameters: {
    action: { type: string; value: string };
    dateRange: string;
    filters: Filter[];
    match?: 'all' | 'any';  // 非 'all' 时才保存
  };
}
```

**API 调用**：
- 新建：`POST /api/websites/{websiteId}/segments`
- 更新：`POST /api/websites/{websiteId}/segments/{segmentId}`

**后端处理**（`src/app/api/websites/[websiteId]/segments/route.ts`）：
1. 使用 Zod 验证请求体
2. 权限检查：`canUpdateWebsite`
3. 调用 `createSegment` 或 `updateSegment` 保存到数据库
4. 保存成功后触发 `touch('cohorts')` 使缓存失效

## 四、统计窗口机制

### 4.1 日期范围解析

统计窗口通过 `dateRange` 字符串定义，在 `src/lib/date.ts` 的 `parseDateRange` 函数中解析为具体的起止日期。

### 4.2 Cohort 窗口与查询窗口的关系

Cohort 有两个独立的时间维度：

1. **Cohort 定义窗口**：保存在 `parameters.dateRange` 中，用于筛选进入队列的用户
2. **报表查询窗口**：用户在报表页面选择的时间范围，用于分析该时间段内的用户行为

这两个窗口可以不同。例如：
- Cohort 定义窗口：最近 30 天（筛选在过去 30 天内完成指定动作的用户）
- 报表查询窗口：最近 7 天（分析这些用户在最近 7 天内的行为）

## 五、查询聚合流程

### 5.1 Cohort 参数注入

当查询带有 `cohort` 参数时，在 `src/lib/request.ts:getQueryFilters` 中进行参数转换：

```typescript
if (params.cohort) {
  // 1. 获取 Cohort 配置
  const cohortParams = (await getWebsiteSegment(websiteId, params.cohort))?.parameters;
  
  // 2. 解析 Cohort 的日期范围
  const { startDate, endDate } = parseDateRange(cohortParams.dateRange);
  
  // 3. 转换过滤器：添加 'cohort_' 前缀
  const cohortFilters = cohortParams.filters.map(({ name, ...props }) => ({
    ...props,
    name: `cohort_${name}`,
  }));
  
  // 4. 添加 action 作为额外过滤条件
  cohortFilters.push({
    name: `cohort_${cohortParams.action.type}`,
    operator: OPERATORS.equals,
    value: cohortParams.action.value,
  });
  
  // 5. 注入到查询参数
  Object.assign(filters, {
    ...filtersArrayToObject(cohortFilters),
    cohort_startDate: startDate,
    cohort_endDate: endDate,
    cohort_match: cohortParams.match,
    cohort_actionName: `cohort_${cohortParams.action.type}`,
  });
}
```

### 5.2 SQL 查询生成

在 `src/lib/prisma.ts:getCohortQuery` 或 `src/lib/clickhouse.ts:getCohortQuery` 中生成 Cohort 子查询：

```sql
join (
  select distinct website_event.session_id
  from website_event
  join session on session.session_id = website_event.session_id
    and session.website_id = website_event.website_id
  where website_event.website_id = {{websiteId}}
    and website_event.created_at between {{cohort_startDate}} and {{cohort_endDate}}
    -- Cohort 过滤条件（action + filters）
    and website_event.path = {{cohort_path}}  -- 示例
    and session.os = {{cohort_os}}            -- 示例
) cohort
on cohort.session_id = website_event.session_id
```

**关键逻辑**：
1. 子查询筛选出在 Cohort 时间窗口内满足条件的所有 `session_id`
2. 通过 `INNER JOIN` 将主查询限制为这些会话
3. 主查询的时间窗口可以独立于 Cohort 窗口

### 5.3 过滤器处理

在 `getFilterQuery` 中处理 Cohort 过滤器：

```typescript
function getFilterQuery(filters, options) {
  const { isCohort, cohortMatch, cohortActionName } = options;
  const isOr = isCohort ? cohortMatch === 'any' : filters.match === 'any';
  
  filtersObjectToArray(filters, options).forEach(({ name, column, operator }) => {
    if (isCohort) {
      // 去掉 'cohort_' 前缀，映射到实际列名
      column = FILTER_COLUMNS[name.slice('cohort_'.length)];
    }
    
    // action 条件始终使用 AND 连接
    const isAlwaysAnd = name === 'eventType' || (isCohort && name === cohortActionName);
    
    // 根据 match 决定使用 AND 还是 OR 连接
  });
}
```

## 六、报表展示

### 6.1 留存报表（Retention Report）

留存报表是 Cohort 分析的主要应用场景，位于 `src/app/(main)/websites/[websiteId]/(reports)/retention/Retention.tsx`。

**数据查询**：
通过 `useResultQuery('retention', { websiteId, startDate, endDate })` 获取数据。

**后端统计 SQL**（`src/queries/sql/reports/getRetention.ts`）：

```sql
WITH cohort_items AS (
  -- 步骤1：按首次访问日期对用户分组
  select
    min(date_trunc('day', website_event.created_at)) as cohort_date,
    website_event.session_id
  from website_event
  ${cohortQuery}  -- 注入 Cohort 过滤
  where website_event.website_id = {{websiteId}}
    and website_event.created_at between {{startDate}} and {{endDate}}
  group by website_event.session_id
),
user_activities AS (
  -- 步骤2：计算每个用户在不同日期的活跃度
  select distinct
    website_event.session_id,
    (date_trunc('day', created_at) - cohort_items.cohort_date) as day_number
  from website_event
  join cohort_items on website_event.session_id = cohort_items.session_id
  where website_id = {{websiteId}}
    and created_at between {{startDate}} and {{endDate}}
),
cohort_size as (
  -- 步骤3：计算每个队列的初始大小
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
where c.day_number <= 31
order by 1, 2
```

### 6.2 前端展示逻辑

```typescript
// 按 cohort_date 分组，整理成矩阵形式
const rows = data.reduce((arr, row) => {
  if (row.day === 0) {
    return arr.concat({
      date: row.date,
      visitors: row.visitors,
      records: [1, 2, 3, 4, 5, 6, 7, 14, 21, 28].map(day => 
        data.find(x => x.date === row.date && x.day === day)
      ).filter(n => n),
    });
  }
  return arr;
}, []);
```

展示为一个矩阵：
- **行**：按首次访问日期分组的 Cohort
- **列**：第 N 天的留存率
- **单元格**：该 Cohort 在第 N 天的留存百分比

## 七、架构设计要点

### 7.1 双数据库支持

Umami 同时支持 PostgreSQL 和 ClickHouse，Cohort 查询在两个数据库中都有实现：

| 数据库 | 实现文件 | 特点 |
|--------|----------|------|
| PostgreSQL | `src/lib/prisma.ts` | 通用实现，使用 `$queryRawUnsafe` |
| ClickHouse | `src/lib/clickhouse.ts` | 高性能实现，使用参数化查询 |

### 7.2 缓存机制

- 使用 React Query 进行前端缓存，`queryKey` 包含 `websiteId`、`cohortId`、`modified`
- `useModified('cohorts')` 提供版本戳，保存后触发缓存失效

### 7.3 权限控制

- 查看：`canViewWebsite`
- 创建/更新：`canUpdateWebsite`
- 删除：`canDeleteWebsite`

## 八、关键代码索引

| 功能 | 文件位置 |
|------|----------|
| 类型定义 | `src/lib/types.ts` |
| Schema 验证 | `src/lib/schema.ts:301-321` |
| Cohort 列表查询 | `src/components/hooks/queries/useWebsiteCohortsQuery.ts` |
| Cohort 详情查询 | `src/components/hooks/queries/useWebsiteCohortQuery.ts` |
| 编辑表单 | `src/app/(main)/websites/[websiteId]/cohorts/CohortEditForm.tsx` |
| 列表展示 | `src/app/(main)/websites/[websiteId]/cohorts/CohortsTable.tsx` |
| 参数转换 | `src/lib/request.ts:111-178` |
| 过滤器处理 | `src/lib/params.ts` |
| Prisma Cohort 查询 | `src/lib/prisma.ts:152-173` |
| ClickHouse Cohort 查询 | `src/lib/clickhouse.ts:142-161` |
| 留存报表 SQL | `src/queries/sql/reports/getRetention.ts` |
| 留存报表展示 | `src/app/(main)/websites/[websiteId]/(reports)/retention/Retention.tsx` |
| Segments API | `src/app/api/websites/[websiteId]/segments/route.ts` |
