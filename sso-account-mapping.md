# Umami SSO 账号归属规则分析

## 概述

本文档基于 Umami v3.1.0 开源版本代码库，分析企业 SSO 登录链路的账号归属规则。

> **重要说明**：开源版本仅提供 SSO 接入的应用层框架。完整的 OAuth/SAML 协议处理、JIT 用户自动创建、邮件域到 Team 自动映射、IdP 角色组映射等企业级 SSO 功能**未在开源版本中实现**，需依赖 Cloud 商业服务或自行构建外部适配层。

---

## 1. 两层边界：外部适配层 vs 应用层

SSO 登录链路存在一条清晰的职责分界线——**令牌签发**。外部适配层用用户名或邮箱定位账号并签发令牌；应用层只校验令牌里的用户编号或会话键，不再接触用户名/邮箱。

### 1.1 边界划分

```
                         ← 职责边界：令牌签发 →
                         ─────────────────────

  外部适配层（开源未实现）           应用层（开源已实现）
  ─────────────────────           ─────────────────────
  输入：用户名 / 邮箱               输入：JWT 令牌（含 userId 或 authKey）
  操作：                           操作：
   1. 对接 OAuth/SAML IdP          1. 解密 + 验签 JWT
   2. 用 username/email 查本地用户  2. 从 payload 取 userId 或 authKey
   3. 不存在则 JIT 创建用户          3. userId → getUser(id) 查库
   4. 邮件域 → Team 自动映射        4. authKey → Redis 取 { userId } 再查库
   5. IdP 组 → Umami 角色映射       5. 附加 isAdmin 标记，返回 auth 对象
   6. 签发含 userId 的 JWT 令牌      6. saveAuth 写 Redis，返回新令牌
  输出：签发好的 JWT 令牌            输出：应用 Session 令牌 + 用户信息
```

### 1.2 关键差异

| 维度 | 外部适配层 | 应用层 |
|------|-----------|--------|
| **用户定位方式** | `username` / `email`（可读标识） | `userId`（UUID）/ `authKey`（Redis 会话键） |
| **核心操作** | 把 IdP 身份映射为本地 userId | 用 userId 或 authKey 还原用户对象 |
| **涉及代码** | 不在开源仓库中 | `src/lib/auth.ts` · `src/app/api/auth/sso/route.ts` |
| **是否可独立工作** | 否，需调用应用层 API 写入用户 | 是，但不做账号创建和协议适配 |
| **匹配函数** | `getUserByUsername(username)` | `getUser(userId)` — 只按 UUID 查 |

---

## 2. 应用层令牌校验机制（开源已实现）

应用层的一切认证都从解析 JWT 令牌开始，只认两种 payload 结构。

### 2.1 两种令牌 payload

**令牌类型 A — 内嵌 userId（无 Redis 时）**

```typescript
// 签发：src/app/api/auth/login/route.ts#L39
token = createSecureToken({ userId: user.id, role }, secret());

// payload 结构：
{ userId: "01234567-89ab-cdef-...", role: "user" }
```

**令牌类型 B — 内嵌 authKey（有 Redis 时）**

```typescript
// 签发：src/lib/auth.ts#L62-74
// saveAuth 先把 { userId, role } 写入 Redis，key 为 "auth:<random>"
// 再签发只含 authKey 的 JWT
const authKey = `auth:${createAuthKey()}`;
await redis.client.set(authKey, { userId: id, role });
return createSecureToken({ authKey }, secret());

// payload 结构：
{ authKey: "auth:a1b2c3d4e5f6..." }
```

### 2.2 checkAuth：应用层的唯一认证入口

**文件**: `src/lib/auth.ts`

```typescript
export async function checkAuth(request: Request) {
  const token = getBearerToken(request);        // 1. 从 Authorization header 取 Bearer token
  const payload = parseSecureToken(token, secret()); // 2. AES-256-GCM 解密 → JWT 验签 → 还原 payload
  const shareToken = await parseShareToken(request);  // 3. 独立的共享令牌通道

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

  // ... 认证失败处理、isAdmin 标记 ...

  return { token, authKey, shareToken, user };
}
```

