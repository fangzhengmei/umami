# Umami Revenue 指标查询分析

## 1. 系统架构概览

### 1.1 核心组件关系

```
前端 Revenue 页面 (Revenue.tsx)
        ↓ (HTTP POST)
API 路由 (route.ts)
        ↓ (并行调用)
┌─────────────────┬──────────────────┬────────────────────┐
│ getRevenue      │ getRevenueStats  │ getRevenueMetrics  │
│ (图表数据)      │ (统计指标)       │ (维度分析)         │
└─────────────────┴──────────────────┴────────────────────┘
        ↓
双数据库适配层 (PostgreSQL / ClickHouse)
```

### 1.2 代码文件位置

| 模块 | 文件路径 |
|------|----------|
| API 路由 | `src/app/api/reports/revenue/route.ts` |
| 图表查询 | `src/queries/sql/reports/getRevenue.ts` |
| 统计指标 | `src/queries/sql/reports/getRevenueStats.ts` |
| 维度分析 | `src/queries/sql/reports/getRevenueMetrics.ts` |
| 会话列表 | `src/queries/sql/reports/getRevenueSessions.ts` |
| 数据写入 | `src/queries/sql/events/saveRevenue.ts` |
| 前端页面 | `src/app/(main)/websites/[websiteId]/(reports)/revenue/Revenue.tsx` |

---

## 2. 数据库模型

### 2.1 Revenue 表结构

**Prisma Schema** (`prisma/schema.prisma:268-286`):
```prisma
model Revenue {
  id        String    @id() @map("revenue_id") @db.Uuid
  websiteId String    @map("website_id") @db.Uuid
  sessionId String    @map("session_id") @db.Uuid
  eventId   String    @map("event_id") @db.Uuid
  eventName String    @map("event_name") @db.VarChar(50)
  currency  String    @db.VarChar(10)
  revenue   Decimal?  @db.Decimal(19, 4)
  createdAt DateTime? @default(now()) @map("created_at") @db.Timestamptz(6)

  website Website @relation(fields: [websiteId], references: [id])
  session Session @relation(fields: [sessionId], references: [id])
}
```

**关键字段说明：**
- `revenue`: 金额，使用 `Decimal(19, 4)` 精度存储
- `currency`: 货币代码（如 USD, EUR, CNY），长度 10 字符
- `eventName`: 事件名称（如 "Purchase", "Checkout"）
- 关联 `website`、`session`、`event`（通过 eventId）

---

## 3. 筛选条件处理流程

### 3.1 输入参数结构

**RevenueParameters** (`src/queries/sql/reports/getRevenue.ts:6-13`):
```typescript
interface RevenuParameters {
  startDate: Date;      // 开始日期
  endDate: Date;        // 结束日期
  unit: string;         // 时间粒度 (minute/hour/day/month/year)
  timezone: string;     // 时区
  currency: string;     // 货币代码
  compare?: string;     // 对比周期 (prev: 上一周期)
}
```

**QueryFilters** (`src/lib/types.ts:66-73`):
```typescript
interface QueryFilters extends DateParams, FilterParams, SortParams, PageParams, SegmentParams {
  cohortFilters?: QueryFilters;  // 同期群筛选
}
```

### 3.2 筛选条件解析流程

**核心解析函数** `parseFilters()` (`src/lib/prisma.ts:232-253`):

```
输入 filters 对象
    ↓
1. 检测是否需要关联 session 表
   (筛选条件包含 referrer 或 SESSION_COLUMNS 中字段)
    ↓
2. 分离 cohort 筛选条件 (前缀 cohort_)
    ↓
3. 生成各部分 SQL:
   ├─ joinSessionQuery: 内连接 session 表
   ├─ dateQuery: 日期范围过滤
   ├─ filterQuery: 业务筛选条件
   ├─ cohortQuery: 同期群子查询
   └─ queryParams: 预处理参数值
```

### 3.3 支持的筛选字段

**FILTER_COLUMNS** (`src/lib/constants.ts:72-97`):

| 前端字段 | 数据库字段 | 所属表 |
|---------|-----------|--------|
| path, entry, exit | url_path | website_event |
| referrer, domain | referrer_domain | website_event |
| hostname | hostname | website_event |
| query | url_query | website_event |
| event | event_name | website_event |
| os, browser, device, country, region, city, language | 同名 | session |
| utmSource, utmMedium, utmCampaign, utmContent, utmTerm | utm_* | website_event |

### 3.4 筛选操作符

**OPERATORS** (`src/lib/constants.ts:137-154`):

