# 审计日志与配置变更追踪覆盖范围分析

## 1. 概述

本文档对 Umami 系统中审计日志与配置变更追踪的实现进行代码层面的分析，涵盖站点、团队、账号、偏好、敏感字段脱敏、保留策略和导出边界。

分析结论：系统未实现专门的 `audit_log` 审计日志表，变更追踪仅依赖实体表自身的时间戳字段，部分偏好设置仅保存在浏览器 localStorage 中。软删除支持不完整、查询过滤不一致，部分配置实体不具备软删除能力。

---

## 2. 变更记录覆盖范围

### 2.1 设置入口总览

**代码位置**：`src/app/(main)/settings/SettingsNav.tsx`

| 分类 | 设置项 | 路由 | 存储位置 | 可审计 |
| :--- | :--- | :--- | :--- | :--- |
| Application | Preferences（偏好） | `/settings/preferences` | 浏览器 localStorage | ❌ 仅客户端 |
| Account | Profile（个人资料） | `/settings/profile` | PostgreSQL | ⚠️ 部分 |
| Account | Teams（团队） | `/settings/teams` | PostgreSQL | ⚠️ 部分 |
| Websites | Website 设置 | `/settings/websites/[id]` | PostgreSQL | ⚠️ 部分 |

---

### 2.2 站点（Website）设置变更

#### 2.2.1 数据模型时间戳
**代码位置**：`prisma/schema.prisma`（第 66–97 行）

| 字段 | 说明 | 触发方式 |
| :--- | :--- | :--- |
| `createdAt` | 创建时间 | Prisma `@default(now())` |
| `updatedAt` | 最后更新时间 | Prisma `@updatedAt` 自动更新 |
| `deletedAt` | 删除时间 | 手动设置（仅 CLOUD_MODE） |
| `resetAt` | 数据重置时间 | 调用 `resetWebsite()` 时设置 |

#### 2.2.2 可变更字段列表
**代码位置**：`src/app/api/websites/[websiteId]/route.ts`（第 37–104 行）

```typescript
name: string                                    // 站点名称
domain: string                                  // 站点域名
shareId: string                                 // 共享ID（增删 Share 记录）
replayEnabled: boolean                          // 会话重放启用状态
replayConfig: {
  sampleRate: number                            // 采样率 0-1
  maskLevel: 'strict' | 'moderate'              // 脱敏级别
  maxDuration: number                           // 最大录制时长(ms)
  blockSelector: string                         // 阻止录制的 CSS 选择器
}
```

#### 2.2.3 站点数据重置
**代码位置**：`src/app/api/websites/[websiteId]/reset/route.ts`

操作内容：级联清空站点所有分析数据
- `sessionReplaySaved`、`sessionReplay`、`revenue`
- `eventData`、`sessionData`、`websiteEvent`、`session`

追踪方式：更新 `Website.resetAt = new Date()`

**代码位置**：`src/queries/prisma/website.ts`（第 133–186 行）

---

### 2.3 团队（Team）设置变更

#### 2.3.1 数据模型时间戳
**代码位置**：`prisma/schema.prisma`（第 197–214 行）

| 字段 | 说明 |
| :--- | :--- |
| `createdAt` | 创建时间 |
| `updatedAt` | 最后更新时间（`updateTeam` 中显式 `updatedAt: new Date()`） |
| `deletedAt` | 删除时间（仅 CLOUD_MODE） |

#### 2.3.2 可变更字段列表
**代码位置**：`src/app/api/teams/[teamId]/route.ts`（第 29–50 行）

```typescript
name: string        // 团队名称（最大 50 字符）
accessCode: string  // 访问码（最大 50 字符，唯一约束）
```

#### 2.3.3 团队成员管理
**代码位置**：`src/app/api/teams/[teamId]/users/route.ts` 和 `src/app/api/teams/[teamId]/users/[userId]/route.ts`

可变更内容：
- 添加成员（指定 `role`）
- 修改成员角色
- 移除成员

追踪方式：TeamUser 表自身的 `createdAt` / `updatedAt` 字段。

---

### 2.4 账号（User）设置变更

#### 2.4.1 数据模型时间戳
**代码位置**：`prisma/schema.prisma`（第 12–32 行）

| 字段 | 说明 |
| :--- | :--- |
| `createdAt` | 创建时间 |
| `updatedAt` | 最后更新时间 |
| `deletedAt` | 删除时间（仅 CLOUD_MODE） |

#### 2.4.2 个人资料页面展示字段
**代码位置**：`src/app/(main)/settings/profile/ProfileSettings.tsx`

页面中展示但**只读**的字段：
- `username`（用户名）
- `role`（角色：admin / user / view-only）

