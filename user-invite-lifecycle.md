# 用户邀请生命周期与账号状态流转分析

基于 Umami 代码库的邀请机制与账号生命周期关联分析

---

## 一、核心数据模型

### 1.1 数据结构定义

#### User 模型（用户账号）
`prisma/schema.prisma:12-32`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String (UUID) | 用户唯一标识 |
| `username` | String | 用户名，唯一 |
| `password` | String | 密码哈希（bcrypt，60位） |
| `role` | String | 系统级角色 |
| `deletedAt` | DateTime? | 删除时间（NULL 表示激活状态） |
| `createdAt` | DateTime | 创建时间 |
| `updatedAt` | DateTime | 更新时间 |

#### Team 模型（团队）
`prisma/schema.prisma:197-214`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String (UUID) | 团队唯一标识 |
| `name` | String | 团队名称 |
| `accessCode` | String? | 访问码（用于加入团队），唯一 |
| `deletedAt` | DateTime? | 删除时间 |

#### TeamUser 模型（团队-用户关联）
`prisma/schema.prisma:216-230`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String (UUID) | 关联记录唯一标识 |
| `teamId` | String | 团队ID |
| `userId` | String | 用户ID |
| `role` | String | 团队内角色 |
| `createdAt` | DateTime | 创建时间 |
| `updatedAt` | DateTime | 更新时间 |

---

## 二、角色与权限体系

### 2.1 角色定义
`src/lib/constants.ts:164-217`

#### 系统级角色（User.role）
```typescript
ROLES = {
  admin: 'admin',           // 系统管理员
  user: 'user',             // 普通用户
  viewOnly: 'view-only',    // 仅查看用户
}
```

#### 团队级角色（TeamUser.role）
```typescript
ROLES = {
  teamOwner: 'team-owner',      // 团队所有者
  teamManager: 'team-manager',  // 团队管理员
  teamMember: 'team-member',    // 团队成员
  teamViewOnly: 'team-view-only', // 团队仅查看
}
```

### 2.2 权限映射（ROLE_PERMISSIONS）
`src/lib/constants.ts:186-217`

| 角色 | 权限 |
|------|------|
| `admin` | `all`（所有权限） |
| `user` | `website:create`, `website:update`, `website:delete`, `team:create` |
| `viewOnly` | 无 |
| `teamOwner` | `team:update`, `team:delete`, `website:create`, `website:update`, `website:delete`, `website:transfer-to-team`, `website:transfer-to-user` |
| `teamManager` | `team:update`, `website:create`, `website:update`, `website:delete`, `website:transfer-to-team` |
| `teamMember` | `website:create`, `website:update`, `website:delete` |
| `teamViewOnly` | 无 |

### 2.3 权限检查函数

#### 团队权限检查
`src/permissions/team.ts`

| 函数 | 权限要求 |
|------|----------|
| `canViewTeam` | 系统管理员 OR 团队成员 |
| `canUpdateTeam` | 系统管理员 OR 团队内有 `team:update` 权限（owner/manager） |
| `canDeleteTeam` | 系统管理员 OR 团队内有 `team:delete` 权限（owner） |
| `canDeleteTeamUser` | 系统管理员 OR 删除自己 OR 团队内有 `team:update` 权限 |

#### 用户权限检查
`src/permissions/user.ts`

| 函数 | 权限要求 |
|------|----------|
| `canCreateUser` | 系统管理员 |
| `canViewUser` | 系统管理员 OR 查看自己 |
| `canUpdateUser` | 系统管理员 OR 编辑自己 |
| `canDeleteUser` | 系统管理员 |

### 2.4 邀请链路的真实入口与权限边界

> ✅ **已证实**：邀请链路（即"添加团队成员"）与"网站转移（归属变更）"是**完全独立**的两个功能，不应混为一谈。

---

#### 功能一：发起邀请（添加团队成员）
**API 端点**：`POST /api/teams/{teamId}/users`  
**权限检查函数**：`canUpdateTeam(auth, teamId)`  
`src/permissions/team.ts:30-42`

```typescript
export async function canUpdateTeam({ user }: Auth, teamId: string) {
  if (!user) return false;
  if (user.isAdmin) return true;  // ✅ 代码证实：admin 直接通过
  
  const teamUser = await getTeamUser(teamId, user.id);
  return teamUser && hasPermission(teamUser.role, PERMISSIONS.teamUpdate);
}
```

**权限要求**：
- ✅ **已证实**：系统管理员（admin）- 直接通过
- ✅ **已证实**：团队所有者（teamOwner）- 拥有 `team:update` 权限
- ✅ **已证实**：团队管理员（teamManager）- 拥有 `team:update` 权限

**核心逻辑**：只需要在当前团队中有管理权限即可，不需要其他条件。

**邀请链路的另一个入口**：用户通过访问码主动加入  
**API 端点**：`POST /api/teams/join`  
**权限检查**：无特殊权限要求，只要登录用户 + 有效访问码即可加入

---

#### 功能二：网站转移（归属变更）
⚠️ **重要修正**：这不属于邀请链路，是独立的资源归属管理功能。

**API 端点**：`POST /api/websites/{websiteId}/transfer`  
**权限检查函数**：`canTransferWebsiteToTeam(auth, websiteId, teamId)`  
`src/permissions/website.ts:134-152`

