# Team 成员数据结构与展示链路分析

## 一、数据库层设计

### 1.1 核心数据表

#### TeamUser 关联表 (`prisma/schema.prisma:216-230`)

Team 与 User 的多对多关系通过 `TeamUser` 中间表实现：

```prisma
model TeamUser {
  id        String    @id() @map("team_user_id") @db.Uuid
  teamId    String    @map("team_id") @db.Uuid
  userId    String    @map("user_id") @db.Uuid
  role      String    @db.VarChar(50)
  createdAt DateTime? @default(now()) @map("created_at") @db.Timestamptz(6)
  updatedAt DateTime? @updatedAt @map("updated_at") @db.Timestamptz(6)

  team Team @relation(fields: [teamId], references: [id])
  user User @relation(fields: [userId], references: [id])

  @@index([teamId])
  @@index([userId])
  @@map("team_user")
}
```

**关键字段说明：**
- `id`: 主键，UUID 格式
- `teamId`: 外键，关联 Team 表
- `userId`: 外键，关联 User 表
- `role`: 成员在团队中的角色
- `createdAt`: 创建时间（成员加入时间）
- `updatedAt`: 更新时间

#### User 表 (`prisma/schema.prisma:12-32`)

```prisma
model User {
  id          String    @id() @map("user_id") @db.Uuid
  username    String    @unique @db.VarChar(255)
  password    String    @db.VarChar(60)
  role        String    @map("role") @db.VarChar(50)
  logoUrl     String?   @map("logo_url") @db.VarChar(2183)
  displayName String?   @map("display_name") @db.VarChar(255)
  createdAt   DateTime? @default(now()) @map("created_at") @db.Timestamptz(6)
  updatedAt   DateTime? @updatedAt @map("updated_at") @db.Timestamptz(6)
  deletedAt   DateTime? @map("deleted_at") @db.Timestamptz(6)

  teams TeamUser[]
  // ... 其他关联
}
```

#### Team 表 (`prisma/schema.prisma:197-214`)

```prisma
model Team {
  id         String    @id() @map("team_id") @db.Uuid
  name       String    @db.VarChar(50)
  accessCode String?   @unique @map("access_code") @db.VarChar(50)
  logoUrl    String?   @map("logo_url") @db.VarChar(2183)
  createdAt  DateTime? @default(now()) @map("created_at") @db.Timestamptz(6)
  updatedAt  DateTime? @updatedAt @map("updated_at") @db.Timestamptz(6)
  deletedAt  DateTime? @map("deleted_at") @db.Timestamptz(6)

  members TeamUser[]
  // ... 其他关联
}
```

### 1.2 ER 关系图

```
User ───< TeamUser >─── Team
  │         │  │         │
  │         │  │         ├─ id
  │         │  │         ├─ name
  │         │  │         └─ accessCode
  │         │  │
  │         │  └─ userId
  │         │  └─ teamId
  │         │  └─ role
  │         │
  │         └─ id
  │         └─ username
  └─ id
  └─ username
  └─ role (全局角色)
```

---

## 二、角色系统设计

### 2.1 角色定义 (`src/lib/constants.ts:164-172`)

```typescript
export const ROLES = {
  admin: 'admin',                    // 系统管理员
  user: 'user',                      // 普通用户
  viewOnly: 'view-only',             // 全局只读用户
  teamOwner: 'team-owner',           // 团队所有者
  teamManager: 'team-manager',       // 团队管理员
  teamMember: 'team-member',         // 团队成员
  teamViewOnly: 'team-view-only',    // 团队只读成员
} as const;
```

### 2.2 团队角色权限 (`src/lib/constants.ts:195-217`)

```typescript
export const ROLE_PERMISSIONS = {
  [ROLES.teamOwner]: [
    PERMISSIONS.teamUpdate,      // ✓ 可以更新团队
    PERMISSIONS.teamDelete,      // ✓ 可以删除团队
    PERMISSIONS.websiteCreate,
    PERMISSIONS.websiteUpdate,
    PERMISSIONS.websiteDelete,
    PERMISSIONS.websiteTransferToTeam,
    PERMISSIONS.websiteTransferToUser,
  ],
  [ROLES.teamManager]: [
    PERMISSIONS.teamUpdate,      // ✓ 可以更新团队
    PERMISSIONS.websiteCreate,
    PERMISSIONS.websiteUpdate,
    PERMISSIONS.websiteDelete,
    PERMISSIONS.websiteTransferToTeam,
    // 注意：teamManager 没有 teamDelete 权限
  ],
  [ROLES.teamMember]: [
    PERMISSIONS.websiteCreate,
    PERMISSIONS.websiteUpdate,
    PERMISSIONS.websiteDelete,
    // 注意：teamMember 没有 teamUpdate 权限
  ],
  [ROLES.teamViewOnly]: [],
} as const;
```

