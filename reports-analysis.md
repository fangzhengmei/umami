# Umami 三大关键报表协作差异分析报告

## 概述

本文档分析 Umami 中三个核心报表（漏斗 Funnels、用户路径 Journeys、留存 Retention）从筛选条件、时间参数、SQL 查询到结果展示的完整协作流程，重点梳理共用逻辑和差异化设计，并探讨缓存策略和团队权限对报表的影响。

---

## 一、整体架构概览

### 1.1 数据流转管道

```
页面组件 → useResultQuery Hook → API Route → 权限校验 → 参数处理 → SQL 查询 → 结果格式化 → 前端展示
```

### 1.2 核心共用模块

| 模块 | 位置 | 功能 |
|------|------|------|
| `useResultQuery` | `src/components/hooks/queries/useResultQuery.ts` | 统一数据查询 Hook，处理查询参数和缓存 |
| `parseRequest` | `src/lib/request.ts` | 解析 API 请求，验证 Schema 和权限 |
| `getQueryFilters` | `src/lib/request.ts` | 统一处理筛选条件（日期范围、过滤器、分段、群组） |
| `setWebsiteDate` | `src/lib/request.ts` | 根据网站权限（团队/订阅）限制日期范围 |
| `canViewWebsite` | `src/permissions/website.ts` | 网站级别的查看权限校验 |
| `parseFilters` | ClickHouse/Prisma 查询模块 | SQL 级别的过滤条件生成 |

---

## 二、三个报表逐维对比

### 2.1 API 路由层对比

| 维度 | 漏斗 (Funnel) | 用户路径 (Journey) | 留存 (Retention) |
|------|-------------|------------------|----------------|
| **API 路径** | `/api/reports/funnel` | `/api/reports/journey` | `/api/reports/retention` |
| **权限校验** | `canViewWebsite` | `canViewWebsite` | `canViewWebsite` |
| **调用 setWebsiteDate** | ✅ 是 | ❌ 否 | ✅ 是 |
| **特殊参数处理** | 无 | 将 `eventType` 合并到 `filters` | 无 |

**关键差异点**：
- 用户路径报表不调用 `setWebsiteDate`，意味着它不受团队订阅数据保留期限限制
- 用户路径报表将 `eventType` 参数从 `parameters` 移到 `filters` 中处理

### 2.2 参数结构对比

#### 漏斗报表参数 (`FunnelParameters`)
```typescript
{
  startDate: Date,        // 开始日期
  endDate: Date,          // 结束日期
  window: number,         // 转化窗口（分钟）
  steps: Array<{          // 漏斗步骤
    type: string,         // 'path' 或 'event'
    value: string,        // 路径/事件值
    filters?: Array<{     // 步骤级筛选
      property: string,
      operator: string,
      value: string
    }>
  }>
}
```

#### 用户路径参数 (`JourneyParameters`)
```typescript
{
  startDate: Date,        // 开始日期  
  endDate: Date,          // 结束日期
  steps: number,          // 步骤数量
  startStep?: string,     // 起始步骤过滤
  endStep?: string,       // 结束步骤过滤
  view?: string,          // 视图类型（pages/events）
  eventType?: number      // 事件类型（移到 filters）
}
```

#### 留存报表参数 (`RetentionParameters`)
```typescript
{
  startDate: Date,        // 开始日期
  endDate: Date,          // 结束日期
  timezone?: string       // 时区
}
```

**参数差异分析**：

| 特性 | 漏斗 | 用户路径 | 留存 |
|------|------|--------|------|
| **时间参数** | startDate, endDate | startDate, endDate | startDate, endDate, timezone |
| **步骤配置** | 多步骤配置（路径/事件+筛选） | 仅步骤数量 + 起止过滤 | 无步骤配置 |
| **特有参数** | window（转化窗口） | startStep, endStep, view | timezone |
| **层级筛选** | 步骤级细粒度筛选 | 路径级筛选 | 无内置筛选 |

### 2.3 前端 Hook 调用对比

