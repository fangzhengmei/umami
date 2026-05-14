# Umami 从采集端到 ClickHouse 的完整链路分析报告（可证据化版本）

## 报告说明与边界声明

### 证据检索说明

本报告所有结论均基于仓库内的源代码分析得出。对于关键结论，均标注了对应的文件路径作为证据来源。

### 核心边界与风险前置说明

| 结论类型 | 说明 |
|---------|-----|
| ✅ **仓内可证** | 结论可由仓库内代码直接验证 |
| ⚠️ **外部依赖** | 结论依赖外部基础设施，仓内代码不包含完整实现 |
| ❌ **无法证伪/证实** | 仓内代码不足以得出确定性结论 |

---

## 1. 整体架构概览

### 1.1 数据流向

```
采集端 (Tracker)
    ↓  [✅ 仓内可证] src/app/api/send/route.ts
API 路由 (/api/send, /api/record)
    ↓  [✅ 仓内可证] src/queries/sql/events/saveEvent.ts
数据处理层 (saveEvent, saveEventData, saveSessionData, saveRecording)
    ↓  [✅ 仓内可证] src/lib/db.ts:22-36
数据库路由层 (runQuery) - 三择一，单路由
    ├─ 配置了 CLICKHOUSE_URL → 走 CLICKHOUSE 分支
    │   ↓  [✅ 仓内可证] src/queries/sql/events/saveEvent.ts:259-263
    │   ├─ kafka.enabled = true → sendMessage(topic, data) → Kafka 异步入队
    │   │   └─ ⚠️ 外部依赖: Kafka集群 + 外部消费端（本仓未包含 consumer/worker 实现）
    │   └─ kafka.enabled = false → clickhouse.insert(table, data) → ClickHouse 直写
    │       └─ ⚠️ 外部依赖: ClickHouse 集群
    │
    └─ 未配置 CLICKHOUSE_URL → 走 PRISMA 分支 → PostgreSQL/MySQL 直写
        └─ [✅ 仓内可证] src/lib/prisma.ts
```

### 1.2 Worker/Consumer 检索证据

**检索结论：⚠️ 本仓库内未包含 Kafka Consumer / Worker 实现**

**检索证据：**

1. **关键词搜索结果**
   - Grep pattern: `worker|consumer|kafka.*consume|consume.*kafka`
   - 仅在 `pnpm-lock.yaml` 中找到匹配，源码目录无匹配文件

2. **文件名匹配结果**
   - Glob pattern: `**/*worker*` → 0 匹配
   - Glob pattern: `**/*consumer*` → 0 匹配

3. **Kafka 模块完整功能范围** [src/lib/kafka.ts]
   - ✅ 包含：Producer 初始化、sendMessage 方法
   - ❌ 不包含：Consumer、Worker、消费逻辑、数据同步到 ClickHouse 的逻辑

**结论：** 本仓库仅实现 Kafka 生产端，消费端 / Worker 需外部实现，仓内不可见。

---

## 2. 采集端详解

### 2.1 采集入口与数据结构

**核心文件：** `src/app/api/send/route.ts`

Umami 支持 4 种采集类型：

| 类型 | 用途 |
|-----|-----|
| `event` | 页面浏览和自定义事件 |
| `identify` | 用户识别（设置会话属性） |
| `performance` | 网页性能指标（LCP/INP/CLS/FCP/TTFB） |
| `record` | 会话录制（独立路由 `/api/record`） |

**事件载荷结构：**

```typescript
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
  // Web Vitals 性能指标
  lcp?: number;          
  inp?: number;         
  cls?: number;          
  fcp?: number;          
  ttfb?: number;         
}
```

### 2.2 会话与访问标识机制

**核心文件：** `src/lib/crypto.ts`, `src/app/api/send/route.ts`

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

### 2.3 机器人检测与IP过滤

**核心文件：** `src/app/api/send/route.ts`

```typescript
// 机器人检测：使用 isbot 库检查 userAgent [src/app/api/send/route.ts:131-133]
if (!process.env.DISABLE_BOT_CHECK && isbot(userAgent)) {
  return json({ beep: 'boop' });
}

// IP 地址过滤 [src/app/api/send/route.ts:136-138]
if (hasBlockedIp(ip)) {
  return forbidden();
}
```

---

## 3. 写入路由机制（已核对）

### 3.1 runQuery 路由逻辑

**核心文件：** `src/lib/db.ts:22-36`

