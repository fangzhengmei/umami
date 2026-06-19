# Umami Tracker 多域名上报与 Session 合并机制分析

## 概述

Umami 的多域名上报与 Session 合并涉及五层关键机制：域名规范化、跨子域 Session 复用、无 Cookie 存储策略、多入口采集、以及统计层面的合并边界。本文档从源码层面梳理完整脉络，并纠正前版中对访问次数、页面浏览、录制入口、www 匹配正则、页面域名入库差异的不准确表述。

---

## 1. 域名规范化逻辑 (Domain Normalization)

### 1.1 www 匹配正则：前端与后端不一致（代码事实修正）

**前版错误**：未区分前端和后端 www 正则的差异。

**代码事实**：

| 位置 | 正则表达式 | 匹配范围 | 代码位置 |
|------|-----------|----------|----------|
| 前端采集（页面域名） | `/^www./` | `www` 后任意字符（含非点号，如 `wwwexample.com` 也会被匹配） | [send/route.ts#L184](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L184-L184) |
| 前端采集（来源域名） | `/^www\./` | `www.` 精确匹配（必须有点号） | [send/route.ts#L214](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L214-L214) |
| 后端统计查询（PostgreSQL） | `'^www.'` | PostgreSQL `regexp_replace` 的正则，含义同 JS `/^www./` | [prisma.ts#L134](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/prisma.ts#L134-L136) |

```typescript
// 页面域名：/^www./  —— 注意没有转义点号
const urlDomain = currentUrl.hostname.replace(/^www./, '');

// 来源域名：/^www\./  —— 注意转义了点号
referrerDomain = referrerUrl.hostname.replace(/^www\./, '');
```

**正则差异影响**：
- `www.example.com` → 两者都正确替换为 `example.com`
- `wwwexample.com`（无点号的边缘情况）→ `/^www./` 会错误地把 `wwwe` 替换掉 → `xample.com`；`/^www\./` 不会匹配 → 保留原样
- 实际生产中，合法域名不会出现 `wwwexample.com` 这种形式，因此差异通常不可感知

### 1.2 规范化应用场景与完整调用链

www 规范化出现在以下 **9 处**代码位置：

| 位置 | 用途 | 代码位置 |
|------|------|----------|
| 采集时 urlDomain | 页面域名规范化，作为 hostname 的 fallback | [send/route.ts#L184](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L184-L184) |
| 采集时 referrerDomain | 来源域名规范化入库 | [send/route.ts#L214](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L214-L214) |
| referrer 过滤器 | 筛选 referrer 时排除自引用 | [prisma.ts#L134](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/prisma.ts#L134-L136) |
| getPageviewMetrics | 来源域名维度统计时排除自引用 | [getPageviewMetrics.ts#L50](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/pageviews/getPageviewMetrics.ts#L50-L51) |
| getPageviewExpandedMetrics | 来源域名扩展维度排除自引用 | [getPageviewExpandedMetrics.ts#L54](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/pageviews/getPageviewExpandedMetrics.ts#L54-L55) |
| getValues | 获取可筛选值时排除自引用 | [getValues.ts#L26](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getValues.ts#L26-L27) |
| getChannelMetrics | 渠道统计中判定 referral 渠道 | [getChannelMetrics.ts#L65](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getChannelMetrics.ts#L65-L65) |
| getChannelExpandedMetrics | 扩展渠道统计判定 referral | [getChannelExpandedMetrics.ts#L96](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getChannelExpandedMetrics.ts#L96-L96) |
| getAttribution | 归因分析中排除自引用来源 | [getAttribution.ts#L118](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/reports/getAttribution.ts#L118-L118) |
| getRevenueMetrics | 收入统计中判定 referral 渠道 | [getRevenueMetrics.ts#L218](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/reports/getRevenueMetrics.ts#L218-L218) |

### 1.3 页面域名入库差异：hostname vs urlDomain（代码事实修正）

**前版错误**：hostname 存储"原始 hostname（带 www）"。

**代码事实**：hostname 的入库规则是 `hostname || urlDomain`（payload 优先，规范化后的值做兜底）。

```typescript
// send/route.ts#L234
await saveEvent({
  // ...
  hostname: hostname || urlDomain,   // payload.hostname 存在就用原始值；否则用去 www 后的 urlDomain
  referrerDomain,                     // 永远是去 www 后的值（/^www\./ 正则）
  // ...
});
```

**数据库存储实际差异**：

| 字段 | 数据来源 | www 处理 | 示例输入 | 存储结果 |
|------|----------|----------|----------|----------|
| `website_event.hostname` | `payload.hostname \|\| urlDomain` | 有 payload.hostname → 保留原样；无 → 去 www（`/^www./`） | `www.example.com` 有 payload → 存 `www.example.com` |
| `website_event.hostname` | 同上 | 无 payload.hostname → 去 www | `www.example.com` 无 payload → 存 `example.com` |
| `website_event.referrer_domain` | `referrerUrl.hostname.replace(/^www\./, '')` | 永远去 www（`/^www\./`） | `www.google.com` → 存 `google.com` |

**前端 tracker 的 hostname 来源**：

```javascript
// tracker/index.js#L14, #69
const { hostname, href, origin } = location;   // 来自 window.location.hostname，原始值，带 www

const getPayload = () => ({
  // ...
  hostname,   // 透传 window.location.hostname，不做任何规范化
  // ...
});
```

→ **默认情况下**，前端会透传带 www 的原始 hostname 给后端，因此后端走 `payload.hostname` 分支，`hostname` 字段存储带 www 的原始值。
→ **特殊情况**：如果前端不传递 hostname（如像素/链接追踪、或手动 API 调用），后端用 `urlDomain`（去 www 后的值）作为 fallback。

---

## 2. 跨子域复用机制 (Cross-Subdomain Session Reuse)

### SessionID 生成算法（关键！域名不参与计算）

**代码位置**：[send/route.ts#L147](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L147-L147)

```typescript
// 两种生成模式
const sessionId = id 
  ? uuid(sourceId, id)                              // 有自定义标识时（identify 调用过）
  : uuid(sourceId, ip, userAgent, sessionSalt);     // 无自定义标识时（默认）
```

**UUID 生成函数**：[crypto.ts#L60-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/crypto.ts#L60-L66)

```typescript
export function uuid(...args: any) {
  if (args.length) {
    // v5 确定性 UUID：相同输入 → 相同输出
    return v5(hash(...args, secret()), v5.DNS);
  }
  return process.env.USE_UUIDV7 ? v7() : v4();
}
```

### 跨子域复用原理

```
用户访问 app.example.com → 生成 sessionId = uuid(websiteId, ip, UA, salt)
用户访问 admin.example.com → 生成 sessionId = uuid(websiteId, ip, UA, salt)
                          → 两者相同！后端识别为同一 Session
```

**关键条件**：
- ✅ 同一 `sourceId`（同一个 website/link/pixel 配置）
- ✅ 同一 `ip`（同一网络出口）
- ✅ 同一 `userAgent`（同一浏览器）
- ✅ 同一 `sessionSalt` 周期（默认按月轮换）
- ❌ **域名/hostname 不参与计算** → 跨子域自动复用

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

- 默认：`SALT_ROTATION=month` → 每月生成新 salt → 新 sessionId
- 可配置：`day` / `week` / `month`
- salt 轮换会导致即使同一用户同一设备，跨周期也被识别为新 Visitor

---

## 3. Cookie 模式与存储策略

### Umami 不使用 Cookie！

**代码验证**：全局搜索 `document.cookie` 或 `Set-Cookie` 无任何结果。

### 存储架构

| 存储位置 | 用途 | 代码位置 |
|----------|------|----------|
| **前端内存变量 `cache`** | 存储 JWT token（含 sessionId、visitId、iat、websiteId） | [index.js#L400](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js#L400-L400) |
| **`x-umami-cache` Header** | 请求时传递 session 信息给后端 | [index.js#L181](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js#L181-L182) |
| **`localStorage`** | 仅用于检查 `umami.disabled` 禁用标志 | [index.js#L159](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js#L159-L159) |

**重要限制**：cache 变量仅存在于页面 JS 内存中 → 刷新页面、跨标签页、跨子域 **均不共享**。每次页面刷新后首次请求不带 cache，需要后端重新计算 sessionId（但由于确定性 UUID 算法，结果相同）。

### Cache Token 结构

**代码位置**：[send/route.ts#L17-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L17-L22)

```typescript
interface Cache {
  websiteId: string;   // 网站ID
  sessionId: string;   // 会话ID
  visitId: string;     // 访问ID（30分钟过期）
  iat: number;         // 签发时间戳（Unix 秒）
}
```

**Token 流转**：
1. 首次请求 → 无 cache header → 后端解析 JWT 失败 → 重新计算 sessionId、生成 visitId
2. 后端返回 JWT token：`createToken({ websiteId, sessionId, visitId, iat }, secret())`
3. 前端将 token 保存到内存变量 `cache`
4. 后续请求在 `x-umami-cache` header 中携带此 token
5. 后端解析 token，直接复用 sessionId、visitId（跳过重新计算）

### Visit 过期机制（访问次数 vs 访客数）

**代码位置**：[send/route.ts#L171-L175](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L171-L175)

```typescript
// Expire visit after 30 minutes
if (!timestamp && now - iat > 1800) {
  visitId = uuid(sessionId, visitSalt);  // 生成新 visitId（确定性：同 sessionId + visitSalt → 同 visitId）
  iat = now;                             // 重置签发时间
}
```

- `visitId` 每 **30 分钟**（1800 秒）过期
- `sessionId` 保持不变 → **Visitor 计数不变，Visits（访问次数）+1**
- 条件 `!timestamp` 表示：如果前端传入了自定义 timestamp（历史数据导入场景），不触发 visit 过期逻辑

### fetch-credentials 配置

**代码位置**：[index.js#L38](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js#L38-L38)

```javascript
const credentials = config('fetch-credentials') || 'omit';
```

- 仅控制 fetch 请求的跨域凭证模式（`omit` / `include` / `same-origin`）
- 不涉及任何 Session 存储逻辑，因为 Umami 不使用 Cookie

---

## 4. 采集入口 (Collector Entry Points)

### 4.1 前端 Tracker 入口

**代码位置**：[tracker/index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js)

**域名白名单过滤**：

```javascript
// 配置（第37行）
const domain = config('domains') || '';
const domains = domain.split(',').map(n => n.trim());

// 过滤逻辑（第156-161行）
const trackingDisabled = () =>
  disabled ||
  !website ||
  localStorage?.getItem('umami.disabled') ||
  (domain && !domains.includes(hostname)) ||  // 域名白名单检查：精确匹配 hostname
  (dnt && hasDoNotTrack());
```

⚠️ **注意**：`domains.includes(hostname)` 是精确匹配，不是通配符匹配。配置 `data-domains="example.com"` 时，`app.example.com` 不会被匹配。需要把所有要上报的子域名都列出来。

**自动追踪功能**：
- 页面浏览（PageView）：`track()` — DOM ready 后自动调用
- SPA 路由变化：hook `history.pushState` / `replaceState`
- 点击事件：事件委托监听 `document.click`，识别 `data-umami-event` 属性
- 性能指标：Web Vitals（LCP、INP、CLS、FCP、TTFB）

**手动 API**：
- `umami.track(eventName?, data?)` — 追踪事件或页面浏览
- `umami.identify(id, data?)` — 设置用户标识（distinct_id），会重置 cache
- `umami.getSession()` — 返回 `{ cache, website }`，供 recorder 等依赖方使用

### 4.2 前端 Recorder 入口（会话录制，代码事实补充）

**代码位置**：[recorder/index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/recorder/index.js)

**前版缺失**：Recorder 是独立于 Tracker 的另一个前端脚本，有独立加载逻辑和依赖关系。

**加载与依赖关系**：

```javascript
// recorder/index.js#L101-L110
const waitForSession = (attempts = 0) => {
  if (attempts > 50) return;    // 最多等待 5 秒 (50 × 100ms)
  const session = window.umami?.getSession?.();
  if (session?.cache) {
    beginRecording();           // 必须等到 tracker 返回 cache token 才开始录制
  } else {
    setTimeout(() => waitForSession(attempts + 1), 100);
  }
};
```

→ **Recorder 强依赖 Tracker**：必须等 tracker 初始化完成并拿到 `cache` JWT token 后才能开始录制。如果 tracker 被禁用（域名不在白名单、DNT 等），recorder 也不会工作（永远等不到 cache）。

**Recorder 配置项**：
- `data-website-id`：同 tracker
- `data-host-url`：同 tracker
- `data-sample-rate`：采样率，默认 `0.15`（15%），`1.0` 表示 100%
- `data-mask-level`：脱敏级别，`moderate`（默认，仅掩码输入框）或 `strict`（掩码所有文本）
- `data-max-duration`：最大录制时长，默认 `300000`ms（5 分钟）
- `data-block-selector`：额外屏蔽的 CSS 选择器

**刷新策略**：
- 事件缓冲满 100 条 → 立即上报
- 定时刷新：每 10 秒上报一次
- 页面隐藏（visibilitychange → hidden）→ keepalive 上报
- 页面卸载（beforeunload）→ keepalive 上报（body < 60KB 时）

### 4.3 后端 API 入口（6 个，补充 link/pixel 的 hostname 来源）

| 入口 | 方法 | 用途 | hostname 来源 | 代码位置 |
|------|------|------|--------------|----------|
| `/api/send` | `POST` | 主采集入口（event/identify/performance） | `payload.hostname \|\| urlDomain` | [send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts) |
| `/api/batch` | `POST` | 批量采集（内部循环调用 `/api/send`） | 同 `/api/send` | [batch/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/batch/route.ts) |
| `/api/record` | `POST` | 会话录制（**必须**带 cache token） | 不存 hostname，存到 session_replay 表关联 sessionId | [record/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/record/route.ts) |
| `/p/[slug]` | `GET` | 像素追踪（返回 1x1 GIF），内部构造 POST 调用 `/api/send` | **不传 hostname payload** → 走 urlDomain fallback（去 www） | [p/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/(collect)/p/[slug]/route.ts) |
| `/q/[slug]` | `GET` | 链接追踪（重定向 + 上报），内部构造 POST 调用 `/api/send` | **不传 hostname payload** → 走 urlDomain fallback（去 www） | [q/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/(collect)/q/[slug]/route.ts) |

**像素/链接追踪的 hostname 特殊处理**：

```typescript
// p/[slug]/route.ts — 构造内部转发请求时
const payload = {
  type: 'event',
  payload: {
    pixel: pixel.id,
    url: request.url,           // 请求像素图片的完整 URL
    referrer: request.headers.get("referer") || undefined,
    // ⚠️ 没有 hostname 字段！
  },
};
```

→ 像素和链接追踪不传 `hostname` payload → 后端走 `hostname || urlDomain` 中的 `urlDomain` 分支 → `new URL(url, base).hostname.replace(/^www./, '')` → hostname 字段永远是去 www 后的值。

### 4.4 `/api/record` 录制入口详细流程（代码事实补充）

**代码位置**：[record/route.ts#L27-L124](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/record/route.ts#L27-L124)

```
1. Zod schema 验证 → type 必须是 'record'
2. 读取 x-umami-cache header → 没有直接报错 "Missing session token"
3. 解析 JWT → 必须包含 sessionId 和 visitId
4. 查询 website 配置（绕过 Redis 缓存，避免 replayEnabled 变更未生效）
5. 校验 website.replayEnabled === true → false 直接返回 ok:false
6. CLOUD_MODE 下校验是否为 Business 订阅
7. Bot/IP 检查（同 /api/send）
8. 从 events 数组中提取最小/最大时间戳
9. 调用 saveRecording() → 存到 session_replay 表
   ├─ session_id, visit_id（来自 cache token）
   ├─ chunk_index（用于同一会话内分片排序）
   ├─ events（gzip 压缩后的 rrweb 事件数组）
   └─ started_at, ended_at
10. 返回 { ok: true }
```

**录制数据与 Session 的关联**：session_replay 表通过 `session_id` + `visit_id` 关联到 website_event，因此查询录制时可以拿到对应 session 的 browser/os/country 等元数据。

### 4.5 主采集入口 `/api/send` 处理流程

**代码位置**：[send/route.ts#L66-L322](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L66-L322)

```
1. 解析请求 → Zod schema 验证（必须且只能提供 website/link/pixel 中的一个）
2. Cache 检查 → 解析 x-umami-cache header（JWT），拿到 websiteId/sessionId/visitId/iat
3. 校验 websiteId（仅 website 类型查 Redis + DB）
4. 客户端信息 → IP、UA、设备、浏览器、OS、地理位置
5. Bot/IP 拦截 → isbot 检查 + IP 黑名单/CIDR
6. SessionID 生成 → uuid(sourceId, ip, UA, salt) 或 uuid(sourceId, distinctId)
7. 非 ClickHouse 模式下写入 session 表（INSERT ... ON CONFLICT DO NOTHING）
8. VisitID 处理 → 30分钟过期检查（timestamp 自定义模式跳过）
9. 事件存储 → saveEvent() / saveSessionData()（根据 type: event/identify/performance）
10. 域名规范化 → urlDomain/referrerDomain 去 www
11. hostname 入库 → payload.hostname || urlDomain
12. 返回 Cache Token（JWT）+ sessionId + visitId
```

---

## 5. 统计合并边界 (Session Merge Boundaries)

### 5.1 三个核心指标的准确统计逻辑（代码事实修正）

**前版错误**：将 pageviews 简单等同于 event_type=1 的计数。实际有两套不同的统计查询路径，过滤条件不同。

#### 指标定义

| 指标名 | 英文名 | 统计逻辑 |
|--------|--------|----------|
| 访客数 | Visitors | `count(distinct session_id)` |
| 访问次数 | Visits | `count(distinct visit_id)` |
| 页面浏览 | Pageviews | 两套逻辑，见下方详细说明 |
| 事件数 | Events | `count(* where event_type=2)` 或 `sum(case event_type=2 then 1 else 0 end)` |

#### 页面浏览（Pageviews）的两套统计路径

**路径 A：Dashboard 主面板 — getWebsiteStats（排除自定义事件和性能事件）**

[getWebsiteStats.ts#L40-L61](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getWebsiteStats.ts#L40-L61)

```sql
-- Prisma/PostgreSQL 版本
select
  sum(t.c) as "pageviews",                 -- 内层 count(*) 汇总
  count(distinct t.session_id) as "visitors",
  count(distinct t.visit_id) as "visits",
  ...
from (
  select session_id, visit_id, count(*) as "c"
  from website_event
  where website_id = {{websiteId}}
    and created_at between {{startDate}} and {{endDate}}
    and event_type NOT IN (2, 5)          -- ⚠️ 排除 event_type=2（自定义事件）和 5（性能事件）
  group by session_id, visit_id            -- 先按 visit 分组，支持弹回率计算
) as t
```

→ **主面板 Pageviews = pageView(1) + linkEvent(3) + pixelEvent(4)**（排除了自定义事件和性能事件）

**路径 B：Sessions 面板 — getWebsiteSessionStats（不过滤 event_type）**

[getWebsiteSessionStats.ts#L37-L54](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/sessions/getWebsiteSessionStats.ts#L37-L54)

```sql
-- Prisma/PostgreSQL 版本
select
  count(*) as "pageviews",                                    -- ⚠️ 不过滤 event_type，所有事件都算
  count(distinct website_event.session_id) as "visitors",
  count(distinct website_event.visit_id) as "visits",
  count(distinct session.country) as "countries",
  sum(case when website_event.event_type = 2 then 1 else 0 end) as "events"  -- 单独统计自定义事件数
from website_event
join session on ...
where website_event.website_id = {{websiteId}}
  and website_event.created_at between {{startDate}} and {{endDate}}
```

→ **Sessions 面板 Pageviews = 所有 event_type 行数之和**（包含自定义事件和性能事件），然后用 events 列单独展示 `event_type=2` 的数量

**ClickHouse 模式下的差异**：
- 使用原始表 `website_event`：同 Prisma 逻辑
- 使用聚合表 `website_event_stats_hourly`：已预先按 (website_id, session_id, visit_id, event_type, hostname, ...) 维度聚合了 `views` 字段，直接 sum(views)

#### 访问次数（Visits）的准确含义

```sql
count(distinct visit_id) as "visits"
```

visit_id 的生成规则：
- `visitId = uuid(sessionId, visitSalt)`，其中 `visitSalt = hash(startOfHour(createdAt).toUTCString())`
- 是**确定性 UUID**：同一 session 在同一小时内 → 同一 visitId
- 但有 **30 分钟过期机制**：如果 `now - iat > 1800`，即使同小时也生成新 visitId
- 因此 Visits 既受小时边界影响，也受 30 分钟空闲超时影响

#### 弹回率（Bounce Rate）

```sql
-- 内层按 visit 分组 count(*) = 1 的就是弹回
sum(case when t.c = 1 then 1 else 0 end) as "bounces"
```

→ 弹回 = 某 visit 内只有 1 条非自定义/非性能事件

### 5.2 合并判定核心：SessionID 唯一性

所有统计查询都基于 `session_id` 进行去重计数：

```sql
-- 典型统计查询模式
count(distinct website_event.session_id) as "visitors"
```

**代码证据**（63+ 处查询）：
- [getWebsiteSessionStats.ts#L40](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/sessions/getWebsiteSessionStats.ts#L40-L40)
- [getWebsiteStats.ts#L44](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getWebsiteStats.ts#L44-L44)
- [getPageviewMetrics.ts#L76](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/pageviews/getPageviewMetrics.ts#L76-L76)
- [getAttribution.ts#L55](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/reports/getAttribution.ts#L55-L55)

### 5.3 合并条件（同一 Session）

✅ **合并** → 相同 `sessionId`

| 条件 | 要求 |
|------|------|
| sourceId (websiteId/linkId/pixelId) | 必须相同 |
| ip | 必须相同（同一网络出口） |
| userAgent | 必须相同（同一浏览器/设备） |
| sessionSalt | 必须相同（同一 salt 周期） |
| hostname/domain | **不要求**（跨子域自动合并） |

### 5.4 分割条件（不同 Session）

❌ **不合并** → 不同 `sessionId`

| 场景 | 原因 | 影响 |
|------|------|------|
| 不同 sourceId | 即使同域名，不同 website/link/pixel 也分开 | Visitors ×N |
| IP 变化 | 切换 Wi-Fi/移动网络 | Visitors ×2 |
| UA 变化 | 切换浏览器/设备 | Visitors ×2 |
| Salt 周期变化 | 跨周期（默认跨月） | Visitors ×2 |
| 调用 identify(newId) | distinctId 变化 → 新 uuid(sourceId, newId) | 新 sessionId |

### 5.5 Hostname 在统计中的作用

`hostname` 字段**不影响** session 合并（它根本不参与 sessionId 计算），仅用于：

1. **维度拆分**：按 hostname 统计各子域名流量
   ```sql
   select hostname, count(distinct session_id) 
   from website_event 
   group by hostname
   ```

2. **自引用排除**：判断来源是否为站内跳转
   ```sql
   and referrer_domain != regexp_replace(hostname, '^www.', '')
   and referrer_domain != ''
   ```
   [prisma.ts#L134](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/prisma.ts#L134-L136)

3. **渠道识别**：判定 referral 渠道
   ```sql
   when referrer_domain != regexp_replace(hostname, '^www.', '') 
     and referrer_domain != '' then 'referral'
   ```
   [getChannelMetrics.ts#L65](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getChannelMetrics.ts#L65-L65)

⚠️ **注意**：由于 hostname 入库时可能是原始值（带 www）或规范化值（不带 www），而 referrer_domain 永远是规范化值（不带 www），所以查询时必须用 `regexp_replace(hostname, '^www.', '')` 把 hostname 也做规范化，才能和 referrer_domain 正确比较。

### 5.6 多域名合并示例（修正 hostname 存储值）

假设配置 `websiteId = xxx`，用户使用 Chrome 浏览器，IP 不变，同一 salt 月内：

| 访问序列 | Hostname(原始) | payload.hostname | 入库 hostname 字段 | sessionId | Visitor | Visit |
|----------|---------------|------------------|-------------------|-----------|---------|-------|
| 1. `www.example.com`（tracker） | `www.example.com` | ✅ 传了 | 存 `www.example.com` | sid-123 | +1 | +1 |
| 2. `app.example.com`（tracker） | `app.example.com` | ✅ 传了 | 存 `app.example.com` | sid-123 | +0（合并） | +0（<30min） |
| 3. 像素追踪 `/p/xxx` | 像素 URL 域名 | ❌ 没传 | 存去 www 后的值 | sid-123 | +0（合并） | +0（<30min） |
| 4. 45 分钟后 `admin.example.com` | `admin.example.com` | ✅ 传了 | 存 `admin.example.com` | sid-123 | +0（合并） | +1（超时） |
| 5. 次月 `www.example.com` | `www.example.com` | ✅ 传了 | 存 `www.example.com` | sid-456（新salt） | +1 | +1 |

---

## 6. 完整数据流转图

```
浏览器（app.example.com）
    ↓
[Tracker] index.js
    ├─ 读取 data-* 属性（website-id, domains, fetch-credentials 等）
    ├─ 检查域名白名单：domains.includes(hostname)  ← 精确匹配 hostname
    ├─ 构造 payload：hostname = window.location.hostname（原始值，可能带 www）
    ├─ 内存读取 cache（JWT token）← 刷新/跨子域后为空
    └─ POST /api/send
        ├─ Header: x-umami-cache: <JWT or undefined>
        └─ Body: { type, payload: { website, hostname, url, referrer, ... } }
            ↓
[Backend] send/route.ts
    ├─ 1. 解析 JWT cache → { websiteId, sessionId, visitId, iat } 或 null
    ├─ 2. 生成 sessionSalt（按月） + visitSalt（按小时）
    ├─ 3. 计算 sessionId = uuid(sourceId, ip, UA, salt)
    │      （域名/hostname 完全不参与！）
    ├─ 4. 检查 visit 过期（now - iat > 1800s）→ 生成新 visitId
    ├─ 5. URL 解析 + 域名规范化
    │     ├─ urlDomain = hostname.replace(/^www./, '')     ← 注意无转义点
    │     └─ referrerDomain = referrerHostname.replace(/^www\./, '') ← 有转义点
    ├─ 6. 入库
    │     ├─ session 表：INSERT ON CONFLICT DO NOTHING（ClickHouse 模式跳过）
    │     └─ website_event 表
    │           ├─ session_id: sessionId         ← 合并判定依据
    │           ├─ visit_id: visitId             ← 访问次数判定依据
    │           ├─ hostname: payload.hostname || urlDomain
    │           │   （tracker 传值 → 原始值；像素/链接 → 去 www）
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
            ├─ 解析 cache → sessionId, visitId（不能重新生成！）
            ├─ 校验 replayEnabled
            └─ 存 session_replay 表（关联 session_id + visit_id）

[Statistics] 查询时
    ├─ Visitors = count(distinct session_id)        ← 跨子域合并
    ├─ Visits   = count(distinct visit_id)          ← 30min+ 过期 + 小时边界
    ├─ Pageviews = 两套逻辑（主面板排除 event_type 2/5；Sessions 面板全量）
    ├─ 按 hostname group by → 拆分各子域名数据
    └─ 渠道判定：referrer_domain != regexp_replace(hostname, '^www.', '') → referral
```

---

## 7. 关键配置项

| 配置项 | 位置 | 默认值 | 作用 |
|--------|------|--------|------|
| `data-domains` | Tracker script 属性 | 空 | 域名白名单，逗号分隔，**精确匹配** hostname |
| `data-fetch-credentials` | Tracker script 属性 | `omit` | fetch 跨域凭证模式（omit/include/same-origin） |
| `data-sample-rate` | Recorder script 属性 | `0.15` | 录制采样率（0.0 ~ 1.0） |
| `data-mask-level` | Recorder script 属性 | `moderate` | 录制脱敏级别（moderate/strict） |
| `data-max-duration` | Recorder script 属性 | `300000` | 单次录制最大时长（毫秒） |
| `SALT_ROTATION` | 环境变量 | `month` | Session salt 轮换周期（day/week/month） |
| `REMOVE_TRAILING_SLASH` | 环境变量 | `false` | 是否移除 URL 尾部斜杠 |
| `DISABLE_BOT_CHECK` | 环境变量 | `false` | 是否禁用 Bot 检测 |
| `CLOUD_MODE` | 环境变量 | `false` | 是否启用云模式（影响录制权限校验等） |

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
| SessionID 生成 | [src/lib/crypto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/crypto.ts) |
| JWT Cache Token | [src/lib/jwt.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/jwt.ts) |
| 统计查询过滤器（含 www 规范化） | [src/lib/prisma.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/prisma.ts) |
| 事件存储 | [src/queries/sql/events/saveEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/events/saveEvent.ts) |
| 录制存储 | [src/queries/sql/replays/saveRecording.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/replays/saveRecording.ts) |
| Dashboard 统计（含 pageviews 过滤逻辑） | [src/queries/sql/getWebsiteStats.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/getWebsiteStats.ts) |
| Sessions 统计 | [src/queries/sql/sessions/](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/sessions/) |
| 数据库 Schema | [prisma/schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/prisma/schema.prisma) |
| 常量定义 | [src/lib/constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/constants.ts) |

---

## 9. 常见疑问解答

**Q: 为什么跨子域访问会被合并为同一个 Visitor？**
A: 因为 `sessionId` 基于 `(sourceId, ip, userAgent, sessionSalt)` 生成，**域名/hostname 完全不参与计算**。同一浏览器访问同一 websiteId 下的不同子域名，会生成相同的 sessionId。

**Q: 如何让不同子域名分开统计？**
A: 为每个子域名创建独立的 website 配置（不同 websiteId），这样 sessionId 生成的第一个因子就不同了。

**Q: Umami 支持第三方 Cookie 吗？**
A: 不支持。Umami 完全不使用 Cookie，session 信息通过 JWT token 在 `x-umami-cache` 请求头中传递，客户端仅在 JS 内存中保存（刷新即丢失，但后端可重算）。

**Q: 为什么配置了 `data-domains="example.com"` 但 `app.example.com` 没有数据？**
A: `data-domains` 是 **精确匹配** `window.location.hostname`，不做通配。需要写成 `data-domains="example.com,app.example.com,admin.example.com"`。

**Q: Session 会在什么时候分割？**
A: 五种情况：sourceId 变化、IP 变化、UA 变化、salt 周期轮换（默认跨月）、调用 `umami.identify(newId)` 传入新的 distinctId。

**Q: 为什么同一个 referrer 在 hostname 是 `www.example.com` 和 `example.com` 时判定不同？**
A: 查询时用 `regexp_replace(hostname, '^www.', '')` 规范化 hostname 后再和 referrer_domain 比较。但如果前端没传 hostname payload（像素/链接追踪），入库时 hostname 已经是 urlDomain（去 www 后的值），所以不会有问题。

**Q: Pageviews 在 Dashboard 和 Sessions 面板数字对不上？**
A: 正常。Dashboard 用 `getWebsiteStats`，排除了 `event_type IN (2, 5)`（自定义事件 + 性能事件）；Sessions 面板用 `getWebsiteSessionStats`，count(*) 包含所有 event_type，然后用独立的 events 列展示自定义事件数。

**Q: Recorder 为什么有时候不录制？**
A: 检查：(1) `replayEnabled` 是否开启；(2) tracker 是否正常初始化并返回 cache（域名白名单、DNT 等）；(3) 采样率 `data-sample-rate` 默认仅 15%；(4) Cloud 模式下需要 Business 订阅。
