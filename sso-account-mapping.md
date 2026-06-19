# Umami SSO 账号归属规则分析

## 概述

本文档基于 Umami v3.1.0 开源版本代码库，分析企业 SSO 登录链路的账号归属规则。

> **重要说明**：开源版本仅提供 SSO 接入的基础框架和用户/团队数据模型。完整的 OAuth/SAML 协议处理、JIT 用户自动创建、邮件域到 Team 自动映射、IdP 角色组映射等企业级 SSO 功能**未在开源版本中实现**，需依赖 Cloud 商业服务或自行扩展。

---

## 1. 实现状态总览

| 功能模块 | 开源版本实现状态 | 说明 |
|---------|-----------------|------|
| SSO 回调入口页面 | ✅ 已实现 | `/sso` 页面，接收 token 和 url 参数 |
| SSO 认证 API | ✅ 已实现 | `/api/auth/sso`，验证并创建应用 Session |
| 本地用户匹配 | ✅ 已实现 | 通过 `username` 字段精确匹配 |
| 团队与成员模型 | ✅ 已实现 | Team + TeamUser 关联表 |
| 角色与权限系统 | ✅ 已实现 | 系统级 + 团队级双层角色体系 |
| OAuth 协议处理 | ❌ 未实现 | 依赖外部服务（Cloud/自建 IdP 适配层） |
| SAML 协议处理 | ❌ 未实现 | 依赖外部服务（Cloud/自建 IdP 适配层） |
| JIT 自动创建用户 | ❌ 未实现 | 需自行扩展或使用 Cloud 服务 |
| 邮件域 → Team 自动映射 | ❌ 未实现 | 需自行扩展或使用 Cloud 服务 |
| IdP 角色/组映射 | ❌ 未实现 | 需自行扩展或使用 Cloud 服务 |

---

## 2. SSO 整体架构

### 2.1 分层架构

```
┌───────────────────────────────────────────────────┐
│  企业 IdP (OAuth / SAML)                          │
│  （第三方服务，非 Umami 代码）                    │
└──────────────────────┬────────────────────────────┘
                       │ SSO 认证
                       ▼
┌───────────────────────────────────────────────────┐
│  SSO 适配层                                       │
│  ⚠️  开源版本未实现 — 需自建或使用 Cloud 服务     │
│  • OAuth/SAML 协议处理                            │
│  • 回调验证与用户信息解析                          │
│  • JIT 用户创建逻辑                               │
│  • 邮件域 → Team 映射                             │
│  • IdP 角色 → Umami 角色映射                      │
└──────────────────────┬────────────────────────────┘
                       │ 传递认证后的 userId / token
                       ▼
┌───────────────────────────────────────────────────┐
│  Umami 应用层（开源版本已实现）                    │
│  • /api/auth/sso 接收认证                          │
│  • 本地用户匹配 (username)                         │
│  • 权限验证 (Auth + TeamUser)                      │
│  • Session 管理 (Redis / JWT)                      │
└───────────────────────────────────────────────────┘
```

### 2.2 关键入口点

| 入口 | 文件路径 | 作用 |
|------|---------|------|
| SSO 页面 | `src/app/sso/page.tsx` | 接收 SSO 回调后的 url 和 token，完成客户端登录跳转 |
| SSO API | `src/app/api/auth/sso/route.ts` | 验证 SSO 认证，生成应用 Session Token |
| 登录页 | `src/app/login/page.tsx` | Cloud 模式下禁用本地登录页 |

---

## 3. SSO 回调流程（开源已实现部分）

### 3.1 SSO 回调页面

**文件**: `src/app/sso/SSOPage.tsx`

```typescript
export function SSOPage() {
  const router = useRouter();
  const search = useSearchParams();
  const url = search.get('url');    // 登录后跳转的目标 URL
  const token = search.get('token'); // SSO 认证 Token

  useEffect(() => {
    if (url && token) {
      setClientAuthToken(token);  // 存储 Token 到客户端 (localStorage)
      router.push(url);           // 跳转到目标页面
    }
  }, [router, url, token]);

  return <Loading placement="absolute" />;
}
```