✅ **已证实（代码原文）**：
```typescript
export async function canTransferWebsiteToTeam({ user }: Auth, websiteId: string, teamId: string) {
  if (!user) return false;
  // ❗ 关键发现：这里没有 if (user.isAdmin) return true; !!!!
  // 与 canUpdateTeam 等其他函数不一致！
  
  const website = await getWebsite(websiteId);
  if (!website) return false;
  
  // 条件1：必须是网站的当前个人所有者
  if (website.userId && website.userId === user.id) {
    // 条件2：必须在目标团队中有转移权限
    const teamUser = await getTeamUser(teamId, user.id);
    return teamUser && hasPermission(teamUser.role, PERMISSIONS.websiteTransferToTeam);
  }
  
  return false;
}
```

**权限要求（双重验证，admin 也不例外）**：
1. ✅ **已证实**：资源所有权验证 — 必须是网站的当前个人所有者（`website.userId === user.id`）
2. ✅ **已证实**：目标团队权限验证 — 在目标团队中必须拥有 `website:transfer-to-team` 权限
   - teamOwner：拥有此权限
   - teamManager：拥有此权限
   - teamMember：**没有**此权限
3. ❗ **已证实（代码缺失）**：系统管理员（admin）**没有豁免权**，也必须满足上述两个条件

**反向转移（团队→个人）** 的权限函数：`canTransferWebsiteToUser`  
`src/permissions/website.ts:114-132`

✅ **已证实**：
```typescript
export async function canTransferWebsiteToUser({ user }: Auth, websiteId: string, userId: string) {
  if (!user) return false;
  // ❗ 同样没有 if (user.isAdmin) return true;
  
  const website = await getWebsite(websiteId);
  if (!website) return false;
  
  // 条件1：网站必须是团队网站，且转移给当前用户自己
  if (website.teamId && user.id === userId) {
    // 条件2：在该团队中必须有 website:transfer-to-user 权限
    const teamUser = await getTeamUser(website.teamId, userId);
    return teamUser && hasPermission(teamUser.role, PERMISSIONS.websiteTransferToUser);
  }
  
  return false;
}
```

这意味着：
- ✅ **已证实**：只能把团队网站转移给自己，不能转移给其他人
- ✅ **已证实**：admin 也不能例外

**前端过滤**：`src/app/(main)/websites/[websiteId]/settings/WebsiteTransferForm.tsx:39-45`
```typescript
const items = teams?.data?.filter(({ members }) =>
  members.some(
    ({ role, userId }) =>
      [ROLES.teamOwner, ROLES.teamManager].includes(role) && userId === user.id,
  ),
) || [];
```
前端在 UI 层就过滤掉了用户没有 owner/manager 角色的团队。

---

#### 权限函数一致性对比

| 权限函数 | 是否有 `isAdmin` 豁免 | 代码位置 |
|---------|----------------------|----------|
| `canViewTeam` | ✅ 有 | `src/permissions/team.ts:11` |
| `canUpdateTeam` | ✅ 有 | `src/permissions/team.ts:35` |
| `canDeleteTeam` | ✅ 有 | `src/permissions/team.ts:49` |
| `canViewWebsite` | ✅ 有 | `src/permissions/website.ts:8` |
| `canUpdateWebsite` | ✅ 有 | `src/permissions/website.ts:63` |
| `canDeleteWebsite` | ✅ 有 | `src/permissions/website.ts:91` |
| `canTransferWebsiteToTeam` | ❌ **没有** | `src/permissions/website.ts:134` |
| `canTransferWebsiteToUser` | ❌ **没有** | `src/permissions/website.ts:114` |

⚠️ **已证实**：这是一个明显的设计不一致。网站转移是**唯一**没有 admin 豁免的权限检查。

---

#### 邀请 vs 转移 权限差异对比表

| 权限维度 | 发起邀请（添加成员） | 网站转移（归属变更） |
|---------|---------------------|-------------------|
| 功能定位 | 团队人员管理 | 资源归属管理 |
| 检查函数 | `canUpdateTeam` | `canTransferWebsiteToTeam` |
| 权限类型 | `team:update` | `website:transfer-to-team` |
| 资源所有权要求 | ❌ 不需要 | ✅ 必须是网站所有者 |
| 目标团队角色要求 | owner/manager | owner/manager |
| 系统管理员 | ✅ 直接通过 | ❌ **不豁免**，也必须是所有者 |
| 团队 owner | ✅ | ✅（同时要是网站所有者） |
| 团队 manager | ✅ | ✅（同时要是网站所有者） |
| 团队 member | ❌ | ❌ |
| 跨团队操作 | ❌ 只能在当前团队操作 | ✅ 可转移到自己有管理权的其他团队 |

**关键差异**：
- 发起邀请是**团队内部管理行为**，只看当前团队的角色
- 网站转移是**资源所有权转移行为**，需要同时验证资源所有权和目标团队的管理权
- 最关键差异：admin 在邀请中拥有完全权限，但在转移中没有豁免权

---

## 三、生命周期状态流转

### 3.1 状态说明

用户账号没有显式的 `status` 字段，而是通过 `deletedAt` 字段来标识状态：

- **激活状态**：`deletedAt IS NULL`
- **禁用/删除状态**：`deletedAt IS NOT NULL`（云模式）或物理删除（非云模式）

