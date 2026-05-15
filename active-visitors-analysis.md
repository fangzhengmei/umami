# 在线访客数计算流程分析报告

## 一、概述

本报告详细梳理了 Umami 网站分析系统中「当前在线访客数」指标从前端事件采集到后端数据存储、计算，最终在前端展示的完整技术流程。

## 二、完整流程架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  前端 Tracker   │────▶│  API 收集层    │────▶│  数据存储层    │────▶│  数据查询层    │
│  (事件采集)    │     │  (/api/send)   │     │  (数据库)      │     │  (SQL 查询)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
                                                              │
                                                              ▼
                                                    ┌─────────────────┐
                                                    │  API 展示层    │
                                                    │  (/active)     │
                                                    └─────────────────┘
                                                              │
                                                              ▼
                                                    ┌─────────────────┐
                                                    │  前端展示层    │
                                                    │  (React 组件)  │
                                                    └─────────────────┘
```

---

## 三、详细流程分解

### 3.1 第一阶段：事件采集 (Frontend Tracker)

#### 3.1.1 Tracker 脚本
- **位置**: 嵌入在被监控网站的页面中
- **功能**: 自动采集页面浏览事件和自定义事件

#### 3.1.2 采集的数据字段
```typescript
interface TrackedProperties {
  hostname: string;      // 网站域名
  language: string;      // 浏览器语言
  referrer: string;      // 页面来源
  screen: string;        // 屏幕分辨率
  title: string;         // 页面标题
  url: string;           // 页面路径
  website: string;       // 网站 ID (必需)
}
```

#### 3.1.3 触发时机
1. 页面加载完成时自动触发 pageview 事件
2. 通过 `umami.track()` 手动触发自定义事件
3. 支持传递自定义事件数据 `EventData`

---

### 3.2 第二阶段：API 收集层 (`/api/send`)

#### 3.2.1 核心文件
- **路径**: `src/app/api/send/route.ts`

#### 3.2.2 处理流程

**步骤 1: 请求验证与解析**
- 解析请求体，验证 schema
- 提取 `type` (event/identify/performance) 和 `payload`
- 验证网站存在性

**步骤 2: 客户端信息检测**
```typescript
// 从请求中提取
const { ip, userAgent, device, browser, os, country, region, city } = await getClientInfo(request, payload);
```

**步骤 3: 机器人检测**
```typescript
if (!process.env.DISABLE_BOT_CHECK && isbot(userAgent)) {
  return json({ beep: 'boop' }); // 不记录机器人访问
}
```

**步骤 4: IP 黑名单检查**
```typescript
if (hasBlockedIp(ip)) {
  return forbidden();
}
```

**步骤 5: Session 生成**
```typescript
// 基于 IP + UserAgent + Salt 生成唯一 sessionId
const sessionId = id ? uuid(sourceId, id) : uuid(sourceId, ip, userAgent, sessionSalt);
```

**步骤 6: Visit 过期处理**
```typescript
// Visit 30 分钟过期
const VISIT_EXPIRE_TIME = 1800; // 秒
if (!timestamp && now - iat > VISIT_EXPIRE_TIME) {
  visitId = uuid(sessionId, visitSalt); // 重新生成 visitId
  iat = now;
}
```

**步骤 7: 事件分类处理**
| 事件类型 | 说明 | eventType 值 |
|---------|------|-------------|
| pageView | 页面浏览 | 1 |
| customEvent | 自定义事件 | 2 |
| linkEvent | 链接事件 | 3 |
| pixelEvent | Pixel 事件 | 4 |
| performance | 性能指标 | 5 |

**步骤 8: 保存事件**
```typescript
await saveEvent({
  websiteId: sourceId,
  sessionId,
  visitId,
  eventType,
  createdAt,
  // ... 其他字段
});
```

**步骤 9: 返回缓存 Token**
```typescript
const token = createToken({ websiteId, sessionId, visitId, iat }, secret());
return json({ cache: token, sessionId, visitId });
```

---

### 3.3 第三阶段：数据存储层

#### 3.3.1 核心文件
- **路径**: `src/queries/sql/events/saveEvent.ts`

#### 3.3.2 双数据库支持
系统同时支持关系型数据库 (MySQL/PostgreSQL) 和 ClickHouse 列式数据库。

**关系型数据库存储 (Prisma)**
```typescript
await prisma.client.websiteEvent.create({
  data: {
    id: websiteEventId,         // UUID
    websiteId,                  // 网站 ID
    sessionId,                  // 会话 ID (访客唯一标识)
    visitId,                    // 访问 ID
    urlPath,                    // URL 路径
    urlQuery,                   // URL 查询参数
    referrerPath,               // 来源路径
    referrerQuery,              // 来源查询参数
    referrerDomain,             // 来源域名
    pageTitle,                  // 页面标题
    eventType,                  // 事件类型
    eventName,                  // 事件名称
    createdAt,                  // 创建时间
    // ... 其他字段
  },
});
```

**ClickHouse 存储 (高性能分析)**
```typescript
const message = {
  website_id: websiteId,
  session_id: sessionId,       // 关键：用于访客去重
  visit_id: visitId,
  event_id: eventId,
  country,
  region,
  city,
  url_path: urlPath,
  created_at: getUTCString(createdAt),
  browser,
  os,
  device,
  // ... 其他字段
};