**关键结论：**
- `teamOwner` 和 `teamManager` 都有 `teamUpdate` 权限
- 只有 `teamOwner` 有 `teamDelete` 权限
- `teamMember` 和 `teamViewOnly` 都没有 `teamUpdate` 权限

### 2.3 Schema 验证 (`src/lib/schema.ts:88`)

```typescript
export const teamRoleParam = z.enum(['team-member', 'team-view-only', 'team-manager']);
```

> **注意**: `teamOwner` 不在此枚举中，无法通过 API 直接设置或修改，它是团队创建时默认分配的角色。

---

## 三、数据查询层

### 3.1 Prisma 查询封装 (`src/queries/prisma/teamUser.ts`)

#### 获取团队成员列表

```typescript
export async function getTeamUsers(criteria: TeamUserFindManyArgs, filters?: QueryFilters) {
  const { search } = filters;

  const where: Prisma.TeamUserWhereInput = {
    ...criteria.where,
    ...prisma.getSearchParameters(search, [{ user: { username: 'contains' } }]),
  };

  return prisma.pagedQuery(
    'teamUser',
    {
      ...criteria,
      where,
    },
    filters,
  );
}
```

**功能特性：**
- 支持分页查询
- 支持按用户名搜索（模糊匹配）
- 支持传入自定义查询条件

#### 其他查询方法

```typescript
// 查找单个团队用户（按 teamId + userId）
export async function getTeamUser(teamId: string, userId: string)

// 创建团队成员
export async function createTeamUser(userId: string, teamId: string, role: string)

// 更新团队成员
export async function updateTeamUser(teamUserId: string, data: Prisma.TeamUserUpdateInput)

// 删除团队成员
export async function deleteTeamUser(teamId: string, userId: string)
```

---

## 四、API 接口层

### 4.1 获取团队成员列表 - GET `/api/teams/{teamId}/users`

**文件位置**: `src/app/api/teams/[teamId]/users/route.ts:8-52`

#### 请求参数

```typescript
const schema = z.object({
  ...pagingParams,    // page, pageSize
  ...searchParams,    // search
});
```

#### 权限检查

```typescript
if (!(await canViewTeam(auth, teamId))) {
  return unauthorized({ message: 'You must be a member of this team.' });
}
```

#### 查询逻辑

```typescript
const users = await getTeamUsers(
  {
    where: {
      teamId,
      user: {
        deletedAt: null,  // 过滤已删除用户
      },
    },
    include: {
      user: {
        select: {
          id: true,
          username: true,
        },
      },
    },
    orderBy: {
      createdAt: 'asc',  // 按加入时间升序
    },
  },
  filters,
);
```

#### 返回数据结构

```typescript
{
  data: [
    {
      id: string,           // TeamUser ID
      teamId: string,
      userId: string,
      role: string,
      createdAt: DateTime,
      updatedAt: DateTime,
      user: {
        id: string,         // User ID
        username: string
      }
    }
  ],
  count: number,            // 总数
  page: number,
  pageSize: number
}
```

### 4.2 添加团队成员 - POST `/api/teams/{teamId}/users`

**文件位置**: `src/app/api/teams/[teamId]/users/route.ts:54-82`

#### 请求体 Schema

```typescript
const schema = z.object({
  userId: z.uuid(),
  role: teamRoleParam,  // 'team-member' | 'team-view-only' | 'team-manager'
});
```

#### 权限检查

```typescript
if (!(await canUpdateTeam(auth, teamId))) {
  return unauthorized({ message: 'You must be the owner/manager of this team.' });
}
```

**注意**: `canUpdateTeam` 检查 `teamUpdate` 权限，只有 teamOwner 和 teamManager 有此权限。

#### 业务逻辑

