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

### 3.2 机器人识别 ([send/route.ts:131-133](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L131-L133))

#### 判断逻辑

```typescript
if (!process.env.DISABLE_BOT_CHECK && isbot(userAgent)) {
  return json({ beep: 'boop' });
}
```

#### 关键特性

- **使用库**：`isbot`（基于 user-agent 字符串匹配）
- **开关**：`DISABLE_BOT_CHECK` 环境变量可禁用
- **返回值**：返回 `{ beep: 'boop' }` 而非错误，伪装成正常响应
- **位置**：在 IP 黑名单检查**之前**

### 3.3 IP 黑名单 ([detect.ts:140-170](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L140-L170))

#### 判断逻辑

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

#### 边界条件

- 环境变量未设置时直接返回 `false`（不拦截）
- IPv4 和 IPv6 需要同类型才能匹配（`addr.kind() === range[0].kind()`）
- 解析失败会抛出异常（未做 try-catch）

### 3.4 地理信息 ([detect.ts:79-124](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L79-L124))

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

### 3.5 Session 生成 ([crypto.ts:60-77](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/crypto.ts#L60-L77))

#### UUID 生成方式

```typescript
export function uuid(...args: any) {
  if (args.length) {
    return v5(hash(...args, secret()), v5.DNS);  // 确定性 UUID v5
  }
  return process.env.USE_UUIDV7 ? v7() : v4();   // 随机 UUID v4/v7
}
```

#### Session ID 生成逻辑

在 [send/route.ts:147](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L147)：

```typescript
const sessionId = id ? uuid(sourceId, id) : uuid(sourceId, ip, userAgent, sessionSalt);
```

**两种模式：**

1. **有 `id` 参数**（identify 模式）：`uuid(sourceId, id)`
   - 基于网站 ID + 用户自定义 ID 生成
   - 盐值不参与，ID 固定则 session 固定

2. **无 `id` 参数**（匿名模式）：`uuid(sourceId, ip, userAgent, sessionSalt)`
   - 基于网站 ID + IP + UA + 盐值生成
   - 盐值轮转会导致 session 周期性重置

#### 盐值轮转策略

`getSalt()` 函数根据 `SALT_ROTATION` 环境变量决定：

| 轮转周期 | 说明 |
|----------|------|
| `day` | 每天 00:00 UTC 重置 |
| `week` | 每周一 00:00 UTC 重置 |
| `month`（默认） | 每月 1 号 00:00 UTC 重置 |

```typescript
export function getSalt(saltRotation: string, createdAt: Date): string {
  return hash(
    (saltRotation === 'day' ? startOfDay : saltRotation === 'week' ? startOfWeek : startOfMonth)(
      createdAt,
    ).toUTCString(),
  );
}
```

### 3.6 Visit 生成 ([send/route.ts:168-175](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L168-L175))

```typescript
let visitId = cache?.visitId || uuid(sessionId, visitSalt);
let iat = cache?.iat || now;

// Expire visit after 30 minutes
if (!timestamp && now - iat > 1800) {
  visitId = uuid(sessionId, visitSalt);
  iat = now;
}
```

**关键特性：**

- **visitSalt**：`hash(startOfHour(createdAt).toUTCString())` —— 每小时变化
- **过期机制**：30 分钟无活动则生成新的 visitId
- **时间戳模式**：如果请求带了 `timestamp` 参数，则跳过 30 分钟过期检查
- **缓存优先**：有缓存（`x-umami-cache`）时使用缓存中的 visitId

---

## 四、边界条件汇总

### 4.1 机器人识别边界

| 条件 | 结果 | 说明 |
|------|------|------|
| `DISABLE_BOT_CHECK=true` | 跳过机器人检测 | 环境变量禁用 |
| `userAgent` 为空/undefined | 由 `isbot` 库决定 | 通常返回 false |
| 命中机器人 | 返回 `{ beep: 'boop' }` | HTTP 200，伪装成功 |
| 位置 | IP 黑名单检查**之前** | 先过滤机器人，再检查 IP |

### 4.2 IP 黑名单边界

