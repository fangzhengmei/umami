# Umami 网站权限系统协作分析

## 概述

Umami 的网站权限系统是一个基于 **用户-团队-网站** 三层架构的权限体系，结合了角色权限（Role-Based Permissions）和实体归属（Entity Ownership）两种授权方式。系统支持：

- 用户级权限（独立网站所有者）
- 团队级权限（团队成员角色）
- 共享令牌访问（Share Token）
- 管理员超级权限

---

## 一、数据模型层（Prisma Schema）

### 1.1 核心实体关系

```
User (1) ──< Website (N)
  |
  ∧
  |
TeamUser (N) ──< Team (1) ──< Website (N)
```

### 1.2 模型定义

#### User 模型
- **id**: UUID，用户唯一标识
- **role**: String，用户全局角色（admin/user/viewOnly）
- **websites**: Website[]，用户拥有的网站
- **teams**: TeamUser[]，用户所属的团队成员关系

#### Website 模型
- **id**: UUID，网站唯一标识
- **userId**: String | null，个人所有者（互斥于 teamId）
- **teamId**: String | null，团队所有者（互斥于 userId）
- **createdBy**: String，创建者用户ID
- **user**: User | null，关联个人用户
- **team**: Team | null，关联团队

**关键设计**：网站只能属于用户或团队其中之一，不能同时属于两者。

#### Team 模型
- **id**: UUID，团队唯一标识
- **name**: String，团队名称
- **websites**: Website[]，团队拥有的网站
- **members**: TeamUser[]，团队成员

#### TeamUser 模型
- **id**: UUID，关系唯一标识
- **teamId**: String，团队ID
- **userId**: String，用户ID
- **role**: String，团队内角色（teamOwner/teamManager/teamMember/teamViewOnly）

---

## 二、常量定义层（Constants）

### 2.1 角色定义（ROLES）

| 角色 | 说明 | 适用范围 |
|------|------|----------|
| `admin` | 超级管理员，拥有所有权限 | 全局 |
| `user` | 普通用户 | 全局 |
| `viewOnly` | 只读用户 | 全局 |
| `teamOwner` | 团队所有者，团队最高权限 | 团队内 |
| `teamManager` | 团队管理员 | 团队内 |
| `teamMember` | 团队成员 | 团队内 |
| `teamViewOnly` | 团队只读成员 | 团队内 |

### 2.2 权限定义（PERMISSIONS）

| 权限 | 说明 |
|------|------|
| `all` | 所有权限（admin 专属） |
| `websiteCreate` | 创建网站 |
| `websiteUpdate` | 更新网站 |
| `websiteDelete` | 删除网站 |
| `websiteTransferToTeam` | 将网站转移到团队 |
| `websiteTransferToUser` | 将网站从团队转移给用户 |
| `teamCreate` | 创建团队 |
| `teamUpdate` | 更新团队 |
| `teamDelete` | 删除团队 |

### 2.3 角色权限映射（ROLE_PERMISSIONS）

```typescript
{
  admin: ['all'],                                    // 超级管理员：所有权限
  user: [                                            // 普通用户
    'website:create', 'website:update', 'website:delete', 'team:create'
  ],
  viewOnly: [],                                      // 只读用户：无权限
  
  teamOwner: [                                       // 团队所有者
    'team:update', 'team:delete', 
    'website:create', 'website:update', 'website:delete',
    'website:transfer-to-team', 'website:transfer-to-user'
  ],
  teamManager: [                                     // 团队管理员
    'team:update',
    'website:create', 'website:update', 'website:delete',
    'website:transfer-to-team'
  ],
  teamMember: [                                      // 团队成员
    'website:create', 'website:update', 'website:delete'
  ],
  teamViewOnly: []                                   // 团队只读成员：无权限
}
```

---

## 三、权限检查层（Permissions）

### 3.1 核心权限函数

#### 网站权限：`canViewWebsite(auth, websiteId)`

**位置**：`src/permissions/website.ts`

**权限判定流程**：

```
用户请求查看网站
    │
    ├─► 是否为 admin？ ──是──► 允许 ✓
    │
    ├─► 是否携带 share token？
    │   └─► token 是否包含该 websiteId？ ──是──► 允许 ✓
    │
    └─► 获取实体（Website/Link/Pixel/Board）
        │
        ├─► 实体不存在？ ──► 拒绝 ✗
        │
        ├─► 实体有 userId？ ──用户ID匹配──► 允许 ✓
        │
        └─► 实体有 teamId？
             └─► 查询 TeamUser（teamId + userId）
                  └─► 存在记录？ ──是──► 允许 ✓
```

