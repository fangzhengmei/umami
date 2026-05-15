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
- `createdAt`: 创建时间
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

### 2.2 团队角色权限 (`src/lib/constants.ts:195-216`)

```typescript
export const ROLE_PERMISSIONS = {
  [ROLES.teamOwner]: [
    PERMISSIONS.teamUpdate,
    PERMISSIONS.teamDelete,
    PERMISSIONS.websiteCreate,
    PERMISSIONS.websiteUpdate,
    PERMISSIONS.websiteDelete,
    PERMISSIONS.websiteTransferToTeam,
    PERMISSIONS.websiteTransferToUser,
  ],
  [ROLES.teamManager]: [
    PERMISSIONS.teamUpdate,
    PERMISSIONS.websiteCreate,
    PERMISSIONS.websiteUpdate,
    PERMISSIONS.websiteDelete,
    PERMISSIONS.websiteTransferToTeam,
  ],
  [ROLES.teamMember]: [
    PERMISSIONS.websiteCreate,
    PERMISSIONS.websiteUpdate,
    PERMISSIONS.websiteDelete,
  ],
  [ROLES.teamViewOnly]: [],
} as const;
```

### 2.3 Schema 验证 (`src/lib/schema.ts:88`)

```typescript
export const teamRoleParam = z.enum(['team-member', 'team-view-only', 'team-manager']);
```

> **注意**: TeamOwner 角色不能通过 API 直接设置，它是团队创建时默认分配的角色。

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

---

## 五、前端数据获取层

### 5.1 React Query Hook (`src/components/hooks/queries/useTeamMembersQuery.ts`)

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

**特性说明：**
- 使用 `usePagedQuery` 封装分页查询
- `modified` 标记用于缓存失效，当成员列表变更时自动刷新
- `enabled` 控制查询启用时机

### 5.2 数据修改触发机制

通过 `useModified` Hook 实现缓存刷新：

```typescript
// 当添加/编辑/删除成员后，调用:
modified();  // 触发 'teams:members' 缓存失效
```

---

## 六、前端展示层

### 6.1 组件层级结构

```
TeamSettings
    └── TeamMembersDataTable
          ├── DataGrid (搜索、分页包装)
          └── TeamMembersTable
                ├── DataTable (基础表格)
                ├── DataColumn (用户名)
                ├── DataColumn (角色)
                └── DataColumn (操作: 编辑 / 删除)
```

### 6.2 TeamMembersDataTable (`src/app/(main)/teams/[teamId]/TeamMembersDataTable.tsx`)

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

**Props 说明：**
- `teamId`: 团队 ID
- `allowEdit`: 是否允许编辑操作

### 6.3 TeamMembersTable (`src/app/(main)/teams/[teamId]/TeamMembersTable.tsx`)

#### 角色本地化映射

```typescript
const roles = {
  [ROLES.teamOwner]: t(labels.teamOwner),
  [ROLES.teamManager]: t(labels.teamManager),
  [ROLES.teamMember]: t(labels.teamMember),
  [ROLES.teamViewOnly]: t(labels.viewOnly),
};
```

#### 表格列定义

1. **用户名列** - 显示 `row.user.username`
2. **角色列** - 显示本地化后的角色名称
3. **操作列** (仅 `allowEdit=true` 时显示):
   - TeamOwner 不显示操作按钮
   - 其他角色显示：编辑按钮 + 删除按钮

#### 权限控制逻辑

```typescript
if (row?.role === ROLES.teamOwner) {
  return null;  // 团队所有者不能被编辑/删除
}
```

### 6.4 权限判断逻辑 (`src/app/(main)/teams/[teamId]/TeamSettings.tsx:21-31`)

```typescript
// 判断是否为团队所有者
const isTeamOwner =
  !!team?.members?.find(
    ({ userId, role }) => role === ROLES.teamOwner && userId === user.id
  ) && user.role !== ROLES.viewOnly;

// 判断是否可编辑
const canEdit =
  user.isAdmin ||
  (!!team?.members?.find(
    ({ userId, role }) =>
      (role === ROLES.teamOwner || role === ROLES.teamManager) && userId === user.id
  ) && user.role !== ROLES.viewOnly);
```

---

## 七、完整数据链路

### 7.1 数据流向图

```
┌─────────────────┐
│  Database       │
│  ├─ TeamUser    │
│  ├─ User        │
│  └─ Team        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     Prisma ORM
│  teamUser.ts    │
│  └─ getTeamUsers│
└────────┬────────┘
         │
         ▼
┌─────────────────┐     Next.js API Route
│  route.ts (GET) │
│  ├─ 权限校验    │
│  ├─ 参数校验    │
│  └─ 查询数据    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     React Query
│ useTeamMembers  │
│  └─ 缓存管理    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   DataGrid      │     搜索 + 分页
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ TeamMembersTable│     表格渲染 + 操作按钮
└─────────────────┘
```

### 7.2 数据转换过程

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
    ↓ (前端映射)
表格展示:
┌──────────────────────┬────────────┬──────────┐
│ 用户名               │ 角色       │ 操作     │
├──────────────────────┼────────────┼──────────┤
│ john@example.com     │ 团队成员   │ 编辑/删除│
└──────────────────────┴────────────┴──────────┘
```

---

## 八、关键设计决策

### 8.1 软删除处理

```typescript
where: {
  teamId,
  user: {
    deletedAt: null,  // 自动过滤已删除用户
  },
}
```

### 8.2 搜索实现

```typescript
prisma.getSearchParameters(search, [{ user: { username: 'contains' } }])
```

- 仅支持按用户名搜索
- 使用 `contains` 模糊匹配

### 8.3 排序策略

```typescript
orderBy: {
  createdAt: 'asc',  // 按加入时间升序，最早加入的排在前面
}
```

### 8.4 团队所有者保护

1. **API 层**: TeamOwner 不在 `teamRoleParam` 枚举中，无法通过 API 设置
2. **UI 层**: TeamOwner 的行不显示编辑/删除按钮

---

## 九、相关文件清单

| 文件路径 | 功能说明 |
|---------|----------|
| `prisma/schema.prisma:216-230` | TeamUser 表定义 |
| `src/lib/constants.ts:164-217` | 角色与权限定义 |
| `src/lib/schema.ts:88` | 团队角色参数验证 |
| `src/queries/prisma/teamUser.ts` | 团队成员数据库查询 |
| `src/app/api/teams/[teamId]/users/route.ts` | 团队成员 API 接口 |
| `src/components/hooks/queries/useTeamMembersQuery.ts` | 前端数据获取 Hook |
| `src/app/(main)/teams/[teamId]/TeamMembersDataTable.tsx` | 成员表格容器 |
| `src/app/(main)/teams/[teamId]/TeamMembersTable.tsx` | 成员表格渲染 |
| `src/app/(main)/teams/[teamId]/TeamSettings.tsx` | 团队设置页面 |
