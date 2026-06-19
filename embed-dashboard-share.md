# Umami 嵌入组件与公开 Dashboard 实现边界分析

Umami 有三条完全独立的产品线，它们的 URL 空间、认证方式、CSP 头和缓存策略互不重叠：

| 产品线 | URL 入口 | 用途 | 认证 |
|--------|---------|------|------|
| 采集脚本 | `/script.js` + `/api/send` | 在被测网站中运行，上报访问数据 | 无（`skipAuth`） |
| 公开 Dashboard（Share 页面） | `/share/:slug` | 浏览器直接访问，只读展示统计 | Share Token (JWT) |
| iframe 嵌入 | `/share/:slug` 嵌入到宿主页面 | 与 Share 页面完全同源，只是交付方式不同 | 同 Share 页面 |

容易混淆的地方在于：**采集脚本从不出现在 Share 页面中**。采集脚本跑在被测网站上，向 `/api/send` 写数据；Share 页面跑在查看者的浏览器里，通过 `/api/websites/:id/stats` 等只读 API 读数据。二者的 URL、HTTP 头、认证机制完全隔离。

---

## 1. 三条产品线的 URL 与代码边界

### 1.1 采集脚本：`/script.js` → `/api/send`

**Tracker 脚本**由 Next.js 的 `rewrites` 配置从 `/script.js` 映射到 rollup 构建产物。脚本在**被测网站的页面**中运行，通过 `POST /api/send` 上报事件。

`src/app/api/send/route.ts` 是采集端点，核心特征：

- `parseRequest(request, schema, { skipAuth: true })` — **跳过认证**，任何来源均可上报
- 接收 payload 中 `website` / `pixel` / `link` 三选一作为数据归属 ID
- 通过 `fetchWebsite()` 查找 Website 实体（走 Redis 缓存，TTL 86400 秒）
- 返回一个 `x-umami-cache` Token 用于后续请求的会话复用

`next.config.ts` 中为 Tracker 脚本配置了独立的响应头（仅生产环境）：

```
source: TRACKER_SCRIPT   // 即 "/script.js"
headers:
  - Access-Control-Allow-Origin: *         // 允许任何域加载此脚本
  - Cache-Control: public, max-age=86400, must-revalidate  // 浏览器缓存 24h
```

此外还有**自定义脚本名**机制（环境变量 `TRACKER_SCRIPT_NAME`），可以为脚本起别名，如 `/analytics.js`，走相同的 trackerHeaders。

**Pixel 和 Link 的采集端点**也属于采集线，但走完全不同的 URL：

- `src/app/(collect)/p/[slug]/route.ts` — Pixel 追踪，返回 1×1 GIF，内部转发到 `POST /api/send`
- `src/app/(collect)/q/[slug]/route.ts` — Link 跳转追踪，302 重定向到目标 URL，内部转发到 `POST /api/send`

这两个端点都标记了 `export const dynamic = 'force-dynamic'`，Pixel 的响应头为 `Cache-Control: no-cache, no-store, must-revalidate`，不缓存。

### 1.2 Share 页面：`/share/:slug`

**路由结构**：

```
src/app/share/
├── ShareProvider.tsx                    ← Context Provider，签发 Token
└── [slug]/
    ├── layout.tsx                       ← 挂载 ShareProvider
    └── [[...path]]/                     ← 可选子路径（events / sessions 等）
        ├── page.tsx                     → SharePage
        ├── SharePage.tsx                ← 页面分发 + 布局
        ├── ShareNav.tsx                 ← 侧边导航（按 parameters 过滤）
        ├── ShareBranding.tsx            ← Logo / 白标签
        └── ShareFooter.tsx              ← 底部（仅白标签时显示）
```

**加载时序**：

```
1. 浏览器访问 /share/abc123def456
2. layout.tsx 拿到 slug，包裹 <ShareProvider slug={slug}>
3. ShareProvider 调用 useShareTokenQuery(slug)
   → GET /api/share/abc123def456
   → 后端签发 JWT，返回 { shareId, shareType, parameters, token, websiteId?, boardId?, ... }
4. setShareData(data, { token }) 写入 Zustand store（src/store/app.ts）
5. ShareContext.Provider 向子组件提供完整 share 数据
6. SharePage 根据 shareType 分发到对应页面组件
7. 子组件内所有 API 调用经过 useApi()，自动附加：
     x-umami-share-token: <jwt>
     x-umami-share-context: 1
```