**核心结论**：

- `checkAuth` **从不**按 `username` 或 `email` 查用户，只按 `userId`（UUID）或 `authKey`（Redis 键）
- `username` / `email` 的解析和匹配完全在外部适配层完成
- 外部适配层的职责就是：把 IdP 返回的用户名/邮箱解析为本地 `userId`，然后签发含该 `userId` 的令牌

### 2.3 两条路径的完整对比

以 `POST /api/auth/login`（本地登录）和 `POST /api/auth/sso`（SSO 登录）为例，观察令牌在不同路径下的流转：

**本地登录路径**（`src/app/api/auth/login/route.ts`）:

```
用户提交 { username, password }
       │
       ▼
getUserByUsername(username)  ← 用 username 查库，拿到 userId
       │
       ▼
checkPassword(password, user.password)  ← 验证密码
       │
       ▼
Redis 模式: saveAuth({ userId: id, role })  → 返回含 authKey 的 JWT
无 Redis:   createSecureToken({ userId, role }, secret())  → 返回含 userId 的 JWT
```

> 注意：本地登录路径本身就在做"用户名 → userId"的适配，相当于一个最小化的内置适配层。`getUserByUsername` 只在这一处被认证流程调用。

**SSO 登录路径**（`src/app/api/auth/sso/route.ts`）:

```
请求已携带有效的 JWT（含 userId 或 authKey）
       │
       ▼
parseRequest → checkAuth  ← 只验令牌，不做 username 查找
       │
       ▼
auth.user.id 已存在  ← userId 由外部适配层在令牌中提供
       │
       ▼
saveAuth({ userId: auth.user.id }, 86400)  ← 将 userId 存入 Redis，返回新 authKey JWT
```

> 注意：SSO API **不再做任何用户名匹配**。它只做一件事：把已验证的 userId 存入 Redis，签发新的应用 Session 令牌。用户名/邮箱到 userId 的解析工作已由外部适配层完成。

### 2.4 SSO API 的前置条件

`POST /api/auth/sso` 要求请求本身已携带有效令牌（即请求必须通过 `checkAuth`）：

```typescript
// src/app/api/auth/sso/route.ts
export async function POST(request: Request) {
  const { auth, error } = await parseRequest(request);  // 内部调用 checkAuth

  if (error) {
    return error();  // 令牌无效 → 401
  }

  if (!redis.enabled) {
    return serverError({ message: 'Redis is disabled' });  // 无 Redis → 500
  }

  // 此时 auth.user.id 已由 checkAuth 通过 userId 或 authKey 还原
  const token = await saveAuth({ userId: auth.user.id }, 86400);

  return json({ user: auth.user, token });
}
```

这意味着调用 SSO API 时，**令牌中的 userId 必须对应一个已存在的 User 记录**。如果用户不存在于数据库，`checkAuth` 会返回 `null`，请求直接被拒绝为 401。**不存在 JIT 自动创建用户的逻辑。**

---

## 3. 外部适配层职责（开源未实现）

### 3.1 外部适配层必须完成的工作

在令牌到达应用层之前，外部适配层需要完成以下全部工作：

| 步骤 | 操作 | 对应的开源函数（可复用） |
|------|------|------------------------|
| 1 | 对接 OAuth/SAML IdP，完成协议交互 | ❌ 无现成代码 |
| 2 | 从 IdP 断言中提取用户标识（username/email） | ❌ 无现成代码 |
| 3 | 用 `getUserByUsername(username)` 查找本地用户 | ✅ `src/queries/prisma/user.ts` |
| 4a | 用户存在 → 读取其 `id`（UUID） | ✅ `getUserByUsername` 返回值含 `id` |
| 4b | 用户不存在 → JIT 创建 → 读取新 `id` | ⚠️ 可复用 `createUser`，但调用逻辑需自建 |
| 5 | 根据邮箱域映射，自动加入 Team | ⚠️ 可复用 `createTeamUser`，但映射逻辑需自建 |
| 6 | 根据 IdP 组映射，设定角色 | ⚠️ 可写入 `role` / `TeamUser.role`，但映射逻辑需自建 |
| 7 | 签发含 `userId` 的 JWT 令牌 | ✅ `createSecureToken({ userId, role }, secret())` |