**代码实现关键点**：
- 支持共享令牌（Share Token）访问
- 支持跨实体类型（Website/Link/Pixel/Board）统一检查
- 通过 `getEntity()` 并行查询多种实体类型

#### 网站权限：`canUpdateWebsite(auth, websiteId)`

**权限判定流程**：

```
用户请求更新网站
    │
    ├─► 是否为 admin？ ──是──► 允许 ✓
    │
    ├─► 查询 website 实体
    │   │
    │   ├─► 不存在？ ──► 拒绝 ✗
    │   │
    │   ├─► website.userId 存在？
    │   │   └─► 匹配当前用户？ ──是──► 允许 ✓
    │   │
    │   └─► website.teamId 存在？
    │        └─► 查询 TeamUser（teamId + userId）
    │             └─► 存在记录 且 角色有 website:update 权限？
    │                  └─► 是 ──► 允许 ✓
    │
    └─► 其他情况 ──► 拒绝 ✗
```

#### 网站权限：`canDeleteWebsite(auth, websiteId)`

逻辑与 `canUpdateWebsite` 类似，但检查 `website:delete` 权限。

#### 网站转移权限

- `canTransferWebsiteToUser`：团队内网站转移给用户，需要 `website:transfer-to-user` 权限
- `canTransferWebsiteToTeam`：个人网站转移到团队，需要团队角色有 `website:transfer-to-team` 权限

### 3.2 团队权限函数

**位置**：`src/permissions/team.ts`

| 函数 | 权限检查 |
|------|----------|
| `canViewTeam` | admin 或 团队成员即可 |
| `canCreateTeam` | admin 或 角色有 `team:create` |
| `canUpdateTeam` | admin 或 团队角色有 `team:update` |
| `canDeleteTeam` | admin 或 团队角色有 `team:delete` |
| `canCreateTeamWebsite` | admin 或 团队角色有 `website:create` |

**特殊权限**：`canDeleteTeamUser`
- admin 可以删除任何成员
- 用户可以自己离开团队（removeUserId === userId）
- 团队管理员可以删除其他成员

### 3.3 通用实体权限

**位置**：`src/permissions/entity.ts`

提供跨实体类型的权限检查，支持：
- `canViewEntity`：查看权限（仅检查归属，不检查具体权限）
- `canUpdateEntity`：更新权限
- `canDeleteEntity`：删除权限

这些函数通过 `getEntity()` 自动识别实体类型（Website/Link/Pixel/Board），然后复用相同的权限判定逻辑。

---

## 四、共享库层（Lib）

### 4.1 认证核心：`auth.ts`

#### `checkAuth(request)` - 认证入口

**功能**：解析请求中的认证信息，返回认证上下文

**认证方式**：
1. **Bearer Token**：JWT 令牌，包含 `userId` 或 `authKey`
2. **Share Token**：通过 `x-umami-share-token` 头传递，用于共享访问

**Redis 集成**：
- 当使用 `authKey` 时，从 Redis 中获取用户会话
- 支持会话过期管理

**返回 Auth 对象**：
```typescript
{
  token: string,           // 原始令牌
  authKey: string,         // Redis 会话键
  shareToken: object,      // 共享令牌解析结果
  user: {                  // 用户信息
    id: string,
    role: string,
    isAdmin: boolean,      // 派生属性
    // ...其他用户字段
  }
}
```

#### `hasPermission(role, permission)` - 权限匹配

**功能**：检查角色是否拥有指定权限

**实现逻辑**：
```typescript
function hasPermission(role, permission) {
  // admin 拥有所有权限
  if (role === 'admin') return true;
  
  // 检查 ROLE_PERMISSIONS 映射
  const permissions = ROLE_PERMISSIONS[role];
  return ensureArray(permission).some(p => permissions?.includes(p));
}
```

### 4.2 实体解析：`entity.ts`

#### `getEntity(entityId)` - 多态实体查询