| 操作符 | 说明 | SQL 示例 |
|--------|------|----------|
| eq | 等于 | `column = ANY([value1, value2])` |
| neq | 不等于 | `column != ALL([value1, value2])` |
| c | 包含 | `column ILIKE '%value%'` |
| dnc | 不包含 | `column NOT ILIKE '%value%'` |
| re | 正则匹配 | `column ~* 'pattern'` |
| nre | 正则不匹配 | `column !~* 'pattern'` |

### 3.5 特殊筛选逻辑

**同期群 (Cohort) 筛选:**
- 筛选条件前缀 `cohort_`
- 生成子查询：找出在指定时间段内满足条件的 session_id
- 主查询通过 JOIN 关联这些 session_id

**排除跳出 (Exclude Bounce):**
- 筛选条件 `excludeBounce: true`
- 排除只有一个 pageview 的会话

---

## 4. 数据聚合逻辑

### 4.1 四大核心查询

#### (1) 图表数据 - `getRevenue()`

**返回结构:**
```typescript
{
  chart: Array<{
    x: string;      // eventName (事件名称)
    t: string;      // 时间桶 (格式化日期)
    y: number;      // 金额总和
    count: number;  // 订单数量
  }>
}
```

**核心 SQL (PostgreSQL) `src/queries/sql/reports/getRevenue.ts:51-70`:**
```sql
select
  revenue.event_name x,
  to_char(date_trunc('day', revenue.created_at at time zone 'UTC'), 'YYYY-MM-DD"T"HH24:00:00"Z"') t,
  sum(revenue.revenue) y,
  count(revenue.event_id) count
from revenue
-- 可选 JOIN (当有筛选或 cohort 时)
join (select * from website_event 
      where website_id = {{websiteId}}
        and created_at between {{startDate}} and {{endDate}}
        and event_type = 2) website_event
on website_event.website_id = revenue.website_id
  and website_event.session_id = revenue.session_id
  and website_event.event_id = revenue.event_id
where revenue.website_id = {{websiteId}}
  and revenue.created_at between {{startDate}} and {{endDate}}
  and upper(revenue.currency) = {{currency}}
  -- 动态筛选条件
group by x, t
order by t
```

**时间格式化** (`src/lib/prisma.ts:22-56`):
| unit | PostgreSQL 格式 |
|------|----------------|
| minute | YYYY-MM-DD HH24:MI:00 |
| hour | YYYY-MM-DD HH24:00:00 |
| day | YYYY-MM-DD HH24:00:00 |
| month | YYYY-MM-01 HH24:00:00 |
| year | YYYY-01-01 HH24:00:00 |

---

#### (2) 统计指标 - `getRevenueStats()`

**返回结构:**
```typescript
interface RevenueStatsResult {
  sum: number;          // 总收入
  count: number;        // 订单数
  average: number;      // 平均订单价值 (AOV)
  unique_count: number; // 独立客户数 (去重 session)
  arpu: number;         // 每用户平均收入
  comparison?: {        // 对比周期数据 (可选)
    sum: number;
    count: number;
    average: number;
    unique_count: number;
    arpu: number;
  }
}
```

**核心 SQL (PostgreSQL) `src/queries/sql/reports/getRevenueStats.ts:51-71`:**
```sql
select
  sum(revenue.revenue) as sum,
  count(distinct revenue.event_id) as count,
  count(distinct revenue.session_id) as unique_count,
  (select count(distinct session_id)
   from website_event
   where website_id = {{websiteId}}
     and created_at between {{startDate}} and {{endDate}}) as total_sessions
from revenue
-- 可选 JOIN
where revenue.website_id = {{websiteId}}
  and revenue.created_at between {{startDate}} and {{endDate}}
  and upper(revenue.currency) = {{currency}}
```

**后续计算** (`src/queries/sql/reports/getRevenueStats.ts:73-74`):
```javascript
total.average = total.count > 0 ? Number(total.sum) / Number(total.count) : 0;
total.arpu = total.total_sessions > 0 ? Number(total.sum) / Number(total.total_sessions) : 0;
```

**指标公式汇总:**
| 指标 | 公式 | 说明 |
|------|------|------|
| Total (总收入) | `SUM(revenue)` | 所有订单金额总和 |
| Orders (订单数) | `COUNT(DISTINCT event_id)` | 不同事件 ID 数量 |
| AOV (平均订单价值) | `Total / Orders` | 平均每单金额 |
| Unique Customers | `COUNT(DISTINCT session_id)` | 产生收入的独立会话数 |
| ARPU (每用户平均收入) | `Total / Total Sessions` | 总收入 / 所有会话数 |

