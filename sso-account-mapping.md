# Umami SSO 账号归属规则分析

## 概述

本文档基于 Umami v3.1.0 开源版本代码库，分析企业 SSO 登录链路的账号归属规则。
SSO 功能主要在 **Cloud 模式** 下运行，核心的 OAuth/SAML 认证逻辑由 Cloud 服务层处理，开源版本提供 SSO 入口、用户匹配和权限管理框架。

---

## 1. SSO 整体架构

### 1.1 架构分层

```
┌─────────────────────────────────────────────────┐
│           企业 IdP (OAuth/SAML)                │
└────────────────────┬────────────────────────────┘
                     │ SSO 认证
                     ▼
┌─────────────────────────────────────────────────┐
│           Cloud 服务层 (SSO 核心逻辑)           │
│  - OAuth/SAML 回调处理                          │
│  - JIT 用户预配置                                │
│  - 邮件域 → Team 映射                           │
│  - 角色映射                                      │
└────────────────────┬────────────────────────────┘
                     │ 传递认证后的用户信息
                     ▼
┌─────────────────────────────────────────────────┐
│           Umami 应用层 (开源版本)               │
│  - /api/auth/sso 接收认证 Token                 │
│  - 本地用户匹配 (username)                       │
│  - 权限验证 (Auth + TeamUser)                   │
│  - Session 管理 (Redis/JWT)                     │
└─────────────────────────────────────────────────┘
```

### 1.2 关键入口点

| 入口 | 文件路径 | 作用 |
|------|---------|------|
| SSO 页面 | [src/app/sso/page.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/app/sso/page.tsx) | 接收 SSO 回调后的 url 和 token，完成客户端登录跳转 |
| SSO API | [src/app/api/auth/sso/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/app/api/auth/sso/route.ts) | 验证 SSO 认证，生成应用 Session Token |
| 登录页 | [src/app/login/page.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/app/login/page.tsx) | Cloud 模式下禁用本地登录页 |

---

## 2. OAuth/SAML 回调流程

### 2.1 SSO 回调页面处理

**文件**: [src/app/sso/SSOPage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/app/sso/SSOPage.tsx)

```typescript
export function SSOPage() {
  const router = useRouter();
  const search = useSearchParams();
  const url = search.get('url');    // 登录后跳转的目标 URL
  const token = search.get('token'); // SSO 认证 Token

  useEffect(() => {
    if (url && token) {
      setClientAuthToken(token);  // 存储 Token 到客户端
      router.push(url);           // 跳转到目标页面
    }
  }, [router, url, token]);

  return <Loading placement="absolute" />;
}
```

**流程说明**:
1. 用户从企业 IdP 完成认证后，IdP 重定向回 Umami 的 `/sso` 页面
2. URL 携带两个参数：
   - `token`: SSO 认证凭证（由 Cloud 服务层签发）
   - `url`: 登录成功后跳转的目标页面
3. 前端将 Token 存入本地存储（key: `umami.auth`）
4. 前端跳转到目标页面，后续请求通过 `Authorization` Header 携带 Token

### 2.2 SSO API 验证

**文件**: [src/app/api/auth/sso/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/app/api/auth/sso/route.ts)

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

### 2.3 认证中间件

**文件**: [src/lib/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/lib/auth.ts)

```typescript
export async function checkAuth(request: Request) {
  const token = getBearerToken(request);
  const payload = parseSecureToken(token, secret());
  const shareToken = await parseShareToken(request);

  let user = null;
  const { userId, authKey } = payload || {};

  if (userId) {
    user = await getUser(userId);          // 直接通过 userId 查询
  } else if (redis.enabled && authKey) {
    const key = await redis.client.get(authKey); // 通过 Redis authKey 查询
    if (key?.userId) {
      user = await getUser(key.userId);
    }
  }

  if (!user?.id && !shareToken) {
    return null; // 认证失败
  }

  // ... 权限检查

  return { token, authKey, shareToken, user };
}
```