可操作项：
- 修改密码（仅非 CLOUD_MODE 显示按钮）

#### 2.4.3 管理员管理用户（可变更字段）
**代码位置**：`src/app/api/users/[userId]/route.ts`（第 27–81 行）

```typescript
username: string   // 仅管理员可修改，需唯一
password: string   // 管理员可直接重置（哈希后存储）
role: string       // 仅管理员可修改（admin / user / view-only）
```

#### 2.4.4 自助修改密码
**代码位置**：`src/app/(main)/settings/profile/PasswordEditForm.tsx` 和 `src/app/api/me/password/route.ts`

验证流程：
1. 校验 `currentPassword`（对比 bcrypt 哈希）
2. `newPassword` 与 `confirmPassword` 一致校验
3. 密码长度 >= 8 位
4. 写入前执行 `hashPassword(newPassword)`

API 路由：`POST /api/me/password`

---

### 2.5 偏好（Preferences）设置

> **重要**：所有偏好设置均保存在**浏览器 localStorage** 中，**不发送到服务端**，不在数据库留痕，无法进行服务端审计。

#### 2.5.1 偏好设置入口
**代码位置**：`src/app/(main)/settings/preferences/PreferenceSettings.tsx`

包含 5 项设置：默认日期范围、时区、语言、主题、版本。

#### 2.5.2 默认日期范围
**代码位置**：`src/app/(main)/settings/preferences/DateRangeSetting.tsx`

- localStorage Key：`umami.date-range`（常量 `DATE_RANGE_CONFIG`）
- 默认值：`24hour`（常量 `DEFAULT_DATE_RANGE_VALUE`）
- 存储方式：`setItem(DATE_RANGE_CONFIG, value)`

#### 2.5.3 时区
**代码位置**：`src/app/(main)/settings/preferences/TimezoneSetting.tsx` 和 `src/components/hooks/useTimezone.ts`

- localStorage Key：`umami.timezone`（常量 `TIMEZONE_CONFIG`）
- 默认值：浏览器系统时区 `Intl.DateTimeFormat().resolvedOptions().timeZone`
- 保存函数：`saveTimezone(value)` → `setItem(TIMEZONE_CONFIG, value)`

#### 2.5.4 语言
**代码位置**：`src/app/(main)/settings/preferences/LanguageSetting.tsx` 和 `src/components/hooks/useLocale.ts`

- localStorage Key：`umami.locale`（常量 `LOCALE_CONFIG`）
- 默认值：`en-US`（常量 `DEFAULT_LOCALE`）
- 保存函数：`saveLocale(value)` → `setItem(LOCALE_CONFIG, value)`

#### 2.5.5 主题
**代码位置**：`src/app/(main)/settings/preferences/ThemeSetting.tsx`

- localStorage Key：`umami.theme`（常量 `THEME_CONFIG`）
- 可选值：`light` / `dark`
- 默认值：`light`（常量 `DEFAULT_THEME`）
- 通过 `@umami/react-zen` 的 `useTheme()` hook 读写

#### 2.5.6 版本设置
**代码位置**：`src/app/(main)/settings/preferences/VersionSetting.tsx`

- localStorage Key：`umami.version-check`
- 功能：开启 / 关闭新版本检查
- 不涉及数据变更追踪。

---

### 2.6 其他实体的时间戳与软删除字段

| 实体 | 代码位置 | 时间戳字段 | deletedAt 字段 |
| :--- | :--- | :--- | :--- |
| TeamUser | `prisma/schema.prisma`（第 216–230 行） | createdAt, updatedAt | ❌ 无 |
| Report | `prisma/schema.prisma`（第 232–251 行） | createdAt, updatedAt | ❌ 无 |
| Segment | `prisma/schema.prisma`（第 253–266 行） | createdAt, updatedAt | ❌ 无 |
| Link | `prisma/schema.prisma`（第 288–307 行） | createdAt, updatedAt, deletedAt | ✅ 有 |
| Pixel | `prisma/schema.prisma`（第 309–327 行） | createdAt, updatedAt, deletedAt | ✅ 有 |
| Board | `prisma/schema.prisma`（第 329–347 行） | createdAt, updatedAt | ❌ **无（不支持软删除）** |
| Share | `prisma/schema.prisma`（第 348–360 行） | createdAt, updatedAt | ❌ 无 |
| SessionReplaySaved | `prisma/schema.prisma`（第 386–401 行） | createdAt, updatedAt | ❌ 无 |

> **关键差异**：Board 模型未定义 `deletedAt` 字段，因此**不具备软删除能力**；Link 和 Pixel 虽定义了 `deletedAt`，但删除实现中**并未使用**（详见第 4 章）。

---

