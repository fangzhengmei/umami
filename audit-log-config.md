# 审计日志与配置变更追踪覆盖范围分析

## 1. 概述

本文档对 Umami 系统中审计日志与配置变更追踪的实现进行代码层面的分析，涵盖站点、团队、账号、偏好、敏感字段脱敏、保留策略和导出边界。

分析结论：系统未实现专门的 `audit_log` 审计日志表，变更追踪仅依赖实体表自身的时间戳字段，部分偏好设置仅保存在浏览器 localStorage 中。

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

### 2.6 其他实体的时间戳字段

| 实体 | 代码位置 | 追踪字段 |
| :--- | :--- | :--- |
| TeamUser | `prisma/schema.prisma`（第 216–230 行） | createdAt, updatedAt |
| Report | `prisma/schema.prisma`（第 232–251 行） | createdAt, updatedAt |
| Segment | `prisma/schema.prisma`（第 253–266 行） | createdAt, updatedAt |
| Link | `prisma/schema.prisma`（第 288–307 行） | createdAt, updatedAt, deletedAt |
| Pixel | `prisma/schema.prisma`（第 309–327 行） | createdAt, updatedAt, deletedAt |
| Board | `prisma/schema.prisma`（第 329–347 行） | createdAt, updatedAt |
| Share | `prisma/schema.prisma`（第 348–360 行） | createdAt, updatedAt |
| SessionReplaySaved | `prisma/schema.prisma`（第 386–401 行） | createdAt, updatedAt |

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
- 用户 / 站点 / 团队被执行删除操作

---

### 4.2 删除模式详解

删除行为由环境变量 `CLOUD_MODE` 控制，行为差异如下表：

| 操作 | CLOUD_MODE = true（软删除） | CLOUD_MODE = false（硬删除） |
| :--- | :--- | :--- |
| **删除用户** | 用户名随机化 + `user.deletedAt`；名下网站 `website.deletedAt`；**团队与 teamUser 不处理** | 级联删除 website/event/session 数据、删除用户拥有的团队与 teamUser、删除用户 |
| **删除网站** | 仅 `website.deletedAt = new Date()`；**下属数据（event/session/report/segment/share）不处理** | 级联删除网站所有分析数据 + report + segment + share + website 记录 |
| **删除团队** | 仅 `team.deletedAt = new Date()`；**下属 teamUser / websites / links / pixels / boards 不处理** | 删除 teamUser 关联 + team 记录本身；**下属 websites / links / pixels / boards 不处理** |
| **删除 Link** | 物理删除 `prisma.link.delete()`，**无 CLOUD_MODE 分支** | 同左（物理删除） |
| **删除 Pixel** | 物理删除 `prisma.pixel.delete()`，**无 CLOUD_MODE 分支** | 同左（物理删除） |

#### 4.2.1 用户删除细节
**代码位置**：`src/queries/prisma/user.ts`（第 102–206 行）

CLOUD_MODE 分支（第 129–146 行）仅执行：

```typescript
client.website.updateMany({ data: { deletedAt: new Date() }, where: { id: { in: websiteIds } }),
client.user.update({ data: { username: getRandomChars(32), deletedAt: new Date() }, where: { id: userId } }),
```

注意：用户作为 teamOwner 的团队在 CLOUD_MODE 下**不会被删除或标记 deletedAt**，团队成员关联 `teamUser` 也不处理。

#### 4.2.2 站点删除细节
**代码位置**：`src/queries/prisma/website.ts`（第 188–257 行）

CLOUD_MODE 分支（第 234–243 行）：

```typescript
client.website.update({ data: { deletedAt: new Date() }, where: { id: websiteId } })
```

下属的 `session / websiteEvent / sessionData / eventData / revenue / sessionReplay` 等**不会被清理**，`report / segment / share` 也不清理。

#### 4.2.3 团队删除细节
**代码位置**：`src/queries/prisma/team.ts`（第 143–172 行）

CLOUD_MODE 分支（第 147–158 行）：仅 `team.deletedAt`。

非 CLOUD_MODE 分支（第 160–171 行）：删除 `teamUser` 关联 + `team` 记录。注意团队下的 `websites`、`links`、`pixels`、`boards` 在**两种模式下都不会被删除**，只会变成 `teamId` 指向已删除团队的「孤儿」记录。

#### 4.2.4 Link 删除
**代码位置**：`src/queries/prisma/link.ts`（第 64–66 行）

```typescript
export async function deleteLink(linkId: string) {
  return prisma.client.link.delete({ where: { id: linkId } });
}
```

始终为硬删除，Link 表即使有 `deletedAt` 字段也**未被使用**。

---

### 4.3 软删除过滤保证
所有查询均带有 `deletedAt: null` 过滤条件，确保软删除记录不会出现在正常列表中：
- `getUser` / `getUsers`：`src/queries/prisma/user.ts`（第 21、54 行）
- `getWebsites`：`src/queries/prisma/website.ts`（第 37 行）
- `getUserTeams`：`src/queries/prisma/team.ts`（第 53 行）
- `getUserLinks`：`src/queries/prisma/link.ts`（第 38 行）

---

### 4.4 会话级超时（非数据保留）

#### 4.4.1 鉴权 Token 过期
**代码位置**：`src/lib/auth.ts`（第 62–74 行）