```typescript
// 检查用户是否已在团队中
const teamUser = await getTeamUser(teamId, userId);

if (teamUser) {
  return badRequest({ message: 'User is already a member of the Team.' });
}

// 创建成员
const users = await createTeamUser(userId, teamId, role);
```

### 4.3 编辑团队成员 - POST `/api/teams/{teamId}/users/{userId}`

**文件位置**: `src/app/api/teams/[teamId]/users/[userId]/route.ts:29-58`

#### 请求体 Schema

```typescript
const schema = z.object({
  role: teamRoleParam,  // 仅允许修改角色
});
```

#### 权限检查

```typescript
if (!(await canUpdateTeam(auth, teamId))) {
  return unauthorized({ message: 'You must be the owner/manager of this team.' });
}
```

**注意**: 编辑权限与添加权限相同，均检查 `teamUpdate` 权限。

#### 业务逻辑

```typescript
// 检查用户是否存在于团队中
const teamUser = await getTeamUser(teamId, userId);

if (!teamUser) {
  return badRequest({ message: 'The User does not exists on this team.' });
}

// 更新成员角色
const user = await updateTeamUser(teamUser.id, body);
```

### 4.4 移除团队成员 - DELETE `/api/teams/{teamId}/users/{userId}`

**文件位置**: `src/app/api/teams/[teamId]/users/[userId]/route.ts:60-85`

#### 权限检查 (`src/permissions/team.ts:58-74`)

```typescript
export async function canDeleteTeamUser({ user }: Auth, teamId: string, removeUserId: string) {
  if (!user) {
    return false;
  }

  // 分支1：系统管理员 → 允许删除任何人
  if (user.isAdmin) {
    return true;
  }

  // 分支2：用户删除自己 → 允许离开团队（无需角色权限）
  if (removeUserId === user.id) {
    return true;
  }

  // 分支3：团队成员且有 teamUpdate 权限 → 允许删除他人
  const teamUser = await getTeamUser(teamId, user.id);

  return teamUser && hasPermission(teamUser.role, PERMISSIONS.teamUpdate);
}
```

**权限判定分支总结：**
| 场景 | 条件 | 结果 | 说明 |
|------|------|------|------|
| 管理员删除 | `user.isAdmin === true` | ✓ 允许 | 系统管理员可删除任意成员 |
| 用户自删除 | `removeUserId === user.id` | ✓ 允许 | 成员可随时离开团队，无需角色 |
| 有管理权限删除 | 是团队成员 + 有 teamUpdate 权限 | ✓ 允许 | teamOwner/teamManager 可删他人 |
| 无权限删除 | 普通成员删除他人 | ✗ 拒绝 | teamMember/teamViewOnly 不可删他人 |

#### 业务逻辑

```typescript
// 检查用户是否存在于团队中
const teamUser = await getTeamUser(teamId, userId);

if (!teamUser) {
  return badRequest({ message: 'The User does not exists on this team.' });
}

// 删除成员关联
await deleteTeamUser(teamId, userId);
```

---

## 五、缓存刷新机制详解

### 5.1 Zustand Store 实现 (`src/components/hooks/useModified.ts`)

```typescript
import { create } from 'zustand';

const store = create(() => ({}));

export function touch(key: string) {
  store.setState({ [key]: Date.now() });
}

export function useModified(key?: string) {
  const modified = store(state => state?.[key]);

  return { modified, touch };
}
```

**核心机制：**
- 使用 Zustand 创建一个全局状态 store
- `touch(key)`: 更新指定 key 的值为当前时间戳 `Date.now()`
- `useModified(key)`: 订阅指定 key 的值变化

### 5.2 modified 与 touch 的关系

```
┌─────────────────────────────────────────────────────────┐
│                    Zustand Store                         │
│  { 'teams:members': 1715798400000, ... }                │
└─────────────┬───────────────────────────────────────────┘
              │
      ┌───────┴────────┐
      │                │
      ▼                ▼
  useModified()    touch(key)
      │                │
      │                └─ 更新 store[key] = Date.now()
      │
      └─ 订阅 store[key] 的变化
          返回 { modified: value, touch }
```

**工作原理：**
1. `touch('teams:members')` 触发状态更新，时间戳变化
2. `useTeamMembersQuery` 中的 `modified` 值变化
3. 因为 `modified` 是 `queryKey` 的一部分，React Query 自动重新获取数据

