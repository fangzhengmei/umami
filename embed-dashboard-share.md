# Umami 嵌入组件与公开 Dashboard 实现边界分析

本文档从代码实际出发，梳理 Umami 三条产品线的边界：采集脚本（Tracker）、公开 Dashboard（Share 页面）、iframe 嵌入。每条线有独立的 URL 空间、认证方式、CSP 头和缓存策略。

---

## 1. 三条产品线的边界

### 1.1 采集脚本：`/script.js` → `/api/send`

**脚本来源**：`/script.js` 默认是**静态文件**，由 Rollup 构建到 `public/script.js`（rollup.tracker.config.js: output.file = `public/script.js`）。构建命令：`npm run build-tracker` → `rollup -c rollup.tracker.config.js`。

**Rewrite 是可选机制**，仅在以下三种环境变量存在时触发（next.config.ts）：

| 场景 | 配置 | 行为 |
|------|------|------|
| 自定义脚本域名 | `TRACKER_SCRIPT_URL` | `/script.js` → `TRACKER_SCRIPT_URL` |
| 自定义脚本名 | `TRACKER_SCRIPT_NAME`（可多个，逗号分隔） | 如 `/analytics.js` → rewrite 到 `/script.js`，并附加 trackerHeaders |
| Cloud 模式 | `CLOUD_MODE=1`（仅生产） | `/script.js` → `https://cloud.umami.is/script.js` |

**默认情况（无特殊环境变量）**：无 rewrite，直接由 Next.js 从 `public/` 目录提供静态文件。

**响应头叠加**：`/script.js` 同时匹配两个 header source（按 Next.js 规则多条匹配则叠加）：

- `/:path*` → defaultHeaders（含 CSP + X-DNS-Prefetch-Control）
- `/script.js`（仅生产）→ trackerHeaders（`Access-Control-Allow-Origin: *` + `Cache-Control: public, max-age=86400, must-revalidate`）

注意：CSP 头虽然也出现在 `/script.js` 响应上，但 CSP 对子资源（JS 文件）本身不生效——CSP 由加载它的文档强制执行。

**采集端点**：`POST /api/send`，核心特征 `skipAuth: true`（src/app/api/send/route.ts），任何来源均可上报。返回的 `x-umami-cache` Token 仅用于会话复用，与 Share Token 无关。

### 1.2 Share 页面：`/share/:slug`

**路由结构**：

```
src/app/share/
├── ShareProvider.tsx                    ← Context Provider，触发 Token 获取
└── [slug]/
    ├── layout.tsx                       ← 挂载 ShareProvider
    └── [[...path]]/                     ← 可选子路径：events / sessions / ...
        ├── page.tsx                     → SharePage
        ├── SharePage.tsx                ← 实体分发 + 布局
        ├── ShareNav.tsx                 ← 侧边导航（按 parameters 过滤）
        ├── ShareBranding.tsx            ← Logo / 白标签
        └── ShareFooter.tsx              ← 底部品牌条
```

**加载流程**：

```
1. 浏览器 GET /share/abc123
2. layout.tsx 包裹 <ShareProvider slug={slug}>
3. ShareProvider 调用 useShareTokenQuery(slug)
   → GET /api/share/abc123
   → 后端查 DB 组装 payload → 签 JWT → 返回 { shareId, shareType, parameters, token, ... }
4. setShareData() 写入 Zustand store（src/store/app.ts）
5. ShareContext.Provider 向子组件提供完整 share 数据
6. SharePage 根据 shareType 分发到对应页面组件
7. 子组件内所有 API 调用经 useApi() 自动附加：
     x-umami-share-token: <jwt>
     x-umami-share-context: 1
```

**关键事实**：Share 页面中**没有任何采集脚本**。它是纯读取界面，通过只读 API 拉统计数据，不向 `/api/send` 写数据。

### 1.3 iframe 嵌入：`/share/:slug` 的另一种交付方式

iframe 嵌入和直接访问 Share 页面**在服务端完全相同**——同一路由、同一套组件、同一份 Token。区别仅在宿主侧：

```html
<iframe src="https://umami.example.com/share/abc123"
        width="100%" height="800" frameborder="0">
</iframe>
```

