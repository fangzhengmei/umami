# 在线访客指标计算流程分析报告

## 一、概述

本报告详细梳理 Umami 网站分析系统中**两类在线访客指标**的完整计算流程，包括：
1. **页面头部在线人数** - 各页面右上角显示的实时在线用户数
2. **实时页访客统计** - 「实时」标签页中显示的访客统计数据

本文档将从前端事件采集、后端数据存储、计算逻辑、API 接口到前端展示的完整链路进行深入分析。

---

## 二、指标概览对比

| 维度 | 页面头部在线人数 | 实时页访客统计 |
|------|----------------|--------------|
| **展示位置** | 所有网站分析页面的右上角 | 「实时」标签页顶部指标栏 |
| **对应组件** | `ActiveUsers.tsx` | `RealtimeHeader.tsx` |
| **API 接口** | `/api/websites/{websiteId}/active` | `/api/realtime/{websiteId}` |
| **核心查询** | `getActiveVisitors()` | `getRealtimeData()` |
| **时间窗口** | 5 分钟 | 30 分钟 |
| **前端刷新频率** | 60 秒 | 10 秒 |
| **统计粒度** | 单一数值 | 含时间序列的多维度统计 |

---

## 三、第一阶段：事件采集与 Session 生成

### 3.1 Tracker 事件采集

**核心文件**: `src/tracker/index.d.ts`

**采集属性**:
```typescript
interface TrackedProperties {
  hostname: string;      // 网站域名
  language: string;      // 浏览器语言
  referrer: string;      // 来源页面
  screen: string;        // 屏幕分辨率
  title: string;         // 页面标题
  url: string;           // 页面路径
  website: string;       // 网站 ID（必需）
}
```

**事件类型**:
- `pageView` (type=1): 页面浏览
- `customEvent` (type=2): 自定义事件（支持 `name` + `data`）
- `identify` (type 特殊): 用户识别（传入 `id` 即 distinctId）

---

### 3.2 SessionId 生成分支逻辑

**核心文件**: `src/app/api/send/route.ts` + `src/lib/crypto.ts`

#### 3.2.1 生成流程图

```
                    传入请求参数
                        │
                        ▼
              ┌──────────────────────┐
              │   是否有 distinct_id? │
              │    (payload.id)       │
              └──────────┬───────────┘
                         │
           ┌─────────────┴─────────────┐
           │ 有                         │ 无
           ▼                            ▼
    ┌───────────────┐          ┌─────────────────┐
    │ uuid(         │          │ uuid(           │
    │   websiteId,  │          │   websiteId,    │
    │   id          │          │   ip,           │
    │ )             │          │   userAgent,    │
    └───────┬───────┘          │   sessionSalt   │
            │                  └────────┬────────┘
            ▼                           ▼
    ┌─────────────────────────────────────────────┐
    │            最终 sessionId                    │
    │  v5(hash(...args, secret()), v5.DNS)        │
    └─────────────────────────────────────────────┘
```

#### 3.2.2 分支一：Distinct ID 优先模式

**触发条件**: 当 tracker 调用时传入了 `id` 参数
```javascript
// 前端调用示例
umami.identify('user-123');
// 或
umami.track({ website: 'xxx', id: 'user-123' });
```

**生成算法**:
```typescript
// src/app/api/send/route.ts:147
const sessionId = uuid(sourceId, id);
```

**特点**:
- 基于 `websiteId + 用户自定义 id` 生成
- 跨设备、跨网络可识别同一用户
- 适用于登录态用户追踪

#### 3.2.3 分支二：匿名访客模式

**触发条件**: 未传入 `id` 参数（默认情况）

**生成算法**:
```typescript
// src/app/api/send/route.ts:147
const sessionId = uuid(sourceId, ip, userAgent, sessionSalt);
```

**输入因子**:
1. `sourceId`: 网站 ID (websiteId / linkId / pixelId)
2. `ip`: 客户端 IP 地址
3. `userAgent`: 浏览器 User-Agent 字符串
4. `sessionSalt`: 轮换 Salt 保护隐私

#### 3.2.4 UUID 生成内核

**核心文件**: `src/lib/crypto.ts:60-66`