**核心实现**：
```typescript
async function getEntity(entityId) {
  // 并行查询所有可能的实体类型
  const [website, link, pixel, board] = await Promise.all([
    getWebsite(entityId),
    getLink(entityId),
    getPixel(entityId),
    getBoard(entityId),
  ]);
  
  // 返回第一个匹配的实体
  return website || link || pixel || board;
}
```

**设计意图**：
- 统一的实体权限检查入口
- 支持跨实体类型的共享访问
- 性能优化：并行查询减少等待时间

### 4.3 请求处理：`request.ts`

#### `parseRequest(request, schema, options)` - 请求解析中间件

**功能流程**：
1. 解析 URL 查询参数
2. 解析 JSON 请求体
3. Zod Schema 验证（可选）
4. 调用 `checkAuth()` 进行认证（可跳过）
5. 返回统一的请求上下文对象

**与权限系统的集成**：
- 所有 API 路由在处理业务逻辑前，都通过此函数获取认证上下文
- `auth` 对象传递给后续的权限检查函数

---

## 五、SQL 查询层（Queries/Prisma）

### 5.1 网站查询封装

**位置**：`src/queries/prisma/website.ts`

#### `getWebsite(websiteId)` - 获取单个网站

**功能**：查询网站并附加 `shareId`（共享链接标识）

#### `getUserWebsites(userId, filters)` - 获取用户网站

**查询条件**：`where: { userId, deletedAt: null }`

#### `getTeamWebsites(teamId, filters)` - 获取团队网站

**查询条件**：`where: { teamId, deletedAt: null }`

#### `getAllUserWebsitesIncludingTeamOwner(userId, filters)`

**特殊查询**：获取用户作为团队所有者管理的所有网站
- OR 条件：直接拥有的网站 + 作为 teamOwner 团队下的网站

### 5.2 团队用户查询

**位置**：`src/queries/prisma/teamUser.ts`

#### `getTeamUser(teamId, userId)`

**核心查询**：
```typescript
prisma.client.teamUser.findFirst({
  where: { teamId, userId }
});
```

**性能特征**：
- 使用联合索引（teamId, userId）
- 查询效率 O(log n)
- 权限检查的关键路径

### 5.3 团队查询

**位置**：`src/queries/prisma/team.ts`

#### `getUserTeams(userId, filters)`

**查询特性**：
- 包含成员信息（include: { members: true }）
- 包含统计计数（_count: { websites, members }）
- 过滤已删除的团队和用户

---

## 六、API 路由层集成

### 6.1 网站列表示例

**位置**：`src/app/api/websites/route.ts`

```typescript
// GET /api/websites
export async function GET(request: Request) {
  // 1. 解析请求 + 认证
  const { auth, query, error } = await parseRequest(request, schema);
  
  // 2. 认证失败处理
  if (error) return error();
  
  // 3. 获取用户ID
  const userId = auth.user.id;
  
  // 4. 构建查询过滤器
  const filters = await getQueryFilters(query);
  
  // 5. 数据查询（已隐含权限过滤：仅查询该用户的网站）
  if (query.includeTeams) {
    return json(await getAllUserWebsitesIncludingTeamOwner(userId, filters));
  }
  return json(await getUserWebsites(userId, filters));
}
```

### 6.2 网站创建示例

```typescript
// POST /api/websites
export async function POST(request: Request) {
  const { auth, body, error } = await parseRequest(request, schema);
  
  // 权限检查：团队网站 OR 个人网站
  const canCreate = (teamId && await canCreateTeamWebsite(auth, teamId)) 
                    || await canCreateWebsite(auth);
  
  if (!canCreate) return unauthorized();
  
  // 业务逻辑...
}
```

---

## 七、边界条件处理

### 7.1 空值处理

| 场景 | 处理方式 |
|------|----------|
| website 不存在 | `canViewWebsite` 返回 `false` |
| teamUser 记录不存在 | 团队权限检查返回 `false` |
| auth.user 为 null | 所有权限检查返回 `false` |
| entity 为 null（getEntity 无结果） | 拒绝访问 |

### 7.2 互斥归属校验

网站不能同时属于用户和团队：
```typescript
// 创建网站时的隐含逻辑
if (!teamId) {
  data.userId = auth.user.id;  // 仅设置其一
}
```

### 7.3 权限升级保护

- 普通用户无法将自己提升为 admin
- 团队成员无法自我提升为 teamOwner
- 权限检查始终基于数据库中的真实角色，而非请求参数