- 服务端无"embed 模式"分支
- 唯一服务端影响因素：CSP `frame-ancestors` 决定此页面能否被嵌入（见第 2 节）
- 唯一前端影响因素：视口尺寸由 iframe 容器决定（见第 4 节）

---

## 2. CSP 策略详解

全部定义在 `next.config.ts` 的 `contentSecurityPolicy` 常量中。

### 2.1 默认 CSP（匹配 `/:path*`，覆盖所有页面包括 `/share/*`）

```
default-src 'self';
img-src 'self' https: data:;
script-src 'self' 'unsafe-eval' 'unsafe-inline';
style-src 'self' 'unsafe-inline';
connect-src 'self' https:;
frame-ancestors 'self' ${ALLOWED_FRAME_URLS};
```

**各指令的实际作用对象**：

| 指令 | 在 Share 页面中的用途 | 代码依据 |
|------|----------------------|----------|
| `default-src 'self'` | 默认回退，所有未单独指定的资源类型仅限同源 | next.config.ts:26 |
| `img-src 'self' https: data:` | 白标签 Logo（`ShareBranding.tsx`）、国家旗帜图标、地图瓦片、data URI 内联图 | next.config.ts:27 |
| `script-src 'self' 'unsafe-eval' 'unsafe-inline'` | 页面主 JS、Chart.js 等第三方库需要 eval 和内联脚本 | next.config.ts:28 |
| `style-src 'self' 'unsafe-inline'` | 组件库动态样式、CSS-in-JS | next.config.ts:29 |
| `connect-src 'self' https:` | API 请求（同源，`/api/*`）、实时数据 WebSocket/SSE、远程服务调用 | next.config.ts:30 |
| `frame-ancestors 'self' ${frameAncestors}` | 控制哪些域名可以 iframe 嵌入此页面 | next.config.ts:31 |

### 2.2 frame-ancestors 与 iframe 嵌入

`frameAncestors` 来自环境变量 `ALLOWED_FRAME_URLS`（next.config.ts:19）。

- **不设此变量** → `frame-ancestors 'self'` → 仅同源可嵌入，外部域名 iframe 被浏览器拦截
- **设此变量** → `frame-ancestors 'self' https://trusted.com ...` → 白名单域名可嵌入
- **直接访问 `/share/:slug`** → 不受 `frame-ancestors` 影响（该指令只控制 iframe 嵌入）

**Umami 未使用 `X-Frame-Options`**，完全依赖 CSP `frame-ancestors`。