```typescript
export function uuid(...args: any) {
  if (args.length) {
    // 确定性 UUID v5：相同输入产生相同输出
    return v5(hash(...args, secret()), v5.DNS);
  }

  // 随机 UUID v4 或 v7
  return process.env.USE_UUIDV7 ? v7() : v4();
}

export function hash(...args: string[]) {
  return crypto.createHash('sha512').update(args.join('')).digest('hex');
}
```

**关键点**:
- 使用 **UUID v5**（确定性哈希）保证相同输入产生相同 sessionId
- 哈希算法: SHA-512
- 混入系统 `secret()` 防止逆向工程

---

### 3.3 Salt 轮换配置

**核心文件**: `src/lib/crypto.ts:72-78`

#### 3.3.1 轮换算法

```typescript
export function getSalt(saltRotation: string, createdAt: Date): string {
  return hash(
    (saltRotation === 'day' 
      ? startOfDay 
      : saltRotation === 'week' 
        ? startOfWeek 
        : startOfMonth)(createdAt).toUTCString()
  );
}
```

#### 3.3.2 配置选项

通过环境变量 `SALT_ROTATION` 配置：

| 配置值 | 轮换频率 | 说明 |
|-------|---------|------|
| `day` | 每天 | Salt 每日 UTC 0 点刷新 |
| `week` | 每周 | Salt 每周一 UTC 0 点刷新 |
| `month` | 每月 | **默认**，每月 1 日 UTC 0 点刷新 |

**代码引用**:
```typescript
// src/app/api/send/route.ts:143
const saltRotation = process.env.SALT_ROTATION || 'month';
const sessionSalt = getSalt(saltRotation, createdAt);
```

#### 3.3.3 隐私保护效果

| 场景 | 效果 |
|-----|------|
| 同一用户同一天内访问 | sessionId 相同，可识别为同一访客 |
| 同一用户跨天访问 | Salt 轮换，产生不同 sessionId，无法跨天追踪 |
| 同一用户跨月访问 | 完全不同 sessionId，匿名化程度最高 |

---

## 四、第二阶段：数据存储

**核心文件**: `src/queries/sql/events/saveEvent.ts`

### 4.1 双数据库架构

#### 4.1.1 关系型数据库 (PostgreSQL/MySQL)

```typescript
await prisma.client.websiteEvent.create({
  data: {
    id: websiteEventId,         // 事件 UUID
    websiteId,                  // 网站 ID
    sessionId,                  // ← 访客唯一标识（关键）
    visitId,                    // 访问 ID
    urlPath,                    // URL 路径
    eventType,                  // 事件类型
    eventName,                  // 事件名称
    distinctId,                 // 用户自定义 ID（如有）
    createdAt,                  // 事件时间
    // ... 其他字段
  },
});
```

#### 4.1.2 ClickHouse 列式存储

```typescript
const message = {
  website_id: websiteId,
  session_id: sessionId,       // ← 访客唯一标识（关键）
  visit_id: visitId,
  event_name: eventName,
  distinct_id: distinctId,     // 用户自定义 ID
  created_at: getUTCString(createdAt),
  // ... 其他字段
};

// 可选 Kafka 缓冲
if (kafka.enabled) {
  await sendMessage('event', message);
} else {
  await insert('website_event', [message]);
}
```

### 4.2 关键字段说明

| 字段 | 说明 | 用途 |
|------|------|------|
| `session_id` | 会话唯一标识 | **核心**，用于访客去重统计 |
| `distinct_id` | 用户自定义 ID | identify 调用时传入，优先级更高 |
| `created_at` | 事件时间戳 | 时间窗口过滤 |
| `website_id` | 网站 ID | 数据隔离 |

---

## 五、第三阶段：两类指标计算逻辑

### 5.1 指标一：页面头部在线人数

**核心文件**: `src/queries/sql/getActiveVisitors.ts`

#### 5.1.1 计算逻辑

**时间窗口定义**:
```typescript
// 过去 5 分钟内有任何事件的访客
const ACTIVE_WINDOW_MINUTES = 5;
const startDate = subMinutes(new Date(), ACTIVE_WINDOW_MINUTES);
```

**SQL 查询**:
```sql
-- PostgreSQL/MySQL
SELECT COUNT(DISTINCT session_id) AS visitors
FROM website_event
WHERE website_id = {{websiteId}}
  AND created_at >= {{startDate}}

-- ClickHouse
SELECT COUNT(DISTINCT session_id) AS visitors
FROM website_event
WHERE website_id = {websiteId:UUID}
  AND created_at >= {startDate:DateTime64}
```

