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

## 2. Token 流向全景（按代码拆开的完整链路）

### 2.1 三条关键通道

```
通道 A：SSO 回调令牌  →  localStorage  →  所有请求的 Bearer 头  →  checkAuth
通道 B：verify 返回令牌(可选) →  localStorage 覆盖 →  Bearer 头 →  checkAuth
通道 C：POST /api/auth/sso（服务器内部） → saveAuth 写 Redis → 返回新 authKey 令牌
```

前端实际只走通道 A；通道 B 是本地登录路径；通道 C 是服务端接口，前端不直接调用。

---

## 3. 通道 A：SSO 回调 → 前端直接用回调 Token

### 3.1 第 1 步：外部适配层签发回调 Token 并重定向

外部适配层完成 OAuth/SAML 协议、JIT 创建、域映射、角色映射后，签发 JWT 并重定向到 Umami：

```
GET /sso?url=<target>&token=<jwt>
```

- `url`: 登录成功后前端跳转的目标页面（如 `/`）
- `token`: 外部适配层签发的 JWT，payload 必含 `userId`（因为应用层只认这个）

### 3.2 第 2 步：SSOPage 接收 Token，直接存入 localStorage

**文件**: `src/app/sso/SSOPage.tsx`

```typescript
export function SSOPage() {
  const router = useRouter();
  const search = useSearchParams();
  const url = search.get('url');     // 跳转目标
  const token = search.get('token'); // 回调 Token

  useEffect(() => {
    if (url && token) {
      setClientAuthToken(token);  // 直接存入 localStorage
      router.push(url);           // 跳转到目标页面
    }
  }, [router, url, token]);

  return <Loading placement="absolute" />;
}
```

**关键事实**:
- SSO 页面**不调用**任何 API（不调 `/api/auth/sso`、不调 `/api/auth/verify`）
- 仅做两件事：存 token 到 localStorage + 前端路由跳转
- 回调 Token 被**原封不动**地存入客户端

### 3.3 第 3 步：前端请求自动将 Token 注入 Bearer 头

**文件**: `src/components/hooks/useApi.ts`

```typescript
export function useApi() {
  // ...

  const defaultHeaders = {
    authorization: `Bearer ${getClientAuthToken()}`,  // 每次请求都从 localStorage 读 token
    ...shareHeaders,
  };

  // get/post/put/del 都自动附带 defaultHeaders
}
```

**文件**: `src/lib/client.ts`

```typescript
export function getClientAuthToken() {
  return getItem(AUTH_TOKEN);  // 读 localStorage["umami.auth"]
}
```

**关键事实**:
- 所有前端 API 请求（通过 `useApi` 钩子发出）自动携带 `Authorization: Bearer <token>`
- Token 直接从 localStorage 读取，不经过中间服务
- 前端没有任何令牌刷新/续期逻辑

### 3.4 第 4 步：应用加载时调用 verify，验证 Token 并拉取用户

**文件**: `src/components/hooks/queries/useLoginQuery.ts`

```typescript
export function useLoginQuery() {
  const { post, useQuery } = useApi();
  const user = useApp(selector);

  const query = useQuery({
    queryKey: ['login'],
    queryFn: async () => {
      const data = await post('/auth/verify');  // POST /api/auth/verify
      setUser(data);                            // 存入全局 store
      return data;
    },
    enabled: !user,  // 用户未加载时触发
  });

  return { user, setUser, ...query };
}
```

`useLoginQuery` 在应用主入口被调用：

**文件**: `src/app/(main)/App.tsx`

```typescript
export function App({ children }) {
  const { user, isLoading, error } = useLoginQuery();  // 应用加载即触发
  // ...
  if (error) {
    window.location.href = config.cloudMode
      ? `${process.env.cloudUrl}/login`  // Cloud 模式：跳回 Cloud 登录页
      : `${process.env.basePath || ''}/login`;  // 自托管：跳本地登录页
    return null;
  }
}
```

**文件**: `src/app/api/auth/verify/route.ts`

```typescript
export async function POST(request: Request) {
  const { auth, error } = await parseRequest(request);  // 内部调用 checkAuth

  if (error) {
    return error();  // token 无效 → 401，前端重定向到登录页
  }

  const user = { ...auth.user };
  const teams = await getAllUserTeams(user.id);

  // Cloud 模式下附加订阅信息 ...

  return json({ ...user, teams });  // 返回用户信息 + 所属团队
}
```

