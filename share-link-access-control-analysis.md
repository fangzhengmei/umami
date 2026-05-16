# Umami 共享链接访问控制机制分析（纠偏版）

## 1. Share 记录实际落库字段与 Token 存储事实

### 1.1 Share 表实际字段
`Share` 模型（`prisma/schema.prisma`）的核心字段为：
- `id`
- `entityId`
- `name`
- `shareType`
- `slug`（唯一）
- `parameters`（JSON）
- `createdAt` / `updatedAt`

数据访问入口在 `src/queries/prisma/share.ts`，主要按 `id` 或 `slug` 查询。

### 1.2 关键纠偏：Token 不落库
共享 Token 并不存储在 `Share` 表中。Token 是在访问共享入口时动态签发：
- 访问 `src/app/api/share/[slug]/route.ts`
- 后端按 `slug` 找到 share 记录后拼装 payload
- 调 `createToken(data, secret())` 生成 token 并返回给前端

因此，`Share` 负责持久化“共享配置”，而 Token 负责“访问时会话凭据”。

## 2. 共享链接短码（slug）的全链路来源分支

### 2.1 网站创建时写入共享短码（支持自定义）
`src/app/api/websites/route.ts` 在创建网站时接收可选 `shareId`：
- 若 `shareId` 存在，创建 share 记录并将 `slug: shareId`
- 若 `shareId` 不存在，不创建对应 share 记录

权限门槛：`canCreateWebsite`（若创建到团队还需 `canCreateTeamWebsite`）。

冲突处理：该路径未显式捕获 slug 唯一冲突，数据库唯一约束错误会直接冒泡。

### 2.2 网站更新时写入共享短码（支持自定义 + 三分支）
`src/app/api/websites/[websiteId]/route.ts` 的 `shareId` 分三种语义：
- `shareId === null`：删除该网站关联 share（`deleteSharesByEntityId`）
- `shareId` 为非空字符串：创建新 share，`slug: shareId`
- `shareId === undefined`：保持现状，回读 `getShareByEntityId`

权限门槛：`canUpdateWebsite`。

冲突处理：显式捕获 `unique constraint` 并返回友好错误 `"That share ID is already taken."`。

### 2.3 网站专用共享接口创建（固定随机）
`src/app/api/websites/[websiteId]/shares/route.ts` 的 POST 逻辑固定 `getRandomChars(16)`：
- 请求 schema 不含 `slug`
- 该路径不支持自定义 slug

权限门槛：`canUpdateWebsite`。

### 2.4 通用共享创建接口（可自定义，缺省回退随机）
`src/app/api/share/route.ts` 的 `slug` 为可选：
- 传入 `slug`：按传入值写入
- 未传 `slug`：`slug || getRandomChars(16)`

权限门槛：`canUpdateEntity`。

### 2.5 统一口径矩阵（避免片面结论）

| 分支 | API | slug 策略 | 权限门槛 | 冲突处理 |
| --- | --- | --- | --- | --- |
| 网站创建 | `POST /api/websites` | `shareId` 自定义（可选） | `canCreateWebsite` / `canCreateTeamWebsite` | 无显式捕获 |
| 网站更新 | `POST /api/websites/[websiteId]` | `shareId` 自定义 / 删除 / 保持 | `canUpdateWebsite` | 有显式捕获并返回友好错误 |
| 网站专用共享 | `POST /api/websites/[websiteId]/shares` | 固定随机 16 位 | `canUpdateWebsite` | 无显式捕获 |
| 通用共享 | `POST /api/share` | 自定义或随机回退 | `canUpdateEntity` | 无显式捕获 |

结论：只有“网站专用共享接口”是固定随机；网站创建、网站更新、通用共享接口都存在自定义 slug 路径，因此“网站共享只能随机 slug”是片面结论。

## 3. 访客访问时的权限判定路径（含 board 展开）

### 3.1 共享身份建立
前端通过 `useShareTokenQuery` 获取 `/api/share/[slug]` 响应，把 token 存到全局状态；后续在 `/share/*` 路径下由 `useApi` 自动追加：
- `x-umami-share-token`
- `x-umami-share-context: 1`

后端 `checkAuth`（`src/lib/auth.ts`）会：
- 校验普通登录 token（若有）
- 解析 share token
- 对“无用户但有 share token”的请求，强制要求 `SHARE_CONTEXT_HEADER` 存在，否则拒绝

### 3.2 board share 的 ID 展开
`src/app/api/share/[slug]/route.ts` 在 `shareType === board` 时，会从 board 定义中展开实体集合并写入 token payload：
- `websiteIds`
- `pixelIds`
- `linkIds`

这意味着 board 共享不是单 ID 判定，而是集合判定。

### 3.3 canViewWebsite 如何命中这些展开 ID
`src/permissions/website.ts` 的 `canViewWebsite` 判定顺序为：
1. 管理员放行
2. share token 命中（`websiteId/pixelId/linkId` 或 `websiteIds/pixelIds/linkIds` 包含）
3. 普通用户所有权或团队成员关系判定

因此，board 场景下由展开数组命中可读权限，是共享访客访问网站类数据的关键路径。

## 4. 团队成员权限与共享访客权限边界

- 团队成员：基于账号与团队角色，可进入“读 + 管理”权限体系（能力取决于角色）。
- 共享访客：基于 share token 的只读通道，不具备资源管理能力。
- 两者都会经过后端权限函数；差异在身份来源与可执行操作集合，而非“是否走权限闸门”。

## 5. 后端权限放行 vs 前端 parameters 导航限制

### 5.1 前端层（可见性/导航限制）
`src/app/share/[slug]/[[...path]]/SharePage.tsx` 与 `src/app/share/ShareProvider.tsx` 使用 `parameters` 控制：
- 导航菜单展示
- 页面路由重定向

这层主要作用是“共享页体验与入口约束”。

### 5.2 后端层（真正的数据放行）
网站数据 API（如 `src/app/api/websites/[websiteId]/stats/route.ts`）本质依赖 `canViewWebsite`。
只要后端判定“对该 websiteId 可读”，就会返回该接口定义的数据口径；并不会逐项读取 `parameters` 做字段级裁剪。

### 5.3 返回口径结论
- 共享访客的数据口径由“后端可读权限 + 查询条件”决定。
- `parameters` 主要影响共享页导航与可见入口，不是后端数据裁剪规则本身。
- 同一资源、同一查询条件下，共享访客与具备读权限的成员通常得到同口径结果；同时仍受全局日期窗等通用约束（如 `setWebsiteDate`）。

## 6. 总结

- Share 表持久化的是共享配置，不含 token。
- token 是访问 `/api/share/[slug]` 时动态签发的会话凭据。
- slug 来源存在明确分支差异：网站创建/更新与通用共享支持自定义，网站专用共享接口固定随机。
- board share 通过 `websiteIds/pixelIds/linkIds` 展开参与 `canViewWebsite` 判定。
- 前端 `parameters` 是导航限制，后端 `canViewWebsite` 才是数据放行边界。
