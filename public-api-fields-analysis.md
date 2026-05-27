# Umami 对外开放 API 字段分析

> 版本：v3.1.0 | 分析日期：2026-05-27

## 一、鉴权识别机制

### 1.1 双轨鉴权体系

Umami 的 API 层采用 **双轨鉴权** 设计，`checkAuth()` 函数同时处理两种凭证：

| 凭证类型 | Header | 加密方式 | 典型场景 |
|---------|--------|---------|---------|
| **Bearer Token** | `Authorization: Bearer <token>` | JWT → AES 加密（`parseSecureToken`） | 已登录用户、管理后台 |
| **Share Token** | `x-umami-share-token: <token>` | 纯 JWT（`parseToken`） | 分享链接、公开看板 |

核心判定逻辑位于 `src/lib/auth.ts:17-60`：

```
checkAuth(request):
  ├─ 解析 Bearer Token → parseSecureToken() → 获取 userId/authKey
  ├─ 解析 Share Token  → parseToken()       → 获取 shareToken 对象
  ├─ 若 userId 存在 → 查数据库获取 user
  ├─ 若 authKey 存在且 Redis 开启 → 从 Redis 取 user
  ├─ 若 user 和 shareToken 均为空 → 未授权 (null)
  └─ 若仅有 shareToken → 必须同时携带 x-umami-share-context: '1'
```

关键约束：**Share Token 不能在 share context 之外使用**（`src/lib/auth.ts:42-48`）。这意味着外部调用方必须在请求头中同时设置：

```
x-umami-share-token: <share_jwt>
x-umami-share-context: 1
```

### 1.2 Share Token 的签发流程

**签发入口**：`GET /api/share/[slug]`（`src/app/api/share/[slug]/route.ts:47-106`）

**完整签发流程**：

```
GET /api/share/[slug]
  │
  ├─ 1. 根据 slug 查询 Share 记录 (getShareByCode)
  │   ├─ Share 表字段：id, entityId, name, shareType, slug, parameters, createdAt
  │   └─ 若无记录 → 404
  │
  ├─ 2. 初始化基础 payload
  │   ├─ shareId = share.id
  │   ├─ shareType = share.shareType
  │   └─ parameters = share.parameters (section 白名单等)
  │
  ├─ 3. 根据 shareType 查询对应实体并扩展 payload
  │   ├─ shareType = board (4):
  │   │   ├─ 查询 Board (entityId)
  │   │   ├─ boardId = share.entityId
  │   │   ├─ 解析 boardEntityIds → websiteIds[], pixelIds[], linkIds[]
  │   │   └─ entity = board
  │   ├─ shareType = website (1):
  │   │   ├─ 查询 Website (entityId)
  │   │   ├─ websiteId = share.entityId
  │   │   └─ entity = website
  │   ├─ shareType = pixel (3):
  │   │   ├─ 查询 Pixel (entityId)
  │   │   ├─ websiteId = share.entityId  ⚠️ pixelId 同时赋值给 websiteId
  │   │   ├─ pixelId = share.entityId
  │   │   └─ entity = pixel
  │   └─ shareType = link (2):
  │       ├─ 查询 Link (entityId)
  │       ├─ websiteId = share.entityId   ⚠️ linkId 同时赋值给 websiteId
  │       ├─ linkId = share.entityId
  │       └─ entity = link
  │
  ├─ 4. 签发 JWT
  │   └─ token = createToken(data, secret())  ⚠️ 纯 JWT，无 AES 加密
  │
  └─ 5. 附加 whiteLabel（仅 Redis 模式）
      ├─ 从 entity 获取 userId/teamId
      ├─ 若 teamId → 查询 teamOwner 获取 accountId
      └─ 若 accountId 存在 → 从 Redis 取 white-label:${accountId}
```

**Share Token payload 字段清单**：

| 字段 | 类型 | 说明 | 适用 shareType |
|------|------|------|---------------|
| `shareId` | string | 分享记录 UUID | 全部 |
| `shareType` | number | 实体类型：1=website, 2=link, 3=pixel, 4=board | 全部 |
| `parameters` | object | 分享参数，包含可见 section 白名单 | 全部 |
| `websiteId` | string | 关联 ID（⚠️ pixel/link 场景下实际是 pixelId/linkId） | website, pixel, link |
| `pixelId` | string | 像素 ID | pixel |
| `linkId` | string | 链接 ID | link |
| `boardId` | string | 看板 ID | board |
| `websiteIds` | string[] | 关联网站 ID 列表 | board |
| `pixelIds` | string[] | 关联像素 ID 列表 | board |
| `linkIds` | string[] | 关联链接 ID 列表 | board |
| `token` | string | 签发的 JWT Token 字符串 | 全部 |
| `whiteLabel` | object | 白标配置（仅 Redis 开启且有配置时） | 全部（可选） |

