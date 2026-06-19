# Umami Tracker 多域名上报与 Session 合并机制分析

## 概述

Umami 的多域名上报与 Session 合并涉及五层关键机制：域名规范化、跨子域 Session 复用、无 Cookie 存储策略、多入口采集、以及统计层面的合并边界。本文档从源码层面梳理完整脉络。

---

## 1. 域名规范化逻辑 (Domain Normalization)

### 核心规则：去除 `www.` 前缀

**代码位置**：
- 采集入口：[send/route.ts#L184](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L184-L184)
- 统计查询：[prisma.ts#L134](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/prisma.ts#L134-L136)

```typescript
// 采集时规范化
const urlDomain = currentUrl.hostname.replace(/^www./, '');

// 查询时规范化（用于排除内部来源）
referrer_domain != regexp_replace(hostname, '^www.', '')
```

### 规范化应用场景

| 场景 | 原始值 | 规范化后 | 代码位置 |
|------|--------|----------|----------|
| 页面域名 | `www.example.com` | `example.com` | [send/route.ts#L184](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L184-L184) |
| 来源域名 | `www.google.com` | `google.com` | [send/route.ts#L214](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L214-L214) |
| 自引用判断 | `www.example.com` | `example.com` | [prisma.ts#L134](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/prisma.ts#L134-L136) |

### 数据库存储

- `website_event.hostname` 字段存储**原始 hostname**（带 www）
- `website_event.referrer_domain` 字段存储**规范化后的来源域名**（不带 www）

---

## 2. 跨子域复用机制 (Cross-Subdomain Session Reuse)

### SessionID 生成算法（关键！域名不参与计算）

**代码位置**：[send/route.ts#L147](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L147-L147)

```typescript
// 两种生成模式
const sessionId = id 
  ? uuid(sourceId, id)                              // 有自定义标识时
  : uuid(sourceId, ip, userAgent, sessionSalt);     // 无自定义标识时
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
- ✅ 同一 `websiteId`（同一个网站配置）
- ✅ 同一 `ip`（同一网络出口）
- ✅ 同一 `userAgent`（同一浏览器）
- ✅ 同一 `sessionSalt` 周期（默认按月轮换）
- ❌ **域名不参与计算** → 跨子域自动复用

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

---

## 3. Cookie 模式与存储策略

### Umami 不使用 Cookie！

**代码验证**：全局搜索 `document.cookie` 或 `Set-Cookie` 无结果。

### 存储架构

| 存储位置 | 用途 | 代码位置 |
|----------|------|----------|
| **内存变量 `cache`** | 存储 JWT token（含 sessionId、visitId、iat） | [index.js#L400](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js#L400-L400) |
| **`x-umami-cache` Header** | 请求时传递 session 信息 | [index.js#L181](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js#L181-L182) |
| **`localStorage`** | 仅用于检查 `umami.disabled` 禁用标志 | [index.js#L159](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js#L159-L159) |

### Cache Token 结构

**代码位置**：[send/route.ts#L17-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L17-L22)

```typescript
interface Cache {
  websiteId: string;   // 网站ID
  sessionId: string;   // 会话ID
  visitId: string;     // 访问ID（30分钟过期）
  iat: number;         // 签发时间戳
}
```

**Token 流转**：
1. 首次请求 → 无 cache → 后端生成 sessionId、visitId
2. 后端返回 JWT token：`createToken({ websiteId, sessionId, visitId, iat }, secret())`
3. 前端内存保存 `cache` 变量
4. 后续请求在 `x-umami-cache` header 中携带 token
5. 后端解析 token 复用 sessionId、visitId

### Visit 过期机制

**代码位置**：[send/route.ts#L171-L175](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L171-L175)

```typescript
// Expire visit after 30 minutes
if (!timestamp && now - iat > 1800) {
  visitId = uuid(sessionId, visitSalt);  // 生成新 visitId
  iat = now;                             // 重置时间
}
```

- `visitId` 每 30 分钟过期（1800秒）
- `sessionId` 保持不变 → **Visitor 计数不变，Visits 计数+1**

### fetch-credentials 配置

**代码位置**：[index.js#L38](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js#L38-L38)

```javascript
const credentials = config('fetch-credentials') || 'omit';
```

- 仅控制 fetch 请求的跨域凭证模式（`omit` / `include` / `same-origin`）
- 不涉及任何 Session 存储逻辑

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
  (domain && !domains.includes(hostname)) ||  // 域名白名单检查
  (dnt && hasDoNotTrack());
```

**自动追踪功能**：
- 页面浏览（PageView）：`track()`
- SPA 路由变化：hook `history.pushState` / `replaceState`
- 点击事件：监听 `document.click`，识别 `data-umami-event` 属性
- 性能指标：Web Vitals（LCP、INP、CLS、FCP、TTFB）

**手动 API**：
- `umami.track(eventName?, data?)` - 追踪事件
- `umami.identify(id, data?)` - 设置用户标识
- `umami.getSession()` - 获取当前 session 信息

### 4.2 后端 API 入口

| 入口 | 方法 | 用途 | 代码位置 |
|------|------|------|----------|
| `/api/send` | `POST` | 主采集入口（event/identify/performance） | [send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts) |
| `/api/batch` | `POST` | 批量采集（内部循环调用 send） | [batch/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/batch/route.ts) |
| `/api/record` | `POST` | 会话录制（需 cache token） | [record/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/record/route.ts) |
| `/p/[slug]` | `GET` | 像素追踪（返回 1x1 GIF） | [p/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/(collect)/p/[slug]/route.ts) |
| `/q/[slug]` | `GET` | 链接追踪（重定向 + 上报） | [q/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/(collect)/q/[slug]/route.ts) |

### 4.3 主采集入口处理流程

**代码位置**：[send/route.ts#L66-L322](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts#L66-L322)

```
1. 解析请求 → Zod schema 验证
2. Cache 检查 → 解析 x-umami-cache header（JWT）
3. 客户端信息 → IP、UA、设备、地理位置
4. Bot/IP 拦截 → isbot 检查 + IP 黑名单
5. SessionID 生成 → uuid(sourceId, ip, UA, salt)
6. VisitID 处理 → 30分钟过期检查
7. 事件存储 → saveEvent() / saveSessionData()
8. 返回 Cache Token → 供下次请求复用
```

---

## 5. 统计合并边界 (Session Merge Boundaries)

### 5.1 合并判定核心：SessionID 唯一性

所有统计查询都基于 `session_id` 进行去重计数：

```sql
-- 典型统计查询模式
count(distinct website_event.session_id) as "visitors"
```

**代码证据**（63+ 处查询）：
- [getWebsiteSessionStats.ts#L40](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/sessions/getWebsiteSessionStats.ts#L40-L40)
- [getPageviewMetrics.ts#L76](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/pageviews/getPageviewMetrics.ts#L76-L76)
- [getAttribution.ts#L55](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/reports/getAttribution.ts#L55-L55)

### 5.2 合并条件（同一 Session）

✅ **合并** → 相同 `sessionId`

| 条件 | 要求 |
|------|------|
| websiteId | 必须相同（同一个网站配置） |
| ip | 必须相同（同一网络出口） |
| userAgent | 必须相同（同一浏览器/设备） |
| sessionSalt | 必须相同（同一 salt 周期） |
| hostname | **不要求**（跨子域自动合并） |

### 5.3 分割条件（不同 Session）

❌ **不合并** → 不同 `sessionId`

| 场景 | 原因 | 影响 |
|------|------|------|
| 不同 websiteId | 即使同域名，不同网站配置也分开 | Visitors ×2 |
| IP 变化 | 切换 Wi-Fi/移动网络 | Visitors ×2 |
| UA 变化 | 切换浏览器/设备 | Visitors ×2 |
| Salt 周期变化 | 跨月（默认） | Visitors ×2 |
| 自定义 id 变化 | 调用 `umami.identify(newId)` | 新 sessionId |

### 5.4 Hostname 在统计中的作用

`hostname` 字段**不影响** session 合并，仅用于：

1. **维度拆分**：按 hostname 统计各子域名流量
   ```sql
   select hostname, count(distinct session_id) 
   from website_event 
   group by hostname
   ```

2. **自引用排除**：排除站内跳转（来源域名 = 当前域名）
   ```sql
   and referrer_domain != regexp_replace(hostname, '^www.', '')
   ```
   [prisma.ts#L134](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/prisma.ts#L134-L136)

### 5.5 多域名合并示例

假设配置 `websiteId = xxx`，绑定域名 `example.com`（未配置子域名白名单）：

| 访问序列 | Hostname | sessionId | Visitor 计数 | Visit 计数 |
|----------|----------|-----------|-------------|------------|
| 1. `www.example.com` | `www.example.com` | `sid-123` | +1 | +1 |
| 2. `app.example.com` | `app.example.com` | `sid-123` | +0（合并） | +0（30分钟内） |
| 3. `admin.example.com` | `admin.example.com` | `sid-123` | +0（合并） | +0（30分钟内） |
| 4. 30分钟后 `app.example.com` | `app.example.com` | `sid-123` | +0（合并） | +1（visit 过期） |
| 5. 次月 `www.example.com` | `www.example.com` | `sid-456`（新 salt） | +1（新月） | +1 |

---

## 6. 完整数据流转图

```
浏览器（app.example.com）
    ↓
[Tracker] index.js
    ├─ 读取 data-* 属性（website-id, domains 等）
    ├─ 检查域名白名单：domains.includes(hostname)
    ├─ 构造 payload（含 hostname）
    ├─ 内存读取 cache（JWT token）
    └─ POST /api/send
        ├─ Header: x-umami-cache: <JWT token>
        └─ Body: { type, payload: { website, hostname, url, ... } }
            ↓
[Backend] send/route.ts
    ├─ 1. 解析 JWT cache → { sessionId, visitId, iat }
    ├─ 2. 生成 sessionSalt（按月）
    ├─ 3. 计算 sessionId = uuid(websiteId, ip, UA, salt)
    │   （域名不参与！）
    ├─ 4. 检查 visit 过期（30分钟）→ 生成新 visitId
    ├─ 5. 规范化域名：hostname.replace(/^www./, '')
    ├─ 6. 存储事件：saveEvent()
    │   ├─ session_id: sessionId
    │   ├─ visit_id: visitId
    │   ├─ hostname: 原始 hostname
    │   └─ referrer_domain: 规范化后域名
    └─ 7. 返回新 JWT cache → 前端内存保存
        ↓
[Statistics] 查询时
    ├─ count(distinct session_id) → 合并跨子域访问
    ├─ group by hostname → 按子域名拆分统计
    └─ 排除自引用：referrer_domain != regexp_replace(hostname, '^www.', '')
```

---

## 7. 关键配置项

| 配置项 | 位置 | 默认值 | 作用 |
|--------|------|--------|------|
| `data-domains` | Tracker script 属性 | 空 | 域名白名单，逗号分隔 |
| `data-fetch-credentials` | Tracker script 属性 | `omit` | fetch 跨域凭证模式 |
| `SALT_ROTATION` | 环境变量 | `month` | Session salt 轮换周期 |
| `REMOVE_TRAILING_SLASH` | 环境变量 | `false` | 是否移除 URL 尾部斜杠 |
| `DISABLE_BOT_CHECK` | 环境变量 | `false` | 是否禁用 Bot 检测 |

---

## 8. 核心文件速查表

| 功能模块 | 文件路径 |
|----------|----------|
| 前端 Tracker | [src/tracker/index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/tracker/index.js) |
| 主采集入口 | [src/app/api/send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/app/api/send/route.ts) |
| SessionID 生成 | [src/lib/crypto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/crypto.ts) |
| JWT Cache Token | [src/lib/jwt.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/jwt.ts) |
| 事件存储 | [src/queries/sql/events/saveEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/events/saveEvent.ts) |
| 统计查询 | [src/queries/sql/sessions/](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/queries/sql/sessions/) |
| 数据库 Schema | [prisma/schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/prisma/schema.prisma) |
| 常量定义 | [src/lib/constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/55-umami/src/lib/constants.ts) |

---

## 9. 常见疑问解答

**Q: 为什么跨子域访问会被合并为同一个 Visitor？**
A: 因为 `sessionId` 基于 `(websiteId, ip, userAgent, salt)` 生成，**域名不参与计算**。同一浏览器访问同一 websiteId 下的不同子域名，会生成相同的 sessionId。

**Q: 如何让不同子域名分开统计？**
A: 为每个子域名创建独立的 website 配置（不同 websiteId），这样 sessionId 生成的第一个因子就不同了。

**Q: Umami 支持第三方 Cookie 吗？**
A: 不支持。Umami 完全不使用 Cookie，session 信息通过 JWT token 在请求头中传递，客户端仅在内存中保存。

**Q: 为什么配置了 `data-domains` 但某些子域名没有数据？**
A: 检查 `data-domains` 的值是否包含完整的 hostname（如 `app.example.com` 而不是 `example.com`）。前端 Tracker 是精确匹配 `hostname`，不做子域名通配。

**Q: Session 会在什么时候分割？**
A: 五种情况：IP 变化、UA 变化、websiteId 变化、salt 周期轮换（默认跨月）、调用 `umami.identify(newId)` 传入新标识。