**Share 页面与主应用的组件复用**：SharePage 直接 import 主应用 `src/app/(main)/` 下的页面组件，如 `WebsitePage`、`EventsPage`、`BoardViewPage` 等。区别仅在于 props：

- `WebsiteHeader showActions={false} allowLink={false}` — 隐藏设置/编辑入口
- `BoardViewPage showActions={false}` — 隐藏编辑/删除/设计按钮
- `PixelPage` / `LinkPage` `showHeaderActions={false}` — 隐藏管理操作

**Share 页面不加载采集脚本**：Share 页面是一个纯读取界面，不需要也不能往 `/api/send` 写数据，因此没有任何引用 `/script.js` 的代码。

### 1.3 iframe 嵌入：`/share/:slug` 的另一种交付

iframe 嵌入和直接访问 Share 页面**在服务端没有任何区别**。区别全部在宿主侧：

```html
<iframe src="https://umami.example.com/share/abc123def456"
        width="100%" height="800" frameborder="0">
</iframe>
```

- 浏览器请求的 URL 完全相同
- 服务端渲染的 HTML 完全相同
- 前端 React 组件树完全相同
- 唯一区别是 CSP 的 `frame-ancestors` 决定了此页面能否被嵌入（见第 2 节）
- 另一个区别是视口尺寸由 iframe 容器决定，而非浏览器窗口（见第 4 节）

---

## 2. CSP：三条线的头策略如何隔离

全部定义在 `next.config.ts` 中，按路径匹配分配不同的响应头组。

### 2.1 默认头（匹配 `/:path*`，覆盖所有页面包括 `/share/*`）

```
Content-Security-Policy:
  default-src 'self';
  img-src 'self' https: data:;
  script-src 'self' 'unsafe-eval' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  connect-src 'self' https:;
  frame-ancestors 'self' ${frameAncestors};
```

其中 `frameAncestors` 来自环境变量 `ALLOWED_FRAME_URLS`（`next.config.ts:19`）。

**`frame-ancestors` 决定了 iframe 嵌入能否生效**：

- 不设 `ALLOWED_FRAME_URLS` → `frame-ancestors 'self'` → 仅同源可嵌入，外部域名 iframe 会被浏览器拦截
- 设 `ALLOWED_FRAME_URLS="https://blog.example.com"` → `frame-ancestors 'self' https://blog.example.com` → 该域名可嵌入
- 直接浏览器访问 `/share/:slug` 不受 `frame-ancestors` 影响

**`connect-src 'self' https:` 的意义**：Share 页面中所有 API 请求（`/api/websites/:id/stats` 等）都走同源，满足 `'self'`。`https:` 则是为了白标签场景下加载远程 Logo 图片等资源。

**`script-src` 含 `unsafe-eval` 和 `unsafe-inline`**：这是 Chart.js 等图表库的需要，与采集脚本无关——采集脚本根本不在 Share 页面中加载。