### 1.3 whiteLabel 字段边界

**类型定义**（`src/lib/types.ts:190-194`）：

```typescript
export interface WhiteLabel {
  displayName: string;
  domainName: string;
  logoUrl: string;
}
```

**边界说明**：
- **仅在 Redis 模式下可用**：`getWhiteLabel()` 函数在 `redis.enabled === false` 时直接返回 `null`
- **基于账号维度**：通过 `userId` 或 `teamOwner.userId` 从 Redis 键 `white-label:${accountId}` 读取
- **仅附加在签发响应中**：whiteLabel 是 `/api/share/[slug]` 接口的响应字段，**不是 JWT payload 的一部分**，不会在后续数据请求中传递
- **前端展示用**：用于自定义分享页面的品牌展示，不影响 API 数据权限

### 1.4 pixel/link 场景下 websiteId 的语义

**关键发现**：在 pixel 和 link 分享场景中，`websiteId` 字段被**重载**，实际存储的是 pixelId 或 linkId：

```typescript
// 代码位置：src/app/api/share/[slug]/route.ts:80-89
} else if (share.shareType === ENTITY_TYPE.pixel) {
  entity = await getPixel(share.entityId);
  data.websiteId = share.entityId;  // ⚠️ 实际是 pixelId
  data.pixelId = share.entityId;
} else if (share.shareType === ENTITY_TYPE.link) {
  entity = await getLink(share.entityId);
  data.websiteId = share.entityId;  // ⚠️ 实际是 linkId
  data.linkId = share.entityId;
}
```

**权限匹配逻辑**（`src/permissions/website.ts:12-19`）：

```typescript
if (
  shareToken?.websiteId === websiteId ||      // 匹配 website 或重载的 pixelId/linkId
  shareToken?.pixelId === websiteId ||        // 精确匹配 pixelId
  shareToken?.linkId === websiteId ||         // 精确匹配 linkId
  shareToken?.websiteIds?.includes(websiteId) ||  // board 场景批量匹配
  shareToken?.pixelIds?.includes(websiteId) ||
  shareToken?.linkIds?.includes(websiteId)
) {
  return true;
}
```

**语义澄清**：
- API 路径中的 `[websiteId]` 参数实际上是 **entityId**，可以是 websiteId、pixelId 或 linkId
- `canViewWebsite()` 函数名有误导性，实际是 **`canViewEntity()`**，支持所有实体类型的权限校验
- pixel/link 分享时，同时设置 `websiteId`（重载）和 `pixelId`/`linkId`（精确）是为了兼容路径参数命名

### 1.5 鉴权上下文传递

客户端通过 `useApi()` hook 自动注入 share headers（`src/components/hooks/useApi.ts:22-28`）：

```typescript
const shareHeaders =
  isSharePath && shareToken?.token
    ? { [SHARE_TOKEN_HEADER]: shareToken.token, [SHARE_CONTEXT_HEADER]: '1' }
    : {};
```

服务端在 `parseRequest()` 中调用 `checkAuth()`，返回 `auth` 对象传递给各权限检查函数。

---

## 二、权限检查与可见性控制

### 2.1 视图权限层级

所有数据端点在返回数据前，都会经过 `canViewWebsite()` / `canViewBoard()` / `canViewPixel()` / `canViewLink()` 权限检查。

以 `canViewWebsite()` 为例（`src/permissions/website.ts:5-40`）：

```
canViewWebsite(auth, websiteId):
  ├─ user.isAdmin → true（管理员直通）
  ├─ shareToken 匹配任一：
  │   ├─ shareToken.websiteId === websiteId    (website 或 重载的 pixelId/linkId)
  │   ├─ shareToken.pixelId === websiteId      (pixel 精确匹配)
  │   ├─ shareToken.linkId === websiteId       (link 精确匹配)
  │   ├─ shareToken.websiteIds.includes(websiteId)
  │   ├─ shareToken.pixelIds.includes(websiteId)
  │   └─ shareToken.linkIds.includes(websiteId)
  │   → true（share token 授权）
  ├─ 无 user → false
  ├─ entity.userId === user.id → true（所有者）
  ├─ entity.teamId 存在且用户为团队成员 → true
  └─ 其他 → false
```

### 2.2 操作权限控制

- **查看（View）**：Share Token 仅拥有查看权限
- **更新（Update）**：需要 user 身份，team 场景需要 `website:update` 权限
- **删除（Delete）**：需要 user 身份，team 场景需要 `website:delete` 权限
- Share Token **无法**执行任何写操作