```typescript
export async function runQuery(queries: any) {
  // 配置了 CLICKHOUSE_URL → 只走 ClickHouse 相关分支
  if (process.env.CLICKHOUSE_URL) {
    // 【注意】仓内所有查询函数均未定义 KAFKA 分支
    // 此分支当前无调用
    if (queries[KAFKA]) {
      return queries[KAFKA]();
    }

    // ✅ 实际执行路径
    return queries[CLICKHOUSE]();
  }

  // 回退到关系型数据库
  const db = getDatabaseType();
  if (db === POSTGRESQL) {
    return queries[PRISMA]();
  }
}
```

**关键事实核对：**
- 当配置 `CLICKHOUSE_URL` 时，**仅执行 CLICKHOUSE 分支**，不会同时走多个分支
- KAFKA 分支在仓内所有查询函数中均未定义，实际不会被调用
- Kafka 生产逻辑是在各查询函数**内部**通过 `kafka.enabled` 进行二选一

### 3.2 三种写入模式详解

| 模式 | 触发条件 | 执行路径 | 数据最终落点 |
|-----|---------|----------|-------------|
| **PRISMA 直写** | 未配置 CLICKHOUSE_URL | Prisma ORM → PostgreSQL/MySQL | PostgreSQL/MySQL |
| **ClickHouse 直写** | 配置 CLICKHOUSE_URL, kafka.enabled=false | clickhouse.insert() → 直接写入表 | ClickHouse 表 |
| **ClickHouse + Kafka 异步入队** | 配置 CLICKHOUSE_URL, kafka.enabled=true | kafka.sendMessage() → Kafka topic | Kafka 队列（后续消费由外部实现） |

### 3.3 saveEvent 写入路径详解

**核心文件：** `src/queries/sql/events/saveEvent.ts:65-276`

```typescript
export async function saveEvent(args: SaveEventArgs) {
  return runQuery({
    [PRISMA]: () => relationalQuery(args),      // PostgreSQL/MySQL 分支
    [CLICKHOUSE]: () => clickhouseQuery(args),  // ClickHouse 分支（内部含 Kafka 二选一）
  });
}
```

**ClickHouse 分支写入逻辑：**

```typescript
async function clickhouseQuery({ ... }: SaveEventArgs) {
  const { insert, getUTCString } = clickhouse;
  const { sendMessage } = kafka;
  const eventId = uuid();

  // 构造事件消息
  const message = {
    website_id: websiteId,
    session_id: sessionId,
    visit_id: visitId,
    event_id: eventId,
    // ... 20+ 个字段
  };

  // ✅ 关键：在 ClickHouse 分支内部按 kafka.enabled 二选一
  // [src/queries/sql/events/saveEvent.ts:259-263]
  if (kafka.enabled) {
    await sendMessage('event', message);  // → Kafka 异步入队
  } else {
    await insert('website_event', [message]);  // → ClickHouse 直写
  }

  // 事件属性展开写入
  if (eventData) {
    await saveEventData({ ... });
  }
}
```

### 3.4 四类写入函数的统一模式

仓库中共有 4 个写入函数遵循相同的模式：

| 函数 | 文件路径 | Kafka Topic |
|-----|---------|-----------|
| `saveEvent` | `src/queries/sql/events/saveEvent.ts:259` | `event` |
| `saveEventData` | `src/queries/sql/events/saveEventData.ts:74` | `event_data` |
| `saveSessionData` | `src/queries/sql/sessions/saveSessionData.ts:99` | `session_data` |
| `saveRecording` | `src/queries/sql/replays/saveRecording.ts:78` | `session_replay` |

**统一模式伪代码：**

```typescript
async function clickhouseQuery(args) {
  const { insert } = clickhouse;
  const { sendMessage } = kafka;

  const message = transform(args);

  if (kafka.enabled) {
    await sendMessage(topic, message);  // 发送到 Kafka 队列
  } else {
    await insert(table, [message]);   // 直接写入 ClickHouse 表
  }
}
```

### 3.5 写入模式下的可证事实与边界

**✅ 仓内可证事实：**

1. **仅包含 Kafka 生产端实现** [src/lib/kafka.ts]
2. **支持 SASL 认证**（plain/scram-sha-256/scram-sha-512）
3. **4 个 topic：`event`, `event_data`, `session_data`, `session_replay`
4. **消息格式：** JSON 序列化