| 条件 | 结果 | 说明 |
|------|------|------|
| `IGNORE_IP` 未设置 | 不拦截 | 直接返回 false |
| 空 IP | 不拦截（`ips.find` 不会匹配） | 隐式行为 |
| IPv4 vs IPv6 不匹配 | 不拦截 | CIDR 匹配时类型必须一致 |
| 命中 | 返回 403 Forbidden | `forbidden()` |
| CIDR 解析失败 | 抛出异常 | 无 try-catch 保护 |

### 4.3 地理信息边界

| 条件 | 结果 | 说明 |
|------|------|------|
| IP 为空 | 返回 `null` | 不进行定位 |
| 本地 IP (localhost) | 返回 `null` | `isLocalhost()` 判断 |
| payload.ip 存在 | 跳过 CDN header | `skipHeaders = true` |
| `SKIP_LOCATION_HEADERS=true` | 跳过 CDN header | 强制使用数据库 |
| CDN header 有值 | 直接使用，不查库 | 优先使用服务商数据 |
| 数据库未命中 | 返回 `undefined` | 隐式返回 |
| region 无 `-` | 自动补全国家前缀 | 如 `CA` → `US-CA` |

### 4.4 Session 生成边界

| 条件 | Session ID 生成方式 |
|------|---------------------|
| 有 `id` 参数 | `uuid(sourceId, id)` |
| 无 `id` 参数 | `uuid(sourceId, ip, userAgent, sessionSalt)` |
| `timestamp` 参数 | 使用该时间计算 salt |
| 无 `timestamp` | 使用当前时间计算 salt |
| 盐值轮转 | 周期性重置 session |

### 4.5 Visit 生成边界

| 条件 | 结果 |
|------|------|
| 有缓存 visitId | 使用缓存值 |
| 无缓存 | 基于 sessionId + visitSalt 生成 |
| `now - iat > 1800` 秒 | 重新生成 visitId（30 分钟过期） |
| 有 `timestamp` 参数 | 跳过 30 分钟过期检查 |
| visitSalt 每小时变化 | 整点后首次请求生成新 visitId |

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
  - 客户端信息: [getClientInfo()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L126-L138)
  - 地理定位: [getLocation()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L79-L124)
  - IP 黑名单: [hasBlockedIp()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L140-L170)
  - 设备检测: [getDevice()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/detect.ts#L49-L61)

### Session/Visit
- UUID 生成: [crypto.ts:uuid()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/crypto.ts#L60-L66)
- 盐值轮转: [crypto.ts:getSalt()](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/lib/crypto.ts#L72-L78)
- Session ID: [send/route.ts:147](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L147)
- Visit ID: [send/route.ts:168-175](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L168-L175)

### 机器人检测调用点
- 事件采集: [send/route.ts:131-133](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/send/route.ts#L131-L133)
- 录屏采集: [record/route.ts:82-84](file:///d:/fz/0601-2/solo-dogfeeding/code/48-umami/src/app/api/record/route.ts#L82-L84)

---

## 六、流程总结

### 6.1 执行顺序

1. **Schema 验证** → 格式错误直接返回
2. **缓存解析** → 有缓存则复用 session/visit
3. **网站校验** → 网站不存在返回 400
4. **客户端信息收集** → IP / UA / 地理 / 设备 / 浏览器 / OS
5. **机器人识别** → 命中返回 200 + `{beep: 'boop'}`
6. **IP 黑名单** → 命中返回 403
7. **Session 生成** → 基于 IP+UA+salt（或自定义 id）
8. **Visit 生成** → 基于 sessionId+visitSalt，30 分钟过期

### 6.2 关键设计要点

- **机器人检测先于 IP 黑名单**：减少黑名单对机器人的无效检查
- **地理定位在过滤之前**：即使被过滤也会执行（`getClientInfo` 统一获取）
- **Session 生成在过滤之后**：被过滤的请求不生成 session
- **CDN Header 优先于本地库**：减少本地库查询，提升性能
- **盐值轮转机制**：保护用户隐私，定期重置 session 标识
- **30 分钟 Visit 过期**：平衡访问计数准确性与数据量
