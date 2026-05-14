# Umami 网站权限系统协作分析（证据化核对版）

## 文档版本：v1.0
核对日期：2026-05-14
核对范围：用户-团队-网站三层权限架构

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
  deletedAt DateTime? @map("deleted_at")

  // 索引定义
  @@index([userId])      // 单列索引：userId
  @@index([teamId])      // 单列索引：teamId
  @@index([createdAt])
  @@index([createdBy])
}
```

**核对结论**：
✅ Website 模型同时存在 userId 和 teamId 两个归属字段，均为可空（String?）。

#### TeamUser 模型与索引

**文件位置**：`prisma/schema.prisma:216-230`

```prisma
model TeamUser {
  id        String    @id() @map("team_user_id") @db.Uuid
  teamId    String    @map("team_id") @db.Uuid
  userId    String    @map("user_id") @db.Uuid
  role      String    @db.VarChar(50)      // 团队内角色

  // 索引定义
  @@index([teamId])    // 单列索引：teamId
  @@index([userId])    // 单列索引：userId
}
```

**核对结论**：
✅ TeamUser 模型仅有**两个单列索引**，分别是 `[teamId]` 和 `[userId]`，**不存在联合索引** `[teamId, userId]`。
⚠️ **重要事实**：权限查询时使用的是 `findFirst({ where: { teamId, userId } })，依赖数据库查询优化器，而非显式联合索引。

### 1.2 角色枚举值核对

**文件位置**：`src/lib/constants.ts:164-172`

```typescript
export const ROLES = {
  admin: 'admin',                    // 实际值：'admin'
  user: 'user',                      // 实际值：'user'
  viewOnly: 'view-only',              // 实际值：'view-only'（含连字符
  teamOwner: 'team-owner',           // 实际值：'team-owner'（含连字符
  teamManager: 'team-manager',         // 实际值：'team-manager'（含连字符
  teamMember: 'team-member',         // 实际值：'team-member'（含连字符
  teamViewOnly: 'team-view-only',    // 实际值：'team-view-only'（含连字符
} as const;
```

**核对结论**：
✅ 所有团队角色枚举值全部使用**连字符（hyphen**，而非下划线或驼峰。

### 1.3 权限定义核对

**文件位置**：`src/lib/constants.ts:174-184`