### 2.3 Share 参数中的 Section 白名单

Share Token 的 `parameters` 字段控制可见的 UI section（`src/app/share/ShareProvider.tsx:61-64`）：

```typescript
const allowedSections =
  isWebsiteShare && share?.parameters
    ? ALL_SECTION_IDS.filter(id => share.parameters[id] === true)
    : [];
```

**完整 Section ID 列表**：

| Section ID | 对应数据 | API 端点 |
|-----------|---------|---------|
| `overview` | 概览统计 | `/websites/[id]/stats` |
| `events` | 事件列表 | `/websites/[id]/events` |
| `sessions` | 会话列表 | `/websites/[id]/sessions` |
| `realtime` | 实时数据 | `/realtime/[id]` |
| `performance` | 性能指标 | `/reports/performance` |
| `compare` | 对比分析 | `/websites/[id]/pageviews` (带 compare) |
| `breakdown` | 细分分析 | `/reports/breakdown` |
| `goals` | 目标转化 | `/reports/goal` |
| `funnels` | 漏斗分析 | `/reports/funnel` |
| `journeys` | 路径分析 | `/reports/journey` |
| `retention` | 留存分析 | `/reports/retention` |
| `utm` | UTM 分析 | `/reports/utm` |
| `revenue` | 收入分析 | `/reports/revenue` |
| `attribution` | 归因分析 | `/reports/attribution` |

> ⚠️ **重要**：`parameters` 中某个 section 设为 `false` 或不存在时，前端会隐藏该 section 的 UI，但 **后端 API 不会因为 `parameters` 字段而拒绝数据请求**。字段裁剪依赖于 section 是否在分享链接的 UI 中被隐藏，而实际 API 调用只要通过了 `canViewWebsite()` 权限检查即可正常返回数据。

---

## 三、字段白名单与映射

### 3.1 核心字段映射表 `FILTER_COLUMNS`

位于 `src/lib/constants.ts:72-97`，定义了前端展示字段名到数据库列名的映射：

| 前端字段名 (key) | 数据库列名 (value) | 说明 |
|-----------------|-------------------|------|
| `path` | `url_path` | 页面路径 |
| `entry` | `url_path` | 进入页面路径 |
| `exit` | `url_path` | 退出页面路径 |
| `referrer` | `referrer_domain` | 来源域名 |
| `domain` | `referrer_domain` | 域名（同 referrer） |
| `hostname` | `hostname` | 主机名 |
| `distinctId` | `distinct_id` | 唯一访客标识 |
| `title` | `page_title` | 页面标题 |
| `query` | `url_query` | URL 查询参数 |
| `os` | `os` | 操作系统 |
| `browser` | `browser` | 浏览器 |
| `device` | `device` | 设备类型 |
| `country` | `country` | 国家 |
| `region` | `region` | 地区 |
| `city` | `city` | 城市 |
| `language` | `language` | 语言 |
| `event` | `event_name` | 事件名称 |
| `tag` | `tag` | 标签 |
| `eventType` | `event_type` | 事件类型编号 |
| `utmSource` | `utm_source` | UTM 来源 |
| `utmMedium` | `utm_medium` | UTM 媒介 |
| `utmCampaign` | `utm_campaign` | UTM 活动 |
| `utmContent` | `utm_content` | UTM 内容 |
| `utmTerm` | `utm_term` | UTM 术语 |

### 3.2 事件维度白名单 `EVENT_COLUMNS`

```
EVENT_COLUMNS = [
  'path', 'entry', 'exit', 'referrer', 'domain', 'title', 'query',
  'event', 'tag', 'hostname', 'utmSource', 'utmMedium', 'utmCampaign',
  'utmContent', 'utmTerm'
]
```

这些是可用于 **按事件维度分组统计** 的字段。在 `getPageviewMetrics()` 和 `getEventMetrics()` 中，`type` 参数必须是 `EVENT_COLUMNS` 或 `SESSION_COLUMNS` 中的成员。

### 3.3 会话维度白名单 `SESSION_COLUMNS`

```
SESSION_COLUMNS = [
  'browser', 'os', 'device', 'screen', 'language',
  'country', 'city', 'region', 'distinctId'
]
```

### 3.4 请求过滤器白名单 `getRequestFilters()`

位于 `src/lib/request.ts:79-90`，该函数从所有请求 query 参数中筛选出能在 `FILTER_COLUMNS` 中找到对应 key 的参数：

```typescript
export function getRequestFilters(query: Record<string, any>) {
  const result: Record<string, any> = {};
  for (const key of Object.keys(query)) {
    const baseName = key.replace(/\d+$/, '');
    if (baseName in FILTER_COLUMNS) {
      result[key] = query[key];
    }
  }
  return result;
}
```

