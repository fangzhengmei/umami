# Umami 采集前置过滤代码分析

## 一、整体架构与入口

### 1.1 采集入口

Umami 有四个数据采集入口，均会经过前置过滤逻辑：

| 入口 | 方法 | 文件 | 说明 |
|------|------|------|------|
| `/api/send` | POST | [route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts) | 主采集接口（事件、identify、性能） |
| `/api/record` | POST | [route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/record/route.ts) | 录屏数据接口 |
| `/p/[slug]` | GET | [route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/(collect)/p/[slug]/route.ts) | 像素追踪，内部调用 `/api/send` |
| `/q/[slug]` | GET | [route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/(collect)/q/[slug]/route.ts) | 短链接追踪，内部调用 `/api/send` |

### 1.2 核心过滤模块

| 模块 | 文件 | 功能 |
|------|------|------|
| IP 处理 | [ip.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/ip.ts) | IP 地址解析、标准化、端口剥离 |
| 检测模块 | [detect.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts) | 客户端信息、地理定位、IP 黑名单、设备检测 |
| 加密/UUID | [crypto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/crypto.ts) | Session ID 生成、盐值轮转 |
| 请求解析 | [request.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/request.ts) | 请求体解析、Schema 验证 |

---

## 二、采集前置过滤完整流程

### 2.1 主流程时序图（以 `/api/send` 为例）

```
请求到达
   │
   ▼
┌─────────────────────┐
│ 1. 请求解析与验证    │  parseRequest() + Zod schema
│    - type 校验       │
│    - payload 校验    │
│    - website/link/pixel 三选一 │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 2. 缓存检查          │  x-umami-cache header (JWT)
│    - 解析 sessionId  │
│    - 解析 visitId    │
│    - 解析 iat        │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 3. 网站查找          │  fetchWebsite()
│    - 验证网站存在    │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 4. 客户端信息收集    │  getClientInfo()
│    ├─ IP 获取        │  getIpAddress()
│    ├─ UA 获取        │  user-agent header
│    ├─ 地理定位       │  getLocation()
│    │   ├─ CDN header 优先 │
│    │   └─ MaxMind 本地库 │
│    ├─ 浏览器检测     │  browserName()
│    ├─ OS 检测        │  detectOS()
│    └─ 设备检测       │  getDevice()
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 5. 机器人识别        │  isbot(userAgent)
│    - isbot 库检测    │
│    - 可通过 DISABLE_BOT_CHECK 关闭 │
│    - 命中返回 {beep: 'boop'} │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 6. IP 黑名单检查     │  hasBlockedIp(ip)
│    - IGNORE_IP 环境变量 │
│    - 精确 IP 匹配    │
│    - CIDR 网段匹配   │
│    - 命中返回 403    │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 7. Session 生成      │  uuid()
│    - 有 id 参数:     │  uuid(sourceId, id)
│    - 无 id 参数:     │  uuid(sourceId, ip, userAgent, sessionSalt)
│    - 盐值轮转策略    │  SALT_ROTATION (默认 month)
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 8. Visit 生成        │  uuid(sessionId, visitSalt)
│    - visitSalt = hash(整点时间) │
│    - 30 分钟过期机制 │
└─────────┬───────────┘
          │
          ▼
    数据持久化
```

---

## 三、各模块详细分析

### 3.1 IP 地址处理 ([ip.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/ip.ts))

#### IP 获取优先级

`getIpAddress()` 按以下优先级从 HTTP headers 中提取 IP：

1. **自定义 header**（`CLIENT_IP_HEADER` 环境变量）
2. `x-umami-client-ip`（仅 Cloud 模式）
3. `true-client-ip`（CDN）
4. `cf-connecting-ip`（Cloudflare）
5. `fastly-client-ip`（Fastly）
6. `x-nf-client-connection-ip`（Netlify）
7. `do-connecting-ip`（Digital Ocean）
8. `x-real-ip`（反向代理）
9. `x-appengine-user-ip`（Google App Engine）
10. `x-forwarded-for`（取第一个 IP）
11. `forwarded`（解析 `for=` 字段）
12. `x-client-ip`
13. `x-cluster-client-ip`
14. `x-forwarded`

#### IP 标准化流程

`normalizeIp()` + `resolveIp()` 处理：

- **IPv4 映射的 IPv6**：`::ffff:192.0.2.1` → `192.0.2.1`
- **端口剥离**：`192.168.1.1:8080` → `192.168.1.1`
- **IPv6 方括号处理**：`[2001:db8::1]:8080` → `[2001:db8::1]`
- **解析失败回退**：返回原始字符串

### 3.2 UA 覆盖链详解 ([detect.ts:126-138](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L126-L138))

#### 覆盖优先级

`getClientInfo()` 中存在一条完整的「payload → 自动检测」覆盖链，调用方可通过 payload 字段完全覆盖服务端的自动检测结果：

```typescript
export async function getClientInfo(request: Request, payload: Record<string, any>) {
  const userAgent = payload?.userAgent || request.headers.get('user-agent');   // UA 覆盖
  const ip        = payload?.ip        || getIpAddress(request.headers);      // IP 覆盖
  const browser   = payload?.browser   ?? browserName(userAgent);              // 浏览器覆盖
  const os        = payload?.os        ?? (detectOS(userAgent) as string);     // OS 覆盖
  const device    = payload?.device    ?? getDevice(userAgent, payload?.screen); // 设备覆盖
  // ...
}
```