### 5.3 React Query 集成 (`src/components/hooks/queries/useTeamMembersQuery.ts`)

```typescript
export function useTeamMembersQuery(teamId: string) {
  const { get } = useApi();
  const { modified } = useModified(`teams:members`);

  return usePagedQuery({
    queryKey: ['teams:members', { teamId, modified }],  // modified 作为查询键的一部分
    queryFn: (params: any) => {
      return get(`/teams/${teamId}/users`, params);
    },
    enabled: !!teamId,
  });
}
```

**关键理解：**
- `modified` 作为 `queryKey` 的一部分
- 当 `touch('teams:members')` 被调用时，`modified` 值（时间戳）变化
- React Query 检测到 `queryKey` 变化，自动触发重新查询

---

## 六、三类成员操作完整处理链路

### 6.1 新增团队成员

#### 触发点：`TeamsMemberAddButton.tsx:18-22`

```typescript
const handleSave = async () => {
  toast(t(messages.saved));
  touch('teams:members');  // 触发缓存刷新
  onSave?.();
};
```

#### 完整链路：

```
1. 点击 "Add Member" 按钮
   ↓
2. 打开 TeamMemberAddForm 表单
   ↓
3. 选择用户和角色，提交表单
   ↓
4. 调用 POST /api/teams/{teamId}/users
   ↓
5. API 成功返回
   ↓
6. 触发 handleSave 回调
   ├─ 显示 "Saved" 提示
   ├─ touch('teams:members') → 更新 Zustand store
   └─ 关闭对话框
   ↓
7. useTeamMembersQuery 中的 modified 值变化
   ↓
8. queryKey 变化，React Query 自动重新获取数据
   ↓
9. GET /api/teams/{teamId}/users 获取最新成员列表
   ↓
10. TeamMembersTable 重新渲染，显示新增成员
```

### 6.2 编辑团队成员

#### 触发点：`TeamMemberEditButton.tsx:22-26`

```typescript
const handleSave = () => {
  touch('teams:members');  // 触发缓存刷新
  toast(t(messages.saved));
  onSave?.();
};
```

#### 完整链路：

```
1. 点击成员行的 Edit 按钮
   ↓
2. 打开 TeamMemberEditForm 表单（预填当前角色）
   ↓
3. 修改角色，提交表单
   ↓
4. 调用 POST /api/teams/{teamId}/users/{userId}
   ↓
5. API 成功返回
   ↓
6. 触发 handleSave 回调
   ├─ touch('teams:members') → 更新 Zustand store
   ├─ 显示 "Saved" 提示
   └─ 关闭对话框
   ↓
7. useTeamMembersQuery 中的 modified 值变化
   ↓
8. queryKey 变化，React Query 自动重新获取数据
   ↓
9. GET /api/teams/{teamId}/users 获取最新成员列表
   ↓
10. TeamMembersTable 重新渲染，显示更新后的角色
```

### 6.3 移除团队成员

#### 6.3.1 管理员/管理者移除成员

**触发点**：`TeamMemberRemoveButton.tsx:22-30`

```typescript
const handleConfirm = async (close: () => void) => {
  await mutateAsync(null, {
    onSuccess: () => {
      touch('teams:members');  // 触发缓存刷新
      onSave?.();
      close();
    },
  });
};
```

#### 完整链路（移除他人）：

```
1. 点击成员行的 Remove 按钮（仅 allowEdit=true 时显示）
   ↓
2. 打开确认对话框
   ↓
3. 点击确认删除
   ↓
4. 调用 DELETE /api/teams/{teamId}/users/{userId}
   ↓
5. API 权限检查：canDeleteTeamUser
   ├─ 管理员：通过
   ├─ teamOwner：通过（有 teamUpdate 权限）
   └─ teamManager：通过（有 teamUpdate 权限）
   ↓
6. API 成功返回
   ↓
7. 触发 onSuccess 回调
   ├─ touch('teams:members') → 更新 Zustand store
   ├─ 关闭对话框
   └─ 执行额外回调 onSave
   ↓
8. useTeamMembersQuery 中的 modified 值变化
   ↓
9. queryKey 变化，React Query 自动重新获取数据
   ↓
10. GET /api/teams/{teamId}/users 获取最新成员列表
   ↓
11. TeamMembersTable 重新渲染，被移除的成员消失
```