这意味着 **只有 `FILTER_COLUMNS` 中定义的字段名（加数字后缀如 `browser1`、`os2`）才能作为过滤器参数**，其他字段会被静默丢弃。

---

## 四、各 API 端点返回字段

### 4.1 `/api/websites/[websiteId]/stats` — 网站概览统计

**数据结构**（`getWebsiteStats` → `WebsiteStatsData`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `pageviews` | number | 页面浏览总数 |
| `visitors` | number | 独立访客数 |
| `visits` | number | 访问次数 |
| `bounces` | number | 跳出数 |
| `totaltime` | number | 总停留时间（毫秒） |
| `comparison` | object | 对比周期的同名统计（可选） |

### 4.2 `/api/websites/[websiteId]/pageviews` — 页面浏览时序

**数据结构**（`getPageviewStats` + `getSessionStats`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `pageviews` | `{ x: string, y: number }[]` | 按时间单位的页面浏览量序列 |
| `sessions` | `{ x: string, y: number }[]` | 按时间单位的访客数序列 |
| `startDate` | string | 起始日期 |
| `endDate` | string | 结束日期 |
| `compare` | object | 对比周期数据（可选） |

### 4.3 `/api/websites/[websiteId]/metrics` — 维度指标

根据 `type` 参数路由到不同查询：

#### type ∈ SESSION_COLUMNS → `getSessionMetrics`

| 字段 | 类型 | 说明 |
|------|------|------|
| `x` | string | 维度值（如 "Chrome"、"Windows"） |
| `y` | number | 独立访客数（count distinct session_id） |
| `country` | string | *仅 city/region 类型额外返回* |

#### type ∈ EVENT_COLUMNS → `getPageviewMetrics`

| 字段 | 类型 | 说明 |
|------|------|------|
| `x` | string | 维度值（如 "/index.html"、"https://google.com"） |
| `y` | number | 独立访客数 |

#### type === `event` → `getEventMetrics`（自定义事件）

| 字段 | 类型 | 说明 |
|------|------|------|
| `x` | string | 事件名称 |
| `y` | number | 事件触发次数（count） |

#### type === `channel` → `getChannelMetrics`

| 字段 | 类型 | 说明 |
|------|------|------|
| `x` | string | 渠道分类（direct / paidAds / referral / organicSearch / paidSocial / email 等） |
| `y` | number | 该渠道的独立访客数 |

### 4.4 `/api/websites/[websiteId]/events` — 事件列表

**数据结构**（`getWebsiteEvents`，SQL 硬编码 19 个字段）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 事件 ID (event_id) |
| `websiteId` | string | 网站 ID |
| `sessionId` | string | 会话 ID |
| `createdAt` | string | 事件创建时间 |
| `hostname` | string | 主机名 |
| `urlPath` | string | URL 路径 |
| `urlQuery` | string | URL 查询参数 |
| `referrerPath` | string | 来源路径 |
| `referrerQuery` | string | 来源查询参数 |
| `referrerDomain` | string | 来源域名 |
| `country` | string | 国家 |
| `city` | string | 城市 |
| `device` | string | 设备类型 |
| `os` | string | 操作系统 |
| `browser` | string | 浏览器 |
| `pageTitle` | string | 页面标题 |
| `eventType` | number | 事件类型（1=pageview, 2=custom, 3=link, 4=pixel, 5=performance） |
| `eventName` | string | 事件名称 |
| `hasData` | boolean | 是否有关联的 event_data 记录 |

> ⚠️ **注意**：事件列表不返回 `region` 和 `language` 字段，这两个字段仅在**会话详情**中返回。

返回分页结构：`{ data, count, page, pageSize }`

### 4.5 `/api/websites/[websiteId]/sessions` — 会话列表

**数据结构**（`getWebsiteSessions`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 会话 ID (session_id) |
| `websiteId` | string | 网站 ID |
| `hostname` | string | 主机名 |
| `browser` | string | 浏览器 |
| `os` | string | 操作系统 |
| `device` | string | 设备类型 |
| `screen` | string | 屏幕分辨率 |
| `language` | string | 语言 |
| `country` | string | 国家 |
| `region` | string | 地区 |
| `city` | string | 城市 |
| `firstAt` | string | 首次活动时间 |
| `lastAt` | string | 最后活动时间 |
| `visits` | number | 访问次数 |
| `views` | number | 页面浏览数 |
| `events` | number | 事件数 |
| `createdAt` | string | 创建时间 |

返回分页结构：`{ data, count, page, pageSize }`

### 4.6 `/api/websites/[websiteId]/sessions/[sessionId]` — 单个会话详情