### 3.5 第 5 步：checkAuth 根据 payload 还原用户

**文件**: `src/lib/auth.ts`

```typescript
export async function checkAuth(request: Request) {
  const token = getBearerToken(request);        // 取 Bearer token
  const payload = parseSecureToken(token, secret()); // AES-256-GCM 解密 → JWT 验签 → payload
  const shareToken = await parseShareToken(request);

  let user = null;
  const { userId, authKey } = payload || {};

  if (userId) {                                  // 路径 A: payload 直接含 userId
    user = await getUser(userId);                //    → 按 UUID 查库
  } else if (redis.enabled && authKey) {         // 路径 B: payload 含 authKey
    const key = await redis.client.get(authKey); //    → 先从 Redis 取 { userId }
    if (key?.userId) {
      user = await getUser(key.userId);          //    → 再按 UUID 查库
    }
  }

  if (!user?.id && !shareToken) {
    return null;  // 认证失败
  }

  if (user) {
    user.isAdmin = user.role === ROLES.admin;
  }

  return { token, authKey, shareToken, user };
}
```

### 3.6 通道 A 完整时序

```
外部适配层                              前端浏览器                           应用层 API
      │                                    │                                   │
      │  1. OAuth/SAML 完成                 │                                   │
      │  2. 用户匹配/JIT/映射                │                                   │
      │  3. 签发 JWT { userId }              │                                   │
      │                                     │                                   │
      │  GET /sso?url=/&token=<jwt>         │                                   │
      └────────────────────────────────────>│                                   │
                                           │                                   │
                                           │  4. SSOPage:                      │
                                           │     setClientAuthToken(token)    │
                                           │     router.push('/')              │
                                           │                                   │
                                           │  5. App 组件加载                   │
                                           │     useLoginQuery() 触发          │
                                           │                                   │
                                           │  POST /api/auth/verify            │
                                           │  Authorization: Bearer <jwt>      │
                                           │──────────────────────────────────>│
                                           │                                   │  6. checkAuth:
                                           │                                   │     decrypt → verify → payload
                                           │                                   │     userId → getUser()
                                           │                                   │
                                           │     { user, teams }                │
                                           │<──────────────────────────────────┘
                                           │                                   │
                                           │  7. setUser(data) → 登录成功       │
                                           │                                   │
                                           │  8. 所有后续请求自动携带 Bearer     │
                                           │  Authorization: Bearer <jwt>      │
                                           │──────────────────────────────────>│
```

---

## 4. 通道 B：本地登录路径（对比用）

本地登录走 `POST /api/auth/login`，调用 `getUserByUsername`，返回新签发的 Token：

```typescript
// src/app/api/auth/login/route.ts
export async function POST(request: Request) {
  const { username, password } = body;

  const user = await getUserByUsername(username, { includePassword: true });

  if (!user || !checkPassword(password, user.password)) {
    return unauthorized();
  }

  let token: string;
  if (redis.enabled) {
    token = await saveAuth({ userId: id, role });   // 有 Redis → authKey 令牌
  } else {
    token = createSecureToken({ userId, role }, secret());  // 无 Redis → userId 令牌
  }

  const teams = await getAllUserTeams(id);

  return json({ token, user: { id, username, role, createdAt, isAdmin, teams } });
}
```

前端 LoginForm 拿到 `{ token, user }` 后，同样走 `setClientAuthToken(token)` 存入 localStorage。

---

## 5. 通道 C：`/api/auth/sso` 服务端内部接口

### 5.1 前端不调用此接口

通过代码检索确认：开源前端代码中**没有任何地方**调用 `/api/auth/sso`。grep 搜索 `auth/sso` 在 `src/` 下 0 个前端调用。

### 5.2 接口作用

**文件**: `src/app/api/auth/sso/route.ts`

```typescript
export async function POST(request: Request) {
  const { auth, error } = await parseRequest(request);  // 要求请求本身已带有效 Token

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

**接口语义**：

- 输入：已验证的 JWT（含 `userId` 或 `authKey`）
- 前置条件：Redis 必须启用
- 动作：将 `userId` 写入 Redis（key = `auth:<random>`，TTL = 86400 秒）
- 输出：新的 JWT（payload 为 `{ authKey }`）+ 用户信息

### 5.3 与回调 Token 的关系

```
外部适配层签发的回调 Token       → payload: { userId }    →  直接用（通道 A）
                                       │
                                       │ 若需要转换为 Redis 会话令牌
                                       ▼