Redis 存储的鉴权 Key 通过 `redis.client.expire(authKey, expire)` 设置 TTL，过期后 Token 失效。此处仅影响登录态有效性，不删除持久化数据。

#### 4.4.2 Visit 超时
**代码位置**：`src/app/api/send/route.ts`（第 171 行）

```javascript
// Expire visit after 30 minutes
```

用于区分「同一会话内连续访问」与「新的一次回访」，30 分钟空闲即视为新 visit。

---

### 4.5 「Retention」说明

系统中出现的 `retention` / `label.retention` / `getRetention` **均指「用户留存分析报告」功能**（分析用户 N 天内回访率），与数据保留策略无关。

- **代码位置**：`src/queries/sql/reports/getRetention.ts`
- 入口路径：`/websites/[id]/retention`

---

### 4.6 ClickHouse 分析数据保留
**代码位置**：`db/clickhouse/schema.sql`

- 所有 MergeTree 表（`website_event`、`event_data`、`session_data`、`session_replay`、`website_revenue`）按 `toYYYYMM(created_at)` 做月分区
- **未配置 TTL 表达式**，数据不会自动过期
- 未提供定期 `DROP PARTITION` / `DELETE` 的定时任务代码

---

### 4.7 手动数据清理入口

#### 4.7.1 站点重置
**代码位置**：`src/queries/prisma/website.ts`（第 133–186 行）

清理范围（PostgreSQL 侧，均为 `deleteMany` 硬删除）：
`sessionReplaySaved` → `sessionReplay` → `revenue` → `eventData` → `sessionData` → `websiteEvent` → `session`

最后更新 `Website.resetAt = new Date()`。

#### 4.7.2 站点删除（非 CLOUD_MODE）
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

## 7. 总结与建议

### 7.1 当前覆盖范围总结

| 维度 | 覆盖情况 | 详细说明 |
| :--- | :--- | :--- |
| 变更记录 | ⚠️ 部分覆盖 | 数据库实体仅依赖 `updatedAt` 时间戳；无审计日志表；无操作人记录；无 before/after 值对比；Website 仅有 `createdBy` 记录创建人 |
| 偏好设置 | ❌ 不可审计 | 语言/时区/主题/默认日期范围/版本检查均为浏览器 localStorage，服务端不落库 |
| 账号资料 | ⚠️ 部分覆盖 | 密码变更有独立 API 但无审计表；用户名/角色变更只能在 `User.updatedAt` 看到最后变更时间 |
| 团队/站点配置 | ⚠️ 部分覆盖 | 可变更字段有时间戳，但无字段级变更历史 |
| 敏感字段脱敏 | ✅ 良好覆盖 | 会话重放提供 strict/moderate 两级脱敏；密码 bcrypt 哈希存储 + 查询时默认排除；CLOUD_MODE 删除用户时用户名随机化 |
| 保留策略 | ❌ 未覆盖 | 无自动化 TTL / 定时清理；CLOUD_MODE 软删除存在残留数据（下属数据不清理）；ClickHouse 无 TTL 配置 |
| 导出边界 | ✅ 良好控制 | 仅允许导出 7 类聚合指标；原始明细/会话重放/收入/用户信息均不暴露；但导出操作本身无审计 |
| 导出操作审计 | ❌ 缺失 | 导出行为不记录任何日志 |

### 7.2 改进建议

1. **新增 `audit_log` 表**：记录所有配置类变更
   - 推荐字段：`entity_type`、`entity_id`、`field_name`、`old_value`、`new_value`、`user_id`、`ip_address`、`user_agent`、`created_at`
   - 覆盖：Website / Team / TeamUser / User（含密码变更）/ Segment / Report / Link / Pixel / Board 的增删改

2. **偏好设置上云（可选）**：若需要审计偏好变更，将 localStorage 项迁移至用户表 JSON 字段或独立 `user_preference` 表，写操作走 API 并触发 audit_log。

3. **补全 CLOUD_MODE 软删除的级联处理**：
   - 删除用户时同步标记其拥有的团队 `deletedAt`、团队成员 `teamUser` 状态
   - 删除团队时同步标记 `websites` / `links` / `pixels` / `boards` 的 `deletedAt`
   - Website 软删除时同步标记 `reports` / `segments` / `shares`
   - Link / Pixel 删除逻辑补充 CLOUD_MODE 分支

4. **实现数据保留策略**：
   - PostgreSQL：增加定时任务清理已超过保留期的 `website_event`、`session`、`session_replay` 等
   - ClickHouse：为 MergeTree 表增加 `TTL created_at + INTERVAL X DAY` 配置，并按月分区定期 DROP PARTITION
   - 提供保留期配置项（按站点或全局）

5. **操作行为审计**：
   - 网站重置、网站删除、用户删除、团队删除、数据导出等高危操作，强制写入 audit_log
   - 导出接口记录 `website_id`、`date_range_start`、`date_range_end`、`exported_by`、`exported_at`

6. **Link / Pixel 统一删除模式**：当前 `deleteLink` / `deletePixel` 未使用 `deletedAt` 字段，建议与 Website/Team/User 保持一致，补充 CLOUD_MODE 软删除分支。

---

**分析日期**：2026-06-20
**代码版本**：Umami 分析系统（基于提交时工作目录快照）
