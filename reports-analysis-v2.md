# Umami 三大报表协作差异深度分析报告（V2 - 校正版）

## 概述

本文档基于源码的完整链路追踪，校正了关键细节，深入分析 Umami 中三个核心报表（漏斗 Funnels、用户路径 Journeys、留存 Retention）从前端参数、API处理、筛选管道、SQL查询到结果展示的全链路协作机制，重点梳理：
1. **用户路径连续重复节点合并的真实发生位置**
2. **时间限制链路完整追踪**
3. **订阅期限、团队权限对各报表的真实影响差异**

---

## 一、架构总览：通用数据管道

### 1.1 标准数据流（漏斗/留存为代表）

```
前端 useResultQuery
    ↓（注入 startDate/endDate/timezone/unit + filters）
API Route
    ├─→ parseRequest（验证 Schema）
    ├─→ canViewWebsite（权限校验）
    ├─→ setWebsiteDate(body.parameters) 【关键1：参数级日期限制】
    └─→ getQueryFilters(body.filters, websiteId)
        ├─→ getRequestDateRange（解析日期）
        ├─→ getRequestFilters（解析筛选条件）
        ├─→ setWebsiteDate(dateRange)  【关键2：筛选级日期限制（双保险）】
        ├─→ 分段(Segment)处理
        ├─→ 群组(Cohort)处理
        └─→ excludeBounce 处理
    ↓
SQL 查询层(relationalQuery/clickhouseQuery)
    ├─→ parseFilters（生成 SQL WHERE 条件）
    ├─→ 报表特定CTE逻辑
    └─→ 结果格式化（parseResult）
    ↓
前端展示
```

### 1.2 核心模块位置速查

| 模块 | 文件路径 | 功能 |
|------|---------|------|
| `useResultQuery` | `src/components/hooks/queries/useResultQuery.ts` | 统一查询Hook，自动注入日期参数 |
| `parseRequest` | `src/lib/request.ts` | 请求解析、Schema验证 |
| `getQueryFilters` | `src/lib/request.ts:111` | 统一筛选管道，包含日期限制 |
| `setWebsiteDate` | `src/lib/request.ts:92` | 基于订阅/团队的数据保留期限制 |
| `canViewWebsite` | `src/permissions/website.ts:7` | 查看权限校验（管理员/分享/团队成员） |
| `parseFilters` | `src/lib/prisma.ts:232` / `src/lib/clickhouse.ts` | SQL筛选条件生成 |

---

## 二、关键校正：用户路径重复节点合并位置

### 2.1 校正结论

**❌ 原错误结论**：相邻重复事件在前端合并  
**✅ 实际结论**：相邻重复事件在 **后端 SQL 查询层的结果格式化阶段** 合并

### 2.2 完整证据链

**源码位置**：`src/queries/sql/reports/getJourney.ts:256-274`

```typescript
// 第256-268行：重复项合并函数（后端）
function combineSequentialDuplicates(array: any) {
  if (array.length === 0) return array;

  const result = [array[0]];

  for (let i = 1; i < array.length; i++) {
    if (array[i] !== array[i - 1]) {
      result.push(array[i]);
    }
  }

  return result;
}

// 第270-274行：结果格式化（Prisma和ClickHouse均调用）
function parseResult(data: any) {
  return data.map(({ e1, e2, e3, e4, e5, e6, e7, count }) => ({
    items: combineSequentialDuplicates([e1, e2, e3, e4, e5, e6, e7]),
    count: +Number(count),
  }));
}
```

**调用链确认**：
- Prisma版：第143行 → `.then(parseResult)`
- ClickHouse版：第253行 → `.then(parseResult)`

### 2.3 前端路径展示逻辑（无合并）

**源码位置**：`src/app/(main)/websites/[websiteId]/(reports)/journeys/Journey.tsx`

前端仅做以下处理（无任何合并逻辑）：
1. 按列分组展示节点
2. 计算节点间连线（up/down/flat）
3. 点击节点高亮路径
4. 鼠标悬停显示流失/转化率

---

