# Umami 网站权限系统协作分析（最终统一版）

## 文档信息
- **版本**：v1.2
- **日期**：2026-05-14
- **范围**：用户-团队-网站三层权限架构
- **验证级别**：代码级全量核查

### v1.2 统一修订说明
本版本对报告进行了全量一致性核查，重点修正：
1. ✅ 统一 TeamUser 删除语义表述：物理删除，无软删除机制
2. ✅ 删除所有与"物理删除后记录不存在"冲突的说法
3. ✅ 重新分类 Team.deletedAt 联动过滤风险，单独归类
4. ✅ 创建统一的结论清单与证据映射表，确保可追溯性

---

## 一、数据模型层事实核对

### 1.1 Prisma Schema 实体关系模型

#### Website 模型字段定义

**文件位置**：`prisma/schema.prisma:66-96`

```prisma
model Website {
  id        String    @id() @map("website_id") @db.Uuid
  name      String    @db.VarChar(100)
  domain    String?   @db.VarChar(500)
  userId    String?   @map("user_id") @db.Uuid      // 个人所有者ID，可空
  teamId    String?   @map("team_id") @db.Uuid      // 团队所有者ID，可空
  createdBy String?   @map("created_by") @db.Uuid
  createdAt DateTime? @default(now())
  deletedAt DateTime? @map("deleted_at")            // ✅ 支持软删除

  // 索引定义
  @@index([userId])      // 单列索引：userId
  @@index([teamId])      // 单列索引：teamId
}
```

#### Team 模型字段定义

**文件位置**：`prisma/schema.prisma:197-215`

```prisma
model Team {
  id          String     @id() @map("team_id") @db.Uuid
  name        String     @db.VarChar(100)
  companyName String?    @map("company_name") @db.VarChar(100)
  websiteId   String?    @map("website_id") @db.Uuid
  createdAt   DateTime?  @default(now()) @db.Timestamptz(6)
  updatedAt   DateTime?  @updatedAt @map("updated_at") @db.Timestamptz(6)
  deletedAt   DateTime?  @map("deleted_at")              // ✅ 支持软删除

  websites    Website[]
  members     TeamUser[]
}
```

#### TeamUser 模型与索引（关键核查点）

**文件位置**：`prisma/schema.prisma:216-230`

```prisma
model TeamUser {
  id        String    @id() @map("team_user_id") @db.Uuid
  teamId    String    @map("team_id") @db.Uuid
  userId    String    @map("user_id") @db.Uuid
  role      String    @db.VarChar(50)      // 团队内角色
  createdAt DateTime? @default(now()) @db.Timestamptz(6)
  updatedAt DateTime? @updatedAt @map("updated_at") @db.Timestamptz(6)

  // ❌ 无 deletedAt 字段 → 不支持软删除

  // 索引定义
  @@index([teamId])    // 单列索引：teamId
  @@index([userId])    // 单列索引：userId
}
```

**模型层统一结论**：
| 实体 | 软删除支持 | 删除语义 |
|------|-----------|---------|
| Website | ✅ 有 deletedAt | 支持软删除 |
| Team | ✅ 有 deletedAt | 支持软删除（Cloud模式）/物理删除 |
| TeamUser | ❌ 无 deletedAt | **仅支持物理删除** |

---

### 1.2 角色枚举值与权限定义

**文件位置**：`src/lib/constants.ts:164-217`

```typescript
// 角色枚举值（连字符格式）
export const ROLES = {
  admin: 'admin',
  user: 'user',
  viewOnly: 'view-only',
  teamOwner: 'team-owner',           // 连字符格式
  teamManager: 'team-manager',        // 连字符格式
  teamMember: 'team-member',          // 连字符格式
  teamViewOnly: 'team-view-only',     // 连字符格式
} as const;

// 权限定义
export const PERMISSIONS = {
  all: 'all',
  websiteCreate: 'website:create',
  websiteUpdate: 'website:update',
  websiteDelete: 'website:delete',
  websiteTransferToTeam: 'website:transfer-to-team',
  websiteTransferToUser: 'website:transfer-to-user',
  teamCreate: 'team:create',
  teamUpdate: 'team:update',
  teamDelete: 'team:delete',
} as const;
```