**调用方式**（三个报表都使用 `useResultQuery`）：

```typescript
// 漏斗
useResultQuery('funnel', { websiteId, ...parameters })

// 用户路径  
useResultQuery('journey', { 
  websiteId, 
  steps, 
  startStep, 
  endStep, 
  view,
  eventType 
})

// 留存
useResultQuery('retention', { websiteId, startDate, endDate })
```

**`useResultQuery` 内部共用逻辑**：
1. 自动注入日期参数（`startDate`, `endDate`, `timezone`, `unit`）
2. 自动注入全局筛选条件（通过 `useFilterParameters`）
3. 生成 `queryKey` 用于 React Query 缓存
4. 通过 `useApi.post` 发送请求

**关键差异**：
- 用户路径的 `eventType` 会在 API 层被移到 filters 中，这是一个特殊处理
- 漏斗参数包含复杂的嵌套结构（steps），但 queryKey 会正确序列化

### 2.4 SQL 查询逻辑对比

#### 共用查询模式

三个报表都遵循相同的查询抽象模式：

```typescript
export async function getXXX(
  websiteId: string, 
  parameters: XXXParameters, 
  filters: QueryFilters
) {
  return runQuery({
    [PRISMA]: () => relationalQuery(...args),    // PostgreSQL
    [CLICKHOUSE]: () => clickhouseQuery(...args), // ClickHouse
  });
}
```

#### 漏斗报表 SQL 特性

**核心逻辑**：多步骤渐进式过滤，每一步基于前一步的结果

```sql
-- ClickHouse 实现模式
WITH level0 AS (-- 基础事件集，应用全局筛选
  SELECT ... FROM website_event WHERE ...),
level1 AS (-- 第一步筛选
  SELECT * FROM level0 WHERE url_path/event_name = {param0}),
level2 AS (-- 第二步，基于第一步的会话+时间窗口
  SELECT DISTINCT ... FROM level1
  JOIN level0 ON level1.session_id = level0.session_id
  WHERE created_at BETWEEN level1.created_at 
    AND level1.created_at + INTERVAL {window} MINUTE
    AND ...)
-- 聚合各层结果
SELECT 1 as level, count(DISTINCT session_id) FROM level1
UNION ALL
SELECT 2 as level, count(DISTINCT session_id) FROM level2
...
```

**关键技术点**：
- 支持 `path`（页面浏览）和 `event`（自定义事件）两种步骤类型
- 步骤间通过 `session_id` + 时间窗口关联
- 每个步骤可独立设置事件数据筛选（`event_data` 子查询）
- 支持通配符匹配（`*` → `LIKE`）
- Prisma 版本使用 `EXISTS` 子查询，ClickHouse 版本使用 `IN`

#### 用户路径 SQL 特性

**核心逻辑**：会话内事件序列提取 + 路径频率统计

```sql
WITH events AS (
  SELECT DISTINCT
    visit_id,
    coalesce(nullIf(event_name, ''), url_path) AS "event",
    row_number() OVER (PARTITION BY visit_id ORDER BY created_at) AS event_number
  FROM website_event
  WHERE ...),
sequences AS (-- 按访问分组，提取第1/2/3...N步事件
  SELECT s.e1, s.e2, ..., s.e7, count(*) count
  FROM (
    SELECT visit_id,
      max(CASE WHEN event_number = 1 THEN "event" END) AS e1,
      max(CASE WHEN event_number = 2 THEN "event" END) AS e2,
      ...
    FROM events GROUP BY visit_id) s
  GROUP BY s.e1, s.e2, ..., s.e7)
-- 可选：起止步骤过滤 + 排序限制
SELECT * FROM sequences
WHERE e1 = {startStep}  -- 起始过滤
  AND (... OR e7 = {endStep})  -- 结束过滤
ORDER BY count DESC LIMIT 100
```

**关键技术点**：
- 使用窗口函数 `row_number()` 标记会话内事件顺序
- 通过 `max(CASE...)` 行转列提取各步骤事件
- 相邻重复事件在前端会被合并（`combineSequentialDuplicates`）
- 固定最多 7 步路径分析
- Prisma 版本多了 `joinSessionQuery` 关联查询