**流程说明**:
1. 用户从 SSO 适配层完成认证后，重定向回 Umami 的 `/sso` 页面
2. URL 携带两个参数：
   - `token`: 已验证的认证凭证（由 SSO 适配层签发）
   - `url`: 登录成功后跳转的目标页面
3. 前端将 Token 存入本地存储（key: `umami.auth`）
4. 前端跳转到目标页面，后续请求通过 `Authorization: Bearer <token>` 携带

### 3.2 SSO 认证 API

**文件**: `src/app/api/auth/sso/route.ts`

```typescript
export async function POST(request: Request) {
  const { auth, error } = await parseRequest(request);

  if (error) {
    return error();
  }

  if (!redis.enabled) {
    return serverError({ message: 'Redis is disabled' });
  }

  const token = await saveAuth({ userId: auth.user.id }, 86400);

  return json({ user: auth.user, token });
}
```

**验证流程**:
1. `parseRequest` 调用 `checkAuth` 解析请求中的认证信息
2. `checkAuth` 从 JWT Token 中解析 `userId` 或 `authKey`
3. 通过 `getUser(userId)` 从数据库查询用户
4. 调用 `saveAuth` 将用户信息存入 Redis（有效期 86400 秒 = 24 小时）
5. 返回新的应用 Token 和用户信息

> **注意**: SSO API 本身不做 OAuth/SAML 协议处理，仅接收已验证的用户身份并创建应用会话。

### 3.3 认证中间件

**文件**: `src/lib/auth.ts`

```typescript
export async function checkAuth(request: Request) {
  const token = getBearerToken(request);
  const payload = parseSecureToken(token, secret());
  const shareToken = await parseShareToken(request);

  let user = null;
  const { userId, authKey } = payload || {};

  if (userId) {
    user = await getUser(userId);          // 方式1: 直接通过 userId 查询
  } else if (redis.enabled && authKey) {
    const key = await redis.client.get(authKey); // 方式2: 通过 Redis authKey 查询
    if (key?.userId) {
      user = await getUser(key.userId);
    }
  }

  if (!user?.id && !shareToken) {
    return null; // 认证失败
  }

  if (user) {
    user.isAdmin = user.role === ROLES.admin;
  }

  return { token, authKey, shareToken, user };
}
```

**认证方式优先级**:
1. **JWT 直接认证**: Token 中包含 `userId`，直接查数据库
2. **Redis Session 认证**: Token 中包含 `authKey`，从 Redis 查用户
3. **Share Token 认证**: 共享链接的临时访问令牌

---

## 4. 本地用户匹配规则（开源已实现）

### 4.1 用户匹配键

**核心匹配字段**: `username`

**文件**: `src/queries/prisma/user.ts`

```typescript
export async function getUserByUsername(username: string, options: GetUserOptions = {}) {
  return findUser({ where: { username } }, options);
}
```

**数据模型**: `prisma/schema.prisma`

```prisma
model User {
  id          String    @id() @map("user_id") @db.Uuid
  username    String    @unique @db.VarChar(255)  // 唯一索引，SSO 匹配的唯一句柄
  password    String    @db.VarChar(60)
  role        String    @map("role") @db.VarChar(50)
  logoUrl     String?   @map("logo_url") @db.VarChar(2183)
  displayName String?   @map("display_name") @db.VarChar(255)
  createdAt   DateTime? @default(now()) @map("created_at") @db.Timestamptz(6)
  updatedAt   DateTime? @updatedAt @map("updated_at") @db.Timestamptz(6)
  deletedAt   DateTime? @map("deleted_at") @db.Timestamptz(6)
  // ...
}
```

### 4.2 匹配规则

| 匹配方式 | 说明 | 状态 |
|---------|------|------|
| `username` 精确匹配 | SSO 返回的用户名（通常是邮箱）与 User 表的 `username` 字段完全匹配 | ✅ 已实现（唯一匹配方式） |
| 邮箱正则匹配 | 通过邮箱格式或域名模糊匹配 | ❌ 未实现 |
| 外部 ID 匹配 | 通过 IdP 提供的唯一外部 ID 匹配 | ❌ 未实现（无 external_id 字段） |
| 多字段匹配 | 结合姓名、邮箱等多字段匹配 | ❌ 未实现 |