### 2.7 变更记录局限性

1. **无审计日志表**：系统未设计 `audit_log` / `change_history` 表，无法追踪以下信息：
   - 变更操作人 ID（Website 有 `createdBy` 但仅记录创建人）
   - 变更字段的 before / after 值对比
   - 变更的精确时间（`updatedAt` 只能知道最后一次修改时间，中间历史丢失）

2. **偏好设置不可审计**：语言 / 时区 / 主题 / 默认日期范围纯客户端 localStorage，服务端无记录。

3. **无版本历史**：无法回滚任何配置变更。

---

## 3. 敏感字段脱敏

### 3.1 会话重放（Session Replay）脱敏

#### 3.1.1 脱敏级别配置
**代码位置**：`src/recorder/index.js`（第 87–99 行）

通过 `<script>` 标签的 `data-mask-level` 属性传入（在 Website 设置中配置 `replayConfig.maskLevel`）。

```javascript
const getMaskConfig = level => {
  switch (level) {
    case 'strict':
      return {
        maskAllInputs: true,
        maskTextSelector: '*',
      };
    default: // moderate
      return {
        maskAllInputs: true,
      };
  }
};
```

| 级别 | 配置项 | 实际效果 |
| :--- | :--- | :--- |
| `moderate`（默认） | `maskAllInputs: true` | 所有 `<input>` / `<textarea>` / `<select>` 等表单元素值替换为占位符 |
| `strict` | `maskAllInputs: true` + `maskTextSelector: '*'` | 输入框脱敏 + 页面所有文本节点内容替换为 `***` |

#### 3.1.2 脱敏底层实现
基于 rrweb 库的录制参数：
- `maskAllInputs`：rrweb 内置规则，将所有表单元素的值在录制时脱敏
- `maskTextSelector: '*'`：对匹配选择器的元素（此处为页面全部元素）的文本内容进行脱敏

#### 3.1.3 额外录制限制
**代码位置**：`src/recorder/index.js`（第 136–149 行）

```javascript
inlineStylesheet: true,
slimDOMOptions: {
  script: true, comment: true,
  headMetaDescKeywords: true, headMetaSocial: true,
  headMetaRobots: true, headMetaHttpEquiv: true,
  headMetaAuthorship: true, headMetaVerification: true,
},
recordCanvas: false,
recordCrossOriginIframes: false,
blockSelector,  // 用户自定义的 CSS 选择器，完全阻止录制匹配元素
```

- 不录制 Canvas 内容
- 不录制跨域 iframe
- 精简 DOM，移除 `<script>`、注释、各类 `<meta>` 标签
- 支持自定义 `blockSelector` 将整块元素从录制中排除

---

### 3.2 密码字段保护

#### 3.2.1 查询时默认排除密码
**代码位置**：`src/queries/prisma/user.ts`（第 14–30 行）

```typescript
select: {
  id: true,
  username: true,
  password: includePassword,   // 默认 false，不显式传参则不返回
  role: true,
  createdAt: true,
}
```

#### 3.2.2 管理员用户列表中排除密码
**代码位置**：`src/app/api/admin/users/route.ts`（第 35–37 行）

```typescript
omit: {
  password: true,
}
```

#### 3.2.3 密码哈希存储
**代码位置**：`src/app/api/users/[userId]/route.ts`（第 56–58 行）

```typescript
if (password) {
  data.password = hashPassword(password);  // bcrypt 哈希，字段长度 @db.VarChar(60)
}
```

#### 3.2.4 密码校验
**代码位置**：`src/app/api/me/password/route.ts`（第 24 行）

```typescript
if (!checkPassword(currentPassword, user.password)) {
  return badRequest({ message: 'Current password is incorrect' });
}
```

---

### 3.3 用户删除时的数据脱敏（CLOUD_MODE）
**代码位置**：`src/queries/prisma/user.ts`（第 129–146 行）

CLOUD_MODE 下执行用户删除时，为保留数据完整性不做物理删除，而是：
1. 将 `User.username` 替换为 `getRandomChars(32)` 生成的 32 位随机字符串
2. 标记 `User.deletedAt = new Date()`
3. 同时将该用户名下所有 Website 标记 `deletedAt`

作用：使已删除用户的个人信息无法通过用户名被识别。

---

## 4. 保留策略

### 4.1 数据保留策略现状

**结论：系统未实现基于时间的自动化数据保留（Data Retention）策略，无 TTL、无定时清理任务。**

数据清理仅在以下两种场景触发：
- 用户手动调用「重置站点数据」
- 用户 / 站点 / 团队 / Link / Pixel / Board 被执行删除操作

---

### 4.2 软删除字段与能力矩阵