#### 5.1.2 API 接口

**路径**: `src/app/api/websites/[websiteId]/active/route.ts`

```typescript
export async function GET(request, { params }) {
  const { auth, error } = await parseRequest(request);
  const { websiteId } = await params;

  // 权限校验
  if (!(await canViewWebsite(auth, websiteId))) {
    return unauthorized();
  }

  // 调用计算逻辑
  const visitors = await getActiveVisitors(websiteId);

  return json(visitors);
}
```

**响应格式**:
```json
{
  "visitors": 42
}
```

---

### 5.2 指标二：实时页访客统计

**核心文件**: `src/queries/sql/getRealtimeData.ts`

#### 5.2.1 计算逻辑

**时间窗口定义**:
```typescript
// src/app/api/realtime/[websiteId]/route.ts:27
const REALTIME_RANGE = 30;  // 过去 30 分钟

const filters = await getQueryFilters(
  {
    ...query,
    startAt: subMinutes(startOfMinute(new Date()), REALTIME_RANGE).getTime(),
    endAt: Date.now(),
  },
  websiteId,
);
```

**多维度聚合逻辑**:
```typescript
export async function getRealtimeData(websiteId: string, filters: QueryFilters) {
  // 并行查询三类数据
  const [activity, pageviews, sessions] = await Promise.all([
    getRealtimeActivity(websiteId, filters),  // 最近 100 条活动
    getPageviewStats(websiteId, filters),     // 浏览量时间序列
    getSessionStats(websiteId, filters),      // 访客数时间序列
  ]);

  const uniques = new Set();  // 用于去重统计

  const { countries, urls, referrers, events } = activity.reverse().reduce(
    (obj, event) => {
      const { sessionId, urlPath, referrerDomain, country } = event;

      // 首次出现的会话计入国家统计
      if (!uniques.has(sessionId)) {
        uniques.add(sessionId);
        increment(countries, country);
      }

      increment(urls, urlPath);
      increment(referrers, referrerDomain);

      return obj;
    },
    { countries: {}, urls: {}, referrers: {}, events: [] }
  );

  return {
    countries,    // 国家分布
    urls,         // 页面分布
    referrers,    // 来源分布
    events,       // 事件日志
    series: {     // 时间序列
      views: pageviews,
      visitors: sessions,
    },
    totals: {     // ← 顶部指标栏数值
      views: pageviews.reduce((sum, { y }) => Number(sum) + Number(y), 0),
      visitors: sessions.reduce((sum, { y }) => Number(sum) + Number(y), 0),
      events: activity.filter(e => e.eventName).length,
      countries: Object.keys(countries).length,
    },
    timestamp: Date.now(),
  };
}
```

#### 5.2.2 访客数时间序列计算

**核心文件**: `src/queries/sql/sessions/getSessionStats.ts`

```sql
-- ClickHouse 按分钟粒度聚合
SELECT
  ${getDateSQL('created_at', 'minute', timezone)} AS t,
  uniq(session_id) AS y          -- uniq = ClickHouse 近似去重
FROM website_event
WHERE website_id = {websiteId}
  AND created_at BETWEEN {startDate} AND {endDate}
  AND event_type NOT IN (2, 5)   -- 排除自定义事件、性能事件
GROUP BY t
ORDER BY t
```

**要点**:
- 排除 `event_type=2`（自定义事件）和 `event_type=5`（性能事件）
- 只统计页面浏览类事件
- 按分钟粒度聚合，用于绘制实时趋势图

---

## 六、第四阶段：前端展示层

### 6.1 页面头部在线人数展示

**组件**: `src/components/metrics/ActiveUsers.tsx`

**调用链**:
```
WebsiteHeader.tsx
    ↳ ActiveUsers.tsx
        ↳ useActyiveUsersQuery()  ← 注意拼写（typo）
            ↳ GET /api/websites/{websiteId}/active
```

**代码**:
```typescript
export function ActiveUsers({
  websiteId,
  value,
  refetchInterval = 60000,  // ← 默认 60 秒刷新
}: {
  websiteId: string;
  value?: number;
  refetchInterval?: number;
}) {
  const { t, labels } = useMessages();
  const { data } = useActyiveUsersQuery(websiteId, { refetchInterval });

  const count = data?.visitors || 0;

  if (count === 0) return null;

  return (
    <StatusLight variant="success">
      <Text size="sm" weight="medium">
        {count} {t(labels.online)}
      </Text>
    </StatusLight>
  );
}
```