**数据结构**（`getWebsiteSession`，SQL 硬编码 17 个字段）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 会话 ID |
| `websiteId` | string | 网站 ID |
| `distinctId` | string | 唯一访客标识（distinct_id） |
| `browser` | string | 浏览器 |
| `os` | string | 操作系统 |
| `device` | string | 设备类型 |
| `screen` | string | 屏幕分辨率 |
| `language` | string | 语言 |
| `country` | string | 国家 |
| `region` | string | 地区 |
| `city` | string | 城市 |
| `firstAt` | string | 首次活动时间 |
| `lastAt` | string | 最后活动时间 |
| `visits` | number | 访问次数 |
| `views` | number | 页面浏览数 |
| `events` | number | 事件数 |
| `totaltime` | number | 总停留时间（毫秒） |

### 4.7 `/api/websites/[websiteId]/sessions/[sessionId]/activity` — 会话活动轨迹

**数据结构**（`getSessionActivity`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `eventId` | string | 事件 ID |
| `eventType` | number | 事件类型 |
| `eventName` | string | 事件名称 |
| `urlPath` | string | URL 路径 |
| `urlQuery` | string | URL 查询参数 |
| `referrerDomain` | string | 来源域名 |
| `visitId` | string | 访问 ID |
| `hostname` | string | 主机名 |
| `createdAt` | string | 创建时间 |
| `hasData` | boolean | 是否有关联的 event_data 记录 |

### 4.8 `/api/websites/[websiteId]/sessions/[sessionId]/properties` — 单会话数据属性

**数据结构**（`getSessionData`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `websiteId` | string | 网站 ID |
| `sessionId` | string | 会话 ID |
| `dataKey` | string | 数据键名 |
| `dataType` | number | 数据类型（1=string, 2=number, 3=boolean, 4=date, 5=array） |
| `stringValue` | string | 字符串值 |
| `numberValue` | number | 数值 |
| `dateValue` | string | 日期值 |
| `createdAt` | string | 创建时间 |

### 4.9 `/api/websites/[websiteId]/session-data/properties` — 会话数据属性汇总

**数据结构**（`getSessionDataProperties`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `propertyName` | string | 数据键名 (data_key) |
| `total` | number | 具有该属性的独立会话数（count distinct session_id） |

> 限制 `LIMIT 500`

### 4.10 `/api/websites/[websiteId]/session-data/values` — 会话数据值

**数据结构**（`getSessionDataValues`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `value` | string | 处理后的值（数值去尾零、日期截断到小时） |
| `total` | number | 具有该值的独立会话数（count distinct session_id） |

> 限制 `LIMIT 100`

### 4.11 `/api/websites/[websiteId]/event-data` — 事件数据列表（聚合）

**数据结构**（`getEventData` 经 eventMap 转换后）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `websiteId` | string | 网站 ID |
| `eventId` | string | 事件 ID |
| `eventName` | string | 事件名称 |
| `eventProperties` | array | 事件属性数组，每个属性包含：dataKey, stringValue, numberValue, dateValue, dataType, createdAt |

返回分页结构：`{ data, count, page, pageSize }`

### 4.12 `/api/websites/[websiteId]/event-data/[eventId]` — 单个事件数据详情

**数据结构**（`getEventDataById`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `websiteId` | string | 网站 ID |
| `eventId` | string | 事件 ID |
| `eventName` | string | 事件名称 |
| `dataKey` | string | 数据键名 |
| `stringValue` | string | 字符串值 |
| `numberValue` | number | 数值 |
| `dateValue` | string | 日期值 |
| `dataType` | number | 数据类型 |
| `createdAt` | string | 创建时间 |

### 4.13 `/api/websites/[websiteId]/event-data/events` — 事件数据分组统计

**数据结构**（`getEventDataEvents`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `eventName` | string | 事件名称 |
| `propertyName` | string | 属性名 (data_key) |
| `dataType` | number | 数据类型 |
| `propertyValue` | string | 属性值（仅当指定 `event` 查询参数时返回） |
| `total` | number | 记录数 |

> 限制 `LIMIT 500`；指定 `event` 参数时按 eventName+propertyName+propertyValue 分组，否则按 eventName+propertyName 分组

### 4.14 `/api/websites/[websiteId]/event-data/stats` — 事件数据统计概览

**数据结构**（`getEventDataStats`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `events` | number | 有数据的事件数（count distinct event_id） |
| `properties` | number | 不同属性键数（count distinct data_key） |
| `records` | number | 总记录数（sum of counts） |

### 4.15 `/api/websites/[websiteId]/event-data/fields` — 事件数据字段