// 可选：Kafka 消息队列
if (kafka.enabled) {
  await sendMessage('event', message);
} else {
  await insert('website_event', [message]);
}
```

#### 3.3.3 关键数据模型
**website_event 表核心字段**:
| 字段 | 类型 | 说明 |
|------|------|------|
| session_id | VARCHAR(36) | **访客唯一标识，用于在线访客统计** |
| website_id | VARCHAR(36) | 网站 ID |
| created_at | DATETIME | 事件发生时间 |
| event_type | INT | 事件类型 |

---

### 3.4 第四阶段：在线访客数计算

#### 3.4.1 核心文件
- **路径**: `src/queries/sql/getActiveVisitors.ts`

#### 3.4.2 计算逻辑

**时间窗口定义**
```typescript
// 在线访客的定义：过去 5 分钟内有活动的访客
const ACTIVE_WINDOW_MINUTES = 5;
const startDate = subMinutes(new Date(), ACTIVE_WINDOW_MINUTES);
```

**SQL 查询核心**
```sql
-- 关系型数据库 (MySQL/PostgreSQL)
SELECT COUNT(DISTINCT session_id) AS visitors
FROM website_event
WHERE website_id = ?
  AND created_at >= ?

-- ClickHouse
SELECT COUNT(DISTINCT session_id) AS visitors
FROM website_event
WHERE website_id = ?
  AND created_at >= ?
