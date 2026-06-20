# Umami SSO 账号归属规则分析

## 概述

本文档基于 Umami v3.1.0 开源版本代码库，系统梳理所有 Token 签发入口、验证接口、以及 SSO 登录链路的账号归属规则。

> **重要说明**：开源版本仅提供 SSO 接入的应用层框架。完整的 OAuth/SAML 协议处理、JIT 用户自动创建、邮件域到 Team 自动映射、IdP 角色组映射等企业级 SSO 功能**未在开源版本中实现**，需依赖 Cloud 商业服务或自行构建外部适配层。

---

## 1. Token 体系总览

### 1.1 两种 Token 类型

Umami 有两套独立的 Token 体系：

| 类型 | 加密方式 | 签发函数 | 验证函数 | HTTP 头 | 用途 |
|------|---------|---------|---------|---------|------|
| **Secure Token** | AES-256-GCM 加密 + JWT 签名 | `createSecureToken()` | `parseSecureToken()` | `Authorization: Bearer` | 用户认证（登录态） |
| **Share Token** | 纯 JWT 签名（不加密） | `createToken()` | `parseToken()` | `x-umami-share-token` | 共享链接访问 |

两种 Token 使用同一个 `secret()` 密钥，但验证路径完全独立。SSO 只涉及 Secure Token 体系。

### 1.2 所有服务端 Token 签发入口（共 3 个）

| 接口 | 方法 | Token 类型 | 签发方式 | 是否产生新 Token |
|------|------|-----------|---------|-----------------|
| `/api/auth/login` | POST | Secure | `saveAuth()` 或 `createSecureToken()` | ✅ 是 |
| `/api/auth/sso` | POST | Secure | `saveAuth()` | ✅ 是 |
| `/api/share/[slug]` | GET | Share | `createToken()` | ✅ 是（独立体系） |

### 1.3 所有仅验证 Token 并返回用户态的接口（共 2 个）

| 接口 | 方法 | 返回内容 | 是否签发新 Token | 用途 |
|------|------|---------|-----------------|------|
| `/api/auth/verify` | POST | `{ user, teams, ... }` | ❌ 否 | 前端主入口验证登录态 |
| `/api/me` | GET | `{ token, authKey, shareToken, user }` | ❌ 否 | 获取原始 auth 对象 |

### 1.4 客户端 Token 写入点（共 2 个）

| 位置 | 触发场景 | Token 来源 |
|------|---------|-----------|
| `src/app/sso/SSOPage.tsx` | SSO 回调 | URL query 参数 `token` |
| `src/app/login/LoginForm.tsx` | 本地登录 | `/api/auth/login` 响应体 |

### 1.5 登出入口（共 2 个）

| 位置 | 操作 |
|------|------|
| `POST /api/auth/logout` | 删除 Redis 中的 authKey（如果启用 Redis） |
| `src/app/logout/LogoutPage.tsx` | 前端调用 `removeClientAuthToken()` 删除 localStorage |

---

## 2. 四个核心接口的边界对比

本章精确定义 `/api/auth/login`、`/api/auth/sso`、`/api/auth/verify`、SSO 回调页面四者的职责边界。

### 2.1 边界总表

| 维度 | 本地登录<br>`/api/auth/login` | SSO 会话转换<br>`/api/auth/sso` | 验证态<br>`/api/auth/verify` | SSO 回调页面<br>`/sso` |
|------|------------------------------|--------------------------------|-----------------------------|-----------------------|
| **类型** | 认证 + 签发 | 验证 + 重新签发 | 仅验证 | 客户端存储 |
| **输入** | `{ username, password }` | Authorization: Bearer \<已有 token> | Authorization: Bearer \<token> | URL query: `token`, `url` |
| **用户定位** | `getUserByUsername(username)` | 通过 token 的 userId/authKey | 通过 token 的 userId/authKey | 不涉及 |
| **验证方式** | 密码比对 `checkPassword` | `checkAuth` 验签解密 | `checkAuth` 验签解密 | 不验证 |
| **是否签发新 Token** | ✅ 是 | ✅ 是 | ❌ 否 | ❌ 否（原封不动存本地） |
| **Redis 依赖** | 可选（有则用 authKey，无则用 userId） | ✅ 必须启用 | ❌ 不依赖 | ❌ 不依赖 |
| **返回用户信息** | ✅ 是（含 teams） | ✅ 是 | ✅ 是（含 teams） | ❌ 否 |
| **前端是否调用** | ✅ 是（LoginForm） | ❌ 否（前端零调用） | ✅ 是（useLoginQuery） | ✅ 是（路由页面） |

