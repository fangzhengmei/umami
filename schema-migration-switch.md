# Umami 数据库迁移与多库切换代码链路分析

## 1. 整体架构概览

Umami 是一个多数据库支持的 Web 分析系统，核心采用 **双存储引擎 + 查询路由** 架构：

- **Prisma + PostgreSQL**：关系型数据存储（用户、网站、团队、权限等元数据 + 事件数据）
- **ClickHouse**：列式存储，专用于高吞吐事件数据分析（可选启用）
- **Kafka**：消息队列，作为 ClickHouse 写入的缓冲层（可选启用）
- **查询路由层**：`runQuery()` 根据环境变量自动选择执行引擎

数据流：

```
采集请求
   ↓
API 路由层
   ↓
查询函数 (saveEvent / getWebsiteStats / ...)
   ↓
runQuery() 路由 ──┬── CLICKHOUSE_URL 存在 ──→ ClickHouse (或经 Kafka)
                  └── 默认 ───────────────────→ Prisma / PostgreSQL
```

---

## 2. Prisma / PostgreSQL 体系

### 2.1 Schema 定义

Schema 文件：[prisma/schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/prisma/schema.prisma)

关键配置：
- **Provider**: `postgresql`
- **Client 输出**: `../src/generated/prisma`
- **Relation Mode**: `prisma`（应用层维护外键关系）

核心数据模型（20 张表）：

| 模型 | 表名 | 用途 |
|------|------|------|
| `User` | `user` | 用户账号 |
| `Website` | `website` | 被监控网站 |
| `Team` / `TeamUser` | `team` / `team_user` | 团队与成员 |
| `Session` | `session` | 访问会话（含浏览器、OS、设备、地域等） |
| `WebsiteEvent` | `website_event` | 页面浏览 / 事件（核心事实表） |
| `EventData` | `event_data` | 自定义事件属性（EAV 模式） |
| `SessionData` | `session_data` | 会话级自定义属性 |
| `Revenue` | `revenue` | 电商收入数据 |
| `Report` / `Segment` / `Board` | `report` / `segment` / `board` | 报表、分群、看板 |
| `Link` / `Pixel` / `Share` | `link` / `pixel` / `share` | 短链、追踪像素、分享 |
| `SessionReplay` / `SessionReplaySaved` | `session_replay` / `session_replay_saved` | 会话录像 |

### 2.2 Prisma Client 初始化

代码位置：[src/lib/prisma.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/lib/prisma.ts)

```typescript
// 关键函数 getClient() [L370-L416]
function getClient() {
  const url = process.env.DATABASE_URL;
  const replicaUrl = process.env.DATABASE_REPLICA_URL;
  const schema = getSchema(); // 从 DATABASE_URL 的 searchParams 解析 schema

  const baseAdapter = new PrismaPg({ connectionString: url }, { schema });
  const baseClient = new PrismaClient({ adapter: baseAdapter, ... });

  if (!replicaUrl) return baseClient;

  // 只读副本配置（使用 @prisma/extension-read-replicas）
  const replicaAdapter = new PrismaPg({ connectionString: replicaUrl }, { schema });
  const replicaClient = new PrismaClient({ adapter: replicaAdapter, ... });
  return baseClient.$extends(readReplicas({ replicas: [replicaClient] }));
}
```

特性：
- 基于 `@prisma/adapter-pg`（非默认驱动）
- 支持 `DATABASE_URL?schema=xxx` 指定 schema
- 支持 `DATABASE_REPLICA_URL` 只读副本（查询走 `client.$replica()`）
- 全局单例缓存到 `globalThis['prisma']`

### 2.3 原始查询封装

`prisma.rawQuery()` [L255-L282] 实现了模板语法 `{{param::type}}` → PostgreSQL `$N::type` 转换：

```typescript
// 示例 SQL
await rawQuery(
  `select * from website_event where website_id = {{websiteId::uuid}} and created_at between {{startDate}} and {{endDate}}`,
  { websiteId, startDate, endDate }
);
```

---

## 3. ClickHouse 体系

### 3.1 Schema 定义

Schema 文件：[db/clickhouse/schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/db/clickhouse/schema.sql)

核心表结构：

