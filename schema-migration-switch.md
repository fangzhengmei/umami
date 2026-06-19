# Umami 数据库迁移与多库切换代码链路分析

## 1. 整体架构概览

Umami 是一个多数据库支持的 Web 分析系统，核心采用 **双存储引擎 + 查询路由 + 可选缓冲** 架构：

- **Prisma + PostgreSQL**：默认主存储，负责全部元数据（用户/网站/团队/权限）+ 事件事实数据
- **ClickHouse**：列式分析存储，可选启用（设置 `CLICKHOUSE_URL` 即切换），负责高吞吐事件数据写入与聚合查询
- **Kafka**：仅作为 ClickHouse 写入侧的**可选缓冲层**（不是路由层的独立选项），在 ClickHouse 分支内由 `kafka.enabled` 判断是否走消息队列
- **查询路由层**：`runQuery()` 按环境变量在 PostgreSQL / ClickHouse 之间二选一

```
采集请求 → API 路由 → 查询函数 (saveEvent / getWebsiteStats / …)
                                 │
                                 ▼
                    ┌──────── runQuery() 路由 ────────┐
                    │                                   │
          无 CLICKHOUSE_URL                    有 CLICKHOUSE_URL
                    │                                   │
                    ▼                                   ▼
          PostgreSQL 写入/查询                 ClickHouse 写入/查询
                    │                               │
                    │                     ┌─────────┴─────────┐
                    │                     │                   │
                    │              KAFKA_URL &&          仅 CLICKHOUSE_URL
                    │              KAFKA_BROKER 配置完整
                    │                     │                   │
                    │                     ▼                   ▼
                    │              发送 Kafka topic     直写 ClickHouse 表
                    │          (event/event_data/       (INSERT … VALUES)
                    │           session_data/session_replay)
                    │
                    └─────► 元数据类操作始终走 Prisma ORM，
                            不经过 runQuery() 路由
```

---

## 2. Prisma / PostgreSQL 体系

### 2.1 Schema 定义

Schema 文件：[`prisma/schema.prisma`](prisma/schema.prisma)

关键配置：
- **Provider**: `postgresql`
- **Client 输出**: `../src/generated/prisma`
- **Relation Mode**: `prisma`（应用层维护外键关系，DB 层无真实外键约束）

核心数据模型（共 20 张表）：

| 模型 | 表名 | 用途 |
|------|------|------|
| `User` | `user` | 用户账号 |
| `Website` | `website` | 被监控网站 |
| `Team` / `TeamUser` | `team` / `team_user` | 团队与成员关系 |
| `Session` | `session` | 访问会话（浏览器、OS、设备、地域等维度） |
| `WebsiteEvent` | `website_event` | 核心事实表：页面浏览 + 自定义事件 |
| `EventData` | `event_data` | 自定义事件属性（EAV 展开模式） |
| `SessionData` | `session_data` | 会话级自定义属性（EAV 展开模式） |
| `Revenue` | `revenue` | 电商收入（由事件属性中 revenue + currency 显式写入） |
| `Report` / `Segment` / `Board` | `report` / `segment` / `board` | 报表定义、用户分群、看板 |
| `Link` / `Pixel` / `Share` | `link` / `pixel` / `share` | 短链追踪、追踪像素、资源分享 |
| `SessionReplay` / `SessionReplaySaved` | `session_replay` / `session_replay_saved` | 会话录像分块、已保存录像索引 |

### 2.2 Prisma Client 初始化

代码位置：[`src/lib/prisma.ts`](src/lib/prisma.ts)

```typescript
// 关键函数 getClient() [L370-L416]
function getClient() {
  const url = process.env.DATABASE_URL;
  const replicaUrl = process.env.DATABASE_REPLICA_URL;
  // 从 DATABASE_URL query string 解析 schema 参数
  const schema = getSchema();

  const baseAdapter = new PrismaPg({ connectionString: url }, { schema });
  const baseClient = new PrismaClient({ adapter: baseAdapter, errorFormat: 'pretty', … });

  if (!replicaUrl) {
    globalThis[PRISMA] ??= baseClient;
    return baseClient;
  }

  // 只读副本：@prisma/extension-read-replicas 扩展
  const replicaAdapter = new PrismaPg({ connectionString: replicaUrl }, { schema });
  const replicaClient = new PrismaClient({ adapter: replicaAdapter, … });
  const extended = baseClient.$extends(readReplicas({ replicas: [replicaClient] }));
  globalThis[PRISMA] ??= extended;
  return extended;
}
```

特性：
- 基于 `@prisma/adapter-pg`（自定义 pg adapter，非 Prisma 默认驱动）
- 支持 `DATABASE_URL?schema=xxx` 指定 schema
- 支持 `DATABASE_REPLICA_URL` 只读副本，查询自动走 `client.$replica()`
- 全局单例缓存到 `globalThis['prisma']`，HMR 热更新不复用连接

### 2.3 原始查询封装