#### 6.3.2 用户主动离开团队（自删除）

**触发点**：`TeamLeaveButton.tsx:13-16` → `TeamLeaveForm.tsx:21-29`

```typescript
// TeamLeaveForm.tsx
const handleConfirm = async () => {
  await mutateAsync(null, {
    onSuccess: async () => {
      touch('teams:members');
      onSave();
      onClose();
    },
  });
};
```

**完整链路（自删除）：**

```
1. 用户不是 teamOwner 且不是 Admin → 页面顶部显示 "Leave" 按钮
   ↓
2. 点击 "Leave" 按钮
   ↓
3. 打开确认对话框（TeamLeaveForm）
   ↓
4. 点击确认离开
   ↓
5. 调用 DELETE /api/teams/{teamId}/users/{userId}
   ↓
6. API 权限检查：canDeleteTeamUser
   └─ removeUserId === user.id → 通过（用户可删除自己，无需角色权限）
   ↓
7. API 成功返回
   ↓
8. 触发 onSuccess 回调
   ├─ touch('teams:members') → 更新成员列表
   ├─ touch('teams') → 更新团队列表
   ├─ 关闭对话框
   └─ 跳转到 /settings/teams 页面
```

> **重要发现**：用户自删除（离开团队）不需要任何团队角色权限，即便是 teamMember 或 teamViewOnly 角色的用户也可以随时通过独立的 "Leave" 按钮离开团队。

### 6.4 三类操作通用时序图

```
UI 操作      API 调用      成功回调     touch()    Zustand    React Query   重新渲染
  │            │             │           │          │            │            │
  ├───────────>│             │           │          │            │            │
  │            ├────────────>│           │          │            │            │
  │            │             ├──────────>│          │            │            │
  │            │             │           ├─────────>│            │            │
  │            │             │           │          ├────────────>│            │
  │            │             │           │          │            ├────────────>│
  │            │             │           │          │            │            │
  └<──────────────────────────────────────────────────────────────────────────┘
```

---

## 七、前端展示层与权限控制

### 7.1 数据获取 Hook (`src/components/hooks/queries/useTeamMembersQuery.ts`)

```typescript
export function useTeamMembersQuery(teamId: string) {
  const { get } = useApi();
  const { modified } = useModified(`teams:members`);

  return usePagedQuery({
    queryKey: ['teams:members', { teamId, modified }],
    queryFn: (params: any) => {
      return get(`/teams/${teamId}/users`, params);
    },
    enabled: !!teamId,
  });
}
```

### 7.2 分页查询封装 (`src/components/hooks/usePagedQuery.ts`)

```typescript
export function usePagedQuery<TData = any, TError = Error>({
  queryKey,
  queryFn,
  ...options
}) {
  const {
    query: { page, search },
  } = useNavigation();
  const { useQuery } = useApi();

  return useQuery<PageResult<TData>, TError>({
    queryKey: [...queryKey, page, search] as const,  // page 和 search 也影响查询键
    queryFn: () => queryFn({ page, search }),
    ...options,
  });
}
```

### 7.3 组件层级结构

```
TeamSettings
    ├── TeamLeaveButton (条件显示：非 Owner 且非 Admin)
    │   └── TeamLeaveForm
    ├── TeamMembersDataTable
    │   ├── DataGrid (搜索、分页包装)
    │   └── TeamMembersTable
    │       ├── DataTable (基础表格)
    │       ├── DataColumn (用户名)
    │       ├── DataColumn (角色)
    │       └── DataColumn (操作: 编辑 / 删除) [条件显示: allowEdit=true]
    │           ├── TeamMemberEditButton (不显示给 teamOwner)
    │           │   └── TeamMemberEditForm
    │           └── TeamMemberRemoveButton (不显示给 teamOwner)
    │               └── ConfirmationForm
    └── TeamsMemberAddButton (仅 Admin 可见)
```

### 7.4 团队页面权限判断 (`src/app/(main)/teams/[teamId]/TeamSettings.tsx:21-31`)