### 3.2 用户名/邮箱匹配：只发生在外部适配层

开源代码中，`getUserByUsername` 在认证流程中的调用点**仅有一处**：

```typescript
// src/app/api/auth/login/route.ts#L26
const user = await getUserByUsername(username, { includePassword: true });
```

这是本地登录的入口。SSO 路径（`/api/auth/sso`）**不调用** `getUserByUsername`。

换言之：
- **外部适配层**负责"用户名/邮箱 → userId"的解析
- **应用层**只负责"userId/authKey → 用户对象"的还原
- 两层之间的**契约**就是 JWT payload 中的 `userId` 字段

### 3.3 令牌签发：两层之间的唯一握手点

外部适配层签发令牌的方式有两种：

**方式 A：直接签发内嵌 userId 的 JWT**

```typescript
import { createSecureToken } from '@/lib/jwt';
import { secret } from '@/lib/crypto';

const token = createSecureToken({ userId: user.id, role: user.role }, secret());
```

此令牌可被应用层 `checkAuth` 的路径 A 直接解析，无需 Redis。

**方式 B：通过 Redis authKey 签发**

```typescript
import { saveAuth } from '@/lib/auth';

const token = await saveAuth({ userId: user.id, role: user.role }, 86400);
```

此令牌由 `saveAuth` 将 `{ userId, role }` 写入 Redis（key: `auth:<random>`），返回的 JWT payload 只含 `authKey`。

> 两种方式的区别仅在于令牌体积和是否依赖 Redis，对应用层 `checkAuth` 而言都是透明的。

### 3.4 SSO 前端回调

**文件**: `src/app/sso/SSOPage.tsx`

```typescript
export function SSOPage() {
  const router = useRouter();
  const search = useSearchParams();
  const url = search.get('url');     // 登录后跳转目标
  const token = search.get('token'); // 外部适配层已签发的 JWT

  useEffect(() => {
    if (url && token) {
      setClientAuthToken(token);  // 存入 localStorage (key: "umami.auth")
      router.push(url);
    }
  }, [router, url, token]);

  return <Loading placement="absolute" />;
}
```

外部适配层完成认证后，将用户重定向到 `/sso?url=<target>&token=<jwt>`。前端仅做两件事：存储令牌、跳转页面。后续所有 API 请求都会在 `Authorization: Bearer <token>` 中携带此令牌。

---

## 4. 本地用户匹配（外部适配层职责）

### 4.1 用户模型

**文件**: `prisma/schema.prisma`

```prisma
model User {
  id          String    @id() @map("user_id") @db.Uuid
  username    String    @unique @db.VarChar(255)  // 唯一索引
  password    String    @db.VarChar(60)
  role        String    @map("role") @db.VarChar(50)
  // ...
}
```

### 4.2 匹配规则

| 匹配方式 | 职责层 | 状态 | 说明 |
|---------|--------|------|------|
| `username` 精确匹配 | 外部适配层 | ✅ 可用 | `getUserByUsername` — 唯一的匹配函数 |
| 邮箱域名匹配 | 外部适配层 | ❌ 需自建 | 无 `email` 字段，无域名解析逻辑 |
| 外部 ID 匹配 | 外部适配层 | ❌ 需自建 | 无 `external_id` 字段 |
| userId UUID 查询 | 应用层 | ✅ 已实现 | `getUser(userId)` — 应用层唯一使用的查询 |

### 4.3 用户创建（手动方式）

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