#### 留存报表 SQL 特性

**核心逻辑**：群组分析（Cohort Analysis）- 按首次访问日期分组 + 后续回访追踪

```sql
-- ClickHouse 版本
WITH cohort_items AS (
  SELECT
    min(toDate(created_at, timezone)) as cohort_date,
    session_id
  FROM website_event
  WHERE ...
  GROUP BY session_id),
user_activities AS (
  SELECT DISTINCT
    we.session_id,
    toInt32((toDate(we.created_at, timezone) - cohort_items.cohort_date) / 86400) as day_number
  FROM website_event we
  JOIN cohort_items ON we.session_id = cohort_items.session_id
  WHERE ...),
cohort_size AS (
  SELECT cohort_date, count(*) as visitors FROM cohort_items GROUP BY 1),
cohort_date AS (
  SELECT c.cohort_date, a.day_number, count(*) as visitors
  FROM user_activities a
  JOIN cohort_items c ON a.session_id = c.session_id
  GROUP BY 1, 2)
-- 最终留存矩阵
SELECT
  c.cohort_date as date,
  c.day_number as day,
  s.visitors,
  c.visitors as returnVisitors,
  c.visitors * 100 / s.visitors as percentage
FROM cohort_date c
JOIN cohort_size s ON c.cohort_date = s.cohort_date
WHERE c.day_number <= 31
ORDER BY 1, 2
```

**关键技术点**：
- 双重会话识别：`cohort_items` 定首次访问日期，`user_activities` 定后续活动日
- 时区感知：使用 `toDate(created_at, timezone)` 确保跨时区日期计算准确
- 留存天数计算：日期差值 / 86400（秒/天）
- 前端仅展示特定天数（1, 2, 3, 4, 5, 6, 7, 14, 21, 28）
- Prisma 版本使用数据库特定日期函数 `getDateSQL` / `getDayDiffQuery`

#### SQL 查询差异总结

| 维度 | 漏斗 | 用户路径 | 留存 |
|------|------|--------|------|
| **核心算法** | 多步渐进过滤 | 序列提取+频次统计 | 群组(Cohort)分析 |
| **关联键** | session_id + 时间窗口 | visit_id | session_id |
| **CTE 数量** | 2 + N（步骤数） | 2（events + sequences） | 4（cohort_items, user_activities, cohort_size, cohort_date） |
| **结果粒度** | 步骤级访客数 | 路径级出现次数 | 日期×天数留存率 |
| **结果行数** | 步骤数（固定） | 路径数量（最多100） | 群组数×天数（最多31天） |
| **时间窗口** | 分钟级转化窗口 | 整个会话内 | 天级留存周期 |

---

## 三、结果展示层差异

### 3.1 漏斗报表展示

**组件**：`Funnel.tsx`

**展示特点**：
1. 垂直步骤列表，带连接线表示流程
2. 每个步骤显示：步骤类型（页面/事件）图标、步骤值、操作符标签、筛选条件
3. 关键指标：访客数、流失数（相对上一步）、转化率（相对第一步）、流失率
4. 进度条可视化转化漏斗
5. 支持编辑功能（非分享页面时）

**数据处理**：
```typescript
// formatResults 函数计算
visitors = 当前步骤访客数
previous = 上一步访客数
dropped = previous - visitors
dropoff = 1 - visitors / previous      // 相对流失率
remaining = visitors / 第一步访客数    // 累计转化率
```

### 3.2 用户路径展示

**组件**：`Journey.tsx`

**展示特点**：
1. 横向列布局，每列代表一步
2. 节点表示该步骤的页面/事件，大小按访客数排序
3. 连线表示路径流转（向上/向下/平直三种样式）
4. 交互：点击节点高亮该节点出发的所有路径
5. 鼠标悬停显示流失率和转化率提示