## 三、时间限制链路完整追踪

### 3.1 通用时间限制机制：setWebsiteDate

**源码位置**：`src/lib/request.ts:92-109`

```typescript
export async function setWebsiteDate(websiteId: string, data: Record<string, any>) {
  const website = await fetchWebsite(websiteId);
  const cloudMode = !!process.env.CLOUD_MODE;

  // 规则1：非团队个人网站且无订阅 → 最多回溯6个月
  if (cloudMode && website && !website.teamId) {
    const account = await fetchAccount(website.userId);
    if (!account?.hasSubscription) {
      data.startDate = maxDate(data.startDate, startOfMonth(subMonths(new Date(), 6)));
    }
  }

  // 规则2：网站有重置日期 → 不能早于重置日期
  if (website?.resetAt) {
    data.startDate = maxDate(data.startDate, new Date(website?.resetAt));
  }

  return data;
}
```

**两个限制规则**：
| 限制类型 | 触发条件 | 限制逻辑 |
|---------|---------|---------|
| 订阅期限 | Cloud模式 + 非团队网站 + 无订阅 | `startDate` 不早于6个月前 |
| 网站重置 | 网站设置了 `resetAt` | `startDate` 不早于重置日期 |

---

### 3.2 三大报表时间链路逐案分析

#### ▶️ 报表1：漏斗（Funnel）

**API源码**：`src/app/api/reports/funnel/route.ts`

```typescript
// 第20行：对 parameters 应用日期限制
const parameters = await setWebsiteDate(websiteId, body.parameters);

// 第21行：对 filters 独立应用日期限制（getQueryFilters内部调用setWebsiteDate）
const filters = await getQueryFilters(body.filters, websiteId);

// 第23行：传入查询
const data = await getFunnel(websiteId, parameters as FunnelParameters, filters);
```

**SQL层参数流向**（`getFunnel.ts:47-54`）：
```typescript
const { startDate, endDate, window, steps } = parameters; // 已受限
const { filterQuery, ..., queryParams } = parseFilters({
  ...filters,       // 筛选条件中的日期也已受限（双重保险）
  websiteId,
  startDate,        // 使用 parameters 中的受限日期
  endDate,
});
```

**✅ 漏斗时间限制完整链路**：
```
前端选择时间范围
    ↓
parameters.startDate ─→ setWebsiteDate() ──┐
                                          ├─→ 双重限制保障
filters.startDate ───→ getQueryFilters() ─→ setWebsiteDate() ─┘
    ↓
SQL WHERE created_at BETWEEN {{startDate}} AND {{endDate}}
```

---

#### ▶️ 报表2：留存（Retention）

**API源码**：`src/app/api/reports/retention/route.ts`

```typescript
// 第20行：对 filters 应用日期限制
const filters = await getQueryFilters(body.filters, websiteId);

// 第21行：对 parameters 独立应用日期限制
const parameters = await setWebsiteDate(websiteId, body.parameters);

// 第23行：传入查询
const data = await getRetention(websiteId, parameters as RetentionParameters, filters);
```

**SQL层参数流向**（`getRetention.ts:34-44`）：
```typescript
const { startDate, endDate, timezone } = parameters; // 已受限
const { filterQuery, ..., queryParams } = parseFilters({
  ...filters,       // 筛选条件中的日期也已受限
  websiteId,
  startDate,        // 使用 parameters 中的受限日期
  endDate,
  timezone,
});
```

**✅ 留存时间限制完整链路**：  
与漏斗完全一致 → **双重限制保障**

---

#### ▶️ 报表3：用户路径（Journey）

**API源码**：`src/app/api/reports/journey/route.ts`

```typescript
// ❌ 第25行前：未对 parameters 调用 setWebsiteDate
const { websiteId, parameters, filters } = body;
const { eventType } = parameters;

// ... 权限检查 ...

// eventType 特殊处理
if (eventType) {
  filters.eventType = eventType;
}

// 第25行：仅对 filters 应用日期限制
const queryFilters = await getQueryFilters(filters, websiteId);

// 第27行：未受限的 parameters + 受限的 filters 传入查询
const data = await getJourney(websiteId, parameters, queryFilters);
```