**数据结构**（`getEventDataFields`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `propertyName` | string | 数据键名 (data_key) |
| `dataType` | number | 数据类型（1=string, 2=number, 3=boolean, 4=date, 5=array） |
| `value` | string | 处理后的值（数值去尾零、日期截断到小时） |
| `total` | number | 该值出现次数 |

> 限制 `LIMIT 100`

### 4.16 `/api/websites/[websiteId]/event-data/properties` — 事件数据属性汇总

**数据结构**（`getEventDataProperties`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `eventName` | string | 事件名称 |
| `propertyName` | string | 数据键名 (data_key) |
| `total` | number | 出现次数 |

> 限制 `LIMIT 500`

### 4.17 `/api/websites/[websiteId]/event-data/values` — 事件数据值

**数据结构**（`getEventDataValues`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `value` | string | 处理后的值（数值去尾零、日期截断到小时） |
| `total` | number | 出现次数 |

> 限制 `LIMIT 100`

### 4.18 `/api/websites/[websiteId]/sessions/stats` — 会话统计

**数据结构**（`getWebsiteSessionStats`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `pageviews` | number | 页面浏览数 |
| `visitors` | number | 访客数 |
| `visits` | number | 访问数 |
| `bounces` | number | 跳出数 |
| `totaltime` | number | 总停留时间 |

### 4.19 `/api/websites/[websiteId]/events/stats` — 事件统计

**数据结构**（`getWebsiteEventStats`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `events` | number | 事件总数 |
| `visitors` | number | 触发事件的访客数 |

### 4.20 `/api/websites/[websiteId]/events/series` — 事件时序

**数据结构**（`getEventStats`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `x` | string | 时间点 |
| `y` | number | 事件数 |

### 4.21 `/api/websites/[websiteId]/metrics/expanded` — 扩展指标

返回合并的页面浏览+会话+事件扩展指标。

### 4.22 `/api/realtime/[websiteId]` — 实时数据

**数据结构**（`getRealtimeData`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `countries` | `{ [country]: number }` | 各国在线访客数 |
| `urls` | `{ [url]: number }` | 各页面当前访问数 |
| `referrers` | `{ [domain]: number }` | 各来源域名访问数 |
| `events` | array | 实时事件流（含 `__type` 标记：session/event/pageview） |
| `series.views` | array | 页面浏览时序 |
| `series.visitors` | array | 访客时序 |
| `totals.views` | number | 总浏览数 |
| `totals.visitors` | number | 总访客数 |
| `totals.events` | number | 总事件数 |
| `totals.countries` | number | 覆盖国家数 |
| `timestamp` | number | 时间戳 |

### 4.23 `/api/reports/breakdown` — 细分分析报告

**数据结构**（`getBreakdown`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `views` | number | 浏览数 |
| `visitors` | number | 访客数 |
| `visits` | number | 访问数 |
| `bounces` | number | 跳出数 |
| `totaltime` | number | 总停留时间 |
| 动态维度字段 | string | 由 `fields` 参数决定，如 `browser`、`os`、`country` 等 |

### 4.24 `/api/websites/[websiteId]/sessions/[sessionId]/replays` — 会话回放列表

**数据结构**（`getSessionReplays`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 回放 ID (visit_id) |
| `sessionId` | string | 会话 ID |
| `websiteId` | string | 网站 ID |
| `browser` | string | 浏览器 |
| `os` | string | 操作系统 |
| `device` | string | 设备类型 |
| `country` | string | 国家 |
| `city` | string | 城市 |
| `eventCount` | number | 事件总数 |
| `chunkCount` | number | 分块数 |
| `startedAt` | string | 开始时间 |
| `endedAt` | string | 结束时间 |
| `duration` | number | 持续时间（毫秒） |
| `createdAt` | string | 创建时间 |

返回分页结构：`{ data, count, page, pageSize }`

### 4.25 event-data 家族端点暴露链路

```
event-data 端点家族（共 7 个，全部通过 canViewWebsite 校验）
  │
  ├─ /event-data                → getEventData()         → 事件数据列表（按 eventId 聚合）
  ├─ /event-data/[eventId]      → getEventDataById()     → 单个事件的所有属性
  ├─ /event-data/events         → getEventDataEvents()   → 按事件+属性分组统计
  ├─ /event-data/stats          → getEventDataStats()    → 事件数据总体统计（events/properties/records）
  ├─ /event-data/fields         → getEventDataFields()   → 字段名+类型+值
  ├─ /event-data/properties     → getEventDataProperties() → 事件名+属性名+计数
  └─ /event-data/values         → getEventDataValues()   → 按属性值统计
```

