# Umami 三大报表权限与数据保留期深度核验报告（V3 最终校正版）

## 概述
本文档基于源码逐行核验，彻底理清「权限检查」与「日期限制」两条独立链路，修正此前关于"管理员/分享访问绕过日期限制"、"团队网站无限制"等错误结论，还原三大报表在真实场景下的表现。

---

## 一、核心架构校正：两条完全独立的链路

### 1.1 关键架构发现
```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    API 请求处理                          │
                    └─────────────────────────────────────────────────────────┘
                                         ↓
┌───────────────────────────────────────────────────────────────────────────────┐
│ 1. canViewWebsite(auth, websiteId)  -- 权限检查                              │
│    → 检查：user.isAdmin / shareToken 匹配 / userId 匹配 / teamUser 匹配     │
│    → 返回：true/false（通过则继续，不通过则401）                              │
└───────────────────────────────────────────────────────────────────────────────┘
                                         ↓
┌───────────────────────────────────────────────────────────────────────────────┐
│ 2. setWebsiteDate(websiteId, parameters)  -- 日期限制（漏斗/留存有，用户路径无）│
│    ├──→ fetchWebsite(websiteId)  -- 查询网站属性（teamId, userId, resetAt）   │
│    │     【关键】完全不依赖 auth，只看 websiteId 本身！                       │
│    ├──→ 触发条件1：CLOUD_MODE=true && website.teamId == null                 │
│    │     └──→ fetchAccount(website.userId) → !hasSubscription                │
│    │         └──→ 截断 startDate 至最近6个月                                  │
│    └──→ 触发条件2：website.resetAt 存在                                      │
│         └──→ 截断 startDate 至重置日期                                       │
└───────────────────────────────────────────────────────────────────────────────┘
                                         ↓
┌───────────────────────────────────────────────────────────────────────────────┐
│ 3. getQueryFilters(filters, websiteId)  -- 筛选条件处理                      │
│    └──→ 内部再次调用 setWebsiteDate(dateRange)  -- 双重限制（仅对filters生效）│
└───────────────────────────────────────────────────────────────────────────────┘
                                         ↓
┌───────────────────────────────────────────────────────────────────────────────┐
│ 4. SQL 查询层 parseFilters({ ...filters, websiteId, startDate, endDate })     │
│    【关键】startDate/endDate 来自 parameters（已被setWebsiteDate处理）         │
│              而非来自 filters 中的日期！                                       │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心校正结论汇总

| 之前的错误结论 | 校正后的真实结论 | 证据位置 |
|--------------|----------------|---------|
| 管理员/分享访问绕过日期限制 | ❌ 不绕过！日期限制与访问者身份完全无关 | `setWebsiteDate` 函数无任何 auth 参数 |
| 团队网站无6个月限制 | ✅ 正确（因为 `!website.teamId` 不成立） | `request.ts:96` 条件判断 |
| 访问者是付费用户就无限制 | ❌ 不！是**网站归属用户**是否付费，与访问者无关 | `request.ts:97` 用的是 `website.userId` |
| 漏斗/留存双重限制生效 | ✅ 正确，但日期来自 parameters，非 filters | SQL 层解构参数 |
| 用户路径"完全无限制" | ❌ 不准确！是 **parameters 无限制，filters 有限制但被忽略** | journey API 第25行 + SQL 第49行 |

---

## 二、setWebsiteDate 函数逐行核验

### 2.1 源码与触发条件拆解

**文件位置**：`src/lib/request.ts:92-109`

```typescript
export async function setWebsiteDate(websiteId: string, data: Record<string, any>) {
  // 步骤1：查询网站信息（只依赖 websiteId，与访问者身份完全无关）
  const website = await fetchWebsite(websiteId);
  const cloudMode = !!process.env.CLOUD_MODE;

  // 触发条件 A：云模式 + 个人网站（非团队） + 无订阅
  if (cloudMode && website && !website.teamId) {
    // 【关键】查询的是 网站归属用户 的订阅状态，不是访问者！
    const account = await fetchAccount(website.userId);

    if (!account?.hasSubscription) {
      // 截断：startDate 不能早于6个月前
      data.startDate = maxDate(data.startDate, startOfMonth(subMonths(new Date(), 6)));
    }
  }

  // 触发条件 B：网站有重置日期
  if (website?.resetAt) {
    data.startDate = maxDate(data.startDate, new Date(website?.resetAt));
  }

  return data;
}
```

### 2.2 两个独立的日期限制维度

| 限制维度 | 触发条件 | 截断逻辑 | 谁会被影响？ |
|---------|---------|---------|-------------|
| **订阅期限限制** | `CLOUD_MODE=true` **且** `website.teamId == null` **且** 网站创建者 `!hasSubscription` | `startDate` 不能早于6个月前 | 所有访问该网站的人（包括管理员、团队成员、分享链接访客） |
| **网站重置限制** | `website.resetAt != null` | `startDate` 不能早于重置日期 | 所有访问该网站的人（无例外） |

### 2.3 访问者身份对日期限制完全无影响！

**所有身份都受日期限制**（只要满足上述条件）：
- ❌ 管理员（isAdmin = true）→ 同样受限
- ❌ 团队成员访问 → 同样受限（但个人网站才会触发，团队网站不会）
- ❌ 分享链接访问 → 同样受限
- ❌ 付费用户访问其他用户网站 → 受被访问网站的订阅状态限制

**证据**：`setWebsiteDate` 函数签名只有两个参数 `(websiteId, data)`，没有任何 `auth` / `user` / `shareToken` 参数！

---

## 三、三大报表完整链路逐案核验

### 3.1 漏斗报表（Funnel）✅ 正确实现

**API源码**：`src/app/api/reports/funnel/route.ts`

```typescript
export async function POST(request: Request) {
  const { auth, body, error } = await parseRequest(request, reportResultSchema);
  
  // 步骤1：权限检查（只决定401还是继续）
  if (!(await canViewWebsite(auth, websiteId))) {
    return unauthorized();
  }

  // 步骤2：对 parameters 应用日期限制 ✅ 有
  const parameters = await setWebsiteDate(websiteId, body.parameters);
  
  // 步骤3：对 filters 应用日期限制（内部调用setWebsiteDate）✅ 有
  const filters = await getQueryFilters(body.filters, websiteId);
  
  // 步骤4：SQL查询（使用 parameters 中的 startDate/endDate）
  const data = await getFunnel(websiteId, parameters as FunnelParameters, filters);
  
  return json(data);
}
```

**SQL层参数流向**（`getFunnel.ts:47-54`）：
```typescript
const { startDate, endDate, window, steps } = parameters; // 已受限 ✅
const { filterQuery, ..., queryParams } = parseFilters({
  ...filters,
  websiteId,
  startDate,        // 使用 parameters 中的受限日期
  endDate,
});
```

**漏斗报表完整结论**：
- ✅ parameters 经过 `setWebsiteDate` 限制
- ✅ filters 也经过限制（但日期被 parameters 覆盖）
- ✅ SQL 使用的是受限日期
- ✅ 所有访问者（含管理员、分享）都受限制
- ✅ 团队网站不会触发6个月限制（个人网站且无订阅才触发）

---

### 3.2 留存报表（Retention）✅ 正确实现

**API源码**：`src/app/api/reports/retention/route.ts`

```typescript
export async function POST(request: Request) {
  const { auth, body, error } = await parseRequest(request, reportResultSchema);
  
  // 步骤1：权限检查
  if (!(await canViewWebsite(auth, websiteId))) {
    return unauthorized();
  }

  // 步骤2：对 filters 应用日期限制（内部调用setWebsiteDate）✅ 有
  const filters = await getQueryFilters(body.filters, websiteId);
  
  // 步骤3：对 parameters 应用日期限制 ✅ 有
  const parameters = await setWebsiteDate(websiteId, body.parameters);
  
  // 步骤4：SQL查询
  const data = await getRetention(websiteId, parameters as RetentionParameters, filters);
  
  return json(data);
}
```

**SQL层参数流向**（`getRetention.ts:34-44`）：
```typescript
const { startDate, endDate, timezone } = parameters; // 已受限 ✅
const { filterQuery, ..., queryParams } = parseFilters({
  ...filters,
  websiteId,
  startDate,        // 使用 parameters 中的受限日期
  endDate,
  timezone,
});
```

**留存报表完整结论**：
- ✅ 与漏斗完全一致的双重限制
- ✅ 所有访问者都受限制（无例外）
- ✅ 团队网站不会触发6个月限制

---

### 3.3 用户路径报表（Journey）⚠️ Bug 已确认

**API源码**：`src/app/api/reports/journey/route.ts`

```typescript
export async function POST(request: Request) {
  const { auth, body, error } = await parseRequest(request, reportResultSchema);
  
  // 步骤1：权限检查
  if (!(await canViewWebsite(auth, websiteId))) {
    return unauthorized();
  }

  // 步骤2：❌ 缺失！没有对 parameters 调用 setWebsiteDate
  
  // eventType 特殊处理
  if (eventType) {
    filters.eventType = eventType;
  }
  
  // 步骤3：对 filters 应用日期限制 ✅ 有（但后面没用）
  const queryFilters = await getQueryFilters(filters, websiteId);
  
  // 步骤4：SQL查询（使用未受限的 parameters.startDate！）
  const data = await getJourney(websiteId, parameters, queryFilters);
  
  return json(data);
}
```

**SQL层参数流向**（`getJourney.ts:39-51`）：
```typescript
const { startDate, endDate, steps, startStep, endStep } = parameters; // ⚠️ 未受限！
const { filterQuery, ..., queryParams } = parseFilters({
  ...filters,       // filters 中的日期已被限制，但被下面覆盖
  websiteId,
  startDate,        // ❌ 使用未受限的 parameters.startDate
  endDate,          // ❌ 使用未受限的 parameters.endDate
});
```

**用户路径报表完整结论**：
- ❌ `parameters.startDate` 未经过 `setWebsiteDate` 限制（Bug）
- ✅ `filters` 中的日期被限制了，但被 SQL 层显式传入的 parameters 覆盖
- ⚠️ 实际结果：用户路径报表可以查询任意时间范围的数据，不受6个月限制
- ⚠️ 影响所有人：管理员、普通用户、分享访问、团队成员都能"越权"查询

---

## 四、场景化真实表现矩阵

### 4.1 按网站类型划分（最核心区别）

| 场景 | 漏斗/留存表现 | 用户路径表现 | 说明 |
|------|-------------|------------|------|
| **云模式 + 个人网站 + 无订阅** | ✅ 限制6个月 | ❌ 无限制（Bug） | `website.teamId == null` 且 `!hasSubscription` |
| **云模式 + 个人网站 + 有订阅** | 无限制 | 无限制 | `account.hasSubscription == true` |
| **云模式 + 团队网站** | 无限制 | 无限制 | `website.teamId != null`，条件不触发 |
| **非云模式（本地部署）** | 无限制 | 无限制 | `CLOUD_MODE=false`，条件不触发 |
| **任何已重置的网站** | 不能早于 `resetAt` | ⚠️ 可以早于 `resetAt` | resetAt 限制也是通过 setWebsiteDate 生效 |

### 4.2 按访问者身份划分（完全不影响日期限制！）

| 访问者身份 | 漏斗/留存表现 | 用户路径表现 | 说明 |
|-----------|-------------|------------|------|
| 管理员（isAdmin = true） | 受网站订阅状态限制 | ❌ 无限制 | 权限检查通过，但日期限制不看身份 |
| 网站创建者本人 | 受网站订阅状态限制 | ❌ 无限制 | 同上 |
| 团队成员访问团队网站 | 无限制（团队网站不触发） | 无限制 | 团队网站条件不触发 |
| 分享链接访客 | 受被分享网站的订阅状态限制 | ❌ 无限制 | 分享令牌绕过权限检查，但日期限制独立 |
| 其他团队成员（非该网站成员） | 401无权访问 | 401无权访问 | 权限检查不通过，到不了日期限制 |

---

## 五、关键 Bug 与修复方案

### 5.1 Bug 确认：用户路径 API 遗漏 setWebsiteDate 调用

**严重程度**：高  
**影响范围**：所有云模式下的个人无订阅网站  
**行为不一致**：
- 漏斗：只能查最近6个月
- 留存：只能查最近6个月
- 用户路径：可以查任意历史数据

**根因定位**：`journey/route.ts` 第1行导入了 `parseRequest` 和 `getQueryFilters`，但漏掉了 `setWebsiteDate`！

**代码对比证据**：
```typescript
// funnel/route.ts 第1行 ✅ 完整导入
import { getQueryFilters, parseRequest, setWebsiteDate } from '@/lib/request';