### 3.2 完整生命周期流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        账号生命周期                              │
└─────────────────────────────────────────────────────────────────┘
                │
                ▼
        ┌───────────────┐
        │  创建用户账号  │  POST /api/users
        │  (deletedAt=  │  - 仅 admin 可创建
        │   NULL)       │  - 初始角色：admin/user/view-only
        └───────┬───────┘
                │ 激活
                ▼
        ┌───────────────┐
        │   账号激活     │  getUser() 默认过滤 deletedAt=null
        │  (可登录使用)  │  登录验证：checkAuth() → getUser()
        └───────┬───────┘
                │
                │ ┌──────────────────────────────────────────┐
                │ │            团队邀请/加入流程             │
                │ └──────────────────────────────────────────┘
                │
                ▼
        ┌───────────────┐
        │  方式1：直接添加 │ POST /api/teams/{teamId}/users
        │  (管理员邀请)  │ - 权限：canUpdateTeam
        │               │ - 调用 createTeamUser(userId, teamId, role)
        └───────┬───────┘
                │
                └───────────┐
                            │
                            ▼
                    ┌───────────────┐
                    │  成为团队成员  │ TeamUser 记录创建
                    │  (role 生效)  │ 权限：基于 TeamUser.role
                    └───────┬───────┘
                            │
        ┌───────────────────┘
        │
        ▼
┌───────────────┐
│  方式2：访问码加入 │ POST /api/teams/join
│  (用户主动)    │ - 提供 accessCode
│               │ - 调用 findTeam({ accessCode })
│               │ - 自动授予 team-member 角色
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  角色变更      │ POST /api/teams/{teamId}/users/{userId}
│  (团队内)     │ - 权限：canUpdateTeam
│               │ - 调用 updateTeamUser(teamUserId, { role })
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  移除团队成员  │ DELETE /api/teams/{teamId}/users/{userId}
│               │ - 权限：canDeleteTeamUser
│               │ - 调用 deleteTeamUser(teamId, userId)
│               │ - 仅删除 TeamUser 关联，不影响用户账号
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  账号禁用/删除 │ DELETE /api/users/{userId}
│               │ - 权限：canDeleteUser（仅 admin）
│               │ - 不能删除自己
│               │ - 级联处理：
│               │   云模式：软删除（deletedAt=now，username随机化）
│               │   非云模式：物理删除 + 级联删除关联数据
└───────────────┘
```

---

## 四、各阶段详细分析

### 4.1 阶段一：邀请发起

#### 方式 A：管理员直接添加成员
**API 端点**：`POST /api/teams/{teamId}/users`  
`src/app/api/teams/[teamId]/users/route.ts:54-83`

**请求参数**：
```typescript
{
  userId: string,  // 被邀请用户ID（UUID）
  role: string     // 团队角色：team-manager / team-member / team-view-only
}
```

**权限检查**：
- `canUpdateTeam(auth, teamId)`：系统管理员 OR 团队 owner/manager
- 检查用户是否已是团队成员：`getTeamUser(teamId, userId)`

**业务逻辑**：
```typescript
// 1. 权限校验
if (!(await canUpdateTeam(auth, teamId))) {
  return unauthorized({ message: 'You must be the owner/manager of this team.' });
}

// 2. 重复性检查
const teamUser = await getTeamUser(teamId, userId);
if (teamUser) {
  return badRequest({ message: 'User is already a member of the Team.' });
}

// 3. 创建关联
const users = await createTeamUser(userId, teamId, role);
```

**数据层操作**：`src/queries/prisma/teamUser.ts:39-48`
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

#### 方式 B：生成访问码（用户主动加入）
访问码在团队设置中配置（`Team.accessCode`），通过 `TeamEditForm` 管理。

**前端界面**：`src/app/(main)/teams/[teamId]/TeamSettings.tsx:39`
```typescript
<TeamEditForm teamId={teamId} allowEdit={canEdit} showAccessCode={canEdit} />
```

### 4.2 阶段二：接受邀请

#### 方式 A：管理员直接添加
- 无需用户主动接受，`createTeamUser` 执行完成后即生效
- 用户下次登录时自动获得团队访问权限

#### 方式 B：用户通过访问码加入
**API 端点**：`POST /api/teams/join`  
`src/app/api/teams/join/route.ts:7-39`

**请求参数**：
```typescript
{
  accessCode: string  // 团队访问码
}
```

**业务逻辑**：
```typescript
// 1. 根据访问码查找团队
const team = await findTeam({ where: { accessCode } });
if (!team) {
  return notFound({ message: 'Team not found.', code: 'team-not-found' });
}

// 2. 检查是否已加入
const teamUser = await getTeamUser(team.id, auth.user.id);
if (teamUser) {
  return badRequest({ message: 'User is already a team member.' });
}

// 3. 创建关联，自动授予 team-member 角色
const user = await createTeamUser(auth.user.id, team.id, ROLES.teamMember);
```

### 4.3 阶段三：激活状态

用户账号创建后默认为激活状态（`deletedAt = NULL`）。

#### 激活验证逻辑
**用户查询**：`src/queries/prisma/user.ts:14-31`
```typescript
async function findUser(criteria, options: GetUserOptions = {}) {
  const { showDeleted = false } = options;
  
  return prisma.client.user.findUnique({
    ...criteria,
    where: {
      ...criteria.where,
      ...(showDeleted ? {} : { deletedAt: null }),  // 默认只查激活用户
    },
    // ...
  });
}
```

**团队成员查询**：`src/app/api/teams/[teamId]/users/route.ts:28-49`
```typescript
const users = await getTeamUsers({
  where: {
    teamId,
    user: {
      deletedAt: null,  // 只显示激活的团队成员
    },
  },
  // ...
});
```

**登录验证**：`src/app/api/auth/login/route.ts:26-30`
```typescript
const user = await getUserByUsername(username, { includePassword: true });
// getUserByUsername 默认过滤 deletedAt: null，禁用用户无法登录
```

### 4.4 阶段四：禁用/删除

#### 场景 A：移除团队成员（不影响账号）
**API 端点**：`DELETE /api/teams/{teamId}/users/{userId}`  
`src/app/api/teams/[teamId]/users/[userId]/route.ts:60-85`

**业务逻辑**：
```typescript
// 1. 权限校验
if (!(await canDeleteTeamUser(auth, teamId, userId))) {
  return unauthorized({ message: 'You must be the owner/manager of this team.' });
}