**认证方式优先级**:
1. **JWT 直接认证**: Token 中包含 `userId`，直接查数据库
2. **Redis Session 认证**: Token 中包含 `authKey`，从 Redis 查用户
3. **Share Token 认证**: 共享链接的临时访问令牌

---

## 3. 本地用户匹配规则

### 3.1 用户匹配键

**核心匹配字段**: `username`

**文件**: [src/queries/prisma/user.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/queries/prisma/user.ts)

```typescript
export async function getUserByUsername(username: string, options: GetUserOptions = {}) {
  return findUser({ where: { username } }, options);
}
```

**数据模型**: [prisma/schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/prisma/schema.prisma)

```prisma
model User {
  id          String    @id() @map("user_id") @db.Uuid
  username    String    @unique @db.VarChar(255)  // 唯一索引，用于 SSO 匹配
  password    String    @db.VarChar(60)
  role        String    @map("role") @db.VarChar(50)
  // ...
}
```

### 3.2 匹配规则

| 匹配方式 | 说明 | 优先级 |
|---------|------|--------|
| `username` 精确匹配 | SSO 返回的用户名（通常是邮箱）与 User 表的 `username` 字段完全匹配 | 唯一匹配方式 |

> **注意**: 开源版本中，SSO 用户匹配完全依赖 `username` 字段。Cloud 模式下的 SSO 可能会使用邮箱作为 username 进行匹配。

### 3.3 用户创建 (JIT 前置条件)

**文件**: [src/app/api/users/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/app/api/users/route.ts)

```typescript
export async function POST(request: Request) {
  // ... 权限检查

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
- `username` 全局唯一（含已删除用户）
- 默认角色为 `user`
- 必须设置密码（SSO 用户可能由 Cloud 层自动生成随机密码）

---

## 4. 邮件域到 Team 映射

### 4.1 团队模型

**文件**: [prisma/schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/prisma/schema.prisma)

```prisma
model Team {
  id         String    @id() @map("team_id") @db.Uuid
  name       String    @db.VarChar(50)
  accessCode String?   @unique @map("access_code") @db.VarChar(50)
  // ...
  members  TeamUser[]
}

model TeamUser {
  id        String    @id() @map("team_user_id") @db.Uuid
  teamId    String    @map("team_id") @db.Uuid
  userId    String    @map("user_id") @db.Uuid
  role      String    @db.VarChar(50)     // 团队内角色
  createdAt DateTime? @default(now()) @map("created_at")
  // ...
  team Team @relation(fields: [teamId], references: [id])
  user User @relation(fields: [userId], references: [id])
}
```

### 4.2 团队加入方式

**方式 1: 访问码加入**

**文件**: [src/app/api/teams/join/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/app/api/teams/join/route.ts)

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

**文件**: [src/app/api/teams/[teamId]/users/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/app/api/teams/%5BteamId%5D/users/route.ts)

```typescript
export async function POST(request: Request, { params }) {
  // ... 权限检查 (canUpdateTeam)

  const { userId, role } = body;

  const teamUser = await getTeamUser(teamId, userId);

  if (teamUser) {
    return badRequest({ message: 'User is already a member of the Team.' });
  }

  const users = await createTeamUser(userId, teamId, role);

  return json(users);
}
```

### 4.3 邮件域映射机制 (Cloud 模式)

开源版本中没有直接的邮件域到 Team 的自动映射代码。根据 Cloud 模式的架构推断：

| 机制 | 说明 | 位置 |
|------|------|------|
| 企业域名配置 | 在 Cloud 管理后台配置企业邮箱域名与 Team 的绑定关系 | Cloud 服务层 |
| 自动加入 | SSO 用户首次登录时，根据其邮箱后缀自动加入对应 Team | Cloud 服务层调用 `createTeamUser` |
| 多 Team 支持 | 一个邮箱域可映射到多个 Team，或一个 Team 可绑定多个邮箱域 | Cloud 服务层配置 |

---

## 5. JIT (Just-In-Time) 配置

### 5.1 JIT 用户创建

开源版本中，用户创建需要管理员手动操作。Cloud 模式下 JIT 功能的推断行为：

| JIT 配置项 | 说明 | 默认值 |
|-----------|------|--------|
| `JIT_ENABLED` | 是否启用自动创建用户 | false |
| `JIT_DEFAULT_ROLE` | 自动创建用户的默认角色 | `user` |
| `JIT_EMAIL_DOMAINS` | 允许自动创建用户的邮箱域白名单 | 空 (全部允许或按配置) |
| `JIT_TEAM_ID` | 用户自动加入的 Team ID | 空 (不自动加入) |

### 5.2 JIT 流程 (推断)

```
SSO 回调
   │
   ▼