POST /api/auth/sso (通道 C)     → saveAuth({ userId }) → 返回 { authKey }
```

**使用场景（推断）**：
- 服务端到服务端的场景（Cloud 服务代用户调用 Umami）
- 需要把短期/一次性回调 Token 换成长期 Redis 会话 Token
- 前端 SSO 路径中跳过此步骤，回调 Token 直接被用作 Bearer 凭证

### 5.4 两种 Token 的服务端兼容

应用层 `checkAuth` 对两种 Token 一视同仁：

| Token 来源 | payload 结构 | checkAuth 还原路径 | Redis 依赖 |
|-----------|-------------|-------------------|-----------|
| 外部适配层直接签发 | `{ userId, role? }` | 路径 A：直接 `getUser(userId)` | 不依赖 |
| `/api/auth/sso` 返回 | `{ authKey }` | 路径 B：`Redis.get(authKey)` → `getUser(key.userId)` | 依赖 Redis |
| `/api/auth/login` 返回（有 Redis） | `{ authKey }` | 路径 B | 依赖 Redis |
| `/api/auth/login` 返回（无 Redis） | `{ userId, role }` | 路径 A | 不依赖 |

---

## 6. 应用层令牌校验机制（开源已实现）

### 6.1 两种令牌 payload（已在 checkAuth 中实现）

**令牌类型 A — 内嵌 userId（无 Redis 时，或外部适配层直接签发）**

```typescript
// payload 结构：
{ userId: "01234567-89ab-cdef-...", role: "user" }
```

**令牌类型 B — 内嵌 authKey（有 Redis 时，经 saveAuth 转换）**

```typescript
// saveAuth 实现：
const authKey = `auth:${createAuthKey()}`;
await redis.client.set(authKey, { userId, role });
if (expire) {
  await redis.client.expire(authKey, expire);  // SSO 接口默认 86400s
}
return createSecureToken({ authKey }, secret());

// payload 结构：
{ authKey: "auth:a1b2c3d4e5f6..." }
```

### 6.2 两条路径的完整对比

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
Redis 模式: saveAuth({ userId, role })  → 返回含 authKey 的 JWT
无 Redis:   createSecureToken({ userId, role }, secret())  → 返回含 userId 的 JWT
```

> 注意：本地登录路径本身就在做"用户名 → userId"的适配，相当于一个最小化的内置适配层。`getUserByUsername` 只在这一处被认证流程调用。

**SSO 路径**（通道 A + 通道 C 的关系）:

```
通道 A（前端默认走）:
外部适配层签发 { userId } JWT → 存入 localStorage → 所有请求 Bearer → checkAuth 路径 A

通道 C（可选的服务端转换）:
外部适配层 JWT → POST /api/auth/sso → saveAuth → Redis 存入 → 返回 { authKey } JWT
                                                                  │
                                                                  ▼
                                                         存入 localStorage → Bearer
                                                                  │
                                                                  ▼
                                                         checkAuth 路径 B
```

> 注意：`/api/auth/sso` **不再做任何用户名匹配**。它只做一件事：把已验证的 userId 存入 Redis，签发新的应用 Session 令牌。

### 6.3 SSO API 的前置条件

`POST /api/auth/sso` 要求请求本身已携带有效令牌（即必须先通过 `checkAuth`）：

- 如果传入的 `userId` 对应不上 User 记录 → `checkAuth` 返回 `null` → 请求被拒为 401
- 不存在 JIT 自动创建用户的逻辑

---

## 7. 外部适配层职责（开源未实现）

### 7.1 外部适配层必须完成的工作

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

### 7.2 用户名/邮箱匹配：只发生在外部适配层

开源代码中，`getUserByUsername` 在认证流程中的调用点**仅有一处**：

```typescript
// src/app/api/auth/login/route.ts#L26
const user = await getUserByUsername(username, { includePassword: true });
```

这是本地登录的入口。SSO 路径（`/api/auth/sso` 和所有通过 Bearer 的请求）**不调用** `getUserByUsername`。

换言之：
- **外部适配层**负责"用户名/邮箱 → userId"的解析
- **应用层**只负责"userId/authKey → 用户对象"的还原
- 两层之间的**契约**就是 JWT payload 中的 `userId` 字段

### 7.3 令牌签发：两层之间的唯一握手点

外部适配层签发令牌的方式有两种：