| 字段 | 优先级（高 → 低） | Schema 定义 | 默认行为（tracker.js） |
|------|-------------------|-------------|------------------------|
| `userAgent` | `payload.userAgent` → `headers['user-agent']` | `z.string().optional()` | **不传**，走 header |
| `ip` | `payload.ip` → `getIpAddress(headers)` | `z.string().optional()` | **不传**，走 header 解析 |
| `browser` | `payload.browser` → `browserName(userAgent)` | `z.string().optional()` | **不传**，自动检测 |
| `os` | `payload.os` → `detectOS(userAgent)` | `z.string().optional()` | **不传**，自动检测 |
| `device` | `payload.device` → `getDevice(userAgent, screen)` | `z.string().optional()` | **不传**，自动检测 |

#### 关键设计细节

1. **tracker.js 默认不发送**：[tracker/index.js:64-74](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/tracker/index.js#L64-L74) 中 `getPayload()` 只包含 `website/screen/language/title/hostname/url/referrer/tag/id`，**不包含** userAgent/ip/browser/os/device。因此在正常浏览器采集场景下，所有字段均通过服务端自动检测。

2. **payload.ip 触发地理定位旁路**：当 payload 显式传入 `ip` 时，[detect.ts:129](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L129) 会将 `skipHeaders=true` 传给 `getLocation()`，导致**跳过所有 CDN 地理 header**，直接走 MaxMind 本地数据库查询。

3. **UA 为空的传递链**：如果 payload 不传 userAgent 且 header 中也无 `user-agent`，则 `userAgent = undefined`。该 undefined 会继续传递给：
   - `isbot(undefined)` → 机器人检测（通常返回 false）
   - `browserName(undefined)` → 浏览器检测（返回 null/undefined）
   - `detectOS(undefined)` → OS 检测（返回 null/undefined）
   - `getDevice(undefined, screen)` → 设备检测（fallback 为 'desktop'）
   - `uuid(sourceId, ip, undefined, sessionSalt)` → sessionId 生成（undefined 参与 hash）

4. **`??` vs `\|\|` 的差异**：`browser/os/device` 使用 `??`（空值合并），意味着 payload 传**空字符串 `''` 时会覆盖自动检测结果**；而 `userAgent/ip` 使用 `\|\|`，传空字符串会 fallback 到自动检测。

---

### 3.3 机器人识别 ([send/route.ts:131-133](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L131-L133))

#### 判断逻辑

```typescript
if (!process.env.DISABLE_BOT_CHECK && isbot(userAgent)) {
  return json({ beep: 'boop' });
}
```

#### 关键特性

- **使用库**：`isbot`（基于 user-agent 字符串匹配）
- **开关**：`DISABLE_BOT_CHECK` 环境变量可禁用
- **返回值**：HTTP 200 + `{ beep: 'boop' }`，伪装成正常响应，避免被机器人探测到过滤逻辑
- **位置**：在 IP 黑名单检查**之前**，优先过滤机器人流量，减少后续黑名单匹配开销

### 3.4 IP 黑名单基础 ([detect.ts:140-170](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L140-L170))

#### 函数签名与主逻辑

```typescript
export function hasBlockedIp(clientIp: string) {
  const ignoreIps = process.env.IGNORE_IP;
  if (ignoreIps) {
    const ips = ignoreIps.split(',').map(n => n.trim());
    return ips.find(ip => {
      if (ip === clientIp) return true;           // 精确匹配
      if (ip.indexOf('/') > 0) {                  // CIDR 网段
        const addr = ipaddr.parse(clientIp);
        const range = ipaddr.parseCIDR(ip);
        if (addr.kind() === range[0].kind() && addr.match(range)) {
          return true;
        }
      }
      return false;
    });
  }
  return false;
}
```

#### 支持的匹配方式

1. **精确 IP 匹配**：`192.168.1.1`
2. **CIDR 网段匹配**：`192.168.0.0/16`
3. **多个 IP**：逗号分隔，如 `1.1.1.1, 2.2.2.2/32`

#### 基础边界条件

- 环境变量未设置时直接返回 `false`（不拦截）
- 精确匹配使用 `===`，类型不同（string vs undefined）永远不命中
- IPv4 和 IPv6 需要同类型才能匹配（`addr.kind() === range[0].kind()`）
- **CIDR 分支无空值防御**：`clientIp` 为 undefined/''/null 时，`ipaddr.parse()` 会抛异常
- **异常无捕获**：整个函数无 try-catch，异常直接冒泡到调用方

### 3.5 地理信息 ([detect.ts:79-124](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L79-L124))

#### 获取流程

```
IP 地址
   │
   ├─▶ 本地 IP 检查 ── isLocalhost(ip) ── 是 ──▶ 返回 null
   │
   ├─▶ CDN Header 优先（SKIP_LOCATION_HEADERS 未设置）
   │     ├─ x-umami-client-* (Cloud 模式)
   │     ├─ cf-ip* (Cloudflare)
   │     ├─ x-vercel-ip-* (Vercel)
   │     ├─ cloudfront-viewer-* (CloudFront)
   │     └─ eo-ip* (EdgeOne)
   │
   └─▶ MaxMind 本地数据库
         └─ GeoLite2-City.mmdb
```

#### 返回字段

```typescript
{
  country: string;   // ISO 国家代码
  region: string;    // 国家-区域 格式，如 US-CA
  city: string;      // 城市名（英文）
}
```

#### 边界条件

- **本地 IP**（`isLocalhost`）直接返回 `null`，不进行地理定位
- **payload.ip 存在时**：跳过 CDN header 检查（`skipHeaders = !!payload?.ip`）
- **数据库未命中**：返回 `undefined`（隐式）
- **Header 编码**：使用 `latin1` 解码后转 `utf-8`
- **Region 格式**：不含 `-` 时自动补全国家前缀

### 3.6 缓存 token 对 sessionId 计算的逐行影响

**缓存生效的前置条件**：[send/route.ts:103](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L103)

```typescript
if (websiteId) {
  // ... 只有 payload.website 存在时才解析缓存
}
```

| payload 中的来源 | `websiteId` | cache 是否会被赋值 |
|---|---|---|
| `payload.website` | 有值 | ✅ 可能被赋值 |
| `payload.link` | undefined | ❌ 永远 `null`（整个 if 不进入） |
| `payload.pixel` | undefined | ❌ 永远 `null`（整个 if 不进入） |

**短链接追踪 `/q/[slug]` 和 像素追踪 `/p/[slug]` 的内部转发，使用的是 linkId 和 pixelId，cache 永远为 null！**

#### sessionId 计算行：完全不受 cache 影响

[send/route.ts:147](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L147)：

```typescript
const sessionId = id ? uuid(sourceId, id) : uuid(sourceId, ip, userAgent, sessionSalt);
```

逐字分析：**这一行代码没有任何一个字符引用了 `cache` 变量**。无论 cache 是 null 还是 `{sessionId:"..."}`，sessionId 的值完全由以下四者决定：

| 分支 | 输入 | 说明 |
|---|---|---|
| `id` 存在（identify 模式） | `sourceId` + `id` | 与 IP、UA、salt **完全无关** |
| `id` 不存在（匿名模式） | `sourceId` + `ip` + `userAgent` + `sessionSalt` | 与 cache.sessionId **完全无关** |

**结论：缓存中的 `cache.sessionId` 从不参与 sessionId 的值计算。**

---

### 3.7 缓存 token 对 createSession 会话记录创建的逐行影响

[send/route.ts:149-165](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L149-L165)

```typescript
// Create a session if not found
if (!clickhouse.enabled && !cache?.sessionId) {
  await createSession({
    id: sessionId,   // ← 注意！这里用的是「重新计算」的 sessionId，不是 cache.sessionId
    // ...
  });
}
```

#### if 条件拆解（两个取反 AND）

```
条件 = NOT clickhouse.enabled  AND  NOT cache?.sessionId
```

| `clickhouse.enabled` | `cache?.sessionId` | `!clickhouse.enabled && !cache?.sessionId` | 是否执行 createSession |
|---|---|---|---|
| true (ClickHouse 模式) | 任意 | false | **永不执行** |
| false (Prisma/SQL 模式) | 有值（truthy 字符串） | false | **不执行**（即使 sessionId 已被重算为新值） |
| false (Prisma/SQL 模式) | undefined/null/空串 | true | **执行** |

#### 危险边界：sessionId 重算值 ≠ cache.sessionId 时

场景示例：
```
时间点1（首次请求）
  IP = 1.1.1.1, UA = Chrome/120
  sessionId 计算为 = "SESSION_A"
  cache = null → !cache?.sessionId = true
  → createSession(id: "SESSION_A") ✓ 插入成功
  → 返回 token 中 sessionId = "SESSION_A"

时间点2（浏览器升级，UA 变了，同一次会话内）
  IP = 1.1.1.1, UA = Chrome/121  ← UA 变化了
  客户端传来缓存 token，cache.sessionId = "SESSION_A"
  新 sessionId 计算为 = "SESSION_B"  ← 因为 UA 变了
  if 条件：!cache?.sessionId = !"SESSION_A" = false
  → createSession 不执行（认为 session 已经存在）
  → 但后续 saveEvent 写入的 sessionId = "SESSION_B"
  → ⚠️  事件表中 session_id = "SESSION_B"，但会话表中根本没有这条记录！
```

**在非 ClickHouse 模式下，sessionId 一旦因 IP/UA/salt 变化而重算为新值，而缓存中存在旧 sessionId，就会产生孤儿事件数据。**

#### identify() 触发缓存清空后的会话创建

[tracker/index.js:219](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/tracker/index.js#L219)：`cache = ''`（空字符串）

空字符串传给 [jwt.ts:8-14](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/jwt.ts#L8-L14) 的 `parseToken()`：
```typescript
export function parseToken(token: string, secret: any) {
  try {
    return jwt.verify(token, secret);  // jwt.verify('') → 抛 JsonWebTokenError
  } catch {
    return null;   // ← catch 后返回 null
  }
}
```
所以 identify 之后 cache = null → `!cache?.sessionId = !null?.sessionId = !undefined = true` → createSession **必然执行**。

---

### 3.8 缓存 token 对 visitId + 30 分钟过期的逐行影响

[send/route.ts:167-175](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L167-L175)

```typescript
// Line 168
let visitId = cache?.visitId || uuid(sessionId, visitSalt);
// Line 169
let iat = cache?.iat || now;
// Line 172
if (!timestamp && now - iat > 1800) {
  visitId = uuid(sessionId, visitSalt);
  iat = now;
}
```

#### Line 168: visitId 初始化（`cache?.visitId \|\| uuid(...)` 的真值表）

| cache 状态 | `cache?.visitId` 的 JavaScript 值 | `\|\|` 左侧 truthy? | 最终 visitId |
|---|---|---|---|
| cache = null | `undefined` | ❌ | `uuid(sessionId, visitSalt)` |
| cache = { } (visitId 字段缺失) | `undefined` | ❌ | `uuid(...)` |
| cache.visitId = `""` | `""` (空串 falsy) | ❌ | `uuid(...)` |
| cache.visitId = `"ABC"` | `"ABC"` | ✅ | `"ABC"` |

#### Line 169: iat 初始化（`cache?.iat \|\| now` 的真值表）

| cache 状态 | `cache?.iat` 的 JavaScript 值 | `\|\|` 左侧 truthy? | 最终 iat |
|---|---|---|---|
| cache = null | `undefined` | ❌ | `now`（当前时间戳秒） |
| cache.iat = `0` | `0`（falsy） | ❌ | `now` ← ⚠️  iat=0 会被误判为「从未初始化」 |
| cache.iat = `1718000000` | `1718000000`（truthy） | ✅ | `1718000000` |

#### Line 172: 30 分钟过期判断的三重 AND 拆解

```
条件 A = !timestamp       （payload.timestamp 未传）
条件 B = now - iat > 1800 （当前时间 - 初始化的 iat > 30 分钟）
触发 = A && B
```

两个条件**同时满足**才会触发重算：

| 场景 | 条件 A | 条件 B | 是否触发过期重算 |
|---|---|---|---|
| 普通请求 + 29 分钟不活动 | true | `1740 > 1800` = false | ❌ 不触发 |
| 普通请求 + 31 分钟不活动 | true | `1860 > 1800` = true | ✅ 触发 |
| 传了 timestamp 参数 + 31 分钟不活动 | **false** | true | ❌ 不触发（A 短路） |
| identify 清缓存（iat=now） | true | `0 > 1800` = false | ❌ 不触发 |
| cache.iat = 0（iat 误判为 now） | true | 取决于 now | 与「普通请求」场景一致 |

#### 触发过期后的实际变化

```typescript
visitId = uuid(sessionId, visitSalt);   // ← 不使用 cache.visitId
iat = now;                               // ← 重置为当前时间
```

**重算 visitId 的输入是当前 sessionId + 当前 visitSalt，不使用缓存。** 因此：

- **同一小时内**（visitSalt 未变）+ **sessionId 也没变** → 重算的 visitId 与缓存值**完全相同** → 相当于只重置了 iat
- **跨整点**（visitSalt 变了）+ **sessionId 没变** → visitId **变化**（salt 变了，hash 结果不同）
- **sessionId 变了**（IP/UA 变了）→ visitId **必然变化**（输入不同）

**结论：30 分钟「过期」机制在同一小时内实际上不会换 visitId，只起到刷新 iat 时间戳的作用。真正会导致 visitId 变化的原因只有两个：整点跨边界、sessionId 本身变化。**

---

### 3.9 空 IP + CIDR 黑名单匹配的异常边界（按代码事实）

相关代码：[detect.ts:140-170](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L140-L170) + [send/route.ts:314-320](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L314-L320)

#### clientIp = undefined 是如何产生的

1. 调用方 payload 中**没有**传 `ip` 字段（[tracker.js 默认行为](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/tracker/index.js#L64-L74)）
2. [getIpAddress()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/ip.ts#L73-L76) 中找不到任何 IP header：
   ```typescript
   const header = IP_ADDRESS_HEADERS.find(name => headers.get(name));
   if (!header) {
     return undefined;   // ← 所有 14 个候选 header 都没有值
   }
   ```
3. getClientInfo 中 `payload?.ip || undefined` → 最终 `ip = undefined`
4. 传给 hasBlockedIp 的参数为 `undefined`

#### hasBlockedIp(undefined) 的逐行执行

```typescript
export function hasBlockedIp(clientIp: string) {  // TS 声明为 string，但运行时是 undefined
  const ignoreIps = process.env.IGNORE_IP;         // 假设 = "10.0.0.0/8,192.168.1.1"
  if (ignoreIps) {                                 // ✓ 进入
    const ips = ignoreIps.split(',').map(n => n.trim());
    // ips = ["10.0.0.0/8", "192.168.1.1"]

    return ips.find(ip => {
      // ┌─────────── 第 1 轮：ip = "10.0.0.0/8" ───────────┐
      if (ip === clientIp) {         // "10.0.0.0/8" === undefined → false
        return true;
      }
      if (ip.indexOf('/') > 0) {     // "10.0.0.0/8".indexOf('/') = 7 > 0 → 进入 CIDR
        const addr = ipaddr.parse(clientIp);  // ipaddr.parse(undefined)
        // ipaddr.js v2 源码:
        //   if (typeof addr !== 'string') {
        //     throw new TypeError("ipaddr.parse: string expected, got undefined");
        //   }
        // → ⚠️  TypeError 被抛出！
        const range = ipaddr.parseCIDR(ip);   // ← 永远执行不到

        if (addr.kind() === range[0].kind() && addr.match(range)) {
          return true;                        // ← 永远执行不到
        }
      }
      return false;                           // ← 永远执行不到
    });
    // ↑ 异常从 ips.find() 的回调中冒泡出来
  }
  return false;                               // ← 永远执行不到
}
```

#### 异常冒泡链与最终 HTTP 响应

```
hasBlockedIp() 中 ipaddr.parse() 抛 TypeError
    ↓ （无 try-catch）
ips.find() 终止迭代并向外冒泡
    ↓ （无 try-catch）
send/route.ts:136 → if (hasBlockedIp(ip)) { ... }
    ↓ （无 try-catch）
send/route.ts:66 外层 try { ... } catch (e) {
    const error = serializeError(e);
    console.log(error);
    return serverError({ errorObject: error });  // ← HTTP 500
  }
```

**最终结果：HTTP 500 Internal Server Error，控制台有 TypeError 日志。**

#### 所有配置组合的结果矩阵（clientIp = undefined 时）

| `IGNORE_IP` 环境变量值 | ips 数组内容 | 执行路径 | 结果 |
|---|---|---|---|
| 未设置 | - | `if (ignoreIps)` 不进入 | ✅ 放行 |
| `"192.168.1.1"` | `["192.168.1.1"]`（纯精确，无 CIDR） | 精确匹配全 false，CIDR 分支不进入 | ✅ 放行 |
| `"10.0.0.0/8"` | `["10.0.0.0/8"]`（纯 CIDR） | 第 1 次循环进入 CIDR → parse(undefined) 抛 | ❌ **500** |
| `"192.168.1.1,10.0.0.0/8"` | `["192.168.1.1","10.0.0.0/8"]`（精确在前） | 第 1 次精确 false，第 2 次进入 CIDR → 抛 | ❌ **500** |
| `"10.0.0.0/8,192.168.1.1"` | `["10.0.0.0/8","192.168.1.1"]`（CIDR 在前） | 第 1 次循环进入 CIDR → 抛 | ❌ **500** |

#### clientIp = ""（空字符串）时的表现

与 undefined 路径几乎一致：
- 精确匹配：`"10.0.0.0/8" === ""` → false
- CIDR 分支：`ipaddr.parse("")` → ipaddr.js 也会抛出（空串不是合法 IP 格式）
- **结果同样是 HTTP 500**

#### clientIp = `null` 时的表现

理论上不会出现（getIpAddress 返回 undefined，payload.ip 也被 Zod 限定为 string），但若出现：
- `typeof null !== 'string'` → ipaddr.js 同样抛 TypeError
- 结果也是 HTTP 500

**结论：只要 IGNORE_IP 中包含任意 CIDR 条目（含 `/`），且 clientIp 不是合法非空字符串（undefined/''/null），就一定会触发 500 错误。**

---

## 四、边界条件汇总（逐行验证版）

### 4.1 入口参数  cache 是否生效

基于 [send/route.ts:103](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L103) 的 `if (websiteId)` 判断：

| payload 中的来源 | `websiteId` | cache 分支是否进入 | cache 最终值 | 说明 |
|---|---|---|---|---|
| `payload.website`（事件采集） | 有值 |  进入 | 解析 token 成功 = JWT 内容；失败 / 无 token = `null` | 仅 website 模式支持缓存 |
| `payload.link`（短链接） | undefined |  不进入 | 恒为 `null` | 整个 if 块跳过 |
| `payload.pixel`（像素追踪） | undefined |  不进入 | 恒为 `null` | 整个 if 块跳过 |

**结论：`/q/[slug]` 短链接和 `/p/[slug]` 像素追踪完全不经过缓存逻辑，cache 恒为 null。**

---

### 4.2 cache  sessionId 计算

基于 [send/route.ts:147](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L147)：

`	ypescript
const sessionId = id ? uuid(sourceId, id) : uuid(sourceId, ip, userAgent, sessionSalt);
`

| 场景 | `id` (identify) | `ip` | `userAgent` | `sessionSalt` | `cache.sessionId` 影响 | sessionId 结果 |
|---|---|---|---|---|---|---|
| 1. identify 模式 | 有值 | 任意 | 任意 | 任意 |  完全无关 | `uuid(sourceId, id)`（确定性） |
| 2. 首次匿名请求（无缓存） | 无 | 正常 | 正常 | 正常 |  完全无关 | `uuid(sourceId, ip, ua, salt)` |
| 3. 有缓存且 IP/UA 未变 | 无 | 正常 | 正常 | 正常 |  完全无关 | 与缓存值相同（因为输入相同） |
| 4. 有缓存但 IP 变了 | 无 | 新 IP | 正常 | 正常 |  完全无关 | 新值（与缓存不同） |
| 5. 有缓存但 UA 变了 | 无 | 正常 | 新 UA | 正常 |  完全无关 | 新值（与缓存不同） |
| 6. 有缓存但跨 salt 周期 | 无 | 正常 | 正常 | 新 salt |  完全无关 | 新值（与缓存不同） |
| 7. IP + UA 都变了 | 无 | 新 IP | 新 UA | 正常 |  完全无关 | 新值（与缓存不同） |

**结论：sessionId 的计算完全不引用 cache 变量，缓存只影响 createSession 判断，不影响 sessionId 的值本身。**

---

### 4.3 cache  createSession 执行判断

基于 [send/route.ts:150](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L150)：

`	ypescript
if (!clickhouse.enabled && !cache?.sessionId) {
`

| `clickhouse.enabled` | `cache?.sessionId` | `!cache?.sessionId` | 条件结果 | 是否执行 createSession | 说明 |
|---|---|---|---|---|---|
| true | 任意 |  | false |  永不执行 | CH 模式异步合并，不实时插入 |
| false | 有值（非空字符串） | false | false |  不执行 | 即使 sessionId 已被重算为新值 |
| false | `undefined` (cache=null) | true | true |  执行 | 首次请求 / 缓存失效 |
| false | `""` (空字符串) | true | true |  执行 | `cache?.sessionId = ""`，`!"" = true` |
| false | `null` | true | true |  执行 | null 为 falsy |

**危险边界：第 2 行场景  sessionId 重算值  cache.sessionId 时，跳过 createSession 后产生孤儿事件。**

---

### 4.4 cache  visitId + 30 分钟过期

基于 [send/route.ts:168-175](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L168-L175)。

#### 4.4.1 visitId 初始化真值表（`cache?.visitId || uuid(...)`）

| cache 状态 | `cache?.visitId` 的值 | `\|\|` 左侧 truthy? | 最终 visitId |
|---|---|---|---|
| cache = null | `undefined` |  | `uuid(sessionId, visitSalt)` |
| cache = { } (visitId 字段缺失) | `undefined` |  | `uuid(...)` |
| cache.visitId = `""` | `""` (空串 falsy) |  | `uuid(...)` |
| cache.visitId = `"ABC"` | `"ABC"` |  | `"ABC"` |

#### 4.4.2 iat 初始化真值表（`cache?.iat || now`）

| cache 状态 | `cache?.iat` 的值 | `\|\|` 左侧 truthy? | 最终 iat | 说明 |
|---|---|---|---|---|
| cache = null | `undefined` |  | `now` | 首次请求 |
| cache.iat = `0` | `0` (falsy) |  | `now` |  误判为「从未初始化」 |
| cache.iat = `1718000000` | `1718000000` (truthy) |  | `1718000000` | 正常缓存 |

#### 4.4.3 30 分钟过期触发判断（`!timestamp && now - iat > 1800`）

| 场景 | `!timestamp` (条件 A) | `now - iat > 1800` (条件 B) | 触发重算? | 说明 |
|---|---|---|---|---|
| 普通请求 + 29 分钟不活动 | true | `1740 > 1800 = false` |  | 正常窗口内 |
| 普通请求 + 31 分钟不活动 | true | `1860 > 1800 = true` |  | 触发过期 |
| 传了 timestamp 参数 + 31 分钟 | **false** | true |  | A 短路，跳过过期检查 |
| identify 清缓存后首次（iat=now） | true | `0 > 1800 = false` |  | 新 session 立即开始新窗口 |
| cache.iat = 0（误判） | true | 取决于 now | 与「普通请求」一致 | iat 被重置为 now，相当于新开窗口 |
| 同一小时内过期触发（salt 未变） | true | true |  | 重算的 visitId 与原值**相同**（输入未变），只刷新 iat |
| 跨整点过期触发（salt 变了） | true | true |  | 重算的 visitId **不同**（visitSalt 变了） |

**核心结论：30 分钟「过期」在同一小时内实际上不换 visitId，只刷新 iat。真正导致 visitId 变化的只有两个原因：整点跨边界、sessionId 本身变化。**

---

### 4.5 空 IP + CIDR 黑名单异常边界

基于 [detect.ts:156-157](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L156-L157)。

`IGNORE_IP` 配置  `clientIp` 值 的完整组合矩阵：

| `IGNORE_IP` 配置 | `clientIp` 值 | 精确匹配分支 | CIDR 匹配分支 | 最终结果 | HTTP 状态 |
|---|---|---|---|---|---|
| 未设置（undefined） | 任意 |  |  | `false`（不拦截） | 正常 200 |
| `"1.1.1.1"`（仅精确） | `"1.1.1.1"` |  命中 | 不进入 | `true`（拦截） | 403 |
| `"1.1.1.1"`（仅精确） | `"2.2.2.2"` |  不命中 | 不进入（无 `/`） | `false` | 200 |
| `"1.1.1.1"`（仅精确） | `undefined` |  (`===` 类型不同) | 不进入 | `false` | 200 |
| `"10.0.0.0/8"`（仅 CIDR） | `"10.0.0.1"` |  不匹配 |  命中 | `true` | 403 |
| `"10.0.0.0/8"`（仅 CIDR） | `"11.0.0.1"` |  |  不命中 | `false` | 200 |
| `"10.0.0.0/8"`（仅 CIDR） | `undefined` |  |  `ipaddr.parse(undefined)` 抛 TypeError | **异常冒泡** | **500** |
| `"10.0.0.0/8"`（仅 CIDR） | `""` (空串) |  |  `ipaddr.parse("")` 抛错误 | **异常冒泡** | **500** |
| `"1.1.1.1, 10.0.0.0/8"`（混合） | `undefined` |  第一轮精确不命中 |  第二轮 CIDR 抛异常 | **异常冒泡** | **500** |
| `"10.0.0.0/8, 1.1.1.1"`（CIDR 在前） | `undefined` |  |  第一轮 CIDR 就抛异常 | **异常冒泡** | **500** |

**异常冒泡链：** `hasBlockedIp()`  `ips.find()` 回调  无 try-catch  send/route.ts 外层 try-catch  `serverError()`  HTTP 500

---

### 4.6 UA / IP / Browser 等字段覆盖链

基于 [detect.ts:126-137](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L126-L137)。

| 字段 | 代码 | 操作符 | payload 传 `undefined` | payload 传 `""` (空串) | 说明 |
|---|---|---|---|---|---|
| `userAgent` | `payload?.userAgent \|\| request.headers.get('user-agent')` | `\|\|` | fallback 到 header | fallback 到 header | `undefined` 和 `""` 都是 falsy |
| `ip` | `payload?.ip \|\| getIpAddress(request.headers)` | `\|\|` | fallback 到 getIpAddress | fallback 到 getIpAddress | 同上 |
| `browser` | `payload?.browser ?? browserName(userAgent)` | `??` | fallback 到自动检测 | **直接使用空串** | 空值合并只排除 null/undefined |
| `os` | `payload?.os ?? detectOS(userAgent)` | `??` | fallback 到自动检测 | **直接使用空串** | 同上 |
| `device` | `payload?.device ?? getDevice(userAgent, payload?.screen)` | `??` | fallback 到自动检测 | **直接使用空串** | 同上 |

**核心差异：** `||` 是「falsy 合并」（空串会 fallback），`??` 是「空值合并」（只有 null/undefined 才 fallback）。

---
## 五、关键代码位置索引

### 主入口
- 事件采集: [send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts)
- 录屏采集: [record/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/record/route.ts)
- 像素追踪: [p/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/(collect)/p/[slug]/route.ts)
- 链接追踪: [q/[slug]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/(collect)/q/[slug]/route.ts)

### 核心过滤
- IP 处理: [ip.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/ip.ts)
- 检测模块: [detect.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts)
  - UA/IP 覆盖链: [getClientInfo()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L126-L138)
  - 地理定位: [getLocation()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L79-L124)
  - IP 黑名单: [hasBlockedIp()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L140-L170)
  - 设备检测: [getDevice()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L49-L61)

### Session / Visit / Token
- UUID 生成: [crypto.ts:uuid()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/crypto.ts#L60-L66)
- Hash 函数: [crypto.ts:hash()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/crypto.ts#L48-L50)
- 盐值轮转: [crypto.ts:getSalt()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/crypto.ts#L72-L78)
- Session ID 计算: [send/route.ts:147](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L147)
- Session 创建判断: [send/route.ts:150](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L150)
- Visit ID 计算: [send/route.ts:168-175](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L168-L175)
- Token 签发: [send/route.ts:311](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L311)
- Token 校验: [jwt.ts:parseToken()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/jwt.ts#L8-L14)
- 缓存检查入口: [send/route.ts:100-122](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L100-L122)
- 录屏 Token 强制: [record/route.ts:41-52](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/record/route.ts#L41-L52)

### 机器人检测调用点
- 事件采集: [send/route.ts:131-133](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L131-L133)
- 录屏采集: [record/route.ts:82-84](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/record/route.ts#L82-L84)

### Tracker 客户端
- 采集 payload 构造: [tracker/index.js:64-74](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/tracker/index.js#L64-L74)
- identify() 缓存清空: [tracker/index.js:214-220](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/tracker/index.js#L214-L220)
- 请求发送 + 缓存 header: [tracker/index.js:175-190](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/tracker/index.js#L175-L190)
- 录屏缓存检查: [recorder/index.js:37-39](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/recorder/index.js#L37-L39)

---

## 六、流程总结

### 6.1 执行顺序（完整版）

```
请求到达
   │
   ▼
[1] Schema 验证
   ├─ type ∈ [event|identify|performance|record]
   ├─ website/link/pixel 三选一
   └─ 字段长度约束
   │ 错误 → 400 badRequest
   │
   ▼
[2] 缓存检查（仅 website 模式）
   ├─ 解析 x-umami-cache header (JWT)
   ├─ 成功 → cache = { websiteId, sessionId, visitId, iat }
   ├─ 失败 → cache = null
   └─ 有 cache.websiteId → 跳过 fetchWebsite
   │
   ▼
[3] 网站校验
   ├─ 有 link/pixel → Redis / Prisma 查找
   └─ 不存在 → 404 notFound / 400 badRequest
   │
   ▼
[4] 客户端信息收集（getClientInfo）
   ├─ userAgent: payload.userAgent || headers.user-agent
   ├─ ip:        payload.ip || getIpAddress(headers)
   ├─ location:  getLocation(ip, headers, skipHeaders=!!payload.ip)
   │   ├─ 本地 IP → null
   │   ├─ CDN headers（payload.ip 时跳过）→ 优先使用
   │   └─ MaxMind 本地库
   ├─ browser:   payload.browser ?? browserName(userAgent)
   ├─ os:        payload.os ?? detectOS(userAgent)
   └─ device:    payload.device ?? getDevice(userAgent, screen)
   │
   ▼
[5] 机器人识别
   ├─ DISABLE_BOT_CHECK → 跳过
   └─ isbot(userAgent) → 返回 200 { beep: 'boop' }
   │
   ▼
[6] IP 黑名单
   ├─ IGNORE_IP 未设置 → 跳过
   ├─ 精确匹配 → 命中: 403 forbidden
   ├─ CIDR 匹配
   │   ├─ clientIp 为 undefined/'' → ⚠️ ipaddr.parse() 抛异常 → 500
   │   └─ addr.kind !== range.kind → 不匹配
   │   └─ addr.match(range) → 命中: 403
   └─ 未命中 → 继续
   │
   ▼
[7] Session 生成
   ├─ id 存在 → uuid(websiteId, id)   （identify 模式）
   └─ id 不存在 → uuid(websiteId, ip, userAgent, sessionSalt)
   │   ├─ ip/ua 都是 undefined → 同站+同周期内碰撞为同一 session
   │   └─ sessionSalt = getSalt(SALT_ROTATION, createdAt)
   │
   ▼
[8] Session 创建（非 ClickHouse）
   └─ cache?.sessionId 不存在 → INSERT sessions
   │  ⚠️ 即使有缓存，sessionId 也已在 [7] 中重算
   │
   ▼
[9] Visit 生成
   ├─ cache?.visitId → 取缓存值
   │   ├─ now - iat <= 1800 → 保持不变
   │   ├─ 超时 + 无 timestamp → uuid(sessionId, visitSalt) 重算
   │   │   └─ 同一小时内 visitSalt 不变 → visitId 实际不变
   │   │   └─ 跨整点 visitSalt 变化 → visitId 变化
   │   └─ 有 timestamp → 跳过超时检查
   └─ 无缓存 → 直接 uuid(sessionId, visitSalt)
   │
   ▼
[10] 数据持久化
   ├─ saveEvent / saveSessionData / saveRecording
   └─ 返回 { cache: newToken, sessionId, visitId }
```

### 6.2 关键设计要点

1. **机器人检测先于 IP 黑名单**：减少黑名单匹配对机器人流量的无效开销
2. **地理定位在过滤之前**：统一在 `getClientInfo` 中获取，即使后续被过滤也已执行
3. **Session 生成在过滤之后**：被机器人检测/IP 黑名单拦截的请求不生成 session
4. **CDN Header 优先本地库**：利用服务商现成数据，减少 MaxMind 本地查询
5. **sessionId「重算」+「跳过创建」的缓存策略**：
   - 每次请求都会根据 IP/UA/salt **重新计算** sessionId 以保证一致性
   - 但用 `cache?.sessionId` 控制是否执行 INSERT，避免重复写库
   - 副作用：IP/UA 变化时，新 sessionId 与缓存中不一致，跳过 createSession 后产生孤儿事件
6. **盐值轮转机制**：按月/周/日重置 salt，即使 IP/UA 相同也会生成新 session，保护隐私
7. **30 分钟 Visit 过期 + 整点变化的双重机制**：
   - 同小时内超时只会更新 iat，visitId 实际不变
   - 跨整点 visitSalt 变化，visitId 必然变化
8. **identify() 主动清缓存**：强制触发 session 记录重建，切换到 id 模式
9. **录屏接口强依赖缓存**：无缓存 token 直接拒绝，不重新计算 session/visit

### 6.3 隐藏风险与 Bug 点（逐行验证版）

| # | 风险 | 影响 | 精确代码位置 |
|---|------|------|--------------|
| 1 | CIDR 配置 + 空 IP（undefined/''）→ `ipaddr.parse()` 抛 TypeError，外层 catch 返回 500 | 裸机部署且配了 CIDR 黑名单的环境必现 | [detect.ts:157](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L157) |
| 2 | 空 IP + 空 UA → `hash()` 中 `args.join('')` 将 undefined 拼成空串，所有匿名请求碰撞为同一个 sessionId | unique visitors 统计严重失真 | [crypto.ts:49](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/crypto.ts#L49) + [send/route.ts:147](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L147) |
| 3 | UA/IP 中途变化 → 新 sessionId 与 cache 中旧值不同，但因 cache.sessionId 存在跳过 createSession → 事件表引用不存在的会话 | 非 ClickHouse 模式产生孤儿事件记录，外键可能失败或数据孤立 | [send/route.ts:147](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L147) + [send/route.ts:150](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L150) |
| 4 | `payload.browser=''` 等空串覆盖 → `??` 操作符不走 fallback，browser/os/device 字段被写为空串 | 报表中出现空值浏览器/操作系统/设备 | [detect.ts:133-135](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L133-L135) |
| 5 | `/p /q` 采集入口使用 linkId/pixelId → `if (websiteId)` 缓存分支完全不进入 → cache=null → 每次请求都执行 createSession | 像素/短链接场景完全没有缓存优化，重复写库 | [send/route.ts:103](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L103) |
| 6 | sessionId 变了（如 IP 切换）但 visitId 仍沿用缓存中的旧值（场景 4.2 #7） | 同一个 visitId 下挂了不同 sessionId 的事件 → 会话/访问归属混乱 | [send/route.ts:168](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L168) |
| 7 | `cache.iat = 0` → `0 || now` 被误判为未初始化 → iat 重置为当前时间 | 意外地延长了 visit 的「30 分钟窗口」，应过期的 visit 不会过期 | [send/route.ts:169](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L169) |
