# Umami 从采集端到 ClickHouse 的完整流程解析

## 1. 整体架构概述

Umami 采用 **混合存储架构**，将关系型数据库（PostgreSQL/MySQL via Prisma）与列式数据库（ClickHouse）结合使用，实现了高效的事件采集、存储和查询分析。

```
采集端 (Tracker)
    ↓
API 路由 (/api/send, /api/record)
    ↓
数据处理层 (saveEvent, saveEventData)
    ↓
┌───────────────────────┬───────────────────────┐
│   Prisma (关系型DB)   │   ClickHouse (列式DB) │
│   - 用户/网站配置     │   - 事件原始数据      │
│   - 权限/团队管理     │   - 事件属性数据      │
│   - 报表/看板配置     │   - 会话数据          │
│   - 共享链接          │   - 聚合统计视图      │
└───────────────────────┴───────────────────────┘
    ↓
查询层 (API 路由 → SQL 查询)
    ↓
前端展示层
```

---

## 2. 采集端（Tracker）详解

### 2.1 数据采集入口

**核心文件**：`src/app/api/send/route.ts`

Umami 支持四种采集类型：
- `event`：页面浏览和自定义事件
- `identify`：用户识别（设置会话属性）
- `performance`：网页性能指标
- `record`：会话录制（独立路由 `/api/record`）

### 2.2 采集数据结构

```typescript
// 事件数据结构
interface EventPayload {
  website?: string;      // 网站 UUID
  link?: string;         // 短链接 ID
  pixel?: string;        // 像素 ID
  data?: object;         // 自定义事件属性
  hostname?: string;     // 主机名
  language?: string;     // 浏览器语言
  referrer?: string;     // 来源URL
  screen?: string;       // 屏幕尺寸
  title?: string;        // 页面标题
  url?: string;          // 页面URL
  name?: string;         // 事件名称
  tag?: string;          // 事件标签
  ip?: string;           // IP 地址
  userAgent?: string;    // 用户代理
  timestamp?: number;    // 时间戳
  id?: string;           // 用户唯一标识 (distinct_id)
  // Web Vitals
  lcp?: number;          // Largest Contentful Paint
  inp?: number;          // Interaction to Next Paint
  cls?: number;          // Cumulative Layout Shift
  fcp?: number;          // First Contentful Paint
  ttfb?: number;         // Time to First Byte
}
```

### 2.3 会话与访问标识机制

**核心文件**：`src/lib/crypto.ts`

```
会话标识生成逻辑：
├── sessionId: uuid(sourceId, ip, userAgent, sessionSalt)
│   └── saltRotation: 可配置 (月/周/日), 默认按月轮转
│
├── visitId: uuid(sessionId, visitSalt)
│   └── visitSalt: 基于当前小时的 hash
│
└── 缓存机制: 通过 JWT Token 存储在 x-umami-cache 头
    ├── 有效期: 30分钟（1800秒）
    └── 内容: { sessionId, visitId, iat }
```

### 2.4 机器人检测与IP过滤

```typescript
// 机器人检测：使用 isbot 库检查 userAgent
if (!process.env.DISABLE_BOT_CHECK && isbot(userAgent)) {
  return json({ beep: 'boop' });
}

// IP 地址过滤
if (hasBlockedIp(ip)) {
  return forbidden();
}
```

---

## 3. 事件写入 ClickHouse 流程

### 3.1 写入入口函数

**核心文件**：`src/queries/sql/events/saveEvent.ts`

```typescript
// 双写路由：根据环境变量选择数据库
export async function saveEvent(args: SaveEventArgs) {
  return runQuery({
    [PRISMA]: () => relationalQuery(args),      // PostgreSQL/MySQL
    [CLICKHOUSE]: () => clickhouseQuery(args),  // ClickHouse
  });
}
```

### 3.2 ClickHouse 写入流程