检查用户是否存在 (getUserByUsername)
   │
   ├─ 存在 → 直接登录，更新用户信息
   │
   └─ 不存在 →
        │
        ├─ JIT 未启用 → 登录失败
        │
        └─ JIT 已启用 →
             │
             ├─ 检查邮箱域名白名单
             │
             ├─ 创建用户 (createUser)
             │   - username: SSO 返回的邮箱
             │   - password: 随机生成 (SSO 用户不使用本地密码)
             │   - role: JIT_DEFAULT_ROLE
             │
             ├─ 邮件域 → Team 映射
             │   - 查找匹配的 Team
             │   - 创建 TeamUser 关系
             │   - 角色: 默认 team-member 或按映射规则
             │
             └─ 返回登录成功
```

### 5.3 支撑代码

**用户创建函数**: [src/queries/prisma/user.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/queries/prisma/user.ts)

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

**团队成员创建函数**: [src/queries/prisma/teamUser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/queries/prisma/teamUser.ts)

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

---

## 6. 角色映射规则

### 6.1 系统级角色

**定义**: [src/lib/constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/lib/constants.ts#L164-L172)

| 角色 | 常量值 | 说明 |
|------|--------|------|
| admin | `admin` | 系统管理员，拥有所有权限 |
| user | `user` | 普通注册用户，可创建网站和团队 |
| view-only | `view-only` | 只读用户 (系统级，预留) |

### 6.2 团队级角色

**定义**: [src/lib/constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/lib/constants.ts#L168-L171)

| 角色 | 常量值 | 说明 |
|------|--------|------|
| team-owner | `teamOwner` | 团队所有者，拥有团队的所有管理权限 |
| team-manager | `teamManager` | 团队管理员，可管理成员和网站 |
| team-member | `teamMember` | 团队成员，可创建和管理网站 |
| team-view-only | `teamViewOnly` | 团队只读成员，只能查看 |

### 6.3 角色权限映射

**定义**: [src/lib/constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/lib/constants.ts#L186-L217)

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

### 6.4 SSO 角色映射 (推断)

Cloud 模式下，SSO 断言 (Assertion) 中的角色/组信息会映射到 Umami 角色：

| IdP 角色/组 | 映射到系统角色 | 映射到团队角色 |
|------------|---------------|---------------|
| `umami:admin` | `admin` | - |
| `umami:user` | `user` | - |
| `team:{teamId}:owner` | - | `team-owner` |
| `team:{teamId}:manager` | - | `team-manager` |
| `team:{teamId}:member` | - | `team-member` |
| `team:{teamId}:view-only` | - | `team-view-only` |
| 默认 (无匹配) | `user` (JIT 创建时) | `team-member` (自动加入时) |

### 6.5 权限检查函数

**文件**: [src/lib/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/lib/auth.ts)

```typescript
export async function hasPermission(role: string, permission: string | string[]) {
  return ensureArray(permission).some(e => ROLE_PERMISSIONS[role]?.includes(e));
}
```

**文件**: [src/permissions/team.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/permissions/team.ts)

```typescript
export async function canUpdateTeam({ user }: Auth, teamId: string) {
  if (!user) return false;
  if (user.isAdmin) return true; // 系统管理员绕过团队权限检查

  const teamUser = await getTeamUser(teamId, user.id);
  return teamUser && hasPermission(teamUser.role, PERMISSIONS.teamUpdate);
}
```

---

## 7. 账号归属决策树

```
SSO 认证请求
     │
     ▼