| 表名 | Engine | 说明 |
|------|--------|------|
| `website_event` | MergeTree | 原始事件（含 session、pageview、event 宽表，反范式设计） |
| `event_data` | MergeTree | 自定义事件属性 |
| `session_data` | **ReplacingMergeTree** | 会话属性（按 `(website_id, session_id, data_key)` 去重，天然 upsert） |
| `website_event_stats_hourly` | **AggregatingMergeTree** | 小时级预聚合 |
| `website_revenue` | MergeTree | 收入数据（由物化视图自动生成） |
| `session_replay` | MergeTree | 会话录像分块 |

**关键设计差异**：相比 PostgreSQL，ClickHouse 的 `website_event` 是一张**宽表**，将 session 维度（browser、os、device、country 等）直接冗余存储，避免 JOIN。

### 3.2 物化视图与预聚合

```sql
-- 小时级统计 MV: website_event → website_event_stats_hourly
CREATE MATERIALIZED VIEW umami.website_event_stats_hourly_mv
TO umami.website_event_stats_hourly AS
SELECT ... groupArray, argMinState, argMaxState, sumIf ...
FROM umami.website_event
GROUP BY website_id, session_id, visit_id, ..., toStartOfHour(created_at);

-- 收入 MV: event_data JOIN event_data(currency) → website_revenue
CREATE MATERIALIZED VIEW umami.website_revenue_mv TO umami.website_revenue AS
SELECT DISTINCT ed.website_id, ..., coalesce(toDecimal64(ed.number_value, 2), ...) revenue
FROM umami.event_data ed
JOIN (SELECT event_id, string_value as currency FROM umami.event_data WHERE ...) c
  ON c.event_id = ed.event_id
WHERE positionCaseInsensitive(data_key, 'revenue') > 0;
```

### 3.3 Projections（投影索引）

```sql
ALTER TABLE umami.website_event ADD PROJECTION website_event_url_path_projection (
  SELECT * ORDER BY toStartOfDay(created_at), website_id, url_path, created_at
);
```

ClickHouse 会自动在查询时选择匹配的 projection，类似 PostgreSQL 的部分索引/物化视图。

### 3.4 ClickHouse Client 初始化

代码位置：[src/lib/clickhouse.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/lib/clickhouse.ts)

```typescript
// 启用条件 [L22]
const enabled = Boolean(process.env.CLICKHOUSE_URL);

// 连接解析 [L24-L48]
function getClient() {
  const { hostname, port, pathname, protocol, username = 'default', password } =
    new URL(process.env.CLICKHOUSE_URL);
  return createClient({
    url: `${protocol}//${hostname}:${port}`,
    database: pathname.replace('/', ''),  // URL 路径部分作为 database
    username, password,
  });
}
```

参数化查询使用 ClickHouse 原生语法 `{name:Type}`，区别于 Prisma 的 `{{name::type}}`。

---

## 4. 数据库切换 / 查询路由机制

### 4.1 核心路由函数

代码位置：[src/lib/db.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/lib/db.ts)

```typescript
export const PRISMA = 'prisma';
export const POSTGRESQL = 'postgresql';
export const CLICKHOUSE = 'clickhouse';
export const KAFKA = 'kafka';

export function getDatabaseType(url = process.env.DATABASE_URL) {
  const type = url?.split(':')[0];
  return type === 'postgres' ? POSTGRESQL : type;
}