**数据处理**：
- 后端返回路径列表 `{ e1, e2, ..., e7, count }`
- 前端合并连续重复项 `combineSequentialDuplicates`
- 计算各节点出现频次、路径流量分配
- 动态生成连线坐标和样式

### 3.3 留存报表展示

**组件**：`Retention.tsx`

**展示特点**：
1. 热力图矩阵布局（日期×天数）
2. 行：群组日期（按首次访问日期分组）+ 群组大小
3. 列：第 N 天留存率（仅展示 D1/D2/D3/D4/D5/D6/D7/D14/D21/D28）
4. 单元格颜色深浅表示留存率高低

**数据处理**：
- 后端返回完整的 31 天数据
- 前端按 `day=0` 提取群组基本信息
- 通过二次查找匹配各天留存记录
- 超出可用天数的单元格留空

---

## 四、筛选条件与参数处理共用逻辑

### 4.1 `getQueryFilters` 统一处理流程

```
输入参数
   ↓
┌─────────────────────────────────────┐
│ 1. getRequestDateRange              │
│    - 解析 startAt/endAt/timezone/unit│
│    - 验证 unit 有效性                │
└─────────────────────────────────────┘
   ↓
┌─────────────────────────────────────┐
│ 2. getRequestFilters                │
│    - 提取 FILTER_COLUMNS 中的参数   │
│    - 支持带编号的过滤器（browser1）  │
└─────────────────────────────────────┘
   ↓
┌─────────────────────────────────────┐
│ 3. setWebsiteDate (若有 websiteId)  │
│    - 团队订阅限制：回溯最多6个月     │
│    - 网站重置日期限制                │
└─────────────────────────────────────┘
   ↓
┌─────────────────────────────────────┐
│ 4. 分段(Segment)处理                │
│    - 加载分段定义                   │
│    - 合并分段筛选器                 │
│    - 应用 match 逻辑                │
└─────────────────────────────────────┘
   ↓
┌─────────────────────────────────────┐
│ 5. 群组(Cohort)处理                 │
│    - 前缀 cohort_ 区分群组筛选       │
│    - 设置 cohort_start/endDate      │
└─────────────────────────────────────┘
   ↓
┌─────────────────────────────────────┐
│ 6. 其他参数                        │
│    - excludeBounce                  │
│    - page/pageSize/orderBy          │
│    - search/compare                 │
└─────────────────────────────────────┘
   ↓
输出 QueryFilters 对象
```

### 4.2 三个报表对筛选条件的支持度

| 筛选类型 | 漏斗 | 用户路径 | 留存 | 说明 |
|---------|------|--------|------|------|
| **日期范围** | ✅ | ✅ | ✅ | 都支持 |
| **时区** | ✅ | ✅ | ✅ | 留存特别需要 |
| **设备/浏览器/OS等** | ✅ | ✅ | ✅ | 通过 parseFilters 注入 SQL |
| **分段(Segment)** | ✅ | ✅ | ✅ | 合并到 filters |
| **群组(Cohort)** | ✅ | ✅ | ✅ | cohort_ 前缀字段 |
| **excludeBounce** | ✅ | ✅ | ✅ | 排除跳出会话 |
| **步骤级筛选** | ✅ | ❌ | ❌ | 漏斗特有，每个步骤可独立设条件 |
| **路径起止过滤** | ❌ | ✅ | ❌ | 用户路径特有 |

---

## 五、缓存策略分析

### 5.1 React Query 前端缓存

**缓存键生成** (`useResultQuery.ts`)：

```typescript
queryKey: [
  'reports',
  {
    type,              // 'funnel' | 'journey' | 'retention'
    websiteId,
    startDate,
    endDate,
    timezone,
    unit,
    ...params,         // 报表特有参数（steps/window等）
    ...filters,        // 所有筛选条件
  },
]
```

**缓存特性**：
1. **细粒度缓存**：每个（报表类型+网站+时间范围+筛选组合）独立缓存
2. **自动失效**：当任何筛选条件变化时自动重新查询
3. **后台刷新**：React Query 默认的 stale-while-revalidate 机制
4. **组件间共享**：相同 queryKey 在不同组件间共享缓存数据