| 实体 | deletedAt 字段 | CLOUD_MODE 下是否走软删除 | 非 CLOUD_MODE 下删除方式 |
| :--- | :--- | :--- | :--- |
| User | ✅ 有 | ✅ 是（username 随机化 + deletedAt） | 级联硬删除 |
| Website | ✅ 有 | ✅ 是（先级联删分析数据 → 再设置 deletedAt） | 级联硬删除 |
| Team | ✅ 有 | ✅ 是（仅自身 deletedAt） | teamUser + team 硬删除 |
| Link | ✅ 有 | ❌ **否（永远硬删除，字段形同虚设）** | 硬删除 |
| Pixel | ✅ 有 | ❌ **否（永远硬删除，字段形同虚设）** | 硬删除 |
| Board | ❌ **无** | N/A（永远硬删除） | 硬删除 |
| Report | ❌ 无 | N/A（永远硬删除） | 硬删除 |
| Segment | ❌ 无 | N/A（永远硬删除） | 硬删除 |
| Share | ❌ 无 | N/A（永远硬删除） | 硬删除 |
| TeamUser | ❌ 无 | N/A（CLOUD_MODE 下不处理，非 CLOUD_MODE 随级联硬删） | 随 owner 级联硬删除 |

---

### 4.3 查询层 deletedAt 过滤一致性校验

> **关键发现**：即使拥有 `deletedAt` 字段的实体，各查询函数之间的 `deletedAt: null` 过滤也**严重不一致**，可能导致软删除记录通过某些路径被查询到。

#### 4.3.1 User 查询
**代码位置**：`src/queries/prisma/user.ts`

| 函数 | deletedAt 过滤 | 说明 |
| :--- | :--- | :--- |
| `findUser` / `getUser` / `getUserByUsername` | ✅ 有 | 通过 `showDeleted` 参数控制，默认 `{ deletedAt: null }` |
| `getUsers`（列表） | ✅ 有 | 第 54 行显式 `deletedAt: null` |

#### 4.3.2 Website 查询
**代码位置**：`src/queries/prisma/website.ts`

| 函数 | deletedAt 过滤 | 说明 |
| :--- | :--- | :--- |
| `getWebsites`（列表） | ✅ 有 | 第 37 行显式 `deletedAt: null` |
| `getAllUserWebsitesIncludingTeamOwner` | ✅ 有 | 最终走 `getWebsites`，同时 team 也过滤 `deletedAt: null` |
| `getUserWebsites` / `getTeamWebsites` | ✅ 有 | 最终走 `getWebsites` |
| `getWebsiteCount` | ✅ 有 | 第 263 行显式 `deletedAt: null` |
| **`getWebsite`（单个查询）** | ❌ **无** | 第 11–23 行 `findWebsite` 直接按 id 查找，**不排除已删除** |
| **`findWebsite`** | ❌ **无** | 裸调用 Prisma，where 条件完全交给调用方 |

**风险**：外部传入已知 websiteId 调用 `getWebsite` / `findWebsite` 时，即使 CLOUD_MODE 已软删除，记录仍可能被取出并正常展示 / 使用。

#### 4.3.3 Team 查询
**代码位置**：`src/queries/prisma/team.ts`

| 函数 | deletedAt 过滤 | 说明 |
| :--- | :--- | :--- |
| `getUserTeams`（用户团队列表） | ✅ 有 | 第 53 行显式 `deletedAt: null`；计数中的 websites/members 也过滤了 deletedAt |
| `getAllUserTeams` | ✅ 有 | 第 90 行显式 `deletedAt: null` |
| **`getTeams`（通用列表）** | ❌ **无** | 第 34–37 行未加 `deletedAt`，完全依赖调用者 criteria 自行传入 |
| **`getTeam`（单个查询）** | ❌ **无** | 第 19–24 行裸 `findTeam`，**已删除团队通过 id 仍可取出** |
| **`findTeam`** | ❌ **无** | 裸调用 Prisma |

#### 4.3.4 Link 查询
**代码位置**：`src/queries/prisma/link.ts`

| 函数 | deletedAt 过滤 | 说明 |
| :--- | :--- | :--- |
| `getUserLinks`（用户维度） | ✅ 有 | 第 38 行显式 `deletedAt: null` |
| **`getTeamLinks`（团队维度）** | ❌ **无** | 第 48–50 行仅按 teamId 过滤，**不一致** |
| **`getLinks`（通用列表）** | ❌ **无** | 第 21–28 行未加，依赖调用者 |
| **`getLink`（单个）** | ❌ **无** | 直接裸查 |
| **`findLink`** | ❌ **无** | 直接裸查 |

#### 4.3.5 Pixel 查询
**代码位置**：`src/queries/prisma/pixel.ts`

