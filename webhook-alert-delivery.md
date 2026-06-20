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

| 代码 | 实际用途 | 非告警/Webhook 原因 |
|------|----------|-------------------|
| [kafka.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/kafka.ts) | **内部数据管道**：将事件异步投递到 Kafka | 是**Inbound 数据写入** producer；仓库内无 consumer，ClickHouse schema 中无 Kafka Engine；消费者在项目外部（如独立的 ClickHouse Connector 或另一个服务） |
| [/api/send](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts) | **客户端数据上报入口**：接收 tracker 脚本的页面事件 | 是接收浏览器端数据，不是向外部发送告警 |
| [/api/record](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/record/route.ts) | **Session Replay 数据上报入口** | 接收浏览器端录制数据，不是向外部发送告警 |
| [getGoal.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/reports/getGoal.ts) | **目标转化统计报告** | 仅在用户主动查询时返回数据，无定时评估和自动触发机制 |
| [WEB_VITALS_THRESHOLDS](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/constants.ts#L106-L112) | **前端性能评级常量** | 仅用于 UI 展示颜色区分，不触发任何告警或通知 |
| [telemetry.js](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/scripts/telemetry.js) | **Umami 官方匿名遥测** | 向 `api.umami.is` 上报版本/平台信息，不是用户可配置的 Webhook |
| [version.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/store/version.ts) | **Umami 版本更新检查** | 向 `api.umami.is/v1/updates` 检查新版本，不是用户告警通道 |

---

## 1. 告警与 Webhook 缺失后的真实边界

### 1.1 缺失边界全景图

```
浏览器端                    服务端                              外部系统
─────────                  ──────                            ────────
tracker.js  ──→ /api/send   ──→ saveEvent      ──→ Kafka (producer only)
recorder.js ──→ /api/record ──→ saveRecording  ──→ Kafka (producer only)
                                     │
                                     ├─→ PostgreSQL (Prisma)
                                     └─→ ClickHouse (直写 fallback)

                              ❌ 无告警评估引擎
                              ❌ 无阈值触发器
                              ❌ 无通知路由
                              ❌ 无 Webhook dispatcher
                              ❌ 无出向 HTTP POST (除了遥测和版本检查)
                              ❌ 无投递日志 / 重试 / 死信
```

当前系统只有**数据流入 → 存储**管道，**完全没有数据流出 → 通知**管道（除了下面描述的两种固定出向通信）。

### 1.2 现有"出向"通信的真实边界

项目中仅有以下两种服务端对外 HTTP 通信，但**都不是用户可配置的 Webhook**：

| 通信 | 方向 | 目标 | 触发条件 | 失败处理 |
|------|------|------|----------|----------|
| 构建遥测上报 | 出向 | `https://api.umami.is/v1/telemetry` | postbuild 时触发 | 空 catch 完全静默 |
| 页面遥测像素 | 出向（浏览器端） | `https://telemetry.umami.is` | production + 非 share 路径 | `<img>` 标签加载，浏览器静默失败 |
| 版本更新检查 | 出向（浏览器端） | `https://api.umami.is/v1/updates` | admin 登录 + production | `res.ok` 为 false 时返回 null，静默退出 |

---

## 2. 遥测上报（Telemetry）代码分析

### 2.1 构建时遥测（服务端）

**代码位置**：[postbuild.js](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/scripts/postbuild.js) → [telemetry.js](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/scripts/telemetry.js)

```javascript
const url = 'https://api.umami.is/v1/telemetry';

export async function sendTelemetry(type) {
  // ...
  try {
    await fetch(url, {
      method: 'post',
      cache: 'no-cache',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
  } catch {
    // Ignore
  }
}
```

**载荷**：`{ type: 'build', payload: { version, node, platform, arch, os, is_docker, is_ci } }`

**禁用条件**：`process.env.DISABLE_TELEMETRY` 存在时不触发

**失败处理事实**：
- `catch {}` 空块，完全静默
- 不记录错误日志
- 不影响构建结果（构建不会因遥测失败而失败）
- 无重试

### 2.2 运行时页面遥测（浏览器端）

**代码位置**：[telemetry/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/scripts/telemetry/route.ts)

```typescript
export async function GET() {
  if (
    process.env.NODE_ENV !== 'production' ||
    process.env.DISABLE_TELEMETRY ||
    process.env.PRIVATE_MODE
  ) {
    return new Response('/* telemetry disabled */', { headers: { 'content-type': 'text/javascript' } });
  }

  const script = `
    (()=>{const i=document.createElement('img');
      i.setAttribute('src','${TELEMETRY_PIXEL}?v=${CURRENT_VERSION}');
      i.setAttribute('style','width:0;height:0;position:absolute;pointer-events:none;');
      document.body.appendChild(i);})();
  `;
  return new Response(script.replace(/\s\s+/g, ''), { ... });
}
```

**加载位置**：[App.tsx#L70-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/(main)/App.tsx#L70-L72)

```tsx
{process.env.NODE_ENV === 'production' && !pathname.includes('/share/') && (
  <Script src={`${process.env.basePath || ''}/telemetry.js`} />
)}
```

**失败处理事实**：
- 使用 `<img>` 标签加载 1×1 像素，浏览器不对外暴露加载失败
- 无错误处理
- 无重试
- 仅 production + 非 share 路径 + 非 PRIVATE_MODE + 非 DISABLE_TELEMETRY 时生效

### 2.3 遥测与告警/Webhook 的边界

| 维度 | 遥测 | Webhook（需实现） |
|------|------|-------------------|
| **发起方** | 构建脚本 / 浏览器像素 | 告警引擎（服务端） |
| **目标 URL** | 硬编码为 `api.umami.is` / `telemetry.umami.is` | 用户可配置的任意 URL |
| **载荷内容** | 版本/平台元数据 | 告警详情（指标、阈值、当前值） |
| **触发机制** | postbuild 自动 / 页面加载时 | 指标越界时 |
| **失败处理** | 完全静默 | 必须记录日志、重试、死信队列 |

---

## 3. 版本检查（Version Check）代码分析

**代码位置**：[version.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/store/version.ts)

```typescript
export async function checkVersion() {
  const { current } = store.getState();

  const data = await fetch(`${UPDATES_URL}?v=${current}`, {
    method: 'GET',
    headers: { Accept: 'application/json' },
  }).then(res => {
    if (res.ok) {
      return res.json();
    }
    return null;
  });

  if (!data) {
    return;  // ← 数据为 null 时直接静默退出
  }
  // ... 更新 store
}
```

**触发位置**：[UpdateNotice.tsx#L39-L43](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/(main)/UpdateNotice.tsx#L39-L43)

**失败处理事实**：
- fetch 网络异常：由 `await fetch(...).then(...)` 链处理——fetch 抛异常时 Promise 会 reject，由于没有 `.catch()`，异常会被**静默吞掉**（Zustand action 内的未捕获异常不会崩溃应用）
- HTTP 非 2xx：`res.ok` 为 false，显式返回 `null`，`if (!data) return` 静默退出
- 两种失败情况**都不会在 UI 上提示用户**，也不会打日志
- 无重试
- `VERSION_CHECK` localStorage 项用于去重，但代码中未见写入逻辑

---

## 4. Session Replay 投递代码分析

### 4.1 录制数据上报入口

**代码位置**：[/api/record](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/record/route.ts)

```typescript
const schema = z.object({
  type: z.literal('record'),
  payload: z.object({
    website: z.uuid(),
    events: z.array(z.any()).max(200),     // 最多 200 条/请求
    timestamp: z.coerce.number().int().optional(),
  }),
});
```

**认证方式**：`skipAuth: true`，但**强制要求 `x-umami-cache` header**

```typescript
const cacheHeader = request.headers.get('x-umami-cache');
if (!cacheHeader) {
  return badRequest({ message: 'Missing session token.' });
}
const cache = (await parseToken(cacheHeader, secret())) as Cache | null;
if (!cache?.sessionId || !cache?.visitId) {
  return badRequest({ message: 'Invalid session token.' });
}
```

### 4.2 录制功能启用检查

```typescript
if (!website.replayEnabled) {
  return json({ ok: false, reason: 'replay_disabled' });
}

// Cloud 模式下需要 Business 订阅
if (process.env.CLOUD_MODE) {
  const account = website.teamId
    ? await fetchTeam(website.teamId)
    : website.userId ? await fetchAccount(website.userId) : null;
  if (!account?.isBusiness && !account?.isNoBilling) {
    return forbidden({ message: 'Business subscription required.' });
  }
}
```

### 4.3 录制数据存储

**代码位置**：[saveRecording.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/replays/saveRecording.ts)

**Prisma 路径**：gzip 压缩后写入 `sessionReplay` 表

```typescript
const compressed = gzipSync(Buffer.from(JSON.stringify(events), 'utf-8'));
return prisma.client.sessionReplay.create({
  data: { id: uuid(), websiteId, sessionId, visitId, chunkIndex,
          events: compressed as any, eventCount, startedAt, endedAt },
});
```

**ClickHouse 路径**：通过 Kafka `session_replay` topic 或直接 insert

```typescript
if (kafka.enabled) {
  return sendMessage('session_replay', message);
}
return insert('session_replay', [message]);
```

### 4.4 Replay 与告警/Webhook 的边界

Replay 数据流是**浏览器 → 服务端 → 存储**，方向与 Webhook 完全相反。

---

## 5. Kafka 多 Topic 事件投递关系

### 5.1 关键事实：仓库内只有 Producer，无 Consumer

经过完整搜索：
- **全项目无 `consumer` / `kafka.consumer()` / `kafka.consume` 代码**
- [ClickHouse schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/db/clickhouse/schema.sql) 中所有表均为 `MergeTree` / `ReplacingMergeTree` / `AggregatingMergeTree`，**无 `ENGINE = Kafka` 表**
- 消费者逻辑必然在本仓库之外（独立服务或 ClickHouse Connector 部署）

### 5.2 Topic 全景（4 个 Producer 写入 Topic）

| Topic | 写入方（Producer） | 目标存储（Consumer 在仓库外） |
|-------|-------------------|---------------------------|
| `event` | [saveEvent.ts#L260](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEvent.ts#L260) | `website_event` (MergeTree) |
| `event_data` | [saveEventData.ts#L75](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEventData.ts#L75) | `event_data` (MergeTree) |
| `session_data` | [saveSessionData.ts#L100](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/sessions/saveSessionData.ts#L100) | `session_data` (ReplacingMergeTree) |
| `session_replay` | [saveRecording.ts#L79](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/replays/saveRecording.ts#L79) | `session_replay` (MergeTree, ZSTD(3)) |

### 5.3 写入链路图

```
/api/send (type=event|identify|performance)
    │
    ├─ saveEvent()
    │   ├─ kafka.enabled=true  → sendMessage('event', message)
    │   └─ kafka.enabled=false → insert('website_event', [message])
    │
    └─ saveEventData()  [当 eventData 非空时]
        ├─ kafka.enabled=true  → sendMessage('event_data', messages)
        └─ kafka.enabled=false → insert('event_data', messages)
              │
              └─ saveRevenue()  [当 revenue > 0 且 currency 非空时]
                  └─ 直接写入（无 Kafka 路径）

/api/record (type=record)
    │
    └─ saveRecording()
        ├─ kafka.enabled=true  → sendMessage('session_replay', message)
        └─ kafka.enabled=false → insert('session_replay', [message])

/api/send (type=identify)
    │
    └─ saveSessionData()
        ├─ kafka.enabled=true  → sendMessage('session_data', messages)
        └─ kafka.enabled=false → insert('session_data', messages)
```

### 5.4 Topic 间依赖关系

| 依赖 | 说明 | 风险 |
|------|------|------|
| `event` → `event_data` 串行 | 两条独立 sendMessage 调用 | `event` 成功但 `event_data` 失败 → 有事件记录无自定义属性 |
| `event_data` → `revenue` 串行（仅 Prisma 路径） | revenue 无 Kafka 路径 | Kafka 模式下 revenue 可能与 event_data 不同步 |
| `session_data` ClickHouse 直接 insert | 无去重 | 同一 session 的 data 可能重复写入（但 `session_data` 表是 ReplacingMergeTree，有最终一致性去重） |

---

## 6. 失败重试：KafkaJS 默认传输行为 vs 应用层无死信/回退

### 6.1 Kafka 连接与 Producer 配置

**代码位置**：[kafka.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/kafka.ts#L1-L86)

```typescript
const CONNECT_TIMEOUT = 5000;
const SEND_TIMEOUT = 3000;
const ACKS = 1;

const client: Kafka = new Kafka({
  clientId: 'umami',
  brokers: brokers,
  connectionTimeout: CONNECT_TIMEOUT,
  logLevel: logLevel.ERROR,
  ...ssl,
});

// producer.send() 参数
return producer.send({
  topic,
  messages: [...],
  timeout: SEND_TIMEOUT,  // 3000
  acks: ACKS,              // 1
});
```

### 6.2 KafkaJS v2.1.0 的默认重试配置（项目未显式修改）

项目使用 `kafkajs@^2.1.0`（见 [package.json#L90](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/package.json#L90)）。项目**没有**传入任何 `retry` 配置给 `new Kafka()` 或 `producer.send()`，因此使用 KafkaJS 的默认值：

| KafkaJS 配置项 | 默认值（KafkaJS v2） | 项目是否显式设置 |
|----------------|---------------------|-----------------|
| `retry.retries` | 5 | ❌ 未设置 → 使用默认 5 |
| `retry.initialRetryTime` | 300 (ms) | ❌ 未设置 → 使用默认 300ms |
| `retry.maxRetryTime` | 30000 (ms) | ❌ 未设置 → 使用默认 30s |
| `retry.multiplier` | 2 | ❌ 未设置 → 使用默认 2（指数退避） |
| `retry.factor` | 0 | ❌ 未设置 → 使用默认 |
| `acks` | -1 (all ISR) | ✅ 设为 `1` → 只等 leader 确认 |
| `timeout` | 30000 | ✅ 设为 `3000` → 3 秒超时 |
| `idempotent` | false | ❌ 未设置 → 默认 false |

> **澄清**：项目**没有显式禁用重试**，而是保留了 KafkaJS v2.1.0 的默认 `retries: 5` + 指数退避。同时通过 `acks: 1` 和 `timeout: 3000` 调整了可靠性和等待时长。

### 6.3 KafkaJS 传输层重试覆盖范围

默认 5 次传输层重试可以覆盖：
- ✅ Broker 短暂不可达（网络抖动）
- ✅ Leader 切换期间的 NOT_LEADER_FOR_PARTITION
- ✅ 连接超时（在 5 次重试窗口内可恢复）
- ✅ 超过 `request.required.acks` 副本数的场景

**不覆盖**（会抛异常进入应用层 catch）：
- ❌ Topic 不存在（auto.create=false 时）
- ❌ 消息体过大（MessageSizeTooLarge）
- ❌ Producer 连接失败且重试耗尽
- ❌ Kafka 全集群不可用
- ❌ 认证失败（SASL/SSL 错误）
- ❌ 消息序列化失败

### 6.4 应用层无死信队列 / 无降级回退

**代码位置**：[kafka.ts#L66-L91](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/kafka.ts#L66-L91)

```typescript
async function sendMessage(topic, message) {
  try {
    await connect();
    return producer.send({ topic, messages: [...], timeout: SEND_TIMEOUT, acks: ACKS });
  } catch (e) {
    // eslint-disable-next-line no-console
    console.log('KAFKA ERROR:', serializeError(e));
    // ← 只打日志，不做更多处理
  }
}
```

**应用层缺失清单**：

| 缺失项 | 说明 | 影响 |
|--------|------|------|
| 不向上层抛异常 | `catch` 内无 `throw`，调用方 `await sendMessage(...)` 收到 `undefined` 以为成功 | 数据静默丢失 |
| 无死信队列（DLQ） | 重试耗尽的消息不写入持久化存储 | 无法事后追回 |
| 无降级回退 | Kafka 模式下即使 sendMessage 完全失败，也不会执行 `insert()` 直写 ClickHouse | 数据断层 |
| 无投递日志 | 成功/失败无数据库记录 | 无法审计和排查 |
| 无人工重放机制 | 无 UI/API 可以重新投递失败消息 | 故障恢复后需手动补数 |

### 6.5 影响范围

4 个 topic 的写入方都经过 `sendMessage()`，因此上述缺陷**影响所有数据类型**：

| 调用方 | Topic | 失败后果 |
|--------|-------|----------|
| [saveEvent.clickhouseQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEvent.ts#L260) | `event` | 页面浏览/自定义事件静默丢失 |
| [saveEventData.clickhouseQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEventData.ts#L75) | `event_data` | 事件自定义属性静默丢失 |
| [saveSessionData.clickhouseQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/sessions/saveSessionData.ts#L100) | `session_data` | 用户属性（identify）静默丢失 |
| [saveRecording.clickhouseQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/replays/saveRecording.ts#L79) | `session_replay` | 录制数据静默丢失 |

---

## 7. 签名验证：Share Token 权限匹配

### 7.1 Share Token 完整链路

```
1. 用户访问 /share/:slug
2. 前端 GET /api/share/:slug
3. 后端查 Prisma `share` 表，根据 shareType 组装 payload
   ├─ website:   { websiteId, shareId, shareType, parameters }
   ├─ pixel:     { websiteId, pixelId, ... }
   ├─ link:      { websiteId, linkId, ... }
   └─ board:     { boardId, websiteIds[], pixelIds[], linkIds[], ... }
4. createToken(payload, secret()) → 生成裸 JWT（无加密，无 expiresIn）
5. 前端存入 Zustand store (shareToken.token)
6. 每次请求携带两个 header:
   ├─ x-umami-share-token: <jwt>
   └─ x-umami-share-context: '1'   (仅当 pathname.startsWith('/share'))
7. checkAuth() 用 parseToken() 验证 JWT 签名，得到 shareToken 对象
8. 各权限函数用 shareToken 中的 ID 匹配资源
```

**Token 生成代码**：[share/[slug]/route.ts#L94](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/share/[slug]/route.ts#L94)

```typescript
data.token = createToken(data, secret());
```

**前端发送代码**：[useApi.ts#L25-L28](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/components/hooks/useApi.ts#L25-L28)

```typescript
const shareHeaders =
  isSharePath && shareToken?.token
    ? { [SHARE_TOKEN_HEADER]: shareToken.token, [SHARE_CONTEXT_HEADER]: '1' }
    : {};
```

### 7.2 checkAuth 中的 Share Token 处理

**代码位置**：[auth.ts#L17-L60](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/auth.ts#L17-L60)

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

  // 分支 1：既没有 user 也没有 shareToken → 拒绝
  if (!user?.id && !shareToken) {
    return null;
  }

  // 分支 2：没有 user 但有 shareToken → 必须有 share context header
  if (!user?.id && shareToken) {
    const shareContext = request.headers.get(SHARE_CONTEXT_HEADER);
    if (!shareContext) {
      return null;
    }
  }

  return { token, authKey, shareToken, user };
}
```

### 7.3 权限函数中的 Share Token 匹配

4 个权限文件都只在 **View（读）操作**中检查 shareToken，**写操作（Update/Delete/Create）只检查 user**：

| 权限文件 | 读函数（检查 shareToken） | 写函数（只检查 user） | Share Token 匹配字段 |
|----------|-------------------------|---------------------|---------------------|
| [website.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/website.ts) | `canViewWebsite` | `canCreateWebsite` / `canUpdateWebsite` / `canDeleteWebsite` | `websiteId` / `pixelId` / `linkId` / `websiteIds[]` / `pixelIds[]` / `linkIds[]` |
| [board.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/board.ts) | `canViewBoard` | `canCreateBoard` / `canUpdateBoard` / `canDeleteBoard` | `boardId` |
| [link.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/link.ts) | `canViewLink` | `canCreateLink` / `canUpdateLink` / `canDeleteLink` | `linkId` / `websiteId` / `linkIds[]` |
| [pixel.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/pixel.ts) | `canViewPixel` | `canCreatePixel` / `canUpdatePixel` / `canDeletePixel` | `pixelId` / `websiteId` / `pixelIds[]` |

**以 `canViewWebsite` 为例**：[website.ts#L7-L21](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/website.ts#L7-L21)

```typescript
export async function canViewWebsite({ user, shareToken }: Auth, websiteId: string) {
  if (user?.isAdmin) {
    return true;
  }

  if (
    shareToken?.websiteId === websiteId ||
    shareToken?.pixelId === websiteId ||
    shareToken?.linkId === websiteId ||
    shareToken?.websiteIds?.includes(websiteId) ||
    shareToken?.pixelIds?.includes(websiteId) ||
    shareToken?.linkIds?.includes(websiteId)
  ) {
    return true;
  }

  // 后续查数据库匹配 userId / teamId（需要 user 存在）...
}
```

### 7.4 写操作不检查 shareToken——是设计事实，不是风险

以 `canUpdateWebsite` 为例：[website.ts#L58-L84](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/website.ts#L58-L84)

```typescript
export async function canUpdateWebsite({ user }: Auth, websiteId: string) {
  // ← 参数解构中没有 shareToken
  if (!user) {
    return false;
  }
  // ... 只检查 user.isAdmin / userId / teamUser.role
}
```

这是**有意的安全设计**：分享链接只授予只读权限，不能通过分享链接修改数据。

### 7.5 真实的 Share Token 权限风险（排除 JWT 已保证和写操作需 user 的部分）

| # | 风险 | 代码位置 | 说明 |
|---|------|----------|------|
| 1 | **Token 无过期时间** | [share/[slug]/route.ts#L94](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/share/[slug]/route.ts#L94) | `createToken(data, secret())` 未传 `expiresIn`，JWT 永不过期；分享链接被撤销（删除 `share` 记录）后，已签发的 token 仍然有效 |
| 2 | **SHARE_CONTEXT_HEADER 只检查非空，不校验值** | [auth.ts#L43-L48](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/auth.ts#L43-L48) | 只要 header 存在（`if (!shareContext)`），任意值（如 `'0'` / `'hacked'`）都通过；本意是防止在非 share 页面使用 share token，但检查过于宽松 |
| 3 | **Share Token 与 Share 记录无关联校验** | 所有权限函数 | token 中的 `websiteId` 是签发时从 share.entityId 拷贝的，但后续权限校验**只比对 token 内的 ID**，不再查 `share` 表确认该分享链接是否仍存在/未过期。分享被删除或修改后，旧 token 仍然生效 |
| 4 | **pixelId/linkId 与 websiteId 同字段比较可能产生 UUID 碰撞** | [website.ts#L13-L18](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/website.ts#L13-L18) | `shareToken?.pixelId === websiteId` 等比较：pixel / link / website 都是 UUID，如果恰好生成相同值（极低概率），会产生越权 |
| 5 | **JWT payload 无签名外的完整性校验字段** | [share/[slug]/route.ts#L94](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/share/[slug]/route.ts#L94) | JWT 标准签名已保证 payload 不被篡改（这是 JWT 本身保证的，不额外算作风险）；但 token 中没有 `shareId` 字段，无法关联到具体分享记录做状态校验 |

---

## 8. 现有代码分析：阈值匹配（非告警场景）

### 8.1 Web Vitals 性能评级阈值

**定位**：前端 UI 展示用评级常量，不是告警触发阈值

**代码位置**：[constants.ts#L106-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/constants.ts#L106-L112)

```typescript
export const WEB_VITALS_THRESHOLDS = {
  lcp:  { good: 2500, poor: 4000, unit: 'ms' },
  inp:  { good:  200, poor:  500, unit: 'ms' },
  cls:  { good:  0.1, poor: 0.25, unit: ''   },
  fcp:  { good: 1800, poor: 3000, unit: 'ms' },
  ttfb: { good:  800, poor: 1800, unit: 'ms' },
} as const;
```

**匹配逻辑**：[PerformanceCard.tsx#L26-L32](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/components/metrics/PerformanceCard.tsx#L26-L32)

```typescript
function getRating(metric: string, value: number): 'good' | 'needs-improvement' | 'poor' {
  const threshold = WEB_VITALS_THRESHOLDS[metric as keyof typeof WEB_VITALS_THRESHOLDS];
  if (!threshold || value <= 0) return 'good';
  if (value <= threshold.good) return 'good';
  if (value <= threshold.poor) return 'needs-improvement';
  return 'poor';
}
```

**异常分支**：
1. `value <= 0` 直接返回 `'good'` —— 数据异常被掩盖
2. 边界值 `value === threshold.poor` 归属 `needs-improvement` 而非 `poor` —— 与语义期望不符
3. 硬编码常量，不支持按网站自定义

### 8.2 Goal 转化目标阈值（无告警代码）

[getGoal.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/reports/getGoal.ts) 返回 `{ num, total }`，前端画进度条，但**没有"转化率低于 X% 即告警"的配置入口和评估逻辑**。

---

## 9. 现有代码分析：防重放（数据采集场景）

### 9.1 SessionId / VisitId 机制

**代码位置**：[send/route.ts#L143-L175](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L143-L175)

- SessionId：基于 `sourceId + IP + UA + 盐值` 确定性生成
- 盐值轮换：[crypto.ts#L72-L78](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/crypto.ts#L72-L78) 按月/周/日轮换
- VisitId：30 分钟过期

**异常分支**：带 `timestamp` 参数时跳过 `!timestamp` 过期判断，可复用旧 visitId；Cache Token 裸 JWT 无过期。

---

## 10. 现有代码分析：节流（Redis RateLimit 未实际应用）

**代码位置**：[redis.ts#L72-L82](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/redis.ts#L72-L82)

`rateLimit` 函数存在但**从未被任何 API 调用**。实际生效的防线是 IP 黑名单（[detect.ts#L140-L169](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/detect.ts#L140-L169)）和 `isbot` 机器人检测。

---

## 11. 缺失部分清单

### 11.1 告警规则模块（缺失）

| 缺失项 | 说明 | 建议实现位置 |
|--------|------|-------------|
| 告警规则数据模型 | 规则定义（名称、指标、阈值、比较符、评估周期、通知渠道） | `prisma/schema.prisma` 新增 `AlertRule` |
| 告警实例数据模型 | 触发记录（规则ID、触发时间、当前值、阈值、状态：触发/恢复） | `prisma/schema.prisma` 新增 `AlertIncident` |
| 规则评估引擎 | 定时拉指标 → 阈值比较 → 判断触发/恢复 | 新增 `src/lib/alert/engine.ts` |
| 规则 CRUD API | 创建/编辑/删除/启停 | 新增 `src/app/api/alerts/route.ts` |
| 告警抑制/静默 | 防止告警风暴 | 状态机 + 静默窗口 |

### 11.2 Webhook 投递模块（缺失）

| 缺失项 | 说明 | 建议实现位置 |
|--------|------|-------------|
| Webhook 配置数据模型 | URL、签名密钥、自定义请求头、订阅类型 | `prisma/schema.prisma` 新增 `WebhookEndpoint` |
| 投递日志数据模型 | 请求体、HTTP 状态、耗时、重试次数 | `prisma/schema.prisma` 新增 `WebhookDeliveryLog` |
| Webhook 投递器 | 构造请求体、HMAC 签名、HTTP POST、超时 | 新增 `src/lib/webhook/dispatcher.ts` |
| 投递重试队列 | 指数退避（max 5 次）+ 死信队列 | Redis List 或数据库轮询表 |
| Webhook 管理 API | 端点 CRUD + 测试发送 | 新增 `src/app/api/webhooks/route.ts` |

### 11.3 两者协作桥接（缺失）

| 缺失项 | 说明 |
|--------|------|
| 告警 → 通知路由 | 告警引擎触发后按规则的通知渠道分发 |
| 通知节流 | 同一告警短时间重复触发时合并/静默 |
| Kafka 告警 topic（可选） | 如走 Kafka 解耦，需新增 `alert_event` + `webhook_delivery` |

---

## 12. 总结

| 项目 | 当前状态 | 关键事实 / 问题 |
|------|----------|----------------|
| **告警规则** | ❌ 不存在 | 无数据表、无评估引擎、无触发 API |
| **Webhook 投递** | ❌ 不存在 | 无端点表、无投递器、无签名与重试 |
| **Kafka Consumer / ClickHouse Kafka Engine** | ❌ 仓库内不存在 | 全项目只有 producer；schema 中所有表均为 MergeTree 系列 |
| **KafkaJS 重试** | ⚠️ 默认 5 次指数退避 | 项目未显式禁用；但 `acks: 1` / `timeout: 3000` 降低了可靠性；应用层 catch 吞错，耗尽后消息丢失 |
| **应用层死信/回退** | ❌ 不存在 | 无 DLQ、无降级直写、无投递日志、无人工重放 |
| **遥测上报失败处理** | ⚠️ 完全静默 | 构建遥测空 catch；页面像素 img 标签无失败回调 |
| **版本检查失败处理** | ⚠️ 完全静默 | fetch 异常或非 2xx 都静默返回 null，无 UI 提示 |
| **Share Token 权限风险** | ⚠️ 4 项真实风险 | Token 无过期、context header 检查过宽、不校验 share 记录状态、跨实体类型 UUID 比较；写操作仅检查 user 是有意设计，不算风险 |
| **阈值匹配** | ⚠️ 仅评级展示 | Web Vitals 有常量但无告警联动；Goal 无阈值配置 |
| **防重放** | ⚠️ 仅数据采集去重 | session/visit 机制，timestamp 可绕过，无请求级 nonce |
| **节流** | ⚠️ 函数存在未用 | rateLimit 没挂到任何 API |