### 2.2 本地登录 `/api/auth/login` — 认证 + 签发

**文件**: `src/app/api/auth/login/route.ts`

```typescript
export async function POST(request: Request) {
  const { username, password } = body;

  // 用 username 查库（唯一调用 getUserByUsername 的认证入口）
  const user = await getUserByUsername(username, { includePassword: true });

  if (!user || !checkPassword(password, user.password)) {
    return unauthorized();
  }

  // 签发新 Token
  let token: string;
  if (redis.enabled) {
    token = await saveAuth({ userId: id, role });  // Redis 模式 → authKey 令牌
  } else {
    token = createSecureToken({ userId, role }, secret());  // 无 Redis → userId 令牌
  }

  const teams = await getAllUserTeams(id);
  return json({ token, user: { id, username, role, createdAt, isAdmin, teams } });
}
```

**边界定义**：
- 是唯一同时做"身份验证"和"Token 签发"的接口
- 是唯一通过 `username` 定位用户的认证接口
- 签发的 Token 可以是 `{ userId }` 型或 `{ authKey }` 型，取决于 Redis 是否启用

### 2.3 SSO 会话转换 `/api/auth/sso` — 验证 + 重新签发

**文件**: `src/app/api/auth/sso/route.ts`

```typescript
export async function POST(request: Request) {
  const { auth, error } = await parseRequest(request);  // 要求请求本身已带有效 Token

  if (error) {
    return error();
  }

  if (!redis.enabled) {
    return serverError({ message: 'Redis is disabled' });  // 必须有 Redis
  }

  // 将已有 userId 写入 Redis，签发新的 authKey 令牌（TTL 86400s）
  const token = await saveAuth({ userId: auth.user.id }, 86400);

  return json({ user: auth.user, token });
}
```

**边界定义**：
- **前置条件**：请求必须已携带有效的 Bearer Token（即 `checkAuth` 必须通过）
- **不做用户名匹配**：用户身份由传入 Token 中的 `userId`/`authKey` 决定
- **必须有 Redis**：没有 Redis 直接返回 500
- **签发新 Token**：把传入的 Token（可能是 userId 型或 authKey 型）统一转换为带 TTL 的 authKey 型
- **前端零调用**：grep 确认 `src/` 下没有任何前端代码调用此接口

**典型使用场景**：
- 服务端到服务端调用（如 Cloud 服务代用户访问 Umami）
- 需要给外部传入的 Token 增加 TTL 控制
- 需要将外部适配层签发的 userId 型 Token 转换为 Redis 会话 Token

### 2.4 验证态 `/api/auth/verify` — 仅验证，不签发

**文件**: `src/app/api/auth/verify/route.ts`

```typescript
export async function POST(request: Request) {
  const { auth, error } = await parseRequest(request);  // checkAuth 验证

  if (error) {
    return error();  // Token 无效 → 401，前端跳登录页
  }

  const user = { ...auth.user };
  const teams = await getAllUserTeams(user.id);

  // Cloud 模式附加订阅信息...

  return json({ ...user, teams });  // 返回用户态，不返回新 Token
}
```

**边界定义**：
- **纯验证接口**：只验证 Token 有效性，返回用户信息和团队列表
- **不签发新 Token**：响应体中不含 `token` 字段
- **不依赖 Redis**：Token 本身是 userId 型就直接查库，是 authKey 型就走 Redis
- **前端主入口**：`useLoginQuery` 在应用加载时调用此接口确认登录态