### 2.3 API 路径头（`/api/:path*`）

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: *
Access-Control-Allow-Methods: GET, DELETE, POST, PUT
Access-Control-Max-Age: 86400
Cache-Control: no-cache
```

对 Share 场景的影响：

- Share 页面通过浏览器同源请求 API → CORS 实际不生效（同源不触发预检）
- `Cache-Control: no-cache` → 每次 API 请求穿透到服务器，统计数据保持新鲜
- 若 Share 页面在 iframe 中跨域嵌入 → iframe 内的页面与 API 仍同源 → CORS 仍不需要

### 2.4 Tracker 脚本头（`/script.js`，仅生产）

```
Access-Control-Allow-Origin: *
Cache-Control: public, max-age=86400, must-revalidate
```

这与 Share 页面完全无关。Tracker 跑在被测网站域名下，需要 CORS `*` 才能向 Umami 后端发数据。

### 2.5 头策略总结

| 路径 | CSP | CORS | Cache-Control | 服务对象 |
|------|-----|------|---------------|---------|
| `/:path*`（含 `/share/*`） | 完整 6 条指令 | 无 | 无（Next.js 默认 SSR） | Share 页面 / 主应用 |
| `/api/:path*` | 无（被更上层的 `/:path*` 覆盖） | `*` | `no-cache` | 所有 API 消费者 |
| `/script.js`（生产） | 继承自 `/:path*`（但对子资源无效） | `*` | `public, max-age=86400` | 被测网站中的 Tracker |
| `/p/:slug`（Pixel） | 继承自 `/:path*` | 无 | `no-cache, no-store, must-revalidate` | Pixel 追踪 |

---

## 3. Share Token 的保护范围

### 3.1 Token 签发

`src/app/api/share/[slug]/route.ts` 的 GET 处理函数签发 JWT。

**签发接口本身无认证**：`GET /api/share/:slug` 不需要任何凭证，只要知道 slug 就能拿到 Token。slug 本身就是秘密（16 字符随机串，可自定义）。

**Payload 内容**（按 shareType 不同）：

| shareType | payload 字段 |
|-----------|-------------|
| `website` (1) | `shareId`, `shareType`, `parameters`, `websiteId` |
| `board` (4) | `shareId`, `shareType`, `parameters`, `boardId`, `websiteIds[]`, `pixelIds[]`, `linkIds[]` |
| `pixel` (3) | `shareId`, `shareType`, `parameters`, `websiteId`, `pixelId` |
| `link` (2) | `shareId`, `shareType`, `parameters`, `websiteId`, `linkId` |

可选附加：`whiteLabel`（从 Redis 读取，仅启用 Redis 时有）。

签发函数：`createToken(data, secret())`（src/lib/jwt.ts），标准 `jsonwebtoken` 库。

### 3.2 Token 传递：双请求头

前端 `src/components/hooks/useApi.ts` 仅在 `pathname?.startsWith('/share')` 时注入：

```typescript
const isSharePath = pathname?.startsWith('/share');
const shareHeaders = isSharePath && shareToken?.token
  ? {
      [SHARE_TOKEN_HEADER]: shareToken.token,    // x-umami-share-token
      [SHARE_CONTEXT_HEADER]: '1',               // x-umami-share-context
    }
  : {};
```

后端 `src/lib/auth.ts` 的 `checkAuth()` 校验：

```typescript
const shareToken = await parseShareToken(request);

if (!user?.id && shareToken) {
  const shareContext = request.headers.get(SHARE_CONTEXT_HEADER);
  if (!shareContext) {
    return null;  // 拒绝：Share Token 脱离上下文
  }
}
```

**关于 `x-umami-share-context` 的定位**：

这是一个**路径作用域标记**，不是强安全措施。值固定为 `"1"`，任何能设置自定义请求头的工具（curl / Postman / 攻击者脚本）都可以附加。它的作用是：

1. 防止 Share Token 在非 `/share/*` 路径下被意外/滥用
2. 作为一种轻量 CSRF 防御（普通跨站表单提交无法设置此头）

但如果攻击者已经拿到了 Token JWT，他们也能设置这个头。真正的安全边界在于：Token 本身只授予只读权限，且只能访问 Token 中列出的实体。

### 3.3 受保护的 API

Share Token 通过 `canViewWebsite / canViewBoard / canViewLink / canViewPixel` 四个权限函数保护只读 API。

**受保护的 API 示例**（均调用 `canViewWebsite(auth, websiteId)`）：

- `GET /api/websites/:id/stats` — 统计数据
- `GET /api/websites/:id/events` — 事件列表
- `GET /api/websites/:id/sessions` — 会话列表
- `GET /api/websites/:id/pageviews` — 页面浏览
- `GET /api/websites/:id/metrics` — 指标
- `GET /api/websites/:id/session-data/*` — 会话数据
- `GET /api/websites/:id/revenue/*` — 收入数据
- `GET /api/websites/:id/active` — 实时在线
- `GET /api/realtime/:websiteId` — 实时数据

Board / Pixel / Link 同理，各自有一套只读 API。

**不受 Share Token 保护的 API**：

| API | 原因 |
|-----|------|
| `GET /api/share/:slug` | 签发接口本身无需认证，slug 即秘密 |
| `POST /api/send` | `skipAuth: true`，采集端点公开 |
| `GET /api/config` | `skipAuth: true`，公开配置 |
| `POST /api/batch` | `skipAuth: true`，批量采集 |
| `POST /api/auth/login` | 登录接口，无需前置认证 |
| 所有 `canUpdate*` / `canDelete*` 的写接口 | 不检查 shareToken，仅登录用户可写 |
| `/api/users/*` / `/api/teams/*` / `/api/admin/*` | 管理类接口，需登录 |

### 3.4 实体级权限：`canView*`

**Website** — `src/permissions/website.ts` `canViewWebsite()`：

```typescript
if (user?.isAdmin) return true;
if (shareToken?.websiteId === websiteId ||
    shareToken?.pixelId === websiteId ||
    shareToken?.linkId === websiteId ||
    shareToken?.websiteIds?.includes(websiteId) ||
    shareToken?.pixelIds?.includes(websiteId) ||
    shareToken?.linkIds?.includes(websiteId)) return true;
// 然后检查用户所有权
```

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

### 3.5 页面级权限：`parameters` 过滤

仅对 Website 类型分享生效。`src/app/share/ShareProvider.tsx`：

```typescript
const ALL_SECTION_IDS = [
  'overview', 'events', 'sessions', 'realtime', 'performance',
  'compare', 'breakdown', 'goals', 'funnels', 'journeys',
  'retention', 'utm', 'revenue', 'attribution'
];

const allowedSections = isWebsiteShare && share?.parameters
  ? ALL_SECTION_IDS.filter(id => share.parameters[id] === true)
  : [];
```

过滤发生在三层：

1. **ShareNav**（ShareNav.tsx:122-127）：导航菜单只显示 `parameters[id] === true` 的条目
2. **SharePage**（SharePage.tsx:87-88）：访问未授权页面时强制跳回 `/share/:slug`
3. **ShareProvider**（ShareProvider.tsx:66-76）：若仅允许一个非 overview 页面，自动重定向

### 3.6 UI 级裁剪：Props 隐藏操作

| 组件 | Prop | 效果 |
|------|------|------|
| `WebsiteHeader` | `showActions={false}` | 隐藏设置/编辑按钮 |
| `WebsiteHeader` | `allowLink={false}` | 禁止点击跳转到主应用 |
| `BoardViewPage` | `showActions={false}` | 隐藏编辑/删除/设计 |
| `PixelPage` | `showHeaderActions={false}` | 隐藏管理操作 |
| `LinkPage` | `showHeaderActions={false}` | 隐藏管理操作 |

---

## 4. Pixel / Link 复用 `websiteId` 的意义

### 4.1 数据模型事实

`prisma/schema.prisma` 中三个实体是**互相独立**的：

```prisma
model Website {
  id        String  @id @map("website_id") @db.Uuid
  name      String
  domain    String?
  userId    String?
  teamId    String?
  // ...
}

model Link {
  id     String  @id @map("link_id") @db.Uuid
  name   String
  url    String
  slug   String  @unique
  userId String?
  teamId String?
  // 注意：没有 websiteId 字段
}

model Pixel {
  id     String  @id @map("pixel_id") @db.Uuid
  name   String
  slug   String  @unique
  userId String?
  teamId String?
  // 注意：没有 websiteId 字段
}
```

**Link 和 Pixel 在数据库层面不关联到 Website**，它们是平级的独立产品（都有自己的 slug、userId、teamId）。

### 4.2 Share Token 中的 `websiteId` 是别名

在 `src/app/api/share/[slug]/route.ts` 中：

```typescript
} else if (share.shareType === ENTITY_TYPE.pixel) {
  entity = await getPixel(share.entityId);
  if (!entity) return notFound();
  data.websiteId = share.entityId;   // ← Pixel 自己的 ID 被赋给 websiteId
  data.pixelId = share.entityId;
} else if (share.shareType === ENTITY_TYPE.link) {
  entity = await getLink(share.entityId);
  if (!entity) return notFound();
  data.websiteId = share.entityId;   // ← Link 自己的 ID 被赋给 websiteId
  data.linkId = share.entityId;
}
```

**为什么这么做？兼容性复用。**

`canViewWebsite()` 权限函数接受一个 `websiteId` 参数，内部检查 `shareToken.websiteId === websiteId`。如果 Pixel/Link 的 Share Token 也设置了 `websiteId`，那么同一个权限函数不经修改就能服务于 Pixel/Link 场景——调用方只需传实体 ID，`canViewWebsite(id)` 对三种类型都返回 true。

同样的逻辑也体现在 `canViewPixel()` 和 `canViewLink()` 中，它们也检查 `shareToken.websiteId === targetId`。

### 4.3 这意味着什么

- **概念混淆**：变量名叫 `websiteId`，但对 Pixel/Link 分享来说它实际上是"实体 ID"
- **副作用**：Pixel/Link 的 Share Token 可以通过 `canViewWebsite(pixelId)` 检查——因为 `shareToken.websiteId` 存的就是 pixelId 本身
- **为什么不用统一命名**：`canViewWebsite` 是最早的权限函数，Pixel/Link 是后加的产品，通过兼容别名减少改动量
- **实际安全**：没有实质安全问题，因为 Pixel/Link 本身的统计 API 还是调用各自的 `canViewPixel/canViewLink`，只是底层 ID 比对逻辑复用了

### 4.4 采集端的对应关系

采集侧也是同理：Pixel 和 Link 的追踪端点（`/p/:slug`、`/q/:slug`）内部都通过 `fetchWebsite()` 查找实体，但 `fetchWebsite` 实际上对 Website / Pixel / Link 都能查，靠的就是"实体 ID 当 websiteId 用"的约定。

`src/lib/load.ts` 的 `fetchWebsite(websiteId)` 注释里写明了这个模式：

> 此函数名虽叫 fetchWebsite，但实际上用于加载 Website / Pixel / Link 三类实体。历史上只有 Website，后加的 Pixel 和 Link 复用了同一套加载逻辑。

---

## 5. 尺寸适配

### 5.1 Share 页面的两种布局

**Website 类型**：带侧边导航的 Grid 布局（SharePage.tsx:121）：

```typescript
<Grid columns={{ base: '1fr', lg: `${navCollapsed ? '60px' : '240px'} 1fr` }} width="100%">
```

| 视口 | 导航 | 列结构 |
|------|------|--------|
| < 1024px（base） | 顶部汉堡菜单 | 单列 `1fr` |
| ≥ 1024px（lg），展开 | 左侧 240px 固定侧边栏 | `240px 1fr` |
| ≥ 1024px（lg），收起 | 左侧 60px 图标栏 | `60px 1fr` |

**Board / Pixel / Link 类型**：极简单列布局（SharePage.tsx:105-111）：

```typescript
<Column>
  {entityPage}
  <ShareFooter />
</Column>
```

无侧边导航，宽度完全由容器决定。

### 5.2 ShareNav 尺寸细节

`src/app/share/[slug]/[[...path]]/ShareNav.tsx:136-143`：

```typescript
<Column
  position={isMobile ? undefined : 'fixed'}
  width={isMobile ? '100%' : collapsed ? '60px' : '240px'}
  maxHeight="100dvh"
  height="100dvh"
>
```

`100dvh`（dynamic viewport height）兼容移动浏览器地址栏收缩/展开。

折叠状态持久化到 `localStorage`（键 `share:navCollapsed`）。

### 5.3 iframe 嵌入的尺寸约束

- **无 `postMessage` 高度同步**：Umami 未实现 iframe ↔ 父窗口的高度通信，宿主需手动设 `height` 或用滚动
- **最小宽度**：图表在 < 320px 下可能渲染异常
- **最小高度**：完整 Dashboard 建议 ≥ 600px
- **URL 参数控主题**：`?theme=light` / `?theme=dark`（SharePage.tsx:78-84）

### 5.4 尺寸适配与 CSP、权限的关系

- **尺寸 × CSP**：只有 `frame-ancestors` 放行后，iframe 才会加载，尺寸适配才有意义。直接访问不受影响
- **尺寸 × 权限**：互相独立。但小视口下侧边导航变为汉堡菜单，如果只分享了一个页面，菜单里只有一个选项，体验上可以直接用 Board 类型简化

---

## 6. 缓存策略与 Share 签发的关系

### 6.1 三层缓存架构

```
┌──────────────────────────────────────────────────────────────────────┐
│  HTTP Cache-Control（next.config.ts headers）                        │
│  ├─ /api/*      → no-cache          ← Share 页面读数据走这          │
│  ├─ /script.js  → 24h must-revalidate ← 采集脚本走这              │
│  └─ /p/*        → no-store           ← Pixel 采集走这              │
├──────────────────────────────────────────────────────────────────────┤
│  React Query（src/app/Providers.tsx, staleTime=60s）                │
│  └─ Share 页面 + 主应用共用 QueryClient                             │
│     Share Token 查询 key: ['share', slug]  staleTime=1h            │
│     统计数据查询 staleTime=60s                                       │
├──────────────────────────────────────────────────────────────────────┤
│  Redis（src/lib/redis.ts, 可选，REDIS_URL 启用）                    │
│  ├─ white-label:*  → 无 TTL        ← Share 白标签                  │
│  ├─ website:*      → 86400s        ← fetchWebsite()               │
│  ├─ session:*      → 86400s        ← fetchSession()               │
│  └─ auth:*         → 可配置        ← 用户登录态                    │
└──────────────────────────────────────────────────────────────────────┘
```

### 6.2 Share 签发接口与缓存的关系

`GET /api/share/:slug` 是 Share Token 的签发入口。它与缓存的关系：

**1. Share 记录查找 — 无缓存**

```typescript
const share = await getShareByCode(slug);  // src/queries/prisma/share.ts
```

`getShareByCode` 直接 `prisma.client.share.findUnique({ where: { slug } })`，**不经过 Redis**。每次请求都查数据库。

**2. 实体数据查找 — 无缓存**

```typescript
entity = await getWebsite(share.entityId);  // 或 getPixel / getLink / getBoard
```

这些函数都在 `src/queries/prisma/` 下，直接走 Prisma 查询。虽然 `src/queries/prisma/website.ts` 顶部 import 了 `redis`，但 `getWebsite` 函数本身并不用 Redis（Redis 缓存版本在 `src/lib/load.ts` 的 `fetchWebsite()` 中，但 share 路由不用它）。

**3. 白标签配置 — Redis 缓存（无 TTL）**

```typescript
const whiteLabel = await getWhiteLabel(accountId);
```

`getWhiteLabel`（share/[slug]/route.ts:33-45）：

```typescript
async function getWhiteLabel(accountId: string): Promise<WhiteLabel | null> {
  if (!redis.enabled) return null;
  const data = await redis.client.get(`white-label:${accountId}`);
  if (data) return data as WhiteLabel;
  return null;
}
```

- 仅在 `redis.enabled` 时生效
- Redis key: `white-label:${accountId}`
- **无 TTL**，写入后长期有效
- 没有找到主动写入/失效的代码（推测由白标签管理界面的其他路径写入）

**4. HTTP 层 — no-cache**

`/api/*` 路径的 `Cache-Control: no-cache` 也适用于 `/api/share/:slug`，因此浏览器不会缓存 Token 响应。

**5. React Query 层 — 1 小时**

前端 `useShareTokenQuery`（`src/components/hooks/queries/useShareTokenQuery.ts`）：

```typescript
useQuery({
  queryKey: ['share', slug],
  queryFn: () => getShare(slug),
  staleTime: 60 * 60 * 1000, // 1 hour
});
```

Token 在前端缓存 1 小时。期间刷新页面或重新访问同 slug 的 Share 页面，不会重新签发 Token。

### 6.3 权限变更的感知延迟

结合缓存层，Share 权限变更的生效时间：

| 变更类型 | 生效延迟 | 原因 |
|---------|---------|------|
| 修改 `parameters`（页面可见性） | 最长 1 小时（前端缓存） | React Query staleTime=1h |
| 修改 Share 名称 | 最长 1 小时 | 同上 |
| 删除 Share | 最长 1 小时（前端）+ 无后端缓存 | 前端缓存期内仍可访问；缓存过期后 404 |
| 修改白标签 | 立即（Redis 无 TTL，但需主动更新 Redis 键） | 白标签直接读 Redis |
| 修改实体名（Website / Pixel / Link） | 立即（签发时直接查 DB） | 签发接口不用 Redis 缓存实体 |

### 6.4 Redis 软删除机制

`src/lib/redis.ts:84-98` 的 `fetch()` 方法：

```typescript
async fetch(key: string, query: () => Promise<any>, time?: number) {
  const result = await this.get(key);
  if (result === DELETED) return null;     // 软删除哨兵
  if (!result && query) {
    const data = await query();
    if (data) await this.set(key, data, time);
    return data;
  }
  return result;
}
```

`DELETED = '__DELETED__'` 是哨兵值，`remove(key, soft=true)` 写入此值而非删键。防止缓存击穿——删除后再次请求直接返回 null，不触发 DB 查询。

但 Share 签发接口不使用 `fetch()`，所以软删除机制对它不生效。

---

## 7. 全景链路图

### 7.1 iframe 嵌入 Share 页面的完整链路

```
1. 宿主页面 <iframe src="https://umami.example.com/share/abc123">
   │
   ├─ 浏览器检查 CSP frame-ancestors
   │  └─ 'self' + ALLOWED_FRAME_URLS → 放行 / 拦截
   │
2. GET /share/abc123（页面请求）
   │  └─ Next.js SSR → 返回 HTML
   │  └─ CSP 响应头随 HTML 一起返回
   │
3. 前端 JS 执行 → ShareProvider → useShareTokenQuery
   │  └─ GET /api/share/abc123（签发 Token，无认证）
   │     └─ DB: getShareByCode(slug)  ← 无缓存
   │     └─ DB: getWebsite/getPixel/getLink/getBoard  ← 无缓存
   │     └─ Redis: white-label:${accountId}  ← 可选，无 TTL
   │     └─ 签 JWT → 返回 { token, shareId, shareType, ... }
   │
4. setShareData() → Zustand store
   │
5. SharePage 渲染
   │  └─ 按 shareType 分发到 WebsitePage / BoardViewPage / ...
   │  └─ parameters 过滤导航项
   │  └─ showActions={false} 隐藏管理按钮
   │
6. 页面组件发起 API（如 GET /api/websites/:id/stats）
   │  └─ useApi() 自动附加 x-umami-share-token + x-umami-share-context
   │  └─ checkAuth() 校验 Token + 上下文头
   │  └─ canViewWebsite() 校验实体 ID 匹配
   │  └─ 返回统计数据（HTTP no-cache）
   │
7. React Query 缓存 60s
```

### 7.2 采集脚本链路（对比）

```
1. 被测网站 <script src="https://umami.example.com/script.js">
   │
2. GET /script.js
   │  └─ 静态文件（public/script.js）
   │  └─ CORS: *, Cache-Control: 24h
   │  └─ CSP 头也附带（但对子资源无效）
   │
3. Tracker 在被测网站中运行 → POST /api/send
   │  └─ skipAuth: true，无认证
   │  └─ fetchWebsite() 查实体（Redis 缓存 24h）
   │  └─ 保存事件数据
   │  └─ 返回 x-umami-cache Token（会话复用）
```

两条链路完全独立：URL 不同、认证方式不同、Token 不同、缓存策略不同。采集脚本的 `x-umami-cache` Token 与 Share 的 `x-umami-share-token` 是两种不同的 JWT，由不同代码签发、不同代码消费。

---

## 8. 边界混淆点速查

| 容易混淆的点 | 事实 |
|------------|------|
| "embed 组件"和 share 页面是两套东西 | 不是。iframe 嵌入和直接访问共用同一 `/share/:slug` 路由和组件树 |
| Share 页面加载了采集脚本 | 没有。Share 是纯读取界面，不向 `/api/send` 写数据 |
| `/script.js` 是通过 rewrite 提供的 | 默认不是。它是 Rollup 构建到 `public/` 的静态文件。Rewrite 仅在特定环境变量下触发 |
| CSP 只作用于 Share 页面 | 不是。`/:path*` 匹配所有路径，包括 `/script.js`，但 CSP 对 JS 子资源响应无效 |
| Share Token 保护所有 API | 不是。只保护 `canView*` 覆盖的只读 API。`/api/share/:slug` 本身无认证，写接口不认 Share Token |
| `x-umami-share-context` 是强安全措施 | 不是。值固定为 "1"，是路径作用域标记/轻量 CSRF 防御 |
| Pixel/Link 有关联的 Website | 数据模型中没有。`websiteId` 在 Share Token 中是兼容别名，存的就是 Pixel/Link 自己的 ID |
| Share 签发用 Redis 缓存 Share 记录 | 不用。`getShareByCode` 直接查 DB。只有白标签走 Redis |
| Token 中 `websiteId` 一定对应 Website 实体 | 不一定。Pixel/Link 分享时它存的是 Pixel/Link 的 ID |