```
事件对象
    ↓
┌─────────────────────────────────┐
│  1. 提取标准化字段              │
│    - 来源/会话/访问ID          │
│    - URL/Referrer 解析         │
│    - 浏览器/OS/设备/国家        │
│    - UTM 参数/点击ID           │
│    - Web Vitals 指标           │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│  2. 事件类型路由                │
│    - pageView(1): 页面浏览     │
│    - customEvent(2): 自定义事件 │
│    - linkEvent(3): 短链接      │
│    - pixelEvent(4): 像素追踪   │
│    - performance(5): 性能指标  │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│  3. Kafka/直接写入              │
│    Kafka 启用 → 发送到 topic    │
│    否则 → 直接执行 INSERT       │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│  4. 事件属性展开 (如有 data)   │
│    → 调用 saveEventData         │
└─────────────────────────────────┘
```

### 3.3 事件属性展开（Event Data Flattening）

**核心文件**：
- `src/queries/sql/events/saveEventData.ts`
- `src/lib/data.ts`

#### 展开机制详解

```
输入: eventData = {
  "product": {
    "name": "Premium Plan",
    "price": 99.99,
    "features": ["analytics", "reports"]
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "success": true
}
    ↓ flattenJSON()
输出: [
  { key: "product.name", value: "Premium Plan", dataType: string(1) },
  { key: "product.price", value: 99.99, dataType: number(2) },
  { key: "product.features", value: '["analytics","reports"]', dataType: array(5) },
  { key: "timestamp", value: "2024-01-15T10:30:00Z", dataType: date(4) },
  { key: "success", value: "true", dataType: boolean(3) }
]
```

#### 数据类型映射

| 数据类型 | 枚举值 | 存储方式 |
|---------|-------|---------|
| string | 1 | 原始字符串 |
| number | 2 | 保留4位小数 |
| boolean | 3 | "true"/"false" |
| date | 4 | ISO 8601 格式 |
| array | 5 | JSON 字符串 |

#### 事件属性表结构设计

**ClickHouse DDL** (`db/clickhouse/schema.sql`):

```sql
CREATE TABLE umami.event_data
(
    website_id UUID,
    session_id UUID,
    event_id UUID,
    url_path String,
    event_name String,
    data_key String,           -- 展开后的属性名（如 product.price）
    string_value Nullable(String),
    number_value Nullable(Decimal(22, 4)),
    date_value Nullable(DateTime('UTC')),
    data_type UInt32,          -- 数据类型枚举
    created_at DateTime('UTC')
)
ENGINE = MergeTree
ORDER BY (website_id, event_id, data_key, created_at)
SETTINGS index_granularity = 8192;
```

**设计要点**：
- **键值对存储**：每个属性一行，支持无限维度扩展
- **多列存储**：不同数据类型存储在不同列，便于查询优化
- **排序键设计**：按 website_id → event_id → data_key 排序，加速按网站/事件查询

---

## 4. ClickHouse 数据模型详解

### 4.1 核心表结构

#### 4.1.1 主事件表：website_event

```sql
CREATE TABLE umami.website_event
(
    website_id UUID,           -- 网站ID (分区维度)
    session_id UUID,           -- 会话ID
    visit_id UUID,             -- 访问ID（30分钟内同一会话）
    event_id UUID,             -- 事件唯一ID

    -- 会话维度
    hostname LowCardinality(String),
    browser LowCardinality(String),
    os LowCardinality(String),
    device LowCardinality(String),
    screen LowCardinality(String),
    language LowCardinality(String),
    country LowCardinality(String),
    region LowCardinality(String),
    city String,               -- ⚠️ 高基数字段，无 LowCardinality

    -- 页面维度
    url_path String,           -- ⚠️ 高基数字段
    url_query String,
    utm_source String,
    utm_medium String,
    utm_campaign String,
    utm_content String,
    utm_term String,
    referrer_path String,
    referrer_query String,
    referrer_domain String,
    page_title String,         -- ⚠️ 高基数字段

    -- 点击ID
    gclid String,
    fbclid String,
    msclkid String,
    ttclid String,
    li_fat_id String,
    twclid String,

    -- 性能指标
    lcp Nullable(Decimal(10, 1)),
    inp Nullable(Decimal(10, 1)),
    cls Nullable(Decimal(10, 4)),
    fcp Nullable(Decimal(10, 1)),
    ttfb Nullable(Decimal(10, 1)),

    -- 事件类型
    event_type UInt32,         -- 1:pageView, 2:customEvent, 3:linkEvent...
    event_name String,         -- ⚠️ 高基数字段（自定义事件名）
    tag String,
    distinct_id String,        -- 用户自定义标识
    created_at DateTime('UTC'),
    job_id Nullable(UUID)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(created_at)         -- 按月分区
ORDER BY (
    toStartOfHour(created_at),            -- 小时级排序，加速时间范围查询
    website_id,
    session_id,
    visit_id,
    created_at
)
PRIMARY KEY (
    toStartOfHour(created_at),
    website_id,
    session_id,
    visit_id
)
SETTINGS index_granularity = 8192;
```