---

## 二、归属约束来源核对

### 2.1 userId/teamId 互斥约束

**数据库层面**：Prisma Schema 中无 CHECK 约束 → 数据库层面允许同时赋值

**业务约定层面**：
```typescript
// 创建时：二选一
if (!teamId) {
  data.userId = auth.user.id;  // 仅当 teamId 不存在时设置 userId
}

// 转移时：显式将另一方设为 null
if (userId) {
  data.userId = userId;
  data.teamId = null;
} else if (teamId) {
  data.teamId = teamId;
  data.userId = null;
}
```

---

## 三、权限判断路径证据化

### 3.1 canViewWebsite 权限判断路径

**文件位置**：`src/permissions/website.ts:7-40`

```typescript
export async function canViewWebsite({ user, shareToken }: Auth, websiteId: string) {
  if (user?.isAdmin) return true;                // 管理员快速路径

  // ShareToken 6种匹配方式，命中即放行
  if (shareTokenMatch()) return true;

  const entity = await getEntity(websiteId);     // 并行4表查询
  if (!entity || !user) return false;

  if (entity.userId) {
    return user.id === entity.userId;            // 个人网站匹配
  }
  if (entity.teamId) {
    const teamUser = await getTeamUser(entity.teamId, user.id);
    return !!teamUser;                            // 仅需成员存在，无需权限
  }
  return false;
}
```

### 3.2 canUpdateWebsite / canDeleteWebsite 权限判断路径

**文件位置**：`src/permissions/website.ts:58-112`

```typescript
export async function canUpdateWebsite({ user }: Auth, websiteId: string) {
  if (!user) return false;
  if (user.isAdmin) return true;

  const website = await getWebsite(websiteId);   // 注意：使用 getWebsite 而非 getEntity
  if (!website) return false;

  if (website.userId) {
    return user.id === website.userId;
  }
  if (website.teamId) {
    const teamUser = await getTeamUser(website.teamId, user.id);
    // 需同时满足：成员存在 + 角色有权限
    return teamUser && hasPermission(teamUser.role, PERMISSIONS.websiteUpdate);
  }
  return false;
}
```

**权限路径统一结论**：
| 权限函数 | 网站实体查询 | 团队网站条件 |
|---------|-------------|-------------|
| `canViewWebsite` | `getEntity()`（4表并行） | 仅需 `getTeamUser()` 存在 |
| `canUpdateWebsite` | `getWebsite()`（仅Website表） | `getTeamUser()` 存在 + 角色权限 |
| `canDeleteWebsite` | `getWebsite()`（仅Website表） | `getTeamUser()` 存在 + 角色权限 |

---

## 四、SQL 查询封装证据化（统一修订版）

### 4.1 getTeamUser 查询与删除语义

**文件位置**：`src/queries/prisma/teamUser.ts:12-19, 59-65`

```typescript
// 查询函数
export async function getTeamUser(teamId: string, userId: string) {
  return prisma.client.teamUser.findFirst({
    where: { teamId, userId },
    // TeamUser 模型本身无 deletedAt 字段 → 无需过滤
  });
}

// 删除函数
export async function deleteTeamUser(teamId: string, userId: string) {
  return prisma.client.teamUser.deleteMany({
    where: { teamId, userId },
    // 物理删除：记录从数据库直接移除
  });
}
```

**TeamUser 删除语义统一结论**：
✅ **模型层**：无 deletedAt 字段 → 不支持软删除
✅ **实现层**：使用 `deleteMany` → 物理删除
✅ **影响层**：记录被物理删除 → `getTeamUser()` 返回 null → 权限检查自动失效
✅ **一致性**：不存在"已删除但仍可被查询到"的情况

**调用路径证据链**：
1. API 调用：`DELETE /api/teams/[teamId]/users/[userId]`
2. 业务逻辑：调用 `deleteTeamUser(teamId, userId)`
3. 数据库操作：物理删除 TeamUser 记录
4. 后续权限检查：`getTeamUser()` 返回 null → 团队网站权限失效