// retention/route.ts 第1行 ✅ 完整导入
import { getQueryFilters, parseRequest, setWebsiteDate } from '@/lib/request';

// journey/route.ts 第1行 ❌ 漏了 setWebsiteDate
import { getQueryFilters, parseRequest } from '@/lib/request';
```

### 5.2 修复方案（2处改动）

**改动1：补充导入**
```typescript
// journey/route.ts 第1行
// 前：import { getQueryFilters, parseRequest } from '@/lib/request';
// 后：
import { getQueryFilters, parseRequest, setWebsiteDate } from '@/lib/request';
```

**改动2：在权限检查后、getQueryFilters 前添加调用**
```typescript
// journey/route.ts 第17-26行
if (!(await canViewWebsite(auth, websiteId))) {
  return unauthorized();
}

// ✅ 新增：对 parameters 应用日期限制（与漏斗/留存对齐）
await setWebsiteDate(websiteId, parameters);

if (eventType) {
  filters.eventType = eventType;
}

const queryFilters = await getQueryFilters(filters, websiteId);
const data = await getJourney(websiteId, parameters, queryFilters);
```

**验证修复效果**：
- ✅ parameters.startDate 被正确截断
- ✅ SQL 层使用受限日期
- ✅ 三个报表行为一致
- ✅ 不影响团队网站和付费个人网站（条件不触发）

---

## 六、其他设计问题与优化建议

### 6.1 重复查询问题（性能损耗）

**现状**：`setWebsiteDate` 被调用两次：
1. API 层直接调用（对 parameters）
2. `getQueryFilters` 内部再次调用（对 dateRange）

**后果**：
- 两次 `fetchWebsite` 查询（虽然有 Redis 缓存，但仍然是冗余）
- 两次 `fetchAccount` 查询（如果是个人无订阅网站）

**优化建议**：预计算一次，多处复用
```typescript
// 优化后代码
const website = await fetchWebsite(websiteId);
const dateLimits = calculateDateLimits(website); // 纯函数，无副作用