### 2.5 SSO 回调页面 `/sso` — 客户端存储，不调 API

**文件**: `src/app/sso/SSOPage.tsx`

```typescript
export function SSOPage() {
  const router = useRouter();
  const search = useSearchParams();
  const url = search.get('url');     // 登录后跳转目标
  const token = search.get('token'); // 回调 Token

  useEffect(() => {
    if (url && token) {
      setClientAuthToken(token);  // 直接存入 localStorage，不调任何 API
      router.push(url);           // 前端路由跳转
    }
  }, [router, url, token]);

  return <Loading placement="absolute" />;
}
```

**边界定义**：
- **纯客户端操作**：不发起任何网络请求
- **Token 原封不动**：回调 Token 直接存入 localStorage，不做任何转换
- **不验证 Token**：有效性留待后续 API 请求时由服务端 `checkAuth` 验证
- **跳转后验证**：跳到目标页后，`useLoginQuery` 自动调 `/api/auth/verify` 验证

---

## 3. SSO 登录完整 Token 流向（按代码拆开）

### 3.1 前端默认路径（通道 A）：回调 Token 直用

这是 SSO 场景下前端实际走的路径，**不经过 `/api/auth/sso`**。

```
外部适配层                              前端浏览器                         应用层 API
    │                                    │                                   │
    │ 1. OAuth/SAML 认证完成              │                                   │
    │ 2. 用 username/email 查本地用户     │                                   │
    │ 3. （可选）JIT 创建/域映射/角色映射  │                                   │
    │ 4. 签发 Secure Token { userId }    │                                   │
    │                                    │                                   │
    │ 重定向 /sso?url=/&token=<jwt>      │                                   │
    └───────────────────────────────────>│                                   │
                                         │                                   │
                                         │ 5. SSOPage:                       │
                                         │    setClientAuthToken(token)      │
                                         │    router.push('/')               │
                                         │     (不调任何 API)                 │
                                         │                                   │
                                         │ 6. App 加载 → useLoginQuery      │
                                         │                                   │
                                         │ POST /api/auth/verify             │
                                         │ Authorization: Bearer <回调token> │
                                         │──────────────────────────────────>│
                                         │                                   │
                                         │              7. checkAuth:        │
                                         │                 parseSecureToken  │
                                         │                 → payload.userId  │
                                         │                 → getUser()       │
                                         │                                   │
                                         │     { user, teams }               │
                                         │<──────────────────────────────────│
                                         │                                   │
                                         │ 8. setUser(data) → 登录成功       │
                                         │                                   │
                                         │ ────────────────────────────────  │
                                         │ │ 所有后续请求自动带 Bearer 头 │  │
                                         │ ────────────────────────────────  │
```

### 3.2 可选的服务端转换（通道 C）：经过 `/api/auth/sso`

前端不走这条路径，但外部适配层或服务端场景可能使用：

```
外部适配层                        应用层 /api/auth/sso
    │                                    │
    │ 1. 签发 Secure Token { userId }    │
    │                                    │
    │ POST /api/auth/sso                 │
    │ Authorization: Bearer <token>      │
    │───────────────────────────────────>│
    │                                    │
    │                     2. checkAuth   │
    │                        → userId    │
    │                                    │
    │                     3. saveAuth:   │
    │                        Redis 写入   │
    │                        { userId }   │
    │                        TTL 86400s  │
    │                                    │
    │                     4. 签发新的     │
    │                        authKey 令牌 │
    │     { user, token }                │
    │<───────────────────────────────────┘
    │
    ▼
  新的 authKey 令牌可用于后续请求
```

### 3.3 两种 Token 类型的服务端兼容性

`checkAuth` 对所有来源的 Secure Token 一视同仁，支持两种 payload：

