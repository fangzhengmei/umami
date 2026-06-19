# 告警规则与 Webhook 投递链路分析报告

---

## 0. 前置结论：告警规则与 Webhook 投递链路不存在

经过对项目代码、数据库 Schema、API 端点和配置文件的全面排查，**本项目（umami v3.1.0）中不存在告警规则（Alert Rules）和 Webhook 投递（Webhook Delivery）链路**。

### 0.1 不存在的具体证据

| 检查维度 | 是否存在 | 证据 |
|----------|----------|------|
| **告警规则数据表** | ❌ 不存在 | [schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/prisma/schema.prisma) 中无 `Alert` / `Rule` / `Trigger` / `Threshold` 等表；[schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/db/clickhouse/schema.sql) 中同样缺失 |
| **Webhook 配置数据表** | ❌ 不存在 | Prisma 和 ClickHouse schema 中无 `WebhookConfig` / `WebhookSubscription` / `WebhookDeliveryLog` 等表 |
| **告警规则 API 端点** | ❌ 不存在 | [src/app/api](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api) 目录下无 alert / rule / trigger 相关路由 |
| **Webhook 管理 API 端点** | ❌ 不存在 | 无 webhook / callback / endpoint 管理 API |
| **Webhook 投递代码** | ❌ 不存在 | 全项目无 webhook / outbound-http / http-callback 相关代码；`src/lib/fetch.ts` 仅为前端 HTTP 客户端，不是服务端 webhook 投递 |
| **环境变量配置** | ❌ 不存在 | [check-env.js](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/scripts/check-env.js) 无告警/Webhook 相关检查；全项目无 `WEBHOOK_*` / `ALERT_*` / `NOTIFY_*` 环境变量定义 |

### 0.2 容易被误判的代码澄清

以下代码常被误归类为"告警"或"Webhook"，但实际上是其他功能：