export async function runQuery(queries: any) {
  // 优先级 1: CLICKHOUSE_URL 存在
  if (process.env.CLICKHOUSE_URL) {
    // 优先级 2: 如果定义了 KAFKA 分支且 Kafka 配置完整
    if (queries[KAFKA]) return queries[KAFKA]();
    return queries[CLICKHOUSE]();
  }
  // 优先级 3: 回退到 Prisma / PostgreSQL
  const db = getDatabaseType();
  if (db === POSTGRESQL) return queries[PRISMA]();
}
```

**切换逻辑总结**：
1. 存在 `CLICKHOUSE_URL` → 走 ClickHouse 分支（可能经 Kafka）
2. 否则走 Prisma / PostgreSQL
3. `DATABASE_URL` 用于解析数据库类型（当前仅 `postgres`）

### 4.2 查询函数的双实现模式

所有同时支持两种数据库的查询函数都遵循同一模式，例如 [saveEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/queries/sql/events/saveEvent.ts#L65-L70)：

```typescript
export async function saveEvent(args: SaveEventArgs) {
  return runQuery({
    [PRISMA]: () => relationalQuery(args),      // PostgreSQL 实现
    [CLICKHOUSE]: () => clickhouseQuery(args),   // ClickHouse 实现
  });
}
```

读取查询也完全一致，例如 [getWebsiteStats](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/queries/sql/getWebsiteStats.ts#L17-L24)：

```typescript
export async function getWebsiteStats(...) {
  return runQuery({
    [PRISMA]: () => relationalQuery(...),
    [CLICKHOUSE]: () => clickhouseQuery(...),
  });
}
```

### 4.3 ClickHouse 查询的性能优化

ClickHouse 查询会根据是否涉及 event 级别的 filter 决定查原始表还是预聚合表，见 [getWebsiteStats](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/queries/sql/getWebsiteStats.ts#L86-L135) 的 clickhouseQuery：

```typescript
if (EVENT_COLUMNS.some(item => Object.keys(filters).includes(item))) {
  // 过滤条件涉及事件级字段 → 查原始 website_event 表
  sql = `select ... from website_event ...`;
} else {
  // 无事件级过滤 → 查 website_event_stats_hourly 预聚合表（快 10~100x）
  sql = `select ... from website_event_stats_hourly "website_event" ...`;
}
```

---

## 5. 迁移顺序与回滚约束

### 5.1 Prisma 迁移体系

迁移目录：[prisma/migrations/](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/prisma/migrations/)

迁移列表（按数字前缀顺序）：

| 序号 | 迁移名 | 主要变更 |
|------|--------|----------|
| 01 | `init` | 基础表：user, session, website, website_event, event_data, team, team_user, team_website |
| 02 | `report_schema_session_data` | 新增 report、session_data 表 |
| 03 | `metric_performance_index` | 大量索引优化 |
| 04 | `team_redesign` | 团队模型重构，website 增加 team_id |
| 05 | `add_visit_id` | website_event 增加 visit_id |
| 06 | `session_data` | session 表字段扩展（hostname、distinct_id 等） |
| 07 | `add_tag` | website_event 增加 tag |
| 08 | `add_utm_clid` | 增加 UTM 参数、点击 ID（gclid、fbclid 等） |
| 09 | `update_hostname_region` | hostname / region 字段调整 |
| 10 | `add_distinct_id` | 增加 distinct_id 字段 |
| 11 | `add_segment` | 新增 segment 表 |
| 12 | `update_report_parameter` | report.parameters 改为 Json 类型 |
| 13 | `add_revenue` | 新增 revenue 表 |
| 14 | `add_link_and_pixel` | 新增 link、pixel 表 |
| 15 | `add_share` | 新增 share 表 |
| 16 | `boards` | 新增 board 表 |
| 17 | `remove_duplicate_key` | 清理重复索引 / 键 |
| 18 | `add_performance` | 增加性能指标（lcp、inp、cls、fcp、ttfb） |
| 19 | `add_session_replay` | 新增 session_replay、session_replay_saved 表 |

### 5.2 迁移执行入口

**开发构建流程**（`npm run build`）：
```
check-env → build-db → check-db → build-tracker → ... → build-app
   ↓            ↓          ↓
 [check-env.js]  │    [check-db.js]
                │       checkEnv() → DATABASE_URL 校验
                │       checkConnection() → 连通性
                │       checkDatabaseVersion() → ≥ 9.4.0
                │       applyMigration() → prisma migrate deploy
                │
                └── build-db-client (prisma generate)
                    build-prisma-client (esbuild 打包 client)
```

**Docker 启动流程**（`pnpm start-docker`）：
```
check-db → update-tracker → start-server
   ↓
 [check-db.js] 同上，自动执行 prisma migrate deploy
```

关键脚本：
- [scripts/check-db.js](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/scripts/check-db.js) — 启动时自动迁移
- [scripts/check-env.js](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/scripts/check-env.js) — 环境变量必填校验

**迁移命令**（package.json）：
```bash
npm run update-db      # prisma migrate deploy（生产环境）
npm run build-db-schema  # prisma db pull（从已有库反向生成 schema）
npm run build-db-client  # prisma generate
```

### 5.3 回滚约束

**Prisma Migrate 不支持自动回滚**，必须手动处理：

1. **不可逆操作**：
   - `DROP TABLE` / `DROP COLUMN` — 数据永久丢失
   - `ALTER COLUMN TYPE` — 类型收缩（如 `VARCHAR(500)` → `VARCHAR(100)`）可能截断
   - 迁移 04 `team_redesign`、05 `add_visit_id` 等结构性变更均无法自动回滚

2. **migration_lock.toml** [prisma/migrations/migration_lock.toml](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/prisma/migrations/migration_lock.toml) 锁定 provider 为 `postgresql`，**不支持跨数据库迁移**。

3. **ClickHouse 无迁移框架**：`db/clickhouse/schema.sql` 是一次性 DDL 脚本，需要手动执行版本管理。

4. **SKIP_DB_MIGRATION 环境变量**：设为任意值可跳过启动时的自动迁移。

---

## 6. 双写一致性（写入路径分析）

### 6.1 写入路由的"单写"本质

注意：**Umami 并不是真正的"双写"**，而是通过 `runQuery()` 选择一个目标写入。同一时刻只会写入 PostgreSQL **或** ClickHouse（经 Kafka），不会同时写两个库。

```
saveEvent(args)
    ↓