`prisma.rawQuery()` [L255-L282] 实现了自定义模板语法 → PostgreSQL 参数占位符转换：

```typescript
// 模板: {{param::type}}  →  $N::type
await rawQuery(
  `select * from website_event where website_id = {{websiteId::uuid}} and created_at between {{startDate}} and {{endDate}}`,
  { websiteId, startDate, endDate },
  'queryName'
);
```

额外处理：若 DATABASE_URL 指定了 schema，执行前先 `SET search_path TO "xxx"`。

---

## 3. ClickHouse 体系

### 3.1 Schema 定义

Schema 文件：[`db/clickhouse/schema.sql`](db/clickhouse/schema.sql)

核心表结构（共 6 张表）：

| 表名 | Engine | 主键 / 排序键 | 说明 |
|------|--------|---------------|------|
| `website_event` | MergeTree | `(toStartOfHour(created_at), website_id, session_id, visit_id)` | **宽表**：冗余 session 全部维度，避免 JOIN |
| `event_data` | MergeTree | `(website_id, event_id, data_key, created_at)` | 自定义事件属性 |
| `session_data` | **ReplacingMergeTree** | `(website_id, session_id, data_key)` | 会话属性，同键覆盖，天然 upsert |
| `website_event_stats_hourly` | **AggregatingMergeTree** | `(website_id, event_type, toStartOfHour(created_at), cityHash64(visit_id), visit_id)` | 小时级预聚合（由 MV 自动生成） |
| `website_revenue` | MergeTree | `(website_id, session_id, created_at)` | 收入表（由 MV 从 event_data 自动 JOIN 生成） |
| `session_replay` | MergeTree | `(replay_id, website_id, session_id, visit_id, chunk_index)` | 录像分块 |

**PostgreSQL vs ClickHouse 建模关键差异**：
- PG 采用 **规范化建模**：Session、WebsiteEvent、EventData 三张独立表，靠外键 JOIN；CH 采用 **宽表反范式**：session 维度（browser、os、device、country 等）直接嵌入 `website_event` 宽表。
- PG 收入由应用显式写 `revenue` 表；CH 收入由**物化视图**自动从 event_data 两表 JOIN 生成。
- CH `session_data` 用 ReplacingMergeTree + 排序键做 upsert，语义同 PG 的 `updateMany + 条件 create`。

### 3.2 物化视图

```sql
-- MV1: website_event → website_event_stats_hourly（小时级聚合）
CREATE MATERIALIZED VIEW umami.website_event_stats_hourly_mv
TO umami.website_event_stats_hourly AS
SELECT
  website_id, session_id, visit_id, ...
  argMinState(url_path, created_at) entry_url,      -- entry/exit 聚合
  argMaxState(url_path, created_at) exit_url,
  sumIf(1, event_type NOT IN (2, 5)) views,         -- 过滤 event_type=2/5 的视图数
  ...
FROM umami.website_event
GROUP BY website_id, session_id, visit_id, ... , toStartOfHour(created_at);

-- MV2: event_data → website_revenue（自动解析 revenue/currency 属性）
CREATE MATERIALIZED VIEW umami.website_revenue_mv TO umami.website_revenue AS
SELECT DISTINCT
  ed.website_id, ed.session_id, ed.event_id, ed.event_name,
  c.currency,
  coalesce(toDecimal64(ed.number_value, 2), toDecimal64(ed.string_value, 2)) revenue,
  ed.created_at
FROM umami.event_data ed
JOIN (SELECT event_id, string_value as currency
      FROM umami.event_data WHERE positionCaseInsensitive(data_key, 'currency') > 0) c
  ON c.event_id = ed.event_id
WHERE positionCaseInsensitive(data_key, 'revenue') > 0;
```

### 3.3 Projections（投影索引）

ClickHouse 原生支持按列重排的"投影"，类似 PostgreSQL 的部分物化索引：

```sql
ALTER TABLE umami.website_event ADD PROJECTION website_event_url_path_projection (
  SELECT * ORDER BY toStartOfDay(created_at), website_id, url_path, created_at
);
ALTER TABLE umami.website_event MATERIALIZE PROJECTION website_event_url_path_projection;

ALTER TABLE umami.website_event ADD PROJECTION website_event_referrer_domain_projection (
  SELECT * ORDER BY toStartOfDay(created_at), website_id, referrer_domain, created_at
);
ALTER TABLE umami.website_event MATERIALIZE PROJECTION website_event_referrer_domain_projection;
```

查询优化器在命中 url_path / referrer_domain 过滤时会自动选用对应 projection。

### 3.4 ClickHouse Client 初始化

代码位置：[`src/lib/clickhouse.ts`](src/lib/clickhouse.ts)

