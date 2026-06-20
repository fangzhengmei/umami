# Umami Tracker 多域名上报与 Session 合并机制分析

## 概述

Umami 的多域名上报与 Session 合并涉及五层关键机制：域名规范化、跨子域 Session 复用、无 Cookie 存储策略、多入口采集、以及统计层面的合并边界。本文档从源码层面梳理完整脉络，重点澄清访问次数（visit）的 30 分钟过期与小时盐值的关系、www 规范化的准确位置数量、以及访问标识复用与新增的完整触发条件。

---

## 1. 域名规范化逻辑 (Domain Normalization)

### 1.1 www 匹配正则：前端与后端不一致（代码事实）

**两处正则不一样**：

| 位置 | 正则表达式 | 匹配范围 | 代码位置 |
|------|-----------|----------|----------|
| 页面域名 urlDomain | `/^www./` | `www` 后任意字符（含非点号，边缘情况 `wwwexample.com` 会被错误替换） | [send/route.ts#L184](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L184-L184) |
| 来源域名 referrerDomain | `/^www\./` | `www.` 精确匹配（必须有点号） | [send/route.ts#L214](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L214-L214) |
| 统计查询（PostgreSQL） | `'^www.'` | PostgreSQL `regexp_replace` 用，含义同 JS `/^www./` | [prisma.ts#L134](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/prisma.ts#L134-L136) |

```typescript
// 页面域名：注意没有转义点号
const urlDomain = currentUrl.hostname.replace(/^www./, '');

// 来源域名：注意转义了点号
referrerDomain = referrerUrl.hostname.replace(/^www\./, '');
```

**实际影响**：合法域名不会出现 `wwwexample.com` 这种形式，因此差异通常不可感知。

### 1.2 规范化应用场景与完整调用链（Prisma 10 处 + ClickHouse 0 处）

经源码逐处核对，www 规范化在 **Prisma/PostgreSQL 模式** 下有 **10 处**；**ClickHouse 模式 完全不做** www 规范化。以下是两层（框架层 + 查询层）的对称差异对照：

#### 框架层差异（`getFilterQuery` 函数）

| 框架 | referrer 过滤器逻辑 | 代码位置 |
|------|-------------------|----------|
| Prisma | `referrer_domain != regexp_replace(hostname, '^www.', '')` | [prisma.ts#L132-L136](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/prisma.ts#L132-L136) |
| ClickHouse | `referrer_domain != hostname`（无 www 规范化） | [clickhouse.ts#L125-L127](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/clickhouse.ts#L125-L127) |

#### 查询层 7 个函数的对称差异（Prisma vs ClickHouse 对照）

| # | 查询函数 | Prisma 规范化位置 | ClickHouse 缺规范化位置 |
|---|---------|-----------------|---------------------|
| 1 | getPageviewMetrics | [getPageviewMetrics.ts#L50](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/pageviews/getPageviewMetrics.ts#L50) `regexp_replace(hostname, '^www.', '')` | [getPageviewMetrics.ts#L118](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/pageviews/getPageviewMetrics.ts#L118) `referrer_domain != hostname` |
| 2 | getPageviewExpandedMetrics | [getPageviewExpandedMetrics.ts#L54](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/pageviews/getPageviewExpandedMetrics.ts#L54) `regexp_replace(...)` | [getPageviewExpandedMetrics.ts#L136](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/pageviews/getPageviewExpandedMetrics.ts#L136) `referrer_domain != hostname` |
| 3 | getValues | [getValues.ts#L26](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getValues.ts#L26) `regexp_replace(...)` | [getValues.ts#L83](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getValues.ts#L83) `referrer_domain != hostname` |
| 4 | getChannelMetrics | [getChannelMetrics.ts#L65](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getChannelMetrics.ts#L65) `regexp_replace(...)` | [getChannelMetrics.ts#L120](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getChannelMetrics.ts#L120) `referrer_domain != hostname` |
| 5 | getChannelExpandedMetrics | [getChannelExpandedMetrics.ts#L96](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getChannelExpandedMetrics.ts#L96) `regexp_replace(...)` | [getChannelExpandedMetrics.ts#L167](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getChannelExpandedMetrics.ts#L167) `referrer_domain != hostname` |
| 6 | getAttribution | [getAttribution.ts#L118](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/reports/getAttribution.ts#L118) `regexp_replace(we.hostname, ...)` | [getAttribution.ts#L335](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/reports/getAttribution.ts#L335) `we.referrer_domain != hostname` |
| 7 | getRevenueMetrics | [getRevenueMetrics.ts#L218](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/reports/getRevenueMetrics.ts#L218) `regexp_replace(...)` | [getRevenueMetrics.ts#L420](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/reports/getRevenueMetrics.ts#L420) `referrer_domain != hostname` |

#### 采集层 2 处（两种模式共享，入库前处理）

| # | 位置 | 正则 | 代码位置 |
|---|------|------|----------|
| 1 | 采集时 urlDomain | `/^www./`（无转义点号） | [send/route.ts#L184](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L184) |
| 2 | 采集时 referrerDomain | `/^www\./`（有转义点号） | [send/route.ts#L214](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L214) |

#### Prisma 模式 10 处汇总

```
采集层 2 处（入库前规范化）
  ├─ urlDomain     = hostname.replace(/^www./, '')
  └─ referrerDomain = referrerHostname.replace(/^www\./, '')

框架层 1 处（过滤器公共函数）
  └─ prisma.ts getFilterQuery: referrer_domain != regexp_replace(hostname, '^www.', '')

查询层 7 处（各统计查询内联）
  ├─ getPageviewMetrics
  ├─ getPageviewExpandedMetrics
  ├─ getValues
  ├─ getChannelMetrics
  ├─ getChannelExpandedMetrics
  ├─ getAttribution
  └─ getRevenueMetrics
```

⚠️ **ClickHouse 模式统计差异示例**：

| 场景 | hostname | referrer_domain | Prisma 判定 | ClickHouse 判定 |
|------|----------|-----------------|------------|----------------|
| 站内跳转（带 www → 不带 www） | `www.example.com` | `example.com` | 自引用（排除） | **误判为外部 referral** |
| 站内跳转（不带 → 不带） | `example.com` | `example.com` | 自引用（排除） | 自引用（排除） |
| 真正外部来源 | `example.com` | `google.com` | referral | referral |

### 1.3 页面域名入库差异：hostname vs urlDomain

hostname 入库规则是 **`hostname || urlDomain`**（payload 优先，规范化后的值做兜底）：

```typescript
// send/route.ts#L234
await saveEvent({
  // ...
  hostname: hostname || urlDomain,   // payload.hostname 存在就用原始值；否则用去 www 后的 urlDomain
  referrerDomain,                     // 永远是去 www 后的值（/^www\./ 正则）
  // ...
});
```

**字段存储对比**：

| 字段 | 数据来源 | www 处理 | 常见值示例 |
|------|----------|----------|-----------|
| `website_event.hostname` | `payload.hostname \|\| urlDomain` | 有 payload → 原样；无 payload → 去 www（`/^www./`） | tracker 上报 → `www.example.com`；像素追踪 → `example.com` |
| `website_event.referrer_domain` | `referrerUrl.hostname.replace(/^www\./, '')` | 永远去 www（`/^www\./`） | `google.com`、`example.com` |

**前端 tracker 的 hostname 来源**：

```javascript
// tracker/index.js — 透传 window.location.hostname，不做任何规范化
const { hostname } = location;
```

→ 默认情况下 tracker 透传原始 hostname（带 www），hostname 字段存原始值；像素/链接追踪不传 hostname payload，走 urlDomain fallback，存去 www 后的值。

---

## 2. 跨子域复用机制 (Cross-Subdomain Session Reuse)

### SessionID 生成算法（域名不参与计算）

**代码位置**：[send/route.ts#L147](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L147-L147)

```typescript
const sessionId = id 
  ? uuid(sourceId, id)                              // 有自定义标识（identify 调用过）
  : uuid(sourceId, ip, userAgent, sessionSalt);     // 默认模式
```

**UUID 生成**：[crypto.ts#L60-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/crypto.ts#L60-L66)

```typescript
export function uuid(...args: any) {
  if (args.length) {
    return v5(hash(...args, secret()), v5.DNS);  // v5 确定性 UUID：同入 → 同出
  }
  return process.env.USE_UUIDV7 ? v7() : v4();
}
```

### 跨子域复用原理

```
访问 app.example.com   → sessionId = uuid(websiteId, ip, UA, salt)
访问 admin.example.com → sessionId = uuid(websiteId, ip, UA, salt)
                        → 两者相同！后端识别为同一 Visitor
```

**合并条件**（全部满足才合并）：
- 同一 `sourceId`（website/link/pixel 配置）
- 同一 `ip`
- 同一 `userAgent`
- 同一 `sessionSalt` 周期
- ❌ **hostname 不参与** → 跨子域自动合并

### Salt 轮换策略

**代码位置**：[crypto.ts#L72-L78](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/crypto.ts#L72-L78)

```typescript
export function getSalt(saltRotation: string, createdAt: Date): string {
  return hash(
    (saltRotation === 'day' ? startOfDay : saltRotation === 'week' ? startOfWeek : startOfMonth)(
      createdAt,
    ).toUTCString(),
  );
}
```

- 默认 `SALT_ROTATION=month` → 每月新 salt → 新 sessionId
- 可配置 `day` / `week` / `month`
- salt 轮换周期决定 Visitor 的"自然分割"周期

---

## 3. Cookie 模式与存储策略

### Umami 完全不使用 Cookie

全局搜索 `document.cookie` 或 `Set-Cookie` 无任何结果。

### 存储架构

| 存储位置 | 用途 | 代码位置 |
|----------|------|----------|
| 前端内存变量 `cache` | 存 JWT token（sessionId、visitId、iat、websiteId） | [tracker/index.js#L400](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js#L400-L400) |
| `x-umami-cache` Header | 请求时传递给后端 | [tracker/index.js#L181](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js#L181-L182) |
| `localStorage` | 仅检查 `umami.disabled` 禁用标志 | [tracker/index.js#L159](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js#L159-L159) |

**重要限制**：cache 仅存在于页面 JS 内存 → 刷新页面、跨标签页、跨子域 **均不共享**。但因 sessionId 是确定性 UUID，刷新后重新计算结果相同。

### Cache Token 结构

```typescript
interface Cache {
  websiteId: string;   // 网站ID
  sessionId: string;   // 会话ID
  visitId: string;     // 访问ID
  iat: number;         // 签发时间戳（Unix 秒）
}
```

**Token 流转**：
1. 首次请求 → 无 cache → 后端计算 sessionId、生成 visitId
2. 后端返回 JWT token
3. 前端内存保存 cache 变量
4. 后续请求在 `x-umami-cache` header 中携带
5. 后端解析复用 sessionId、visitId（跳过计算）

### Visit 生成与过期机制（访问次数核心逻辑）

**代码位置**：[send/route.ts#L145-L175](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L145-L175)

#### visitSalt：按小时的确定性盐

```typescript
const visitSalt = hash(startOfHour(createdAt).toUTCString());
```

- 基于 **事件时间 `createdAt` 的小时起点** 生成（不是当前服务器时间 `now`）
- 正常模式下 `createdAt ≈ now`，每小时变化一次
- `visitId = uuid(sessionId, visitSalt)` 是**确定性 UUID**：同 sessionId + 同 visitSalt → 同一 visitId

#### 完整生成逻辑

```typescript
// 1. 初始化：优先用 cache 中的 visitId，没有则用当前小时 visitSalt 生成
let visitId = cache?.visitId || uuid(sessionId, visitSalt);
let iat = cache?.iat || now;

// 2. 30 分钟过期检查（仅正常模式，自定义 timestamp 跳过）
if (!timestamp && now - iat > 1800) {
  visitId = uuid(sessionId, visitSalt);  // 用当前小时 visitSalt 重新生成
  iat = now;                              // 重置签发时间
}
```

#### 关键机制：30 分钟超时与小时盐的交互

两者不是"并列两个过期条件"，而是**两层控制**：

| 场景 | cache 中的 visitId 来自 | now - iat | visitSalt 是否变化 | 最终 visitId | 访问次数 |
|------|-----------------------|-----------|-------------------|-------------|---------|
| 同小时内，间隔 < 30min | 10:00 的 salt | 15 min | 否 | **复用 cache** | 不变 |
| 同小时内，间隔 > 30min | 10:00 的 salt | 35 min | 否 | **重新生成但同结果**（确定性 UUID + 同 salt） | 不变 |
| 跨小时，间隔 < 30min | 10:00 的 salt | 15 min | 是（11:00） | **复用 cache**（保留旧 hour 的 visitId） | 不变 |
| 跨小时，间隔 > 30min | 10:00 的 salt | 45 min | 是（11:00） | **重新生成新结果** | +1 |
| 刷新页面 / 新标签页（无 cache） | 无 | — | 当前小时 | `uuid(sid, visitSalt)` | 同小时不变，跨小时 +1 |

#### iat 的刷新机制

`iat` 是 visitId 的"签发时间"，**不是最后活跃时间**，只有在 30 分钟超时时才会重置：

- 10:20 首次 → `iat = 10:20`, `visitId = uuid(sid, salt_10)`
- 10:35 → `iat=10:20`, 差 15min < 30 → **不重置**
- 10:50:01 → `iat=10:20`, 差 30min1s > 30 → **重置 iat=10:50:01**，visitId 仍 = `uuid(sid, salt_10)`（同小时不变）
- 11:15 → `iat=10:50:01`, 差 24min59s < 30 → **不重置**，继续复用 cache 中 10 点的 visitId

#### 结论

- **30 分钟超时** 是"续期"机制：只要持续活跃（每次重置周期内都有请求），visitId 可以**跨小时延续**，不被小时边界切断
- **小时盐** 是"底座"：没有 cache 或 cache 过期时，visitId 由 sessionId + 小时盐决定，天然按小时分段
- `!timestamp` 条件：自定义 timestamp 模式（历史数据导入）不触发 30 分钟过期逻辑，完全由小时盐决定
- `sessionId` 始终不变 → **Visitor 计数不变，仅 Visits 变化**

### fetch-credentials 配置

仅控制 fetch 跨域凭证模式（`omit` / `include` / `same-origin`），不涉及任何 Session 存储。

---

## 4. 采集入口 (Collector Entry Points)

### 4.1 前端 Tracker 入口

**代码位置**：[tracker/index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js)

**域名白名单过滤**（精确匹配 hostname，非通配）：

```javascript
const domain = config('domains') || '';
const domains = domain.split(',').map(n => n.trim());

const trackingDisabled = () =>
  disabled ||
  !website ||
  localStorage?.getItem('umami.disabled') ||
  (domain && !domains.includes(hostname)) ||  // 精确匹配
  (dnt && hasDoNotTrack());
```

**自动追踪**：
- 页面浏览（PageView）：DOM ready 后自动 `track()`
- SPA 路由变化：hook `history.pushState` / `replaceState`
- 点击事件：事件委托监听 `document.click`，识别 `data-umami-event`
- 性能指标：Web Vitals（LCP、INP、CLS、FCP、TTFB）

**手动 API**：
- `umami.track(eventName?, data?)` — 追踪事件或页面浏览
- `umami.identify(id, data?)` — 设置用户标识，会重置 cache
- `umami.getSession()` — 返回 `{ cache, website }`，供 recorder 等依赖方使用

### 4.2 前端 Recorder 入口（会话录制）

**代码位置**：[recorder/index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/recorder/index.js)

**强依赖 Tracker**：必须等 tracker 初始化完成并拿到 cache token 后才能开始录制。

```javascript
const waitForSession = (attempts = 0) => {
  if (attempts > 50) return;    // 最多等待 5 秒 (50 × 100ms)
  const session = window.umami?.getSession?.();
  if (session?.cache) {
    beginRecording();
  } else {
    setTimeout(() => waitForSession(attempts + 1), 100);
  }
};
```

→ 如果 tracker 被禁用（域名不在白名单、DNT 等），recorder 永远等不到 cache，不会录制。

**Recorder 配置**：
- `data-sample-rate`：采样率，默认 `0.15`（15%）
- `data-mask-level`：脱敏级别，`moderate`（默认）或 `strict`
- `data-max-duration`：最大录制时长，默认 `300000`ms（5 分钟）
- `data-block-selector`：额外屏蔽的 CSS 选择器

**刷新策略**：缓冲满 100 条 / 每 10 秒 / 页面隐藏 / 页面卸载（keepalive）

### 4.3 后端 API 入口汇总

| 入口 | 方法 | 用途 | hostname 来源 | 代码位置 |
|------|------|------|--------------|----------|
| `/api/send` | `POST` | 主采集入口（event/identify/performance） | `payload.hostname \|\| urlDomain` | [send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts) |
| `/api/batch` | `POST` | 批量采集（内部循环调用 `/api/send`） | 同 `/api/send` | [batch/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/batch/route.ts) |
| `/api/record` | `POST` | 会话录制（必须带 cache token） | 不存 hostname，存 session_replay 表关联 sessionId | [record/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/record/route.ts) |
| `/p/[slug]` | `GET` | 像素追踪（返回 1x1 GIF），内部构造 POST | **不传 hostname** → urlDomain fallback（去 www） | [p/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/(collect)/p/[slug]/route.ts) |
| `/q/[slug]` | `GET` | 链接追踪（重定向 + 上报），内部构造 POST | **不传 hostname** → urlDomain fallback（去 www） | [q/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/(collect)/q/[slug]/route.ts) |

### 4.4 `/api/record` 录制入口流程

```
1. Zod schema 验证 → type 必须是 'record'
2. 读取 x-umami-cache header → 没有直接报错 "Missing session token"
3. 解析 JWT → 必须包含 sessionId 和 visitId（不能重新生成）
4. 查询 website 配置（绕过 Redis 缓存，确保 replayEnabled 准确）
5. 校验 website.replayEnabled === true
6. CLOUD_MODE 下校验 Business 订阅
7. Bot/IP 检查
8. 从 events 数组提取最小/最大时间戳
9. saveRecording() → 存 session_replay 表（gzip 压缩 rrweb 事件）
10. 返回 { ok: true }
```

### 4.5 `/api/send` 主采集入口流程

```
1. Zod schema 验证（且只能提供 website/link/pixel 中的一个）
2. Cache 检查 → 解析 x-umami-cache header（JWT）
3. 校验 sourceId（website 类型查 Redis + DB）
4. 客户端信息 → IP、UA、设备、浏览器、OS、地理位置
5. Bot/IP 拦截
6. SessionID 生成 → uuid(sourceId, ip, UA, salt) 或 uuid(sourceId, distinctId)
7. 非 ClickHouse 模式下写 session 表（ON CONFLICT DO NOTHING）
8. VisitID 处理 → cache 优先 / 30 分钟过期重置
9. 事件存储 → saveEvent() / saveSessionData()
10. 域名规范化 → urlDomain / referrerDomain 去 www
11. hostname 入库 → payload.hostname || urlDomain
12. 返回 Cache Token（JWT）+ sessionId + visitId
```

---

## 5. 统计合并边界 (Session Merge Boundaries)

### 5.1 三个核心指标的准确统计逻辑

#### 指标定义

| 指标名 | 英文名 | 统计逻辑 |
|--------|--------|----------|
| 访客数 | Visitors | `count(distinct session_id)` |
| 访问次数 | Visits | `count(distinct visit_id)` |
| 页面浏览 | Pageviews | 两套逻辑，见下方 |
| 事件数 | Events | `sum(event_type = 2)` |

#### 页面浏览（Pageviews）的两套统计路径

**路径 A：Dashboard 主面板 — getWebsiteStats（排除自定义事件和性能事件）**

```sql
select
  sum(t.c) as "pageviews",
  count(distinct t.session_id) as "visitors",
  count(distinct t.visit_id) as "visits",
  ...
from (
  select session_id, visit_id, count(*) as "c"
  from website_event
  where website_id = {{websiteId}}
    and created_at between {{startDate}} and {{endDate}}
    and event_type NOT IN (2, 5)   -- 排除自定义事件(2)和性能事件(5)
  group by session_id, visit_id      -- 先按 visit 分组，支持弹回率计算
) as t
```

→ **主面板 Pageviews = pageView(1) + linkEvent(3) + pixelEvent(4)**

**路径 B：Sessions 面板 — getWebsiteSessionStats（不过滤 event_type）**

```sql
select
  count(*) as "pageviews",                -- 不过滤，所有事件都算
  count(distinct website_event.session_id) as "visitors",
  count(distinct website_event.visit_id) as "visits",
  sum(case when event_type = 2 then 1 else 0 end) as "events"  -- 单独统计自定义事件
from website_event
...
```

→ **Sessions 面板 Pageviews = 所有 event_type 行数之和**，然后用 events 列单独展示 `event_type=2`

**ClickHouse 模式差异**：
- 原始表 `website_event`：同 Prisma 逻辑
- 聚合表 `website_event_stats_hourly`：预先按维度聚合了 `views` 字段，直接 sum

### 5.2 访问标识复用 vs 新增访问的完整触发条件

**✅ 复用（同一 visitId）— 访问次数不变**：

| 条件 | 说明 |
|------|------|
| 同一 sessionId + 同一小时 + 无 cache | 确定性 UUID 生成相同 visitId |
| 有 cache 且 `now - iat ≤ 1800s` | 直接复用 cache 中的 visitId（可跨小时） |
| 同一 sessionId + 同一小时 + cache 已过期 | 重新生成但 visitSalt 没变 → 结果相同 |

**❌ 新增访问（不同 visitId）— 访问次数 +1**：

| 触发场景 | 原因 |
|----------|------|
| 不同 sessionId | 不同访客，天然不同 visit |
| 跨小时 + 无 cache（刷新、新标签页、像素/链接） | 新小时 visitSalt 变化 → 新 visitId |
| 跨小时 + cache 已过期（>30 分钟空闲） | 超时后用新小时 visitSalt 重新生成 → 新 visitId |
| 调用 `umami.identify(newId)` | sessionId 变化 → visitId 随之重生成 |
| 切换子域名且页面刷新 + 跨小时 | cache 丢失 + 新小时 visitSalt → 新 visitId |

⚠️ **关键结论**：
- 持续活跃（每次 iat 重置周期内都有请求）→ visitId 可**跨小时延续**，不被小时边界切断
- 无 cache 场景 → visitId 严格**按小时分段**
- 同小时内即使 cache 过期重置，visitId 也不变（visitSalt 没变）
- 域名变化本身不影响 visitId，影响的只是 hostname 字段存储值

### 5.3 合并判定核心：SessionID 唯一性

所有统计查询都基于 `session_id` 去重计数。全局 60+ 处 `count(distinct session_id)` 或 `uniq(session_id)` 查询。

**合并条件**（同一 Visitor）：
- 同一 sourceId
- 同一 ip
- 同一 userAgent
- 同一 sessionSalt 周期
- hostname 不要求（跨子域自动合并）

**分割条件**（不同 Visitor）：
- 不同 sourceId
- IP 变化
- UA 变化
- salt 周期变化
- 调用 identify(newId)

### 5.4 Hostname 在统计中的作用

`hostname` 字段**不影响** session 合并（不参与 sessionId 计算），仅用于：

1. **维度拆分**：按 hostname 统计各子域名流量
2. **自引用排除**：判断来源是否为站内跳转（需先做 www 规范化比较）
3. **渠道识别**：判定 referral 渠道

⚠️ 因为 hostname 入库可能带 www 或不带（取决于是否传 payload），而 referrer_domain 永远不带 www，所以查询时必须用 `regexp_replace(hostname, '^www.', '')` 规范化后才能正确比较。

### 5.5 多域名合并示例

假设 `websiteId = xxx`，Chrome 浏览器，IP 不变，同一 salt 月，从 10:20 开始：

| # | 访问场景 | Hostname | 时间 | cache | sessionId | Visitor | Visit | 原因 |
|---|----------|----------|------|-------|-----------|---------|-------|------|
| 1 | `www.example.com`（tracker） | `www.example.com` | 10:20 | 无 | sid-123 | +1 | +1 | 首次访问 |
| 2 | `app.example.com`（SPA 跳转） | `app.example.com` | 10:35 | 有 | sid-123 | +0 | +0 | 有 cache + 间隔 15min < 30min |
| 3 | 像素追踪 `/p/xxx` | 像素域名 | 10:40 | 无 | sid-123 | +0 | +0 | 无 cache，但同小时 visitSalt → 同 visitId |
| 4 | `admin.example.com`（新标签页） | `admin.example.com` | 11:10 | 无 | sid-123 | +0 | +1 | 跨小时 + 无 cache → 新 visitSalt |
| 5 | `app.example.com`（原标签页） | `app.example.com` | 11:15 | 有 | sid-123 | +0 | +0 | 有 cache + iat 距上次重置 25min < 30min |

**更多场景**：

| # | 场景 | 时间 | visitId 变化 | 访问次数 |
|---|------|------|-------------|---------|
| 6 | 持续活跃（每 10 分钟一次） | 10:20→11:50 | 始终同一个（10 点盐生成的） | +0 |
| 7 | 离开 35 分钟后回来（同小时） | 10:20 走，10:55 回 | 同小时 salt → 同 visitId | +0 |
| 8 | 离开 45 分钟后回来（跨小时） | 10:20 走，11:05 回 | 新小时 salt → 新 visitId | +1 |
| 9 | 跨子域名 + 页面刷新 + 跨小时 | — | 无 cache + 跨小时 → 新 visitId | +1 |
| 10 | 跨子域名 + SPA 跳转（不刷新） | — | 有 cache → 同 visitId | +0 |

---

## 6. 完整数据流转图

```
浏览器（app.example.com）
    ↓
[Tracker] index.js
    ├─ 读取 data-* 属性（website-id, domains 等）
    ├─ 检查域名白名单：domains.includes(hostname) ← 精确匹配
    ├─ 构造 payload：hostname = window.location.hostname（原始值，可能带 www）
    ├─ 内存读取 cache（JWT token）← 刷新/跨子域/新标签页为空
    └─ POST /api/send
        ├─ Header: x-umami-cache: <JWT or undefined>
        └─ Body: { type, payload: { website, hostname, url, referrer, ... } }
            ↓
[Backend] send/route.ts
    ├─ 1. 解析 JWT cache → { websiteId, sessionId, visitId, iat } 或 null
    ├─ 2. 生成 sessionSalt（按月） + visitSalt（按小时，基于 createdAt）
    ├─ 3. 计算 sessionId = uuid(sourceId, ip, UA, salt)
    │      （域名/hostname 完全不参与！）
    ├─ 4. visitId 处理
    │     ├─ 有 cache 且 now-iat ≤ 1800s → 复用 cache visitId（可跨小时）
    │     ├─ 无 cache 或超时 → uuid(sessionId, visitSalt) ← 确定性生成
    │     └─ 注意：iat 不是滑动窗口，只有超时时才重置
    ├─ 5. URL 解析 + 域名规范化
    │     ├─ urlDomain = hostname.replace(/^www./, '')   ← 无转义点
    │     └─ referrerDomain = referrerHostname.replace(/^www\./, '') ← 转义点
    ├─ 6. 入库
    │     ├─ session 表：INSERT ON CONFLICT DO NOTHING（ClickHouse 跳过）
    │     └─ website_event 表
    │           ├─ session_id: sessionId        ← Visitor 合并依据
    │           ├─ visit_id: visitId            ← 访问次数依据
    │           ├─ hostname: payload.hostname || urlDomain
    │           │   （tracker → 原始值；像素/链接 → 去 www）
    │           └─ referrer_domain: 永远去 www 后的值
    └─ 7. 返回新 JWT cache → 前端内存保存
        ↓
[Recorder] recorder/index.js（可选，独立加载）
    ├─ 等待 window.umami.getSession() 返回 cache（最多 5 秒）
    ├─ rrweb 录制事件，缓冲 100 条或 10 秒
    └─ POST /api/record
        ├─ Header: x-umami-cache: <必须有 JWT>
        └─ Body: { type: 'record', payload: { website, events } }
            ↓
            ├─ 解析 cache → sessionId, visitId（不能重新生成）
            ├─ 校验 replayEnabled
            └─ 存 session_replay 表（关联 session_id + visit_id）

[Statistics] 查询时
    ├─ Visitors = count(distinct session_id)    ← 跨子域合并
    ├─ Visits   = count(distinct visit_id)      ← 小时盐 + 30min 续期两层控制
    ├─ Pageviews = 两套逻辑（Dashboard 排除 2/5；Sessions 全量）
    ├─ 按 hostname group by → 拆分各子域名数据
    └─ Prisma: referrer_domain != regexp_replace(hostname, '^www.', '') → referral
       ClickHouse: referrer_domain != hostname  → 无 www 规范化，有差异
```

---

## 7. 关键配置项

| 配置项 | 位置 | 默认值 | 作用 |
|--------|------|--------|------|
| `data-domains` | Tracker script 属性 | 空 | 域名白名单，逗号分隔，**精确匹配** hostname |
| `data-fetch-credentials` | Tracker script 属性 | `omit` | fetch 跨域凭证模式 |
| `data-sample-rate` | Recorder script 属性 | `0.15` | 录制采样率 |
| `data-mask-level` | Recorder script 属性 | `moderate` | 录制脱敏级别 |
| `data-max-duration` | Recorder script 属性 | `300000` | 单次录制最大时长（毫秒） |
| `SALT_ROTATION` | 环境变量 | `month` | Session salt 轮换周期（day/week/month） |
| `REMOVE_TRAILING_SLASH` | 环境变量 | `false` | 是否移除 URL 尾部斜杠 |
| `DISABLE_BOT_CHECK` | 环境变量 | `false` | 是否禁用 Bot 检测 |
| `CLOUD_MODE` | 环境变量 | `false` | 是否启用云模式 |
| `CLICKHOUSE_URL` | 环境变量 | 空 | 启用 ClickHouse 模式（www 规范化行为不同） |

---

## 8. 核心文件速查表

| 功能模块 | 文件路径 |
|----------|----------|
| 前端 Tracker | [src/tracker/index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js) |
| 前端 Recorder | [src/recorder/index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/recorder/index.js) |
| 主采集入口 | [src/app/api/send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts) |
| 批量采集入口 | [src/app/api/batch/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/batch/route.ts) |
| 录制入口 | [src/app/api/record/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/record/route.ts) |
| 像素追踪入口 | [src/app/(collect)/p/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/(collect)/p/[slug]/route.ts) |
| 链接追踪入口 | [src/app/(collect)/q/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/(collect)/q/[slug]/route.ts) |
| SessionID / visitSalt 生成 | [src/lib/crypto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/crypto.ts) |
| JWT Cache Token | [src/lib/jwt.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/jwt.ts) |
| Prisma 查询框架（含 www 规范化） | [src/lib/prisma.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/prisma.ts) |
| ClickHouse 查询框架（无 www 规范化） | [src/lib/clickhouse.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/clickhouse.ts) |
| 事件存储 | [src/queries/sql/events/saveEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/events/saveEvent.ts) |
| 录制存储 | [src/queries/sql/replays/saveRecording.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/replays/saveRecording.ts) |
| Dashboard 统计 | [src/queries/sql/getWebsiteStats.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getWebsiteStats.ts) |
| Sessions 统计 | [src/queries/sql/sessions/](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/sessions/) |
| 数据库 Schema | [prisma/schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/prisma/schema.prisma) |
| 常量定义 | [src/lib/constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/constants.ts) |

---

## 9. 常见疑问解答

**Q: 为什么跨子域访问会被合并为同一个 Visitor？**
A: 因为 `sessionId` 基于 `(sourceId, ip, userAgent, sessionSalt)` 生成，**hostname 完全不参与**。同一浏览器访问同一 websiteId 下的不同子域名，生成相同的 sessionId。

**Q: 如何让不同子域名分开统计？**
A: 为每个子域名创建独立的 website 配置（不同 websiteId），sessionId 生成的第一个因子就不同了。

**Q: Umami 支持第三方 Cookie 吗？**
A: 不支持。Umami 完全不使用 Cookie，session 信息通过 JWT token 在 `x-umami-cache` 请求头中传递，客户端仅在 JS 内存中保存。

**Q: 为什么 `data-domains="example.com"` 但 `app.example.com` 没有数据？**
A: `data-domains` 是 **精确匹配** `window.location.hostname`，不做通配。需要写成 `data-domains="example.com,app.example.com,admin.example.com"`。

**Q: Session / Visit 什么时候分割？**
A: Session 分割（Visitor +1）：sourceId 变、IP 变、UA 变、salt 周期轮换、调用 identify(newId)。
Visit 分割（访问次数 +1）：跨小时且（无 cache 或 cache 已过期）、sessionId 变化。

**Q: 30 分钟过期和小时盐是什么关系？**
A: 两层控制。小时盐是"底座"——无 cache 时 visitId 按小时分段；30 分钟超时是"续期"机制——只要持续活跃，visitId 可以跨小时延续，不被小时边界切断。同小时内即使超时重置，visitId 也不变（因为 visitSalt 没变，确定性 UUID）。

**Q: 为什么同一个 referrer 在 hostname 带 www 和不带 www 时判定不同？**
A: Prisma 模式下查询会用 `regexp_replace(hostname, '^www.', '')` 规范化后比较，没问题；但 **ClickHouse 模式下**直接比较 `referrer_domain != hostname`，如果 hostname 带 www 而 referrer_domain 不带，会误判为外部来源。

**Q: Pageviews 在 Dashboard 和 Sessions 面板数字对不上？**
A: 正常。Dashboard 用 `getWebsiteStats`，排除了 `event_type IN (2, 5)`（自定义事件 + 性能事件）；Sessions 面板用 `getWebsiteSessionStats`，count(*) 包含所有 event_type。

**Q: Recorder 为什么有时候不录制？**
A: 检查：(1) `replayEnabled` 是否开启；(2) tracker 是否正常初始化并返回 cache（域名白名单、DNT 等）；(3) 采样率 `data-sample-rate` 默认仅 15%；(4) Cloud 模式下需要 Business 订阅。

**Q: hostname 字段到底带不带 www？**
A: 看数据来源。tracker 上报 → 透传 `window.location.hostname`（通常带 www）；像素/链接追踪 → 走 urlDomain fallback（去 www 后的值）。referrer_domain 永远是去 www 后的值。