### 5.2 三个报表缓存特性对比

| 特性 | 漏斗 | 用户路径 | 留存 |
|------|------|--------|------|
| **缓存键复杂度** | 高（steps 数组嵌套） | 中 | 低 |
| **缓存命中率** | 低（配置多变） | 中（常用默认配置） | 高（仅时间参数） |
| **数据量大小** | 小（步骤数固定） | 中（最多100条路径） | 大（群组×31天） |
| **计算成本** | 高（多轮 JOIN） | 中（窗口函数） | 高（4层 CTE） |

### 5.3 后端缓存现状

**当前实现**：
- ❌ 无应用级缓存（Redis/Memcached）
- ❌ 无查询结果缓存
- ✅ 依赖数据库层缓存（PostgreSQL/ClickHouse 自带查询缓存）
- ✅ Prisma/ClickHouse 连接池复用

**潜在优化点**：
1. 高成本的留存和漏斗查询可加短期缓存（如 5 分钟）
2. 缓存键设计：`{reportType}:{websiteId}:{hash(filters+parameters)}`
3. 失效策略：数据采集后主动失效，或基于 TTL

---

## 六、团队权限与数据访问控制

### 6.1 `canViewWebsite` 权限校验流程

```
请求到达
   ↓
┌─────────────────────────────────────┐
│ 是管理员？                          │
│   → 是：直接通过 ✅                  │
└─────────────────────────────────────┘
   ↓ 否
┌─────────────────────────────────────┐
│ 是分享令牌访问？                     │
│   → 检查 shareToken 的 websiteId 匹配 │
└─────────────────────────────────────┘
   ↓ 不匹配
┌─────────────────────────────────────┐
│ 查询 entity（可能是 website/pixel/link）│
│   → 不存在：拒绝 ❌                  │
└─────────────────────────────────────┘
   ↓ 存在
┌─────────────────────────────────────┐
│ entity.userId 存在？                │
│   → 匹配当前用户 id：通过 ✅         │
└─────────────────────────────────────┘
   ↓ 不匹配
┌─────────────────────────────────────┐
│ entity.teamId 存在？                │
│   → 查询 teamUser 关系存在？        │
│     → 是：通过 ✅                    │
│     → 否：拒绝 ❌                    │
└─────────────────────────────────────┘
```

### 6.2 团队权限对报表的影响

#### 数据保留期限限制 (`setWebsiteDate`)

```typescript
// src/lib/request.ts
if (cloudMode && website && !website.teamId) {
  const account = await fetchAccount(website.userId);
  // 非团队个人用户且无订阅 → 最多回溯6个月
  if (!account?.hasSubscription) {
    data.startDate = maxDate(data.startDate, startOfMonth(subMonths(new Date(), 6)));
  }
}

// 网站重置日期限制
if (website?.resetAt) {
  data.startDate = maxDate(data.startDate, new Date(website?.resetAt));
}
```

**影响范围**：
- ✅ 漏斗报表：受限制（调用 setWebsiteDate）
- ❌ 用户路径：不受限（未调用）→ **潜在权限漏洞**
- ✅ 留存报表：受限制（调用 setWebsiteDate）

#### 团队成员角色权限

对于团队网站（`teamId` 存在）：
- **查看权限**：只要是团队成员即可查看所有报表（`canViewWebsite` 只检查 teamUser 存在性）
- **编辑权限**：需要 `websiteUpdate` 权限（用于保存/修改报表配置）
- **团队协作**：同一团队所有成员看到相同的报表配置和数据

### 6.3 权限与报表交互矩阵

| 报表 | 查看权限 | 数据保留限制 | 团队共享 | 报表配置共享 |
|-----|---------|------------|---------|-------------|
| **漏斗** | canViewWebsite | 6个月（非团队无订阅） | ✅ | ✅（保存在 report 表） |
| **用户路径** | canViewWebsite | ❌ 无限制（Bug?） | ✅ | ❌（无持久化配置） |
| **留存** | canViewWebsite | 6个月（非团队无订阅） | ✅ | ❌（无持久化配置） |