```

**关键技术点**:
1. 使用 `COUNT(DISTINCT session_id)` 进行访客去重
2. 时间窗口过滤：只统计过去 5 分钟内的事件
3. 按 website_id 隔离不同网站的数据

#### 3.4.3 两种数据库实现

**Prisma (关系型)**
```typescript
async function relationalQuery(websiteId: string) {
  const { rawQuery } = prisma;
  const startDate = subMinutes(new Date(), 5);

  const result = await rawQuery(
    `
    select count(distinct session_id) as "visitors"
    from website_event
    where website_id = {{websiteId::uuid}}
    and created_at >= {{startDate}}
    `,
    { websiteId, startDate },
    'getActiveVisitors',
  );

  return result?.[0] ?? null;
}
```

**ClickHouse (列式)**
```typescript
async function clickhouseQuery(websiteId: string): Promise<{ x: number }> {
  const { rawQuery } = clickhouse;
  const startDate = subMinutes(new Date(), 5);

  const result = await rawQuery(
    `
    select
      count(distinct session_id) as "visitors"
    from website_event
    where website_id = {websiteId:UUID}
      and created_at >= {startDate:DateTime64}
    `,
    { websiteId, startDate },
    'getActiveVisitors',
  );

  return result[0] ?? null;
}
```

---

### 3.5 第五阶段：API 展示层

#### 3.5.1 获取在线访客数 API

**核心文件**: `src/app/api/websites/[websiteId]/active/route.ts`

**请求处理流程**:
```typescript
export async function GET(request: Request, { params }) {
  // 1. 解析请求，验证权限
  const { auth, error } = await parseRequest(request);
  const { websiteId } = await params;

  // 2. 权限检查
  if (!(await canViewWebsite(auth, websiteId))) {
    return unauthorized();
  }

  // 3. 查询在线访客数
  const visitors = await getActiveVisitors(websiteId);

  // 4. 返回结果
  return json(visitors);
}
```

**响应格式**:
```json
{
  "visitors": 42
}
```

#### 3.5.2 实时数据 API (补充)

**核心文件**: `src/app/api/realtime/[websiteId]/route.ts`

**时间范围**:
```typescript
// 实时数据统计过去 30 分钟
const REALTIME_RANGE = 30; // 分钟

const filters = await getQueryFilters(
  {
    ...query,
    startAt: subMinutes(startOfMinute(new Date()), REALTIME_RANGE).getTime(),
    endAt: Date.now(),
  },
  websiteId,
);
```

---

### 3.6 第六阶段：前端展示层

#### 3.6.1 数据查询 Hook

**核心文件**: `src/components/hooks/queries/useRealtimeQuery.ts`

```typescript
export function useRealtimeQuery(websiteId: string) {
  const { get, useQuery } = useApi();
  
  // 每 10 秒自动刷新一次
  const REALTIME_INTERVAL = 10000; // 10 秒

  const { data, isLoading, error } = useQuery<RealtimeData>({
    queryKey: ['realtime', { websiteId }],
    queryFn: async () => {
      return get(`/realtime/${websiteId}`);
    },
    enabled: !!websiteId,
    refetchInterval: REALTIME_INTERVAL, // 自动轮询
  });

  return { data, isLoading, error };
}
```

#### 3.6.2 展示组件

**核心文件**: `src/app/(main)/websites/[websiteId]/realtime/RealtimeHeader.tsx`

```typescript
export function RealtimeHeader({ data }) {
  const { t, labels } = useMessages();
  const { totals } = data || {};

  return (
    <MetricsBar>
      <MetricCard label={t(labels.views)} value={totals.views} />
      <MetricCard label={t(labels.visitors)} value={totals.visitors} />
      <MetricCard label={t(labels.events)} value={totals.events} />
      <MetricCard label={t(labels.countries)} value={totals.countries} />
    </MetricsBar>
  );
}
```

---

## 四、关键配置常量

| 常量名称 | 值 | 说明 | 文件位置 |
|---------|-----|------|---------|
| `ACTIVE_WINDOW_MINUTES` | 5 | 在线访客时间窗口（分钟） | `getActiveVisitors.ts` |
| `REALTIME_RANGE` | 30 | 实时数据统计范围（分钟） | `lib/constants.ts:32` |
| `REALTIME_INTERVAL` | 10000 | 前端自动刷新间隔（毫秒） | `lib/constants.ts:33` |
| `VISIT_EXPIRE_TIME` | 1800 | Visit 过期时间（秒） | `api/send/route.ts:172` |

---

## 五、核心技术要点

### 5.1 访客唯一性保证
- **算法**: `UUID(websiteId + IP + UserAgent + Salt)`
- **Salt 轮换**: 按月轮换 Salt 保护隐私
- **优点**: 无需 Cookie，跨设备同一 IP 会被识别为同一访客

### 5.2 在线状态定义
- **定义**: 过去 5 分钟内有任何事件（页面浏览、自定义事件等）
- **优点**: 实时性强，简单高效
- **注意**: 用户关闭页面后最长 5 分钟后才会从在线统计中消失

### 5.3 性能优化
1. **ClickHouse 优化**: 列式存储，COUNT DISTINCT 性能优异
2. **缓存机制**: 前端使用 React Query 缓存，每 10 秒刷新
3. **索引优化**: `website_event` 表需对 `(website_id, created_at, session_id)` 建立联合索引

### 5.4 隐私保护
- 不使用 Cookie 追踪用户
- Salt 轮换机制，无法长期追踪同一用户
- 支持 IP 黑名单过滤

---

## 六、数据流时序图

```
  浏览器 (访客)          Umami Tracker      Umami Server        Database        仪表盘
      │                    │                   │                  │              │
      │  访问页面          │                   │                  │              │
      │───────────────────▶│                   │                  │              │
      │                    │  采集页面信息     │                  │              │
      │                    │  hostname, url    │                  │              │
      │                    │──────────────────▶│                  │              │
      │                    │                   │  生成 sessionId  │              │
      │                    │                   │  UUID(ip, ua)    │              │
      │                    │                   │─────────────────▶│              │
      │                    │                   │  INSERT event    │              │
      │                    │                   │                  │              │
      │                    │                   │◀─────────────────│              │
      │                    │◀──────────────────│                  │              │
      │                    │     cache token   │                  │              │
      │                    │                   │                  │              │
      │                    │                   │                  │              │
      │                    │                   │  [10秒后]         │              │
      │                    │                   │◀────────────────────────────────│
      │                    │                   │  GET /realtime   │              │
      │                    │                   │─────────────────▶│              │
      │                    │                   │  COUNT(DISTINCT  │              │
      │                    │                   │    session_id)    │              │
      │                    │                   │  过去 5 分钟      │              │
      │                    │                   │◀─────────────────│              │
      │                    │                   │                  │              │
      │◀────────────────────────────────────────────────────────────────────────│
      │    { visitors: 42 }