// 2. 检查存在性
const teamUser = await getTeamUser(teamId, userId);
if (!teamUser) {
  return badRequest({ message: 'The User does not exists on this team.' });
}

// 3. 删除关联（仅删除 TeamUser 记录）
await deleteTeamUser(teamId, userId);
```

**注意**：只删除团队关联关系，用户账号本身不受影响。

#### 场景 B：禁用/删除用户账号
**API 端点**：`DELETE /api/users/{userId}`  
`src/app/api/users/[userId]/route.ts:83-106`

**权限要求**：`canDeleteUser`（仅系统管理员）

**业务逻辑**：
```typescript
// 不能删除自己
if (userId === auth.user.id) {
  return badRequest({ message: 'You cannot delete yourself.' });
}

await deleteUser(userId);
```

**删除实现**：`src/queries/prisma/user.ts:102-206`

分两种模式：

**云模式（CLOUD_MODE=true）** - 软删除：
```typescript
return transaction([
  // 1. 软删除用户的网站
  client.website.updateMany({
    data: { deletedAt: new Date() },
    where: { id: { in: websiteIds } },
  }),
  // 2. 软删除用户（随机化用户名避免唯一冲突）
  client.user.update({
    data: {
      username: getRandomChars(32),
      deletedAt: new Date(),
    },
    where: { id: userId },
  }),
]);
```

**非云模式** - 物理删除（级联清理）：
```typescript
return transaction([
  client.eventData.deleteMany({ where: { websiteId: { in: websiteIds } } }),
  client.sessionData.deleteMany({ where: { websiteId: { in: websiteIds } } }),
  client.websiteEvent.deleteMany({ where: { websiteId: { in: websiteIds } } }),
  client.session.deleteMany({ where: { websiteId: { in: websiteIds } } }),
  // 删除用户的团队关联和其作为 owner 的团队
  client.teamUser.deleteMany({
    where: {
      OR: [
        { teamId: { in: teamIds } },  // 用户作为 owner 的团队的所有成员
        { userId },                    // 用户加入的所有团队
      ],
    },
  }),
  client.team.deleteMany({ where: { id: { in: teamIds } } }),  // 删除用户拥有的团队
  client.report.deleteMany({ /* ... */ }),
  client.website.deleteMany({ where: { id: { in: websiteIds } } }),
  client.user.delete({ where: { id: userId } }),
]);
```

**级联清理范围**：
- 用户创建的所有网站及其关联数据（事件、会话等）
- 用户拥有的团队（teamOwner 角色）及团队内所有成员关联
- 用户加入的所有团队的成员关联
- 用户创建的所有报表

### 4.5 禁用状态下的边界行为分析

#### 核心问题：`findTeam` 和 `getTeamUser` 是否过滤 `deletedAt`？

✅ **已证实（代码原文）**：`src/queries/prisma/team.ts:9-25`
```typescript
export async function findTeam(criteria: Prisma.TeamFindUniqueArgs): Promise<Team> {
  return prisma.client.team.findUnique(criteria);  // ❌ 没有过滤 deletedAt!
}

export async function getTeam(teamId: string, options = {}) {
  return findTeam({
    where: {
      id: teamId,  // ❌ 没有过滤 deletedAt!
    },
    // ...
  });
}
```

✅ **已证实（代码原文）**：`src/queries/prisma/teamUser.ts:12-19`
```typescript
export async function getTeamUser(teamId: string, userId: string) {
  return prisma.client.teamUser.findFirst({
    where: {
      teamId,
      userId,  // ❌ 没有关联检查 team.deletedAt 或 user.deletedAt!
    },
  });
}
```

---

#### 场景 A：团队被禁用后，访问码加入行为

**API 端点**：`POST /api/teams/join`  
`src/app/api/teams/join/route.ts:18-38`

✅ **已证实（代码原文）**：
```typescript
// 1. 根据访问码查找团队
const team = await findTeam({
  where: {
    accessCode,  // ❌ findTeam 没有过滤 deletedAt
  },
});

// 2. 检查是否已加入
const teamUser = await getTeamUser(team.id, auth.user.id);  // ❌ 也不检查 team.deletedAt

// 3. 创建成员关系
const user = await createTeamUser(auth.user.id, team.id, ROLES.teamMember);  // ✅ 仍然会写入!
```

✅ **已证实**：团队被禁用（`deletedAt IS NOT NULL`）后，**访问码仍然有效**，用户仍然可以通过访问码加入已禁用的团队。

**风险**：
- 已禁用的团队可能仍然有新成员加入
- 团队禁用的语义不明确（是临时禁用还是永久删除？）
- 可能导致数据不一致

---

#### 场景 B：团队被禁用后，管理员添加成员行为

**API 端点**：`POST /api/teams/{teamId}/users`  
`src/app/api/teams/[teamId]/users/route.ts:68-82`

✅ **已证实（代码原文）**：
```typescript
// 1. 权限检查
if (!(await canUpdateTeam(auth, teamId))) {
  return unauthorized(...);
}