---

### 4.2 getWebsite 查询实现

**文件位置**：`src/queries/prisma/website.ts:11-23`

```typescript
export async function getWebsite(websiteId: string) {
  const website = await findWebsite({
    where: {
      id: websiteId,
      // ❌ 无 website.deletedAt 过滤
    },
  });
  return website ? attachShareIdToWebsite(website) : null;
}
```

**Website 删除语义结论**：
✅ 列表查询 `getWebsites()` 有 `deletedAt: null` 过滤
❌ 单条查询 `getWebsite()` 无软删除过滤
❌ 权限检查调用 `getWebsite()` → 已删除网站仍参与权限判断

---

### 4.3 Team.deletedAt 联动过滤风险（单独归类）

#### 风险定义：团队软删除后权限残留风险

#### 证据链汇总：

**证据1：列表查询有过滤**
```typescript
// getAllUserWebsitesIncludingTeamOwner
where: {
  team: {
    deletedAt: null,  // ✅ 列表查询过滤已删除团队
    members: { some: { role: ROLES.teamOwner, userId } }
  }
}
```

**证据2：权限检查无过滤**
```typescript
// canViewWebsite / canUpdateWebsite / canDeleteWebsite
const entity = await getEntity(websiteId);  // ❌ 无 team.deletedAt 过滤
if (entity.teamId) {
  const teamUser = await getTeamUser(entity.teamId, user.id);  // ❌ 仅检查成员
  return !!teamUser;  // TeamUser 存在即通过，不检查 Team 是否已删除
}
```

**证据3：Cloud模式下Team删除实现**
```typescript
// Cloud 模式：软删除 Team，但保留 TeamUser 记录
if (isCloud) {
  await deleteTeam(req, team.id);  // 软删除：设置 team.deletedAt
  // TeamUser 记录保留，不删除
} else {
  await prisma.team.delete({ where: { id: team.id } });  // 物理删除 + 级联
}
```

#### 风险场景矩阵：

| 模式 | Team 删除方式 | TeamUser 状态 | 权限检查结果 |
|------|-------------|--------------|-------------|
| Cloud 模式 | 软删除（设置 deletedAt） | 记录保留 | ✅ TeamUser 存在 → 权限检查通过 → 仍可访问 |
| 非 Cloud 模式 | 物理删除 | 级联删除 | ❌ TeamUser 不存在 → 权限检查失败 |

#### Team.deletedAt 过滤覆盖范围表：

| 查询场景 | 文件位置 | team.deletedAt 过滤 |
|---------|---------|---------------------|
| `getAllUserWebsitesIncludingTeamOwner`（列表） | `website.ts:45-69` | ✅ 有 |
| `getTeamUsers`（列表） | `teamUser.ts:21-37` | ✅ 有 |
| `getUserTeams`（列表） | `team.ts:49-85` | ✅ 有 |
| `canViewWebsite`（权限检查） | `website.ts:23-37` | ❌ 无 |
| `canUpdateWebsite`（权限检查） | `website.ts:77-81` | ❌ 无 |
| `canDeleteWebsite`（权限检查） | `website.ts:95-109` | ❌ 无 |
| `canViewTeam`（权限检查） | `team.ts:6-16` | ❌ 无 |
| `getTeam`（单条查询） | `team.ts:13-25` | ❌ 无 |

**Team.deletedAt 风险统一结论**：
⚠️ **列表-单条不一致风险**：团队被软删除后，用户在列表中看不到该团队及其网站，但通过直接调用 API 仍可访问到团队下的网站（Cloud模式）

---

## 五、边界条件详细核对表（统一修订版）