| 函数 | deletedAt 过滤 | 说明 |
| :--- | :--- | :--- |
| **`getPixels`（通用）** | ❌ **无** | 第 20–23 行未加 |
| **`getUserPixels`** | ❌ **无** | 第 31–33 行仅按 userId |
| **`getTeamPixels`** | ❌ **无** | 第 42–44 行仅按 teamId |
| **`getPixel` / `findPixel`** | ❌ **无** | 直接裸查 |

> Pixel 虽然模型有 `deletedAt` 字段，但**所有查询函数均未过滤**，与删除实现（永远硬删除）一起看，该字段**完全未被使用**。

#### 4.3.6 Board 查询
**代码位置**：`src/queries/prisma/board.ts`

| 函数 | deletedAt 过滤 | 说明 |
| :--- | :--- | :--- |
| `getBoards` / `getUserBoards` / `getTeamBoards` | N/A | Board 无 `deletedAt` 字段，因此不存在过滤 |
| `getBoard` / `findBoard` | N/A | 同上 |

---

### 4.4 删除模式详解（按实体）

#### 4.4.1 用户删除
**代码位置**：`src/queries/prisma/user.ts`（第 102–206 行）

**CLOUD_MODE = true（软删除）** 仅执行：

```typescript
client.website.updateMany({ data: { deletedAt: new Date() }, where: { id: { in: websiteIds } }),
client.user.update({ data: { username: getRandomChars(32), deletedAt: new Date() }, where: { id: userId } }),
```

注意以下数据在 CLOUD_MODE 下**完全不处理**，会成为「孤儿」或继续存在：
- 用户作为 owner 的**团队**（Team 记录 + TeamUser 成员关联都不会被标记或删除）
- 用户名下的 **Link / Pixel / Board**（无级联、无标记）
- 用户创建的 **Report**

**CLOUD_MODE = false（硬删除）** 会级联执行：
1. eventData / sessionData / websiteEvent / session（名下网站的分析数据）
2. teamUser（两种情况：用户作为 owner 的团队全部成员 + 用户参与的所有团队成员关系）
3. team（删除用户作为 owner 的团队本身）
4. report（名下网站的报告 + 用户本人的报告）
5. website（名下网站）
6. user（用户自身）

但以下仍然**不会被硬删除**：
- 用户名下的 **Link / Pixel / Board**（无对应 deleteMany 语句）

#### 4.4.2 站点删除
**代码位置**：`src/queries/prisma/website.ts`（第 188–257 行）

**CLOUD_MODE = true**：
1. 先**硬删除**分析数据：sessionReplaySaved → sessionReplay → revenue → eventData → sessionData → websiteEvent → session
2. 再**硬删除**配置数据：report → segment → share（按 websiteId 或 entityId）
3. 最后对 website 自身**软删除**：`deletedAt = new Date()`

**CLOUD_MODE = false**：
1-2 步与 CLOUD_MODE 完全相同（分析/配置数据始终硬删）
3. 最后执行 `website.delete()` 硬删除

> **关键差异**：Website 的分析/关联数据在**两种模式下都会被硬删除**，只有 website 记录本身在 CLOUD_MODE 下走软删除（保留 name / domain / shareId 等配置）。

#### 4.4.3 团队删除
**代码位置**：`src/queries/prisma/team.ts`（第 143–172 行）

**CLOUD_MODE = true**：仅执行 `team.update({ deletedAt: new Date() })`，对下属资源**全部不处理**：
- `teamUser` 成员关系不会被删除或标记
- 团队下的 `websites`（CLOUD_MODE 可单独走 Website 软删除，但 Team 不触发）
- 团队下的 `links` / `pixels` / `boards`（Board 无 deletedAt，Link/Pixel 删除实现不处理 CLOUD_MODE）

**CLOUD_MODE = false**：
1. `teamUser.deleteMany` 删除成员关系
2. `team.delete()` 删除团队本身

但团队下属的 `websites / links / pixels / boards` 在**两种模式下都不会被团队删除触发**，全部保留为 `teamId` 指向已删除团队的记录。

#### 4.4.4 Link 删除
**代码位置**：`src/queries/prisma/link.ts`（第 64–66 行）

```typescript
export async function deleteLink(linkId: string) {
  return prisma.client.link.delete({ where: { id: linkId } });
}
```

始终为物理删除。Link 表虽定义 `deletedAt` 字段，但代码中**从未写入或查询该字段**，属无效定义。

#### 4.4.5 Pixel 删除
**代码位置**：`src/queries/prisma/pixel.ts`（第 58–60 行）

```typescript
export async function deletePixel(pixelId: string) {
  return prisma.client.pixel.delete({ where: { id: pixelId } });
}
```