#### 4.1.2 会话数据表：session_data

```sql
CREATE TABLE umami.session_data
(
    website_id UUID,
    session_id UUID,
    data_key String,
    string_value Nullable(String),
    number_value Nullable(Decimal(22, 4)),
    date_value Nullable(DateTime('UTC')),
    data_type UInt32,
    distinct_id String,
    created_at DateTime('UTC')
)
ENGINE = ReplacingMergeTree      -- 同一键值自动去重，保留最新
ORDER BY (website_id, session_id, data_key);
```

### 4.2 物化视图：小时级聚合

**核心文件**：`db/clickhouse/schema.sql` (website_event_stats_hourly)

```sql
-- 聚合视图
CREATE TABLE umami.website_event_stats_hourly
(
    website_id UUID,
    session_id UUID,
    visit_id UUID,
    hostname SimpleAggregateFunction(groupArrayArray, Array(String)),
    browser LowCardinality(String),
    os LowCardinality(String),
    device LowCardinality(String),
    screen LowCardinality(String),
    language LowCardinality(String),
    country LowCardinality(String),
    region LowCardinality(String),
    city String,
    entry_url AggregateFunction(argMin, String, DateTime('UTC')),
    exit_url AggregateFunction(argMax, String, DateTime('UTC')),
    url_path SimpleAggregateFunction(groupArrayArray, Array(String)),
    url_query SimpleAggregateFunction(groupArrayArray, Array(String)),
    utm_source SimpleAggregateFunction(groupArrayArray, Array(String)),
    utm_medium SimpleAggregateFunction(groupArrayArray, Array(String)),
    utm_campaign SimpleAggregateFunction(groupArrayArray, Array(String)),
    utm_content SimpleAggregateFunction(groupArrayArray, Array(String)),
    utm_term SimpleAggregateFunction(groupArrayArray, Array(String)),
    referrer_domain SimpleAggregateFunction(groupArrayArray, Array(String)),
    page_title SimpleAggregateFunction(groupArrayArray, Array(String)),
    event_type UInt32,
    event_name SimpleAggregateFunction(groupArrayArray, Array(String)),
    views SimpleAggregateFunction(sum, UInt64),
    min_time SimpleAggregateFunction(min, DateTime('UTC')),
    max_time SimpleAggregateFunction(max, DateTime('UTC')),
    tag SimpleAggregateFunction(groupArrayArray, Array(String)),
    distinct_id String,
    created_at Datetime('UTC')
)
ENGINE = AggregatingMergeTree
PARTITION BY toYYYYMM(created_at)
ORDER BY (
    website_id,
    event_type,
    toStartOfHour(created_at),
    cityHash64(visit_id),     -- 采样键
    visit_id
)
SAMPLE BY cityHash64(visit_id);
```

**物化视图同步机制**：

```sql
CREATE MATERIALIZED VIEW umami.website_event_stats_hourly_mv
TO umami.website_event_stats_hourly
AS
SELECT
    website_id,
    session_id,
    visit_id,
    arrayFilter(x -> x != '', groupArray(hostname)) hostnames,
    browser,
    os,
    device,
    screen,
    language,
    country,
    region,
    city,
    argMinState(url_path, created_at) entry_url,   -- 入口页面
    argMaxState(url_path, created_at) exit_url,     -- 出口页面
    arrayFilter(x -> x != '', groupArray(url_path)) as url_paths,
    -- ... 其他维度聚合
    event_type,
    if(event_type = 2, groupArray(event_name), []) event_name,
    sumIf(1, event_type NOT IN (2, 5)) views,       -- 排除自定义事件和性能
    min(created_at) min_time,
    max(created_at) max_time,
    distinct_id,
    toStartOfHour(created_at) timestamp
FROM umami.website_event
GROUP BY website_id, session_id, visit_id, hostname, 
         browser, os, device, screen, language, 
         country, region, city, event_type, distinct_id, timestamp;
```