// canUpdateTeam 内部调用 getTeamUser，也不检查 team.deletedAt
// 只要操作者在 TeamUser 表中有记录且角色是 owner/manager，就通过

// 2. 检查是否已加入
const teamUser = await getTeamUser(teamId, userId);  // ❌ 不检查 team.deletedAt

// 3. 创建成员关系
const users = await createTeamUser(userId, teamId, role);  // ✅ 仍然会写入!
```

✅ **已证实**：团队被禁用后，**管理员仍然可以向该团队添加新成员**。

---

#### 场景 C：用户被禁用后，添加成员行为

**API 端点**：`POST /api/teams/{teamId}/users`

✅ **已证实（代码原文）**：
```typescript
const { userId, role } = body;

// 检查是否已是成员
const teamUser = await getTeamUser(teamId, userId);
if (teamUser) {
  return badRequest(...);
}

// ❌ 关键缺失：没有调用 getUser(userId) 检查用户是否被禁用!
// 即使 userId 对应的用户 deletedAt IS NOT NULL，仍然会创建 TeamUser 记录

const users = await createTeamUser(userId, teamId, role);  // ✅ 仍然会写入!
```

✅ **已证实**：用户被禁用（`deletedAt IS NOT NULL`）后，**管理员仍然可以向该用户添加团队成员关系**。

✅ **已证实**：虽然 TeamUser 记录会创建成功，但在团队成员列表查询时，会过滤掉禁用用户：

`src/app/api/teams/[teamId]/users/route.ts:28-35`
```typescript
const users = await getTeamUsers({
  where: {
    teamId,
    user: {
      deletedAt: null,  // ✅ 这里会过滤掉禁用用户
    },
  },
  // ...
});
```

所以表现为：
- ✅ **已证实**：数据库中会创建 TeamUser 记录
- ✅ **已证实**：前端成员列表中看不到该用户
- ⚠️ **待验证**：权限检查时（如 `canViewTeam`），由于 `getUser` 默认过滤 `deletedAt: null`，禁用用户无法登录，实际也无法访问团队资源
  - *验证方法：创建测试用户，禁用后尝试直接调用 API 访问团队资源*

---

#### 禁用状态行为总结表

| 操作 | 团队已禁用 | 用户已禁用 |
|------|-----------|-----------|
| 用户通过访问码加入 | ✅ 仍然可以加入（已证实） | - |
| 管理员添加成员到团队 | ✅ 仍然可以添加（已证实） | ✅ 仍然可以添加，但列表不显示（已证实） |
| 团队成员列表显示 | ❌ 团队不会显示在列表中（已证实） | ❌ 用户不会显示在成员列表中（已证实） |
| 禁用用户登录 | - | ❌ 无法登录（已证实） |
| 禁用用户访问团队 | - | ❌ 无法访问（待验证） |

**设计缺陷**：
1. ✅ **已证实**：`findTeam` 和 `getTeam` 没有默认过滤 `deletedAt`，与 `findUser` 的行为不一致
2. ✅ **已证实**：添加成员时没有验证目标用户的激活状态
3. ✅ **已证实**：团队禁用的语义不明确，实际表现为"隐藏"而非"禁用"

---

### 4.6 删除账号时的团队成员关系处理差异及风险

#### 差异根源：云模式 vs 非云模式的设计目标不同

✅ **已证实**：
- **云模式**（CLOUD_MODE=true）：数据保留优先，支持恢复
- **非云模式**（自建部署）：数据彻底清除，避免残留

---

#### 云模式删除用户的实现
✅ **已证实（代码原文）**：`src/queries/prisma/user.ts:129-147`
```typescript
if (cloudMode) {
  return transaction([
    // 1. 软删除用户的网站
    client.website.updateMany({
      data: { deletedAt: new Date() },
      where: { id: { in: websiteIds } },
    }),
    // 2. 软删除用户（随机化用户名避免唯一冲突）
    client.user.update({
      data: {
        username: getRandomChars(32),
        deletedAt: new Date(),
      },
      where: { id: userId },
    }),
    // ❌ 完全没有处理 TeamUser 关联!
    // ❌ 完全没有处理用户拥有的团队!
  ]);
}
```

**云模式处理清单**：
| 处理项 | 是否处理 | 说明 |
|--------|---------|------|
| 用户账号 | ✅ 软删除（已证实） | `deletedAt = now()`，用户名随机化 |
| 用户的网站 | ✅ 软删除（已证实） | `deletedAt = now()` |
| 用户作为 owner 的团队 | ❌ 不处理（已证实） | 团队仍然存在，`deletedAt = NULL` |
| owner 团队的成员关系 | ❌ 不处理（已证实） | 所有 TeamUser 记录保留 |
| 用户加入的其他团队 | ❌ 不处理（已证实） | 用户的 TeamUser 记录保留 |
| 用户的报表 | ❌ 不处理（已证实） | 随网站软删除隐式隐藏 |

**云模式风险**：
1. ✅ **已证实**：**僵尸成员关系** — 用户被删除后，其 TeamUser 记录仍然存在
   - 在团队成员列表中，由于查询时过滤 `user.deletedAt = null`，该用户不会显示
   - 但数据库中存在无效的 TeamUser 记录，可能导致数据不一致
   
2. ✅ **已证实**：**无主团队（Orphaned Team）**：
   - 用户作为 teamOwner 的团队仍然存在
   - 团队没有了 owner，但 `getTeamOwner()` 仍然会返回该用户（如果不过滤 deletedAt）
   - ⚠️ **待验证**：其他成员无法管理团队（因为只有 owner 可以删除团队，manager 不能删除）
     - *验证方法：创建测试团队，删除 owner 用户后用 manager 账号尝试删除团队*
   - ✅ **已证实（代码原文）**：`src/queries/prisma/team.ts:103-108`
     ```typescript
     export async function getTeamOwner(teamId: string) {
       return prisma.client.teamUser.findFirst({
         where: { teamId, role: ROLES.teamOwner },  // ❌ 不过滤 user.deletedAt
         select: { userId: true },
       });
     }
     ```

3. ⚠️ **待验证**：**恢复风险** — 如果后续"取消删除"用户（设置 `deletedAt = NULL`），用户会：
   - 自动恢复所有团队成员身份
   - 自动恢复所有团队所有权
   - 可能导致权限意外恢复
   - *验证方法：创建测试用户，软删除后手动更新 `deletedAt = NULL`，检查权限是否恢复*

4. ⚠️ **待验证**：**统计数据偏差** — 团队成员计数可能不准确
   - *验证方法：检查 `getTeamUsersCount` 等统计函数是否过滤 `deletedAt`*

---

#### 非云模式删除用户的实现
✅ **已证实（代码原文）**：`src/queries/prisma/user.ts:149-206`
```typescript
return transaction([
  // ... 删除网站的所有关联数据（eventData, sessionData, websiteEvent, session）...
  
  // 关键：处理团队成员关系
  client.teamUser.deleteMany({
    where: {
      OR: [
        { teamId: { in: teamIds } },  // 1. 用户作为 owner 的团队的所有成员
        { userId },                    // 2. 用户加入的所有团队
      ],
    },
  }),
  
  // 删除用户拥有的团队
  client.team.deleteMany({
    where: { id: { in: teamIds } },
  }),
  
  // ... 删除报表、网站，最后物理删除用户 ...
  client.user.delete({ where: { id: userId } }),
]);
```

**非云模式处理清单**：
| 处理项 | 是否处理 | 说明 |
|--------|---------|------|
| 用户账号 | ✅ 物理删除（已证实） | 从数据库彻底清除 |
| 用户的网站 | ✅ 物理删除（已证实） | 级联删除所有关联数据 |
| 用户作为 owner 的团队 | ✅ 删除团队（已证实） | 整个团队被物理删除 |
| owner 团队的成员关系 | ✅ 删除全部（已证实） | 团队内所有成员的 TeamUser 记录被删除 |
| 用户加入的其他团队 | ✅ 移除用户（已证实） | 只删除该用户的 TeamUser 记录，团队保留 |
| 用户的报表 | ✅ 删除全部（已证实） | 用户创建的所有报表被删除 |

**非云模式风险**：
1. ✅ **已证实**：**级联删除影响面大**：
   - 删除一个用户可能导致多个团队被删除
   - 团队内所有成员失去该团队的访问权限
   
2. ⚠️ **待验证**：团队内的所有网站数据被删除（如果网站归团队所有）
   - *验证方法：创建团队+团队网站，删除 owner 用户后检查网站是否存在*
   
3. ✅ **已证实**：**不可逆** — 物理删除无法恢复，操作需极其谨慎

4. ⚠️ **待验证**：**数据完整性** — 删除用户作为 teamOwner 的团队时，团队的网站也会被级联删除吗？
   - 代码中没有显式删除团队的网站，但团队删除后，网站的 `teamId` 变为无效
   - 实际测试需要确认外键约束行为和查询逻辑

---

#### 两种模式对比表

| 维度 | 云模式（软删除） | 非云模式（物理删除） |
|------|----------------|-------------------|
| 用户账号 | 软删除，保留记录（已证实） | 物理删除，彻底清除（已证实） |
| 团队成员关系 | 完全保留（已证实） | 级联清除（已证实） |
| 用户拥有的团队 | 保留，变成无主（已证实） | 彻底删除（已证实） |
| 可恢复性 | ✅ 可恢复（待验证） | ❌ 不可恢复（已证实） |
| 数据一致性 | ⚠️ 存在僵尸数据风险（已证实） | ✅ 彻底清理（已证实） |
| 影响范围 | 小（仅用户和其网站）（已证实） | 大（级联影响团队成员）（已证实） |
| 适用场景 | SaaS 云服务，需要审计和恢复（推断） | 自建部署，数据安全优先（推断） |

---

#### 云模式下的团队删除对比

为了完整性，对比一下团队删除在两种模式下的处理：
✅ **已证实（代码原文）**：`src/queries/prisma/team.ts:143-172`
```typescript
export async function deleteTeam(teamId: string) {
  if (cloudMode) {
    return transaction([
      client.team.update({
        data: { deletedAt: new Date() },  // ✅ 软删除团队
        where: { id: teamId },
      }),
      // ❌ 没有处理 TeamUser 关联!
      // ❌ 没有处理团队的网站!
    ]);
  }

  return transaction([
    client.teamUser.deleteMany({ where: { teamId } }),  // ✅ 删除所有成员关联
    client.team.delete({ where: { id: teamId } }),       // ✅ 物理删除团队
  ]);
}
```

✅ **已证实**：即使是删除团队，云模式下也不会处理 TeamUser 关联和团队网站，这会导致：
- 团队被软删除后，TeamUser 记录仍然存在
- 团队的网站 `teamId` 仍然指向已删除的团队
- 查询用户团队列表时，由于过滤 `team.deletedAt = null`，用户看不到该团队，但数据库中存在无效关联

---

## 五、关键代码索引

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| 角色权限定义 | `src/lib/constants.ts` | 164-217 |
| 权限检查（团队） | `src/permissions/team.ts` | 1-92 |
| 权限检查（用户） | `src/permissions/user.ts` | 1-37 |
| 添加团队成员 API | `src/app/api/teams/[teamId]/users/route.ts` | 54-83 |
| 访问码加入 API | `src/app/api/teams/join/route.ts` | 7-39 |
| 团队成员管理 API | `src/app/api/teams/[teamId]/users/[userId]/route.ts` | 1-85 |
| 用户创建 API | `src/app/api/users/route.ts` | 11-45 |
| 用户删除 API | `src/app/api/users/[userId]/route.ts` | 83-106 |
| 登录验证 | `src/app/api/auth/login/route.ts` | 12-48 |
| 用户查询 | `src/queries/prisma/user.ts` | 1-206 |
| 团队操作 | `src/queries/prisma/team.ts` | 1-172 |
| 团队用户关联操作 | `src/queries/prisma/teamUser.ts` | 1-66 |
| 认证中间件 | `src/lib/auth.ts` | 17-60 |
| 添加成员表单 | `src/app/(main)/teams/TeamMemberAddForm.tsx` | 1-72 |
| 加入团队表单 | `src/app/(main)/teams/TeamJoinForm.tsx` | 1-40 |
| 团队设置 | `src/app/(main)/teams/[teamId]/TeamSettings.tsx` | 1-55 |

---

## 六、设计特点总结

### 6.1 优点
1. **双轨角色体系**：系统角色 + 团队角色，权限粒度清晰
2. **软删除机制**：云模式下保留数据，便于审计和恢复
3. **权限分级**：管理员、团队所有者、团队管理员、成员各司其职
4. **级联清理**：非云模式彻底删除，避免数据残留
5. **双重权限验证**：网站转移需要同时验证资源所有权和目标团队管理权

### 6.2 已知风险与设计缺陷
1. **无显式邀请状态**：没有 pending/accepted 状态，管理员添加即生效
2. **无邮件通知**：当前代码未实现邀请邮件发送机制
3. **无过期机制**：访问码无过期时间，永久有效直到手动修改
4. **删除不可逆**：非云模式下删除为物理删除，不可恢复
5. **`findTeam` 未过滤 `deletedAt`**：与 `findUser` 行为不一致，导致禁用团队仍可被访问和加入
6. **添加成员不验证用户状态**：可以向已禁用用户添加团队成员关系
7. **云模式数据一致性风险**：删除用户/团队时保留 TeamUser 关联，产生僵尸数据
8. **无主团队风险**：云模式下删除 teamOwner 用户后，团队可能无人管理

### 6.3 权限边界
| 操作 | admin | teamOwner | teamManager | teamMember | teamViewOnly |
|------|-------|-----------|-------------|------------|--------------|
| 创建用户 | ✅ | ❌ | ❌ | ❌ | ❌ |
| 删除用户 | ✅ | ❌ | ❌ | ❌ | ❌ |
| 添加团队成员 | ✅ | ✅ | ✅ | ❌ | ❌ |
| 移除团队成员 | ✅ | ✅ | ✅ | 仅自己 | ❌ |
| 修改成员角色 | ✅ | ✅ | ✅ | ❌ | ❌ |
| 删除团队 | ✅ | ✅ | ❌ | ❌ | ❌ |
| 创建网站 | ✅ | ✅ | ✅ | ✅ | ❌ |
| 转移网站到团队 | ⚠️ | ✅ | ✅ | ❌ | ❌ |

> **重要说明**：
> - ✅ 表示有权限
> - ⚠️ **关键修正**：`转移网站到团队时，admin **没有豁免权**，也必须是网站的当前个人所有者 + 在目标团队中有 transfer 权限
> - 其他操作中 admin 都有豁免权，唯有网站转移例外