### 4.3 用户创建（手动方式）

**文件**: `src/app/api/users/route.ts`

```typescript
export async function POST(request: Request) {
  // ... 权限检查 (仅 admin 可创建)

  const existingUser = await getUserByUsername(username, { showDeleted: true });

  if (existingUser) {
    return badRequest({ message: 'User already exists' });
  }

  const user = await createUser({
    id: id || uuid(),
    username,
    password: hashPassword(password),
    role: role ?? ROLES.user,  // 默认角色: user
  });

  return json(user);
}
```

**创建规则**:
- 只有 `admin` 角色可以创建用户
- `username` 全局唯一（含已删除用户也占用用户名）
- 默认角色为 `user`
- 必须设置密码字段

> **⚠️ JIT 自动创建用户未实现**: 开源版本中不存在任何自动创建用户的逻辑。SSO 场景下，如果用户不存在，认证将失败。需自行扩展 JIT 逻辑或使用 Cloud 服务。

---

## 5. 邮件域到 Team 映射

### 5.1 团队数据模型（开源已实现）

**文件**: `prisma/schema.prisma`

```prisma
model Team {
  id         String    @id() @map("team_id") @db.Uuid
  name       String    @db.VarChar(50)
  accessCode String?   @unique @map("access_code") @db.VarChar(50)
  logoUrl    String?   @map("logo_url") @db.VarChar(2183)
  createdAt  DateTime? @default(now()) @map("created_at")
  updatedAt  DateTime? @updatedAt @map("updated_at")
  deletedAt  DateTime? @map("deleted_at")
  websites   Website[]
  members    TeamUser[]
  // ...
}

model TeamUser {
  id        String    @id() @map("team_user_id") @db.Uuid
  teamId    String    @map("team_id") @db.Uuid
  userId    String    @map("user_id") @db.Uuid
  role      String    @db.VarChar(50)     // 团队内角色
  createdAt DateTime? @default(now()) @map("created_at")
  updatedAt DateTime? @updatedAt @map("updated_at")
  team Team @relation(fields: [teamId], references: [id])
  user User @relation(fields: [userId], references: [id])
}
```

### 5.2 团队加入方式（开源已实现）

**方式 1: 访问码加入**

**文件**: `src/app/api/teams/join/route.ts`

```typescript
export async function POST(request: Request) {
  const { accessCode } = body;

  const team = await findTeam({ where: { accessCode } });

  if (!team) {
    return notFound({ message: 'Team not found.' });
  }

  const teamUser = await getTeamUser(team.id, auth.user.id);

  if (teamUser) {
    return badRequest({ message: 'User is already a team member.' });
  }

  const user = await createTeamUser(auth.user.id, team.id, ROLES.teamMember);

  return json(user);
}
```

**方式 2: 管理员添加成员**

**文件**: `src/app/api/teams/[teamId]/users/route.ts`

```typescript
export async function POST(request: Request, { params }) {
  // ... 权限检查 (需 canUpdateTeam)

  const { userId, role } = body;

  const teamUser = await getTeamUser(teamId, userId);

  if (teamUser) {
    return badRequest({ message: 'User is already a member of the Team.' });
  }

  const users = await createTeamUser(userId, teamId, role);

  return json(users);
}
```

### 5.3 邮件域自动映射（❌ 未实现）

开源版本中**不存在**任何基于邮箱域名自动加入团队的逻辑。

| 相关功能 | 实现状态 | 说明 |
|---------|---------|------|
| 企业域名配置 | ❌ 未实现 | 无相关配置项，无对应数据库字段 |
| 邮箱域 → Team 绑定 | ❌ 未实现 | 无映射关系表，无匹配逻辑 |
| SSO 登录自动加入 Team | ❌ 未实现 | 登录流程中无自动加入团队的代码 |
| 多 Team 映射 | ❌ 未实现 | 上述基础能力均未实现 |