```typescript
// 启用条件：仅看 CLICKHOUSE_URL 是否存在 [L22]
const enabled = Boolean(process.env.CLICKHOUSE_URL);

// 连接解析 [L24-L48]
function getClient() {
  const { hostname, port, pathname, protocol, username = 'default', password } =
    new URL(process.env.CLICKHOUSE_URL);
  return createClient({
    url: `${protocol}//${hostname}:${port}`,
    database: pathname.replace('/', ''),  // URL path 段 → database 名
    username, password,
  });
}
```

参数化查询使用 ClickHouse 原生语法 `{name:Type}`（例：`{websiteId:UUID}`），区别于 Prisma 的 `{{name::type}}`。

---

## 4. 数据库切换 / 查询路由机制

### 4.1 核心路由函数 runQuery()

代码位置：[`src/lib/db.ts`](src/lib/db.ts)

```typescript
export const PRISMA = 'prisma';
export const POSTGRESQL = 'postgresql';
export const CLICKHOUSE = 'clickhouse';
export const KAFKA = 'kafka';        // 仅定义，当前所有调用均未使用

export function getDatabaseType(url = process.env.DATABASE_URL) {
  const type = url?.split(':')[0];
  return type === 'postgres' ? POSTGRESQL : type;
}

export async function runQuery(queries: any) {
  // 分支 1: 存在 CLICKHOUSE_URL → 走 ClickHouse
  if (process.env.CLICKHOUSE_URL) {
    // 注意：queries[KAFKA] 在现有 56 个 runQuery() 调用中从未传入过
    // Kafka 不是路由层的切换选项，而是 ClickHouse 分支内部的缓冲开关
    if (queries[KAFKA]) return queries[KAFKA]();
    return queries[CLICKHOUSE]();
  }
  // 分支 2: 默认走 PostgreSQL / Prisma
  const db = getDatabaseType();
  if (db === POSTGRESQL) return queries[PRISMA]();
}
```

**路由规则简化理解**：

| 环境变量 | 路由目标 |
|----------|----------|
| 设置了 `CLICKHOUSE_URL` | ClickHouse 分支（写入侧内部再决定是否经 Kafka） |
| 未设置 `CLICKHOUSE_URL` | PostgreSQL / Prisma 分支 |

⚠️ **重要修正**：虽然 `runQuery` 代码里写了 `if (queries[KAFKA])` 的优先分支，但**全仓库 56 个调用 `runQuery` 的文件里没有任何一个传入 `[KAFKA]` 键**。Kafka 不是路由层的"第三选项"，而是 ClickHouse 写入函数内部的二级开关（详见第 6 章）。

### 4.2 查询函数的双实现模式

每个事件相关的查询函数都包含两套独立实现，由 `runQuery()` 选其一执行：

写入示例 [`src/queries/sql/events/saveEvent.ts`](src/queries/sql/events/saveEvent.ts)：
```typescript
export async function saveEvent(args: SaveEventArgs) {
  return runQuery({
    [PRISMA]: () => relationalQuery(args),      // PostgreSQL 代码路径
    [CLICKHOUSE]: () => clickhouseQuery(args),   // ClickHouse 代码路径
  });
}
```

读取示例 [`src/queries/sql/getWebsiteStats.ts`](src/queries/sql/getWebsiteStats.ts)：
```typescript
export async function getWebsiteStats(websiteId, filters) {
  return runQuery({
    [PRISMA]: () => relationalQuery(websiteId, filters),
    [CLICKHOUSE]: () => clickhouseQuery(websiteId, filters),
  });
}
```

### 4.3 ClickHouse 查询的二级路由（原始表 vs 预聚合表）

ClickHouse 分支内部还会根据 filter 类型自动选表，以 [`getWebsiteStats`](src/queries/sql/getWebsiteStats.ts) 的 clickhouseQuery 为例 [L86-L135]：

```typescript
if (EVENT_COLUMNS.some(item => Object.keys(filters).includes(item))) {
  // 过滤条件涉及 event_name / tag / url_path 等事件级细粒度字段
  // → 查原始 website_event 表（精度保证）
  sql = `select sum(t.c) as "pageviews", ... from website_event ...`;
} else {
  // 仅时间 + 普通 session 维度过滤
  // → 查 website_event_stats_hourly 预聚合表（性能提升 10~100x）
  sql = `select sum(t.c) as "pageviews", ... from website_event_stats_hourly "website_event" ...`;
}
```

### 4.4 绕过路由层的操作

以下操作**始终只走 PostgreSQL / Prisma**，不经过 `runQuery`：
- **用户 / 团队 / 权限**：`User`、`Team`、`TeamUser`、`Website`（成员判断等）
- **配置类对象**：`Report`、`Segment`、`Board`、`Link`、`Pixel`、`Share`、`SessionReplaySaved`
- **读取侧 ORM 操作**：所有 [`src/queries/prisma/`](src/queries/prisma/) 目录下的查询

代码结构上，事件类查询在 [`src/queries/sql/`](src/queries/sql/)（含路由 + 双实现），元数据类查询在 [`src/queries/prisma/`](src/queries/prisma/)（直连 Prisma Client）。

---

## 5. 迁移顺序与回滚约束

### 5.1 Prisma 迁移列表

迁移目录：[`prisma/migrations/`](prisma/migrations/)

共 19 个版本迁移，按数字前缀顺序执行：

| 序号 | 迁移名 | 主要变更 |
|------|--------|----------|
| 01 | `init` | 基础表：user, session, website, website_event, event_data, team, team_user, team_website，附带默认 admin 账号 |
| 02 | `report_schema_session_data` | 新增 report、session_data 表 |
| 03 | `metric_performance_index` | 大量组合索引优化（website_id+created_at+各维度） |
| 04 | `team_redesign` | 团队模型重构，website 增加 team_id 字段（去除 team_website 中间表） |
| 05 | `add_visit_id` | website_event 增加 visit_id 字段，新增 visit_id 相关索引 |
| 06 | `session_data` | session 表字段扩展：hostname、distinct_id |
| 07 | `add_tag` | website_event 增加 tag 字段 |
| 08 | `add_utm_clid` | 增加 UTM 参数（utm_source/medium/campaign/content/term），点击 ID（gclid/fbclid/msclkid/ttclid/li_fat_id/twclid） |
| 09 | `update_hostname_region` | hostname 移至 website_event 表，region / subdivision 字段重构 |
| 10 | `add_distinct_id` | event / session 维度增加 distinct_id 匿名访客标识 |
| 11 | `add_segment` | 新增 segment 分群表 |
| 12 | `update_report_parameter` | report.parameters 类型由字符串改为 Json |
| 13 | `add_revenue` | 新增 revenue 收入表 |
| 14 | `add_link_and_pixel` | 新增 link（短链）、pixel（追踪像素）表 |
| 15 | `add_share` | 新增 share 分享资源表 |
| 16 | `boards` | 新增 board 看板表 |
| 17 | `remove_duplicate_key` | 清理冗余唯一约束与重复索引 |
| 18 | `add_performance` | website_event 增加 Web Vitals（lcp / inp / cls / fcp / ttfb） |
| 19 | `add_session_replay` | 新增 session_replay、session_replay_saved 录像表 |

### 5.2 迁移执行入口

**构建流程**（`npm run build`）：
```
check-env → build-db → check-db → build-tracker → build-recorder → build-geo → build-app
   ↓           ↓           ↓
 [脚本]        │       [check-db.js]
               │         ① checkEnv()       → DATABASE_URL 必选
               │         ② checkConnection()  → prisma.$connect 连通性
               │         ③ checkDatabaseVersion() → PostgreSQL ≥ 9.4.0
               │         ④ applyMigration()    → 执行 prisma migrate deploy
               │
               └── build-db-client (prisma generate)
                   build-prisma-client (esbuild 打包成 ESM client.js)