### 6.4 云模式 vs 非云模式风险对比

| 风险类型 | 云模式 | 非云模式 |
|---------|--------|----------|
| 僵尸数据 | ⚠️ 高（TeamUser 关联残留） | ✅ 低（彻底清除） |
| 无主团队 | ⚠️ 高（owner 删除后团队保留） | ✅ 低（团队随 owner 删除） |
| 级联影响 | ✅ 低（仅影响用户和其网站） | ⚠️ 高（影响团队所有成员） |
| 可恢复性 | ✅ 高（软删除可恢复） | ❌ 无 |
| 数据残留 | ⚠️ 高 | ✅ 低 |
| 操作不可逆风险 | ✅ 低 | ⚠️ 高 |

### 6.5 关键边界行为总结

#### 禁用状态下的写入行为
| 场景 | 是否允许写入 | 说明 |
|------|-------------|------|
| 禁用团队 + 访问码加入 | ✅ 允许 | `findTeam` 不过滤 `deletedAt` |
| 禁用团队 + 添加成员 | ✅ 允许 | `canUpdateTeam` 不过滤 `deletedAt` |
| 禁用用户 + 添加成员 | ✅ 允许（数据库） | 但成员列表不显示，用户无法登录 |

#### 关键查询函数的过滤行为
| 函数 | 是否过滤 `deletedAt` | 备注 |
|------|---------------------|------|
| `findUser` / `getUser` | ✅ 默认过滤 | `showDeleted` 选项可关闭 |
| `findTeam` / `getTeam` | ❌ 不过滤 | 可能返回已禁用的团队 |
| `getTeamUser` | ❌ 不过滤 | 不关联检查 team/user 的状态 |
| `getTeamUsers` | ✅ 过滤 `user.deletedAt` | 成员列表只显示激活用户 |
| `getUserTeams` | ✅ 过滤 `team.deletedAt` | 只显示未删除的团队 |