**SQL层参数流向**（`getJourney.ts:39-51`）：
```typescript
const { startDate, endDate, steps, startStep, endStep } = parameters; // ⚠️ 未受限！
const { filterQuery, ..., queryParams } = parseFilters({
  ...filters,
  websiteId,
  startDate,        // ❌ 使用未受限的 parameters.startDate
  endDate,          // ❌ 使用未受限的 parameters.endDate
});
```

**❌ 用户路径时间限制链路（有缺陷）**：
```
前端选择时间范围
    ↓
parameters.startDate ───────────────────────→ 未经过 setWebsiteDate()！
    ↓
filters.startDate ───→ getQueryFilters() ─→ setWebsiteDate() ✓
    ↓
parseFilters({ ...filters, startDate: parameters.startDate })
    ↓
⚠️ SQL 实际使用的是未受限日期！
```

---

### 3.3 时间限制链路对比总结表

| 维度 | 漏斗（Funnel） | 留存（Retention） | 用户路径（Journey） |
|------|-------------|-----------------|-------------------|
| API调用 `setWebsiteDate(parameters)` | ✅ 是（第20行） | ✅ 是（第21行） | ❌ **否（缺失）** |
| `getQueryFilters` 内部限制 | ✅ 是 | ✅ 是 | ✅ 是 |
| **双重限制保障** | ✅ 是 | ✅ 是 | ❌ 否（只有一重） |
| SQL实际使用日期来源 | parameters.startDate（已受限） | parameters.startDate（已受限） | parameters.startDate（**未受限**） |
| filters中日期是否起作用 | ❌ 不起作用（被parameters覆盖） | ❌ 不起作用（被parameters覆盖） | ❌ 不起作用（被parameters覆盖） |

---

## 四、订阅期限与团队权限的影响差异

### 4.1 权限与数据保留期的关系矩阵

| 场景 | 触发条件 | 数据保留期限制 | 受影响报表 |
|------|---------|---------------|-----------|
| **个人免费用户** | Cloud模式 + `website.teamId = null` + `account.hasSubscription = false` | 最多回溯**6个月** | 漏斗✅、留存✅、用户路径❌ |
| **团队成员** | `website.teamId != null` | **无6个月限制**（团队功能本身付费） | 全部不受限 |
| **个人付费用户** | Cloud模式 + 无teamId + hasSubscription = true | **无6个月限制** | 全部不受限 |
| **网站已重置** | `website.resetAt` 有值 | 不能早于重置日期 | 漏斗✅、留存✅、用户路径❌ |
| **管理员** | `user.isAdmin = true` | **无限制**（权限绕过） | 全部不受限 |
| **分享链接访问** | shareToken 匹配网站ID | **无限制**（分享令牌绕过） | 全部不受限 |

### 4.2 用户路径"特权"的技术根源

**为什么用户路径不受6个月限制？**

```
根源：API route.ts 遗漏调用 setWebsiteDate(parameters)

漏斗API：    第20行 await setWebsiteDate(websiteId, body.parameters)
留存API：    第21行 await setWebsiteDate(websiteId, body.parameters)
用户路径API： 缺失此行！

后果：
  1. parameters.startDate 直接来自前端选择，未被截断
  2. 虽然 getQueryFilters 内部对 filters 调用了 setWebsiteDate
  3. 但 SQL 层实际使用的是 parameters.startDate（未受限）
  4. filters 中的受限日期被 parseFilters 的入参覆盖
```

**代码对比**：
```typescript
// funnel/route.ts 第20行（正确）
const parameters = await setWebsiteDate(websiteId, body.parameters);

// retention/route.ts 第21行（正确）
const parameters = await setWebsiteDate(websiteId, body.parameters);

// journey/route.ts ❌ 无此行！
```

### 4.3 团队权限的影响范围

**团队网站 vs 个人网站对比**：

