# Umami 嵌入组件与公开 Dashboard 实现边界分析

本文档深入分析 Umami 项目中嵌入组件（Embedded Components）和公开 Dashboard（Public Dashboard/Share）的实现边界，涵盖六大维度：Share Token、iframe/script 渲染、CSP、权限裁剪、尺寸适配和缓存策略。

---

## 1. Share Token 机制

### 1.1 数据模型

Share 实体定义于 [schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/prisma/schema.prisma#L348-L360)：

```prisma
model Share {
  id         String    @id @map("share_id") @db.Uuid
  entityId   String    @map("entity_id") @db.Uuid      // 关联的实体ID（website/board/link/pixel）
  name       String    @db.VarChar(200)
  shareType  Int       @map("share_type") @db.Integer  // ENTITY_TYPE: website=1, link=2, pixel=3, board=4
  slug       String    @unique @db.VarChar(100)        // 公开访问URL的唯一标识
  parameters Json                                      // 权限/配置参数（可见页面等）
  createdAt  DateTime?
  updatedAt  DateTime?
}
```

**关键点**：
- `slug` 是公开访问的入口，通过 `getRandomChars(16)` 自动生成 16 字符随机串，也支持用户自定义
- `shareType` 区分四种实体类型：网站(1)、短链接(2)、像素(3)、看板(4)
- `parameters` 是 JSON 字段，存储页面级权限（如 `{"overview": true, "events": true}`）

### 1.2 Token 创建流程

Token 在 [share/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/api/share/%5Bslug%5D/route.ts#L47-L105) 的 GET 接口中签发：

```
请求路径: GET /api/share/:slug
```

**步骤**：

1. **查找 Share 记录**：通过 `slug` 从数据库查找 Share 实体
2. **解析实体类型**：根据 `shareType` 加载对应的实体（website/board/link/pixel）
3. **构建 Token Payload**：
   - `shareId`, `shareType`, `parameters`
   - Board 类型额外包含：`boardId`, `websiteIds[]`, `pixelIds[]`, `linkIds[]`
   - Website/Pixel/Link 类型额外包含：`websiteId`
   - 可选：`whiteLabel`（白标签配置，从 Redis 获取）
4. **签发 JWT**：`createToken(data, secret())`，使用全局密钥签名

Token 生成代码位于 [jwt.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/jwt.ts#L4-L6)：
```typescript
export function createToken(payload: any, secret: any, options?: any) {
  return jwt.sign(payload, secret, options);
}
```

### 1.3 Token 传递机制

**前端存储**：
- 获取 Token 后，通过 [useShareTokenQuery.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/components/hooks/queries/useShareTokenQuery.ts#L4-L18) 存入 Zustand store
- Store 定义在 [app.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/store/app.ts#L35-L40)：
```typescript
export function setShareData(share: object | null, shareToken: { token?: string } | null) {
  store.setState({ share, shareToken });
}
```

**请求头传递**：
- [useApi.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/components/hooks/useApi.ts#L20-L33) 自动检测路径并注入头：
```typescript
const isSharePath = pathname?.startsWith('/share');
const shareHeaders = isSharePath && shareToken?.token
  ? { 
      [SHARE_TOKEN_HEADER]: shareToken.token,    // x-umami-share-token
      [SHARE_CONTEXT_HEADER]: '1'                // x-umami-share-context
    }
  : {};
```

**后端校验**：
- [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/auth.ts#L17-L60) 的 `checkAuth()` 函数：
```typescript
// 1. 解析 Bearer Token（用户登录态）
// 2. 解析 Share Token（公开访问态）
const shareToken = await parseShareToken(request);

// 关键安全检查：Share Token 必须配合上下文头使用
if (!user?.id && shareToken) {
  const shareContext = request.headers.get(SHARE_CONTEXT_HEADER);
  if (!shareContext) {
    return null;  // 拒绝直接用 Share Token 脱离上下文调用 API
  }
}
```

**核心安全设计**：
- `SHARE_CONTEXT_HEADER`（`x-umami-share-context`）是防止 CSRF 的关键
- 前端仅在 `/share/*` 路径下自动附加此头，阻止攻击者通过其他路径滥用 Share Token

### 1.4 Share 创建授权

创建 Share 接口在 [share/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/api/share/route.ts#L10-L41)：

```typescript
// 必须通过 canUpdateEntity 验证：只有实体所有者/团队成员才能创建分享链接
if (!(await canUpdateEntity(auth, entityId))) {
  return unauthorized();
}
```

---

## 2. iframe/script 嵌入渲染机制

### 2.1 路由结构

```
/share/[slug]/[[...path]]
  ├── layout.tsx          → ShareProvider 注入
  ├── ShareProvider.tsx   → 获取 Token、权限裁剪、Context 提供
  └── [[...path]]/
       ├── page.tsx       → SharePage 组件入口
       ├── SharePage.tsx  → 根据类型渲染对应页面
       ├── ShareNav.tsx   → 侧边导航
       ├── ShareBranding.tsx → Logo/白标签
       └── ShareFooter.tsx
```

**布局嵌套**：
- 全局 layout → [share/[slug]/layout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/share/%5Bslug%5D/layout.tsx) → ShareProvider 包裹子路由

### 2.2 ShareProvider 工作流

定义于 [ShareProvider.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/share/ShareProvider.tsx#L54-L87)：

```
加载阶段:
  1. useShareTokenQuery(slug)  → 调用 /api/share/:slug 获取 Token
  2. setShareData(data, { token }) → 存入 Zustand store
  3. 等待加载完成（Loading 状态）

权限阶段:
  1. 识别 shareType（website / board / pixel / link）
  2. 计算 allowedSections：从 parameters 中提取为 true 的页面ID
  3. 单页面分享自动重定向：若仅允许一个非 overview 页面，直接跳转

渲染阶段:
  <ShareContext.Provider value={{ ...share, slug }}>
    {children}
  </ShareContext.Provider>
```

### 2.3 SharePage 实体分发

[SharePage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/share/%5Bslug%5D/%5B%5B...path%5D%5D/SharePage.tsx#L61-L142) 实现实体类型分发：

```typescript
const entityPage =
  shareType === ENTITY_TYPE.board && boardId ? (
    <BoardViewPage boardId={boardId} showActions={false} />
  ) : shareType === ENTITY_TYPE.pixel && pixelId ? (
    <PixelPage pixelId={pixelId} showHeaderActions={false} />
  ) : shareType === ENTITY_TYPE.link && linkId ? (
    <LinkPage linkId={linkId} showHeaderActions={false} />
  ) : null;
```

**Website 类型复用主应用组件**：
```typescript
const PAGE_COMPONENTS: Record<string, React.ComponentType<{ websiteId: string }>> = {
  '': WebsitePage,
  overview: WebsitePage,
  events: EventsPage,
  sessions: SessionsPage,
  realtime: RealtimePage,
  performance: PerformancePage,
  compare: ComparePage,
  breakdown: BreakdownPage,
  goals: GoalsPage,
  funnels: FunnelsPage,
  journeys: JourneysPage,
  retention: RetentionPage,
  utm: UTMPage,
  revenue: RevenuePage,
  attribution: AttributionPage,
};
```

**关键边界差异**：
- `WebsiteHeader showActions={false}` → 隐藏编辑/设置按钮
- Board/Pixel/Link 各自的 `showActions={false}` → 禁用管理操作
- 与主应用（`/websites/:id`）复用完全相同的底层页面组件，仅通过 props 控制操作区显示

### 2.4 iframe 嵌入方式

Umami 的嵌入使用 **iframe 直接引用 share URL** 的模式：

```html
<iframe 
  src="https://your-umami-instance.com/share/abc123def456" 
  width="100%" 
  height="800" 
  frameborder="0">
</iframe>
```

**嵌入与独立访问的区别**：
- 独立访问：完整的侧边导航 + 响应式布局
- iframe 嵌入：依赖 CSP 的 `frame-ancestors` 放行，尺寸由 iframe 容器决定
- 两者使用完全相同的 React 组件树，无特殊 embed 模式分支

---

## 3. CSP（内容安全策略）

### 3.1 配置位置

全部定义于 [next.config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/next.config.ts#L25-L95)。

### 3.2 默认响应头（全路径 `/:path*`）

```
Content-Security-Policy:
  default-src 'self';
  img-src 'self' https: data:;
  script-src 'self' 'unsafe-eval' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  connect-src 'self' https:;
  frame-ancestors 'self' ${ALLOWED_FRAME_URLS};
```

**iframe 嵌入控制（frame-ancestors）**：
- 默认仅允许 `'self'`（同源嵌入）
- 通过环境变量 `ALLOWED_FRAME_URLS` 追加信任域
- 示例：`ALLOWED_FRAME_URLS="https://example.com https://trusted.com"`
- 若不配置此变量，外部域名的 iframe 将被浏览器拦截

**各指令说明**：

| 指令 | 值 | 用途 |
|------|----|------|
| `default-src` | `'self'` | 默认加载源限制为同源 |
| `img-src` | `'self' https: data:` | 允许自托管、HTTPS 远程、Data URI 图片 |
| `script-src` | `'self' 'unsafe-eval' 'unsafe-inline'` | 内联脚本和 eval（Chart.js 等依赖） |
| `style-src` | `'self' 'unsafe-inline'` | 内联样式（UI 组件库动态样式） |
| `connect-src` | `'self' https:` | API 请求和 WebSocket |
| `frame-ancestors` | `'self' + ALLOWED_FRAME_URLS` | iframe 嵌入白名单 |

### 3.3 API 路径头（`/api/:path*`）

```
Access-Control-Allow-Origin: *        → 允许跨域 API 调用
Access-Control-Allow-Headers: *
Access-Control-Allow-Methods: GET, DELETE, POST, PUT
Access-Control-Max-Age: 86400
Cache-Control: no-cache               → 不缓存 API 响应
```

**设计意图**：
- Share 场景下，iframe 中的页面会通过浏览器直接向同域 API 发送请求
- 若 tracker 脚本跨域部署（如自定义域名），则依赖 CORS 的 `*` 放行

### 3.4 Tracker 脚本头（仅生产环境）

```
Access-Control-Allow-Origin: *
Cache-Control: public, max-age=86400, must-revalidate
```

注意：Tracker 脚本（`/script.js`）与 Share/Dashboard 嵌入机制相互独立，仅用于数据采集。

### 3.5 X-Frame-Options 说明

Umami **未使用** `X-Frame-Options` 头，完全依赖 CSP 的 `frame-ancestors` 指令。现代浏览器对 `frame-ancestors` 的支持已覆盖所有主流版本。

---

## 4. 权限裁剪（Permission Scoping）

### 4.1 权限层级结构

```
┌─────────────────────────────────────────────────────────┐
│  层级 1：实体级（Entity-Level）                          │
│  canViewWebsite / canViewBoard / canViewLink / canViewPixel │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  层级 2：Share Token 级（Token Payload 字段匹配）         │
│  shareToken.websiteId == targetId                       │
│  shareToken.websiteIds.includes(targetId)               │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  层级 3：页面/模块级（parameters JSON）                   │
│  parameters.events === true                             │
│  ShareProvider.allowedSections 过滤                     │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  层级 4：UI 操作级（Props 控制）                         │
│  showActions={false} / allowLink={false}                │
└─────────────────────────────────────────────────────────┘
```

### 4.2 实体级权限检查

**Website** - [permissions/website.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/permissions/website.ts#L7-L40)：

```typescript
export async function canViewWebsite({ user, shareToken }: Auth, websiteId: string) {
  // 1. 管理员直接放行
  if (user?.isAdmin) return true;
  
  // 2. Share Token 匹配（核心裁剪逻辑）
  if (
    shareToken?.websiteId === websiteId ||
    shareToken?.pixelId === websiteId ||
    shareToken?.linkId === websiteId ||
    shareToken?.websiteIds?.includes(websiteId) ||
    shareToken?.pixelIds?.includes(websiteId) ||
    shareToken?.linkIds?.includes(websiteId)
  ) return true;
  
  // 3. 登录用户检查（所有者或团队成员）
  const entity = await getEntity(websiteId);
  if (entity.userId) return user.id === entity.userId;
  if (entity.teamId) return !!await getTeamUser(entity.teamId, user.id);
  
  return false;
}
```

**Board** - [permissions/board.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/permissions/board.ts#L6-L36)：
```typescript
if (shareToken?.boardId === boardId) return true;
```

**Link** - [permissions/link.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/permissions/link.ts#L6-L32)：
```typescript
if (shareToken?.linkId === linkId || 
    shareToken?.websiteId === linkId || 
    shareToken?.linkIds?.includes(linkId)) return true;
```

**Pixel** - [permissions/pixel.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/permissions/pixel.ts#L6-L32)：
```typescript
if (shareToken?.pixelId === pixelId || 
    shareToken?.websiteId === pixelId || 
    shareToken?.pixelIds?.includes(pixelId)) return true;
```

### 4.3 页面级权限裁剪

**ShareProvider 中的导航过滤** - [ShareProvider.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/share/ShareProvider.tsx#L61-L76)：

```typescript
const ALL_SECTION_IDS = [
  'overview', 'events', 'sessions', 'realtime', 'performance',
  'compare', 'breakdown', 'goals', 'funnels', 'journeys',
  'retention', 'utm', 'revenue', 'attribution'
];

// 只保留 parameters 中标记为 true 的页面
const allowedSections =
  isWebsiteShare && share?.parameters
    ? ALL_SECTION_IDS.filter(id => share.parameters[id] === true)
    : [];

// 单页面分享自动跳转
const shouldRedirect =
  isWebsiteShare &&
  allowedSections.length === 1 &&
  allowedSections[0] !== 'overview' &&
  (path === undefined || path === '' || path === 'overview');
// → router.replace(`/share/${slug}/${allowedSections[0]}`)
```

**ShareNav 中的导航项过滤** - [ShareNav.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/share/%5Bslug%5D/%5B%5B...path%5D%5D/ShareNav.tsx#L121-L127)：

```typescript
const items = allItems
  .map(section => ({
    label: section.label,
    items: section.items.filter(item => parameters[item.id] === true),
  }))
  .filter(section => section.items.length > 0);
```

**SharePage 中的非法路径拦截** - [SharePage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/share/%5Bslug%5D/%5B%5B...path%5D%5D/SharePage.tsx#L86-L103)：

```typescript
const pageKey = path || '';
const isAllowed = pageKey === '' || parameters[pageKey] === true;

useEffect(() => {
  if (!isAllowed) {
    router.replace(`/share/${slug}`);  // 未授权页面强制跳回首页
  }
}, [isAllowed, slug, router]);
```

### 4.4 UI 操作级裁剪

Share 场景下通过 props 禁用管理操作：

| 组件 | Prop | 效果 |
|------|------|------|
| `BoardViewPage` | `showActions={false}` | 隐藏编辑/删除/设计按钮 |
| `PixelPage` | `showHeaderActions={false}` | 隐藏像素管理操作 |
| `LinkPage` | `showHeaderActions={false}` | 隐藏链接管理操作 |
| `WebsiteHeader` | `showActions={false}, allowLink={false}` | 隐藏网站设置/编辑菜单，禁用点击跳转 |

### 4.5 Auth 类型定义

[types.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/types.ts#L15-L31) 中定义了 Share Token 的结构：

```typescript
export interface Auth {
  user?: { id: string; username: string; role: string; isAdmin: boolean };
  shareToken?: {
    websiteId?: string;       // 单网站分享
    websiteIds?: string[];    // 多网站看板
    boardId?: string;         // 看板分享
    pixelId?: string;         // 单像素
    pixelIds?: string[];      // 多像素
    linkId?: string;          // 单链接
    linkIds?: string[];       // 多链接
  };
}
```

---

## 5. 尺寸适配（Responsive Sizing）

### 5.1 响应式断点系统

Umami 使用 `@umami/react-zen` UI 库的断点系统，在代码中表现为：

```typescript
columns={{ base: '1fr', lg: `${navCollapsed ? '60px' : '240px'} 1fr` }}
display={{ base: 'flex', lg: 'none' }}
```

断点约定（推断自 CSS 类名）：
- `base` / 无后缀：移动端（< 1024px）
- `lg`：桌面端（≥ 1024px）

### 5.2 SharePage 布局

[SharePage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/share/%5Bslug%5D/%5B%5B...path%5D%5D/SharePage.tsx#L120-L141) 的 Grid 布局：

```typescript
<Grid columns={{ base: '1fr', lg: `${navCollapsed ? '60px' : '240px'} 1fr` }} width="100%">
  <!-- 移动端：汉堡菜单 -->
  <Row display={{ base: 'flex', lg: 'none' }}>
    <MobileMenuButton>{({ close }) => <ShareNav onItemClick={close} />}</MobileMenuButton>
  </Row>
  
  <!-- 桌面端：侧边栏 -->
  <Column display={{ base: 'none', lg: 'flex' }}>
    <ShareNav collapsed={navCollapsed} onCollapse={handleCollapse} />
  </Column>
  
  <!-- 主内容区 -->
  <PageBody gap>
    <WebsiteProvider websiteId={websiteId}>...</WebsiteProvider>
  </PageBody>
</Grid>
```

**布局行为**：

| 场景 | 断点 | 导航样式 | 列宽 |
|------|------|----------|------|
| 小屏 iframe / 手机 | < 1024px | 顶部汉堡菜单，点击弹出抽屉 | 单列 1fr |
| 大屏 iframe / 桌面（展开） | ≥ 1024px | 左侧固定侧边栏 | 240px + 1fr |
| 大屏 iframe / 桌面（收起） | ≥ 1024px | 左侧窄条图标栏 | 60px + 1fr |

### 5.3 ShareNav 尺寸细节

[ShareNav.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/share/%5Bslug%5D/%5B%5B...path%5D%5D/ShareNav.tsx#L135-L144)：

```typescript
<Column
  position={isMobile ? undefined : 'fixed'}  // 桌面端固定定位
  paddingX={collapsed ? '1' : '3'}
  paddingY="3"
  width={isMobile ? '100%' : collapsed ? '60px' : '240px'}
  maxHeight="100dvh"   // 动态视口高度（兼容移动浏览器地址栏）
  height="100dvh"
  border={isMobile ? undefined : 'right'}
>
```

**状态持久化**：
```typescript
const [navCollapsed, setNavCollapsed] = useState(
  () => typeof window !== 'undefined' && localStorage.getItem('share:navCollapsed') === 'true'
);

const handleCollapse = (value: boolean) => {
  localStorage.setItem('share:navCollapsed', String(value));
  setNavCollapsed(value);
};
```

### 5.4 Board/Pixel/Link 单列布局

非 Website 类型的 Share 使用极简单列布局：
```typescript
<Column>
  {entityPage}       //  BoardViewPage / PixelPage / LinkPage
  <ShareFooter />
</Column>
```
- 无侧边导航，完全由页面组件自身决定宽度
- 适合嵌入固定尺寸的 iframe

### 5.5 iframe 嵌入的注意事项

1. **最小宽度**：图表组件（Chart.js）在 < 320px 宽度下可能渲染异常
2. **最小高度**：完整 dashboard（含图表 + 数据表格）建议 ≥ 600px
3. **dvh 兼容性**：使用 `100dvh`（dynamic viewport height）避免移动浏览器地址栏抖动
4. **无自适应通信**：Umami 未实现 `postMessage` 进行 iframe↔父窗口的高度同步，需宿主手动设置固定高度或使用滚动

---

## 6. 缓存策略

### 6.1 三层缓存架构

```
┌──────────────────────────────────────┐
│  层级 1：HTTP 响应头 Cache-Control   │
│  (next.config.ts headers)            │
├──────────────────────────────────────┤
│  层级 2：React Query 前端缓存        │
│  (Providers.tsx staleTime)           │
├──────────────────────────────────────┤
│  层级 3：Redis 服务端缓存            │
│  (redis.ts + load.ts)                │
└──────────────────────────────────────┘
```

### 6.2 HTTP 缓存层

**全路径** - [next.config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/next.config.ts#L34-L50)：
- 无显式 Cache-Control → Next.js 默认 SSR 行为（no-cache 用于动态渲染）
- Share 页面（`/share/[slug]`）是动态路由，每次请求服务器渲染

**API 路径** (`/api/:path*`)：
```
Cache-Control: no-cache
```
- 所有 API 强制不缓存，保证 Share 数据的实时性
- 包括 `/api/share/:slug` Token 签发接口

**Tracker 脚本** (`/script.js`)：
```
Cache-Control: public, max-age=86400, must-revalidate
```
- 与 Share 机制无关，仅用于采集脚本缓存 24 小时

### 6.3 React Query 前端缓存

配置在 [Providers.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/Providers.tsx#L11-L19)：

```typescript
const client = new QueryClient({
  defaultOptions: {
    queries: {
      retry: false,                          // 失败不重试
      refetchOnWindowFocus: false,           // 窗口聚焦不重拉
      staleTime: 1000 * 60,                  // 数据 60 秒内视为新鲜
    },
  },
});
```

**Share Token 查询** - [useShareTokenQuery.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/components/hooks/queries/useShareTokenQuery.ts#L4-L18)：
```typescript
queryKey: ['share', slug]
// 缓存键按 slug 隔离，不同 share 链接互不影响
// 60 秒内重新访问 share 页面，Token 不会重复签发
```

**影响范围**：
- 所有通过 `useApi().useQuery` 发起的数据请求（图表数据、列表等）
- Share 场景下统计数据 60 秒内走缓存，超过则重新请求 API

### 6.4 Redis 服务端缓存

Redis 客户端封装于 [redis.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/redis.ts)，可选启用（通过 `REDIS_URL` 环境变量）。

**基础能力**：

```typescript
export const DEFAULT_TTL = 3600;  // 默认 1 小时

async fetch(key: string, query: () => Promise<any>, time?: number) {
  const result = await this.get(key);
  if (result === DELETED) return null;
  if (!result && query) {
    const data = await query();
    if (data) await this.set(key, data, time);
    return data;
  }
  return result;
}
```

**Share 场景下的缓存点**：

1. **白标签配置** - [share/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/api/share/%5Bslug%5D/route.ts#L33-L45)：
```typescript
async function getWhiteLabel(accountId: string): Promise<WhiteLabel | null> {
  if (!redis.enabled) return null;
  const data = await redis.client.get(`white-label:${accountId}`);
  // 无 TTL，白标签配置通常长期不变
  return data;
}
```

2. **Website 实体** - [load.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/load.ts#L6-L20)：
```typescript
await redis.client.fetch(
  `website:${websiteId}`, 
  () => getWebsite(websiteId), 
  86400  // TTL: 24 小时
);
```

3. **Session 实体** - [load.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/load.ts#L22-L40)：
```typescript
await redis.client.fetch(
  `session:${sessionId}`, 
  () => getWebsiteSession(websiteId, sessionId), 
  86400  // TTL: 24 小时
);
```

4. **用户登录态** - [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/auth.ts#L62-L74)：
```typescript
// authKey 存储在 Redis，可设置过期时间
await redis.client.set(authKey, data);
if (expire) await redis.client.expire(authKey, expire);
```

**缓存失效策略**：
- 未找到主动失效代码（如更新 Website 后删除 Redis 键）
- 依赖 TTL 自然过期，配合 `DELETED` 标记实现软删除
- `fetch()` 方法中检查 `DELETED` 标记并返回 null

### 6.5 Share 数据实时性权衡

| 数据类型 | 缓存层 | TTL | 实时性 |
|---------|--------|-----|--------|
| Share Token | React Query | 60 秒 | 低（Token 内容不变） |
| Share 权限配置 | 无 | - | 高（每次请求数据库） |
| 白标签 | Redis | 无限期 | 低 |
| Website 元数据 | Redis | 24 小时 | 低 |
| 统计图表数据 | React Query | 60 秒 | 中 |
| API 响应 | HTTP | no-cache | 高（穿透到后端） |

---

## 7. 实现边界总结表

| 维度 | 嵌入组件 (Embed via iframe) | 公开 Dashboard (Share Link) | 主应用登录态 |
|------|---------------------------|----------------------------|-------------|
| **访问入口** | `/share/:slug` 通过 iframe src | `/share/:slug` 直接访问 | `/websites/:id` 等 |
| **认证方式** | Share Token (JWT) | Share Token (JWT) | Bearer Token (Secure JWT) |
| **安全头** | `x-umami-share-token` + `x-umami-share-context` | 同左 | `Authorization: Bearer` |
| **CSP frame-ancestors** | 需 `ALLOWED_FRAME_URLS` 放行 | `'self'` 即可 | 不适用 |
| **权限范围** | Token Payload 字段 + parameters 页面 | 同左 | 用户角色权限（admin/owner/team member） |
| **导航菜单** | 根据 parameters 过滤，可收起 | 同左 | 完整导航 |
| **操作按钮** | `showActions={false}` 全部禁用 | 同左 | 完整操作（编辑/删除/设置） |
| **尺寸适配** | 依赖 iframe 容器尺寸 + 响应式断点 | 浏览器全屏响应式 | 浏览器全屏响应式 |
| **数据缓存** | React Query 60s + Redis（可选） | 同左 | 同左 |
| **UI 品牌** | 支持白标签（whiteLabel） | 同左 | Umami 默认品牌 |
| **路径隔离** | `/share/*` 路由组 | `/share/*` 路由组 | `/(main)/*` 路由组 |

---

## 8. 关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| Share 数据模型 | [schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/prisma/schema.prisma#L348-L360) |
| Share 数据库查询 | [queries/prisma/share.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/queries/prisma/share.ts) |
| Share Token 签发 API | [api/share/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/api/share/%5Bslug%5D/route.ts) |
| Share 创建 API | [api/share/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/api/share/route.ts) |
| ShareProvider（Token 获取 + 权限） | [ShareProvider.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/share/ShareProvider.tsx) |
| SharePage（页面分发） | [SharePage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/share/%5Bslug%5D/%5B%5B...path%5D%5D/SharePage.tsx) |
| ShareNav（导航过滤） | [ShareNav.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/share/%5Bslug%5D/%5B%5B...path%5D%5D/ShareNav.tsx) |
| CSP + 响应头配置 | [next.config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/next.config.ts) |
| 认证检查（Share Token 校验） | [lib/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/auth.ts) |
| JWT Token 工具 | [lib/jwt.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/jwt.ts) |
| API 请求头注入 | [hooks/useApi.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/components/hooks/useApi.ts) |
| Website 权限 | [permissions/website.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/permissions/website.ts) |
| Board 权限 | [permissions/board.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/permissions/board.ts) |
| Link 权限 | [permissions/link.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/permissions/link.ts) |
| Pixel 权限 | [permissions/pixel.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/permissions/pixel.ts) |
| React Query 缓存配置 | [Providers.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/app/Providers.tsx) |
| Redis 缓存实现 | [lib/redis.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/redis.ts) |
| 实体数据缓存 | [lib/load.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/load.ts) |
| Zustand Store（Share 状态） | [store/app.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/store/app.ts) |
| 常量定义 | [lib/constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/constants.ts) |
| Auth 类型定义 | [lib/types.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/50-umami/src/lib/types.ts#L15-L31) |