```typescript
// 判断是否为团队所有者
const isTeamOwner =
  !!team?.members?.find(
    ({ userId, role }) => role === ROLES.teamOwner && userId === user.id
  ) && user.role !== ROLES.viewOnly;

// 判断是否可编辑（决定操作按钮是否显示）
const canEdit =
  user.isAdmin ||
  (!!team?.members?.find(
    ({ userId, role }) =>
      (role === ROLES.teamOwner || role === ROLES.teamManager) && userId === user.id
  ) &&
  user.role !== ROLES.viewOnly);
```

**UI 权限判断总结：**
- `canEdit = true` → 显示编辑/删除按钮 → 需要是 Admin 或 teamOwner/teamManager
- `!isTeamOwner && !isAdmin` → 显示 "Leave" 按钮 → 普通成员可主动离开

### 7.5 TeamMembersDataTable (`src/app/(main)/teams/[teamId]/TeamMembersDataTable.tsx`)

```typescript
export function TeamMembersDataTable({
  teamId,
  allowEdit = false,
}: {
  teamId: string;
  allowEdit?: boolean;
}) {
  const queryResult = useTeamMembersQuery(teamId);

  return (
    <DataGrid query={queryResult} allowSearch>
      {({ data }) => <TeamMembersTable data={data} teamId={teamId} allowEdit={allowEdit} />}
    </DataGrid>
  );
}
```

### 7.6 TeamMembersTable 中的权限限制 (`src/app/(main)/teams/[teamId]/TeamMembersTable.tsx`)

```typescript
return (
  <DataTable data={data}>
    <DataColumn id="username" label={t(labels.username)}>
      {(row: any) => row?.user?.username}
    </DataColumn>
    <DataColumn id="role" label={t(labels.role)}>
      {(row: any) => roles[row?.role]}
    </DataColumn>
    {allowEdit && (                            // 外层权限：只有 canEdit=true 才显示操作列
      <DataColumn id="action" align="end">
        {(row: any) => {
          if (row?.role === ROLES.teamOwner) {  // 内层权限：对 teamOwner 不显示操作按钮
            return null;
          }

          return (
            <Row alignItems="center" maxHeight="20px">
              <TeamMemberEditButton teamId={teamId} userId={row?.user?.id} role={row?.role} />
              <TeamMemberRemoveButton
                teamId={teamId}
                userId={row?.user?.id}
                userName={row?.user?.username}
              />
            </Row>
          );
        }}
      </DataColumn>
    )}
  </DataTable>
);
```

**前端 UI 限制总结：**
1. **外层限制**：`allowEdit=false` → 不显示操作列 → 普通成员看不到任何编辑/删除按钮
2. **内层限制**：即便是 `allowEdit=true`，`teamOwner` 的行也不显示操作按钮
3. **自删除例外**：自删除不通过成员列表中的删除按钮，而是通过页面顶部的独立 "Leave" 按钮

---

## 八、前端限制与后端授权的差异分析

### 8.1 权限判定对照表

| 操作场景 | 前端 UI 是否显示按钮 | 后端 API 实际权限 | 一致性 |
|---------|---------------------|------------------|--------|
| **管理员删除任意成员** | ✗ 不在成员列表中显示（Admin 路由单独处理） | ✓ 允许（`user.isAdmin === true`） | 特殊情况 |
| **teamOwner 删除其他成员** | ✓ 显示（`canEdit=true` + 目标不是 owner） | ✓ 允许（有 `teamUpdate` 权限） | ✓ 一致 |
| **teamOwner 删除其他 owner** | ✗ 不显示（内层限制：跳过 role=teamOwner 的行） | ✓ API 允许（但无 UI 入口） | ✗ 不一致 |
| **teamManager 删除成员** | ✓ 显示 | ✓ 允许（有 `teamUpdate` 权限） | ✓ 一致 |
| **teamMember 删除他人** | ✗ 不显示（`allowEdit=false`） | ✗ API 拒绝 | ✓ 一致 |
| **用户删除自己（离开团队）** | ✗ 不通过成员列表按钮 | ✓ API 允许（自删除例外） | ✗ 不一致（通过独立 Leave 按钮） |

### 8.2 关键差异详解

#### 差异1：删除 teamOwner 的 UI 限制与 API 能力
- **前端限制**：TeamMembersTable 中 `row?.role === ROLES.teamOwner` 时返回 null → 不显示编辑/删除按钮
- **后端能力**：API 层面允许删除 teamOwner（只要调用者有 teamUpdate 权限，包括其他 owner）
- **影响**：无法通过 UI 删除团队所有者，但理论上可通过直接调用 API 实现
- **设计意图**：保护团队所有者不被误删除