```typescript
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

**核对结论**：
✅ 共定义了 9 种权限，使用冒号 `:` 作为分隔符。

### 1.4 角色权限映射表核对

**文件位置**：`src/lib/constants.ts:186-217`

```typescript
export const ROLE_PERMISSIONS = {
  [ROLES.admin]: [PERMISSIONS.all],
  [ROLES.user]: [
    PERMISSIONS.websiteCreate,
    PERMISSIONS.websiteUpdate,
    PERMISSIONS.websiteDelete,
    PERMISSIONS.teamCreate,
  ],
  [ROLES.viewOnly]: [],
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

**核对结论**：
✅ `teamOwner 拥有 7 种权限
✅ teamManager 拥有 5 种权限（无 teamDelete、websiteTransferToUser）
✅ teamMember 拥有 3 种权限（仅网站增删改）
✅ teamViewOnly 无任何权限

---

## 二、归属约束来源核对

### 2.1 userId/teamId 互斥约束

#### 数据库层面

**文件位置**：`prisma/schema.prisma:66-96`

**核对结论**：
❌ Prisma Schema 中**不存在数据库级别的 CHECK 约束**来强制 userId 和 teamId 的互斥性。
两个字段都只是独立的可空字段，数据库层面允许同时赋值。

#### 业务约定层面

**证据1：网站创建逻辑**
**文件位置**：`src/app/api/websites/route.ts:71-81`

```typescript
const data: any = {
  id: id ?? uuid(),
  createdBy: auth.user.id,
  name,
  domain,
  teamId,
};

if (!teamId) {
  data.userId = auth.user.id;  // 仅当 teamId 不存在时设置 userId
}
```

**证据2：网站转移逻辑**
**文件位置**：`src/app/api/websites/[websiteId]/transfer/route.ts:25-46`

```typescript
if (userId) {
  // 转移给用户时，显式设置 teamId 为 null
  const website = await updateWebsite(websiteId, {
    userId,
    teamId: null,
  });
} else if (teamId) {
  // 转移给团队时，显式设置 userId 为 null
  const website = await updateWebsite(websiteId, {
    userId: null,
    teamId,
  });
}
```

**证据3：权限判断逻辑**
**文件位置**：`src/permissions/website.ts:29-37, 73-81`

```typescript
// canViewWebsite 逻辑：
if (entity.userId) {
  return user.id === entity.userId;  // 有 userId 仅按用户判断
}
if (entity.teamId) {
  const teamUser = await getTeamUser(entity.teamId, user.id);
  return !!teamUser;                 // 有 teamId 按团队成员判断
}
```

**核对结论**：
✅ **互斥约束是业务代码约定，非数据库强制约束
✅ 创建时：teamId 存在 → 不设置 userId；teamId 不存在 → 设置 userId 为当前用户
✅ 转移时：显式将另一方设为 null
✅ 权限判断时：按 if-else 分支，仅判断存在的字段

---

## 三、权限判断路径证据化

### 3.1 canViewWebsite 权限判断路径

**文件位置**：`src/permissions/website.ts:7-40`

```typescript
export async function canViewWebsite({ user, shareToken }: Auth, websiteId: string) {
  // 路径1：管理员直接放行
  if (user?.isAdmin) {
    return true;
  }

  // 路径2：共享令牌访问（6种匹配方式）
  if (
    shareToken?.websiteId === websiteId ||
    shareToken?.pixelId === websiteId ||
    shareToken?.linkId === websiteId ||
    shareToken?.websiteIds?.includes(websiteId) ||
    shareToken?.pixelIds?.includes(websiteId) ||
    shareToken?.linkIds?.includes(websiteId)
  ) {
    return true;
  }

  // 路径3：通过 getEntity 查询实体
  const entity = await getEntity(websiteId);

  // 边界条件：实体不存在 或 用户未登录
  if (!entity || !user) {
    return false;
  }

  // 路径4：用户个人网站匹配
  if (entity.userId) {
    return user.id === entity.userId;
  }

  // 路径5：团队网站，查询 TeamUser
  if (entity.teamId) {
    const teamUser = await getTeamUser(entity.teamId, user.id);
    return !!teamUser;  // 仅需存在，无需权限检查
  }

  // 路径6：其他情况（无归属）
  return false;
}
```

**调用的查询封装**：
- `getEntity(websiteId)` → 并行4次查询（见 3.4 节
- `getTeamUser(teamId, userId)` → TeamUser 存在性检查

**边界条件证据**：
✅ `!entity || !user` → 返回 false
✅ 团队网站查看**不检查角色权限**，仅需是团队成员

### 3.2 canUpdateWebsite 权限判断路径

**文件位置**：`src/permissions/website.ts:58-84`

```typescript
export async function canUpdateWebsite({ user }: Auth, websiteId: string) {
  // 路径1：管理员直接放行
  if (!user) return false;
  if (user.isAdmin) return true;

  // 路径2：查询网站实体（注意：此处用 getWebsite 而非 getEntity）
  const website = await getWebsite(websiteId);
  if (!website) return false;

  // 路径3：个人网站所有者
  if (website.userId) {
    return user.id === website.userId;
  }

  // 路径4：团队网站
  if (website.teamId) {
    const teamUser = await getTeamUser(website.teamId, user.id);
    // 需同时满足：团队成员存在 且 角色拥有 website:update 权限
    return teamUser && hasPermission(teamUser.role, PERMISSIONS.websiteUpdate);
  }

  return false;
}
```

**关键差异证据**：
⚠️ 与 canViewWebsite 不同：
1. 使用 `getWebsite()` 而非 `getEntity()` → 仅查询 Website 表
2. 团队网站需要 `hasPermission()` 权限校验

### 3.3 canDeleteWebsite 权限判断路径

**文件位置**：`src/permissions/website.ts:86-112`

代码结构与 canUpdateWebsite 完全一致，仅权限校验改为 `PERMISSIONS.websiteDelete`。

### 3.4 getEntity 多态实体查询

**文件位置**：`src/lib/entity.ts:4-14`

```typescript
export async function getEntity(entityId: string): Promise<Website | Link | Pixel | Board | null> {
  // 并行查询4个实体表
  const [website, link, pixel, board] = await Promise.all([
    getWebsite(entityId),
    getLink(entityId),
    getPixel(entityId),
    getBoard(entityId),
  ]);

  // 返回第一个非空结果
  return website || link || pixel || board;
}
```

**性能影响证据**：
✅ 查询成本：4次独立数据库查询
✅ 适用场景：canViewWebsite、canViewEntity、canUpdateEntity、canDeleteEntity
❌ 不适用：canUpdateWebsite、canDeleteWebsite（这两个用专属 getWebsite）

### 3.5 hasPermission 权限匹配算法

**文件位置**：`src/lib/auth.ts:76-78`

```typescript
export async function hasPermission(role: string, permission: string | string[]) {
  // 将 permission 转数组后，只要有一项匹配即通过
  return ensureArray(permission).some(e => ROLE_PERMISSIONS[role]?.includes(e));
}
```

**算法行为证据**：
✅ admin 特殊处理：在调用前已通过 `user.isAdmin` 判断，实际 hasPermission 对 admin 无特殊处理
✅ 角色不存在：`ROLE_PERMISSIONS[role]` 返回 undefined → `?.includes()` 返回 undefined → some 返回 false
✅ 支持数组参数：可同时检查多个权限（OR 关系）

---

## 四、SQL 查询封装证据化

### 4.1 getTeamUser 查询实现

**文件位置**：`src/queries/prisma/teamUser.ts:12-19`

```typescript
export async function getTeamUser(teamId: string, userId: string) {
  return prisma.client.teamUser.findFirst({
    where: {
      teamId,
      userId,
    },
    // 注意：无 deletedAt 过滤条件！
  });
}
```

**索引使用证据**：
✅ 使用 `findFirst` + 两个 where 条件
✅ 数据库有 `@@index([teamId]) 和 `@@index([userId])` 单列索引
✅ **无联合索引**，依赖数据库查询优化器选择最优索引
❌ **无软删除过滤**：已删除的团队成员关系仍可被查询到

### 4.2 getWebsite 查询实现

**文件位置**：`src/queries/prisma/website.ts:11-23`

```typescript
export async function getWebsite(websiteId: string) {
  const website = await findWebsite({
    where: {
      id: websiteId,
      // 注意：无 deletedAt 过滤条件！
    },
  });

  if (!website) {
    return null;
  }

  return attachShareIdToWebsite(website);
}
```

**边界条件证据**：
❌ **无软删除过滤：已删除的网站仍可被查询到
✅ 附加 shareId 字段用于共享功能

### 4.3 getWebsites（列表查询）过滤条件

**文件位置**：`src/queries/prisma/website.ts:25-38`

```typescript
export async function getWebsites(criteria: Prisma.WebsiteFindManyArgs, filters: QueryFilters) {
  const where: Prisma.WebsiteWhereInput = {
    ...criteria.where,
    ...getSearchParameters(...),
    deletedAt: null,  // ✅ 列表查询有软删除过滤！
  };
  // ...
}
```

**关键不一致证据**：
⚠️ **重要发现**：
- 单条查询 `getWebsite` **无** `deletedAt: null` 过滤
- 列表查询 `getWebsites` **有** `deletedAt: null` 过滤
- 权限检查调用的是 `getWebsite`（单条查询，可能查询到已删除网站

### 4.4 getUserWebsites 与 getTeamWebsites

均调用 `getWebsites` → 继承软删除过滤。

---

## 五、边界条件详细核对表

| 边界条件 | 代码位置 | 实际行为 |
|---------|---------|---------|
| **website 不存在** | `website.ts:69-71` | `getWebsite` 返回 null → 权限函数返回 false |
| **teamUser 记录不存在** | `website.ts:77-80` | `getTeamUser` 返回 null → `teamUser && ...` 返回 false |
| **auth.user 为 null** | `website.ts:59` | `!user` → 返回 false |
| **entity 为 null** | `website.ts:25-27` | `!entity` → 返回 false |
| **website 同时有 userId 和 teamId** | N/A | 业务代码无防御，按代码顺序 userId 分支优先 |
| **已删除 website（单条查询）** | `website.ts:11-16` | 可被查询到，权限判断正常执行 |
| **已删除 website（列表查询）** | `website.ts:37` | 被 `deletedAt: null` 过滤，不可见 |
| **已删除 teamUser 关系** | `teamUser.ts:12-19` | 无 `deletedAt` 过滤，可被查询到 |
| **shareToken 查看网站** | `website.ts:12-21` | 绕过归属检查，直接返回 true |
| **teamViewOnly 查看网站** | `website.ts:33-37` | 仅需 teamUser 存在即可，无权限检查 → **可查看 |
| **teamViewOnly 更新网站** | `website.ts:77-81` | `hasPermission('team-view-only', 'website:update') → ROLE_PERMISSIONS['team-view-only'] 为空数组 → 返回 false |

---

## 六、团队权限函数核对表

| 函数名 | 文件位置 | 管理员 | 团队成员条件 |
|--------|---------|--------|-------------|
| `canViewTeam` | `team.ts:6-16` | ✅ 直接放行 | 仅需 getTeamUser 存在（无权限检查） |
| `canCreateTeam` | `team.ts:18-28` | ✅ 直接放行 | 需用户全局角色有 `team:create` 权限 |
| `canUpdateTeam` | `team.ts:30-42` | ✅ 直接放行 | 需团队角色有 `team:update` 权限 |
| `canDeleteTeam` | `team.ts:44-56` | ✅ 直接放行 | 需团队角色有 `team:delete` 权限 |
| `canDeleteTeamUser` | `team.ts:58-74` | ✅ 直接放行 | 特殊逻辑：<br>1. 自己离开：✅<br>2. 他人：需团队角色有 `team:update` |
| `canCreateTeamWebsite` | `team.ts:76-88` | ✅ 直接放行 | 需团队角色有 `website:create` 权限 |
| `canViewAllTeams` | `team.ts:90-92` | ✅ 仅管理员 | 普通用户返回 false |

---

## 七、isAdmin 派生逻辑证据

**文件位置**：`src/lib/auth.ts:51`

```typescript
if (user) {
  user.isAdmin = user.role === ROLES.admin;  // 硬编码判断
}
```

**核对结论**：
✅ isAdmin 是在认证阶段派生的布尔属性
✅ 判断逻辑是严格相等：`user.role === 'admin'`
✅ 不通过 `hasPermission` 函数

---

## 八、关键设计决策证据汇总

| 决策点 | 代码证据 | 实际行为 |
|--------|---------|---------|
| **getEntity 并行4表查询** | `entity.ts:5-10` | 每次权限检查触发4次DB查询 |
| **ShareToken 绕过归属检查** | `website.ts:12-21` | 6种匹配方式，命中即放行 |
| **userId/teamId 业务层互斥** | `websites/route.ts:79-81` | 创建时二选一，转移时设null |
| **查看 vs 更新权限差异** | `website.ts:33-37 vs 77-81` | 查看：仅需成员存在<br>更新：需成员 + 角色权限 |
| **单条 vs 列表软删除过滤不一致** | `website.ts:14 vs 37` | 单条查询：可查到已删除<br>列表查询：过滤已删除 |
| **TeamUser 无联合索引** | `schema.prisma:227-228` | 仅有单列索引，依赖查询优化器 |
| **权限函数无 deletedAt 过滤** | `teamUser.ts:12-19` | 已删除的成员关系参与权限判断 |

---

## 九、审计与安全实现核对

### 9.1 软删除机制不一致

**已确认事实**：
✅ Website 列表查询 `getWebsites` 有 `deletedAt: null` 过滤
❌ Website 单条查询 `getWebsite` 无软删除过滤
❌ TeamUser 查询 `getTeamUser` 无软删除过滤
❌ 权限检查调用的是无过滤版本

### 9.2 创建者追踪

**文件位置**：`websites/route.ts:73`

```typescript
createdBy: auth.user.id,  // 创建时记录
```

✅ Website.createdBy 字段全程保留，含转移后不变。

### 9.3 认证日志

**文件位置**：`auth.ts:9, 35`

```typescript
const log = debug('umami:auth');
log({ token, payload, authKey, shareToken, user });
```

✅ 使用 debug 模块记录认证上下文
✅ 包含 shareToken 存在性记录

---

## 十、完整调用链证据化

```
HTTP Request
    │
    ▼
┌───────────────────────────────────────────┐
│ parseRequest(request, schema)              │
│  - Zod 参数校验                         │
│  - 调用 checkAuth() 认证                 │
│  - 返回 { auth, body, query, error }    │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────┐
│ checkAuth()                              │
│  - 解析 Bearer Token / ShareToken     │
│  - 查询 getUser(userId)                    │
│  - user.isAdmin = (role === 'admin')      │
│  - 返回 Auth 对象                         │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────┐
│ 权限检查函数 canXxxWebsite()             │
│  - admin 快速路径                       │
│  - getWebsite() / getEntity()           │
│  - getTeamUser(teamId, userId)         │
│  - hasPermission(role, permission)     │
└────────────────────┬────────────────────┘
                     │
                     ▼
           ┌─────────┴─────────┐
           │                   │
           ▼                   ▼
    ┌───────────┐       ┌──────────────┐
    │ getWebsite │       │ getTeamUser  │
    │ 无软删除   │       │  无软删除    │
    └─────┬─────┘       └──────┬───────┘
           │                   │
           └─────────┬─────────┘
                     │
                     ▼
         ┌─────────────────────┐
         │ hasPermission()     │
         │ ROLE_PERMISSIONS   │
         │ 数组包含匹配        │
         └─────────┬─────────┘
                   │
                   ▼
         业务逻辑执行
```

---

## 核对总结

### 已确认的正确结论：

1. ✅ **角色枚举值**：团队角色使用连字符格式 `team-owner`，而非下划线或驼峰
2. ✅ **TeamUser 索引**：仅有 `[teamId]` 和 `[userId]` 两个单列索引，无联合索引
3. ✅ **归属互斥**：是业务代码约定，非 Prisma 数据库级约束
4. ✅ **查看权限**：团队成员查看网站无需权限检查，仅需存在成员关系
5. ✅ **getEntity**：并行查询 Website/Link/Pixel/Board 四张表
6. ⚠️ **软删除不一致**：单条查询无 deletedAt 过滤，列表查询有过滤
7. ✅ **canDeleteTeamUser**：用户可以自己离开团队，无需权限
8. ✅ **isAdmin**：是认证阶段派生的硬编码判断，不等同 `hasPermission('all')`
9. ✅ **ShareToken**：有6种匹配方式，绕过归属检查
10. ✅ **权限判断顺序**：userId 优先于 teamId 判断

### 潜在风险点：

1. ⚠️ 已删除网站在权限检查中仍可被访问到
2. ⚠️ 已删除团队成员关系仍可参与权限判断
3. ⚠️ 无数据库级约束保证 userId/teamId 互斥，脏数据可能导致权限判断结果不一致
4. ⚠️ TeamUser 无联合索引，高并发下可能存在性能隐患
