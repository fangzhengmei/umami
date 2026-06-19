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
| [kafka.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/kafka.ts) | **内部数据管道**：将事件异步投递到 ClickHouse | 是**Inbound 数据写入**，不是对外部系统的 HTTP 回调；Kafka topic 消费者是 umami 自身的 ClickHouse consumer |
| [/api/send](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts) | **客户端数据上报入口**：接收 tracker 脚本的页面事件 | 是接收浏览器端数据，不是向外部发送告警 |
| [/api/record](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/record/route.ts) | **Session Replay 数据上报入口**：接收 recorder 脚本的录制事件 | 是接收浏览器端录制数据，不是向外部发送告警 |
| [getGoal.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/reports/getGoal.ts) | **目标转化统计报告** | 仅在用户主动查询时返回数据，无定时评估和自动触发机制 |
| [WEB_VITALS_THRESHOLDS](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/constants.ts#L106-L112) | **前端性能评级常量** | 仅用于 UI 展示颜色区分，不触发任何告警或通知 |
| [telemetry.js](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/scripts/telemetry.js) | **Umami 官方匿名遥测** | 向 `api.umami.is` 上报版本/平台信息，不是用户可配置的 Webhook |
| [version.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/store/version.ts) | **Umami 版本更新检查** | 向 `api.umami.is/v1/updates` 检查新版本，不是用户告警通道 |

---

## 1. 告警与 Webhook 缺失后的真实边界

### 1.1 缺失边界的全景图

```
浏览器端                    服务端                         外部系统
─────────                  ──────                        ────────
tracker.js ──→ /api/send ──→ saveEvent ──→ Kafka/ClickHouse
recorder.js ──→ /api/record ──→ saveRecording ──→ Kafka/ClickHouse

                              ❌ 无告警评估引擎
                              ❌ 无阈值触发器
                              ❌ 无通知路由
                              ❌ 无 Webhook dispatcher
                              ❌ 无出向 HTTP POST
                              ❌ 无投递日志 / 重试 / 死信
```

当前系统只有**数据流入 → 存储**管道，**完全没有数据流出 → 通知**管道。

### 1.2 现有"出向"通信的真实边界

项目中仅有以下两种服务端对外 HTTP 通信，但**都不是用户可配置的 Webhook**：

| 通信 | 方向 | 目标 | 触发条件 | 失败处理 |
|------|------|------|----------|----------|
| 遥测上报 | 出向 | `https://api.umami.is/v1/telemetry` | 构建时自动触发 | 静默吞错（`catch {}`） |
| 版本检查 | 出向 | `https://api.umami.is/v1/updates` | 管理员登录后前端触发 | 失败返回 `null`，无 UI 提示 |

---

## 2. 遥测上报（Telemetry）代码分析

### 2.1 构建时遥测

**代码位置**：[postbuild.js](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/scripts/postbuild.js) → [telemetry.js](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/scripts/telemetry.js)

```javascript
const url = 'https://api.umami.is/v1/telemetry';

export async function sendTelemetry(type) {
  const data = {
    type,                           // 'build'
    payload: {
      version: pkg.version,
      node: process.version,
      platform: os.platform(),
      arch: os.arch(),
      os: `${os.type()} ${os.version()}`,
      is_docker: isDocker(),
      is_ci: isCI,
    },
  };

  try {
    await fetch(url, { method: 'post', body: JSON.stringify(data) });
  } catch {
    // Ignore
  }
}
```

**禁用条件**：`process.env.DISABLE_TELEMETRY`

### 2.2 运行时页面遥测（1×1 像素追踪）

**代码位置**：[telemetry/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/scripts/telemetry/route.ts)

```typescript
export async function GET() {
  if (process.env.NODE_ENV !== 'production' ||
      process.env.DISABLE_TELEMETRY ||
      process.env.PRIVATE_MODE) {
    return new Response('/* telemetry disabled */', { ... });
  }

  const script = `
    (()=>{const i=document.createElement('img');
      i.setAttribute('src','${TELEMETRY_PIXEL}?v=${CURRENT_VERSION}');
      ...})();
  `;
  return new Response(script, { ... });
}
```

**加载位置**：[App.tsx#L70-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/(main)/App.tsx#L70-L72)

```tsx
{process.env.NODE_ENV === 'production' && !pathname.includes('/share/') && (
  <Script src={`${process.env.basePath || ''}/telemetry.js`} />
)}
```

### 2.3 遥测与告警/Webhook 的边界

| 维度 | 遥测 | Webhook（需实现） |
|------|------|-------------------|
| **发起方** | umami 自身（构建脚本 / 前端像素） | 告警引擎 |
| **目标** | 固定：`api.umami.is` | 用户配置的任意 URL |
| **载荷** | 版本/平台元数据 | 告警详情（指标、阈值、当前值） |
| **失败处理** | 完全静默 | 必须记录、重试、死信 |
| **频率** | 构建一次 + 每次页面加载 | 按告警触发频率 |

**异常分支风险**：

| # | 问题 | 影响 |
|---|------|------|
| 1 | 构建遥测 `catch {}` 完全静默 | 构建失败不会发现是遥测网络问题导致 |
| 2 | 页面遥测通过 `<img>` 标签，无法感知加载失败 | 无法统计遥测覆盖率 |
| 3 | 遥测 URL 硬编码 | 无法通过环境变量自定义遥测端点（如私有部署场景） |

---

## 3. 版本检查（Version Check）代码分析

### 3.1 前端版本检查

**代码位置**：[version.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/store/version.ts)

```typescript
export async function checkVersion() {
  const { current } = store.getState();

  const data = await fetch(`${UPDATES_URL}?v=${current}`, {
    method: 'GET',
    headers: { Accept: 'application/json' },
  }).then(res => {
    if (res.ok) return res.json();
    return null;              // ← 失败直接返回 null
  });

  if (!data) return;         // ← 静默退出

  const hasUpdate = !!(latest && lastCheck?.version !== latest && semver.gt(latest, current));
  // ...
}
```

**触发位置**：[UpdateNotice.tsx#L39-L43](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/(main)/UpdateNotice.tsx#L39-L43)

```tsx
useEffect(() => {
  if (allowUpdate) {
    checkVersion();
  }
}, [allowUpdate]);
```

**显示条件**：
- `NODE_ENV === 'production'`
- `user?.isAdmin`
- `!config?.updatesDisabled && !config?.privateMode`
- 非 share 路径
- 非 cloud 模式

### 3.2 版本检查与告警的边界

版本检查是**拉模型**（前端主动请求），告警通知是**推模型**（服务端主动投递）。两者在触发机制和失败处理上完全不同。

**异常分支风险**：

| # | 问题 | 影响 |
|---|------|------|
| 1 | `fetch` 失败静默返回 `null` | 网络问题导致管理员永远看不到更新提示 |
| 2 | 无本地缓存过期策略 | `VERSION_CHECK` localStorage 永不过期，一旦存储后不再检查 |
| 3 | `UPDATES_URL` 硬编码为 `api.umami.is` | 私有部署无法指向内部更新服务器 |

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

**认证方式**：`skipAuth: true`，但**强制要求 `x-umami-cache` header** 中的 sessionId/visitId

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

**ClickHouse 路径**：通过 Kafka topic `session_replay` 或直接 insert

```typescript
if (kafka.enabled) {
  return sendMessage('session_replay', message);
}
return insert('session_replay', [message]);
```

### 4.4 Replay 与告警/Webhook 的边界

Replay 数据流是**浏览器 → 服务端 → 存储**，方向与 Webhook（服务端 → 外部系统）完全相反。

Replay 的 `events` 字段包含用户行为序列（点击、滚动、输入），理论上可以作为"异常行为检测"的数据源，但当前**没有任何代码消费 replay 事件做告警判断**。

**异常分支风险**：

| # | 问题 | 影响 |
|---|------|------|
| 1 | `/api/record` 的 `x-umami-cache` 用裸 JWT 无过期 | 录制 token 泄露可永久伪造 session 数据 |
| 2 | `events: z.array(z.any()).max(200)` 无内容校验 | 恶意大 payload 可消耗 gzip/存储资源 |
| 3 | `gzipSync` 同步压缩，大事件列表阻塞事件循环 | 200 条 × 大 DOM mutation 可导致 CPU 尖刺 |
| 4 | 录制数据无 TTL | ClickHouse/Prisma 无自动清理，磁盘无限增长 |
| 5 | `replayEnabled` 绕过 Redis 缓存直接查数据库 | 高频录制请求下增加数据库压力（代码注释：`Query directly to avoid stale Redis cache`） |

---

## 5. Kafka 多 Topic 事件投递关系

### 5.1 Topic 全景

当前项目有 **4 个 Kafka topic**，分别对应 4 种数据类型：

| Topic | 写入方 | ClickHouse 目标表 | Prisma 目标表 |
|-------|--------|-------------------|---------------|
| `event` | [saveEvent.ts#L260](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEvent.ts#L260) | `website_event` | `websiteEvent` |
| `event_data` | [saveEventData.ts#L75](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEventData.ts#L75) | `event_data` | `eventData` |
| `session_data` | [saveSessionData.ts#L100](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/sessions/saveSessionData.ts#L100) | `session_data` | `sessionData` |
| `session_replay` | [saveRecording.ts#L79](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/replays/saveRecording.ts#L79) | `session_replay` | `sessionReplay` |

### 5.2 写入链路图

```
/api/send (type=event|identify|performance)
    │
    ├─ saveEvent()
    │   ├─ ClickHouse: sendMessage('event', message)      ← Topic 1
    │   └─ Prisma: websiteEvent.create()
    │
    └─ saveEventData()  [当 eventData 非空时]
        ├─ ClickHouse: sendMessage('event_data', messages) ← Topic 2
        └─ Prisma: eventData.createMany()
              │
              └─ saveRevenue()  [当 revenue > 0 且 currency 非空时]
                  ├─ ClickHouse: sendMessage('event_data', ...) 或 insert
                  └─ Prisma: 直接写入（无 Kafka 路径）

/api/record (type=record)
    │
    └─ saveRecording()
        ├─ ClickHouse: sendMessage('session_replay', message) ← Topic 4
        └─ Prisma: sessionReplay.create() (gzip 压缩)

/api/send (type=identify)
    │
    └─ saveSessionData()
        ├─ ClickHouse: sendMessage('session_data', messages)  ← Topic 3
        └─ Prisma: sessionData.updateMany + create (upsert 模式)
```

### 5.3 Topic 间的依赖关系

| 依赖关系 | 说明 | 风险 |
|----------|------|------|
| `event` → `event_data` | 事件和事件数据**串行写入** | `event` 成功但 `event_data` 失败 → 有点击记录无自定义属性 |
| `event` → `event_data` → `revenue` | 三层串行 | 前两层成功但 `revenue` 失败 → 收入数据丢失 |
| `session_data` 的 upsert | Prisma 路径用 `updateMany` + `create`，ClickHouse 路径直接 insert | ClickHouse 中同一 session_data 可能重复写入（无去重） |
| `session_replay` 与 `event` | 同一 sessionId/visitId，但不同 API 入口 | replay 写入成功但对应 event 写入失败 → 孤立 replay |

### 5.4 Topic 与告警/Webhook 的缺失连接

如果后续实现告警引擎，需要**消费上述 topic 的数据**或**直接查询 ClickHouse/Prisma**来评估指标。当前缺少的关键环节：

| 缺失环节 | 说明 |
|----------|------|
| **指标计算层** | 无定时聚合（如每 5 分钟计算 p95 LCP、页面浏览量、错误率） |
| **指标 → 告警规则路由** | 无从指标到规则的匹配管道 |
| **告警事件 topic** | 如果走 Kafka，需新增 `alert_event` topic |
| **Webhook 投递 topic** | 如果解耦投递，需新增 `webhook_delivery` topic |

---

## 6. 失败重试：KafkaJS 传输重试 vs 应用层无死信/回退

### 6.1 KafkaJS 默认传输重试（存在但未充分利用）

KafkaJS 的 `producer.send()` 自带**传输层重试**机制，但本项目的配置**显式禁用了大部分重试**：

**代码位置**：[kafka.ts#L1-L10](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/kafka.ts#L1-L10)

```typescript
const CONNECT_TIMEOUT = 5000;   // 连接超时 5 秒
const SEND_TIMEOUT = 3000;      // 发送超时 3 秒
const ACKS = 1;                  // 只等 leader 确认
```

**KafkaJS 默认行为 vs 项目实际行为**：

| KafkaJS 配置项 | 默认值 | 项目是否设置 | 实际效果 |
|----------------|--------|-------------|----------|
| `retries` | 5 | ❌ 未设置 | 使用默认 5 次重试 |
| `retry.initialRetryTime` | 300ms | ❌ 未设置 | 默认 300ms 指数退避 |
| `retry.multiplier` | 0.2 | ❌ 未设置 | 默认乘数 |
| `retry.maxRetryTime` | 30000ms | ❌ 未设置 | 默认最大退避 30 秒 |
| `acks` | -1 (all ISR) | ✅ 设为 `1` | **降级为只等 leader，牺牲持久性换速度** |
| `idempotent` | false | ❌ 未设置 | 不保证精确一次语义 |

**关键发现**：KafkaJS 传输层确实有默认 5 次重试，**但这只覆盖瞬时网络故障和 leader 切换**。不覆盖以下场景：

| 场景 | KafkaJS 传输重试是否覆盖 | 应用层是否有处理 |
|------|--------------------------|-----------------|
| Broker 短暂不可达（网络抖动） | ✅ 覆盖（默认 5 次） | ❌ 超过 5 次后静默丢消息 |
| Leader 切换 | ✅ 覆盖 | ❌ |
| Topic 不存在（auto.create=false） | ❌ 不覆盖 | ❌ sendMessage catch 吞错 |
| 消息体过大 | ❌ 不覆盖 | ❌ sendMessage catch 吞错 |
| Producer 连接失败 | ❌ 不覆盖 | ❌ sendMessage catch 吞错 |
| Kafka 全集群不可用 | ❌ 不覆盖 | ❌ 无回退到 ClickHouse 直写 |
| 消息格式错误 | ❌ 不覆盖 | ❌ 无校验 |

### 6.2 应用层无死信队列

**代码位置**：[kafka.ts#L66-L91](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/kafka.ts#L66-L91)

```typescript
async function sendMessage(topic, message) {
  try {
    await connect();
    return producer.send({ topic, messages: [...], timeout: SEND_TIMEOUT, acks: ACKS });
  } catch (e) {
    console.log('KAFKA ERROR:', serializeError(e));
    // ← 关键问题：只打日志！
    // 1. 不抛出异常 → 调用方 await sendMessage(...) 收到 undefined 以为成功
    // 2. 不写入死信队列 → 消息永久丢失
    // 3. 不回退到 ClickHouse 直写 → 数据断层
    // 4. 不记录到数据库 → 事后无法追回
  }
}
```

### 6.3 应用层无降级回退

**代码位置**：[saveEvent.ts#L259-L263](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEvent.ts#L259-L263)

```typescript
if (kafka.enabled) {
  await sendMessage('event', message);        // ← 失败了？返回 undefined
} else {
  await insert('website_event', [message]);   // ← 只有 kafka.enabled=false 才走这里
}
```

当 `kafka.enabled = true` 时，**即使 `sendMessage` 完全失败（返回 `undefined`），代码也不会回退到 `insert` 直写**。

### 6.4 传输重试 vs 应用层缺失对比图

```
KafkaJS 传输层（存在）                   应用层（缺失）
────────────────────                    ────────────
retries: 5 (默认)                       ❌ 无应用层重试
retry.backoff: 指数退避 (默认)           ❌ 无自定义退避策略
覆盖: 网络抖动 / Leader 切换             ❌ 不覆盖: Topic 不存在 / 格式错 / 集群不可用
producer.send() 失败会抛异常              ❌ catch 吞错，不向上传播
                                        ❌ 无死信队列（DLQ）
                                        ❌ 无降级回退到 ClickHouse 直写
                                        ❌ 无投递日志持久化
                                        ❌ 无人工重放机制
```

### 6.5 影响范围

由于 `sendMessage` 被 4 个 topic 共用，上述缺陷**影响所有数据类型**：

| 调用方 | Topic | 失败后果 |
|--------|-------|----------|
| [saveEvent.clickhouseQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEvent.ts#L260) | `event` | 页面浏览/自定义事件静默丢失 |
| [saveEventData.clickhouseQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/events/saveEventData.ts#L75) | `event_data` | 事件自定义属性静默丢失 |
| [saveSessionData.clickhouseQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/sessions/saveSessionData.ts#L100) | `session_data` | 用户属性（identify）静默丢失 |
| [saveRecording.clickhouseQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/replays/saveRecording.ts#L79) | `session_replay` | 录制数据静默丢失 |

---

## 7. 签名验证：Share Token 后续权限匹配

### 7.1 Share Token 完整链路

```
1. 用户访问 /share/:slug
2. 前端通过 useShareTokenQuery(slug) 获取 share token
3. JWT payload 包含：{ websiteId, linkId, pixelId, boardId, websiteIds[], linkIds[], pixelIds[] }
4. 前端将 token 存入 Zustand store (shareToken)
5. 每次请求通过 x-umami-share-token + x-umami-share-context 发送
6. 服务端 checkAuth() 解析 token
7. 各权限函数根据 token payload 匹配资源 ID
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
      return null;     // ← share token 没有在 share 页面中使用
    }
  }

  return { token, authKey, shareToken, user };
}
```

### 7.3 Share Token 在权限函数中的匹配

**4 个权限文件**都实现了相同的模式：

| 权限文件 | 函数 | Share Token 匹配字段 |
|----------|------|---------------------|
| [website.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/website.ts#L7-L21) | `canViewWebsite` | `websiteId` / `pixelId` / `linkId` / `websiteIds[]` / `pixelIds[]` / `linkIds[]` |
| [board.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/board.ts#L6-L13) | `canViewBoard` | `boardId` |
| [link.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/link.ts#L6-L13) | `canViewLink` | `linkId` / `websiteId` / `linkIds[]` |
| [pixel.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/pixel.ts#L6-L13) | `canViewPixel` | `pixelId` / `websiteId` / `pixelIds[]` |

**以 `canViewWebsite` 为例**：[website.ts#L7-L21](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/permissions/website.ts#L7-L21)

```typescript
export async function canViewWebsite({ user, shareToken }: Auth, websiteId: string) {
  if (user?.isAdmin) return true;

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

  // 后续查数据库匹配 userId / teamId ...
}
```

### 7.4 权限匹配的异常分支风险

| # | 问题 | 具体代码 | 影响 |
|---|------|----------|------|
| 1 | **Share Token 仅授予 View 权限**，但无字段区分读/写 | `shareToken` payload 无 `permission` 字段 | 拿到 share token 的人理论上只能查看，但代码层面没有显式限制写操作 |
| 2 | **`SHARE_CONTEXT_HEADER` 只检查非空** | [auth.ts#L43](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/auth.ts#L43) `if (!shareContext)` | 攻击者设 `x-umami-share-context: anything` 即可通过检查，无需真正在 share 页面 |
| 3 | **Share Token 无过期时间** | `parseShareToken` 调用 `parseToken`，无 `exp` 校验 | Share 链接对应的 JWT 永不过期 |
| 4 | **单一字段匹配多个实体类型** | `shareToken?.pixelId === websiteId` | 如果 pixelId 和 websiteId 碰撞（都是 UUID），可能产生越权 |
| 5 | **数组字段无长度限制** | `websiteIds[]` / `pixelIds[]` / `linkIds[]` | 恶意构造超大数组的 JWT 可能导致 `includes()` 性能问题 |
| 6 | **写操作不检查 shareToken** | `canUpdateWebsite` / `canDeleteWebsite` 只检查 `user` | 这是正确的安全设计，但缺少显式拒绝 share token 写操作的逻辑 |
| 7 | **Share Token payload 未签名完整性校验** | JWT 签名保证完整性，但 payload 中 `websiteId` 等字段未与 `slug` 关联 | 理论上可以构造合法签名的 JWT，将 `websiteId` 替换为另一个网站的 ID（需要 secret） |

### 7.5 对未来 Webhook 的启示

| 当前 Share Token 模式 | Webhook 签名应采用的改进 |
|----------------------|------------------------|
| JWT 签名（对称密钥） | HMAC-SHA256 签名（每个 Webhook 独立密钥） |
| 无过期 | 必须包含 `timestamp`，接收方校验 5 分钟窗口 |
| `SHARE_CONTEXT_HEADER` 形式检查 | Webhook 不需要 context header，但需要 `nonce` 防重放 |
| 静默吞异常 | 必须区分「签名不匹配」vs「时间戳过期」vs「nonce 重复」 |
| 单一 `secret()` 全局密钥 | 每个 Webhook 端点独立密钥，支持密钥轮换 |

---

## 8. 现有代码分析：阈值匹配（非告警场景）

### 8.1 Web Vitals 性能评级阈值

**定位**：前端 UI 展示用评级常量，**不是告警触发阈值**

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

**异常分支风险**：

| # | 问题 | 影响 |
|---|------|------|
| 1 | `value <= 0` 直接返回 `'good'` | 数据采集异常（如负值）被掩盖 |
| 2 | 边界值均用 `<=`，`threshold.poor` 边界归属 `needs-improvement` | 达到 poor 阈值却仍显示"需改进" |
| 3 | 硬编码常量，不支持按网站自定义 | 多租户无法差异化 |

### 8.2 Goal 转化目标阈值（无代码实现）

[getGoal.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/queries/sql/reports/getGoal.ts) 返回 `{ num, total }`，前端画进度条，但**没有"转化率低于 X% 即告警"的配置入口**。

### 8.3 过滤器比较操作符

[constants.ts#L137-L154](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/constants.ts#L137-L154) 中的 `OPERATORS`（gt/lt/gte/lte/eq/neq）用于数据查询过滤，不是告警规则比较器，但未来告警引擎可复用其语义。

---

## 9. 现有代码分析：防重放（数据采集场景）

### 9.1 SessionId / VisitId 机制

**代码位置**：[send/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L143-L175)

- SessionId：基于 `sourceId + IP + UA + 盐值` 确定性生成
- VisitId：30 分钟过期，过期后重新生成
- 盐值轮换：[crypto.ts#L72-L78](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/crypto.ts#L72-L78) 按月/周/日轮换
- Cache Token 回传：[send/route.ts#L311-L313](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/app/api/send/route.ts#L311-L313)

**异常分支风险**：

| # | 问题 | 影响 |
|---|------|------|
| 1 | 带 `timestamp` 参数绕过 30 分钟过期检查 | 攻击者可构造旧 timestamp 复用过期 visitId |
| 2 | Cache Token 用裸 JWT 无过期 | 抓包后可永久复用 |
| 3 | 盐值轮换边界数据断裂 | 跨轮换点的用户被统计为两个新用户 |
| 4 | 无请求级 nonce | 完全相同的 HTTP 请求可被重复入库 |

---

## 10. 现有代码分析：节流（Redis RateLimit 未实际应用）

**代码位置**：[redis.ts#L72-L82](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/redis.ts#L72-L82)

`rateLimit` 函数存在但**从未被任何 API 调用**。INCR + EXPIRE 非原子，进程崩溃会导致永久限流。IP 黑名单 ([detect.ts#L140-L169](file:///d:/fz/0601-2/solo-dogfeeding/code/54-umami/src/lib/detect.ts#L140-L169)) 和机器人检测（`isbot`）是实际生效的防线。

---

## 11. 缺失部分清单

### 11.1 告警规则模块（缺失）

| 缺失项 | 说明 | 建议实现位置 |
|--------|------|-------------|
| **告警规则数据模型** | 存储规则定义（名称、指标、阈值、比较符、评估周期、通知渠道） | `prisma/schema.prisma` 新增 `AlertRule` 表 |
| **告警实例数据模型** | 存储告警触发记录（规则ID、触发时间、当前值、阈值、状态） | `prisma/schema.prisma` 新增 `AlertIncident` 表 |
| **规则评估引擎** | 定时拉取指标数据，匹配阈值，判断触发/恢复 | 新增 `src/lib/alert/engine.ts` |
| **规则 CRUD API** | 创建/编辑/删除/启停告警规则 | 新增 `src/app/api/alerts/route.ts` |
| **告警抑制/静默** | 防止告警风暴 | `AlertIncident` 状态机 + 静默窗口 |

### 11.2 Webhook 投递模块（缺失）

| 缺失项 | 说明 | 建议实现位置 |
|--------|------|-------------|
| **Webhook 配置数据模型** | 端点 URL、签名密钥、自定义请求头、订阅类型 | `prisma/schema.prisma` 新增 `WebhookEndpoint` 表 |
| **投递日志数据模型** | 每次投递的请求体、HTTP 状态、耗时、重试次数 | `prisma/schema.prisma` 新增 `WebhookDeliveryLog` 表 |
| **Webhook 投递器** | 构造请求体、HMAC 签名、HTTP POST、超时处理 | 新增 `src/lib/webhook/dispatcher.ts` |
| **投递重试队列** | 指数退避重试（max 5 次），死信队列 | Redis List 或数据库轮询 |
| **Webhook 管理 API** | 端点 CRUD + 测试 | 新增 `src/app/api/webhooks/route.ts` |

### 11.3 两者协作桥接（缺失）

| 缺失项 | 说明 |
|--------|------|
| **告警 → 通知路由** | 告警引擎触发后，根据规则的通知渠道分发 |
| **通知节流** | 同一告警短时间重复触发时合并/静默 |
| **Kafka 告警 topic** | 如果走 Kafka，需新增 `alert_event` + `webhook_delivery` topic |

---

## 12. 总结

| 项目 | 当前状态 | 关键问题 |
|------|----------|---------|
| **告警规则** | ❌ 完全不存在 | 无数据表、无评估引擎、无触发 API |
| **Webhook 投递** | ❌ 完全不存在 | 无端点表、无投递器、无签名与重试 |
| **遥测上报** | ✅ 存在但与告警无关 | 向 `api.umami.is` 上报版本/平台，静默吞错 |
| **版本检查** | ✅ 存在但与告警无关 | 向 `api.umami.is/v1/updates` 检查更新，静默失败 |
| **Session Replay** | ✅ 存在但与告警无关 | 浏览器→服务端→存储，无行为异常检测 |
| **Kafka 多 Topic** | ⚠️ 4 个 topic 均为数据写入 | `event` / `event_data` / `session_data` / `session_replay`，无告警消费 |
| **KafkaJS 传输重试** | ✅ 默认 5 次指数退避 | 但应用层 catch 吞错，传输重试耗尽后消息丢失 |
| **应用层死信/回退** | ❌ 完全不存在 | 无 DLQ、无降级直写、无投递日志 |
| **阈值匹配** | ⚠️ 仅评级展示 | Web Vitals 有常量但无告警联动；Goal 无阈值配置 |
| **签名验证** | ✅ 认证场景完善 | 但 Share Token 无过期、context header 仅检查非空、静默吞异常 |
| **防重放** | ⚠️ 数据采集去重层面 | session/visit 机制，无请求级 nonce |
| **节流** | ⚠️ 函数存在但未调用 | rateLimit 没挂到任何 API |
| **外部通知格式** | ❌ 无标准 | 需全新设计，建议参考 CloudEvents 1.0 |