#### 差异2：用户自删除的 UI 入口与权限
- **前端限制**：普通成员在成员列表中看不到 "Remove" 按钮（因为 `allowEdit=false`）
- **后端能力**：API 允许用户删除自己（无需任何角色权限）
- **处理方式**：通过页面顶部的独立 "Leave" 按钮实现，不与成员列表混在一起
- **设计意图**：提供清晰的语义化操作，"Leave" 比 "Remove" 更符合用户心理模型

#### 差异3：teamOwner 删除 teamOwner 的边界情况
- **场景**：团队有多个 owner（虽然 UI 不支持设置，但数据库可能存在）
- **前端行为**：内层限制跳过所有 teamOwner 行 → 任何 owner 都无法通过 UI 删除其他 owner
- **后端行为**：canDeleteTeamUser 只检查 teamUpdate 权限 → 理论上一个 owner 可以通过 API 删除另一个 owner
- **实际影响**：低风险，因为 UI 不支持创建多个 owner

### 8.3 结论与建议

**结论：**
1. 整体权限设计是一致且合理的
2. 前端做了更保守的限制（尤其是对 teamOwner 的保护）
3. 自删除机制通过独立 UI 入口实现，与成员管理操作在概念上分离
4. 边界情况（多个 owner 互删）在数据库层理论可行，但在 UI 层面被有效屏蔽

**设计合理性评价：**
- ✅ 权限分层合理：Admin > teamOwner > teamManager > teamMember > teamViewOnly
- ✅ 自删除机制正确：任何人都可以随时离开团队
- ✅ UI 与 API 差异主要是为了更好的用户体验和操作安全
- ✅ 没有严重的越权漏洞

**潜在改进点：**
- 后端可以增加额外保护：禁止删除 teamOwner 角色的成员（除非是自删）
- 后端可以限制：每个团队至少保留一个 teamOwner

---

## 九、完整数据链路总览

### 9.1 数据流向图

```
┌─────────────────────────┐
│       Database          │
│  ┌─ TeamUser            │
│  ├─ User                │
│  └─ Team                │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐     Prisma ORM
│  src/queries/prisma/    │
│    teamUser.ts          │
│  └─ getTeamUsers        │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐     Next.js API Route
│  /api/teams/[id]/users  │
│  ├─ GET: 获取列表        │
│  ├─ POST: 新增成员       │
│  └─ /[userId]           │
│     ├─ POST: 编辑角色    │
│     └─ DELETE: 移除成员  │
│           └─ canDeleteTeamUser: 3 分支权限判定
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐     Zustand Store
│    useModified.ts       │
│  ├─ touch(key)          │ ◄────────┐
│  └─ useModified(key)    │          │
└────────────┬────────────┘          │
             │                       │
             ▼                       │
┌─────────────────────────┐     React Query       │
│ useTeamMembersQuery.ts  │
│  └─ queryKey 包含 modified  ────────┘
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│    DataGrid 组件        │     搜索 + 分页
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  TeamMembersTable       │     表格渲染
│  ├─ 用户名              │
│  ├─ 角色                │
│  └─ 操作按钮 [canEdit]  │
│     ├─ Edit Button      │
│     └─ Remove Button    │
│       [跳过 teamOwner]  │
└─────────────────────────┘
┌─────────────────────────┐
│  TeamLeaveButton        │     自删除独立入口
│  └─ [!isTeamOwner]      │
└─────────────────────────┘
```

### 9.2 数据转换过程

**原始数据库记录 → API 返回 → 前端展示**