| Token 来源 | payload 结构 | checkAuth 还原路径 | Redis 依赖 |
|-----------|-------------|-------------------|-----------|
| 外部适配层直接签发 | `{ userId, role? }` | 路径 A：直接 `getUser(userId)` | 不依赖 |
| `/api/auth/sso` 返回 | `{ authKey }` | 路径 B：`Redis.get(authKey)` → `getUser(key.userId)` | 依赖 Redis |
| `/api/auth/login`（有 Redis） | `{ authKey }` | 路径 B | 依赖 Redis |
| `/api/auth/login`（无 Redis） | `{ userId, role }` | 路径 A | 不依赖 |

---

## 4. 两层边界：外部适配层 vs 应用层

### 4.1 边界划分

```
                         ← 职责边界：Token 签发 →
                         ───────────────────────

  外部适配层（开源未实现）           应用层（开源已实现）
  ─────────────────────           ─────────────────────
  输入：用户名 / 邮箱               输入：Secure Token（含 userId 或 authKey）
  操作：                           操作：
   1. 对接 OAuth/SAML IdP          1. AES-256-GCM 解密 + JWT 验签
   2. 用 username/email 查本地用户  2. 从 payload 取 userId 或 authKey
   3. 不存在则 JIT 创建用户          3. userId → getUser(id) 查库
   4. 邮件域 → Team 自动映射        4. authKey → Redis 取 { userId } 再查库
   5. IdP 组 → Umami 角色映射       5. 附加 isAdmin 标记，返回 auth 对象
   6. 签发 Secure Token（含 userId） 6. saveAuth 写 Redis，签发新 authKey 令牌
  输出：签发好的 Secure Token        输出：应用 Session Token + 用户信息
```

### 4.2 关键差异

| 维度 | 外部适配层 | 应用层 |
|------|-----------|--------|
| **用户定位方式** | `username` / `email`（可读标识） | `userId`（UUID）/ `authKey`（Redis 键） |
| **核心操作** | 把 IdP 身份映射为本地 userId | 用 userId 或 authKey 还原用户对象 |
| **涉及代码** | 不在开源仓库中 | `src/lib/auth.ts` / `src/app/api/auth/*` |
| **匹配函数** | `getUserByUsername(username)` | `getUser(userId)` — 只按 UUID 查 |
| **契约** | — | JWT payload 中的 `userId` 字段 |

### 4.3 用户名匹配只发生在外部适配层

开源代码中，`getUserByUsername` 在认证流程中的调用点**仅有一处**：

```typescript
// src/app/api/auth/login/route.ts
const user = await getUserByUsername(username, { includePassword: true });
```

这是本地登录的入口。SSO 相关路径（`/api/auth/sso`、`/api/auth/verify`、所有通过 Bearer 的请求）**都不调用** `getUserByUsername`。

换言之：
- **外部适配层**负责"用户名/邮箱 → userId"的解析
- **应用层**只负责"userId/authKey → 用户对象"的还原
- 本地登录接口本身就是一个最小化的内置适配层

---

## 5. 应用层认证机制

### 5.1 checkAuth：应用层的唯一认证入口

**文件**: `src/lib/auth.ts`

```typescript
export async function checkAuth(request: Request) {
  const token = getBearerToken(request);        // 从 Authorization header 取 Bearer token
  const payload = parseSecureToken(token, secret()); // AES-256-GCM 解密 → JWT 验签 → 还原 payload
  const shareToken = await parseShareToken(request);  // 独立的 Share Token 通道

  let user = null;
  const { userId, authKey } = payload || {};

  if (userId) {                                  // 路径 A: payload 直接含 userId
    user = await getUser(userId);                //    → 按 UUID 查 User 表
  } else if (redis.enabled && authKey) {         // 路径 B: payload 含 authKey
    const key = await redis.client.get(authKey); //    → 先从 Redis 取出 { userId }
    if (key?.userId) {
      user = await getUser(key.userId);          //    → 再按 UUID 查 User 表
    }
  }

  if (!user?.id && !shareToken) {
    return null;  // 认证失败
  }

  // Share Token 上下文检查...

  if (user) {
    user.isAdmin = user.role === ROLES.admin;
  }

  return { token, authKey, shareToken, user };
}
```