> **扩展建议**: 如需实现邮件域自动映射，可在 SSO 适配层或自定义扩展中：
> 1. 新增 `team_domain` 配置表存储域名与 Team 的映射关系
> 2. 在 SSO 登录流程中，根据用户邮箱后缀查询匹配的 Team
> 3. 调用 `createTeamUser` 自动建立成员关系

---

## 6. JIT (Just-In-Time) 配置

### 6.1 JIT 状态

| JIT 功能 | 实现状态 | 说明 |
|---------|---------|------|
| JIT 自动创建用户 | ❌ 未实现 | 开源版本无相关代码 |
| 邮箱域白名单 | ❌ 未实现 | 无域名白名单校验逻辑 |
| 默认角色配置 | ❌ 未实现 | 无 JIT 默认角色配置项 |
| 自动加入默认 Team | ❌ 未实现 | 无相关逻辑 |

### 6.2 现有创建流程（手动）

开源版本中，用户只能通过以下两种方式创建：

1. **管理员手动创建**: 通过 `/api/users` POST 接口，需 `admin` 权限
2. **用户注册**: （无内置注册功能，需自行扩展）

### 6.3 支撑代码（可作为 JIT 扩展基础）

**用户创建函数**: `src/queries/prisma/user.ts`

```typescript
export async function createUser(data: {
  id: string;
  username: string;
  password: string;
  role: Role;
}) {
  return prisma.client.user.create({
    data,
    select: { id: true, username: true, role: true },
  });
}
```

**团队成员创建函数**: `src/queries/prisma/teamUser.ts`

```typescript
export async function createTeamUser(userId: string, teamId: string, role: string) {
  return prisma.client.teamUser.create({
    data: {
      id: uuid(),
      userId,
      teamId,
      role,
    },
  });
}
```

> **扩展建议**: 如需实现 JIT，可在 SSO 适配层中：
> 1. SSO 回调后检查用户是否存在（`getUserByUsername`）
> 2. 不存在则调用 `createUser` 创建（自动生成随机密码）
> 3. 根据邮件域映射规则调用 `createTeamUser` 加入团队

---

## 7. 角色映射规则

### 7.1 系统级角色（开源已实现）

**定义**: `src/lib/constants.ts`

| 角色 | 常量值 | 说明 |
|------|--------|------|
| admin | `admin` | 系统管理员，拥有所有权限 |
| user | `user` | 普通注册用户，可创建网站和团队 |
| view-only | `view-only` | 只读用户（系统级，预留） |

### 7.2 团队级角色（开源已实现）

**定义**: `src/lib/constants.ts`

| 角色 | 常量值 | 说明 |
|------|--------|------|
| team-owner | `teamOwner` | 团队所有者，拥有团队的所有管理权限 |
| team-manager | `teamManager` | 团队管理员，可管理成员和网站 |
| team-member | `teamMember` | 团队成员，可创建和管理网站 |
| team-view-only | `teamViewOnly` | 团队只读成员，只能查看 |

### 7.3 角色权限映射（开源已实现）

**定义**: `src/lib/constants.ts`

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

开源版本中**不存在**任何将 IdP（身份提供者）的用户组、角色声明映射到 Umami 角色的逻辑。

| 映射功能 | 实现状态 | 说明 |
|---------|---------|------|
| IdP 组 → 系统角色映射 | ❌ 未实现 | 无相关配置和代码 |
| IdP 组 → 团队角色映射 | ❌ 未实现 | 无相关配置和代码 |
| 角色断言解析 | ❌ 未实现 | 无 SAML/OAuth 断言解析代码 |
| 默认角色分配 | ❌ 未实现 | SSO 创建用户时无默认角色逻辑 |

> **扩展建议**: 如需实现角色映射，可在 SSO 适配层中：
> 1. 从 IdP 返回的断言/令牌中提取角色或组信息
> 2. 根据预配置的映射规则转换为 Umami 的系统角色或团队角色
> 3. 创建用户时设置 `role` 字段，加入团队时设置 TeamUser 的 `role` 字段

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

## 8. 账号归属决策树（开源版本实际行为）