---

## 七、关键设计洞察与建议

### 7.1 设计优点

1. **高度模块化**：`useResultQuery` + API Route + SQL 三层抽象，复用率高
2. **双数据库兼容**：Prisma/ClickHouse 两套实现，切换透明
3. **统一筛选管道**：`getQueryFilters` 统一处理复杂筛选逻辑
4. **权限前置**：API 层首屏权限校验，避免后续无效计算
5. **前端缓存精细**：React Query 细粒度缓存，提升交互体验

### 7.2 潜在问题

1. **用户路径数据保留不一致**：未调用 `setWebsiteDate` 可能导致个人无订阅用户也能查看6个月前数据
2. **无后端缓存**：高成本查询（留存、多步漏斗）每次都重新计算，数据库压力大
3. **步骤级筛选复杂度**：漏斗的步骤级筛选生成复杂子查询，性能随步骤数下降
4. **用户路径固定7步限制**：SQL 硬编码 e1-e7，扩展性差
5. **缓存键未考虑用户权限**：不同权限用户（如管理员 vs 普通成员）可能看到不同数据但共享缓存

### 7.3 优化建议

#### 1. 修复用户路径权限漏洞
```typescript
// src/app/api/reports/journey/route.ts
export async function POST(request: Request) {
  // ... 现有代码 ...
  
  // 增加：统一日期限制
  await setWebsiteDate(websiteId, parameters);  // ← 加入这行
  
  const data = await getJourney(websiteId, parameters, queryFilters);
  return json(data);
}
```

#### 2. 增加查询结果缓存
```typescript
// 建议实现：lib/cache.ts 封装
async function getCachedReport(
  cacheKey: string,
  fetchFn: () => Promise<any>,
  ttl: number = 300 // 5分钟
) {
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);
  
  const result = await fetchFn();
  await redis.setex(cacheKey, ttl, JSON.stringify(result));
  return result;
}
```

#### 3. 留存报表预计算优化
- 每日离线计算各网站留存矩阵，查询时直接读取预计算结果
- 实时数据（当天）走实时查询，历史数据读预计算结果

#### 4. 漏斗查询性能优化
- 步骤数 > 3 时考虑使用渐进式计算（先算第一步，缓存结果后再算后续）
- ClickHouse 版本可尝试使用 `arrayJoin` + 数组函数优化路径匹配

#### 5. 用户路径动态步骤支持
- 移除硬编码的 7 步限制，动态生成 SQL 列
- 前端动态列数适配

---

## 附录：核心文件清单

| 文件 | 说明 |
|------|------|
| `src/app/(main)/websites/[websiteId]/(reports)/funnels/Funnel.tsx` | 漏斗前端组件 |
| `src/app/(main)/websites/[websiteId]/(reports)/journeys/Journey.tsx` | 用户路径前端组件 |
| `src/app/(main)/websites/[websiteId]/(reports)/retention/Retention.tsx` | 留存前端组件 |
| `src/app/api/reports/funnel/route.ts` | 漏斗 API 路由 |
| `src/app/api/reports/journey/route.ts` | 用户路径 API 路由 |
| `src/app/api/reports/retention/route.ts` | 留存 API 路由 |
| `src/queries/sql/reports/getFunnel.ts` | 漏斗 SQL 查询 |
| `src/queries/sql/reports/getJourney.ts` | 用户路径 SQL 查询 |
| `src/queries/sql/reports/getRetention.ts` | 留存 SQL 查询 |
| `src/components/hooks/queries/useResultQuery.ts` | 通用查询 Hook |
| `src/lib/request.ts` | 请求解析与筛选处理 |
| `src/permissions/website.ts` | 网站查看权限校验 |

---

**报告生成时间**：2025-01-14  
**分析版本**：Umami 当前代码库（基于提供的源码）
