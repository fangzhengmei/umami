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

### 6.2 注意事项
1. **无显式邀请状态**：没有 pending/accepted 状态，管理员添加即生效
2. **无邮件通知**：当前代码未实现邀请邮件发送机制
3. **无过期机制**：访问码无过期时间，永久有效直到手动修改
4. **删除不可逆**：非云模式下删除为物理删除，不可恢复

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