> **⚠️ JIT 自动创建用户未实现**: 开源版本中不存在 SSO 场景下的自动创建用户逻辑。如果外部适配层通过 `getUserByUsername` 查不到用户，需自行决定是创建还是拒绝。

---

## 5. 邮件域到 Team 映射（❌ 未实现）

### 5.1 团队加入方式（开源已实现）

| 方式 | 文件 | 说明 |
|------|------|------|
| 访问码加入 | `src/app/api/teams/join/route.ts` | 用户提供 `accessCode`，默认角色 `team-member` |
| 管理员添加 | `src/app/api/teams/[teamId]/users/route.ts` | 需 `canUpdateTeam` 权限，可指定角色 |

### 5.2 邮件域自动映射（❌ 未实现）

开源版本中**不存在**基于邮箱域名自动加入团队的逻辑。

| 相关功能 | 实现状态 | 说明 |
|---------|---------|------|
| 企业域名配置 | ❌ 未实现 | 无相关配置项和数据库字段 |
| 邮箱域 → Team 绑定 | ❌ 未实现 | 无映射关系表 |
| SSO 登录自动加入 Team | ❌ 未实现 | 登录流程中无自动加入代码 |

> **扩展建议**: 在外部适配层中，SSO 认证完成后、签发令牌前，根据用户邮箱后缀查询匹配的 Team，调用 `createTeamUser` 自动建立成员关系。

---

## 6. JIT 配置（❌ 未实现）

| JIT 功能 | 实现状态 | 说明 |
|---------|---------|------|
| JIT 自动创建用户 | ❌ 未实现 | 开源版本无相关代码 |
| 邮箱域白名单 | ❌ 未实现 | 无域名白名单校验逻辑 |
| 默认角色配置 | ❌ 未实现 | 无 JIT 默认角色配置项 |
| 自动加入默认 Team | ❌ 未实现 | 无相关逻辑 |

> **扩展建议**: 在外部适配层中，`getUserByUsername` 返回空时，调用 `createUser` 创建用户（自动生成随机密码），再根据邮件域映射调用 `createTeamUser` 加入团队，最后签发含新 `userId` 的令牌。

---

## 7. 角色映射规则

### 7.1 系统级角色（开源已实现）

**定义**: `src/lib/constants.ts`

| 角色 | 常量值 | 说明 |
|------|--------|------|
| admin | `admin` | 系统管理员，拥有所有权限 |
| user | `user` | 普通用户，可创建网站和团队 |
| view-only | `view-only` | 只读用户（系统级，预留） |

### 7.2 团队级角色（开源已实现）

| 角色 | 常量值 | 说明 |
|------|--------|------|
| team-owner | `teamOwner` | 团队所有者 |
| team-manager | `teamManager` | 团队管理员 |
| team-member | `teamMember` | 团队成员 |
| team-view-only | `teamViewOnly` | 团队只读成员 |

### 7.3 角色权限映射（开源已实现）

| 权限 | admin | user | view-only | team-owner | team-manager | team-member | team-view-only |
|------|-------|------|-----------|------------|--------------|-------------|----------------|
| `all` | ✅ | - | - | - | - | - | - |
| `website:create` | ✅ | ✅ | - | ✅ | ✅ | ✅ | - |
| `website:update` | ✅ | ✅ | - | ✅ | ✅ | ✅ | - |
| `website:delete` | ✅ | ✅ | - | ✅ | ✅ | ✅ | - |
| `website:transfer-to-team` | ✅ | - | - | ✅ | ✅ | - | - |
| `website:transfer-to-user` | ✅ | - | - | ✅ | - | - | - |
| `team:create` | ✅ | ✅ | - | - | - | - | - |
| `team:update` | ✅ | - | - | ✅ | ✅ | - | - |
| `team:delete` | ✅ | - | - | ✅ | - | - | - |

### 7.4 IdP 角色/组映射（❌ 未实现）

开源版本中**不存在**任何将 IdP 用户组、角色声明映射到 Umami 角色的逻辑。