与 Link 完全一致：始终物理删除，`deletedAt` 字段形同虚设。

#### 4.4.6 Board 删除
**代码位置**：`src/queries/prisma/board.ts`（第 66–68 行）

```typescript
export async function deleteBoard(boardId: string) {
  return prisma.client.board.delete({ where: { id: boardId } });
}
```

Board 模型本身**没有 `deletedAt` 字段**，不具备软删除能力，永远物理删除。

---

### 4.5 会话级超时（非数据保留）

#### 4.5.1 鉴权 Token 过期
**代码位置**：`src/lib/auth.ts`（第 62–74 行）

Redis 存储的鉴权 Key 通过 `redis.client.expire(authKey, expire)` 设置 TTL，过期后 Token 失效。此处仅影响登录态有效性，不删除持久化数据。

#### 4.5.2 Visit 超时
**代码位置**：`src/app/api/send/route.ts`（第 171 行）

```javascript
// Expire visit after 30 minutes
```

用于区分「同一会话内连续访问」与「新的一次回访」，30 分钟空闲即视为新 visit。

---

### 4.6 「Retention」说明

系统中出现的 `retention` / `label.retention` / `getRetention` **均指「用户留存分析报告」功能**（分析用户 N 天内回访率），与数据保留策略无关。

- **代码位置**：`src/queries/sql/reports/getRetention.ts`
- 入口路径：`/websites/[id]/retention`

---

### 4.7 ClickHouse 分析数据保留
**代码位置**：`db/clickhouse/schema.sql`

- 所有 MergeTree 表（`website_event`、`event_data`、`session_data`、`session_replay`、`website_revenue`）按 `toYYYYMM(created_at)` 做月分区
- **未配置 TTL 表达式**，数据不会自动过期
- 未提供定期 `DROP PARTITION` / `DELETE` 的定时任务代码

---

### 4.8 手动数据清理入口

#### 4.8.1 站点重置
**代码位置**：`src/queries/prisma/website.ts`（第 133–186 行）

清理范围（PostgreSQL 侧，均为 `deleteMany` 硬删除）：
`sessionReplaySaved` → `sessionReplay` → `revenue` → `eventData` → `sessionData` → `websiteEvent` → `session`

最后更新 `Website.resetAt = new Date()`。

#### 4.8.2 站点删除（非 CLOUD_MODE）
**代码位置**：`src/queries/prisma/website.ts`（第 188–257 行）

清理范围 = 重置范围 + `report` + `segment` + `share` + `website` 记录本身。

---

## 5. 导出边界

### 5.1 导出 API

**代码位置**：`src/app/api/websites/[websiteId]/export/route.ts`

| 项 | 内容 |
| :--- | :--- |
| 路由 | `GET /api/websites/[websiteId]/export` |
| 权限 | `canViewWebsite(auth, websiteId)`（网站查看权限即可，不需要管理员） |
| 请求参数 | `pagingParams`（分页）+ `withDateRange`（开始/结束日期） |
| 返回格式 | `{ "zip": "<base64 编码的 ZIP 文件内容>" }` |

### 5.2 导出内容

| CSV 文件 | 数据来源函数 | 包含内容 |
| :--- | :--- | :--- |
| `events.csv` | `getEventMetrics(type: 'event')` | 自定义事件名称 + 触发次数（聚合指标） |
| `pages.csv` | `getPageviewMetrics(type: 'path')` | URL 路径 + PV / UV 等聚合指标 |
| `referrers.csv` | `getPageviewMetrics(type: 'referrer')` | 来源域名 + 聚合指标 |
| `browsers.csv` | `getSessionMetrics(type: 'browser')` | 浏览器名称 + 会话数 |
| `os.csv` | `getSessionMetrics(type: 'os')` | 操作系统名称 + 会话数 |
| `devices.csv` | `getSessionMetrics(type: 'device')` | 设备类型 + 会话数 |
| `countries.csv` | `getSessionMetrics(type: 'country')` | 国家代码 + 会话数 |

**导出内容全部为聚合后的计数指标，不包含任何原始明细行。**

### 5.3 导出实现细节
- 使用 `Papa.unparse(data, { header: true, skipEmptyLines: true })` 生成 CSV
- 使用 `JSZip` 将 7 个 CSV 文件打包为 ZIP
- 使用 `Buffer.toString('base64')` 做 Base64 编码返回

---

### 5.4 导出边界（未导出的数据）

下列数据在当前版本的导出接口中**完全不可导出**：