runQuery({
  PRISMA: relationalQuery,      // 只在无 CLICKHOUSE_URL 时执行
  CLICKHOUSE: clickhouseQuery   // 有 CLICKHOUSE_URL 时执行
})
```

### 6.2 PostgreSQL 写入链路

以 [saveEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/queries/sql/events/saveEvent.ts#L72-L168) 为例：

```typescript
async function relationalQuery(args: SaveEventArgs) {
  const websiteEventId = uuid();

  // 1. 主表写入（prisma.client.websiteEvent.create）
  await prisma.client.websiteEvent.create({ data: { id: websiteEventId, ... } });

  // 2. 自定义事件属性（如有）
  if (eventData) {
    await saveEventData({ websiteId, sessionId, eventId: websiteEventId, eventData, ... });
    // 3. 收入数据（如有 revenue + currency）
    if (revenue > 0 && currency) {
      await saveRevenue({ ... });
    }
  }
}
```

**一致性机制**：
- `createSession` 使用 `INSERT ... ON CONFLICT DO NOTHING` 保证幂等
- `saveSessionData` 采用先 `updateMany` 再条件 `create` 的 upsert 模式，避免并发竞态

### 6.3 ClickHouse 写入链路（含 Kafka）

同函数的 ClickHouse 分支 [saveEvent.ts#L170-L276](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/queries/sql/events/saveEvent.ts#L170-L276)：

```typescript
async function clickhouseQuery(args: SaveEventArgs) {
  const eventId = uuid();
  const message = { website_id, session_id, visit_id, event_id, ... };

  // Kafka 优先，否则直写 ClickHouse
  if (kafka.enabled) {
    await sendMessage('event', message);           // topic: event
  } else {
    await insert('website_event', [message]);      // 直接 INSERT
  }

  // 事件属性走独立 topic / 表
  if (eventData) {
    await saveEventData({ eventId, eventData, ... });
  }
}
```

Kafka 配置 [src/lib/kafka.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/lib/kafka.ts)：
```typescript
const enabled = Boolean(process.env.KAFKA_URL && process.env.KAFKA_BROKER);

// Topics: 'event' | 'event_data' | 'session_data' | 'session_replay'
async function sendMessage(topic, message) {
  return producer.send({
    topic,
    messages: [{ value: JSON.stringify(message) }],
    acks: 1,      // 只等 leader 确认
    timeout: 3000,
  });
}
```

**Kafka 相关环境变量**：
- `KAFKA_URL`：含用户名密码的连接串（`kafka://user:pass@host`）
- `KAFKA_BROKER`：broker 列表，逗号分隔
- `KAFKA_SASL_MECHANISM`：`plain`（默认）| `scram-sha-256` | `scram-sha-512`

### 6.4 四类写入函数对比

| 函数 | Prisma 实现 | ClickHouse 实现 | Kafka Topic |
|------|-------------|-----------------|-------------|
| `saveEvent` | `websiteEvent.create` | `insert('website_event')` | `event` |
| `saveEventData` | `eventData.createMany` | `insert('event_data')` | `event_data` |
| `saveSessionData` | `updateMany` + 条件 `create` | `insert('session_data')`（ReplacingMergeTree 天然去重） | `session_data` |
| `saveRecording` | `sessionReplay.create`（gzip 压缩 Bytes） | `insert('session_replay')`（JSON 字符串 + ZSTD CODEC） | `session_replay` |

### 6.5 一致性权衡

| 维度 | PostgreSQL | ClickHouse 直写 | ClickHouse + Kafka |
|------|-----------|-----------------|--------------------|
| **实时性** | 立即可读 | 立即可读（最终一致，~ms）| 消费者消费延迟（~秒级） |
| **原子性** | 支持事务（但 saveEvent 系列并未用 `$transaction`） | 无事务，逐批写入 | Kafka 发送成功 ≠ CH 写入成功 |
| **幂等性** | `ON CONFLICT` / upsert | ReplacingMergeTree 按主键去重 | 依赖消费者幂等 |
| **数据丢失风险** | 低（同步写入） | 中（CH 节点异常时丢失） | 低（Kafka 持久化）但需消费者保障 |
| **吞吐** | 中（~千 TPS） | 高（~万 TPS） | 极高（随分区扩展） |