| 边界条件 | 代码位置 | 实际行为 | 验证结论 |
|---------|---------|---------|---------|
| **website 不存在** | `website.ts:69-71` | `getWebsite` 返回 null → 权限返回 false | ✅ 正确 |
| **teamUser 记录不存在** | `website.ts:77-80` | `getTeamUser` 返回 null → 权限返回 false | ✅ 正确 |
| **auth.user 为 null** | `website.ts:59` | `!user` → 返回 false | ✅ 正确 |
| **entity 为 null** | `website.ts:25-27` | `!entity` → 返回 false | ✅ 正确 |
| **website 同时有 userId 和 teamId** | N/A | 业务代码无防御，按代码顺序 userId 分支优先 | ⚠️ 防御缺失 |
| **已删除 website（单条查询）** | `website.ts:11-16` | 可被查询到，权限判断正常执行 | ⚠️ 无过滤 |
| **已删除 website（列表查询）** | `website.ts:37` | 被 `deletedAt: null` 过滤，不可见 | ✅ 有过滤 |
| **移除团队成员（deleteTeamUser）** | `teamUser.ts:59-65` | TeamUser 物理删除 → 记录不存在 → 权限自动失效 | ✅ 正确 |
| **Cloud 模式删除 Team** | `team.ts:143-158` | Team 软删除 → TeamUser 记录保留 → 成员仍可通过权限检查 | ⚠️ 权限残留风险 |
| **非 Cloud 模式删除 Team** | `team.ts:160-171` | Team 物理删除 + TeamUser 级联删除 → 权限自动失效 | ✅ 正确 |
| **shareToken 查看网站** | `website.ts:12-21` | 绕过归属检查，直接返回 true | ✅ 按设计 |
| **teamViewOnly 查看网站** | `website.ts:33-37` | 仅需 teamUser 存在即可 → 可查看 | ✅ 按设计 |
| **teamViewOnly 更新网站** | `website.ts:77-81` | `hasPermission('team-view-only', 'website:update')` → false | ✅ 正确 |

---

## 六、统一结论清单与证据映射

### 6.1 删除语义统一结论（已验证）

| 结论 ID | 结论内容 | 证据文件位置 | 验证状态 |
|---------|---------|-------------|---------|
| D-001 | TeamUser 模型无 deletedAt 字段，不支持软删除 | `prisma/schema.prisma:216-230` | ✅ 已验证 |
| D-002 | deleteTeamUser 使用 deleteMany 物理删除记录 | `src/queries/prisma/teamUser.ts:59-65` | ✅ 已验证 |
| D-003 | TeamUser 被删除后，getTeamUser 返回 null，权限自动失效 | `src/permissions/website.ts:33-37, 77-81` | ✅ 已验证 |
| D-004 | Website 单条查询无 deletedAt 过滤，已删除网站可被查询 | `src/queries/prisma/website.ts:11-23` | ✅ 已验证 |
| D-005 | Website 列表查询有 deletedAt 过滤 | `src/queries/prisma/website.ts:25-38` | ✅ 已验证 |
| D-006 | Cloud 模式下 Team 软删除，TeamUser 记录保留 | `src/app/api/teams/[teamId]/route.ts:85-100` | ✅ 已验证 |
| D-007 | 非 Cloud 模式下 Team 物理删除，TeamUser 级联删除 | `src/app/api/teams/[teamId]/route.ts:101-110` | ✅ 已验证 |

### 6.2 权限检查统一结论（已验证）

| 结论 ID | 结论内容 | 证据文件位置 | 验证状态 |
|---------|---------|-------------|---------|
| P-001 | canViewWebsite 中，团队成员仅需 TeamUser 存在即可，无需权限检查 | `src/permissions/website.ts:33-37` | ✅ 已验证 |
| P-002 | canUpdateWebsite / canDeleteWebsite 需要 TeamUser 存在 + 角色权限 | `src/permissions/website.ts:77-81, 95-109` | ✅ 已验证 |
| P-003 | 所有权限检查函数均不检查 team.deletedAt 状态 | `src/permissions/website.ts:23-112` | ✅ 已验证 |
| P-004 | ShareToken 有 6 种匹配方式，命中即绕过归属检查 | `src/permissions/website.ts:12-21` | ✅ 已验证 |
| P-005 | getEntity 并行查询 Website/Link/Pixel/Board 4张表 | `src/lib/entity.ts:5-10` | ✅ 已验证 |
| P-006 | isAdmin 通过 `user.role === 'admin'` 硬编码判断，不调用 hasPermission | `src/lib/auth.ts:51` | ✅ 已验证 |