| 维度 | 团队网站（teamId存在） | 个人网站（无teamId） |
|------|-----------------------|---------------------|
| 6个月数据限制 | ❌ 不触发（所有报表） | ✅ 触发（漏斗/留存生效，用户路径实际不受限） |
| 成员查看权限 | 所有团队成员均可查看 | 仅创建者本人可查看 |
| 报表配置共享 | ✅ 漏斗/目标等配置团队共享 | ❌ 仅个人可见 |

---

## 五、SQL查询层核心逻辑对比

### 5.1 参数与筛选架构对比

| 维度 | 漏斗 | 用户路径 | 留存 |
|------|------|--------|------|
| **核心参数解构** | `{startDate, endDate, window, steps}` | `{startDate, endDate, steps, startStep, endStep}` | `{startDate, endDate, timezone}` |
| **steps含义** | 数组（每步的类型/值/筛选） | 数字（最多几步） | 无 |
| **特殊筛选** | 每步独立的 event_data 筛选 | 起始/结束路径过滤 | 无内置筛选 |
| **时区依赖** | 无（用原始 timestamp） | 无 | 强依赖（群组日期截断） |

### 5.2 核心算法与CTE结构

#### 🔹 漏斗：渐进式会话过滤

```sql
-- PostgreSQL/ClickHouse 通用模式
WITH level1 AS ( -- 第一步：筛选满足第一个条件的会话
  SELECT DISTINCT session_id, created_at
  FROM website_event
  WHERE [第一步过滤条件] AND created_at BETWEEN ...
),
level2 AS ( -- 第二步：在window时间内，满足第二个条件的会话
  SELECT DISTINCT we.session_id, we.created_at
  FROM level1 l
  JOIN website_event we ON l.session_id = we.session_id
  WHERE we.created_at BETWEEN l.created_at AND l.created_at + [window]
    AND [第二步过滤条件]
),
... -- 后续步骤依此类推
-- 汇总各层数量
SELECT 1 as level, count(*) FROM level1
UNION ALL
SELECT 2 as level, count(*) FROM level2
...
```

**关键特性**：
- 会话级别的渐进式过滤（"越来越少"）
- 每步之间通过 `session_id + 时间窗口(window)` 关联
- 支持每步独立的 `event_data` 属性过滤
- 计算成本随步骤数线性增加

#### 🔹 用户路径：序列提取 + 频次统计

```sql
WITH events AS (
  SELECT DISTINCT
    visit_id,
    COALESCE(NULLIF(event_name, ''), url_path) AS "event",
    ROW_NUMBER() OVER (PARTITION BY visit_id ORDER BY created_at) AS event_number
  FROM website_event
  WHERE created_at BETWEEN ...
),
sequences AS ( -- 按访问分组，提取第1/2/3...N步的事件
  SELECT s.e1, s.e2, ..., s.e7, count(*) count
  FROM (
    SELECT visit_id,
      MAX(CASE WHEN event_number = 1 THEN "event" END) AS e1,
      MAX(CASE WHEN event_number = 2 THEN "event" END) AS e2,
      ...
    FROM events GROUP BY visit_id
  ) s
  GROUP BY s.e1, s.e2, ..., s.e7
)
-- 可选起止过滤 + 排序限流
SELECT * FROM sequences
WHERE e1 = {startStep} -- 起始路径过滤
  AND (... OR e7 = {endStep}) -- 结束路径过滤
ORDER BY count DESC LIMIT 100
```

**关键特性**：
- 访问(visit)级别的序列行为分析
- 固定最多7步（SQL硬编码e1-e7）
- 后端 `parseResult` 阶段合并连续重复节点
- 结果按路径出现频次排序，取Top100
- 窗口函数 `row_number()` 标记事件顺序

#### 🔹 留存：群组(Cohort)分析