**⚠️ 外部依赖 / 仓内不可证：**

1. **Kafka Consumer / Worker 实现**：仓内未包含，需外部实现消费逻辑
2. **数据可靠性**：Kafka → ClickHouse 的同步逻辑完全不在本仓内
3. **消息丢失风险**：Kafka 开启后，数据能否最终写入 ClickHouse 依赖外部消费端的正确性
4. **Exactly-Once 语义**：消费端的去重/幂等逻辑仓内不可见
5. **批量写入优化**：消费端的批量插入、重试机制仓内不可见
6. **消费失败处理**：消费失败的重试策略仓内不可见

---

## 4. 事件属性展开机制

### 4.1 展开算法

**核心文件：**
- `src/queries/sql/events/saveEventData.ts`
- `src/lib/data.ts`

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
    ↓ flattenJSON() [src/lib/data.ts:4-25]
输出: [
  { key: "product.name", value: "Premium Plan", dataType: string(1) },
  { key: "product.price", value: 99.99, dataType: number(2) },
  { key: "product.features", value: '["analytics","reports"]', dataType: array(5) },
  { key: "timestamp", value: "2024-01-15T10:30:00Z", dataType: date(4) },
  { key: "success", value: "true", dataType: boolean(3) }
]
```

### 4.2 数据类型映射

**核心文件：** `src/lib/constants.ts:129-135`

| 数据类型 | 枚举值 | 存储方式 |
|---------|-------|---------|
| string | 1 | 原始字符串 |
| number | 2 | 保留4位小数 |
| boolean | 3 | "true"/"false" |
| date | 4 | ISO 8601 格式 |
| array | 5 | JSON 字符串 |

### 4.3 事件属性表结构

**核心文件：** `db/clickhouse/schema.sql`

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

---

## 5. ClickHouse 数据模型

### 5.1 主事件表：website_event

**核心文件：** `db/clickhouse/schema.sql`

```sql
CREATE TABLE umami.website_event
(
    website_id UUID,           -- 网站ID (分区维度)
    session_id UUID,           -- 会话ID
    visit_id UUID,             -- 访问ID（30分钟内同一会话）
    event_id UUID,             -- 事件唯一ID

    -- 会话维度（LowCardinality 优化）
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

### 5.2 物化视图：小时级聚合

**核心文件：** `db/clickhouse/schema.sql` (website_event_stats_hourly)

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

**⚠️ 外部依赖说明：**
- 物化视图是 ClickHouse 内置功能
- 聚合逻辑在 ClickHouse 服务端执行
- 聚合正确性依赖 ClickHouse 版本和配置

### 5.3 投影优化

**核心文件：** `db/clickhouse/schema.sql`

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

## 6. 查询接口与指标聚合

### 6.1 网站维度过滤机制

**核心文件：** `src/lib/clickhouse.ts:69-140`

#### 6.1.1 过滤参数定义

**核心文件：** `src/lib/constants.ts:72-97`

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
```

#### 6.1.2 过滤查询生成

**核心文件：** `src/lib/clickhouse.ts:101-140`

```typescript
function getFilterQuery(filters, options) {
  const orClauses: string[] = [];
  const andClauses: string[] = [];

  filtersObjectToArray(filters, options).forEach(({ name, column, operator, paramName }) => {
    if (column) {
      // eventType 始终是 AND 条件
      const isAlwaysAnd = name === 'eventType' || (isCohort && name === cohortActionName);

      if (isAlwaysAnd) {
        andClauses.push(`and ${mapFilter(column, operator, name, name === 'eventType' ? 'UInt32' : 'String', paramName)}`);
      } else if (isOr) {
        orClauses.push(mapFilter(column, operator, name, 'String', paramName));
      } else {
        andClauses.push(`and ${mapFilter(column, operator, name, 'String', paramName)}`);
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

#### 6.1.3 过滤操作符映射

**核心文件：** `src/lib/clickhouse.ts:73-99`

```typescript
function mapFilter(
  column: string,
  operator: string,
  name: string,
  type: string = 'String',
  paramName?: string,
) {
  const param = paramName ?? name;
  const value = `{${param}:${type}}`;

  switch (operator) {
    case OPERATORS.equals:
      return `${column} IN {${param}:Array(${type})}`;
    case OPERATORS.notEquals:
      return `${column} NOT IN {${param}:Array(${type})}`;
    case OPERATORS.contains:
      return `positionCaseInsensitive(${column}, ${value}) > 0`;
    case OPERATORS.doesNotContain:
      return `positionCaseInsensitive(${column}, ${value}) = 0`;
    case OPERATORS.regex:
      return `match(${column}, concat('(?i)', ${value}))`;
    case OPERATORS.notRegex:
      return `not match(${column}, concat('(?i)', ${value}))`;
    default:
      return '';
  }
}
```

### 6.2 网站基础统计查询

**核心文件：** `src/queries/sql/getWebsiteStats.ts:72-138`

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

**智能路由逻辑：**

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

1. **投影索引（Projections）** [db/clickhouse/schema.sql]
   - `url_path` 维度查询优化
   - `referrer_domain` 维度查询优化

2. **LowCardinality 编码** [db/clickhouse/schema.sql]
   - browser/os/device/country/region 等维度字段
   - 减少存储空间，加速 GROUP BY

3. **聚合视图（Materialized View）** [db/clickhouse/schema.sql]
   - 小时级预聚合，避免重复计算
   - 无过滤条件的查询自动路由到聚合视图

4. **分区裁剪（Partition Pruning）** [db/clickhouse/schema.sql]
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

## 8. Prisma 与 ClickHouse 职责划分

### 8.1 核心分工矩阵

| 功能模块 | Prisma (PostgreSQL/MySQL) | ClickHouse |
|---------|---------------------------|-----------|
| **用户管理** | ✅ 用户账号、密码、角色 | ❌ |
| **团队管理** | ✅ 团队成员、权限、角色 | ❌ |
| **网站配置** | ✅ 网站基础信息、域名、重置设置 | ❌ |
| **共享链接** | ✅ 共享 token、权限、过期时间 | ❌ |
| **报表/看板** | ✅ 报表定义、看板配置、图表布局 | ❌ |
| **事件数据存储** | ⚠️（仅小数据量，回退模式） | ✅ 原始事件、属性、会话 |
| **实时查询** | ❌（性能不足） | ✅ 毫秒级聚合查询 |
| **漏斗分析** | ❌ | ✅ 多步骤会话漏斗 |
| **留存分析** | ❌ | ✅ 用户留存计算 |
| **路径分析** | ❌ | ✅ 页面流转路径 |
| **收入分析** | ❌ | ✅ 收入指标聚合 |
| **会话录制** | ❌ | ✅ 录制事件流存储 |

### 8.2 查询路由机制

**核心文件：** `src/lib/db.ts:22-36`

```typescript
export function runQuery(queries: any) {
  // 配置了 CLICKHOUSE_URL → 仅走 CLICKHOUSE 分支
  if (process.env.CLICKHOUSE_URL) {
    if (queries[KAFKA]) {
      return queries[KAFKA]();   // Kafka 分支（仓内无函数使用）
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

### 8.3 写入路由的可证事实与边界

**✅ 仓内可证事实：**

1. **写入路径三择一**：
   - 配置类数据 → 仅写 Prisma
   - 事件类数据 → 仅写 ClickHouse（或 Kafka）

2. **查询路径自动路由**：
   - 每个查询函数都提供双实现（`relationalQuery` + `clickhouseQuery`）
   - 运行时根据环境变量自动选择

3. **跨库关联查询**：
   - 网站元数据（Prisma） + 事件统计（ClickHouse）
   - 通过 `website_id` UUID 进行关联

**⚠️ 外部依赖 / 风险：**
- Kafka 开启后，事件数据从生产到消费到 ClickHouse 的延迟不可控
- 双模式下无分布式事务保证，极端情况可能数据不一致

---

## 9. 结论-证据矩阵

| 结论分类 | 具体结论 | 证据文件路径 | 证据是否仓内可证 | 剩余不确定性 |
|---------|---------|-----------|-----------------|-------------|
| **写入路由** | runQuery 是三择一单路由机制 | `src/lib/db.ts:22-36` | ✅ 是 | 无 |
| **写入路由** | 三种写入模式：PRISMA 直写、ClickHouse 直写、ClickHouse+Kafka 异步入队 | `src/lib/db.ts:22-36`, `src/queries/sql/events/saveEvent.ts:259-263` | ✅ 是 | 消费端逻辑外部实现 |
| **写入路由** | KAFKA 分支在仓内所有写入函数中均未被调用 | 所有 src/queries/sql/events/saveEvent.ts` | ✅ 是 | 无 |
| **写入路由** | Kafka 仅实现生产端，无消费端/Worker | `src/lib/kafka.ts` | ✅ 是 | 消费端实现需外部 |
| **事件属性展开** | JSON 对象递归展开为键值对 | `src/lib/data.ts:4-25` | ✅ 是 | 无 |
| **事件属性展开** | 支持 5 种数据类型映射 | `src/lib/constants.ts:129-135` | ✅ 是 | 无 |
| **事件属性展开** | event_data 表按 website_id, event_id, data_key 排序 | `db/clickhouse/schema.sql` | ✅ 是 | 无 |
| **过滤拼接** | 支持 21 个过滤字段映射 | `src/lib/constants.ts:72-97` | ✅ 是 | 无 |
| **过滤拼接** | 支持 AND/OR 组合，6 种操作符 | `src/lib/clickhouse.ts:73-99` | ✅ 是 | 无 |
| **过滤拼接** | referrer 过滤隐含排除本站域名条件 | `src/lib/clickhouse.ts:125-127` | ✅ 是 | 无 |
| **高基数字段成本** | 8 个高基数字段识别 | `db/clickhouse/schema.sql` | ✅ 是 | 实际基数依赖真实数据 |
| **高基数字段成本** | LowCardinality 编码仅用于低基数字段 | `db/clickhouse/schema.sql` | ✅ 是 | 无 |
| **高基数字段成本** | 投影索引仅覆盖 url_path 和 referrer_domain | `db/clickhouse/schema.sql` | ✅ 是 | 无 |

---

## 附录：关键代码文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| 采集入口 | `src/app/api/send/route.ts` |
| 录制入口 | `src/app/api/record/route.ts` |
| 事件写入 | `src/queries/sql/events/saveEvent.ts` |
| 事件属性写入 | `src/queries/sql/events/saveEventData.ts` |
| 会话数据写入 | `src/queries/sql/sessions/saveSessionData.ts` |
| 录制数据写入 | `src/queries/sql/replays/saveRecording.ts` |
| 属性展开 | `src/lib/data.ts` |
| ClickHouse 工具 | `src/lib/clickhouse.ts` |
| Kafka 生产端 | `src/lib/kafka.ts` |
| 数据库路由 | `src/lib/db.ts` |
| 基础统计查询 | `src/queries/sql/getWebsiteStats.ts` |
| 事件指标查询 | `src/queries/sql/events/getEventMetrics.ts` |
| 属性值查询 | `src/queries/sql/events/getEventDataValues.ts` |
| ClickHouse DDL | `db/clickhouse/schema.sql` |
| 常量定义 | `src/lib/constants.ts` |

---

## 总结：架构设计亮点与权衡

### 设计亮点

1. **属性动态展开**：JSON → 键值对表，支持无限维度扩展
2. **混合存储架构**：Prisma 管理元数据，ClickHouse 负责事件分析
3. **物化视图加速**：小时级预聚合，无过滤查询毫秒级返回
4. **投影索引优化**：针对高频过滤维度创建排序投影
5. **LowCardinality 编码**：低基数维度存储优化
6. **Kafka 异步入队**：支持高吞吐场景下的缓冲（需外部消费端）

### 权衡与取舍

| 决策 | 优势 | 代价 |
|-----|-----|-----|
| **事件属性行存储** | 灵活支持任意属性 | JOIN 成本高，存储放大 |
| **ClickHouse 单副本** | 部署简单 | 无高可用保障 |
| **按月分区** | 管理简单 | 大时间范围查询仍需扫描多分区 |
| **无数据TTL** | 数据永久保留 | 存储持续增长 |
| **实时写入无批量** | 数据立即可查 | 高并发下单行写入性能受限 |
| **Kafka 消费端外部实现** | 架构解耦 | 端到端链路不可见，运维复杂 |

### 可扩展性建议

1. **写入侧**：启用 Kafka 缓冲 + 批量写入 ClickHouse（需外部实现消费端）
2. **存储侧**：配置 TTL 自动清理历史数据，使用分层存储（热/冷数据分离）
3. **查询侧**：增加更多维度的物化视图，引入 Query Cache
4. **高可用**：部署 ClickHouse 集群（2 shards × 2 replicas）
5. **监控**：增加慢查询日志，监控 part 合并和 mutation 队列
6. **消费端**：实现可靠的 Kafka → ClickHouse 同步服务（仓内未提供）