**Hook 定义** (`src/components/hooks/queries/useActiveUsersQuery.ts`):
```typescript
export function useActyiveUsersQuery(websiteId: string, options?: ReactQueryOptions) {
  const { get, useQuery } = useApi();
  return useQuery({
    queryKey: ['websites:active', websiteId],
    queryFn: () => get(`/websites/${websiteId}/active`),
    enabled: !!websiteId,
    ...options,
  });
}
```

> ⚠️ **注意**: 函数名存在拼写错误 `useActyiveUsersQuery` (应为 `useActiveUsersQuery`)

---

### 6.2 实时页访客统计展示

**组件**: `src/app/(main)/websites/[websiteId]/realtime/RealtimeHeader.tsx`

**调用链**:
```
RealtimePage.tsx
    ↳ useRealtimeQuery()
        ↳ GET /api/realtime/{websiteId}
            ↳ RealtimeHeader.tsx
                ↳ MetricCard (views / visitors / events / countries)
```

**Hook 定义** (`src/components/hooks/queries/useRealtimeQuery.ts`):
```typescript
export function useRealtimeQuery(websiteId: string) {
  const { get, useQuery } = useApi();
  return useQuery({
    queryKey: ['realtime', { websiteId }],
    queryFn: async () => get(`/realtime/${websiteId}`),
    enabled: !!websiteId,
    refetchInterval: 10000,  // ← 10 秒自动刷新
  });
}
```

**展示组件**:
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

## 七、两类指标详细对比

| 对比维度 | 页面头部在线人数 | 实时页访客统计 |
|---------|---------------|--------------|
| **指标名称** | Active Users / 在线 | Visitors / 访客 |
| **展示位置** | 所有页面右上角绿色圆点旁 | 实时页顶部指标栏 |
| **API 路径** | `/api/websites/{id}/active` | `/api/realtime/{id}` |
| **查询函数** | `getActiveVisitors()` | `getRealtimeData()` |
| **时间窗口** | 5 分钟 | 30 分钟 |
| **刷新频率** | 60 秒 | 10 秒 |
| **输出格式** | 单一数值 `{ visitors: 42 }` | 完整对象 `{ totals: { visitors: 156, ... }, series, countries, ... }` |
| **事件范围** | 所有事件类型（含自定义事件） | 排除 `event_type=2,5`（仅页面浏览类） |
| **去重方式** | `COUNT(DISTINCT session_id)` | 30 分钟内 session 聚合求和 |
| **典型场景** | 快速了解当前人气 | 深入分析实时流量分布 |

---

## 八、完整数据流时序图

```
  访客浏览器             Umami 服务器             数据库           管理员仪表盘
      │                      │                      │                 │
      │ 访问网站             │                      │                 │
      │─────────────────────▶│                      │                 │
      │                      │  生成 sessionId      │                 │
      │                      │  (见 3.2 分支)       │                 │
      │                      │                      │                 │
      │                      │  INSERT website_event│                 │
      │                      │─────────────────────▶│                 │
      │                      │                      │                 │
      │                      │◀─────────────────────│                 │
      │◀─────────────────────│                      │                 │
      │   cache token         │                      │                 │
      │                      │                      │                 │
      │                      │                      │                 │
      │ [ 页面头部在线人数查询 ]                       │                 │
      │                      │                      │                 │
      │                      │◀────────────────────────────────────────│
      │                      │   GET /active (每 60 秒)               │
      │                      │                      │                 │
      │                      │  SELECT COUNT(DISTINCT session_id)     │
      │                      │  WHERE created_at >= NOW() - 5min      │
      │                      │─────────────────────▶│                 │
      │                      │                      │                 │
      │                      │◀─────────────────────│                 │
      │                      │                      │                 │
      │◀─────────────────────────────────────────────────────────────│
      │   { visitors: 42 }   │                      │                 │
      │   【 绿色圆点 + 42 online 】                                     │
      │                      │                      │                 │
      │                      │                      │                 │
      │ [ 实时页访客统计查询 ]                       │                 │
      │                      │                      │                 │
      │                      │◀────────────────────────────────────────│
      │                      │   GET /realtime (每 10 秒)             │
      │                      │                      │                 │
      │                      │  查询过去 30 分钟数据                   │
      │                      │  ── activity (最近 100 条)              │
      │                      │  ── pageviews (按分钟)                  │
      │                      │  ── sessions (按分钟去重)               │
      │                      │─────────────────────▶│                 │
      │                      │                      │                 │
      │                      │◀─────────────────────│                 │
      │                      │                      │                 │
      │                      │  聚合 totals.visitors                  │
      │                      │                      │                 │
      │◀─────────────────────────────────────────────────────────────│
      │   { totals: { visitors: 156, ... } }                          │
      │   【 指标栏：浏览量 892 | 访客 156 | 事件 231 | 国家 12 】    │
```