---

#### (3) 维度分析 - `getRevenueMetrics()`

**返回结构:**
```typescript
interface RevenueMetricsResult {
  country: Array<{ name: string; value: number }>;
  region: Array<{ name: string; value: number; country: string }>;
  referrer: Array<{ name: string; value: number }>;
  channel: Array<{ name: string; value: number }>;
}
```

##### 3.1 国家/地区维度

**SQL 逻辑:**
- JOIN `session` 表获取国家/地区信息
- 按国家/地区分组汇总 revenue
- 按金额降序排列

```sql
select
  session.country as "name",
  sum(revenue) as "value"
from revenue
join session on session.website_id = revenue.website_id
            and session.session_id = revenue.session_id
-- 可选 cohort JOIN
where revenue.website_id = {{websiteId}}
  and revenue.created_at between {{startDate}} and {{endDate}}
  and upper(revenue.currency) = {{currency}}
group by session.country
order by value desc
```

##### 3.2 来源/渠道维度

**归因逻辑** (首次接触归因):
1. 先按 session 汇总 revenue
2. 找出每个 session 的首个事件时间 (`min(created_at)`)
3. 关联首个事件的 referrer_domain、utm 参数等
4. 按来源/渠道分组

```sql
WITH events AS (
  select
    revenue.website_id,
    revenue.session_id,
    sum(revenue.revenue) as "value"
  from revenue
  group by revenue.website_id, revenue.session_id
),
revenue_data AS (
  select
    e.website_id,
    e.session_id,
    e.value,
    we.min_date as created_at
  from events e
  join (select session_id, min(created_at) as min_date
        from website_event group by session_id) we
  on we.session_id = e.session_id
)
-- 关联首个事件的 referrer/utm 信息进行分组
```

##### 3.3 渠道分类规则

**渠道判定优先级** (`src/queries/sql/reports/getRevenueMetrics.ts:207-219`):

| 优先级 | 条件 | 渠道名称 |
|--------|------|----------|
| 1 | referrer_domain = '' 且 url_query = '' | direct |
| 2 | url_query 包含付费广告参数 (gclid, fbclid 等) | paidAds |
| 3 | utm_medium 包含 referral/app/link | referral |
| 4 | utm_medium 包含 affiliate | affiliate |
| 5 | utm_medium 或 utm_source 包含 sms | sms |
| 6 | referrer_domain 属于搜索引擎 或 utm_medium=organic | {prefix}Search |
| 7 | referrer_domain 属于社交媒体 | {prefix}Social |
| 8 | referrer_domain 属于邮箱 或 utm_medium 包含 mail | email |
| 9 | referrer_domain 属于购物网站 或 utm_medium 包含 shop | {prefix}Shopping |
| 10 | referrer_domain 属于视频网站 或 utm_medium 包含 video | {prefix}Video |
| 11 | referrer_domain != hostname 且不为空 | referral |
| 12 | 其他 | Unknown |

**付费/自然 (prefix) 判定:**
```sql
case when utm_medium ilike '%cp%' OR
          utm_medium ilike '%ppc%' OR
          utm_medium ilike '%retargeting%' OR
          utm_medium ilike '%paid%'
     then 'paid' else 'organic' end AS prefix
```

**付费广告参数列表** (`src/lib/constants.ts:354-373`):
```
ad_id=, aid=, dclid=, epik=, fbclid=, gclid=, li_fat_id=,
msclkid=, ob_click_id=, pc_id=, rdt_cid=, scid=, ttclid=,
twclid=, utm_medium=cpc, utm_medium=paid, utm_medium=paid_social,
utm_source=google
```

---

#### (4) 会话列表 - `getRevenueSessions()`

**查询逻辑:**
1. 子查询找出有 revenue 的 session_id 集合
2. 关联 website_event 和 session 表获取详细信息
3. 按 session 分组，统计浏览量、事件数等
4. 支持搜索 (browser/os/device/city) 和分页

---

## 5. 货币处理机制

### 5.1 支持的货币列表

**CURRENCIES** (`src/lib/constants.ts:645-696`):
- 共支持 50 种货币
- 默认货币: `USD` (DEFAULT_CURRENCY)
- 配置存储键: `umami.currency` (CURRENCY_CONFIG)

### 5.2 货币过滤

**查询层过滤** (所有 revenue 查询通用):
```sql
and upper(revenue.currency) = {{currency}}
```