```

**Docker 启动流程**（`pnpm start-docker` / CMD）：
```
check-db → update-tracker → start-server
   ↓
 [check-db.js] 执行同上述 ①~④，启动时自动升级数据库
```

关键脚本：
- [`scripts/check-db.js`](scripts/check-db.js)：启动/构建前自动迁移执行器
- [`scripts/check-env.js`](scripts/check-env.js)：环境变量必选校验
- [`scripts/build-prisma-client.js`](scripts/build-prisma-client.js)：esbuild 二次打包 Prisma Client 为 ESM

**迁移相关 npm scripts**（见 [`package.json`](package.json)）：
```bash
npm run update-db          # prisma migrate deploy（生产环境手动执行）
npm run build-db-schema    # prisma db pull（从现有数据库反向生成 schema）
npm run build-db-client    # prisma generate
npm run build-db           # build-db-client + build-prisma-client
```

### 5.3 回滚约束

**Prisma Migrate 不支持自动回滚**，必须手工处理，约束如下：

1. **不可逆 DDL 普遍存在**：
   - `DROP TABLE` / `DROP COLUMN`：数据永久丢失（本项目历史迁移未出现 DROP，但这是通用约束）
   - `ALTER COLUMN TYPE` 收缩（如 `VARCHAR(500)`→`VARCHAR(100)`）可能截断数据
   - 结构性迁移（04 team_redesign、05 add_visit_id、09 update_hostname_region）涉及数据迁移，无法自动撤销

2. **`migration_lock.toml` 锁定 provider**：[`prisma/migrations/migration_lock.toml`](prisma/migrations/migration_lock.toml) 硬编码 `provider = "postgresql"`，不能改 provider。

3. **ClickHouse 无迁移框架**：[`db/clickhouse/schema.sql`](db/clickhouse/schema.sql) 是一次性手工 DDL 脚本，版本管理完全靠人。新增列 / 新增投影需要自己写 `ALTER TABLE` 并在部署流程中执行。

4. **跳过迁移的开关**：
   - `SKIP_DB_CHECK=1`：完全跳过 check-db.js（连连通性、版本校验都跳过）
   - `SKIP_DB_MIGRATION=1`：仅跳过 `prisma migrate deploy`，保留版本/连通性检查

---

## 6. 写入路径详解：三种模式的原子性 · 幂等 · 失败边界

⚠️ **本章是核心修正章节** —— Umami 实际存在**三种写入路径**，但并非路由层三分，而是：

```
                runQuery() 二选一
                      │
        ┌─────────────┴─────────────┐
        ▼                           ▼
  [路径 A]                    [路径 B/C: ClickHouse 内部分支]
 PostgreSQL 直写                  内部再判 kafka.enabled
                                     │
                            ┌────────┴────────┐
                            ▼                 ▼
                       [路径 B]          [路径 C]
                   ClickHouse 直写   发 Kafka（后续由消费者写 CH）