### 7.4 共享令牌边界

- Share Token 仅用于查看，不能用于更新/删除
- Share Token 需要配合 `x-umami-share-context` 头使用
- 无用户上下文时，仅允许通过 Share Token 访问

---

## 八、审计与安全

### 8.1 创建者追踪

**字段**：`Website.createdBy`
- 记录网站的原始创建者
- 即使网站转移到团队，创建者信息保留
- 用于审计追踪和责任归属

### 8.2 软删除机制

**字段**：`Website.deletedAt`, `Team.deletedAt`, `User.deletedAt`
- 不物理删除数据，保留完整审计轨迹
- 查询时自动过滤 `deletedAt: null`
- 支持数据恢复（如果业务需要）

### 8.3 认证日志

**位置**：`src/lib/auth.ts`
```typescript
const log = debug('umami:auth');
// 记录认证过程的关键节点
log({ token, payload, authKey, shareToken, user });
```

### 8.4 Redis 会话安全

- 使用随机生成的 `authKey` 作为会话标识
- 支持会话过期（expire）
- 用户权限变更时，可通过清除 Redis 会话强制重新认证

---

## 九、协作架构总览

### 9.1 完整调用链

```
  HTTP Request
      │
      ▼
  ┌─────────────────┐
  │  parseRequest   │  请求解析 + Schema 验证
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │   checkAuth     │  认证：Bearer Token / Share Token
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ canXXXWebsite   │  权限检查函数
  └────────┬────────┘
           │
     ┌─────┴─────┐
     │           │
     ▼           ▼
┌──────────┐ ┌──────────┐
│ getWebsite │ │ getTeamUser │  Prisma 查询
└──────────┘ └──────────┘
     │           │
     └─────┬─────┘
           │
           ▼
  ┌─────────────────┐
  │  hasPermission  │  角色权限匹配
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │  业务逻辑执行   │
  └─────────────────┘
```

### 9.2 模块依赖关系

```
src/permissions/*.ts
       │
       ├─► src/lib/auth.ts (hasPermission, checkAuth)
       ├─► src/lib/constants.ts (ROLES, PERMISSIONS, ROLE_PERMISSIONS)
       ├─► src/lib/entity.ts (getEntity)
       └─► src/queries/prisma/*.ts (getWebsite, getTeamUser, etc.)
              │
              └─► @/generated/prisma/client (Prisma Client)
```

### 9.3 关键设计决策

| 决策 | 优点 | 潜在缺点 |
|------|------|----------|
| **并行 getEntity 查询** | 代码简洁，支持多态实体 | 4次 DB 查询，有性能开销 |
| **Share Token 权限旁路** | 灵活的共享机制 | 增加权限检查复杂度 |
| **团队/用户互斥归属** | 数据模型清晰 | 转移操作复杂 |
| **角色权限数组匹配** | 直观易扩展 | 细粒度权限控制需增加数组项 |
| **软删除保留数据** | 审计友好 | 查询时需注意过滤条件 |

---

## 十、关键文件清单

| 文件路径 | 核心职责 |
|----------|----------|
| `prisma/schema.prisma` | 数据模型定义 |
| `src/lib/constants.ts` | 角色、权限常量定义 |
| `src/lib/auth.ts` | 认证、权限匹配函数 |
| `src/lib/entity.ts` | 多态实体解析 |
| `src/lib/request.ts` | 请求解析中间件 |
| `src/permissions/website.ts` | 网站权限检查函数 |
| `src/permissions/team.ts` | 团队权限检查函数 |
| `src/permissions/entity.ts` | 通用实体权限检查 |
| `src/queries/prisma/website.ts` | 网站查询封装 |
| `src/queries/prisma/team.ts` | 团队查询封装 |
| `src/queries/prisma/teamUser.ts` | 团队成员查询封装 |

---

## 总结

Umami 的网站权限系统是一个设计精良的三层权限架构：

1. **数据层**：通过 User-TeamUser-Team-Website 关系模型实现灵活的归属管理
2. **权限层**：细粒度的角色-权限映射，支持全局角色和团队内角色
3. **应用层**：统一的权限检查函数，与 API 路由无缝集成
4. **安全特性**：认证日志、软删除审计、Redis 会话管理

该系统在灵活性与可维护性之间取得了良好平衡，支持个人用户、团队协作、共享访问等多种使用场景。