**设计意图**：
1. **预聚合加速查询**：将小时级数据预先聚合，避免扫描原始数据
2. **状态函数存储**：使用 AggregateFunction 存储中间聚合状态
3. **分组去重**：使用 SimpleAggregateFunction + groupArrayArray 存储多值

### 4.3 投影（Projections）优化

针对高频查询的辅助索引：

```sql
-- URL 路径查询优化
ALTER TABLE umami.website_event
ADD PROJECTION website_event_url_path_projection (
    SELECT * ORDER BY toStartOfDay(created_at), website_id, url_path, created_at
);

-- 来源域名查询优化
ALTER TABLE umami.website_event
ADD PROJECTION website_event_referrer_domain_projection (
    SELECT * ORDER BY toStartOfDay(created_at), website_id, referrer_domain, created_at
);
```

---

## 5. 查询接口与指标聚合

### 5.1 网站维度过滤机制

**核心文件**：`src/lib/clickhouse.ts` → `parseFilters`, `getFilterQuery`

#### 5.1.1 过滤参数定义

```typescript
// 支持的过滤字段映射
export const FILTER_COLUMNS = {
  path: 'url_path',
  entry: 'url_path',
  exit: 'url_path',
  referrer: 'referrer_domain',
  domain: 'referrer_domain',
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

// 支持的操作符
export const OPERATORS = {
  equals: 'eq',
  notEquals: 'neq',
  contains: 'c',
  doesNotContain: 'dnc',
  regex: 're',
  notRegex: 'nre',
  // ...
};
```

#### 5.1.2 过滤查询生成

```typescript
function getFilterQuery(filters, options) {
  const orClauses: string[] = [];
  const andClauses: string[] = [];

  filtersObjectToArray(filters, options).forEach(({ name, column, operator }) => {
    if (column) {
      // eventType 始终是 AND 条件
      const isAlwaysAnd = name === 'eventType' || (isCohort && name === cohortActionName);

      if (isAlwaysAnd) {
        andClauses.push(`and ${mapFilter(column, operator, name, type)}`);
      } else if (isOr) {
        orClauses.push(mapFilter(column, operator, name, 'String'));
      } else {
        andClauses.push(`and ${mapFilter(column, operator, name, 'String')}`);
      }

      // 来源过滤隐含条件：排除本站域名
      if (name === 'referrer') {
        andClauses.push(`and referrer_domain != hostname`);
      }
    }
  });

  const parts: string[] = [];

  if (orClauses.length > 0) {
    parts.push(`and (\n  ${orClauses.join('\n  or ')}\n)`);
  }

  parts.push(...andClauses);

  return parts.join('\n');
}
```

#### 5.1.3 过滤操作符映射

```typescript
function mapFilter(column, operator, name, type = 'String') {
  switch (operator) {
    case OPERATORS.equals:
      return `${column} IN {${name}:Array(${type})}`;

    case OPERATORS.notEquals:
      return `${column} NOT IN {${name}:Array(${type})}`;

    case OPERATORS.contains:
      return `positionCaseInsensitive(${column}, {${name}:String}) > 0`;

    case OPERATORS.doesNotContain:
      return `positionCaseInsensitive(${column}, {${name}:String}) = 0`;

    case OPERATORS.regex:
      return `match(${column}, concat('(?i)', {${name}:String}))`;

    // ... 其他操作符
  }
}
```

### 5.2 核心指标查询

#### 5.2.1 网站基础统计：getWebsiteStats

**核心文件**：`src/queries/sql/getWebsiteStats.ts`

```sql
-- ClickHouse 查询（无过滤条件时使用聚合视图）
SELECT
    sum(t.c) AS "pageviews",           -- 页面浏览量
    uniq(session_id) AS "visitors",    -- 独立访客数
    uniq(visit_id) AS "visits",        -- 访问次数
    sumIf(1, t.c = 1) AS "bounces",    -- 跳出次数（仅浏览1个页面）
    sum(max_time - min_time) AS "totaltime"  -- 总停留时长
FROM (
    SELECT
        session_id,
        visit_id,
        sum(views) c,                  -- 使用预聚合的 views
        min(min_time) min_time,
        max(max_time) max_time
    FROM website_event_stats_hourly
    WHERE website_id = {websiteId:UUID}
      AND created_at BETWEEN {startDate:DateTime64} AND {endDate:DateTime64}
      AND event_type NOT IN (2, 5)     -- 排除自定义事件和性能事件
      ${filterQuery}
    GROUP BY session_id, visit_id
) AS t;
```

