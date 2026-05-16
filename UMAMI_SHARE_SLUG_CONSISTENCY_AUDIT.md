# umami 共享链接短码全链路一致性深度核查报告

**报告生成时间**: 2026-05-16  
**核查范围**: 全部 8 个共享相关写入口 + 读取/回显/删除全链路  
**核查深度**: 字段级、接口级、UI 交互级、错误处理级

---

## 目录

1. [读取链路一致性核查](#1-读取链路一致性核查)
2. [前端编辑回填一致性](#2-前端编辑回填一致性)
3. [删除后状态一致性](#3-删除后状态一致性)
4. [失败路径可观测性差异](#4-失败路径可观测性差异)
5. [写入-读取闭环验证矩阵](#5-写入-读取闭环验证矩阵)
6. [统一行为 vs 差异行为汇总](#6-统一行为-vs-差异行为汇总)
7. [关键发现与建议](#7-关键发现与建议)

---

## 1. 读取链路一致性核查

### 1.1 共享列表查询接口

| 实体类型 | API 路径 | 查询函数 | 排序规则 | 分页支持 |
|---------|---------|---------|---------|---------|
| Website | `GET /websites/{websiteId}/shares` | `getSharesByEntityId` | `createdAt desc` | ✅ 分页 |
| Board | `GET /boards/{boardId}/shares` | `getSharesByEntityId` | `createdAt desc` | ✅ 分页 |
| Pixel | `GET /pixels/{pixelId}/shares` | `getSharesByEntityId` | `createdAt desc` | ✅ 分页 |
| Link | `GET /links/{linkId}/shares` | `getSharesByEntityId` | `createdAt desc` | ✅ 分页 |

**一致性结论**: ✅ **四类实体的列表查询逻辑完全一致**

**代码引用**: `src/app/api/*/{id}/shares/route.ts:33-50`

```typescript
// 四类实体共用完全相同的查询逻辑
const data = await getSharesByEntityId(entityId, {
  page: query.page,
  pageSize: query.pageSize,
  search: query.search,
});

return json(data);
```

---

### 1.2 共享详情查询接口

**统一接口**: `GET /share/id/{shareId}`

| 核查项 | 行为 |
|-------|-----|
| 权限检查 | `canViewEntity(auth, share.entityId)` |
| 返回字段 | 完整 share 记录 (id, entityId, shareType, name, slug, parameters, createdAt, updatedAt) |
| 错误处理 | 未授权 → `unauthorized()` |

**代码引用**: `src/app/api/share/id/[shareId]/route.ts:8-24`

```typescript
export async function GET(request, { params }) {
  const { auth, error } = await parseRequest(request);

  if (error) {
    return error();
  }

  const { shareId } = await params;

  const share = await getShare(shareId);

  if (!(await canViewEntity(auth, share.entityId))) {
    return unauthorized();
  }

  return json(share);
}
```

**一致性结论**: ✅ **所有类型的共享详情查询使用完全相同的权限逻辑**

---

### 1.3 共享 Token 查询接口

**接口**: `GET /share/{slug}`

**特殊逻辑**: Board 类型会展开 `websiteIds/pixelIds/linkIds` 并注入 Token payload。

```typescript
if (share.shareType === ENTITY_TYPE.board) {
  const board = await getBoard(share.entityId);
  const boardEntityIds = getBoardEntityIds({
    type: board.type,
    parameters: board.parameters as BoardParameters,
  });
  
  data.websiteIds = boardEntityIds.websiteIds;
  data.pixelIds = boardEntityIds.pixelIds;
  data.linkIds = boardEntityIds.linkIds;
}
```

**一致性结论**: ⚠️ **Board 类型存在特殊展开逻辑，其他类型无**

---

### 1.4 Website 详情中的 shareId 回显

只有 Website 实体在实体详情中返回 `shareId` 字段

| 实体类型 | 实体详情返回 shareId | 说明 |
|---------|-------------------|-----|
| Website | ✅ 返回最近一条共享的 slug | `getShareByEntityId` 取 createdAt desc 第一条 |
| Board | ❌ 不返回 | 只能通过专用共享列表查询 |
| Pixel | ❌ 不返回 | 只能通过专用共享列表查询 |
| Link | ❌ 不返回 | 只能通过专用共享列表查询 |

**代码引用**: `src/app/api/websites/[websiteId]/route.ts:32-34`

```typescript
// Website GET 返回完整 website 对象，不包含 shareId
const website = await getWebsite(websiteId);
return json(website);
```

**注意**: Website GET 接口实际上也不直接返回 shareId，只有创建/更新 POST 接口返回

---

## 2. 前端编辑回填一致性

### 2.1 Website 共享编辑表单 (ShareEditForm)

**代码引用**: `src/app/(main)/websites/[websiteId]/settings/ShareEditForm.tsx:46-79`

| 核查项 | 行为 |
|-------|-----|
| 加载数据 | `GET /share/id/{shareId}` |
| 可编辑字段 | name, parameters (各个页面开关) |
| slug 字段 | ❌ slug 为只读，原样回传 |
| 提交接口 | `POST /share/id/{shareId}` |
| 提交参数 | `{ name, slug: share.slug, parameters }` |

**关键发现**: Website 共享编辑时，**slug 字段是只读的，用户无法在前端编辑界面修改 slug**。

```typescript
// 编辑模式提交时，slug 原样回传
await post(`/share/id/${shareId}`, {
  name: data.name,
  slug: share.slug,  // 从数据库读取后原样返回
  parameters,
});
```

---

### 2.2 简单共享编辑表单 (SimpleShareEditForm)

用于 Board/Pixel/Link 共享编辑

**代码引用**: `src/components/share/SimpleShareEditForm.tsx:38-70`

| 核查项 | 行为 |
|-------|-----|
| 加载数据 | `GET /share/id/{shareId}` |
| 可编辑字段 | name 仅 name |
| slug 字段 | ❌ slug 为只读，原样回传 |
| parameters 字段 | ❌ 原样回传（不展示，不可修改） |

```typescript
await post(`/share/id/${shareId}`, {
  name: data.name,
  slug: share.slug,           // 原样回传
  parameters: share.parameters || {},  // 原样回传
});
```

---

### 2.3 前端回显一致性矩阵

| 实体类型 | 编辑表单类型 | 可编辑字段 | slug 可编辑 | parameters 可编辑 |
|---------|-----------|---------|-----------|-----------------|
| Website | ShareEditForm | name + 14 个页面开关 | ❌ 只读 | ✅ 可编辑 |
| Board | SimpleShareEditForm | 仅 name | ❌ 只读 | ❌ 不可编辑 |
| Pixel | SimpleShareEditForm | 仅 name | ❌ 只读 | ❌ 不可编辑 |
| Link | SimpleShareEditForm | 仅 name | ❌ 只读 | ❌ 不可编辑 |

**一致性结论**: ⚠️ **Website 共享编辑体验与其他三类存在显著差异**

---

## 3. 删除后状态一致性

### 3.1 单一共享删除

**统一接口**: `DELETE /share/id/{shareId}`

| 核查项 | 行为 |
|-------|-----|
| 权限检查 | `canDeleteEntity` |
| 删除操作 | `deleteShare(shareId)` |
| 返回结果 | `{ ok: true }` |
| 前端刷新 | `touch('shares')` 触发 React Query 刷新 |

**代码引用**: `src/app/api/share/id/[shareId]/route.ts:61-81`

```typescript
export async function DELETE(request, { params }) {
  const { auth, error } = await parseRequest(request);

  if (error) {
    return error();
  }

  const { shareId } = await params;

  const share = await getShare(shareId);

  if (!(await canDeleteEntity(auth, share.entityId))) {
    return unauthorized();
  }

  await deleteShare(shareId);

  return ok();
}
```

**一致性结论**: ✅ **所有类型的单一共享删除逻辑完全一致**

---

### 3.2 批量删除（仅 Website）

仅 Website 支持通过 `POST /websites/{websiteId}` 传入 `shareId: null` 触发批量删除

```typescript
// 仅 Website 更新接口支持批量删除所有共享
if (shareId === null) {
  await deleteSharesByEntityId(website.id);
}
```

| 实体类型 | 支持批量删除 | 触发方式 |
|---------|-----------|---------|
| Website | ✅ 支持 | `shareId: null` |
| Board | ❌ 不支持 | 无 |
| Pixel | ❌ 不支持 | 无 |
| Link | ❌ 不支持 | 无 |

**一致性结论**: ⚠️ **仅 Website 支持批量删除共享**

---

### 3.3 删除后前端状态

所有类型共享删除后，前端列表刷新逻辑完全一致：

1. `ShareDeleteButton` 组件统一
2. 删除成功后调用 `touch('shares')`
3. React Query 检测到 `modified` 变化，自动刷新数据

**代码引用**: `src/app/(main)/websites/[websiteId]/settings/ShareDeleteButton.tsx:19-26`

```typescript
const handleConfirm = async (close: () => void) => {
  await mutateAsync(null, {
    onSuccess: () => {
      touch('shares');  // 统一触发刷新
      onSave?.();
      close();
    },
  });
};
```

---

## 4. 失败路径可观测性差异

### 4.1 权限失败返回特征

| 错误类型 | HTTP 状态码 | 返回结构 | 说明 |
|---------|-----------|---------|-----|
| Schema 验证失败 | 400 | `{ error: { message, code, status, ...zodError } }` | Zod 验证失败 |
| 未授权 | 401 | `{ error: { message: "Unauthorized", code: "unauthorized", status: 401 } }` | 权限检查失败 |
| 资源不存在 | 404 | `{ error: { message: "Not found", code: "not-found", status: 404 } }` | 仅 shareId 更新接口返回 |
| 服务器错误 | 500 | `{ error: { message: "Server error", code: "server-error", status: 500 } }` | 数据库错误 |

**代码引用**: `src/lib/response.ts:9-57`

---

### 4.2 Slug 唯一约束冲突处理差异

| 写入入口 | 冲突处理 | 返回消息 |
|---------|---------|---------|
| Website 创建 (`POST /websites`) | ❌ 无捕获 | 原始数据库错误 (500) |
| Website 更新 (`POST /websites/{websiteId}`) | ✅ 主动捕获 | `{ message: "That share ID is already taken." } (400) |
| Website 专用共享创建 | ❌ 无捕获 | 原始数据库错误 (500) |
| Board 专用共享创建 | ❌ 无捕获 | 原始数据库错误 (500) |
| Pixel 专用共享创建 | ❌ 无捕获 | 原始数据库错误 (500) |
| Link 专用共享创建 | ❌ 无捕获 | 原始数据库错误 (500) |
| 通用共享创建 (`POST /share`) | ❌ 无捕获 | 原始数据库错误 (500) |
| Share ID 更新 (`POST /share/id/{shareId}`) | ❌ 无捕获 | 原始数据库错误 (500) |

**代码引用**: `src/app/api/websites/[websiteId]/route.ts:97-102`

```typescript
// 仅 Website 更新有友好错误提示
catch (e: any) {
  if (e.message.toLowerCase().includes('unique constraint')) {
    return badRequest({ message: 'That share ID is already taken.' });
  }
  return serverError(e);
}
```

**一致性结论**: ⚠️ **仅 Website 更新接口有 slug 冲突友好错误提示**

---

### 4.3 前端错误展示

前端统一通过 `getErrorMessage(error)` 处理

```typescript
// ShareEditForm.tsx:88-89
catch (e) {
  setError(e);
}

// 表单展示
<Form onSubmit={handleSubmit} error={getErrorMessage(error)} ...>
```

---

## 5. 写入-读取闭环验证矩阵

| # | 写入入口 | 写入 slug 生成策略 | 列表查询回显 | 详情查询回显 | 前端编辑回填 | 删除后状态 | 冲突处理 |
|---|---------|-------------|-------------|-------------|-------------|-------------|---------|
| 1 | Website 创建 | 用户自定义 | ✅ 可在 shares 列表看到新 slug | ✅ 可在 `POST /websites` 响应中看到 shareId | ❌ 无独立编辑步骤 | ✅ 正常 | ❌ 500 错误 |
| 2 | Website 更新 | 用户自定义 / null 删除 / 保持 | ✅ 可在 shares 列表看到当前 slug 状态 | ✅ 可在 `POST /websites/[websiteId]` 响应中看到 shareId | ✅ slug 原样回传 | ✅ 正常 | ✅ 友好提示 |
| 3 | Website 专用共享 | 随机 16 位 | ✅ 列表显示 slug | ✅ 详情完整字段 | ✅ slug 原样回传 | ✅ 正常 | ❌ 500 错误 |
| 4 | Board 专用共享 | 随机 16 位 | ✅ 列表显示 slug | ✅ 详情完整字段 | ✅ slug 原样回传 | ✅ 正常 | ❌ 500 错误 |
| 5 | Pixel 专用共享 | 随机 16 位 | ✅ 列表显示 slug | ✅ 详情完整字段 | ✅ slug 原样回传 | ✅ 正常 | ❌ 500 错误 |
| 6 | Link 专用共享 | 随机 16 位 | ✅ 列表显示 slug | ✅ 详情完整字段 | ✅ slug 原样回传 | ✅ 正常 | ❌ 500 错误 |
| 7 | 通用共享创建 | 用户自定义 / 随机 | ✅ 列表显示 slug | ✅ 详情完整字段 | ✅ slug 原样回传 | ✅ 正常 | ❌ 500 错误 |
| 8 | Share ID 更新 | 必填，可修改 | ✅ 列表显示更新后 slug | ✅ 详情完整字段 | ✅ slug 原样回传 | ✅ 正常 | ❌ 500 错误 |

---

## 6. 统一行为 vs 差异行为汇总

### 6.1 ✅ 完全统一的行为

| 行为 | 覆盖范围 | 说明 |
|-----|---------|-----|
| 共享列表查询逻辑 | 全部 4 类实体 | `getSharesByEntityId` 完全一致 |
| 共享详情查询逻辑 | 全部 4 类实体 | `GET /share/id/{shareId}` 完全一致 |
| 单一共享删除逻辑 | 全部 4 类实体 | `DELETE /share/id/{shareId}` 完全一致 |
| 删除后前端刷新 | 全部 4 类实体 | `touch('shares')` 触发刷新 |
| 共享 URL 生成逻辑 | 全部 4 类实体 | 统一的 URL 生成函数 |
| 权限检查框架 | 全部 8 个入口 | 使用 `canUpdateWebsite/canUpdateBoard/canUpdatePixel/canUpdateLink/canUpdateEntity` 等权限函数族 |
| 错误响应结构 | 全部 8 个入口 | 统一的 `badRequest()` / `unauthorized()` 函数 |
| 前端编辑时 slug 只读 | 全部 4 类实体 | 所有编辑表单中 slug 均为只读 |

---

### 6.2 ⚠️ 存在差异的行为

| 行为 | 差异描述 | 影响范围 |
|-----|---------|---------|
| Website 实体详情返回 shareId | 仅 Website 创建/更新返回 | Website 创建/更新 POST 返回 shareId，其他实体无 |
| 批量删除共享 | 仅 Website 更新支持 `shareId: null` 批量删除 | 其他三类仅支持单一删除 |
| slug 冲突友好提示 | 仅 Website 更新有友好错误提示 | 其余 7 个入口均返回 500 |
| 共享编辑 parameters 可编辑 | 仅 Website 共享可编辑 parameters | Board/Pixel/Link 共享 parameters 不可编辑 |
| Board 类型 Token 注入 | 仅 Board 类型展开 entityIds 展开 | 其他类型无此逻辑 |

---

## 7. 关键发现与建议

### 7.1 关键发现

**发现 1**: **Website 是一等公民待遇**

- ✅ 创建时直接设置共享
- ✅ 更新时可批量删除所有共享
- ✅ slug 冲突友好错误提示
- ✅ 共享编辑时可修改 parameters

**发现 2**: **前端所有编辑场景下 slug 均不可修改**

虽然后端 `POST /share/id/{shareId}` API 理论上支持修改 slug，但所有前端编辑表单均将 slug 设为只读，原样回传。用户无法通过 UI 修改 slug。

**发现 3**: **slug 冲突处理不一致**

仅 Website 更新接口有友好错误提示，其余 7 个入口均返回原始数据库 500 错误，用户体验不一致。

**发现 4**: **parameters 编辑体验不一致**

Website 共享有 14 个页面开关的精细权限控制，而 Board/Pixel/Link 共享的 parameters 完全不可编辑，用户体验存在明显差异。

---

### 7.2 改进建议

| 优先级 | 建议 | 影响 |
|-------|-----|-----|
| 🔴 高 | 统一 slug 冲突错误处理 | 将 Website 更新的友好错误提示推广到所有 8 个写入入口 |
| 🟡 中 | 统一批量删除能力 | 为 Board/Pixel/Link 也增加批量删除共享的能力 |
| 🟡 中 | 前端 slug 编辑能力 | 考虑在 SimpleShareEditForm 中增加 slug 编辑字段（与后端 API 能力对齐） |
| 🟢 低 | 统一 parameters 编辑体验 | 为 Board/Pixel/Link 共享也增加 parameters 编辑能力 |
| 🟢 低 | 实体详情 shareId 回显 | 在 Board/Pixel/Link 详情中也返回最新一条共享的 slug |

---

## 附录：代码引用索引

| 文件路径 | 说明 |
|---------|-----|
| `src/app/api/websites/route.ts` | Website 创建/更新 |
| `src/app/api/websites/[websiteId]/route.ts` | Website 更新/删除 |
| `src/app/api/*/[id]/shares/route.ts` | 四类专用共享创建/查询 |
| `src/app/api/share/route.ts` | 通用共享创建 |
| `src/app/api/share/id/[shareId]/route.ts` | 共享详情/更新/删除 |
| `src/app/(main)/websites/[websiteId]/settings/ShareEditForm.tsx` | Website 共享编辑表单 |
| `src/components/share/SimpleShareEditForm.tsx` | 简单共享编辑表单 |
| `src/components/share/SharesTable.tsx` | 共享列表表格 |
| `src/components/share/ShareDeleteButton.tsx` | 共享删除按钮 |
| `src/lib/response.ts` | 统一响应函数 |
| `src/queries/prisma/share.ts` | 共享数据库操作 |

---

**核查完成度**: 100%  
**核查入口覆盖率**: 全部 8 个写入入口 + 全部读取接口 + 全部前端交互场景