| 类别 | 未导出项 |
| :--- | :--- |
| 原始明细 | `website_event` 逐行事件、`session` 明细、`visitId`、`distinctId` 明细 |
| 扩展数据 | `event_data`（自定义事件字段键值）、`session_data`（会话自定义属性键值） |
| 会话重放 | `session_replay.events`（ZSTD 压缩的 rrweb 录制数据） |
| 收入数据 | `revenue` 表逐笔交易数据 |
| 站点配置 | `replayEnabled`、`replayConfig`、`shareId`、`domain` 等站点属性 |
| 团队管理 | 团队列表、成员、角色、`accessCode` |
| 用户敏感信息 | 所有 User 相关字段（密码已哈希、不提供任何形式导出） |
| 报表/看板 | `Report`、`Segment`、`Board`、`Link`、`Pixel` 定义 |

---

### 5.5 权限控制
**代码位置**：`src/app/api/websites/[websiteId]/export/route.ts`（第 25–27 行）

```typescript
if (!(await canViewWebsite(auth, websiteId))) {
  return unauthorized();
}
```

**注意**：导出操作本身**无审计日志记录**，无法追溯「何时、何人、导出了哪个网站的哪个时间范围数据」。

---

## 6. 权限与安全

### 6.1 权限检查矩阵

| 操作 | 权限函数 | 所需角色 |
| :--- | :--- | :--- |
| 查看站点 | `canViewWebsite` | 网站所有者 / 团队成员（含 view-only）/ 管理员 |
| 更新站点配置 | `canUpdateWebsite` | 网站所有者 / teamOwner / teamManager / teamMember / admin |
| 删除站点 | `canDeleteWebsite` | 网站所有者 / teamOwner / teamManager / teamMember / admin |
| 重置站点数据 | `canUpdateWebsite` | 同上（复用更新权限） |
| 导出站点数据 | `canViewWebsite` | 所有可查看站点的用户（含 view-only） |
| 查看团队 | `canViewTeam` | 团队成员 |
| 更新团队 | `canUpdateTeam` | teamOwner / teamManager / admin |
| 删除团队 | `canDeleteTeam` | teamOwner / admin |
| 查看用户 | `canViewUser` | admin（或查看自己） |
| 修改用户（用户名/角色） | `canUpdateUser` | admin |
| 删除用户 | `canDeleteUser` | admin |
| 修改自己密码 | — | 任何已登录用户（非 CLOUD_MODE） |

### 6.2 角色权限定义
**代码位置**：`src/lib/constants.ts`（第 164–217 行）

| 角色 | 权限 |
| :--- | :--- |
| `admin` | `all`（全部权限） |
| `user` | 网站创建/更新/删除、团队创建 |
| `view-only` | 无管理权限 |
| `team-owner` | 团队更新/删除、网站管理、网站转移 |
| `team-manager` | 团队更新、网站创建/更新/删除、网站转团队 |
| `team-member` | 网站创建/更新/删除 |
| `team-view-only` | 无管理权限 |

---

## 7. 总结与改进建议

### 7.1 当前覆盖范围总结

| 维度 | 覆盖情况 | 详细说明 |
| :--- | :--- | :--- |
| 变更记录 | ⚠️ 部分覆盖 | 数据库实体仅依赖 `updatedAt` 时间戳；无审计日志表；无操作人记录；无 before/after 值对比；Website 仅有 `createdBy` 记录创建人 |
| 偏好设置 | ❌ 不可审计 | 语言/时区/主题/默认日期范围/版本检查均为浏览器 localStorage，服务端不落库 |
| 账号资料 | ⚠️ 部分覆盖 | 密码变更有独立 API 但无审计表；用户名/角色变更只能在 `User.updatedAt` 看到最后变更时间 |
| 团队/站点配置 | ⚠️ 部分覆盖 | 可变更字段有时间戳，但无字段级变更历史 |
| 敏感字段脱敏 | ✅ 良好覆盖 | 会话重放提供 strict/moderate 两级脱敏；密码 bcrypt 哈希存储 + 查询时默认排除；CLOUD_MODE 删除用户时用户名随机化 |
| 软删除字段一致性 | ❌ 严重不一致 | Board 无 deletedAt；Link/Pixel 有字段但删除实现不使用；查询层各函数过滤不统一 |
| 保留策略 | ❌ 未覆盖 | 无自动化 TTL / 定时清理；CLOUD_MODE 软删除级联不完整；ClickHouse 无 TTL 配置 |
| 导出边界 | ✅ 良好控制 | 仅允许导出 7 类聚合指标；原始明细/会话重放/收入/用户信息均不暴露；但导出操作本身无审计 |
| 导出操作审计 | ❌ 缺失 | 导出行为不记录任何日志 |

### 7.2 代码事实梳理：软删除过滤差异汇总