⚠️ **重要**：PostgreSQL 写入虽然有事务能力，但当前 saveEvent / saveEventData / saveRevenue 是**三次独立 await**，未包裹在 `prisma.transaction()` 中，存在部分写入成功的中间态。

---

## 7. 启动配置流程

### 7.1 环境变量校验链

```
应用启动
  ├─ check-env.js [scripts/check-env.js#L21-L27]
  │   ├─ 非 SKIP_DB_CHECK 且无 DATABASE_TYPE → 必须有 DATABASE_URL
  │   └─ 有 CLOUD_URL → 必须有 CLOUD_URL, CLICKHOUSE_URL, REDIS_URL
  │
  └─ check-db.js [scripts/check-db.js#L33-L89]
      ├─ checkEnv() → DATABASE_URL 必填
      ├─ checkConnection() → Prisma $connect
      ├─ checkDatabaseVersion() → PostgreSQL ≥ 9.4.0
      └─ applyMigration() → prisma migrate deploy（可被 SKIP_DB_MIGRATION 跳过）
```

### 7.2 Docker 启动命令链

Dockerfile CMD：`pnpm start-docker` → [package.json#L18](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/package.json#L18)

```
npm-run-all check-db update-tracker start-server
     │           │             └─ next start (scripts/start-env.js)
     │           └─ node scripts/update-tracker.js
     └─ node scripts/check-db.js（自动执行 prisma migrate deploy）
```

### 7.3 关键环境变量总览

| 变量 | 必需 | 说明 |
|------|------|------|
| `DATABASE_URL` | ✅ | PostgreSQL 连接串，支持 `?schema=xxx` 指定 schema |
| `DATABASE_REPLICA_URL` | ❌ | 只读副本，启用后查询自动路由 |
| `CLICKHOUSE_URL` | ❌ | ClickHouse 连接串，存在则切换到 CH 引擎 |
| `KAFKA_URL` + `KAFKA_BROKER` | ❌ | 启用后 ClickHouse 写入经 Kafka 缓冲 |
| `KAFKA_SASL_MECHANISM` | ❌ | Kafka 认证方式 |
| `APP_SECRET` | ✅ | JWT / 加密密钥 |
| `SKIP_DB_CHECK` | ❌ | 跳过所有数据库检查与迁移 |
| `SKIP_DB_MIGRATION` | ❌ | 仅跳过迁移，仍做连通性检查 |
| `LOG_QUERY` | ❌ | 输出所有 SQL / CH 查询日志 |
| `REDIS_URL` | ❌ | Redis 缓存（Cloud 模式必需） |
| `CLOUD_URL` | ❌ | Umami Cloud 模式入口 |

---

## 8. 关键文件索引

| 类别 | 文件 |
|------|------|
| Prisma Schema | [prisma/schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/prisma/schema.prisma) |
| Prisma 配置 | [prisma.config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/prisma.config.ts) |
| Prisma Client 封装 | [src/lib/prisma.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/lib/prisma.ts) |
| ClickHouse Schema | [db/clickhouse/schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/db/clickhouse/schema.sql) |
| ClickHouse Client 封装 | [src/lib/clickhouse.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/lib/clickhouse.ts) |
| Kafka Client | [src/lib/kafka.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/lib/kafka.ts) |
| 数据库路由 | [src/lib/db.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/lib/db.ts) |
| 迁移目录 | [prisma/migrations/](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/prisma/migrations/) |
| 启动检查 | [scripts/check-db.js](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/scripts/check-db.js) |
| 环境检查 | [scripts/check-env.js](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/scripts/check-env.js) |
| 写入示例 | [src/queries/sql/events/saveEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/queries/sql/events/saveEvent.ts) |
| 读取示例 | [src/queries/sql/getWebsiteStats.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/src/queries/sql/getWebsiteStats.ts) |
| 构建配置 | [package.json](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/package.json) |
| Docker 构建 | [Dockerfile](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/Dockerfile) |
| Docker Compose | [docker-compose.yml](file:///d:/fz/0601-2/solo-dogfeeding/code/49-umami/docker-compose.yml) |