**设计不一致性**：写入操作不检查状态，但读取操作（列表查询）会过滤状态，导致"写得进，读不出"的现象。

---

## 七、确定事实 vs 待验证假设汇总

本章节汇总全文中所有结论的可信度层级。

### 7.1 已证实（代码原文可证）

✅ **邀请链路相关**：
1. 邀请链路的真实入口只有两个：
   - `POST /api/teams/{teamId}/users` - 管理员直接添加成员
   - `POST /api/teams/join` - 用户通过访问码加入
2. 网站转移（`POST /api/websites/{websiteId}/transfer`）不属于邀请链路，是独立的资源归属管理功能
3. `canUpdateTeam`（用于邀请）有 `isAdmin` 豁免，admin 可直接添加成员
4. `canTransferWebsiteToTeam` **没有** `isAdmin` 豁免，admin 也必须是网站所有者 + 在目标团队有 transfer 权限
5. `canTransferWebsiteToUser` **没有** `isAdmin` 豁免，且只能转移给自己
6. 网站转移是唯一没有 admin 豁免的权限检查，与其他 7 个权限函数不一致
7. `findTeam` 和 `getTeam` 没有过滤 `deletedAt`，与 `findUser` 行为不一致
8. `getTeamUser` 没有关联检查 `team.deletedAt` 或 `user.deletedAt`