**核心结论**：
- `checkAuth` **从不**按 `username` 或 `email` 查用户
- 只按 `userId`（UUID）或 `authKey`（Redis 键）还原用户
- Share Token 是独立通道，不参与 SSO 登录态

### 5.2 saveAuth：Redis 会话令牌签发

**文件**: `src/lib/auth.ts`

```typescript
export async function saveAuth(data: any, expire = 0) {
  if (!redis.enabled) {
    throw new Error('Redis is disabled');
  }

  const { id, ...rest } = data;
  const authKey = `auth:${createAuthKey()}`;  // 生成随机 key: "auth:<16位随机>"

  await redis.client.set(authKey, { userId: id, ...rest });

  if (expire) {
    await redis.client.expire(authKey, expire);  // SSO 接口默认 86400s
  }

  return createSecureToken({ authKey }, secret());  // 签发含 authKey 的 Secure Token
}
```

**saveAuth 的两个调用点**：
1. `src/app/api/auth/login/route.ts` — 本地登录，`saveAuth({ userId, role })`
2. `src/app/api/auth/sso/route.ts` — SSO 转换，`saveAuth({ userId: auth.user.id }, 86400)`

### 5.3 Token 加解密链路

**文件**: `src/lib/jwt.ts`

```typescript
// 签发：payload → JWT 签名 → AES-256-GCM 加密 → Secure Token
export function createSecureToken(payload: any, secret: any, options?: any) {
  return encrypt(createToken(payload, secret, options), secret);
}

// 验证：Secure Token → AES-256-GCM 解密 → JWT 验签 → payload
export function parseSecureToken(token: string, secret: any) {
  try {
    return jwt.verify(decrypt(token, secret), secret);
  } catch {
    return null;
  }
}
```

---

## 6. 本地用户匹配（外部适配层职责）

### 6.1 用户模型

**文件**: `prisma/schema.prisma`

```prisma
model User {
  id          String    @id() @map("user_id") @db.Uuid
  username    String    @unique @db.VarChar(255)  // 唯一索引
  password    String    @db.VarChar(60)
  role        String    @map("role") @db.VarChar(50)
  logoUrl     String?   @map("logo_url") @db.VarChar(2183)
  displayName String?   @map("display_name") @db.VarChar(255)
  createdAt   DateTime? @default(now()) @map("created_at") @db.Timestamptz(6)
  updatedAt   DateTime? @updatedAt @map("updated_at") @db.Timestamptz(6)
  deletedAt   DateTime? @map("deleted_at") @db.Timestamptz(6)
}
```

### 6.2 匹配规则

| 匹配方式 | 职责层 | 状态 | 说明 |
|---------|--------|------|------|
| `username` 精确匹配 | 外部适配层 | ✅ 可用 | `getUserByUsername` — 唯一的匹配函数 |
| 邮箱域名匹配 | 外部适配层 | ❌ 需自建 | 无 `email` 字段，无域名解析逻辑 |
| 外部 ID 匹配 | 外部适配层 | ❌ 需自建 | 无 `external_id` 字段 |
| userId UUID 查询 | 应用层 | ✅ 已实现 | `getUser(userId)` — 应用层唯一使用的查询 |

### 6.3 用户创建（手动方式）

**文件**: `src/app/api/users/route.ts`

```typescript
export async function POST(request: Request) {
  // 仅 admin 可创建
  const existingUser = await getUserByUsername(username, { showDeleted: true });

  if (existingUser) {
    return badRequest({ message: 'User already exists' });
  }

  const user = await createUser({
    id: id || uuid(),
    username,
    password: hashPassword(password),
    role: role ?? ROLES.user,
  });

  return json(user);
}
```

> **⚠️ JIT 自动创建用户未实现**: 开源版本中不存在 SSO 场景下的自动创建用户逻辑。