```
数据库: TeamUser
{
  id: "uuid",
  teamId: "uuid",
  userId: "uuid",
  role: "team-member",
  createdAt: Date,
  updatedAt: Date
}
    ↓ (include user)
API 返回:
{
  id: "uuid",
  teamId: "uuid",
  userId: "uuid",
  role: "team-member",
  createdAt: Date,
  updatedAt: Date,
  user: {
    id: "uuid",
    username: "john@example.com"
  }
}
    ↓ (前端映射 + canEdit 权限判断)
表格展示:
┌──────────────────────┬────────────┬───────────────────────┐
│ 用户名               │ 角色       │ 操作                  │
├──────────────────────┼────────────┼───────────────────────┤
│ owner@example.com    │ 团队所有者  │ (无按钮 - 受保护)     │
├──────────────────────┼────────────┼───────────────────────┤
│ manager@example.com  │ 团队管理员  │ [编辑] [删除]         │
├──────────────────────┼────────────┼───────────────────────┤
│ john@example.com     │ 团队成员    │ [编辑] [删除]         │
│                      │            │ (Owner/Manager 可见)  │
└──────────────────────┴────────────┴───────────────────────┘

页面顶部:
┌───────────────────────────────────────────────────────────┐
│ [Team Name]                                        [Leave] │
│                                         (非 Owner 用户可见)│
└───────────────────────────────────────────────────────────┘
```

---

## 十、关键设计决策总结

### 10.1 软删除处理

```typescript
where: {
  teamId,
  user: {
    deletedAt: null,  // 自动过滤已删除用户
  },
}
```

### 10.2 搜索实现

```typescript
prisma.getSearchParameters(search, [{ user: { username: 'contains' } }])
```

- 仅支持按用户名搜索
- 使用 `contains` 模糊匹配

### 10.3 排序策略

```typescript
orderBy: {
  createdAt: 'asc',  // 按加入时间升序，最早加入的排在前面
}
```

### 10.4 团队所有者保护

1. **API Schema 层**：`teamOwner` 不在 `teamRoleParam` 枚举中，无法通过 API 设置或修改
2. **UI 展示层**：`teamOwner` 的行不显示编辑/删除按钮
3. **权限层**：只有 Owner/Manager 才能执行修改操作

### 10.5 缓存刷新设计

**设计优点：**
- 声明式：组件只需要调用 `touch(key)`，无需关心具体刷新逻辑
- 集中式：所有同类查询共享同一个刷新触发点
- 可扩展：新增同类查询只需监听同一个 key
- 可靠：基于 React Query queryKey 变化自动刷新

**实现特点：**
- 使用时间戳 `Date.now()` 作为版本标记
- Zustand store 作为全局状态通信
- 时间戳作为 queryKey 一部分，变化自动触发重新查询

---

## 十一、相关文件清单

| 文件路径 | 功能说明 |
|---------|----------|
| `prisma/schema.prisma:216-230` | TeamUser 表定义 |
| `src/lib/constants.ts:164-217` | 角色与权限定义 |
| `src/lib/schema.ts:88` | 团队角色参数验证 |
| `src/lib/auth.ts:76-78` | hasPermission 权限检查函数 |
| `src/queries/prisma/teamUser.ts` | 团队成员数据库查询 |
| `src/permissions/team.ts` | 团队权限判定函数 |
| `src/app/api/teams/[teamId]/users/route.ts` | 成员列表/新增 API |
| `src/app/api/teams/[teamId]/users/[userId]/route.ts` | 成员编辑/删除 API |
| `src/components/hooks/useModified.ts` | Zustand 缓存刷新机制 |
| `src/components/hooks/queries/useTeamMembersQuery.ts` | 前端数据获取 Hook |
| `src/components/hooks/usePagedQuery.ts` | 分页查询封装 |
| `src/app/(main)/teams/[teamId]/TeamMembersDataTable.tsx` | 成员表格容器 |
| `src/app/(main)/teams/[teamId]/TeamMembersTable.tsx` | 成员表格渲染（含 UI 权限限制） |
| `src/app/(main)/teams/TeamsMemberAddButton.tsx` | 新增成员按钮 |
| `src/app/(main)/teams/TeamMemberAddForm.tsx` | 新增成员表单 |
| `src/app/(main)/teams/[teamId]/TeamMemberEditButton.tsx` | 编辑成员按钮 |
| `src/app/(main)/teams/[teamId]/TeamMemberEditForm.tsx` | 编辑成员表单 |
| `src/app/(main)/teams/[teamId]/TeamMemberRemoveButton.tsx` | 删除成员按钮 |
| `src/app/(main)/teams/TeamLeaveButton.tsx` | 离开团队按钮（自删除入口） |
| `src/app/(main)/teams/TeamLeaveForm.tsx` | 离开团队表单 |
| `src/app/(main)/teams/[teamId]/TeamSettings.tsx` | 团队设置页面（权限判断） |