┌─────────────────────┐
│  解析 SSO Token     │
│  (Cloud 服务层)     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  提取用户标识        │
│  (username/email)   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  本地用户匹配        │
│  getUserByUsername  │
└──────────┬──────────┘
           │
     ┌─────┴─────┐
     │           │
    存在        不存在
     │           │
     ▼           ▼
  更新信息     JIT 启用?
     │        ┌──┴──┐
     │       是     否
     │        │     │
     │        ▼     ▼
     │    创建用户  拒绝
     │        │
     │        ▼
     │    邮件域映射?
     │     ┌──┴──┐
     │    有     无
     │     │     │
     │     ▼     │
     │   加入 Team│
     │     │     │
     └─────┘     │
           │     │
           ▼     ▼
         登录成功/失败
           │
           ▼
┌─────────────────────┐
│  创建 Session       │
│  saveAuth (Redis)   │
└──────────┬──────────┘
           │
           ▼
      返回 Token
```

---

## 8. 关键代码文件索引

| 模块 | 文件路径 | 说明 |
|------|---------|------|
| SSO 页面 | [src/app/sso/SSOPage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/app/sso/SSOPage.tsx) | SSO 回调页面，处理 Token 存储和跳转 |
| SSO API | [src/app/api/auth/sso/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/app/api/auth/sso/route.ts) | SSO 认证 API，创建应用 Session |
| 认证核心 | [src/lib/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/lib/auth.ts) | checkAuth, saveAuth, hasPermission |
| JWT 处理 | [src/lib/jwt.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/lib/jwt.ts) | Token 加密/解密/验证 |
| 用户查询 | [src/queries/prisma/user.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/queries/prisma/user.ts) | getUser, getUserByUsername, createUser |
| 团队查询 | [src/queries/prisma/team.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/queries/prisma/team.ts) | getTeam, getUserTeams, createTeam |
| 团队成员 | [src/queries/prisma/teamUser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/queries/prisma/teamUser.ts) | getTeamUser, createTeamUser |
| 角色常量 | [src/lib/constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/lib/constants.ts) | ROLES, PERMISSIONS, ROLE_PERMISSIONS |
| 权限检查 | [src/permissions/team.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/permissions/team.ts) | 团队相关权限函数 |
| 权限检查 | [src/permissions/user.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/permissions/user.ts) | 用户相关权限函数 |
| 数据模型 | [prisma/schema.prisma](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/prisma/schema.prisma) | User, Team, TeamUser 模型定义 |
| 请求解析 | [src/lib/request.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/lib/request.ts) | parseRequest (调用 checkAuth) |
| Cloud 数据加载 | [src/lib/load.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/52-umami/src/lib/load.ts) | fetchAccount, fetchTeam (Redis) |

---

## 9. 总结

### 9.1 账号归属核心规则

1. **用户匹配**: 仅通过 `username` 字段精确匹配，通常为邮箱地址
2. **Team 归属**: 通过 `TeamUser` 关联表管理，支持多团队、多角色
3. **角色体系**: 双层角色结构 — 系统级角色 (admin/user) + 团队级角色 (owner/manager/member/view-only)
4. **权限判定**: 系统管理员 > 团队角色权限 > 个人用户权限

### 9.2 Cloud 模式扩展点

开源版本提供了完整的框架，Cloud 模式在此基础上扩展：

- **SSO 认证层**: OAuth/SAML 协议处理、IdP 配置管理
- **JIT 用户创建**: 基于邮箱域白名单的自动用户创建
- **邮件域映射**: 企业邮箱后缀到 Team 的自动绑定
- **角色映射**: IdP 用户组/角色到 Umami 角色的映射规则
- **账号同步**: 定期同步 IdP 用户信息和组织架构

### 9.3 注意事项

- 开源版本**不包含**完整的 OAuth/SAML 协议实现
- 文档中关于 Cloud 模式的部分为基于代码架构的**合理推断**
- 实际生产环境中，SSO 的具体行为以 Cloud 服务的配置为准