---

## 7. 邮件域到 Team 映射（❌ 未实现）

### 7.1 团队数据模型

```prisma
model Team {
  id         String    @id() @map("team_id") @db.Uuid
  name       String    @db.VarChar(50)
  accessCode String?   @unique @map("access_code") @db.VarChar(50)
  // ...
  members    TeamUser[]
}

model TeamUser {
  id        String    @id() @map("team_user_id") @db.Uuid
  teamId    String    @map("team_id") @db.Uuid
  userId    String    @map("user_id") @db.Uuid
  role      String    @db.VarChar(50)
  team Team @relation(fields: [teamId], references: [id])
  user User @relation(fields: [userId], references: [id])
}
```

### 7.2 团队加入方式（开源已实现）

| 方式 | 文件 | 说明 |
|------|------|------|
| 访问码加入 | `src/app/api/teams/join/route.ts` | 用户提供 `accessCode`，默认角色 `team-member` |
| 管理员添加 | `src/app/api/teams/[teamId]/users/route.ts` | 需 `canUpdateTeam` 权限，可指定角色 |

### 7.3 邮件域自动映射（❌ 未实现）

开源版本中**不存在**基于邮箱域名自动加入团队的逻辑。

| 相关功能 | 实现状态 | 说明 |
|---------|---------|------|
| 企业域名配置 | ❌ 未实现 | 无相关配置项和数据库字段 |
| 邮箱域 → Team 绑定 | ❌ 未实现 | 无映射关系表 |
| SSO 登录自动加入 Team | ❌ 未实现 | 登录流程中无自动加入代码 |

> **扩展建议**: 在外部适配层中，SSO 认证完成后、签发令牌前，根据用户邮箱后缀查询匹配的 Team，调用 `createTeamUser` 自动建立成员关系。

---

## 8. JIT 配置（❌ 未实现）

| JIT 功能 | 实现状态 | 说明 |
|---------|---------|------|
| JIT 自动创建用户 | ❌ 未实现 | 开源版本无相关代码 |
| 邮箱域白名单 | ❌ 未实现 | 无域名白名单校验逻辑 |
| 默认角色配置 | ❌ 未实现 | 无 JIT 默认角色配置项 |
| 自动加入默认 Team | ❌ 未实现 | 无相关逻辑 |

> **扩展建议**: 在外部适配层中，`getUserByUsername` 返回空时，调用 `createUser` 创建用户（自动生成随机密码），再根据邮件域映射调用 `createTeamUser` 加入团队，最后签发含新 `userId` 的 Secure Token。

---

## 9. 角色映射规则

### 9.1 系统级角色（开源已实现）

| 角色 | 常量值 | 说明 |
|------|--------|------|
| admin | `admin` | 系统管理员，拥有所有权限 |
| user | `user` | 普通用户，可创建网站和团队 |
| view-only | `view-only` | 只读用户（系统级，预留） |

### 9.2 团队级角色（开源已实现）

| 角色 | 常量值 | 说明 |
|------|--------|------|
| team-owner | `teamOwner` | 团队所有者 |
| team-manager | `teamManager` | 团队管理员 |
| team-member | `teamMember` | 团队成员 |
| team-view-only | `teamViewOnly` | 团队只读成员 |

### 9.3 IdP 角色/组映射（❌ 未实现）

开源版本中**不存在**任何将 IdP 用户组、角色声明映射到 Umami 角色的逻辑。

| 映射功能 | 实现状态 | 说明 |
|---------|---------|------|
| IdP 组 → 系统角色映射 | ❌ 未实现 | 无相关配置和代码 |
| IdP 组 → 团队角色映射 | ❌ 未实现 | 无相关配置和代码 |
| 角色断言解析 | ❌ 未实现 | 无 SAML/OAuth 断言解析代码 |

---

## 10. 账号归属决策树（开源版本实际行为）