**共享权限模式**：所有 7 个端点都遵循相同的鉴权链路：
```
parseRequest → checkAuth → canViewWebsite → getQueryFilters → SQL 查询
     ↓           ↓            ↓                  ↓              ↓
  解析token  双轨鉴权    仅实体级校验      仅过滤入参字段   硬编码SELECT字段
```

---

## 五、字段裁剪总结

### 5.1 三层过滤机制（无字段级裁剪）

```
请求层（Request Layer）
  │
  ├─ [第一层] getRequestFilters()
  │   仅保留 FILTER_COLUMNS 中定义的字段作为过滤条件
  │   任何未知 filter 字段被静默丢弃
  │   ⚠️ 仅作用于入参过滤，不影响返回字段
  │
  ├─ [第二层] checkAuth() → canViewWebsite()
  │   Share Token 只能访问其绑定的 websiteId(s)/pixelId(s)/linkId(s)
  │   访问其他实体返回 401
  │   ⚠️ 仅做实体级权限校验，不校验字段或 section
  │   ⚠️ websiteId 参数名重载，实际可匹配 websiteId/pixelId/linkId
  │
  └─ [第三层] SQL 查询层
      各查询函数硬编码 SELECT 字段，不做动态列选择
      数据库层没有额外的字段裁剪
      ⚠️ 所有通过权限校验的请求返回相同的完整字段
```

### 5.2 Share Token 字段裁剪现状

| 裁剪维度 | 是否实施 | 实施位置 |
|---------|---------|---------|
| **实体范围** | ✅ 是 | `canViewWebsite()` / `canViewBoard()` / `canViewPixel()` / `canViewLink()` |
| **操作类型** | ✅ 是 | Share Token 无写权限，所有 POST/PUT/DELETE 需 user 身份 |
| **Section 可见性** | ⚠️ 前端 | `ShareProvider` 根据 `parameters` 过滤 UI，但 API 层不校验 |
| **字段级裁剪** | ❌ 否 | 所有 API 端点返回完整字段，无基于 token 类型的字段子集 |
| **过滤器字段** | ✅ 是 | `getRequestFilters()` 仅接受 `FILTER_COLUMNS` 中的字段名 |

### 5.3 关键发现

1. **Share Token 无字段级裁剪**：
   - **事件列表** (`/api/websites/[id]/events`) 返回 19 个完整字段：`id`, `websiteId`, `sessionId`, `createdAt`, `hostname`, `urlPath`, `urlQuery`, `referrerPath`, `referrerQuery`, `referrerDomain`, `country`, `city`, `device`, `os`, `browser`, `pageTitle`, `eventType`, `eventName`, `hasData`
   - **会话详情** (`/api/websites/[id]/sessions/[sessionId]`) 返回 17 个完整字段：`id`, `websiteId`, `distinctId`, `browser`, `os`, `device`, `screen`, `language`, `country`, `region`, `city`, `firstAt`, `lastAt`, `visits`, `views`, `events`, `totaltime`
   - 所有字段对 Share Token 完全开放，与登录用户看到的字段完全一致

2. **Section 白名单仅控制前端**：`parameters` 中的 section 开关（如 `overview: true`, `events: false`）仅在 `ShareProvider` 中用于隐藏/展示前端 UI 组件，后端 API 不会因为 section 被禁用而拒绝请求。

3. **distinct_id 暴露**：
   - `SESSION_COLUMNS` 中包含 `distinctId`（映射到 `distinct_id`），该字段在 metrics 端点中可被查询
   - 单个会话详情接口 (`/sessions/[sessionId]`) 直接返回 `distinctId` 字段
   - 存在指纹追踪风险

4. **event_data 完整暴露（7 个端点）**：通过 Share Token 可访问以下端点，返回事件的自定义数据键值对：
   - `/event-data` - 事件数据列表（按 eventId 聚合）
   - `/event-data/[eventId]` - 单个事件的所有属性
   - `/event-data/events` - 按事件+属性分组统计
   - `/event-data/stats` - 事件数据总体统计
   - `/event-data/fields` - 事件数据字段（键名+类型+值）
   - `/event-data/values` - 事件数据值（值+计数）
   - `/event-data/properties` - 事件数据属性（事件名+键名+计数）

5. **session_data 完整暴露**：通过 Share Token 可访问以下端点：
   - `/sessions/[sessionId]/properties` - 单个会话的自定义属性键值对
   - `/session-data/properties` - 会话数据属性汇总（键名+独立会话数）
   - `/session-data/values` - 会话数据值（值+独立会话数）
   - `/sessions/[sessionId]/replays` - 会话回放元数据

6. **endpoint 命名容易混淆**：
   - `/sessions/[sessionId]/properties` 返回单个会话的属性键值（`getSessionData`）
   - `/session-data/properties` 返回所有会话的属性汇总统计（`getSessionDataProperties`）