| 映射功能 | 实现状态 | 说明 |
|---------|---------|------|
| IdP 组 → 系统角色映射 | ❌ 未实现 | 无相关配置和代码 |
| IdP 组 → 团队角色映射 | ❌ 未实现 | 无相关配置和代码 |
| 角色断言解析 | ❌ 未实现 | 无 SAML/OAuth 断言解析代码 |

> **扩展建议**: 在外部适配层中，从 IdP 断言/令牌中提取角色或组信息，根据映射规则调用 `updateUser` 设置系统角色，调用 `createTeamUser` 设置团队角色。

### 7.5 权限检查函数（开源已实现）

**文件**: `src/lib/auth.ts`

```typescript
export async function hasPermission(role: string, permission: string | string[]) {
  return ensureArray(permission).some(e => ROLE_PERMISSIONS[role]?.includes(e));
}
```

**文件**: `src/permissions/team.ts`

```typescript
export async function canUpdateTeam({ user }: Auth, teamId: string) {
  if (!user) return false;
  if (user.isAdmin) return true; // 系统管理员绕过团队权限检查
  const teamUser = await getTeamUser(teamId, user.id);
  return teamUser && hasPermission(teamUser.role, PERMISSIONS.teamUpdate);
}
```

---

## 8. 完整数据流：两层协作时序

```
  浏览器              外部适配层               应用层 (checkAuth)
    │                     │                        │
    │  1. 点击 SSO 登录   │                        │
    │────────────────────>│                        │
    │                     │                        │
    │                     │  2. 重定向到 IdP        │
    │                     │     (OAuth/SAML)       │
    │                     │                        │
    │  3. IdP 认证完成    │                        │
    │<────────────────────│                        │
    │                     │                        │
    │                     │  4. getUserByUsername   │
    │                     │     (username/email)   │
    │                     │  ──────────────────    │
    │                     │  5. 得到 userId (UUID)  │
    │                     │     或 JIT 创建后得到   │
    │                     │                        │
    │                     │  6. 邮件域→Team 映射    │
    │                     │     (如适用)            │
    │                     │                        │
    │                     │  7. IdP 组→角色映射     │
    │                     │     (如适用)            │
    │                     │                        │
    │                     │  8. 签发 JWT            │
    │                     │  payload: { userId }    │
    │                     │  或 { userId, role }    │
    │                     │                        │
    │  9. 重定向到 /sso   │                        │
    │     ?token=<jwt>    │                        │
    │     &url=<target>   │                        │
    │<────────────────────│                        │
    │                     │                        │
    │  10. 存 token 到    │                        │
    │      localStorage   │                        │
    │                     │                        │
    │  11. 后续 API 请求  │                        │
    │      Authorization: │                        │
    │      Bearer <jwt>   │                        │
    │─────────────────────────────────────────────>│
    │                     │                        │
    │                     │  12. checkAuth          │
    │                     │      parseSecureToken   │
    │                     │      → payload.userId   │
    │                     │      → getUser(userId)  │
    │                     │      → auth.user        │
    │                     │                        │
    │  13. 返回数据        │                        │
    │<─────────────────────────────────────────────│
```

---

## 9. 账号归属决策树（开源版本实际行为）

```
请求到达应用层 (Authorization: Bearer <jwt>)
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
         │     (key.userId)  (认证失败)
         │          │
    ┌────┴──────────┴────┐
    │                     │
   用户存在            用户不存在
    │                     │
    ▼                     ▼
  附加 isAdmin          → null
  返回 auth 对象        (认证失败)
    │
    ▼
  继续处理请求
```

> **关键结论**: 应用层只凭 `userId`（UUID）或 `authKey`（Redis 键）还原用户。`username` / `email` 的匹配、JIT 创建、团队映射、角色映射——全部在外部适配层完成，或根本不存在。

---

## 10. 关键代码文件索引