✅ **禁用状态相关**：
1. 团队被禁用（`deletedAt IS NOT NULL`）后，仍然可以通过访问码加入
2. 团队被禁用后，管理员仍然可以向该团队添加新成员
3. 用户被禁用后，管理员仍然可以向该用户添加团队成员关系（数据库写入成功）
4. 团队成员列表查询会过滤 `user.deletedAt = null`，禁用用户不会显示在列表中
5. 用户登录时 `getUserByUsername` 默认过滤 `deletedAt = null`，禁用用户无法登录

✅ **删除账号相关**：
1. 云模式删除用户时，只软删除用户和其网站，**完全不处理** TeamUser 关联和用户拥有的团队
2. 非云模式删除用户时，级联删除：
   - 用户的所有网站及其关联数据
   - 用户作为 owner 的团队的所有成员关联
   - 用户加入的其他团队中该用户的成员关联
   - 用户拥有的团队
   - 用户的所有报表
   - 用户本身（物理删除）
3. 云模式删除团队时，只软删除团队本身，**不处理** TeamUser 关联和团队的网站
4. 非云模式删除团队时，删除所有 TeamUser 关联后物理删除团队
5. `getTeamOwner` 函数不过滤 `user.deletedAt`，可能返回已禁用的用户

### 7.2 待验证（需要运行时测试或边界场景验证）

⚠️ **权限验证类**：
1. 禁用用户通过 API 直接调用 `canViewTeam` 等权限检查是否真的被拒绝
   - *验证方法：创建测试用户，禁用后尝试直接调用 API 访问团队资源，绕过前端*

⚠️ **数据一致性类**：
1. 云模式下，无主团队（owner 被软删除）是否真的无法被管理
   - *验证方法：创建测试团队，删除 owner 用户后用 manager 账号尝试删除团队、修改团队设置*
2. 云模式下，"取消删除"用户（手动设置 `deletedAt = NULL`）后，是否自动恢复所有团队权限
   - *验证方法：创建测试用户，软删除后手动更新数据库，再登录检查权限*
3. 团队成员计数等统计函数是否过滤 `deletedAt`
   - *验证方法：检查 `getTeamUsersCount` 等函数的实现，或添加禁用用户后查看统计数据*

⚠️ **级联删除类**：
1. 非云模式下，删除 teamOwner 用户时，团队的网站会被怎样处理
   - *验证方法：创建团队+团队网站，删除 owner 用户后检查网站记录是否存在、teamId 是否为 NULL*
2. 非云模式下，数据库外键约束在团队删除时的实际行为
   - *验证方法：检查 Prisma schema 中外键定义，删除团队后查询关联网站的状态*

### 7.3 推断（基于代码逻辑的合理推测，但无直接代码证明）

💭 **设计意图推断**：
1. 云模式的设计目标是 SaaS 云服务，需要审计和恢复能力
2. 非云模式的设计目标是自建部署，数据安全和彻底清理优先
3. 网站转移没有 admin 豁免可能是**故意设计**，防止 admin 越权转移用户私有网站
   - *但也可能是代码遗漏，需要结合项目历史或文档确认*

### 7.4 标注说明

| 标记 | 含义 | 可信度 |
|------|------|--------|
| ✅ 已证实 | 代码原文可直接证明，或通过代码逻辑链可明确推导 | 100% |
| ⚠️ 待验证 | 代码逻辑暗示了某种行为，但缺少边界场景测试，或需要运行时验证 | ~70% |
| 💭 推断 | 基于代码模式和常识的合理推测，无直接代码证据 | ~50% |

**使用建议**：
- 对于 ✅ 已证实的结论，可以直接作为开发和问题排查的依据
- 对于 ⚠️ 待验证的结论，建议在关键业务场景中进行实际测试验证
- 对于 💭 推断，仅作理解代码设计的参考，不作为决策依据
