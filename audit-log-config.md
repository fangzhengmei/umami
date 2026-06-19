# 审计日志与配置变更追踪覆盖范围分析

## 1. 概述

本文档对 Umami 系统中审计日志与配置变更追踪的实现进行代码层面的分析，涵盖站点、团队、用户设置变更记录、敏感字段脱敏、保留策略和导出边界。

---

## 2. 变更记录覆盖范围

### 2.1 数据模型时间戳字段

所有核心实体模型均包含时间戳字段，用于追踪实体的创建、更新和删除时间，但**无专门的审计日志表或变更历史记录表**。

#### 2.1.1 用户（User）
- **代码位置**: [schema.prisma#L12-L32](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/prisma/schema.prisma#L12-L32)
- **追踪字段**:
  - `createdAt`: 创建时间（自动生成）
  - `updatedAt`: 更新时间（Prisma `@updatedAt` 自动更新）
  - `deletedAt`: 删除时间（软删除标记）

#### 2.1.2 站点（Website）
- **代码位置**: [schema.prisma#L66-L97](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/prisma/schema.prisma#L66-L97)
- **追踪字段**:
  - `createdAt`: 创建时间
  - `updatedAt`: 更新时间
  - `deletedAt`: 删除时间
  - `resetAt`: 数据重置时间（[resetWebsite](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/prisma/website.ts#L133-L186) 操作时更新

#### 2.1.3 团队（Team）
- **代码位置**: [schema.prisma#L197-L214](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/prisma/schema.prisma#L197-L214)
- **追踪字段**:
  - `createdAt`: 创建时间
  - `updatedAt`: 更新时间（[updateTeam](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/prisma/team.ts#L129-L141) 显式设置 `updatedAt: new Date()`
  - `deletedAt`: 删除时间

#### 2.1.4 其他实体

| 实体 | 代码位置 | 追踪字段 |
|------|----------|----------|
| TeamUser | [schema.prisma#L216-L230](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/prisma/schema.prisma#L216-L230) | createdAt, updatedAt |
| Report | [schema.prisma#L232-L251](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/prisma/schema.prisma#L232-L251) | createdAt, updatedAt |
| Segment | [schema.prisma#L253-L266](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/prisma/schema.prisma#L253-L266) | createdAt, updatedAt |
| Link | [schema.prisma#L288-L307](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/prisma/schema.prisma#L288-L307) | createdAt, updatedAt, deletedAt |
| Pixel | [schema.prisma#L309-L327](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/prisma/schema.prisma#L309-L327) | createdAt, updatedAt, deletedAt |
| Board | [schema.prisma#L329-L347](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/prisma/schema.prisma#L329-L347) | createdAt, updatedAt |
| Share | [schema.prisma#L348-L360](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/prisma/schema.prisma#L348-L360) | createdAt, updatedAt |
| SessionReplaySaved | [schema.prisma#L386-L401](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/prisma/schema.prisma#L386-L401) | createdAt, updatedAt |

### 2.2 站点设置变更追踪

#### 2.2.1 站点可变更字段
- **代码位置**: [route.ts#L37-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/app/api/websites/[websiteId]/route.ts#L37-L104)
- **可变更字段**:
  ```typescript
  name: string           // 站点名称
  domain: string         // 站点域名
  shareId: string        // 共享ID
  replayEnabled: boolean   // 会话重放启用状态
  replayConfig: {         // 会话重放配置
    sampleRate: number    // 采样率 0-1
    maskLevel: 'strict' | 'moderate'  // 脱敏级别
    maxDuration: number // 最大录制时长
    blockSelector: string  // 阻止选择器
  }
  ```

#### 2.2.2 站点数据重置
- **代码位置**: [reset/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/app/api/websites/[websiteId]/reset/route.ts)
- **操作**: 清空站点所有分析数据（session、event、sessionData、eventData、revenue、sessionReplay 等
- **追踪**: 更新 `resetAt` 字段记录重置时间

### 2.3 团队设置变更追踪

#### 2.3.1 团队可变更字段
- **代码位置**: [route.ts#L29-L50](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/app/api/teams/[teamId]/route.ts#L29-L50)
- **可变更字段**:
  ```typescript
  name: string        // 团队名称（最大50字符）
  accessCode: string  // 访问码（最大50字符）
  ```

### 2.4 用户设置变更追踪

#### 2.4.1 用户可变更字段
- **代码位置**: [route.ts#L27-L81](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/app/api/users/[userId]/route.ts#L27-L81)
- **可变更字段**:
  ```typescript
  username: string  // 用户名（仅管理员可改）
  password: string  // 密码（哈希后存储）
  role: string    // 角色（仅管理员可改）
  ```

#### 2.4.2 用户密码变更
- **代码位置**: [password/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/app/api/me/password/route.ts)
- **验证**: 需验证当前密码
- **存储**: 使用 bcrypt 哈希后存储（[hashPassword](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/lib/password.ts)

### 2.5 变更记录局限性

1. **无审计日志表**: 系统未设计专门的 `audit_log` 或 `change_history` 表，无法追踪：
   - 谁做了什么变更
   - 变更前后的值对比
   - 变更的具体时间戳（仅有 `updatedAt` 只能知道最后更新时间）
   - 变更的操作人（仅有 `createdBy` 仅记录创建人）

2. **无变更历史**: 无法回滚或查看历史版本

---

## 3. 敏感字段脱敏

### 3.1 会话重放数据脱敏

#### 3.1.1 脱敏级别配置
- **代码位置**: [recorder/index.js#L87-L99](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/recorder/index.js#L87-L99)
- **配置参数**: `maskLevel`（通过 `data-mask-level` 属性设置）

#### 3.1.2 脱敏级别定义

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

| 级别 | 配置 | 说明 |
|------|------|------|
| **strict** | `maskAllInputs: true, `maskTextSelector: '*' | 所有输入框 + 所有文本内容全部脱敏 |
| **moderate**（默认） | `maskAllInputs: true | 仅所有输入框内容脱敏 |

#### 3.1.3 脱敏实现
- **底层实现**: 基于 rrweb 库的 `maskAllInputs` 和 `maskTextSelector` 配置
- **输入框类型**: 所有 `<input>`、`<textarea>`、`<select>` 等表单元素
- **文本脱敏**: `strict` 模式下所有文本节点内容替换为 `***`

#### 3.1.4 其他录制限制
- **代码位置**: [recorder/index.js#L136-L147](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/recorder/index.js#L136-L147)
- **不录制内容**:
  - `recordCanvas: false` - 不录制 Canvas
  - `recordCrossOriginIframes: false` - 不录制跨域 iframe
  - `slimDOMOptions` - 精简 DOM，移除 script、comment、各类 meta 标签

### 3.2 密码字段保护

#### 3.2.1 查询时排除密码
- **代码位置**: [user.ts#L23-L30](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/prisma/user.ts#L23-L30)
```typescript
select: {
  id: true,
  username: true,
  password: includePassword,  // 默认 false
  role: true,
  createdAt: true,
}
```

#### 3.2.2 API 返回时排除密码
- **代码位置**: [admin/users/route.ts#L35-L37](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/app/api/admin/users/route.ts#L35-L37)
```typescript
omit: {
  password: true,
}
```

#### 3.2.3 密码哈希存储
- **代码位置**: [users/[userId]/route.ts#L56-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/app/api/users/[userId]/route.ts#L56-L58)
```typescript
if (password) {
  data.password = hashPassword(password);
}
```

### 3.3 用户删除时的数据脱敏
- **代码位置**: [user.ts#L129-L146](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/prisma/user.ts#L129-L146)
- **CLOUD_MODE 下用户删除时：
  - 用户名替换为随机字符串：`username: getRandomChars(32)`
  - 标记 `deletedAt` 软删除

---

## 4. 保留策略

### 4.1 数据保留策略现状

**代码分析结论：系统未实现自动化数据保留（Data Retention）策略。**

### 4.2 现有时间相关字段

| 字段 | 用途 | 代码位置 |
|------|------|----------|
| `createdAt` | 记录创建时间 | 所有实体表 |
| `updatedAt` | 记录最后更新时间 | 配置类实体表 |
| `deletedAt` | 软删除标记 | User, Website, Team, Link, Pixel |
| `resetAt` | 站点数据重置时间 | Website 表 |

### 4.3 删除模式

#### 4.3.1 软删除（CLOUD_MODE）
- **触发条件**: `process.env.CLOUD_MODE === true`
- **实现方式**: 设置 `deletedAt = new Date()`
- **影响范围**:
  - User: [user.ts#L129-L146](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/prisma/user.ts#L129-L146)
  - Website: [website.ts#L234-L243](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/prisma/website.ts#L234-L243)
  - Team: [team.ts#L147-L158](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/prisma/team.ts#L147-L158)

#### 4.3.2 硬删除（非 CLOUD_MODE）
- **触发条件**: `process.env.CLOUD_MODE !== true`
- **实现方式**: 物理删除数据库记录

### 4.4 会话超时配置

#### 4.4.1 访问超时
- **代码位置**: [auth.ts#L62-L74](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/lib/auth.ts#L62-L74)
- **配置**: Redis 键过期时间（`expire` 参数）

#### 4.4.2 Visit 超时
- **代码位置**: [send/route.ts#L171](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/app/api/send/route.ts#L171)
- **配置**: Visit 30 分钟后过期

### 4.5 保留报告（Retention Report）

**注意**: 系统中的 "retention" 指的是**用户留存分析报告**，而非数据保留策略。

- **代码位置**: [getRetention.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/sql/reports/getRetention.ts)
- **用途**: 分析用户回访频率，衡量网站用户粘性

### 4.6 ClickHouse 数据保留

- **代码位置**: [schema.sql](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/db/clickhouse/schema.sql)
- **现状**: 未配置 TTL（Time To Live）策略
- **分区**: 按 `toYYYYMM(created_at)` 按月分区
- **存储引擎**: MergeTree 系列引擎，无自动清理机制

### 4.7 手动数据清理

#### 4.7.1 站点数据重置
- **代码位置**: [website.ts#L133-L186](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/prisma/website.ts#L133-L186)
- **清理范围**:
  ```
  sessionReplaySaved
  sessionReplay
  revenue
  eventData
  sessionData
  websiteEvent
  session
  ```
- **触发**: 用户手动操作

#### 4.7.2 站点删除
- **代码位置**: [website.ts#L188-L257](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/prisma/website.ts#L188-L257)
- **清理范围**: 重置范围 + report + segment + share

---

## 5. 导出边界

### 5.1 导出 API

#### 5.1.1 导出接口
- **代码位置**: [export/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/app/api/websites/[websiteId]/export/route.ts)
- **路由**: `GET /api/websites/[websiteId]/export`
- **权限**: `canViewWebsite`（[permissions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/permissions.ts)

#### 5.1.2 导出参数
```typescript
{
  ...pagingParams,    // 分页参数
  ...withDateRange  // 日期范围
}
```

### 5.2 导出内容

| 文件 | 数据来源 | 代码位置 |
|------|----------|----------|
| `events.csv` | 事件指标 | [getEventMetrics](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/sql/events/getEventMetrics.ts) |
| `pages.csv` | 页面浏览指标 | [getPageviewMetrics](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/sql/events/getPageviewMetrics.ts) |
| `referrers.csv` | 来源域名指标 | [getPageviewMetrics](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/sql/events/getPageviewMetrics.ts) |
| `browsers.csv` | 浏览器指标 | [getSessionMetrics](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/sql/sessions/getSessionMetrics.ts) |
| `os.csv` | 操作系统指标 | [getSessionMetrics](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/sql/sessions/getSessionMetrics.ts) |
| `devices.csv` | 设备指标 | [getSessionMetrics](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/sql/sessions/getSessionMetrics.ts) |
| `countries.csv` | 国家指标 | [getSessionMetrics](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/queries/sql/sessions/getSessionMetrics.ts) |

### 5.3 导出格式

- **压缩格式**: ZIP（JSZip 库）
- **文件格式**: CSV（PapaParse 库）
- **传输编码**: Base64 编码的 ZIP 内容
- **返回格式**:
  ```json
  {
    "zip": "base64_encoded_zip_content"
  }
  ```

### 5.4 导出边界限制

#### 5.4.1 不导出的数据

1. **原始事件数据**（`website_event` 表的详细记录）
2. **会话数据**（`session_data`、`event_data`）
3. **会话重放数据**（`session_replay`）
4. **收入数据**（`revenue`）
5. **用户敏感信息**（密码等）
6. **团队管理数据**（团队成员、权限等）

#### 5.4.2 仅导出聚合指标

导出的数据均为**聚合后的指标数据**，而非原始明细数据：
- 事件统计
- 页面浏览统计
- 会话属性统计

### 5.5 权限控制

- **代码位置**: [export/route.ts#L25-L27](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/app/api/websites/[websiteId]/export/route.ts#L25-L27)
```typescript
if (!(await canViewWebsite(auth, websiteId))) {
  return unauthorized();
}
```

---

## 6. 权限与安全

### 6.1 权限检查矩阵

| 操作 | 所需权限 | 代码位置 |
|------|----------|----------|
| 查看站点 | `canViewWebsite` | [permissions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/permissions.ts) |
| 更新站点 | `canUpdateWebsite` | [permissions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/permissions.ts) |
| 删除站点 | `canDeleteWebsite` | [permissions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/permissions.ts) |
| 查看团队 | `canViewTeam` | [permissions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/permissions.ts) |
| 更新团队 | `canUpdateTeam` | [permissions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/permissions.ts) |
| 删除团队 | `canDeleteTeam` | [permissions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/permissions.ts) |
| 查看用户 | `canViewUser` | [permissions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/permissions.ts) |
| 更新用户 | `canUpdateUser` | [permissions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/permissions.ts) |
| 删除用户 | `canDeleteUser` | [permissions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/permissions.ts) |

### 6.2 角色权限定义

- **代码位置**: [constants.ts#L186-L217](file:///d:/fz/0601-2/solo-dogfeeding/code/53-umami/src/lib/constants.ts#L186-L217)

| 角色 | 权限 |
|------|------|
| admin | 所有权限 |
| user | 网站创建/更新/删除、团队创建 |
| team-owner | 团队更新/删除、网站管理、网站转移 |
| team-manager | 团队更新、网站管理 |
| team-member | 网站创建/更新/删除 |
| view-only / team-view-only | 无管理权限 |

---

## 7. 总结与建议

### 7.1 当前覆盖范围总结

| 维度 | 覆盖情况 | 说明 |
|------|----------|------|
| **变更记录 | ⚠️ 部分覆盖 | 仅有 `updatedAt` 时间戳，无审计日志表 |
| **敏感字段脱敏 | ✅ 良好覆盖 | 会话重放两级脱敏，密码哈希保护 |
| **保留策略 | ❌ 未覆盖 | 无自动化数据保留策略 |
| **导出边界 | ✅ 良好控制 | 仅导出聚合指标，不导出原始数据 |

### 7.2 改进建议

1. **增加审计日志表**：
   - 建议新增 `audit_log` 表记录配置变更历史
   - 记录字段：`entity_type`、`entity_id`、`field`、`old_value`、`new_value`、`user_id`、`created_at`

2. **实现数据保留策略**：
   - 配置 ClickHouse TTL 自动清理旧数据
   - 提供可配置的数据保留周期

3. **增加操作人记录**：
   - 在关键操作（删除、重置、权限变更）记录操作人 ID

4. **导出审计**：
   - 记录导出操作日志，包括导出人、导出时间、导出范围

---

**分析日期**: 2026-06-20
**代码版本**: Umami 分析系统