**方式 A：直接签发内嵌 userId 的 JWT（SSO 默认走这个）**

```typescript
import { createSecureToken } from '@/lib/jwt';
import { secret } from '@/lib/crypto';

const token = createSecureToken({ userId: user.id, role: user.role }, secret());
```

此令牌可被应用层 `checkAuth` 的路径 A 直接解析，无需 Redis。前端拿到后直接存入 localStorage，作为 Bearer 凭证使用。

**方式 B：通过 `/api/auth/sso` 转换为 Redis authKey JWT**

```typescript
// 先用方式 A 签发的 Token 调 SSO API
// POST /api/auth/sso  Authorization: Bearer <方式A的token>
// 返回 { user, token } → token 是方式 B 的 authKey JWT
// 前端把这个新 token 存入 localStorage 替换旧的
```

此令牌由 `saveAuth` 将 `{ userId, role }` 写入 Redis（key: `auth:<random>`，TTL 86400s），返回的 JWT payload 只含 `authKey`。

> 两种方式的区别仅在于令牌体积、是否依赖 Redis、以及 TTL 可控性，对应用层 `checkAuth` 而言都是透明的。

---

## 8. 本地用户匹配（外部适配层职责）

### 8.1 用户模型

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

### 8.2 匹配规则

| 匹配方式 | 职责层 | 状态 | 说明 |
|---------|--------|------|------|
| `username` 精确匹配 | 外部适配层 | ✅ 可用 | `getUserByUsername` — 唯一的匹配函数 |
| 邮箱域名匹配 | 外部适配层 | ❌ 需自建 | 无 `email` 字段，无域名解析逻辑 |
| 外部 ID 匹配 | 外部适配层 | ❌ 需自建 | 无 `external_id` 字段 |
| userId UUID 查询 | 应用层 | ✅ 已实现 | `getUser(userId)` — 应用层唯一使用的查询 |

### 8.3 用户创建（手动方式）

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

## 9. 邮件域到 Team 映射（❌ 未实现）

### 9.1 团队加入方式（开源已实现）

| 方式 | 文件 | 说明 |
|------|------|------|
| 访问码加入 | `src/app/api/teams/join/route.ts` | 用户提供 `accessCode`，默认角色 `team-member` |
| 管理员添加 | `src/app/api/teams/[teamId]/users/route.ts` | 需 `canUpdateTeam` 权限，可指定角色 |

### 9.2 邮件域自动映射（❌ 未实现）

开源版本中**不存在**基于邮箱域名自动加入团队的逻辑。

| 相关功能 | 实现状态 | 说明 |
|---------|---------|------|
| 企业域名配置 | ❌ 未实现 | 无相关配置项和数据库字段 |
| 邮箱域 → Team 绑定 | ❌ 未实现 | 无映射关系表 |
| SSO 登录自动加入 Team | ❌ 未实现 | 登录流程中无自动加入代码 |

> **扩展建议**: 在外部适配层中，SSO 认证完成后、签发令牌前，根据用户邮箱后缀查询匹配的 Team，调用 `createTeamUser` 自动建立成员关系。

---

## 10. JIT 配置（❌ 未实现）

| JIT 功能 | 实现状态 | 说明 |
|---------|---------|------|
| JIT 自动创建用户 | ❌ 未实现 | 开源版本无相关代码 |
| 邮箱域白名单 | ❌ 未实现 | 无域名白名单校验逻辑 |
| 默认角色配置 | ❌ 未实现 | 无 JIT 默认角色配置项 |
| 自动加入默认 Team | ❌ 未实现 | 无相关逻辑 |

> **扩展建议**: 在外部适配层中，`getUserByUsername` 返回空时，调用 `createUser` 创建用户（自动生成随机密码），再根据邮件域映射调用 `createTeamUser` 加入团队，最后签发含新 `userId` 的令牌。

---

## 11. 角色映射规则

### 11.1 系统级角色（开源已实现）

**定义**: `src/lib/constants.ts`

| 角色 | 常量值 | 说明 |
|------|--------|------|
| admin | `admin` | 系统管理员，拥有所有权限 |
| user | `user` | 普通用户，可创建网站和团队 |
| view-only | `view-only` | 只读用户（系统级，预留） |

### 11.2 团队级角色（开源已实现）

| 角色 | 常量值 | 说明 |
|------|--------|------|
| team-owner | `teamOwner` | 团队所有者 |
| team-manager | `teamManager` | 团队管理员 |
| team-member | `teamMember` | 团队成员 |
| team-view-only | `teamViewOnly` | 团队只读成员 |