```
SSO 回调请求 (已携带认证信息)
            │
            ▼
    ┌──────────────────┐
    │  checkAuth 验证   │
    │  (解析 JWT Token)│
    └────────┬─────────┘
             │
        ┌────┴────┐
        │         │
     有效       无效
        │         │
        ▼         ▼
    继续执行    认证失败
        │
        ▼
    ┌──────────────────┐
    │  getUser 查询    │
    │  (通过 userId)   │
    └────────┬─────────┘
             │
        ┌────┴────┐
        │         │
      存在      不存在
        │         │
        ▼         ▼
    返回用户    认证失败
    信息        （无 JIT）
        │
        ▼
    ┌──────────────────┐
    │ saveAuth 存 Redis│
    │  创建应用 Session│
    └────────┬─────────┘
             │
             ▼
        返回 Token
```

> **关键结论**: 开源版本的 SSO 是"被动验证"模式 —— 只验证已有用户，不自动创建新用户，不自动分配团队。

---

## 9. 关键代码文件索引

| 模块 | 文件路径 | 说明 |
|------|---------|------|
| SSO 页面 | `src/app/sso/SSOPage.tsx` | SSO 回调页面，处理 Token 存储和跳转 |
| SSO API | `src/app/api/auth/sso/route.ts` | SSO 认证 API，创建应用 Session |
| 认证核心 | `src/lib/auth.ts` | checkAuth, saveAuth, hasPermission |
| JWT 处理 | `src/lib/jwt.ts` | Token 加密/解密/验证 |
| 用户查询 | `src/queries/prisma/user.ts` | getUser, getUserByUsername, createUser |
| 团队查询 | `src/queries/prisma/team.ts` | getTeam, getUserTeams, createTeam |
| 团队成员 | `src/queries/prisma/teamUser.ts` | getTeamUser, createTeamUser |
| 角色常量 | `src/lib/constants.ts` | ROLES, PERMISSIONS, ROLE_PERMISSIONS |
| 团队权限 | `src/permissions/team.ts` | 团队相关权限校验函数 |
| 用户权限 | `src/permissions/user.ts` | 用户相关权限校验函数 |
| 数据模型 | `prisma/schema.prisma` | User, Team, TeamUser 模型定义 |
| 请求解析 | `src/lib/request.ts` | parseRequest (调用 checkAuth) |
| Cloud 数据加载 | `src/lib/load.ts` | fetchAccount, fetchTeam (Redis 缓存) |

---

## 10. 总结

### 10.1 开源版本已实现的能力

1. **用户身份模型**: User 表 + username 唯一索引，支持基于用户名的匹配
2. **团队成员模型**: Team + TeamUser 双层结构，支持多团队、多角色
3. **权限系统**: 系统级角色 + 团队级角色的双层权限体系
4. **SSO 接入点**: `/sso` 页面和 `/api/auth/sso` 接口，可对接外部 SSO 适配层
5. **Session 管理**: Redis + JWT 双模式会话管理

### 10.2 需自行扩展或依赖 Cloud 服务的能力

| 功能 | 扩展点 | 建议实现位置 |
|------|--------|-------------|
| OAuth 协议 | 无现成实现，需从零接入 | SSO 适配层（独立服务或 Next.js API Route） |
| SAML 协议 | 无现成实现，需从零接入 | SSO 适配层（独立服务） |
| JIT 用户创建 | 可复用 `createUser` 函数 | SSO 回调处理逻辑中 |
| 邮件域 → Team 映射 | 需新增映射表和匹配逻辑 | SSO 回调处理逻辑中 |
| IdP 角色映射 | 需新增映射配置 | SSO 回调处理逻辑中 |
| 自动团队加入 | 可复用 `createTeamUser` 函数 | SSO 回调处理逻辑中 |

### 10.3 核心账号归属规则（开源版本）

1. **匹配键唯一**: 仅通过 `username` 字段精确匹配用户
2. **无自动创建**: 用户不存在时认证失败，不会自动创建
3. **团队手动加入**: 需通过访问码或管理员添加，无自动分配
4. **角色双层结构**: 系统级角色决定全局权限，团队级角色决定团队内权限
5. **管理员优先**: 系统管理员 (admin) 绕过所有团队权限检查