| 模块 | 文件路径 | 职责层 | 说明 |
|------|---------|--------|------|
| SSO 回调页面 | `src/app/sso/SSOPage.tsx` | 应用层 | 存储 token + 跳转 |
| SSO API | `src/app/api/auth/sso/route.ts` | 应用层 | 验令牌 → 创建应用 Session |
| 认证核心 | `src/lib/auth.ts` | 应用层 | checkAuth / saveAuth / hasPermission |
| JWT 处理 | `src/lib/jwt.ts` | 应用层 | Token 加密/解密/验签 |
| 加密工具 | `src/lib/crypto.ts` | 应用层 | AES-256-GCM + secret 派生 |
| 用户查询 | `src/queries/prisma/user.ts` | 跨层 | getUser（应用层）/ getUserByUsername（适配层）/ createUser |
| 团队查询 | `src/queries/prisma/team.ts` | 跨层 | getTeam, getUserTeams, createTeam |
| 团队成员 | `src/queries/prisma/teamUser.ts` | 跨层 | getTeamUser, createTeamUser |
| 角色常量 | `src/lib/constants.ts` | 应用层 | ROLES, PERMISSIONS, ROLE_PERMISSIONS |
| 团队权限 | `src/permissions/team.ts` | 应用层 | 团队权限校验函数 |
| 用户权限 | `src/permissions/user.ts` | 应用层 | 用户权限校验函数 |
| 数据模型 | `prisma/schema.prisma` | 跨层 | User, Team, TeamUser 模型定义 |
| 请求解析 | `src/lib/request.ts` | 应用层 | parseRequest → 调用 checkAuth |
| 本地登录 | `src/app/api/auth/login/route.ts` | 内置适配 | 唯一调用 getUserByUsername 的认证入口 |

---

## 11. 总结

### 11.1 两层边界的本质

- **外部适配层**的工作是**身份解析**：把 IdP 返回的用户名/邮箱翻译成本地 `userId`（UUID），完成用户创建、团队映射、角色映射，然后签发令牌
- **应用层**的工作是**身份还原**：从令牌中取出 `userId` 或 `authKey`，还原用户对象，执行权限检查
- 两层之间的**唯一契约**是 JWT payload 中的 `userId` 字段

### 11.2 开源版本已实现的能力

1. **令牌校验与用户还原**: `checkAuth` 按 userId/authKey 还原用户对象
2. **Session 管理**: `saveAuth` 写 Redis，`createSecureToken` 签 JWT
3. **权限系统**: 系统级 + 团队级双层角色体系
4. **SSO 接入点**: `/sso` 页面和 `/api/auth/sso` 接口
5. **可复用的数据操作**: `getUserByUsername`、`createUser`、`createTeamUser` 等函数

### 11.3 需自行扩展或依赖 Cloud 服务的能力

| 功能 | 实现位置 | 可复用的开源函数 |
|------|---------|-----------------|
| OAuth/SAML 协议处理 | 外部适配层 | 无 |
| 用户名/邮箱 → userId 解析 | 外部适配层 | `getUserByUsername` |
| JIT 自动创建用户 | 外部适配层 | `createUser` |
| 邮件域 → Team 映射 | 外部适配层 | `createTeamUser` |
| IdP 角色/组映射 | 外部适配层 | `updateUser`, `createTeamUser` |
| JWT 令牌签发 | 外部适配层 | `createSecureToken`, `secret` |

### 11.4 核心账号归属规则

1. **两层分离**: 外部适配层用 `username`/`email` 定位账号，应用层只凭 `userId`/`authKey` 校验令牌
2. **契约是 userId**: JWT payload 中的 `userId` 是两层之间的唯一传递物
3. **无自动创建**: 用户不存在时应用层直接拒绝，JIT 需在外部适配层实现
4. **团队手动加入**: 需通过访问码或管理员添加，无自动分配
5. **角色双层结构**: 系统级角色决定全局权限，团队级角色决定团队内权限
6. **管理员优先**: 系统管理员 (admin) 绕过所有团队权限检查