**智能路由逻辑**：

```typescript
// 有事件维度过滤 → 查询原始表
if (EVENT_COLUMNS.some(item => Object.keys(filters).includes(item))) {
  // 查询 website_event 原始表
}
// 无过滤 → 使用聚合视图加速
else {
  // 查询 website_event_stats_hourly 聚合视图
}
```

#### 5.2.2 事件指标查询：getEventMetrics

**核心文件**：`src/queries/sql/events/getEventMetrics.ts`

```sql
SELECT
    ${column} AS x,                   -- 维度值（如 browser, os, country...）
    count(*) AS y                     -- 统计数
FROM website_event
WHERE website_id = {websiteId:UUID}
  AND created_at BETWEEN {startDate:DateTime64} AND {endDate:DateTime64}
  AND event_type = 2                  -- 仅自定义事件
  ${filterQuery}
GROUP BY x
ORDER BY y DESC
LIMIT ${limit}
OFFSET ${offset};
```

#### 5.2.3 事件属性值统计：getEventDataValues

**核心文件**：`src/queries/sql/events/getEventDataValues.ts`

```sql
SELECT
    multiIf(
        data_type = 2, replaceAll(string_value, '.0000', ''),  -- 数字去尾零
        data_type = 4, toString(date_trunc('hour', date_value)),  -- 日期按小时聚合
        string_value
    ) AS "value",
    count(*) AS "total"
FROM event_data
ANY LEFT JOIN (
    SELECT *
    FROM website_event
    WHERE website_id = {websiteId:UUID}
      AND created_at BETWEEN {startDate:DateTime64} AND {endDate:DateTime64}
      AND event_type = 2
) website_event ON website_event.event_id = event_data.event_id
WHERE event_data.website_id = {websiteId:UUID}
  AND event_data.created_at BETWEEN {startDate:DateTime64} AND {endDate:DateTime64}
  AND event_data.data_key = {propertyName:String}  -- 按属性名过滤
  ${filterQuery}
GROUP BY value
ORDER BY 2 DESC
LIMIT 100;
```

### 5.3 群组分析（Cohort）支持

```typescript
function getCohortQuery(filters) {
  const cohortMatch = filters.cohort_match;        // any/all
  const cohortActionName = filters.cohort_actionName;

  const filterQuery = getFilterQuery(filters, { isCohort: true, cohortMatch, cohortActionName });

  // 子查询找出符合群组条件的 session_id，然后 JOIN 回主表
  return `
    JOIN (
        SELECT DISTINCT session_id AS cohort_session_id
        FROM website_event
        WHERE website_id = {websiteId:UUID}
          AND created_at BETWEEN {cohort_startDate:DateTime64} AND {cohort_endDate:DateTime64}
          ${filterQuery}
    ) AS cohort
    ON cohort.cohort_session_id = website_event.session_id
  `;
}
```

---

## 6. Prisma 与 ClickHouse 的职责分配

### 6.1 核心分工矩阵

| 功能模块 | Prisma (PostgreSQL/MySQL) | ClickHouse |
|---------|---------------------------|-----------|
| **用户管理** | ✅ 用户账号、密码、角色 | ❌ |
| **团队管理** | ✅ 团队成员、权限、角色 | ❌ |
| **网站配置** | ✅ 网站基础信息、域名、重置设置 | ❌ |
| **共享链接** | ✅ 共享 token、权限、过期时间 | ❌ |
| **报表/看板** | ✅ 报表定义、看板配置、图表布局 | ❌ |
| **事件数据存储** | ⚠️（仅小数据量） | ✅ 原始事件、属性、会话 |
| **实时查询** | ❌（性能不足） | ✅ 毫秒级聚合查询 |
| **漏斗分析** | ❌ | ✅ 多步骤会话漏斗 |
| **留存分析** | ❌ | ✅ 用户留存计算 |
| **路径分析** | ❌ | ✅ 页面流转路径 |
| **收入分析** | ❌ | ✅ 收入指标聚合 |
| **会话录制** | ❌ | ✅ 录制事件流存储 |