7. **websiteId 参数语义重载**：
   - pixel 分享时：`websiteId = pixelId`（同时设置 `pixelId` 字段）
   - link 分享时：`websiteId = linkId`（同时设置 `linkId` 字段）
   - `canViewWebsite()` 实际是 `canViewEntity()`，支持所有实体类型

8. **whiteLabel 字段边界**：
   - 仅在 Redis 模式下可用，非 Redis 部署无此字段
   - 是 `/api/share/[slug]` 响应字段，**不是 JWT payload 的一部分**
   - 仅用于前端品牌展示，不影响 API 数据权限

---

## 六、代码索引

| 模块 | 文件路径 | 行号 |
|------|---------|------|
| 鉴权核心 | `src/lib/auth.ts` | 17-60 |
| Share Token 解析 | `src/lib/auth.ts` | 80-87 |
| 请求解析+鉴权 | `src/lib/request.ts` | 12-53 |
| 过滤器白名单 | `src/lib/request.ts` | 79-90 |
| 字段映射表 | `src/lib/constants.ts` | 72-97 |
| 事件维度白名单 | `src/lib/constants.ts` | 37-53 |
| 会话维度白名单 | `src/lib/constants.ts` | 55-65 |
| Share Token 签发 | `src/app/api/share/[slug]/route.ts` | 47-106 |
| Share 创建 | `src/app/api/share/route.ts` | 10-42 |
| Share CRUD | `src/app/api/share/id/[shareId]/route.ts` | 8-82 |
| Share 权限 | `src/permissions/website.ts` | 5-40 |
| Board 权限 | `src/permissions/board.ts` | 5-35 |
| Pixel 权限 | `src/permissions/pixel.ts` | 5-33 |
| 实体解析 | `src/lib/entity.ts` | 4-15 |
| WhiteLabel 类型 | `src/lib/types.ts` | 190-194 |
| 前端 API Hook | `src/components/hooks/useApi.ts` | 21-34 |
| Share Provider | `src/app/share/ShareProvider.tsx` | 61-64 |
| Share Store | `src/store/app.ts` | 35-40 |
| Prisma Share Model | `prisma/schema.prisma` | 348-360 |
| Prisma Pixel Model | `prisma/schema.prisma` | 309-324 |
| Prisma Link Model | `prisma/schema.prisma` | 288-303 |
| 概览统计查询 | `src/queries/sql/getWebsiteStats.ts` | 9-138 |
| 页面浏览指标 | `src/queries/sql/pageviews/getPageviewMetrics.ts` | 7-197 |
| 会话指标 | `src/queries/sql/sessions/getSessionMetrics.ts` | 7-137 |
| 事件指标 | `src/queries/sql/events/getEventMetrics.ts` | 7-97 |
| 渠道指标 | `src/queries/sql/getChannelMetrics.ts` | 14-149 |
| 事件列表 | `src/queries/sql/events/getWebsiteEvents.ts` | 6-119 |
| 会话列表 | `src/queries/sql/sessions/getWebsiteSessions.ts` | 7-159 |
| 会话详情 | `src/queries/sql/sessions/getWebsiteSession.ts` | 7-113 |
| 会话活动轨迹 | `src/queries/sql/sessions/getSessionActivity.ts` | 8-80 |
| 会话数据 | `src/queries/sql/sessions/getSessionData.ts` | 7-60 |
| 会话数据属性汇总 | `src/queries/sql/sessions/getSessionDataProperties.ts` | 6-75 |
| 会话数据值 | `src/queries/sql/sessions/getSessionDataValues.ts` | 6-85 |
| 事件数据列表 | `src/queries/sql/events/getEventData.ts` | 7-152 |
| 事件数据详情 | `src/queries/sql/events/getEventDataById.ts` | 8-63 |
| 事件数据分组统计 | `src/queries/sql/events/getEventDataEvents.ts` | 8-151 |
| 事件数据统计概览 | `src/queries/sql/events/getEventDataStats.ts` | 8-94 |
| 事件数据属性汇总 | `src/queries/sql/events/getEventDataProperties.ts` | 6-92 |
| 事件数据值 | `src/queries/sql/events/getEventDataValues.ts` | 6-96 |
| 会话回放列表 | `src/queries/sql/replays/getSessionReplays.ts` | 6-148 |
| 实时数据 | `src/queries/sql/getRealtimeData.ts` | 16-78 |
| 细分报告 | `src/queries/sql/reports/getBreakdown.ts` | 7-135 |
| 事件数据字段 | `src/queries/sql/events/getEventDataFields.ts` | 6-88 |