- 使用 `upper()` 进行大小写不敏感匹配
- 货币代码在查询参数中传递，需用户选择

### 5.3 货币格式化

**核心函数** `formatCurrency()` (`src/lib/format.ts:87-104`):
```typescript
export function formatCurrency(value: number, currency: string, locale = 'en-US') {
  try {
    return new Intl.NumberFormat(locale, {
      style: 'currency',
      currency: currency,
    }).format(value);
  } catch {
    // fallback 到默认货币
    return new Intl.NumberFormat(locale, {
      style: 'currency',
      currency: DEFAULT_CURRENCY,
    }).format(value);
  }
}
```

**长数字格式化** `formatLongCurrency()` (`src/lib/format.ts:106-120`):
| 数值范围 | 显示格式 |
|---------|----------|
| >= 1,000,000,000 | `$1.2b` (十亿美元) |
| >= 1,000,000 | `$1.2m` (百万美元) |
| >= 1,000 | `$1.20k` (千美元) |
| < 1,000 | `$120.00` (完整格式) |

### 5.4 前端货币选择

**CurrencySelect 组件** (`Revenue.tsx:57-64`):
```typescript
const [currency, setCurrency] = useState(
  getItem(CURRENCY_CONFIG) || process.env.defaultCurrency || DEFAULT_CURRENCY,
);
```

- 优先级: 本地存储 > 环境变量 > 默认值 (USD)
- 切换时保存到 localStorage

---

## 6. API 请求与响应

### 6.1 API 路由处理

**请求处理流程** (`src/app/api/reports/revenue/route.ts:10-40`):

```typescript
// 1. 解析请求和权限校验
const { auth, body, error } = await parseRequest(request, reportResultSchema);
if (!(await canViewWebsite(auth, websiteId))) return unauthorized();

// 2. 处理日期范围 (考虑云模式限制、数据重置时间)
const parameters = await setWebsiteDate(websiteId, body.parameters);

// 3. 解析筛选条件
const filters = await getQueryFilters(body.filters, websiteId);

// 4. 并行执行三个查询
const [{ chart }, total, metrics] = await Promise.all([
  getRevenue(websiteId, parameters, filters),
  getRevenueStats(websiteId, parameters, filters),
  getRevenueMetrics(websiteId, parameters, filters),
]);

// 5. 查询对比周期数据
const { startDate, endDate } = getCompareDate(compare, parameters.startDate, parameters.endDate);
const comparison = await getRevenueStats(
  websiteId,
  { ...parameters, startDate, endDate },
  filters,
);

// 6. 返回合并结果
return json({ chart, total: { ...total, comparison }, ...metrics });
```

### 6.2 完整响应结构

```json
{
  "chart": [
    { "x": "Purchase", "t": "2024-01-01T00:00:00Z", "y": 1250.50, "count": 15 }
  ],
  "total": {
    "sum": 15000.00,
    "count": 150,
    "average": 100.00,
    "unique_count": 120,
    "arpu": 15.00,
    "comparison": {
      "sum": 12000.00,
      "count": 100,
      "average": 120.00,
      "unique_count": 90,
      "arpu": 12.00
    }
  },
  "country": [
    { "name": "US", "value": 8000.00 }
  ],
  "region": [
    { "name": "California", "value": 4000.00, "country": "US" }
  ],
  "referrer": [
    { "name": "google.com", "value": 6000.00 }
  ],
  "channel": [
    { "name": "organicSearch", "value": 5000.00 }
  ]
}
```

---

## 7. 前端展示流程

### 7.1 页面结构

**Revenue 组件** (`Revenue.tsx:129-261`):
```
Column
├─ Grid → CurrencySelect (货币选择)
└─ LoadingPanel
   └─ Column (当 data 存在时)
      ├─ MetricsBar → 5 个 MetricCard
      │   ├─ Total (总收入)
      │   ├─ AOV (平均订单价值)
      │   ├─ ARPU (每用户平均收入)
      │   ├─ Orders (订单数)
      │   └─ Unique Customers (独立客户)
      ├─ Panel → RevenueChart (图表)
      ├─ Grid (两栏布局)
      │   ├─ Panel (来源)
      │   │   └─ Tabs
      │   │      ├─ Tab referrer → ListTable
      │   │      └─ Tab channel → ListTable
      │   └─ Panel (地理位置)
      │       └─ Tabs
      │          ├─ Tab country → ListTable
      │          └─ Tab region → ListTable
      └─ Panel (客户列表)
          └─ RevenueSessionsDataTable
```