```

### 6.1 四类写入操作总览

四类写入在两种引擎下的实现差异：

| 写入函数 | PostgreSQL 实现 | ClickHouse 实现 | Kafka topic（仅路径 C） |
|----------|-----------------|-----------------|----------------------|
| `saveEvent` | `websiteEvent.create` | `insert('website_event')` | `event` |
| `saveEventData` | `eventData.createMany` | `insert('event_data')` | `event_data` |
| `saveSessionData` | `updateMany` + 条件 `create`（应用层 upsert） | `insert('session_data')` → **ReplacingMergeTree 自动去重** | `session_data` |
| `saveRecording` | `sessionReplay.create`（gzip → `Bytes`） | `insert('session_replay')`（JSON 字符串，表级 ZSTD(3)） | `session_replay` |
| `saveRevenue` | `revenue.create`（**只有 PG 分支**） | 无实现，由 `website_revenue_mv` 物化视图从 event_data 自动生成 | — |
| `createSession` | `INSERT … ON CONFLICT DO NOTHING`（**只有 PG，直调 prisma**） | 无实现，session 维度直接冗余写入 `website_event` 宽表 | — |

### 6.2 路径 A：PostgreSQL 直写

**触发条件**：`CLICKHOUSE_URL` 未设置。

以 [`saveEvent`](src/queries/sql/events/saveEvent.ts#L72-L168) 的 `relationalQuery` 为例：

```typescript
async function relationalQuery(args) {
  const websiteEventId = uuid();

  // 步骤 1：写 website_event 主表
  await prisma.client.websiteEvent.create({ data: { id: websiteEventId, … } });

  // 步骤 2：写 event_data 属性（如有）
  if (eventData) {
    await saveEventData({ websiteId, sessionId, eventId: websiteEventId, eventData, … });

    // 步骤 3：写 revenue（如有 revenue>0 且带 currency）
    if (revenue > 0 && currency) {
      await saveRevenue({ websiteId, sessionId, eventId: websiteEventId, … });
    }
  }
}
```

#### 原子性

- **无显式事务**：步骤 1/2/3 是三次独立 `await`，不包裹在 `prisma.$transaction()` 中。
  - ✅ 如果步骤 1 失败 → 全部未写，一致。
  - ⚠️ 步骤 1 成功但步骤 2 失败 → 出现"有 event 无 event_data"的残缺记录。
  - ⚠️ 步骤 1/2 成功但步骤 3 失败 → 有事件/属性但缺 revenue 记录。

#### 幂等性

- `createSession`：`INSERT … ON CONFLICT (session_id) DO NOTHING`，重复调用安全，幂等。
- `saveSessionData`：先 `updateMany({sessionId,dataKey})`，影响 0 行时再 `create`，语义 upsert，幂等。
- `saveEvent` / `saveEventData` / `saveRevenue`：**不幂等**，每次调用都会新建 UUID 后 `INSERT`，重试会产生重复事件。

#### 失败边界

| 失败点 | 状态 | 恢复手段 |
|--------|------|----------|
| Prisma Client 未连上 | 全部抛错，0 写入 | 检查 `DATABASE_URL` / 网络 |
| `websiteEvent.create` 抛唯一约束 / 类型错误 | 0 写入 | 修正参数 |
| `websiteEvent.create` 成功，`saveEventData` 网络中断 | event 已落盘，event_data 丢失 | 需业务补偿：根据 event_id 重放属性写入 |
| `saveEventData` 成功，`saveRevenue` 失败 | event+event_data 存在，revenue 缺失 | 根据 event_id 重写 revenue |
| `saveSessionData.updateMany` count=0 与 `create` 之间并发写入 | 可能抛唯一约束（race window） | catch后重试或用 `ON CONFLICT DO UPDATE` 代替（当前实现未做） |

---

### 6.3 路径 B：ClickHouse 直写

**触发条件**：`CLICKHOUSE_URL` 已设置 且 未配置 `KAFKA_URL && KAFKA_BROKER`（即 `kafka.enabled === false`）。

以 [`saveEvent`](src/queries/sql/events/saveEvent.ts#L170-L276) 的 `clickhouseQuery` 为例：

```typescript
async function clickhouseQuery(args) {
  const eventId = uuid();
  const message = { website_id, session_id, visit_id, event_id,
                    country, region, city, url_path, … ,
                    event_type, event_name, tag, distinct_id,
                    created_at: getUTCString(createdAt),
                    browser, os, device, screen, language, hostname,  // ← 宽表冗余
                    lcp, inp, cls, fcp, ttfb };

  // 注意：这里根据 kafka.enabled 二选一，路由层不感知
  if (kafka.enabled) {
    await sendMessage('event', message);       // 路径 C（下节）
  } else {
    await insert('website_event', [message]);  // 路径 B：直写
  }

  if (eventData) {
    await saveEventData({ eventId, eventData, … });  // 同样 insert('event_data', …)
  }
  // revenue 无显式写入：由物化视图自动生成
}
```

#### 原子性

- **ClickHouse 不支持事务**（单表 batch insert 内原子，跨表无）。
- `saveEvent` 主表写入 + `saveEventData` 分表写入是两步独立调用：
  - ⚠️ 主表成功 + event_data 失败 → 出现"宽表存在但属性缺"的中间态（与 PG 一致）。
- `website_revenue` 由 MV 在后台异步生成，写入 `event_data` 后 MV 不保证立即可见（最终一致，通常毫秒级）。

#### 幂等性

- **普通 MergeTree 表（website_event / event_data / session_replay）**：不幂等。重试会产生重复行，但查询端通常用 `uniq(session_id)` / `countDistinct` 做近似去重，对计数结果影响有限。
- **ReplacingMergeTree 表（session_data）**：排序键 `(website_id, session_id, data_key)`，相同键在后台合并时只保留最新版本，实现天然 upsert。写入幂等。
- **AggregatingMergeTree 表（website_event_stats_hourly）**：由 MV 写入，源表去重与否不影响聚合结果（sum/uniq 可抵抗重复）。

#### 失败边界

| 失败点 | 状态 | 恢复手段 |
|--------|------|----------|
| ClickHouse 节点不可达 | `clickhouse.insert` 抛错，全部未写 | 配置重试 / 降级到 Kafka |
| `website_event` INSERT 成功，`event_data` INSERT 网络中断 | 宽表存在，event_data 缺 | 补偿：按 event_id 重插属性 |
| INSERT 成功但客户端因网络超时未收到 ACK | 可能已写，重试会重复 | MergeTree 允许重复，查询端用 uniq / countDistinct 抵消 |
| MV `website_revenue_mv` 写入失败 | revenue 表缺行，event_data 已成功 | 后续合并时 MV 自动重试？否，MV 失败会丢行，需手工重建 |

---

### 6.4 路径 C：ClickHouse + Kafka 缓冲

**触发条件**：`CLICKHOUSE_URL` 已设置 且 `KAFKA_URL` 和 `KAFKA_BROKER` 均配置（`kafka.enabled === true`）。

Kafka 客户端见 [`src/lib/kafka.ts`](src/lib/kafka.ts)：

```typescript
// 启用条件 [L14]：两个变量同时存在才启用
const enabled = Boolean(process.env.KAFKA_URL && process.env.KAFKA_BROKER);