```
SSO 回调：/sso?token=<jwt>&url=<target>
            │
            ▼
 setClientAuthToken(token) → localStorage
            │
            ▼
 跳转到 <target>，App 组件加载
            │
            ▼
 useLoginQuery 自动触发
            │
            ▼
 POST /api/auth/verify
 Authorization: Bearer <回调token>
            │
            ▼
    ┌──────────────────────────┐
    │  checkAuth               │
    │  parseSecureToken → payload
    └────────────┬─────────────┘
                 │
         ┌───────┴───────┐
         │               │
    payload 有 userId   payload 有 authKey
         │               │
         ▼               ▼
    getUser(userId)    Redis.get(authKey)
         │               │
         │          ┌────┴────┐
         │          │         │
         │        有值       无值
         │          │         │
         │          ▼         ▼
         │     getUser        → null
         │     (key.userId)  (认证失败 → 前端重定向登录页)
         │          │
    ┌────┴──────────┴────┐
    │                     │
   用户存在            用户不存在
    │                     │
    ▼                     ▼
  getAllUserTeams      → null
  返回 { user, teams }  (认证失败 → 前端重定向登录页)
    │
    ▼
  setUser(data) → 登录成功
```

---

## 11. 关键代码文件索引

| 模块 | 文件路径 | 职责层 | 说明 |
|------|---------|--------|------|
| **Token 签发** | | | |
| 本地登录签发 | `src/app/api/auth/login/route.ts` | 内置适配 | 唯一用 username 定位用户的认证接口 |
| SSO 会话转换 | `src/app/api/auth/sso/route.ts` | 应用层 | 已有 Token → Redis authKey 新 Token |
| 会话签发函数 | `src/lib/auth.ts` (saveAuth) | 应用层 | 写 Redis + 签发 authKey 令牌 |
| Secure Token | `src/lib/jwt.ts` | 应用层 | createSecureToken / parseSecureToken |
| Share Token 签发 | `src/app/api/share/[slug]/route.ts` | 独立体系 | 共享链接令牌，与 SSO 无关 |
| **Token 验证** | | | |
| 验证接口 | `src/app/api/auth/verify/route.ts` | 应用层 | 仅验证，返回用户 + teams，不签发新 Token |
| 当前用户接口 | `src/app/api/me/route.ts` | 应用层 | 返回原始 auth 对象 |
| 认证核心 | `src/lib/auth.ts` (checkAuth) | 应用层 | 按 userId/authKey 还原用户 |
| **客户端 Token** | | | |
| SSO 回调页面 | `src/app/sso/SSOPage.tsx` | 前端 | 存 token + 跳转，不调 API |
| 登录表单 | `src/app/login/LoginForm.tsx` | 前端 | 本地登录，存 token |
| 客户端工具 | `src/lib/client.ts` | 前端 | get/set/remove localStorage token |
| 请求钩子 | `src/components/hooks/useApi.ts` | 前端 | 所有请求自动注入 Bearer token |
| 登录查询 | `src/components/hooks/queries/useLoginQuery.ts` | 前端 | 应用加载时调 /auth/verify |
| 登出页面 | `src/app/logout/LogoutPage.tsx` | 前端 | 删 localStorage |
| 登出 API | `src/app/api/auth/logout/route.ts` | 应用层 | 删 Redis authKey |
| **数据层** | | | |
| 用户查询 | `src/queries/prisma/user.ts` | 跨层 | getUser（应用层）/ getUserByUsername（适配层） |
| 团队查询 | `src/queries/prisma/team.ts` | 跨层 | getTeam, getUserTeams |
| 团队成员 | `src/queries/prisma/teamUser.ts` | 跨层 | getTeamUser, createTeamUser |
| 角色常量 | `src/lib/constants.ts` | 应用层 | ROLES, PERMISSIONS, ROLE_PERMISSIONS |
| 数据模型 | `prisma/schema.prisma` | 跨层 | User, Team, TeamUser 模型定义 |
| **加密** | | | |
| 加密工具 | `src/lib/crypto.ts` | 应用层 | AES-256-GCM / secret 派生 / createAuthKey |