### 6.2 查询路由机制

**核心文件**：`src/lib/db.ts`

```typescript
export function runQuery(queries: any) {
  // 优先使用 ClickHouse（如配置了 CLICKHOUSE_URL）
  if (process.env.CLICKHOUSE_URL) {
    if (queries[KAFKA]) {
      return queries[KAFKA]();   // Kafka 异步写入
    }
    return queries[CLICKHOUSE]();
  }

  // 回退到关系型数据库
  const db = getDatabaseType();
  if (db === POSTGRESQL) {
    return queries[PRISMA]();
  }
}
```

### 6.3 双写一致性保证

1. **写入路径分离**：
   - 配置类数据 → 仅写 Prisma
   - 事件类数据 → 仅写 ClickHouse（或 Kafka）

2. **查询路径自动路由**：
   - 每个查询函数都提供双实现（`relationalQuery` + `clickhouseQuery`）
   - 运行时根据环境变量自动选择

3. **跨库关联查询**：
   - 网站元数据（Prisma） + 事件统计（ClickHouse）
   - 通过 `website_id` UUID 进行关联

---

## 7. 高基数字段查询成本分析

### 7.1 高基数字段识别

ClickHouse 表中的高基数字段（未使用 LowCardinality）：

| 字段名 | 基数估算 | 索引支持 | 性能影响 |
|-------|---------|---------|---------|
| `city` | 数万~数十万 | 排序键第9位 | ⚠️ 中等 |
| `url_path` | 数十万~数百万 | 投影索引 | ⚠️ 中等（有投影优化） |
| `url_query` | 极高 | 无索引 | ⚠️ 严重 |
| `page_title` | 极高 | 无索引 | ⚠️ 严重 |
| `event_name` | 数万~数十万 | 无索引 | ⚠️ 严重 |
| `referrer_path` | 极高 | 无索引 | ⚠️ 严重 |
| `distinct_id` | 用户自定义 | 无索引 | ⚠️ 严重 |
| `gclid/fbclid` | 极高 | 无索引 | ⚠️ 严重 |

### 7.2 查询成本量化

#### 7.2.1 等值查询成本

```sql
-- ✅ 低成本（LowCardinality + 排序键前缀）
SELECT count(*) FROM website_event
WHERE website_id = 'xxx' AND country = 'US';
-- 成本: 扫描少数颗粒度，O(1) ~ O(log n)

-- ⚠️ 中等成本（非排序键但有投影）
SELECT count(*) FROM website_event
WHERE website_id = 'xxx' AND url_path = '/checkout';
-- 成本: 命中投影，仍需扫描多个分区

-- ❌ 高成本（高基数+无索引）
SELECT count(*) FROM website_event
WHERE website_id = 'xxx' AND event_name = 'purchase_completed';
-- 成本: 全表扫描指定时间范围分区，O(n)
```

#### 7.2.2 模糊查询成本

```sql
-- ❌ 极高成本（正则匹配）
SELECT count(*) FROM website_event
WHERE match(url_path, '.*checkout.*');
-- 成本: 逐行正则匹配，CPU 密集型

-- ❌ 高成本（包含匹配）
SELECT count(*) FROM website_event
WHERE positionCaseInsensitive(page_title, 'Premium') > 0;
-- 成本: 字符串扫描，无法利用索引
```

#### 7.2.3 GROUP BY 成本

```sql
-- ✅ 低成本（低基数维度）
SELECT browser, count(*) FROM website_event
WHERE website_id = 'xxx'
GROUP BY browser;

-- ❌ 高成本（高基数维度）
SELECT event_name, count(*) FROM website_event
WHERE website_id = 'xxx'
GROUP BY event_name;
-- 成本: 内存中哈希分组，高基数导致内存压力
```

### 7.3 性能优化策略

#### 7.3.1 已实现的优化

1. **投影索引（Projections）**：
   - `url_path` 维度查询优化
   - `referrer_domain` 维度查询优化

2. **LowCardinality 编码**：
   - browser/os/device/country/region 等维度字段
   - 减少存储空间，加速 GROUP BY