| 代码 | 实际用途 | 非告警/Webhook 原因 |
|------|----------|-------------------|
| [kafka.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/kafka.ts) | **内部数据管道**：将事件异步投递到 ClickHouse | 是**Inbound 数据写入**，不是对外部系统的 HTTP 回调；Kafka topic 消费者是 umami 自身的 ClickHouse |
| [/api/send](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts) | **客户端数据上报入口**：接收 tracker 脚本的页面事件 | 是接收浏览器端数据，不是向外部发送告警 |
| [getGoal.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/reports/getGoal.ts) | **目标转化统计报告**：计算页面浏览/事件触发的转化率 | 仅在用户主动查询时返回数据，无定时评估和自动触发机制 |
| [WEB_VITALS_THRESHOLDS](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/constants.ts#L106-L112) | **前端性能评级常量**：用于给 LCP/INP 等指标打 good/poor 标签 | 仅用于 UI 展示颜色区分，不触发任何告警或通知 |
| [useApi.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/components/hooks/useApi.ts) | **前端 React Query 封装**：浏览器端调用后端 API | 运行在客户端，不是服务端对外投递 |

---

## 1. 缺失部分清单

为了实现"告警规则 + Webhook 投递"的完整能力，本项目需要从头构建以下模块：

### 1.1 告警规则模块（缺失）

| 缺失项 | 说明 | 建议实现位置 |
|--------|------|-------------|
| **告警规则数据模型** | 存储规则定义（名称、类型、指标、阈值、比较符、评估周期、通知渠道） | `prisma/schema.prisma` 新增 `AlertRule` 表 |
| **告警实例数据模型** | 存储告警触发记录（规则ID、触发时间、当前值、阈值、状态：触发/恢复） | `prisma/schema.prisma` 新增 `AlertIncident` 表 |
| **规则评估引擎** | 定时（如每分钟）拉取指标数据，匹配阈值，判断是否触发/恢复 | 新增 `src/lib/alert/engine.ts` |
| **规则 CRUD API** | 创建/编辑/删除/启停告警规则的 REST 端点 | 新增 `src/app/api/alerts/route.ts` |
| **告警状态 API** | 查询告警历史、当前活跃告警、告警确认 | 新增 `src/app/api/alerts/[alertId]/route.ts` |
| **告警抑制/静默** | 防止告警风暴（连续触发只发一次、维护期静默） | `AlertIncident` 状态机 + 静默窗口配置 |

### 1.2 Webhook 投递模块（缺失）

| 缺失项 | 说明 | 建议实现位置 |
|--------|------|-------------|
| **Webhook 配置数据模型** | 存储 Webhook 端点（URL、签名密钥、自定义请求头、订阅的告警类型、启用状态） | `prisma/schema.prisma` 新增 `WebhookEndpoint` 表 |
| **Webhook 投递日志数据模型** | 记录每次投递（端点ID、告警ID、请求体、HTTP状态、耗时、错误信息、重试次数） | `prisma/schema.prisma` 新增 `WebhookDeliveryLog` 表 |
| **Webhook 投递器** | 构造标准请求体、计算 HMAC 签名、发送 HTTP POST、处理超时 | 新增 `src/lib/webhook/dispatcher.ts` |
| **投递重试队列** | 投递失败时按指数退避重试（建议最大 5 次），死信队列 | 基于 Redis List 或数据库轮询表 |
| **Webhook 管理 API** | 创建/编辑/删除/测试 Webhook 端点 | 新增 `src/app/api/webhooks/route.ts` |
| **Webhook 标准通知格式** | 定义触发/恢复通知的 JSON Schema（见第 8 章建议格式） | `src/lib/webhook/schema.ts` |

### 1.3 两者协作桥接（缺失）

| 缺失项 | 说明 |
|--------|------|
| **告警 → 通知 路由** | 告警引擎触发后，根据规则配置的通知渠道（Webhook 列表）分发事件 |
| **通知节流（Alert Throttling）** | 同一告警短时间内重复触发时，合并发送或静默 |
| **批量投递** | 同一时刻多个告警触发，批量打包投递减少 HTTP 调用 |

---

## 2. 现有代码分析：阈值匹配（非告警场景）

### 2.1 Web Vitals 性能评级阈值

**定位**：前端 UI 展示用评级常量，**不是告警触发阈值**

**代码位置**：[constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/constants.ts#L106-L112)

```typescript
export const WEB_VITALS_THRESHOLDS = {
  lcp:  { good: 2500, poor: 4000, unit: 'ms' },
  inp:  { good:  200, poor:  500, unit: 'ms' },
  cls:  { good:  0.1, poor: 0.25, unit: ''   },
  fcp:  { good: 1800, poor: 3000, unit: 'ms' },
  ttfb: { good:  800, poor: 1800, unit: 'ms' },
} as const;
```

**匹配逻辑**：[PerformanceCard.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/components/metrics/PerformanceCard.tsx#L26-L32)

```typescript
function getRating(metric: string, value: number): 'good' | 'needs-improvement' | 'poor' {
  const threshold = WEB_VITALS_THRESHOLDS[metric as keyof typeof WEB_VITALS_THRESHOLDS];
  if (!threshold || value <= 0) return 'good';
  if (value <= threshold.good) return 'good';
  if (value <= threshold.poor) return 'needs-improvement';
  return 'poor';
}
```

**当前用途**：驱动 `Badge` 组件显示绿色/黄色/红色标签，无任何后端告警动作。

**异常分支风险**：
| # | 问题 | 影响 |
|---|------|------|
| 1 | `value <= 0` 直接返回 `'good'` | 数据采集异常（如负值）被掩盖，评级永远正常 |
| 2 | 边界值均用 `<=`，`threshold.poor` 边界归属 `needs-improvement` 而非 `poor` | 语义和预期不符：达到 poor 阈值却仍显示"需改进" |
| 3 | 硬编码常量，不支持按网站自定义 | SaaS 多租户场景无法差异化阈值 |

### 2.2 Goal 转化目标阈值（概念层面，无代码实现）

**代码位置**：[getGoal.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/reports/getGoal.ts)

Goal 报告返回 `{ num, total }`，前端计算 `num/total` 百分比并画进度条，但**没有配置"转化率低于 X% 即告警"的入口**。

**异常分支风险**：
| # | 问题 | 影响 |
|---|------|------|
| 1 | `total === 0` 时转化率为 `0%`（[Goal.tsx#L84](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/(main)/websites/[websiteId]/(reports)/goals/Goal.tsx#L84)） | 分母为零时的除零保护了，但如果这是异常情况（如数据延迟），无法区分 |
| 2 | 目标匹配用 `LIKE` 通配符时，`*` 只替换首尾 | `path = "*foo*bar*"` 只能匹配 `%foo*bar%`，中间的 `*` 未处理 |

### 2.3 过滤器比较操作符

**代码位置**：[constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/constants.ts#L137-L154)

```typescript
export const OPERATORS = {
  equals: 'eq', notEquals: 'neq',
  greaterThan: 'gt', lessThan: 'lt',
  greaterThanEquals: 'gte', lessThanEquals: 'lte',
  contains: 'c', doesNotContain: 'dnc',
  // ...
} as const;
```

这些操作符用于数据查询过滤，**不是告警规则的阈值比较器**，但未来告警引擎可以复用其语义。

---

## 3. 现有代码分析：签名验证（认证场景，非 Webhook）

### 3.1 JWT + AES-256-GCM 双重安全 Token

**定位**：用户登录态认证，**不是 Webhook 请求签名**

**核心代码**：[jwt.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/jwt.ts) + [crypto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/crypto.ts)

```typescript
// 认证 Token：先 JWT 签名，再 AES-256-GCM 加密
export function createSecureToken(payload: any, secret: any, options?: any) {
  return encrypt(createToken(payload, secret, options), secret);
}

export function parseSecureToken(token: string, secret: any) {
  try {
    return jwt.verify(decrypt(token, secret), secret);
  } catch {
    return null;
  }
}
```

**加密算法细节**：
- AES-256-GCM，带 128 位认证标签（[crypto.ts#L5-L10](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/crypto.ts#L5-L10)）
- PBKDF2 密钥派生：10000 轮，SHA-512
- 密钥来源：`secret() = hash(APP_SECRET || DATABASE_URL)`

### 3.2 认证校验流程

**代码位置**：[auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/auth.ts#L17-L60)

```
请求 Authorization Header
       ↓
getBearerToken() 提取 "Bearer xxx"
       ↓
parseSecureToken() → 解密 + JWT 校验
       ↓
├─ userId → getUser(userId)
└─ authKey → redis.get(authKey) → getUser(key.userId)
       ↓
同时检查 x-umami-share-token + x-umami-share-context（分享链接场景）
```

### 3.3 Share Token 验证（分享链接场景）

**代码位置**：[auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/auth.ts#L80-L87)

```typescript
export function parseShareToken(request: Request) {
  try {
    return parseToken(request.headers.get(SHARE_TOKEN_HEADER), secret());
  } catch (e) {
    log(e);
    return null;
  }
}
```

### 3.4 Cache Token 验证（数据采集去重场景）

**代码位置**：[send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L100-L122)

```typescript
const cacheHeader = request.headers.get('x-umami-cache');
if (cacheHeader) {
  const result = await parseToken(cacheHeader, secret()); // 纯 JWT，未加密
  if (result) cache = result;
}
```

**异常分支风险汇总**：

| # | 问题 | 具体代码 | 影响 |
|---|------|----------|------|
| 1 | 所有 parse* 函数**静默吞异常**，统一返回 `null` | [jwt.ts#L8-L14](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/jwt.ts#L8-L14) | 无法区分「签名无效」vs「Token 过期」vs「解密失败」，调试噩梦 |
| 2 | JWT 创建**未强制 expiresIn** | 各处 `createToken(...)` 调用 | 泄露后永久有效，无内置过期机制 |
| 3 | 加密密钥 = JWT 密钥 = 同一个 `secret()` | [crypto.ts#L56-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/crypto.ts#L56-L58) | 密钥用途不分离，违反最小权限原则 |
| 4 | Share Token 验证**只验 header 是否存在** | [auth.ts#L43-L48](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/auth.ts#L43-L48) | 仅判断 `SHARE_CONTEXT_HEADER` 是否为非空，未校验其内容是否与 token 匹配 |
| 5 | Cache Token 用**裸 JWT（未加密）** 且**无过期** | [send/route.ts#L107](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L107) | 抓包后可永久复用 |

> **对未来 Webhook 的启示**：如果后续实现 Webhook，HMAC 签名计算需要参考当前 AES-GCM 的严谨性，但应避免「静默吞异常」的错误处理模式。

---

## 4. 现有代码分析：防重放（数据采集场景）

### 4.1 SessionId 确定性生成

**定位**：将同一用户的多次事件聚合到同一个 session，**防止重复统计**，不是 API 请求级防重放

**代码位置**：[send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L143-L149)

```typescript
const saltRotation = process.env.SALT_ROTATION || 'month';
const sessionSalt = getSalt(saltRotation, createdAt);
const sessionId = id
  ? uuid(sourceId, id)                           // 有 identify() 时基于 distinctId
  : uuid(sourceId, ip, userAgent, sessionSalt);  // 否则基于 IP + UA + 盐值
```

**盐值轮换逻辑**：[crypto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/crypto.ts#L72-L78)

```typescript
export function getSalt(saltRotation: string, createdAt: Date): string {
  return hash(
    (saltRotation === 'day' ? startOfDay : saltRotation === 'week' ? startOfWeek : startOfMonth)(
      createdAt,
    ).toUTCString(),
  );
}
```

### 4.2 VisitId 30 分钟过期

**代码位置**：[send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L168-L175)

```typescript
let visitId = cache?.visitId || uuid(sessionId, visitSalt);
let iat = cache?.iat || now;

// Expire visit after 30 minutes
if (!timestamp && now - iat > 1800) {
  visitId = uuid(sessionId, visitSalt);
  iat = now;
}
```

`visitSalt = hash(startOfHour(createdAt).toUTCString())`，按小时变化。

### 4.3 Cache Token 回传机制

**代码位置**：[send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L311-L313)

```typescript
const token = createToken({ websiteId, sessionId, visitId, iat }, secret());
return json({ cache: token, sessionId, visitId });
```

客户端下次请求通过 `x-umami-cache` header 回传，服务端直接复用 sessionId/visitId，**避免重新计算**（从而达到去重效果）。

**异常分支风险汇总**：

| # | 问题 | 影响 |
|---|------|------|
| 1 | **带 `timestamp` 参数绕过过期检查**：`!timestamp` 条件成立才判断 `now - iat > 1800` | 攻击者可构造旧 timestamp，强制复用过期的 visitId，扭曲访问时长统计 |
| 2 | **Cache Token 无 `exp` 声明**：`createToken` 时未传 `expiresIn` | 即使 visitId 逻辑过期，JWT 本身仍然有效；结合第 1 条可长期复用 |
| 3 | **盐值轮换边界数据断裂**：月/周/日轮换时刻，同一用户 sessionId 突变 | 跨轮换点的用户被统计为两个新用户，数据一致性受损 |
| 4 | **VisitSalt 按小时轮换**：整点时 visitId 突变 | 同一次长会话（跨整点）被拆分为多个 visit |
| 5 | **无请求级 nonce**：没有 `X-Request-ID` / nonce 去重 | 完全相同的 HTTP 请求（复制粘贴重放）会被当作新事件重复入库 |
| 6 | **SessionId 依赖 IP + UA**：用户换网络或浏览器升级，被当新用户 | 统计数据高估独立访客数 |

> **对未来 Webhook 的启示**：Webhook 防重放应独立实现 `timestamp + nonce + 签名过期窗口` 三件套，不要复用数据采集的 session 机制。

---

## 5. 现有代码分析：节流（Redis RateLimit 未实际应用）

### 5.1 Redis 固定窗口限流函数

**代码位置**：[redis.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/redis.ts#L72-L82)

```typescript
async rateLimit(key: string, limit: number, seconds: number): Promise<boolean> {
  await this.connect();
  const res = await this.client.incr(key);
  if (res === 1) {
    await this.client.expire(key, seconds);
  }
  return res >= limit;
}
```

**算法**：固定窗口计数器
- 返回 `true` 表示已超限，应拒绝请求
- 返回 `false` 表示放行

### 5.2 IP 黑名单（CIDR 支持）

**代码位置**：[detect.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/detect.ts#L140-L169)

```typescript
export function hasBlockedIp(clientIp: string) {
  const ignoreIps = process.env.IGNORE_IP;
  // 支持单个 IP 和 CIDR 网段（ipaddr.js match）
}
```

在 `/api/send` 中被调用：[send/route.ts#L135-L138](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L135-L138)

### 5.3 机器人检测

**代码位置**：[send/route.ts#L130-L133](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L130-L133)

使用 `isbot` 库检测爬虫 UA，返回 `{ beep: 'boop' }` 静默不入库。

**异常分支风险汇总**：

| # | 问题 | 影响 |
|---|------|------|
| 1 | **`rateLimit` 函数存在但从未被任何 API 调用** | 核心数据入口 `/api/send` 无限流保护，可被刷量攻击 |
| 2 | **INCR + EXPIRE 非原子**：两步操作之间如果进程崩溃 | Key 永久存在且永远递增，导致**永久限流**该 key |
| 3 | **无 Redis 降级策略**：Redis 连接失败时直接抛异常 | 应配置「Redis 不可用时放行」还是「Redis 不可用时拒绝」 |
| 4 | **固定窗口临界突刺**：窗口边界 1 秒内可能通过 2× limit 请求 | 高并发场景保护不足 |
| 5 | **限流 Key 无命名规范**：函数只接收裸 key，不自动加前缀 | 不同调用方可能冲突（如 `send:ip:1.2.3.4` 被别处使用同名） |

> **对未来 Webhook 的启示**：Webhook **出向**节流（控制向外部端点的发送频率，防止被对端封禁）需要单独实现，与入向限流方向相反。建议使用令牌桶算法。

---

## 6. 现有代码分析：失败重试（实际几乎无重试）

### 6.1 Kafka 消息投递

**代码位置**：[kafka.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/kafka.ts#L66-L91)

```typescript
async function sendMessage(topic, message) {
  try {
    await connect();
    return producer.send({
      topic, messages: [...],
      timeout: 3000,   // SEND_TIMEOUT = 3000
      acks: 1,          // 只等 leader 确认
    });
  } catch (e) {
    console.log('KAFKA ERROR:', serializeError(e));
    // ← 只打日志！不抛出异常，不重试，不回退到 ClickHouse 直写
  }
}
```

**被调用方**：[saveEvent.ts#L259-L263](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEvent.ts#L259-L263)

```typescript
if (kafka.enabled) {
  await sendMessage('event', message);        // 失败了？saveEvent 完全不知情
} else {
  await insert('website_event', [message]);   // ← Kafka 可用时永远不会走这里
}
```

### 6.2 Batch API 错误收集（无重试）

**代码位置**：[batch/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/batch/route.ts#L17-L54)

```typescript
for (const data of body) {
  const response = await send.POST(newRequest);   // 逐条串行调用
  const responseJson = await response.json();
  if (!response.ok) {
    errors.push({ index, response: responseJson });
  }
  index++;
}
// 仅把错误索引记录下来返回给调用方，不做任何重试
```

### 6.3 数据库写入（Prisma / ClickHouse）

[saveEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEvent.ts) 中的 Prisma `create` 和 ClickHouse `insert` 均为**一次性调用**，无 catch，无重试。

**异常分支风险汇总**：

| # | 问题 | 影响 |
|---|------|------|
| 1 | **Kafka 失败静默吞错**：catch 不抛异常，调用方 `await sendMessage(...)` 收到 `undefined` 以为成功 | **数据静默丢失**，最严重问题 |
| 2 | **无任何重试机制**：Kafka / DB / Batch 全部失败即终态 | 瞬时故障（网络抖动）直接丢数据 |
| 3 | **无死信队列**：重试耗尽后无法人工介入 | 故障恢复后补数困难 |
| 4 | **Kafka 不可用时无降级**：配置了 `KAFKA_URL` 就走 Kafka，失败了不会回退直写 | 发送方成功/消费者滞后时，上层无感知 |
| 5 | **Batch 串行调用 + 无短路**：100 条里第 1 条就超时，也要等 99 次尝试 | 雪崩加剧 |

> **对未来 Webhook 的启示**：Webhook 投递必须有指数退避重试（1s, 2s, 4s, 8s, 16s, max 5 次）+ 死信队列 + 人工重放 UI。可以吸取 Kafka 这里的教训，**绝对不能静默吞错**。

---

## 7. 现有代码分析：外部通知 / 事件投递（完全不存在 Webhook）

### 7.1 Kafka 不是 Webhook

| 维度 | Kafka（本项目） | Webhook（告警投递） |
|------|-----------------|-------------------|
| **方向** | 服务内部：应用 → 消息队列 → 本应用的消费者 | 对外：本应用 → 第三方系统 |
| **协议** | Kafka 二进制协议（TCP） | HTTP/HTTPS POST |
| **载荷格式** | 本应用自定义 snake_case JSON | 需标准化（建议 CloudEvents） |
| **消费方** | 本应用 ClickHouse 消费者进程 | 用户配置的任意 URL |
| **认证方式** | SASL PLAIN / SCRAM（Kafka 内部） | HMAC 签名（Header 携带） |
| **重试策略** | 无（消费者失败自处理） | 必须：指数退避 + 死信 |

### 7.2 数据采集 API（/api/send）不是告警通知

`/api/send` 是**入向数据接收**（Inbound）：浏览器 tracker 脚本把页面事件上报给后端。

告警通知是**出向数据发送**（Outbound）：后端把告警事件 POST 到用户的 Webhook URL。

两者方向完全相反，代码模式也完全不同。

### 7.3 前端 React Query（useApi.ts）不是服务端投递

[useApi.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/components/hooks/useApi.ts) 运行在浏览器中，用 `@tanstack/react-query` 做客户端数据缓存。**服务端没有类似的对外 HTTP POST 封装库。**

---

## 8. 建议的 Webhook 通知标准格式（待实现参考）

由于本项目尚无此模块，以下为推荐实现方案（对齐 CNCF CloudEvents 1.0 规范）：

### 8.1 告警触发通知

```http
POST /your-webhook-endpoint HTTP/1.1
Host: example.com
Content-Type: application/cloudevents+json
X-Webhook-Signature: sha256=<HMAC_HEX>
X-Webhook-Timestamp: 1718889600
X-Webhook-Nonce: a1b2c3d4e5f6
X-Webhook-Event: alert.fired
X-Webhook-Delivery-ID: evt_abc123def456
```

```json
{
  "specversion": "1.0",
  "type": "umami.alert.fired",
  "source": "https://umami.example.com/api/alerts",
  "subject": "alert_7f9c2b4a-e6d1-4f3a-8b0c-1d2e3f4a5b6c",
  "id": "evt_abc123def456",
  "time": "2024-06-20T08:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "alert": {
      "id": "alert_7f9c2b4a-e6d1-4f3a-8b0c-1d2e3f4a5b6c",
      "name": "LCP 性能告警",
      "description": "首页 LCP 超过 4 秒",
      "severity": "critical",
      "rule": {
        "metric": "performance.lcp.p95",
        "operator": "gt",
        "threshold": 4000,
        "windowSeconds": 300,
        "websiteId": "web_12345678-1234-1234-1234-1234567890ab"
      },
      "triggeredAt": "2024-06-20T08:00:00Z",
      "status": "firing"
    },
    "currentValue": {
      "metric": "performance.lcp.p95",
      "value": 5231,
      "unit": "ms",
      "samples": 42,
      "windowStart": "2024-06-20T07:55:00Z",
      "windowEnd": "2024-06-20T08:00:00Z"
    },
    "threshold": {
      "metric": "performance.lcp.p95",
      "value": 4000,
      "operator": "greater_than"
    },
    "links": {
      "dashboard": "https://umami.example.com/websites/web_12345678/performance",
      "alertDetails": "https://umami.example.com/alerts/alert_7f9c2b4a-e6d1-4f3a-8b0c-1d2e3f4a5b6c"
    },
    "tags": {
      "website": "example.com",
      "environment": "production",
      "team": "frontend"
    }
  }
}
```

### 8.2 告警恢复通知

```
X-Webhook-Event: alert.resolved
```

```json
{
  "type": "umami.alert.resolved",
  "data": {
    "alert": { "id": "...", "status": "resolved", "resolvedAt": "2024-06-20T08:15:00Z" },
    "resolvedValue": {
      "metric": "performance.lcp.p95",
      "value": 2150,
      "unit": "ms",
      "windowStart": "2024-06-20T08:10:00Z",
      "windowEnd": "2024-06-20T08:15:00Z"
    },
    "durationSeconds": 900,
    "peakValue": 5820
  }
}
```

### 8.3 签名算法（建议）

```
payload_string = JSON.stringify(body)
timestamp = 1718889600
signature_base = f"{timestamp}.{payload_string}"
hmac = HMAC-SHA256(webhook_secret, signature_base)
X-Webhook-Signature = "sha256=" + hex(hmac)
X-Webhook-Signature 支持逗号分隔多个版本（密钥轮换用）
```

接收方校验：
1. 检查 `X-Webhook-Timestamp` 与当前时间差不超过 300 秒（防重放）
2. 校验 HMAC 签名
3. 检查 `id` / `X-Webhook-Nonce` 在去重缓存中是否已处理过（防重放）

### 8.4 投递响应约定

| 接收方响应 | 投递方行为 |
|-----------|-----------|
| `2xx` | 标记成功，归档日志 |
| `401` / `403` | **不重试**，标记失败，告警管理员（密钥失效） |
| `404` | **不重试**，标记失败，告警管理员（URL 失效） |
| `429` | 读取 `Retry-After` header，延迟重试 |
| `5xx` / 超时 / 连接失败 | 指数退避重试（1s, 2s, 4s, 8s, 16s），共 5 次 |

---

## 9. 推荐的端到端实现顺序

如果后续要实现告警规则 + Webhook 投递，建议按以下顺序开发：

```
阶段 1：数据模型 + 告警引擎
  └─ ① AlertRule / AlertIncident 数据表
  └─ ② 告警评估引擎（定时拉指标 → 阈值比较 → 写 incident）
  └─ ③ 规则 CRUD API

阶段 2：Webhook 基础能力
  └─ ④ WebhookEndpoint 数据表
  └─ ⑤ dispatcher（HTTP POST + HMAC 签名 + 超时）
  └─ ⑥ 端点 CRUD + 测试 API

阶段 3：可靠性增强
  └─ ⑦ 指数退避重试队列
  └─ ⑧ WebhookDeliveryLog 表
  └─ ⑨ 死信队列 + 人工重放 UI

阶段 4：质量完善
  └─ ⑩ 告警静默 / 抑制 / 合并
  └─ ⑪ 出向令牌桶限流（防被封禁）
  └─ ⑫ 告警触发时自动截图 / 指标快照链接
```

---

## 10. 总结

| 项目 | 当前状态 | 关键问题 |
|------|----------|---------|
| **告警规则** | ❌ 完全不存在 | 无数据表、无评估引擎、无触发 API |
| **Webhook 投递** | ❌ 完全不存在 | 无端点表、无投递器、无签名与重试 |
| **阈值匹配** | ⚠️ 仅评级展示用 | Web Vitals 有常量但无告警联动；Goal 无阈值配置 |
| **签名验证** | ✅ 认证场景完善 | 但静默吞异常 + 无过期，不可直接套用到 Webhook |
| **防重放** | ⚠️ 数据采集去重层面 | session/visit 机制，无请求级 nonce；有 timestamp 绕过漏洞 |
| **节流** | ⚠️ 函数存在但未调用 | rateLimit 没挂到任何 API；INCR+EXPIRE 非原子 |
| **失败重试** | ❌ 几乎为零 | Kafka 静默吞错最危险；无死信队列 |
| **外部通知格式** | ❌ 无标准 | 需全新设计，建议参考 CloudEvents 1.0 |