---

## 九、核心配置常量汇总

| 常量名 | 值 | 含义 | 所属文件 |
|-------|-----|------|---------|
| **时间窗口** | | | |
| `ACTIVE_WINDOW_MINUTES` | 5 | 页面头部在线人数统计窗口（分钟） | `getActiveVisitors.ts` |
| `REALTIME_RANGE` | 30 | 实时页统计时间范围（分钟） | `lib/constants.ts:32` |
| **刷新频率** | | | |
| `refetchInterval` (ActiveUsers) | 60000 | 页面头部在线人数刷新间隔（毫秒） | `ActiveUsers.tsx:8` |
| `REALTIME_INTERVAL` | 10000 | 实时数据自动刷新间隔（毫秒） | `lib/constants.ts:33` |
| **Salt 轮换** | | | |
| `SALT_ROTATION` | 'month' | 默认 Salt 轮换周期（day/week/month） | `api/send/route.ts:143` |
| **其他** | | | |
| `VISIT_EXPIRE_TIME` | 1800 | Visit ID 过期时间（秒，30 分钟） | `api/send/route.ts:172` |

---

## 十、技术要点总结

### 10.1 访客识别策略

| 层级 | 优先级 | 识别方式 | 特点 |
|-----|-------|---------|------|
| L1 | 最高 | `distinct_id` | 用户自定义 ID，跨设备可追踪 |
| L2 | 默认 | `session_id = hash(websiteId + ip + ua + salt)` | 匿名识别，按月轮换 |

### 10.2 时间窗口设计哲学

- **5 分钟（页面头部）**：定义「真正在线」，反映当前实时人气
- **30 分钟（实时页）**：定义「最近活跃」，提供更完整趋势分析

### 10.3 性能优化策略

1. **双数据库路由**：关系型保证一致性，ClickHouse 保证大数据量查询性能
2. **前端轮询频率差异化**：关键指标（实时页）10 秒一刷，辅助指标（头部）60 秒一刷
3. **React Query 缓存**：避免重复请求，自动后台刷新

### 10.4 隐私保护设计

1. **无 Cookie 设计**：完全不依赖 Cookie，规避 GDPR 合规风险
2. **Salt 轮换机制**：定期重置匿名标识，防止长期追踪
3. **IP + UA 哈希**：单向散列，无法逆向还原原始信息

---

## 十一、代码索引

| 功能模块 | 文件路径 |
|---------|---------|
| Session 生成 | `src/app/api/send/route.ts` |
| UUID / Salt 算法 | `src/lib/crypto.ts` |
| 事件存储 | `src/queries/sql/events/saveEvent.ts` |
| 在线人数计算 | `src/queries/sql/getActiveVisitors.ts` |
| 实时数据聚合 | `src/queries/sql/getRealtimeData.ts` |
| 在线人数 API | `src/app/api/websites/[websiteId]/active/route.ts` |
| 实时数据 API | `src/app/api/realtime/[websiteId]/route.ts` |
| 头部在线组件 | `src/components/metrics/ActiveUsers.tsx` |
| 实时指标栏 | `src/app/(main)/websites/[websiteId]/realtime/RealtimeHeader.tsx` |
| 在线人数 Hook | `src/components/hooks/queries/useActiveUsersQuery.ts` |
| 实时数据 Hook | `src/components/hooks/queries/useRealtimeQuery.ts` |
| 常量定义 | `src/lib/constants.ts` |

---

**报告生成时间**: 2026-05-15
**分析版本**: Umami v2.x
**作者**: Umami 技术架构组