### 2.2 API 头（匹配 `/api/:path*`）

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: *
Access-Control-Allow-Methods: GET, DELETE, POST, PUT
Access-Control-Max-Age: 86400
Cache-Control: no-cache
```

`CORS: *` 和 `Cache-Control: no-cache` 对 Share 场景的影响：

- Share 页面通过浏览器同源请求 API，CORS 实际上不起作用（浏览器同源不触发预检）
- `no-cache` 保证每次 API 请求都到达服务器，统计数据保持新鲜
- 如果 Share 页面通过 iframe 跨域嵌入，浏览器的同源策略仍然放行（iframe 内的页面与 API 同源），CORS 同样不需要

### 2.3 Tracker 脚本头（匹配 `/script.js`，仅生产环境）

```
Access-Control-Allow-Origin: *
Cache-Control: public, max-age=86400, must-revalidate
```

这与 Share 页面完全无关。Tracker 脚本跑在**被测网站**的域名下，需要 CORS `*` 才能向 Umami 后端发送数据。24h 缓存是为了减少浏览器对脚本本身的重复下载。

### 2.4 头策略总结

| 路径 | CSP frame-ancestors | CORS | Cache-Control | 服务对象 |
|------|---------------------|------|---------------|---------|
| `/:path*`（含 `/share/*`） | `'self' + ALLOWED_FRAME_URLS` | 无 | 无（Next.js 默认） | Share 页面 / 主应用 |
| `/api/:path*` | 无 | `*` | `no-cache` | 所有 API 消费者 |
| `/script.js` | 无 | `*` | `public, max-age=86400` | 被测网站中的 Tracker |
| `/p/:slug`（Pixel） | 无 | 无 | `no-cache, no-store` | Pixel 追踪 |
| `/q/:slug`（Link） | 无 | 无 | 无 | Link 跳转 |

---

## 3. 权限裁剪：从 Token 到 UI 的四层收缩

### 3.1 数据模型

`prisma/schema.prisma` 中 Share 模型：

```prisma
model Share {
  id         String    @id @map("share_id") @db.Uuid
  entityId   String    @map("entity_id") @db.Uuid
  name       String    @db.VarChar(200)
  shareType  Int       @map("share_type") @db.Integer
  slug       String    @unique @db.VarChar(100)
  parameters Json
  createdAt  DateTime?
  updatedAt  DateTime?
}
```

- `slug`：16 字符随机串（`getRandomChars(16)`），也可用户自定义，构成公开 URL
- `shareType`：`ENTITY_TYPE.website=1, link=2, pixel=3, board=4`
- `parameters`：JSON，控制 Website 类型分享时可访问的页面

### 3.2 Token 签发（Layer 0：Token Payload 决定可达实体）

`src/app/api/share/[slug]/route.ts` 的 `GET` 处理函数签发 Token：

1. `getShareByCode(slug)` 查数据库
2. 按 `shareType` 加载实体，组装 payload：
   - Website → `{ websiteId }`
   - Board → `{ boardId, websiteIds[], pixelIds[], linkIds[] }`
   - Pixel → `{ websiteId, pixelId }`
   - Link → `{ websiteId, linkId }`
3. `createToken(data, secret())` 签 JWT（`src/lib/jwt.ts`，标准 `jsonwebtoken` 库）
4. 可选附加 `whiteLabel`（从 Redis `white-label:${accountId}` 读取）

**Token 中不包含任何用户身份信息**，只有实体 ID 列表 + parameters。这意味着拿到 Token 的人只能访问 Token 中列出的实体数据。

### 3.3 实体级权限（Layer 1：后端 `canView*` 校验）

每个实体类型有独立的权限函数，优先级：`isAdmin > shareToken 匹配 > 用户所有权`。

**Website** — `src/permissions/website.ts` `canViewWebsite()`：

```typescript
if (user?.isAdmin) return true;
if (shareToken?.websiteId === websiteId ||
    shareToken?.pixelId === websiteId ||
    shareToken?.linkId === websiteId ||
    shareToken?.websiteIds?.includes(websiteId) ||
    shareToken?.pixelIds?.includes(websiteId) ||
    shareToken?.linkIds?.includes(websiteId)) return true;
// ... 用户所有权检查 ...
```

注意 `shareToken?.pixelId === websiteId` 这样的交叉匹配：Pixel/Link 与 Website 共享同一个 `websiteId` 字段（`src/app/api/share/[slug]/route.ts:83-84` 中 `data.websiteId = share.entityId`），因此通过 Pixel/Link 的 Share Token 也能查看对应 Website 的数据。

**Board** — `src/permissions/board.ts` `canViewBoard()`：

```typescript
if (shareToken?.boardId === boardId) return true;
```

**Link** — `src/permissions/link.ts` `canViewLink()`：

```typescript
if (shareToken?.linkId === linkId ||
    shareToken?.websiteId === linkId ||
    shareToken?.linkIds?.includes(linkId)) return true;
```

**Pixel** — `src/permissions/pixel.ts` `canViewPixel()`：

```typescript
if (shareToken?.pixelId === pixelId ||
    shareToken?.websiteId === pixelId ||
    shareToken?.pixelIds?.includes(pixelId)) return true;
```

**写操作（`canUpdate*` / `canDelete*`）一律不检查 shareToken**，只有登录用户才能修改。

### 3.4 上下文头校验（Layer 1.5：防 CSRF）

`src/lib/auth.ts` `checkAuth()` 中的关键逻辑：

```typescript
const shareToken = await parseShareToken(request);  // 从 x-umami-share-token 头解析 JWT

if (!user?.id && shareToken) {
  const shareContext = request.headers.get(SHARE_CONTEXT_HEADER);  // x-umami-share-context
  if (!shareContext) {
    return null;  // 拒绝：Share Token 脱离上下文使用
  }
}
```

前端 `src/components/hooks/useApi.ts` 只在 `pathname?.startsWith('/share')` 时才注入两个头：

```typescript
const isSharePath = pathname?.startsWith('/share');
const shareHeaders = isSharePath && shareToken?.token
  ? { [SHARE_TOKEN_HEADER]: shareToken.token, [SHARE_CONTEXT_HEADER]: '1' }
  : {};
```

这确保了：攻击者即使拿到 Share Token 的 JWT 字符串，也无法从其他网站（非 `/share/*` 路径）直接调用 API，因为缺少 `x-umami-share-context` 头。

### 3.5 页面级权限（Layer 2：`parameters` 过滤可见页面）

仅对 Website 类型分享生效。`src/app/share/ShareProvider.tsx`：

```typescript
const ALL_SECTION_IDS = [
  'overview', 'events', 'sessions', 'realtime', 'performance',
  'compare', 'breakdown', 'goals', 'funnels', 'journeys',
  'retention', 'utm', 'revenue', 'attribution'
];

const allowedSections =
  isWebsiteShare && share?.parameters
    ? ALL_SECTION_IDS.filter(id => share.parameters[id] === true)
    : [];
```

此过滤在三个地方生效：

1. **ShareNav**（`src/app/share/[slug]/[[...path]]/ShareNav.tsx:122-127`）：导航菜单只显示 `parameters[id] === true` 的条目
2. **SharePage**（`src/app/share/[slug]/[[...path]]/SharePage.tsx:87-88`）：访问未授权页面时强制跳回 `/share/:slug`
3. **ShareProvider**（`src/app/share/ShareProvider.tsx:66-76`）：若仅允许一个非 overview 页面，自动重定向到该页面

### 3.6 UI 操作级裁剪（Layer 3：Props 隐藏管理按钮）

| 组件 | Prop | 效果 |
|------|------|------|
| `WebsiteHeader` | `showActions={false}` | 隐藏设置/编辑按钮 |
| `WebsiteHeader` | `allowLink={false}` | 禁止点击跳转到主应用 |
| `BoardViewPage` | `showActions={false}` | 隐藏编辑/删除/设计 |
| `PixelPage` | `showHeaderActions={false}` | 隐藏管理操作 |
| `LinkPage` | `showHeaderActions={false}` | 隐藏管理操作 |

### 3.7 白标签（可选的品牌替换）

`src/app/share/[slug]/[[...path]]/ShareBranding.tsx` 根据 `share.whiteLabel` 替换 Logo：

- `whiteLabel.logoUrl` → 自定义图片
- `whiteLabel.displayName` → 自定义名称
- `whiteLabel.domainName` → 自定义链接

`ShareFooter.tsx` 仅在存在 `whiteLabel` 时渲染底部品牌条。

### 3.8 权限裁剪与 CSP、缓存的关系

```
请求进入
   │
   ├─ CSP frame-ancestors ──→ 是否允许 iframe 嵌入？（交付层）
   │                         不通过则浏览器直接拦截，后续全部不执行
   │
   ├─ checkAuth() ──→ 解析 Share Token + 上下文头（认证层）
   │                   不通过则 API 返回 401
   │
   ├─ canView*() ──→ Token 中的实体 ID 是否匹配？（实体层）
   │                  不通过则 API 返回 401
   │
   ├─ parameters 过滤 ──→ 可访问哪些页面？（页面层）
   │                       不通过则前端跳回首页
   │
   ├─ showActions ──→ 隐藏管理操作（UI 层）
   │
   └─ 缓存策略 ──→ 上述每一层查到的数据如何缓存？
                     实体数据走 Redis 24h，统计数据走 React Query 60s，
                     API 响应走 HTTP no-cache
```

CSP 是最外层门禁——如果 `frame-ancestors` 不放行，iframe 中的 Share 页面根本不会渲染，Token 签发和权限检查都不会发生。对于直接访问 `/share/:slug` 的场景，`frame-ancestors` 不起作用，权限链从 `checkAuth()` 开始。

---

## 4. 尺寸适配

### 4.1 Share 页面的两种布局模式

**Website 类型**：带侧边导航的 Grid 布局（`src/app/share/[slug]/[[...path]]/SharePage.tsx:121`）：

```typescript
<Grid columns={{ base: '1fr', lg: `${navCollapsed ? '60px' : '240px'} 1fr` }} width="100%">
```

| 视口 | 导航 | 列结构 |
|------|------|--------|
| < 1024px（base） | 顶部汉堡菜单 | 单列 `1fr` |
| ≥ 1024px（lg），展开 | 左侧 240px 固定侧边栏 | `240px 1fr` |
| ≥ 1024px（lg），收起 | 左侧 60px 图标栏 | `60px 1fr` |

**Board / Pixel / Link 类型**：极简单列布局（`SharePage.tsx:105-111`）：

```typescript
<Column>
  {entityPage}
  <ShareFooter />
</Column>
```

无侧边导航，宽度完全由容器决定。

### 4.2 ShareNav 尺寸

`src/app/share/[slug]/[[...path]]/ShareNav.tsx:136-143`：

```typescript
<Column
  position={isMobile ? undefined : 'fixed'}
  width={isMobile ? '100%' : collapsed ? '60px' : '240px'}
  maxHeight="100dvh"
  height="100dvh"
>
```

`100dvh`（dynamic viewport height）兼容移动浏览器地址栏的收缩与展开。

导航折叠状态持久化到 `localStorage`（键 `share:navCollapsed`）。

### 4.3 iframe 嵌入的尺寸约束

- **无 postMessage 通信**：Umami 未实现 iframe ↔ 父窗口高度同步。宿主页面必须手动设置 `height` 或使用 `scrolling="auto"`
- **最小宽度**：图表组件（Chart.js）在 < 320px 宽度下可能渲染异常
- **最小高度**：完整 Dashboard（含图表 + 数据表格）建议 ≥ 600px
- **URL 参数控制主题**：`SharePage.tsx:78-84` 支持 `?theme=light` / `?theme=dark`

### 4.4 尺寸适配与 CSP、权限的关系

在 iframe 嵌入场景中，尺寸适配**受 CSP 制约**：只有 `frame-ancestors` 放行后，iframe 才会加载，视口尺寸才有意义。在直接访问场景中，视口就是浏览器窗口，尺寸适配不受 CSP 影响。

权限裁剪与尺寸适配**互相独立**：无论视口多大，`parameters` 过滤和 `showActions` 都照常生效。但在小视口下，侧边导航自动切换为汉堡菜单，用户体验因权限可见页面数不同而异——如果只分享了一个页面，汉堡菜单里只有一个选项，此时可以考虑直接用单列布局的 Board 类型分享来简化 UI。

---

## 5. 缓存策略

### 5.1 三层缓存与各产品线的关系

```
┌──────────────────────────────────────────────────────────────────────┐
│  HTTP Cache-Control（next.config.ts）                                │
│  ├─ /api/*      → no-cache          ← Share 页面读数据走这          │
│  ├─ /script.js  → 24h must-revalidate ← 采集脚本走这              │
│  └─ /p/*        → no-store           ← Pixel 采集走这              │
├──────────────────────────────────────────────────────────────────────┤
│  React Query（src/app/Providers.tsx, staleTime=60s）                │
│  └─ Share 页面 + 主应用共用同一个 QueryClient                      │
│     Share Token 查询 key: ['share', slug]                           │
│     统计数据查询 key: 各 API 路径 + 参数                            │
├──────────────────────────────────────────────────────────────────────┤
│  Redis（src/lib/redis.ts, 可选，REDIS_URL 启用）                    │
│  ├─ white-label:*  → 无 TTL        ← Share 白标签                  │
│  ├─ website:*      → 86400s        ← fetchWebsite()               │
│  ├─ session:*      → 86400s        ← fetchSession()               │
│  ├─ pixel:*        → 86400s        ← Pixel 采集时查找实体          │
│  ├─ link:*         → 86400s        ← Link 跳转时查找实体           │
│  └─ auth:*         → 可配置        ← 用户登录态                    │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 各数据类型的缓存行为

| 数据 | HTTP | React Query | Redis | 实时性 |
|------|------|-------------|-------|--------|
| Share Token 签发 | `no-cache`（API 路径） | 60s（`queryKey: ['share', slug]`） | 无 | 低（Token 内容不变，60s 内复用） |
| 统计图表数据 | `no-cache`（API 路径） | 60s | 无 | 中（60s 后重新请求，API 每次穿透到数据库） |
| Website 实体 | — | — | 86400s | 低（24h 内不会更新实体名/域名等） |
| 白标签配置 | — | — | 无 TTL | 低（长期不变） |
| Tracker 脚本文件 | `24h must-revalidate` | — | — | 低（静态资源） |
| Pixel 追踪响应 | `no-store` | — | — | 高（每次都记录） |
| 登录态 authKey | — | — | 可配置 | 中 |

### 5.3 缓存与权限裁剪的关系

- **Share Token 被 React Query 缓存 60 秒**：如果在此期间管理员修改了 `parameters`（增减了可见页面），查看者需要等缓存过期或刷新页面才能感知变化
- **Website 实体被 Redis 缓存 24 小时**：如果管理员删除了 Website，Share Token 签发接口 `/api/share/:slug` 仍能通过 `getWebsite()` 查到实体（Redis 缓存未失效），但 `canViewWebsite()` 中 `getEntity()` 查到的可能是旧数据。不过 `fetchWebsite()` 在 `src/lib/load.ts:15` 中检查了 `deletedAt`，软删除的实体会返回 null
- **`x-umami-cache` 采集缓存 Token 与 Share Token 完全不同**：前者是 `src/app/api/send/route.ts:311` 签发的，payload 为 `{ websiteId, sessionId, visitId, iat }`，用于采集端避免重复创建 session；后者是 Share 系统签发的，payload 为 `{ shareId, shareType, websiteId?, ... }`，用于只读 API 认证

### 5.4 Redis `fetch()` 的软删除机制

`src/lib/redis.ts:84-98`：

```typescript
async fetch(key: string, query: () => Promise<any>, time?: number) {
  const result = await this.get(key);
  if (result === DELETED) return null;     // 软删除标记
  if (!result && query) {
    const data = await query();
    if (data) await this.set(key, data, time);
    return data;
  }
  return result;
}
```

`DELETED = '__DELETED__'` 是一个哨兵值，`remove(key, soft=true)` 会写入此值而非删除键。这防止了缓存击穿：删除后再次请求时，`fetch()` 先读到 `DELETED` 直接返回 null，不会触发数据库查询。

---

## 6. 全景：请求从进入到响应的完整链路

### 6.1 iframe 嵌入 Share 页面的请求链路

```
1. 宿主页面 <iframe src="https://umami.example.com/share/abc123">
   │
   ├─ 浏览器检查 CSP frame-ancestors
   │  └─ 'self' https://宿主域名 → 放行
   │  └─ 不在白名单 → 浏览器拦截，不加载
   │
2. GET /share/abc123（页面请求）
   │  └─ Next.js SSR，返回 HTML
   │
3. 前端 JS 执行 ShareProvider
   │  └─ GET /api/share/abc123（签发 Token）
   │     └─ 数据库查 Share → 组装 payload → 签 JWT → 返回
   │     └─ Redis 查 white-label:* （可选）
   │
4. setShareData() 写入 Zustand store
   │
5. SharePage 渲染
   │  └─ 按 shareType 分发到 WebsitePage / BoardViewPage / ...
   │  └─ parameters 过滤导航项
   │  └─ showActions={false} 隐藏管理按钮
   │
6. 页面组件发起 API 请求（如 GET /api/websites/:id/stats）
   │  └─ useApi() 自动附加 x-umami-share-token + x-umami-share-context
   │  └─ checkAuth() 校验 Token + 上下文头
   │  └─ canViewWebsite() 校验实体 ID 匹配
   │  └─ 返回统计数据（Cache-Control: no-cache）
   │
7. React Query 缓存 60 秒
```

### 6.2 采集脚本的请求链路（对比）

```
1. 被测网站 <script src="https://umami.example.com/script.js">
   │
2. GET /script.js
   │  └─ 返回 Tracker 脚本（Cache-Control: 24h, CORS: *）
   │
3. Tracker 在被测网站中运行，POST /api/send
   │  └─ skipAuth: true，无认证
   │  └─ fetchWebsite() 查实体（Redis 缓存 24h）
   │  └─ 保存事件数据
   │  └─ 返回 x-umami-cache Token（用于会话复用）
```

两条链路的 URL 空间、认证方式、CSP 头和缓存策略完全隔离。采集脚本的 `x-umami-cache` Token 和 Share 的 `x-umami-share-token` 是不同的 JWT，由不同的代码签发、不同的代码消费。

---

## 7. 关键文件索引

| 功能 | 仓库相对路径 |
|------|-------------|
| Share 数据模型 | `prisma/schema.prisma` (Share model) |
| Share 数据库查询 | `src/queries/prisma/share.ts` |
| Share Token 签发 API | `src/app/api/share/[slug]/route.ts` |
| Share 创建 API | `src/app/api/share/route.ts` |
| Website Share 创建 API | `src/app/api/websites/[websiteId]/shares/route.ts` |
| Board Share 创建 API | `src/app/api/boards/[boardId]/shares/route.ts` |
| Link Share 创建 API | `src/app/api/links/[linkId]/shares/route.ts` |
| Pixel Share 创建 API | `src/app/api/pixels/[pixelId]/shares/route.ts` |
| ShareProvider（Token 获取 + 权限裁剪） | `src/app/share/ShareProvider.tsx` |
| Share Layout（挂载 Provider） | `src/app/share/[slug]/layout.tsx` |
| SharePage（实体分发 + 布局） | `src/app/share/[slug]/[[...path]]/SharePage.tsx` |
| ShareNav（导航按 parameters 过滤） | `src/app/share/[slug]/[[...path]]/ShareNav.tsx` |
| ShareBranding（白标签 Logo） | `src/app/share/[slug]/[[...path]]/ShareBranding.tsx` |
| ShareFooter（白标签底部） | `src/app/share/[slug]/[[...path]]/ShareFooter.tsx` |
| CSP + 响应头 + Tracker 脚本配置 | `next.config.ts` |
| 认证校验（Share Token + 上下文头） | `src/lib/auth.ts` |
| JWT 工具 | `src/lib/jwt.ts` |
| 常量（HEADER 名 / ENTITY_TYPE） | `src/lib/constants.ts` |
| Auth 类型定义 | `src/lib/types.ts` |
| API 请求头注入 | `src/components/hooks/useApi.ts` |
| Share Token 查询 Hook | `src/components/hooks/queries/useShareTokenQuery.ts` |
| Zustand Store | `src/store/app.ts` |
| Website 权限 | `src/permissions/website.ts` |
| Board 权限 | `src/permissions/board.ts` |
| Link 权限 | `src/permissions/link.ts` |
| Pixel 权限 | `src/permissions/pixel.ts` |
| React Query 缓存配置 | `src/app/Providers.tsx` |
| Redis 缓存实现 | `src/lib/redis.ts` |
| 实体数据缓存 | `src/lib/load.ts` |
| 采集端点（API） | `src/app/api/send/route.ts` |
| Pixel 采集端点 | `src/app/(collect)/p/[slug]/route.ts` |
| Link 采集端点 | `src/app/(collect)/q/[slug]/route.ts` |