### 11.3 角色权限映射（开源已实现）

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

### 11.4 IdP 角色/组映射（❌ 未实现）

开源版本中**不存在**任何将 IdP 用户组、角色声明映射到 Umami 角色的逻辑。

| 映射功能 | 实现状态 | 说明 |
|---------|---------|------|
| IdP 组 → 系统角色映射 | ❌ 未实现 | 无相关配置和代码 |
| IdP 组 → 团队角色映射 | ❌ 未实现 | 无相关配置和代码 |
| 角色断言解析 | ❌ 未实现 | 无 SAML/OAuth 断言解析代码 |

> **扩展建议**: 在外部适配层中，从 IdP 断言/令牌中提取角色或组信息，根据映射规则调用 `updateUser` 设置系统角色，调用 `createTeamUser` 设置团队角色。

### 11.5 权限检查函数（开源已实现）

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

## 12. 完整数据流：浏览器 + 适配层 + 应用层协作

```
浏览器/前端                 外部适配层                     应用层 API
    │                           │                             │
    │ 1. 点击 SSO 登录          │                             │
    │──────────────────────────>│                             │
    │                           │                             │
    │                           │ 2. IdP 认证/协议交互          │
    │                           │ 3. 解析断言 username/email   │
    │                           │ 4. getUserByUsername()       │
    │                           │ 5. JIT 创建/域映射/角色映射   │
    │                           │ 6. 签发 JWT { userId }      │
    │                           │                             │
    │ 重定向 /sso?url=/         │                             │
    │          &token=<jwt>     │                             │
    │<──────────────────────────│                             │
    │                           │                             │
    │ 7. SSOPage:                │                             │
    │    setClientAuthToken()   │                             │
    │    router.push('/')       │                             │
    │                           │                             │
    │ 8. App 加载                │                             │
    │    useLoginQuery()        │                             │
    │                           │                             │
    │ POST /api/auth/verify     │                             │
    │ Authorization: Bearer     │                             │
    │   <回调token>             │                             │
    │────────────────────────────────────────────────────────>│
    │                           │                             │
    │                           │              9. checkAuth:   │
    │                           │                 decrypt +   │
    │                           │                 verify →    │
    │                           │                 payload.userId
    │                           │                 getUser()    │
    │                           │                             │
    │     { user, teams }       │                             │
    │<────────────────────────────────────────────────────────│
    │                           │                             │
    │ 10. setUser() → 登录成功  │                             │
    │                           │                             │
    │     ───────────────────────────────────────────        │
    │     │ 所有后续 API 请求自动带 Bearer 头      │        │
    │     └──────────────────────────────────────────        │
    │                           │                             │
    │ POST /api/auth/sso       │ (可选，服务端间调用)         │
    │ Bearer:<回调token>       │                             │
    │────────────────────────────────────────────────────────>│
    │                           │                             │
    │                           │              checkAuth OK   │
    │                           │              saveAuth → Redis│
    │                           │              签发新 authKey  │
    │     { user, token }      │                             │
    │     (新 token 是 authKey) │                             │
    │<────────────────────────────────────────────────────────│
    │                           │                             │
    │ setClientAuthToken(新)   │ (前端如果选择替换)           │
    │                           │                             │
```

---

## 13. 账号归属决策树（开源版本实际行为）

