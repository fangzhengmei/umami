# 告警规则与 Webhook 投递代码协作分析

## 概述

本文档分析 umami 项目中告警规则与 Webhook 投递相关的代码协作机制，涵盖阈值匹配、签名校验、防重放、节流、失败重试和外部通知格式等核心环节，并指出各环节中容易漏掉的异常分支。

## 1. 阈值匹配

### 1.1 Web Vitals 性能阈值

**核心代码**：[constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/constants.ts#L106-L112)

```typescript
export const WEB_VITALS_THRESHOLDS = {
  lcp: { good: 2500, poor: 4000, unit: 'ms' },
  inp: { good: 200, poor: 500, unit: 'ms' },
  cls: { good: 0.1, poor: 0.25, unit: '' },
  fcp: { good: 1800, poor: 3000, unit: 'ms' },
  ttfb: { good: 800, poor: 1800, unit: 'ms' },
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

### 1.2 Goal 目标阈值

**核心代码**：[getGoal.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/reports/getGoal.ts)

Goal 报告通过 `num`（目标达成数）和 `total`（总会话数）计算转化率，但**未设置显式的告警阈值**，仅在前端展示进度条。

### 1.3 过滤器操作符阈值

**核心代码**：[constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/constants.ts#L137-L154)

```typescript
export const OPERATORS = {
  equals: 'eq',
  notEquals: 'neq',
  greaterThan: 'gt',
  lessThan: 'lt',
  greaterThanEquals: 'gte',
  lessThanEquals: 'lte',
  // ...
} as const;
```

### ⚠️ 容易漏掉的异常分支

1. **阈值边界值处理不一致**：`value <= threshold.good` 使用 `<=`，但 `value <= threshold.poor` 也使用 `<=`，边界值归属可能引发争议
2. **负值或零值直接返回 good**：`value <= 0` 时直接返回 `'good'`，可能掩盖数据异常
3. **Goal 无阈值配置**：Goal 报告只有转化率计算，没有触发告警的阈值配置，无法自动触发告警
4. **动态阈值缺失**：所有阈值都是硬编码常量，不支持按网站/用户自定义配置

---

## 2. 签名校验

### 2.1 JWT Token 机制

**核心代码**：[jwt.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/jwt.ts)

```typescript
export function createToken(payload: any, secret: any, options?: any) {
  return jwt.sign(payload, secret, options);
}

export function parseToken(token: string, secret: any) {
  try {
    return jwt.verify(token, secret);
  } catch {
    return null;
  }
}
```

### 2.2 安全 Token（加密 + JWT）

**核心代码**：[jwt.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/jwt.ts#L16-L26)

```typescript
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

### 2.3 加密算法

**核心代码**：[crypto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/crypto.ts)

使用 AES-256-GCM 算法，带认证标签（TAG）验证。

### 2.4 认证校验流程

**核心代码**：[auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/auth.ts#L17-L60)

```typescript
export async function checkAuth(request: Request) {
  const token = getBearerToken(request);
  const payload = parseSecureToken(token, secret());
  const shareToken = await parseShareToken(request);

  let user = null;
  const { userId, authKey } = payload || {};

  if (userId) {
    user = await getUser(userId);
  } else if (redis.enabled && authKey) {
    const key = await redis.client.get(authKey);
    if (key?.userId) {
      user = await getUser(key.userId);
    }
  }
  // ...
}
```

### 2.5 Share Token 校验

**核心代码**：[auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/auth.ts#L80-L87)

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

### ⚠️ 容易漏掉的异常分支

1. **Token 解密失败静默返回 null**：`parseSecureToken` 和 `parseToken` 捕获所有异常返回 `null`，无法区分是签名无效、过期还是解密失败
2. **Auth Key 过期无感知**：Redis 中 authKey 过期后，用户不会收到明确的过期提示
3. **Share Token 上下文校验缺失**：仅检查 `SHARE_CONTEXT_HEADER` 是否存在，未校验上下文与 token 的匹配关系
4. **JWT 未指定过期时间**：`createToken` 调用时未强制设置 `expiresIn`，存在长期有效 token 风险
5. **加密与 JWT 使用同一密钥**：`secret()` 同时用于加密和 JWT 签名，密钥用途不分离
6. **Token 刷新机制缺失**：没有 token 刷新机制，过期后需重新登录
7. **GCM 认证标签验证错误无区分**：`decrypt` 中 `decipher.setAuthTag(tag)` 失败与解密失败混为一谈

---

## 3. 防重放

### 3.1 Session / Visit ID 机制

**核心代码**：[send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L147-L175)

```typescript
const sessionId = id ? uuid(sourceId, id) : uuid(sourceId, ip, userAgent, sessionSalt);
let visitId = cache?.visitId || uuid(sessionId, visitSalt);
let iat = cache?.iat || now;

// Expire visit after 30 minutes
if (!timestamp && now - iat > 1800) {
  visitId = uuid(sessionId, visitSalt);
  iat = now;
}
```

### 3.2 Cache Token 防重放

**核心代码**：[send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L100-L122)

```typescript
if (websiteId) {
  const cacheHeader = request.headers.get('x-umami-cache');
  if (cacheHeader) {
    const result = await parseToken(cacheHeader, secret());
    if (result) {
      cache = result;
    }
  }
  // ...
}
```

### 3.3 盐值轮换

**核心代码**：[crypto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/crypto.ts#L72-L78)

```typescript
export function getSalt(saltRotation: string, createdAt: Date): string {
  return hash(
    (saltRotation === 'day' ? startOfDay : saltRotation === 'week' ? startOfWeek : startOfMonth)(
      createdAt,
    ).toUTCString(),
  );
}
```

### 3.4 Visit 过期机制

- Visit 有效期：30 分钟（1800 秒）
- 过期后重新生成 visitId 和 iat
- 带 timestamp 的请求跳过期校验

### ⚠️ 容易漏掉的异常分支

1. **带 timestamp 的请求跳过期校验**：`!timestamp` 条件下才判断过期，攻击者可通过设置旧 timestamp 绕过 visit 过期检查
2. **Cache Token 无过期校验**：`parseToken` 仅验证签名，不校验 token 本身是否过期（JWT 未设置 exp）
3. **盐值轮换边界问题**：盐值按天/周/月轮换，轮换时刻可能导致同一用户 sessionId 变化，统计数据断裂
4. **SessionId 生成依赖客户端信息**：基于 IP + UserAgent + 盐值生成，用户更换网络/浏览器后 sessionId 会变，无法真正防重放
5. **VisitId 基于小时盐值**：整点切换时 visitId 会变化，可能导致同一会话被拆分为多个 visit
6. **无请求唯一标识**：没有 nonce 或 request ID 机制，无法防止完全相同的请求被重复处理
7. **重放攻击窗口**：30 分钟 visit 有效期内，同一请求可重复提交

---

## 4. 节流

### 4.1 Redis 限流

**核心代码**：[redis.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/redis.ts#L72-L82)

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

### 4.2 限流算法分析

使用 **固定窗口计数器** 算法：
- INCR 递增计数器
- 首次请求设置过期时间
- 返回是否达到限制

### 4.3 IP 黑名单

**核心代码**：[detect.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/detect.ts#L140-L169)

```typescript
export function hasBlockedIp(clientIp: string) {
  const ignoreIps = process.env.IGNORE_IP;
  // ... 支持单个 IP 和 CIDR 网段
}
```

### ⚠️ 容易漏掉的异常分支

1. **固定窗口临界问题**：窗口切换瞬间可能通过双倍请求（如 59 秒和 0 秒各发 limit 个）
2. **INCR 与 EXPIRE 非原子**：如果 INCR 成功后 EXPIRE 失败，key 将永不过期，导致永久限流
3. **限流未应用于核心 API**：`rateLimit` 函数存在但未在 `/api/send` 等核心接口中使用
4. **无降级策略**：Redis 连接失败时如何处理？是放行还是拒绝？目前会抛出异常
5. **限流 Key 设计缺失**：没有统一的限流 key 命名规范，不同接口可能重复或冲突
6. **无滑动窗口支持**：固定窗口精度不足，无法应对突发流量
7. **超限后无延迟惩罚**：仅返回是否超限，没有渐进式延迟或封禁机制

---

## 5. 失败重试

### 5.1 Kafka 消息投递

**核心代码**：[kafka.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/kafka.ts#L66-L91)

```typescript
async function sendMessage(
  topic: string,
  message: Record<string, string | number> | Record<string, string | number>[],
): Promise<RecordMetadata[]> {
  try {
    await connect();
    return producer.send({
      topic,
      messages: Array.isArray(message)
        ? message.map(a => { return { value: JSON.stringify(a) }; })
        : [{ value: JSON.stringify(message) }],
      timeout: SEND_TIMEOUT,
      acks: ACKS,
    });
  } catch (e) {
    console.log('KAFKA ERROR:', serializeError(e));
  }
}
```

### 5.2 Batch API 错误处理

**核心代码**：[batch/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/batch/route.ts)

```typescript
const errors = [];
let index = 0;
for (const data of body) {
  const response = await send.POST(newRequest);
  const responseJson = await response.json();

  if (!response.ok) {
    errors.push({ index, response: responseJson });
  }
  index++;
}
```

### 5.3 数据库写入重试

**核心代码**：[saveEvent.ts](file:///d:/fz/0601-2\solo-dogfeeding\code\54-umami\src\queries\sql\events\saveEvent.ts)

事件保存依赖数据库（PostgreSQL/ClickHouse），但**无内置重试机制**。

### ⚠️ 容易漏掉的异常分支

1. **Kafka 发送失败静默吞错**：catch 块仅打日志，不向上层抛出，调用方无法感知失败
2. **Kafka 无重试配置**：`acks: 1` 仅等待 leader 确认，无重试次数和退避策略
3. **连接失败无降级**：Kafka 不可用时，是否回退到直接写入 ClickHouse？目前没有
4. **Batch 部分失败无补偿**：批量处理中部分失败，仅记录错误，没有自动重试机制
5. **数据库写入失败无重试**：saveEvent 失败直接抛出，没有重试逻辑
6. **失败消息无持久化**：Kafka 发送失败后，消息丢失，没有本地存储重试机制
7. **重试风暴风险**：如果后续添加重试，需注意重试间隔和指数退避，否则可能加剧故障
8. **无死信队列**：多次重试失败的消息没有进入死信队列的机制

---

## 6. 外部通知格式

### 6.1 数据收集 API 请求格式

**核心代码**：[send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L24-L64)

```typescript
const schema = z.object({
  type: z.enum(['event', 'identify', 'performance']),
  payload: z.object({
    website: z.uuid().optional(),
    link: z.uuid().optional(),
    pixel: z.uuid().optional(),
    data: anyObjectParam.optional(),
    hostname: z.string().max(100).optional(),
    // ... 更多字段
  }),
});
```

### 6.2 数据收集 API 响应格式

**成功响应**：
```json
{
  "cache": "eyJhbGciOiJIUzI1NiIs...",
  "sessionId": "uuid",
  "visitId": "uuid"
}
```

**错误响应**：[response.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/response.ts)

```json
{
  "error": {
    "message": "Bad request",
    "code": "bad-request",
    "status": 400
  }
}
```

### 6.3 Kafka 消息格式

**核心代码**：[saveEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEvent.ts#L216-L257)

```typescript
const message = {
  website_id: websiteId,
  session_id: sessionId,
  visit_id: visitId,
  event_id: eventId,
  url_path: urlPath,
  event_type: eventType,
  event_name: eventName,
  created_at: getUTCString(createdAt),
  // ... 更多字段（snake_case 命名）
};
```

### 6.4 Batch API 响应格式

```json
{
  "size": 10,
  "processed": 8,
  "errors": 2,
  "details": [
    { "index": 2, "response": { "error": { ... } } }
  ],
  "cache": "eyJhbGciOiJIUzI1NiIs..."
}
```

### 6.5 签名请求头

- **认证**：`Authorization: Bearer <secure_token>`
- **Share Token**：`x-umami-share-token: <jwt_token>`
- **Share Context**：`x-umami-share-context`
- **Cache Token**：`x-umami-cache: <jwt_token>`

### ⚠️ 容易漏掉的异常分支

1. **字段截断无声失败**：`urlPath?.substring(0, URL_LENGTH)` 等截断操作无日志，数据丢失不可感知
2. **Kafka 与数据库字段命名不一致**：Kafka 使用 snake_case（`website_id`），Prisma 使用 camelCase（`websiteId`），转换遗漏风险
3. **空值处理不一致**：部分字段用 `null`，部分用 `undefined`，序列化后行为不同
4. **时间格式不统一**：ClickHouse 用 UTC 字符串，PostgreSQL 用 Date 对象，边界时区问题
5. **响应无版本号**：API 响应格式没有版本标识，后续变更可能破坏兼容性
6. **错误码不完整**：只有基础的 bad-request/unauthorized/forbidden/not-found/server-error，缺少业务错误码
7. **Batch 响应 cache 语义模糊**：`cache ??= responseJson.cache` 只取第一个成功的 cache，不保证是最新的
8. **无 Webhook 标准格式**：项目目前没有定义 Webhook 通知的标准 payload 格式，如告警触发/恢复的结构

---

## 7. 整体协作流程与风险点

### 7.1 数据流示意图

```
客户端 tracker
    ↓ (HTTP POST + x-umami-cache)
/api/send 端点
    ├─ 签名校验 (JWT + 加密)
    ├─ 防重放 (sessionId/visitId 生成)
    ├─ 数据验证 (Zod schema)
    └─ 事件保存
        ├─ Kafka → (无重试) → ClickHouse 消费者
        └─ 直接写入 PostgreSQL/ClickHouse
```

### 7.2 关键协作风险

| 环节 | 协作接口 | 风险点 | 严重程度 |
|------|----------|--------|----------|
| 阈值匹配 → 告警触发 | Goal 报告无阈值配置 | 无法自动触发告警 | 高 |
| 签名校验 → 权限判断 | checkAuth 返回 null | 无法区分失败原因 | 中 |
| 防重放 → 数据统计 | salt 轮换边界 | 数据统计不准 | 中 |
| 节流 → 请求处理 | rateLimit 未使用 | 可能被流量攻击 | 高 |
| 失败重试 → 数据一致性 | Kafka 失败静默 | 数据丢失 | 高 |
| 外部格式 → 消费方 | 字段命名不一致 | 集成困难 | 中 |

### 7.3 最容易漏掉的异常分支 Top 5

1. **Kafka 发送失败静默吞错**：消息丢失但无任何告警，最危险
2. **带 timestamp 的请求跳过期校验**：可被利用绕过防重放机制
3. **INCR 与 EXPIRE 非原子操作**：Redis 限流可能永久失效
4. **JWT Token 无过期时间**：token 泄露后长期有效
5. **字段截断无声失败**：URL/标题超长被截断但无日志

---

## 8. 改进建议

### 8.1 阈值匹配
- 为 Goal 报告增加告警阈值配置
- 支持按网站/用户自定义性能阈值
- 边界值处理文档化

### 8.2 签名校验
- 强制 JWT 设置过期时间
- 区分不同类型的校验失败（返回具体错误码）
- 加密密钥与签名密钥分离

### 8.3 防重放
- 引入 nonce 机制，请求唯一标识去重
- 修复 timestamp 绕过问题
- 使用滑动窗口替代固定窗口

### 8.4 节流
- 将 rateLimit 应用于核心收集接口
- 使用 Lua 脚本保证 INCR+EXPIRE 原子性
- Redis 故障时降级策略

### 8.5 失败重试
- Kafka 发送失败抛出异常或回调通知
- 添加指数退避重试机制
- 引入死信队列
- 本地持久化失败消息

### 8.6 外部通知格式
- 定义标准 Webhook 告警格式（含告警类型、级别、详情、恢复通知）
- API 版本化管理
- 统一字段命名规范
- 添加请求/响应 Trace ID 便于排查