```sql
WITH cohort_items AS ( -- 第一步：确定每个会话的首次访问日期（群组归属）
  SELECT
    MIN(toDate(created_at, timezone)) AS cohort_date,
    session_id
  FROM website_event
  WHERE created_at BETWEEN ...
  GROUP BY session_id
),
user_activities AS ( -- 第二步：每个会话后续的回访活动（第N天）
  SELECT DISTINCT
    we.session_id,
    toInt32((toDate(we.created_at, timezone) - cohort_items.cohort_date) / 86400) AS day_number
  FROM website_event we
  JOIN cohort_items ON we.session_id = cohort_items.session_id
  WHERE created_at BETWEEN ...
),
cohort_size AS ( -- 第三步：每个群组的初始大小
  SELECT cohort_date, count(*) AS visitors FROM cohort_items GROUP BY 1
),
cohort_date AS ( -- 第四步：每个群组在第N天的回访人数
  SELECT c.cohort_date, a.day_number, count(*) AS visitors
  FROM user_activities a
  JOIN cohort_items c ON a.session_id = c.session_id
  GROUP BY 1, 2
)
-- 最终留存矩阵：群组日期 × 第N天 → 留存率
SELECT
  c.cohort_date AS date,
  c.day_number AS day,
  s.visitors,
  c.visitors AS returnVisitors,
  c.visitors * 100 / s.visitors AS percentage
FROM cohort_date c
JOIN cohort_size s ON c.cohort_date = s.cohort_date
WHERE c.day_number <= 31
ORDER BY 1, 2
```

**关键特性**：
- 双重会话识别（首次+回访活动）
- 时区强依赖（日期截断必须考虑时区）
- 4层CTE级联计算
- 结果是完整的31天留存矩阵（前端只取1/2/3/7/14/21/28天展示）
- 三个报表中计算成本最高

---

## 六、前端展示层差异

### 6.1 漏斗展示（`Funnel.tsx`）

**数据处理**：
- SQL返回 `[{level: 1, count: N}, {level: 2, count: M}, ...]`
- 前端计算：
  - `dropped = 上一步访客数 - 当前访客数`
  - `remaining = 当前访客数 / 第一步访客数`（累计转化率）
  - `dropoff = 1 - 访客数 / 上一步访客数`（步间流失率）

**展示特点**：
- 垂直步骤列表，连接线表示流程
- 每个步骤显示图标（页面/事件）+ 值 + 筛选条件标签
- 进度条可视化转化漏斗
- 支持编辑功能（非分享页面时）

### 6.2 用户路径展示（`Journey.tsx`）

**数据处理**：
- SQL返回 `{e1, e2, e3, e4, e5, e6, e7, count}`
- 后端 `parseResult` 合并连续重复项
- 前端按列分组，计算节点频次，生成连线坐标

**展示特点**：
- 横向列布局（每列代表一步）
- 节点大小按访客数排序
- 连线样式区分流转方向（向上/向下/平直）
- 点击节点高亮该节点出发的所有路径
- 鼠标悬停显示流失率/转化率提示

### 6.3 留存展示（`Retention.tsx`）

**数据处理**：
- SQL返回完整31天数据 `[{date, day, visitors, returnVisitors, percentage}]`
- 前端按 `day=0` 提取群组基本信息
- 二次查找匹配各天留存记录
- 仅展示：第1/2/3/4/5/6/7/14/21/28天

**展示特点**：
- 热力图矩阵布局（日期行 × 天数列）
- 单元格颜色深浅表示留存率高低
- 行首显示群组日期和群组大小

---

## 七、关键设计问题与优化建议

### 7.1 已确认的设计问题

#### 问题1：用户路径API遗漏日期限制（Bug）
**严重程度**：高  
**影响**：个人免费用户可查看6个月前的用户路径数据，绕过订阅限制  
**修复方案**（1行代码）：

```typescript
// src/app/api/reports/journey/route.ts
// 在权限检查后添加：

const { websiteId, parameters, filters } = body;
const { eventType } = parameters;

if (!(await canViewWebsite(auth, websiteId))) {
  return unauthorized();
}

// ✅ 新增：对 parameters 应用日期限制（与漏斗/留存对齐）
await setWebsiteDate(websiteId, parameters);

if (eventType) {
  filters.eventType = eventType;
}

const queryFilters = await getQueryFilters(filters, websiteId);
const data = await getJourney(websiteId, parameters, queryFilters);
```