```
前端 SSO 回调：/sso?token=<jwt>&url=<target>
            │
            ▼
 setClientAuthToken(token) → localStorage
            │
            ▼
 跳转到 <target>，App 加载
            │
            ▼
 useLoginQuery 触发
            │
            ▼
 POST /api/auth/verify  Authorization: Bearer <jwt>
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

> **关键结论**: 前端 SSO 路径中，**回调 Token 直接被作为 Bearer 凭证使用**，不需要经过 `/api/auth/sso`。`/api/auth/sso` 是可选的服务器内部转换接口，用于把回调 Token 换成带 TTL 的 Redis 会话 Token。

---

## 14. 关键代码文件索引

| 模块 | 文件路径 | 职责层 | 说明 |
|------|---------|--------|------|
| SSO 回调页面 | `src/app/sso/SSOPage.tsx` | 前端 | 存 token + 跳转，不调 API |
| SSO API（内部） | `src/app/api/auth/sso/route.ts` | 应用层 | 验令牌 → saveAuth → 返回新 authKey 令牌 |
| Verify API | `src/app/api/auth/verify/route.ts` | 应用层 | 前端加载时验证 token + 拉取用户和 teams |
| 认证核心 | `src/lib/auth.ts` | 应用层 | checkAuth / saveAuth / hasPermission |
| JWT 处理 | `src/lib/jwt.ts` | 应用层 | Token 加密/解密/验签 |
| 加密工具 | `src/lib/crypto.ts` | 应用层 | AES-256-GCM + secret 派生 |
| 客户端 Token | `src/lib/client.ts` | 前端 | get/set/remove localStorage token |
| 请求钩子 | `src/components/hooks/useApi.ts` | 前端 | 所有请求自动注入 Bearer token |
| 登录查询 | `src/components/hooks/queries/useLoginQuery.ts` | 前端 | 应用加载时调 /auth/verify |
| 应用入口 | `src/app/(main)/App.tsx` | 前端 | useLoginQuery，验证失败跳登录页 |
| 登录表单 | `src/app/login/LoginForm.tsx` | 前端 | 本地登录，调 /auth/login |
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

## 15. 总结

### 15.1 Token 流向的本质

SSO 登录链路中 Token 的流动路径：

1. **外部适配层**签发含 `userId` 的 JWT → 重定向到 `/sso?token=<jwt>`
2. **前端 SSOPage**直接存入 localStorage → 自动作为所有请求的 Bearer 凭证
3. **应用层 verify/checkAuth**根据 payload 中的 `userId`（或 `authKey → Redis → userId`）还原用户对象
4. **可选的 `/api/auth/sso`**把回调 Token 换成 Redis 会话 Token（带 TTL）

前端**不主动调用** `/api/auth/sso`，回调 Token 被直接使用。

### 15.2 两层边界的本质

- **外部适配层**的工作是**身份解析**：把 IdP 返回的用户名/邮箱翻译成本地 `userId`（UUID），完成用户创建、团队映射、角色映射，然后签发含 `userId` 的令牌
- **应用层**的工作是**身份还原**：从令牌中取出 `userId` 或 `authKey`，还原用户对象，执行权限检查
- 两层之间的**唯一契约**是 JWT payload 中的 `userId` 字段

### 15.3 开源版本已实现的能力

1. **Token 承载与注入**: SSOPage 存 token + useApi 自动注入 Bearer 头
2. **Token 校验与用户还原**: `checkAuth` 按 userId/authKey 还原用户对象
3. **Session 管理**: `saveAuth` 写 Redis，`createSecureToken` 签 JWT
4. **身份验证**: `/api/auth/verify` 提供用户信息和团队列表
5. **权限系统**: 系统级 + 团队级双层角色体系
6. **SSO 接入点**: `/sso` 页面和 `/api/auth/sso` 接口
7. **可复用的数据操作**: `getUserByUsername`、`createUser`、`createTeamUser` 等函数

### 15.4 需自行扩展或依赖 Cloud 服务的能力

| 功能 | 实现位置 | 可复用的开源函数 |
|------|---------|-----------------|
| OAuth/SAML 协议处理 | 外部适配层 | 无 |
| 用户名/邮箱 → userId 解析 | 外部适配层 | `getUserByUsername` |
| JIT 自动创建用户 | 外部适配层 | `createUser` |
| 邮件域 → Team 映射 | 外部适配层 | `createTeamUser` |
| IdP 角色/组映射 | 外部适配层 | `updateUser`, `createTeamUser` |
| JWT 令牌签发 | 外部适配层 | `createSecureToken`, `secret` |

### 15.5 核心账号归属规则

1. **Token 直用**: SSO 回调 Token 直接存 localStorage，直接作为 Bearer 凭证
2. **两层分离**: 外部适配层用 `username`/`email` 定位账号，应用层只凭 `userId`/`authKey` 校验令牌
3. **契约是 userId**: JWT payload 中的 `userId` 是两层之间的唯一传递物
4. **无自动创建**: 用户不存在时应用层直接拒绝，JIT 需在外部适配层实现
5. **团队手动加入**: 需通过访问码或管理员添加，无自动分配
6. **角色双层结构**: 系统级角色决定全局权限，团队级角色决定团队内权限
7. **管理员优先**: 系统管理员 (admin) 绕过所有团队权限检查
8. **SSO API 可选**: `/api/auth/sso` 是服务端内部转换接口，前端 SSO 路径不调用它