```

---

## 七、总结

### 7.1 完整流程回顾
1. **采集**: 页面加载 → Tracker 采集信息 → 发送 `/api/send`
2. **处理**: 服务器验证 → 生成 sessionId → 识别用户
3. **存储**: 写入 `website_event` 表，携带 session_id
4. **计算**: 查询过去 5 分钟 → `COUNT(DISTINCT session_id)`
5. **展示**: API 返回 → 前端每 10 秒刷新 → 展示在线访客数

### 7.2 核心指标定义
| 指标 | 定义 | 计算方式 |
|------|------|---------|
| 在线访客数 | 过去 5 分钟内有活动的独立访客数 | `COUNT(DISTINCT session_id)` WHERE created_at >= NOW() - 5min |
| 实时访客数 | 过去 30 分钟内的访客统计 | 同上述逻辑，时间窗口 30 分钟 |

### 7.3 关键设计决策
1. **无 Cookie 设计**: 使用 IP + UserAgent + Salt 生成匿名标识
2. **滑动时间窗口**: 5 分钟窗口保证实时性
3. **双数据库架构**: 关系型数据库保证一致性，ClickHouse 保证性能
4. **定时轮询**: 前端 10 秒刷新平衡实时性和服务器压力

---

**报告生成时间**: 2026-05-15
**分析版本**: Umami v2.x
**代码路径参考**:
- Tracker API: `src/tracker/index.d.ts`
- 事件收集: `src/app/api/send/route.ts`
- 事件保存: `src/queries/sql/events/saveEvent.ts`
- 在线访客计算: `src/queries/sql/getActiveVisitors.ts`
- 实时数据 API: `src/app/api/realtime/[websiteId]/route.ts`
- 前端 Hook: `src/components/hooks/queries/useRealtimeQuery.ts`