// 写消息 [L66-L91]
async function sendMessage(topic, message) {
  try {
    await connect();
    return producer.send({
      topic,                               // 'event' | 'event_data' | 'session_data' | 'session_replay'
      messages: Array.isArray(message)
        ? message.map(a => ({ value: JSON.stringify(a) }))
        : [{ value: JSON.stringify(message) }],
      timeout: 3000,
      acks: 1,                            // ⚠️ 只等 leader 确认，不等 ISR
    });
  } catch (e) {
    console.log('KAFKA ERROR:', serializeError(e));  // ⚠️ catch 后仅打印，不重抛
  }
}
```

四个写入函数全部走同一模式：`kafka.enabled ? sendMessage(topic, msg) : insert(table, [msg])`，topic 与表名对应：

| 写入函数 | Topic name | 对应目标 CH 表 |
|----------|------------|----------------|
| `saveEvent` | `event` | `website_event` |
| `saveEventData` | `event_data` | `event_data` |
| `saveSessionData` | `session_data` | `session_data` |
| `saveRecording` | `session_replay` | `session_replay` |

#### 原子性

- 与路径 B 相同：`saveEvent` / `saveEventData` / `saveSessionData` 是多次独立 `sendMessage`，跨 topic 无事务语义。
- **新增风险**：`sendMessage` 内部 `try/catch` 后**仅 `console.log`，不向上抛异常**。调用方看到的是 `Promise.resolve(undefined)`，以为成功，实际可能丢消息。
- `acks=1`：只保证 leader 已接收副本已同步，leader 挂掉仍可能丢。

#### 幂等性

- **Kafka Producer 幂等**：未启用（`kafkajs` 默认不开启幂等生产者，需显式 `idempotent: true`）。同一条消息在网络超时重试时会被重复投递。
- **消费者幂等**：取决于 Umami 的 Kafka→CH 消费端（本仓库不含消费端代码，应由外部部署负责）。建议用 `(event_id, data_key)` 等作为去重键，配合 ClickHouse ReplacingMergeTree 去重。

#### 失败边界

| 失败点 | 客户端感知 | 真实状态 | 风险与恢复 |
|--------|-----------|----------|-----------|
| Kafka broker 全不可达 / 超时 | `sendMessage` catch 吞错 → 返回 undefined（✔️） | 消息未发 | ⚠️ **静默丢消息**。修复：移除 catch 或记录到死信队列 |
| leader 写入成功但响应前 broker 重启（`acks=1`） | 客户端认为失败 → 重试 | 实际可能已写 → 消息重复 | 查询端 uniq 近似去重；业务关键数据需消费端幂等 |
| event topic 发送成功，event_data topic 发送失败（被 catch） | 调用方以为全成功 | 只存 event，缺属性 | ⚠️ 静默不一致。需按 event_id 对照补偿 |
| Kafka→CH 消费端崩溃、消费进度落后 | 客户端无感知 | 事件已入 Kafka，但 CH 查询看不到 | 监控消费 lag，消费端重启后自动追平 |
| MV `website_revenue_mv` 在消费端批量 INSERT 后某行 JOIN 缺 currency | 消费端无感知 | 对应 revenue 行永久缺 | 需离线作业扫描 event_data 里 revenue/currency 不匹配补写 |

---

### 6.5 三种写入路径对比总结

| 维度 | 路径 A：PostgreSQL 直写 | 路径 B：ClickHouse 直写 | 路径 C：ClickHouse + Kafka 缓冲 |
|------|-------------------------|-------------------------|----------------------------------|
| **触发条件** | 无 `CLICKHOUSE_URL` | 有 `CLICKHOUSE_URL`，无 Kafka | 有 `CLICKHOUSE_URL` + `KAFKA_URL` + `KAFKA_BROKER` |
| **路由层级** | `runQuery` 路由决定 | `runQuery` + ClickHouse 分支内部判断 | `runQuery` + ClickHouse 分支内部判断 |
| **写入原子性** | 跨表无事务（saveEvent 三步独立 await） | 跨表无事务（ClickHouse 无事务） | 跨 topic 无事务，且 sendMessage catch 吞错会**静默失败** |
| **幂等性** | `createSession` / `saveSessionData` 幂等；`saveEvent` / `saveEventData` / `saveRevenue` 不幂等 | `session_data`（ReplacingMergeTree）幂等；其余 MergeTree 表不幂等 | Producer 未开幂等 → 重复投递；消费端需幂等消费 |
| **立即可读性** | 写入立即可读（已提交读） | 写入即查可读到（非 quorum 场景） | 取决于 Kafka→CH 消费端 lag（典型 100ms~10s） |
| **吞吐上限** | ~数千 TPS（单 PostgreSQL 实例，受 WAL + 索引约束） | ~数万 TPS（ClickHouse 擅长大批量 INSERT） | ~数十万 TPS（Kafka 分区水平扩展） |
| **数据丢失可能性** | 低（同步写 WAL，fsync） | 中（CH 默认异步 fsync；节点宕机可能丢最近几秒） | 低（Kafka 多副本持久化），但 `acks=1` 仍可能丢 |
| **典型失败场景** | event 成功 / event_data 失败 的中间态 | 同上 + MV 失败缺 revenue 行 | catch 静默吞错造成**完全感知不到的丢消息**；重复消息；跨 topic 部分成功 |
| **失败恢复** | 手工补偿：按 UUID 重放对应子步骤 | 同上 + 重建 MV / 重跑缺失分区 | 监控 lag + 消费端幂等；改 sendMessage catch 为抛错或死信 |

---

## 7. 启动配置流程

### 7.1 环境变量校验链

```
应用启动
  ├─ check-env.js [scripts/check-env.js#L21-L27]
  │   ├─ 未设 SKIP_DB_CHECK 且无 DATABASE_TYPE → 必须有 DATABASE_URL
  │   └─ 有 CLOUD_URL → 必须同时设置 CLOUD_URL, CLICKHOUSE_URL, REDIS_URL
  │
  └─ check-db.js [scripts/check-db.js#L33-L89]
      ├─ checkEnv()          → DATABASE_URL 必填（不校验 CLICKHOUSE_URL）
      ├─ checkConnection()   → Prisma $connect
      ├─ checkDatabaseVersion() → SELECT version()，要求 ≥ 9.4.0
      └─ applyMigration()    → 未设 SKIP_DB_MIGRATION 时执行 prisma migrate deploy
```

### 7.2 Docker 启动命令链

[`Dockerfile`](Dockerfile) 中 CMD：`pnpm start-docker`，对应 [`package.json`](package.json) 脚本：

```
npm-run-all check-db update-tracker start-server
     │           │             └─ node scripts/start-env.js（即 next start）
     │           └─ node scripts/update-tracker.js（同步 tracker.js 到 public）
     └─ node scripts/check-db.js（自动执行 prisma migrate deploy）
```

### 7.3 关键环境变量

| 变量 | 必需 | 说明 |
|------|------|------|
| `DATABASE_URL` | ✅ | PostgreSQL 连接串，支持 `?schema=xxx` 指定 search_path |
| `DATABASE_REPLICA_URL` | ❌ | 只读副本，启用后 SELECT 类查询自动路由 |
| `CLICKHOUSE_URL` | ❌ | ClickHouse 连接串（格式：`clickhouse://user:pass@host:8123/database`）。**一旦设置，读写路径切换到 CH** |
| `KAFKA_URL` + `KAFKA_BROKER` | ❌ | 同时配置后，ClickHouse 写入路径改走 Kafka 缓冲（4 个 topic：event/event_data/session_data/session_replay） |
| `KAFKA_SASL_MECHANISM` | ❌ | Kafka 认证：`plain`（默认）/ `scram-sha-256` / `scram-sha-512` |
| `APP_SECRET` | ✅ | JWT / 加密密钥 |
| `SKIP_DB_CHECK` | ❌ | 跳过全部 check-db（含连通性、版本、迁移） |
| `SKIP_DB_MIGRATION` | ❌ | 仅跳过 `prisma migrate deploy` |
| `LOG_QUERY` | ❌ | 输出全部 Prisma rawQuery 与 ClickHouse query 参数到 debug 日志（namespace: `umami:prisma` / `umami:clickhouse`） |
| `REDIS_URL` | ❌ | Redis 缓存（Cloud 模式强制） |
| `CLOUD_URL` | ❌ | Umami Cloud 模式开关，触发额外变量强制校验 |

---

## 8. 关键文件索引（仓库相对路径）

| 类别 | 文件 |
|------|------|
| Prisma Schema | [`prisma/schema.prisma`](prisma/schema.prisma) |
| Prisma 配置（读取 DATABASE_URL） | [`prisma.config.ts`](prisma.config.ts) |
| Prisma Client 封装 + 原始查询 | [`src/lib/prisma.ts`](src/lib/prisma.ts) |
| ClickHouse Schema DDL | [`db/clickhouse/schema.sql`](db/clickhouse/schema.sql) |
| ClickHouse Client 封装 | [`src/lib/clickhouse.ts`](src/lib/clickhouse.ts) |
| Kafka Producer（ClickHouse 缓冲） | [`src/lib/kafka.ts`](src/lib/kafka.ts) |
| 路由层 runQuery + 数据库类型判断 | [`src/lib/db.ts`](src/lib/db.ts) |
| Prisma 迁移目录（19 个版本） | [`prisma/migrations/`](prisma/migrations/) |
| Prisma 迁移锁（锁定 postgresql provider） | [`prisma/migrations/migration_lock.toml`](prisma/migrations/migration_lock.toml) |
| 启动前 DB 检查 + 自动迁移 | [`scripts/check-db.js`](scripts/check-db.js) |
| 环境变量必填校验 | [`scripts/check-env.js`](scripts/check-env.js) |
| Prisma Client ESM 二次打包 | [`scripts/build-prisma-client.js`](scripts/build-prisma-client.js) |
| 写入示例：saveEvent（含 PG/CH 双路径 + Kafka 缓冲内部分支） | [`src/queries/sql/events/saveEvent.ts`](src/queries/sql/events/saveEvent.ts) |
| 写入：saveEventData | [`src/queries/sql/events/saveEventData.ts`](src/queries/sql/events/saveEventData.ts) |
| 写入：saveSessionData | [`src/queries/sql/sessions/saveSessionData.ts`](src/queries/sql/sessions/saveSessionData.ts) |
| 写入：saveRevenue（仅 PG） | [`src/queries/sql/events/saveRevenue.ts`](src/queries/sql/events/saveRevenue.ts) |
| 写入：createSession（仅 PG） | [`src/queries/sql/sessions/createSession.ts`](src/queries/sql/sessions/createSession.ts) |
| 写入：saveRecording | [`src/queries/sql/replays/saveRecording.ts`](src/queries/sql/replays/saveRecording.ts) |
| 读取示例：getWebsiteStats（含 CH 预聚合/原始表二级路由） | [`src/queries/sql/getWebsiteStats.ts`](src/queries/sql/getWebsiteStats.ts) |
| 读取：getWebsiteEventStats | [`src/queries/sql/events/getWebsiteEventStats.ts`](src/queries/sql/events/getWebsiteEventStats.ts) |
| SQL 查询目录（全部走 runQuery，双引擎实现） | [`src/queries/sql/`](src/queries/sql/) |
| ORM 查询目录（直连 Prisma，只走 PG） | [`src/queries/prisma/`](src/queries/prisma/) |
| npm scripts / 迁移命令 | [`package.json`](package.json) |
| Docker 镜像构建（build-docker → start-docker） | [`Dockerfile`](Dockerfile) |
| Docker Compose（默认 PG 单机） | [`docker-compose.yml`](docker-compose.yml) |