### 6.3 风险点统一清单（已分类）

| 风险 ID | 风险分类 | 风险描述 | 影响范围 | 严重程度 |
|---------|---------|---------|---------|---------|
| R-001 | Team.deletedAt 联动过滤缺失 | Cloud 模式下团队软删除后，成员仍可访问该团队下的网站 | 团队网站访问 | 中 |
| R-002 | Website 软删除不一致 | 已删除网站在单条权限检查中仍可被访问到（列表中不可见） | 单个网站访问 | 低 |
| R-003 | 归属互斥无数据库约束 | 无 CHECK 约束保证 userId/teamId 互斥，脏数据可能导致权限判断不一致 | 网站归属 | 低 |
| R-004 | TeamUser 无联合索引 | `findFirst({ teamId, userId })` 使用单列索引，高并发下可能存在性能隐患 | 权限查询性能 | 低 |

---

## 七、调用链与依赖关系

### 7.1 完整权限检查调用链

```
HTTP Request
    │
    ▼
┌───────────────────────────────────────────┐
│ parseRequest(request, schema)              │
│  - Zod 参数校验                             │
│  - 调用 checkAuth() 认证                     │
│  - 返回 { auth, body, query, error }        │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────┐
│ checkAuth()                                │
│  - 解析 Bearer Token / ShareToken          │
│  - user.isAdmin = (role === 'admin')       │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────┐
│ 权限检查函数 canXxxWebsite()               │
│  - admin 快速路径                           │
│  - getWebsite() / getEntity()              │
│  - getTeamUser(teamId, userId) ← 物理删除后返回 null │
│  - hasPermission(role, permission)          │
└─────────────────────────────────────────────┘
```

### 7.2 模块依赖关系

```
src/permissions/*.ts
       │
       ├─► src/lib/auth.ts (hasPermission, checkAuth)
       ├─► src/lib/constants.ts (ROLES, PERMISSIONS)
       ├─► src/lib/entity.ts (getEntity)
       └─► src/queries/prisma/*.ts
              ├─► getWebsite / getWebsites
              ├─► getTeamUser / deleteTeamUser  ← 物理删除
              └─► getTeam
```

---

## 八、最终核对总结

### 8.1 已确认的正确结论

1. ✅ TeamUser 采用物理删除，无软删除机制 → 删除后权限立即失效
2. ✅ 团队角色枚举值使用连字符格式（`team-owner` 等）
3. ✅ TeamUser 仅有 `[teamId]` 和 `[userId]` 两个单列索引，无联合索引
4. ✅ userId/teamId 互斥是业务代码约定，非数据库级约束
5. ✅ 查看团队网站仅需 TeamUser 存在，无需权限检查
6. ✅ 更新/删除团队网站需 TeamUser 存在 + 角色权限
7. ✅ getEntity 并行查询 4 张表
8. ✅ ShareToken 有 6 种匹配方式，绕过归属检查
9. ✅ isAdmin 是认证阶段派生的硬编码判断
10. ✅ 列表查询均有对应的 deletedAt 过滤

### 8.2 最终风险清单（3类风险）

| 风险类型 | 风险描述 | 触发条件 |
|---------|---------|---------|
| **Team 软删除权限残留** | Cloud 模式下团队软删除后，成员仍可访问团队网站 | Team.deletedAt 非空 + TeamUser 记录存在 |
| **Website 软删除不一致** | 已删除网站在列表中不可见，但直接访问 API 仍可访问 | Website.deletedAt 非空 + 知道 websiteId |
| **性能隐患** | TeamUser 无联合索引，高并发下权限查询可能存在性能问题 | 大量团队成员同时访问 |

---

**文档修订完成标记**：✅ 所有删除语义表述已统一 ✅ 冲突说法已删除 ✅ Team.deletedAt 风险已单独归类 ✅ 可追溯的结论清单与证据映射已创建