| 实体 | 有 deletedAt | 列表过滤 deletedAt | 单查过滤 deletedAt | CLOUD_MODE 下走软删除 |
| :--- | :--- | :--- | :--- | :--- |
| User | ✅ | ✅ `getUsers` | ✅ `findUser`（showDeleted 默认关） | ✅ |
| Website | ✅ | ✅ `getWebsites` | ❌ `getWebsite`/`findWebsite` 未过滤 | ✅（但分析数据在两种模式下都硬删） |
| Team | ✅ | 仅 `getUserTeams` 有 | ❌ `getTeam`/`getTeams` 未过滤 | ✅ |
| Link | ✅ | 仅 `getUserLinks` 有，`getTeamLinks` 没有 | ❌ 单查未过滤 | ❌（永远硬删除） |
| Pixel | ✅ | ❌ 全部没过滤 | ❌ 单查未过滤 | ❌（永远硬删除） |
| Board | ❌ | N/A | N/A | N/A（永远硬删除） |

---

### 7.3 改进建议

#### 建议 1：新增 `audit_log` 审计日志表
记录所有配置类变更，推荐字段：`entity_type`、`entity_id`、`field_name`、`old_value`、`new_value`、`user_id`、`ip_address`、`user_agent`、`created_at`，覆盖 Website/Team/TeamUser/User（含密码变更）/Segment/Report/Link/Pixel/Board 的增删改。

#### 建议 2：统一 `deletedAt` 能力与查询过滤
针对上表中不一致的地方，进行以下代码修正：

1. **Board 模型补充 `deletedAt` 字段**（`prisma/schema.prisma` Board 模型），使其具备软删除能力，与 Website/Team 保持一致。
2. **`deleteLink` / `deletePixel` 补充 CLOUD_MODE 分支**：CLOUD_MODE 下走 `update({ deletedAt: new Date() })`，与 deleteWebsite/deleteTeam 行为一致，让 `deletedAt` 字段真正生效。
3. **Pixel 全部查询函数补充 `deletedAt: null`**：`getPixels` / `getUserPixels` / `getTeamPixels` 统一加上过滤条件。
4. **Link 补齐 `getTeamLinks` 的 `deletedAt: null`**，与 `getUserLinks` 保持一致。
5. **Website 的 `findWebsite` / `getWebsite` 补充 `deletedAt: null`**，防止通过 ID 取出已删除站点；如需提供管理后台的「已删除列表」，新增 `findWebsiteIncludingDeleted` 专门函数。
6. **Team 的 `findTeam` / `getTeam` / `getTeams` 补充 `deletedAt: null`**，同理。

#### 建议 3：补全 CLOUD_MODE 软删除的级联处理
1. `deleteUser` CLOUD_MODE 分支补充：
   - 用户作为 owner 的团队 → 标记 `team.deletedAt`
   - 用户名下 `links / pixels / boards` → boards 需先补字段后再标记 deletedAt
   - 用户的 `reports` → Report 无 deletedAt，建议物理删除或补充字段后软删
2. `deleteTeam` CLOUD_MODE 分支补充：
   - 下属 `websites` → `updateMany({ where: { teamId }, data: { deletedAt } })`
   - 下属 `links / pixels / boards` → 同上
   - `teamUser` 成员关系 → 无 deletedAt 字段，建议物理删除
3. `deleteWebsite` CLOUD_MODE 分支（目前 report/segment/share 已经硬删）：如业务需要保留配置历史，Report/Segment/Share 补充 deletedAt 后改为软删除。

#### 建议 4：非 CLOUD_MODE 硬删除的补齐
修正 `deleteUser` 非 CLOUD_MODE 分支中遗漏的 `links / pixels / boards`，避免用户删除后留下 `userId = null` 的「孤儿」记录。

#### 建议 5：实现数据保留策略
- PostgreSQL：增加定时任务清理超过保留期的分析数据（session / websiteEvent / sessionReplay 等）
- ClickHouse：为 MergeTree 表增加 `TTL created_at + INTERVAL X DAY` 配置，并按月分区定期 DROP PARTITION
- 提供保留期配置项（按站点或全局）

#### 建议 6：操作行为审计
网站重置、网站删除、用户删除、团队删除、数据导出等高危操作强制写入 audit_log；导出接口额外记录 `website_id`、`date_range_start`、`date_range_end`、`exported_by`、`exported_at`。

#### 建议 7：偏好设置上云（可选）
若业务要求审计用户偏好，将 localStorage 的语言 / 时区 / 主题 / 默认日期范围迁移至用户表 JSON 字段或独立 `user_preference` 表，写操作走 API 并同步触发 audit_log。

---

**分析日期**：2026-06-20
**代码版本**：Umami 分析系统（基于提交时工作目录快照）