3. **聚合视图（Materialized View）**：
   - 小时级预聚合，避免重复计算
   - 无过滤条件的查询自动路由到聚合视图

4. **分区裁剪（Partition Pruning）**：
   - 按月分区 + 时间范围查询
   - 自动跳过不相关分区

#### 7.3.2 潜在优化方向

```sql
-- 建议1: 为高频高基数字段增加 Bloom Filter 索引
ALTER TABLE umami.website_event
ADD INDEX event_name_idx event_name TYPE bloom_filter GRANULARITY 1;

ALTER TABLE umami.website_event
ADD INDEX url_path_idx url_path TYPE ngrambf_v1(3, 256, 0, 0) GRANULARITY 1;

-- 建议2: 对 event_name 应用 LowCardinality（如果基数在百万内）
ALTER TABLE umami.website_event
MODIFY COLUMN event_name LowCardinality(String);

-- 建议3: 增加按事件类型的聚合视图
CREATE MATERIALIZED VIEW umami.custom_event_stats_hourly_mv
...
WHERE event_type = 2;  -- 仅自定义事件
```

### 7.4 查询性能最佳实践

| 查询模式 | 推荐做法 | 避免做法 |
|---------|---------|---------|
| **按事件名查询** | 使用 `event_name IN (...)` 等值查询 | 避免 `LIKE '%xxx%'` 模糊匹配 |
| **按URL查询** | 使用投影索引支持的路径前缀匹配 | 避免复杂正则表达式 |
| **按自定义属性查询** | 先缩小时间范围再 JOIN event_data | 避免跨大时间范围的属性查询 |
| **获取 Top N** | 结合 `LIMIT` + 降序排序 | 不要获取全量再在应用层排序 |
| **去重统计** | 使用 `uniq` / `uniqHLL12` 近似函数 | 避免 `count(DISTINCT x)` 精确去重 |

---

## 8. 总结：架构设计亮点与权衡

### 8.1 设计亮点

1. **属性动态展开**：JSON → 键值对表，支持无限维度扩展
2. **混合存储架构**：Prisma 管理元数据，ClickHouse 负责事件分析
3. **物化视图加速**：小时级预聚合，无过滤查询毫秒级返回
4. **投影索引优化**：针对高频过滤维度创建排序投影
5. **LowCardinality 编码**：低基数维度存储优化
6. **Kafka 异步写入**：支持高吞吐场景下的缓冲

### 8.2 权衡与取舍

| 决策 | 优势 | 代价 |
|-----|-----|-----|
| **事件属性行存储** | 灵活支持任意属性 | JOIN 成本高，存储放大 |
| **ClickHouse 单副本** | 部署简单 | 无高可用保障 |
| **按月分区** | 管理简单 | 大时间范围查询仍需扫描多分区 |
| **无数据TTL** | 数据永久保留 | 存储持续增长 |
| **实时写入无批量** | 数据立即可查 | 高并发下单行写入性能受限 |

### 8.3 可扩展性建议

1. **写入侧**：启用 Kafka 缓冲 + 批量写入 ClickHouse
2. **存储侧**：配置 TTL 自动清理历史数据，使用分层存储（热/冷数据分离）
3. **查询侧**：增加更多维度的物化视图，引入 Query Cache
4. **高可用**：部署 ClickHouse 集群（2 shards × 2 replicas）
5. **监控**：增加慢查询日志，监控 part 合并和 mutation 队列

---

## 附录：关键代码文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| 采集入口 | `src/app/api/send/route.ts` |
| 录制入口 | `src/app/api/record/route.ts` |
| 事件写入 | `src/queries/sql/events/saveEvent.ts` |
| 事件属性写入 | `src/queries/sql/events/saveEventData.ts` |
| 属性展开 | `src/lib/data.ts` |
| ClickHouse 工具 | `src/lib/clickhouse.ts` |
| 数据库路由 | `src/lib/db.ts` |
| 基础统计查询 | `src/queries/sql/getWebsiteStats.ts` |
| 事件指标查询 | `src/queries/sql/events/getEventMetrics.ts` |
| 属性值查询 | `src/queries/sql/events/getEventDataValues.ts` |
| ClickHouse DDL | `db/clickhouse/schema.sql` |
| 常量定义 | `src/lib/constants.ts` |