### 7.2 指标卡片展示

**metrics 数据处理** (`Revenue.tsx:76-125`):
```typescript
const metrics = useMemo(() => {
  if (!data) return [];
  const { sum, count, average, unique_count, arpu, comparison } = data.total;
  
  return [
    {
      value: sum,
      label: 'Total',
      change: comparison ? sum - comparison.sum : 0,
      formatValue: (n) => formatLongCurrency(n, currency),
    },
    // ... 其他指标
  ];
}, [data]);
```

- 每个指标计算与对比周期的差值
- 使用 `formatLongCurrency` 格式化金额
- 非 "All Time" 范围显示变化值

### 7.3 图表展示

**RevenueChart** 组件:
- 接收 `data.chart` 数组
- 按 `eventName` 分组显示多条线/柱状图
- X 轴为时间 (`t` 字段)
- Y 轴为金额 (`y` 字段)
- Tooltip 显示货币格式化值

### 7.4 列表展示

**ListTable** 组件 (来源/地理位置):
- 显示 `name` (标签) 和 `value` (金额)
- 计算百分比: `(value / total.sum) * 100`
- 按金额降序排列
- 使用 `formatLongCurrency` 格式化

---

## 8. 双数据库适配

### 8.1 适配层设计

所有 revenue 查询均通过 `runQuery()` 进行数据库适配:
```typescript
export async function getRevenueStats(...args) {
  return runQuery({
    [PRISMA]: () => relationalQuery(...args),     // PostgreSQL
    [CLICKHOUSE]: () => clickhouseQuery(...args),  // ClickHouse
  });
}
```

### 8.2 PostgreSQL vs ClickHouse 差异

| 特性 | PostgreSQL | ClickHouse |
|------|------------|------------|
| 表名 | `revenue` | `website_revenue` |
| 日期格式化 | `to_char(date_trunc(...))` | 自定义 `getDateSQL()` |
| 字符串匹配 | `ILIKE` | `positionCaseInsensitive()` / `multiSearchAny()` |
| 去重计数 | `COUNT(DISTINCT col)` | `uniqExact(col)` |
| JOIN 类型 | `INNER JOIN` | `ANY LEFT JOIN` |
| 数组操作 | `= ANY(arr)` / `!= ALL(arr)` | `has()` 函数 |

**ClickHouse 渠道判定示例:**
```sql
when multiSearchAny(lower(referrer_domain), ['google.', 'bing.com']) != 0
     or position(lower(utm_medium), 'organic') > 0
then concat(prefix, 'Search')
```

---

## 9. 数据写入流程

### 9.1 数据收集

**Tracker 端上报:**
- 事件类型: 自定义事件 (event_type = 2)
- 事件属性包含: `revenue` (金额) 和 `currency` (货币)
- 示例: `umami.track('Purchase', { revenue: 99.99, currency: 'USD' })`

### 9.2 数据存储

**saveRevenue** (`src/queries/sql/events/saveRevenue.ts:15-35`):
```typescript
async function relationalQuery(data: SaveRevenueArgs) {
  const { websiteId, sessionId, eventId, eventName, currency, revenue, createdAt } = data;
  
  await prisma.client.revenue.create({
    data: {
      id: uuid(),
      websiteId,
      sessionId,
      eventId,
      eventName,
      currency,
      revenue,
      createdAt,
    },
  });
}
```

---

## 10. 关键设计决策总结

### 10.1 归因模型
- **首次接触归因**: 将 revenue 归因为用户会话的第一个事件来源
- **实现方式**: 通过 `min(created_at)` 找到会话首个事件，关联其来源信息

### 10.2 货币处理
- **单货币查询**: 每次查询只能选择一种货币，不进行汇率转换
- **大小写不敏感**: 使用 `upper()` 匹配货币代码
- **本地格式化**: 使用浏览器 `Intl.NumberFormat` 进行本地化显示

### 10.3 性能优化
- **并行查询**: 三个主查询通过 `Promise.all` 并行执行
- **可选 JOIN**: 仅当有筛选条件时才 JOIN website_event 表
- **双数据库支持**: 同时支持 PostgreSQL 和 ClickHouse，可根据规模选择
- **读副本**: 支持通过 `DATABASE_REPLICA_URL` 配置读副本

### 10.4 扩展性
- **筛选框架**: 统一的 `parseFilters` 机制，支持动态字段和操作符
- **同期群支持**: 内置 cohort 筛选能力，支持复杂用户分群分析
- **分段 (Segment)**: 支持保存的筛选条件，可复用和共享