---

## 12. 总结

### 12.1 四个核心接口的边界口诀

- **`/api/auth/login`**：身份验证 + Token 签发二合一，用 username 找用户
- **`/api/auth/sso`**：已有 Token 转 Redis 会话，只认 userId/authKey，必须有 Redis
- **`/api/auth/verify`**：纯验证不签发，返回用户态，前端每次加载都调它
- **`/sso` 页面**：纯客户端存 token，不调 API，验证留给后续请求

### 12.2 Token 流向的本质

SSO 登录链路中 Token 的流动路径：

1. **外部适配层**签发含 `userId` 的 Secure Token → 重定向到 `/sso?token=<jwt>`
2. **前端 SSOPage**直接存入 localStorage → 自动作为所有请求的 Bearer 凭证
3. **应用层 verify/checkAuth**根据 payload 中的 `userId`（或 `authKey → Redis → userId`）还原用户对象
4. **可选的 `/api/auth/sso`**把回调 Token 换成 Redis 会话 Token（带 TTL）

前端**不主动调用** `/api/auth/sso`，回调 Token 被直接使用。

### 12.3 两层边界的本质

- **外部适配层**的工作是**身份解析**：把 IdP 返回的用户名/邮箱翻译成本地 `userId`（UUID），完成用户创建、团队映射、角色映射，然后签发含 `userId` 的 Secure Token
- **应用层**的工作是**身份还原**：从 Secure Token 中取出 `userId` 或 `authKey`，还原用户对象，执行权限检查
- 两层之间的**唯一契约**是 JWT payload 中的 `userId` 字段

### 12.4 开源版本已实现的能力

1. **Secure Token 体系**：AES-256-GCM 加密 + JWT 签名，两种 payload 格式
2. **Token 承载与注入**：SSOPage 存 token + useApi 自动注入 Bearer 头
3. **Token 校验与用户还原**：`checkAuth` 按 userId/authKey 还原用户对象
4. **Session 管理**：`saveAuth` 写 Redis，签发带 TTL 的 authKey 令牌
5. **身份验证接口**：`/api/auth/verify` 提供用户信息和团队列表
6. **权限系统**：系统级 + 团队级双层角色体系
7. **SSO 接入点**：`/sso` 页面和 `/api/auth/sso` 接口
8. **可复用的数据操作**：`getUserByUsername`、`createUser`、`createTeamUser` 等

### 12.5 需自行扩展或依赖 Cloud 服务的能力

| 功能 | 实现位置 | 可复用的开源函数 |
|------|---------|-----------------|
| OAuth/SAML 协议处理 | 外部适配层 | 无 |
| 用户名/邮箱 → userId 解析 | 外部适配层 | `getUserByUsername` |
| JIT 自动创建用户 | 外部适配层 | `createUser` |
| 邮件域 → Team 映射 | 外部适配层 | `createTeamUser` |
| IdP 角色/组映射 | 外部适配层 | `updateUser`, `createTeamUser` |
| Secure Token 签发 | 外部适配层 | `createSecureToken`, `secret` |

### 12.6 核心账号归属规则

1. **Token 直用**：SSO 回调 Token 直接存 localStorage，直接作为 Bearer 凭证
2. **两层分离**：外部适配层用 `username`/`email` 定位账号，应用层只凭 `userId`/`authKey` 校验令牌
3. **契约是 userId**：JWT payload 中的 `userId` 是两层之间的唯一传递物
4. **无自动创建**：用户不存在时应用层直接拒绝，JIT 需在外部适配层实现
5. **团队手动加入**：需通过访问码或管理员添加，无自动分配
6. **角色双层结构**：系统级角色决定全局权限，团队级角色决定团队内权限
7. **管理员优先**：系统管理员 (admin) 绕过所有团队权限检查
8. **SSO API 可选**：`/api/auth/sso` 是服务端内部转换接口，前端 SSO 路径不调用它