#### 问题2：重复的双重限制调用
**现状**：`setWebsiteDate` 在两处被调用：
1. API层直接调用 `setWebsiteDate(parameters)`
2. `getQueryFilters` 内部再次调用 `setWebsiteDate(dateRange)`

两次查询数据库（`fetchWebsite` + `fetchAccount`），造成不必要的性能损耗。

**优化建议**：
```typescript
// 方案：一次计算，两处使用
const dateLimits = await getWebsiteDateLimits(websiteId); // 一次查询
const parameters = applyDateLimits(body.parameters, dateLimits);
const filters = await getQueryFilters(body.filters, websiteId, dateLimits); // 传入预计算结果
```

#### 问题3：用户路径硬编码7步限制
**现状**：SQL中硬编码 `e1-e7`，扩展性差  
**优化建议**：动态生成SQL列（需防范SQL注入，参数化处理）

### 7.2 缓存策略现状与建议

**当前状态**：
- ✅ 前端：React Query缓存（`useResultQuery` 自动管理）
- ❌ 后端：无应用级缓存
- ✅ 数据库层：PostgreSQL/ClickHouse自带查询缓存

**高成本查询缓存建议**：

| 报表 | 缓存建议 | TTL建议 | 缓存Key设计 |
|------|---------|---------|-----------|
| 留存 | ✅ 强烈建议 | 10-30分钟 | `retention:{websiteId}:{startDate}:{endDate}:{timezone}:{filtersHash}` |
| 漏斗（步数>3） | ✅ 建议 | 5-10分钟 | `funnel:{websiteId}:{stepsHash}:{window}:{filtersHash}` |
| 用户路径 | ⚠️ 可选 | 5分钟 | `journey:{websiteId}:{steps}:{startStep}:{endStep}:{filtersHash}` |

**原因**：
- 留存查询是4层CTE + 双重聚合，计算成本最高
- 漏斗查询是N层JOIN级联过滤，步数越多成本越高
- 用户路径是窗口函数 + 聚合，成本中等

---

## 附录：三大报表完整链路对比表

| 处理阶段 | 漏斗（Funnel） | 用户路径（Journey） | 留存（Retention） |
|---------|-------------|-------------------|-----------------|
| **前端Hook** | useResultQuery("funnel") | useResultQuery("journey") | useResultQuery("retention") |
| **自动注入日期** | ✅ startDate/endDate/timezone | ✅ startDate/endDate/timezone | ✅ startDate/endDate/timezone |
| **API权限检查** | canViewWebsite | canViewWebsite | canViewWebsite |
| **setWebsiteDate(parameters)** | ✅ 调用 | ❌ 不调用 | ✅ 调用 |
| **getQueryFilters** | ✅ 调用（内部再次setWebsiteDate） | ✅ 调用 | ✅ 调用 |
| **SQL日期来源** | parameters.startDate（已受限） | parameters.startDate（未受限） | parameters.startDate（已受限） |
| **双重限制保障** | ✅ 是 | ❌ 否 | ✅ 是 |
| **SQL核心算法** | 渐进式会话过滤 | 窗口函数序列提取 | 群组(Cohort)分析 |
| **CTE层数** | 2 + N（步骤数） | 2（events + sequences） | 4（cohort_items/user_activities/cohort_size/cohort_date） |
| **后端结果格式化** | formatResults（计算转化率/流失率） | parseResult（合并连续重复节点） | 无（直接返回矩阵） |
| **前端后处理** | 无 | 节点分组 + 连线坐标计算 | 按day=0提取 + 二次查找特定天数 |
| **受6个月限制** | ✅ 是 | ❌ 实际不受限 | ✅ 是 |

---

**报告生成时间**：2025-01-14  
**分析版本**：基于源码完整链路追踪（V2校正版）  
**关键校正点**：
1. 连续重复节点合并发生在后端而非前端
2. 用户路径时间限制的完整链路追踪（双重限制缺失）
3. 明确了 filters 与 parameters 中日期的优先级关系