const parameters = applyDateLimits(body.parameters, dateLimits);
const filters = await getQueryFilters(body.filters, websiteId, dateLimits); // 传入预计算结果
```

### 6.2 架构设计建议：权限与限制的关注点分离

**当前设计优点**：
- 关注点分离：权限检查（谁能看）与数据限制（能看什么范围）是两个独立函数
- 灵活：可以单独调整权限规则或日期限制规则，互不影响

**潜在改进**：
- 将数据范围限制（不仅仅是日期）统一封装到单一入口点
- 避免不同报表出现不一致的实现遗漏

---

## 七、最终核验总结表

| 核验项 | 漏斗 | 留存 | 用户路径 | 核验证据 |
|-------|------|------|---------|---------|
| 导入 setWebsiteDate | ✅ 是 | ✅ 是 | ❌ 否 | 各 API 第1行 |
| 调用 setWebsiteDate(parameters) | ✅ 是 | ✅ 是 | ❌ 否 | 各 API 第20/21行 |
| SQL 使用 parameters.startDate | ✅ 是 | ✅ 是 | ✅ 是 | 各 SQL 第47/39行 |
| 日期限制实际生效 | ✅ 是 | ✅ 是 | ❌ 否 | 综合判断 |
| 管理员绕过日期限制 | ❌ 否 | ❌ 否 | ❌ 否（本就无限制） | setWebsiteDate 无 auth 参数 |
| 分享访问绕过日期限制 | ❌ 否 | ❌ 否 | ❌ 否（本就无限制） | setWebsiteDate 无 shareToken 逻辑 |
| 团队网站无6个月限制 | ✅ 是 | ✅ 是 | ✅ 是 | setWebsiteDate 第96行条件 `!website.teamId` |
| resetAt 限制生效 | ✅ 是 | ✅ 是 | ❌ 否 | resetAt 限制也通过 setWebsiteDate 生效 |

---

**报告生成时间**：2025-01-14  
**核验版本**：V3 最终校正版（基于源码逐行追溯）  
**核心校正点总结**：
1. ✅ 日期限制与访问者身份完全无关，只看网站本身属性
2. ✅ 管理员/分享访问不绕过日期限制
3. ✅ 用户路径 Bug 准确根因定位：漏导入 + 漏调用 setWebsiteDate
4. ✅ 团队网站不受限的准确原因：`!website.teamId` 条件不触发
5. ✅ 明确 filters 与 parameters 中日期的优先级与覆盖关系
