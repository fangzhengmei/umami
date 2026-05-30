# 访问来源处理流程深度分析

## 概述

本文档详细分析 umami 系统中访问来源（referrer）从采集、归一化、识别到最终映射为可观察渠道类别的完整处理流程。

---

## 一、采集端：数据采集与初步归一化

### 1.1 Tracker 端采集

**文件位置**: `src/tracker/index.js`

采集端在浏览器端执行，负责收集访问来源的原始数据：

```javascript
const { referrer } = document;
let currentRef = normalize(referrer.startsWith(origin) ? '' : referrer);
```

**核心逻辑**:
- 从 `document.referrer` 获取原始来源地址
- 如果来源与当前站点同源，则设为空字符串（排除站内跳转）
- 通过 `normalize()` 函数标准化 URL 格式

### 1.2 URL 标准化函数

```javascript
const normalize = raw => {
  if (!raw) return raw;
  try {
    const u = new URL(raw, location.href);
    if (excludeSearch) u.search = '';
    if (excludeHash) u.hash = '';
    return u.toString();
  } catch {
    return raw;
  }
};
```

**功能**:
- 解析相对 URL 为绝对 URL
- 可选移除查询参数和哈希值
- 异常处理：解析失败时返回原始值

### 1.3 数据上报

```javascript
const getPayload = () => ({
  website,
  screen,
  language,
  title: document.title,
  hostname,
  url: currentUrl,
  referrer: currentRef,  // 归一化后的来源地址
  tag,
  id: identity ? identity : undefined,
});
```

---

## 二、服务端：字段解析与存储

### 2.1 接收路由

**文件位置**: `src/app/api/send/route.ts`

服务端接收数据后，对 referrer 进行进一步解析：

```javascript
if (referrer) {
  const referrerUrl = new URL(referrer, base);
  referrerPath = referrerUrl.pathname;
  referrerQuery = referrerUrl.search.substring(1);
  referrerDomain = referrerUrl.hostname.replace(/^www\./, '');  // 移除 www. 前缀
}
```

**字段拆分**:
| 字段 | 说明 | 示例 |
|------|------|------|
| `referrerPath` | 来源路径 | `/blog/post-1` |
| `referrerQuery` | 查询参数 | `utm_source=google` |
| `referrerDomain` | 来源域名（已去 www.） | `google.com` |

### 2.2 UTM 参数与 Click ID 提取

```javascript
// UTM Params
const utmSource = currentUrl.searchParams.get('utm_source');
const utmMedium = currentUrl.searchParams.get('utm_medium');
const utmCampaign = currentUrl.searchParams.get('utm_campaign');
const utmContent = currentUrl.searchParams.get('utm_content');
const utmTerm = currentUrl.searchParams.get('utm_term');

// Click IDs
const gclid = currentUrl.searchParams.get('gclid');      // Google Ads
const fbclid = currentUrl.searchParams.get('fbclid');    // Facebook
const msclkid = currentUrl.searchParams.get('msclkid');  // Microsoft
const ttclid = currentUrl.searchParams.get('ttclid');    // TikTok
const lifatid = currentUrl.searchParams.get('li_fat_id'); // LinkedIn
const twclid = currentUrl.searchParams.get('twclid');    // Twitter
```

**重要说明**: 这些参数从**目标页面 URL** 中提取，而非 referrer URL。这意味着即使用户直接访问带有 UTM 参数的链接，也能被正确识别。

### 2.3 数据存储

**文件位置**: `src/queries/sql/events/saveEvent.ts`

存储到 `website_event` 表的关键字段：

| 数据库字段 | 说明 |
|-----------|------|
| `referrer_domain` | 来源域名 |
| `referrer_path` | 来源路径 |
| `referrer_query` | 来源查询参数 |
| `utm_source` | UTM 来源 |
| `utm_medium` | UTM 媒介 |
| `utm_campaign` | UTM 活动 |
| `utm_content` | UTM 内容 |
| `utm_term` | UTM 关键词 |
| `gclid`, `fbclid`, 等 | 各平台 Click ID |
| `hostname` | 当前网站域名 |

---

## 三、识别规则：渠道映射配置

**文件位置**: `src/lib/constants.ts`

### 3.1 域名分类规则

系统维护了多组域名分类列表，用于识别访问来源类型：

#### 3.1.1 搜索引擎域名 (SEARCH_DOMAINS)
```javascript
export const SEARCH_DOMAINS = [
  'baidu.com',
  'bing.com',
  'chatgpt.com',
  'duckduckgo.com',
  'ecosia.org',
  'google.',      // 包含 google.com, google.co.jp 等
  'msn.com',
  'perplexity.ai',
  'search.brave.com',
  'yandex.',
];
```

#### 3.1.2 社交媒体域名 (SOCIAL_DOMAINS)
```javascript
export const SOCIAL_DOMAINS = [
  'bsky.app',
  'facebook.com',
  'fb.com',
  'ig.com',
  'instagram.com',
  'linkedin.',
  'news.ycombinator.com',
  'pinterest.',
  'reddit.',
  'snapchat.',
  't.co',
  'threads.net',
  'tiktok.',
  'twitter.com',
  'x.com',
];
```

#### 3.1.3 购物平台域名 (SHOPPING_DOMAINS)
```javascript
export const SHOPPING_DOMAINS = [
  'alibaba.com',
  'aliexpress.com',
  'amazon.',
  'bestbuy.com',
  'ebay.com',
  'etsy.com',
  'newegg.com',
  'target.com',
  'walmart.com',
];
```

#### 3.1.4 邮箱域名 (EMAIL_DOMAINS)
```javascript
export const EMAIL_DOMAINS = [
  'gmail.',
  'hotmail.',
  'mail.yahoo.',
  'outlook.',
  'proton.me',
  'protonmail.',
];
```

#### 3.1.5 视频平台域名 (VIDEO_DOMAINS)
```javascript
export const VIDEO_DOMAINS = ['twitch.', 'youtube.'];
```

### 3.2 付费广告参数 (PAID_AD_PARAMS)

```javascript
export const PAID_AD_PARAMS = [
  'ad_id=',
  'aid=',
  'dclid=',
  'epik=',
  'fbclid=',
  'gclid=',
  'li_fat_id=',
  'msclkid=',
  'ob_click_id=',
  'pc_id=',
  'rdt_cid=',
  'scid=',
  'ttclid=',
  'twclid=',
  'utm_medium=cpc',
  'utm_medium=paid',
  'utm_medium=paid_social',
  'utm_source=google',
];
```

### 3.3 域名分组映射 (GROUPED_DOMAINS)

用于将相似域名归并为统一名称：

```javascript
export const GROUPED_DOMAINS = [
  { name: 'Google', domain: 'google.com', match: 'google.' },
  { name: 'Facebook', domain: 'facebook.com', match: 'facebook.' },
  { name: 'Twitter', domain: 'twitter.com', match: ['twitter.', 't.co', 'x.com'] },
  // ... 更多
];
```

---

## 四、渠道映射：查询时的分类逻辑

**文件位置**: `src/queries/sql/getChannelMetrics.ts`

### 4.1 渠道映射优先级

渠道识别在查询时动态执行，遵循以下优先级顺序（从上到下，匹配即停止）：

| 优先级 | 渠道类型 | 判断条件 |
|--------|---------|----------|
| 1 | Direct | `referrer_domain` 为空 且 `url_query` 为空 |
| 2 | Paid Ads | URL 查询参数包含任意 `PAID_AD_PARAMS` |
| 3 | Referral | `utm_medium` 包含 'referral', 'app', 'link' |
| 4 | Affiliate | `utm_medium` 包含 'affiliate' |
| 5 | SMS | `utm_medium` 或 `utm_source` 包含 'sms' |
| 6 | Search | 域名匹配 `SEARCH_DOMAINS` 或 `utm_medium` 包含 'organic' |
| 7 | Social | 域名匹配 `SOCIAL_DOMAINS` |
| 8 | Email | 域名匹配 `EMAIL_DOMAINS` 或 `utm_medium` 包含 'mail' |
| 9 | Shopping | 域名匹配 `SHOPPING_DOMAINS` 或 `utm_medium` 包含 'shop' |
| 10 | Video | 域名匹配 `VIDEO_DOMAINS` 或 `utm_medium` 包含 'video' |
| 11 | Referral | `referrer_domain` 非空且不等于当前 `hostname` |

### 4.2 付费/自然流量前缀判断

对于 Search、Social、Shopping、Video 类型，还会根据 `utm_medium` 进一步判断是付费还是自然流量：

```sql
case when website_event.utm_medium LIKE 'p%' OR
    website_event.utm_medium LIKE '%ppc%' OR
    website_event.utm_medium LIKE '%retargeting%' OR
    website_event.utm_medium LIKE '%paid%' then 'paid' 
else 'organic' end prefix
```

最终渠道名称 = `prefix` + `渠道类型`，例如：
- `organicSearch` - 自然搜索
- `paidSearch` - 付费搜索
- `organicSocial` - 自然社交
- `paidSocial` - 付费社交

### 4.3 PostgreSQL 实现示例

```sql
WITH prefix AS (
  select case when website_event.utm_medium LIKE 'p%' OR
      website_event.utm_medium LIKE '%ppc%' OR
      website_event.utm_medium LIKE '%retargeting%' OR
      website_event.utm_medium LIKE '%paid%' then 'paid' else 'organic' end prefix,
      website_event.referrer_domain,
      website_event.url_query,
      website_event.utm_medium,
      website_event.utm_source,
      website_event.session_id,
      website_event.hostname
  from website_event
  where website_event.website_id = {{websiteId::uuid}}
    and website_event.event_type NOT IN (2, 5)
    ${dateQuery}
    ${filterQuery}),

channels as (
  select case
      when referrer_domain = '' and url_query = '' then 'direct'
      when url_query ilike '%gclid=%' OR url_query ilike '%fbclid=%' ... then 'paidAds'
      when utm_medium ilike '%referral%' OR utm_medium ilike '%app%' then 'referral'
      when utm_medium ilike '%affiliate%' then 'affiliate'
      when utm_medium ilike '%sms%' or utm_source ilike '%sms%' then 'sms'
      when referrer_domain ilike '%google.%' OR referrer_domain ilike '%baidu.com%' then concat(prefix, 'Search')
      when referrer_domain ilike '%facebook.%' OR referrer_domain ilike '%twitter.%' then concat(prefix, 'Social')
      when referrer_domain ilike '%gmail.%' OR utm_medium ilike '%mail%' then 'email'
      when referrer_domain ilike '%amazon.%' OR utm_medium ilike '%shop%' then concat(prefix, 'Shopping')
      when referrer_domain ilike '%youtube.%' OR utm_medium ilike '%video%' then concat(prefix, 'Video')
      when referrer_domain != regexp_replace(hostname, '^www.', '') and referrer_domain != '' then 'referral'
      else '' end AS x,
    count(distinct session_id) y
  from prefix
  group by 1
  order by y desc)

select x, sum(y) y
from channels
where x != ''
group by x
order by y desc;
```

### 4.4 ClickHouse 实现差异

ClickHouse 使用 `multiSearchAny` 函数进行高效的多模式匹配：

```sql
when multiSearchAny(lower(url_query), ['gclid=', 'fbclid=', ...]) != 0 then 'paidAds'
```

---

## 五、时间窗口与过滤器机制

### 5.1 时间窗口处理

**文件位置**: `src/lib/prisma.ts`, `src/lib/clickhouse.ts`

#### 5.1.1 日期范围过滤

```typescript
function getDateQuery(filters: Record<string, any>) {
  const { startDate, endDate } = filters;
  if (startDate) {
    if (endDate) {
      return `and website_event.created_at between {{startDate}} and {{endDate}}`;
    } else {
      return `and website_event.created_at >= {{startDate}}`;
    }
  }
  return '';
}
```

#### 5.1.2 时区处理 (ClickHouse)

```typescript
function getDateQuery(filters: Record<string, any>) {
  const { startDate, endDate, timezone } = filters;
  if (startDate && endDate) {
    if (timezone) {
      return `and created_at between toTimezone({startDate:DateTime64},{timezone:String}) and toTimezone({endDate:DateTime64},{timezone:String})`;
    }
    return `and created_at between {startDate:DateTime64} and {endDate:DateTime64}`;
  }
  return '';
}
```

### 5.2 过滤器解析机制

**文件位置**: `src/lib/params.ts`

#### 5.2.1 过滤器格式

过滤器支持两种格式：

**简洁格式（字符串）**:
```javascript
{
  referrer: 'eq.google.com',       // 等于
  country: 'c.United',            // 包含
  browser: 'neq.Chrome,Firefox'   // 不等于多个值
}
```

**完整格式（对象）**:
```javascript
{
  referrer: { name: 'referrer', operator: 'eq', value: 'google.com' }
}
```

#### 5.2.2 过滤器值解析

```typescript
export function parseFilterValue(param: any) {
  if (typeof param === 'string') {
    const operatorValues = Object.values(OPERATORS).join('|');
    const regex = new RegExp(`^(${operatorValues})\\.(.*)$`);
    const [, operator, value] = param.match(regex) || [];
    
    const resolvedOperator = operator || OPERATORS.equals;
    const resolvedValue = value ?? param;
    
    if (resolvedOperator === OPERATORS.equals || resolvedOperator === OPERATORS.notEquals) {
      return { operator: resolvedOperator, value: resolvedValue.split(',') };
    }
    return { operator: resolvedOperator, value: resolvedValue };
  }
  return { operator: OPERATORS.equals, value: [param] };
}
```

#### 5.2.3 支持的操作符

**文件位置**: `src/lib/constants.ts`

```javascript
export const OPERATORS = {
  equals: 'eq',
  notEquals: 'neq',
  set: 's',
  notSet: 'ns',
  contains: 'c',
  doesNotContain: 'dnc',
  regex: 're',
  notRegex: 'nre',
  true: 't',
  false: 'f',
  greaterThan: 'gt',
  lessThan: 'lt',
  greaterThanEquals: 'gte',
  lessThanEquals: 'lte',
  before: 'bf',
  after: 'af',
} as const;
```

#### 5.2.4 过滤列映射

```javascript
export const FILTER_COLUMNS = {
  path: 'url_path',
  entry: 'url_path',
  exit: 'url_path',
  referrer: 'referrer_domain',    // referrer 过滤使用来源域名
  domain: 'referrer_domain',
  hostname: 'hostname',
  // ... 其他字段
};
```

### 5.3 特殊过滤逻辑

#### 5.3.1 Referrer 过滤的额外条件

当过滤 referrer 时，会自动添加条件排除站内跳转：

```typescript
// PostgreSQL
if (name === 'referrer') {
  andClauses.push(
    `and (website_event.referrer_domain != regexp_replace(website_event.hostname, '^www.', '') or website_event.referrer_domain is null)`,
  );
}

// ClickHouse
if (name === 'referrer') {
  andClauses.push(`and referrer_domain != hostname`);
}
```

#### 5.3.2 逻辑组合 (AND/OR)

```typescript
const isOr = filters.match === 'any';

if (isOr) {
  orClauses.push(clause);
} else {
  andClauses.push(`and ${clause}`);
}
```

---

## 六、完整数据流示意图

```
浏览器端采集
    ↓
[Tracker] document.referrer
    ↓
┌─ 归一化 ───────────────────────┐
│ 1. 同源检测 → 站内跳转设为空   │
│ 2. URL 标准化（去搜索/哈希）   │
└───────────────────────────────┘
    ↓
数据上报 (POST /api/send)
    ↓
服务端处理
    ↓
┌─ 字段解析 ─────────────────────┐
│ referrer → URL 解析            │
│   ├→ referrer_domain (去 www.) │
│   ├→ referrer_path             │
│   └→ referrer_query            │
│                                 │
│ 当前 URL → UTM 参数提取         │
│   ├→ utm_source, utm_medium    │
│   ├→ utm_campaign, utm_content │
│   └→ utm_term                  │
│                                 │
│ Click ID 提取                   │
│   ├→ gclid (Google)            │
│   ├→ fbclid (Facebook)         │
│   └→ ...                       │
└───────────────────────────────┘
    ↓
数据存储 (website_event 表)
    ↓
报表查询
    ↓
┌─ 渠道映射 (查询时) ────────────┐
│ 优先级规则匹配                  │
│   1. Direct                    │
│   2. Paid Ads (Click ID)       │
│   3. Referral (utm_medium)     │
│   4. Affiliate                 │
│   5. SMS                       │
│   6. Search (域名+utm_medium)  │
│   7. Social (域名)             │
│   8. Email (域名)              │
│   9. Shopping (域名)           │
│  10. Video (域名)              │
│  11. Referral (外链)           │
└───────────────────────────────┘
    ↓
┌─ 附加前缀 ─────────────────────┐
│ utm_medium 检测                 │
│   → paid 或 organic             │
│   → 如: organicSearch           │
└───────────────────────────────┘
    ↓
聚合统计 (按 session_id 去重)
    ↓
最终报表展示
```

---

## 七、关键技术要点

### 7.1 统计口径一致性

**重要**: 渠道统计使用 `count(distinct session_id)` 作为访客计数，确保同一用户在一次会话内的多次访问只计一次。

### 7.2 数据去重与会话归并

- `session_id`: 基于 IP + UserAgent + 盐值生成，标识同一会话
- `visit_id`: 基于 session_id + 小时级时间戳生成，标识一次访问（30分钟超时）
- 超时机制: 30分钟无活动视为新访问

### 7.3 性能优化

- ClickHouse 使用 `multiSearchAny` 进行高效多模式匹配
- PostgreSQL 使用 `ilike` 进行大小写不敏感匹配
- 支持只读副本扩展

### 7.4 事件类型过滤

渠道统计排除以下事件类型：
- `2` - 自定义事件 (customEvent)
- `5` - 性能指标 (performance)

```sql
and website_event.event_type NOT IN (2, 5)
```

---

## 八、维护指南

### 8.1 新增域名分类

在 `src/lib/constants.ts` 中对应数组添加域名：

```javascript
// 新增搜索引擎
export const SEARCH_DOMAINS = [
  'baidu.com',
  'bing.com',
  // ... 现有域名
  'new-search-engine.com',  // 新增
];
```

### 8.2 新增付费广告参数

```javascript
export const PAID_AD_PARAMS = [
  'gclid=',
  // ... 现有参数
  'new_platform_id=',  // 新增
];
```

### 8.3 新增渠道类型

修改 `src/queries/sql/getChannelMetrics.ts` 和 `getChannelExpandedMetrics.ts` 中的 case 语句。

---

## 九、常见问题

### Q1: 为什么站内跳转不算 referrer?
A: Tracker 端会判断 referrer 是否与当前 origin 同源，同源则设为空。同时查询时 referrer 过滤器也会排除与当前 hostname 相同的记录。

### Q2: Direct 流量的判断条件是什么?
A: `referrer_domain` 为空 **且** `url_query` 为空。这意味着用户直接输入网址访问，且 URL 不带任何查询参数。

### Q3: 付费/自然前缀如何判断?
A: 主要通过 `utm_medium` 参数判断，包含 'paid', 'ppc', 'retargeting', 'cpc' 等关键词则标记为 paid，否则为 organic。

### Q4: 为什么有些 external link 被归为 referral 而非其他类型?
A: 因为域名不在预定义的分类列表中，或者 UTM 参数没有匹配上特定类型。

---

## 十、三条查询路径的对比分析

### 10.1 查询路径概览

| 查询路径 | 主要用途 | 统计单位 |
|---------|---------|---------|
| `getChannelMetrics` | 渠道概览图表 | session_id 去重 |
| `getChannelExpandedMetrics` | 渠道详情表格 | session_id / visit_id 去重 |
| `getRevenueMetrics` | 收入渠道分布 | 首个事件属性 |

---

### 10.2 来源分类规则一致性校验

#### 10.2.1 优先级顺序对比

**三条路径的优先级顺序完全一致**，均为：

| 优先级 | 渠道 | getChannelMetrics | getChannelExpandedMetrics | getRevenueMetrics |
|--------|------|:-----------------:|:-------------------------:|:-----------------:|
| 1 | Direct | ✅ | ✅ | ✅ |
| 2 | Paid Ads | ✅ | ✅ | ✅ |
| 3 | Referral (utm_medium) | ✅ | ✅ | ✅ |
| 4 | Affiliate | ✅ | ✅ | ✅ |
| 5 | SMS | ✅ | ✅ | ✅ |
| 6 | Search | ✅ | ✅ | ✅ |
| 7 | Social | ✅ | ✅ | ✅ |
| 8 | Email | ✅ | ✅ | ✅ |
| 9 | Shopping | ✅ | ✅ | ✅ |
| 10 | Video | ✅ | ✅ | ✅ |
| 11 | Referral (外链) | ✅ | ✅ | ✅ |

#### 10.2.2 付费/自然前缀判断对比（已修正）

**原始代码逐字对照**：

| 条件 | getChannelMetrics (PG) | getChannelExpandedMetrics (PG) | getRevenueMetrics (PG) | 全部 ClickHouse |
|------|:----------------------:|:------------------------------:|:----------------------:|:---------------:|
| 条件 1 | `LIKE 'p%'` | `LIKE 'p%'` | `ilike '%cp%'` | `multiSearchAny(lower(…), ['cp',…])` |
| 条件 2 | `LIKE '%ppc%'` | `LIKE '%ppc%'` | `ilike '%ppc%'` | 同上（含 'ppc'） |
| 条件 3 | `LIKE '%retargeting%'` | `LIKE '%retargeting%'` | `ilike '%retargeting%'` | 同上（含 'retargeting'） |
| 条件 4 | `LIKE '%paid%'` | `LIKE '%paid%'` | `ilike '%paid%'` | 同上（含 'paid'） |
| 大小写 | 区分大小写 (LIKE) | 区分大小写 (LIKE) | 不区分 (ilike) | 不区分 (lower) |

**关键语义差异推演**：

`LIKE 'p%'` = 匹配以小写 'p' **开头**的任意字符串。这是 PostgreSQL LIKE 的精确语义：`%` 只出现在末尾，表示"p 后跟任意字符"。

`ilike '%cp%'` = 匹配**任意位置包含** 'cp' 子串的字符串，不区分大小写。

`multiSearchAny(lower(utm_medium), ['cp',…])` = 将 utm_medium 转小写后，检查是否**包含** 'cp' 子串。

**三种模式的语义完全不同**，详见下方反例推演。

#### 10.2.3 未匹配处理对比

| 处理方式 | getChannelMetrics | getChannelExpandedMetrics | getRevenueMetrics |
|---------|:-----------------:|:-------------------------:|:-----------------:|
| 未匹配返回 | 空字符串 `''` | 空字符串 `''` | `'Unknown'` |
| 过滤空值 | `where x != ''` | `where name != ''` | 无过滤（包含 Unknown） |

**⚠️ 差异发现**: 
- getChannelMetrics 和 getChannelExpandedMetrics 会过滤掉未匹配的记录
- getRevenueMetrics 会将未匹配的记录归类为 `'Unknown'` 并统计

---

### 10.3 Session 去重规则一致性校验

#### 10.3.1 事件类型过滤

| 事件类型 | getChannelMetrics | getChannelExpandedMetrics | getRevenueMetrics |
|---------|:-----------------:|:-------------------------:|:-----------------:|
| 1 (pageView) | ✅ 包含 | ✅ 包含 | ✅ 包含 |
| 2 (customEvent) | ❌ 排除 | ❌ 排除 | ✅ 关联（收入事件） |
| 3 (linkEvent) | ✅ 包含 | ✅ 包含 | ✅ 包含 |
| 4 (pixelEvent) | ✅ 包含 | ✅ 包含 | ✅ 包含 |
| 5 (performance) | ❌ 排除 | ❌ 排除 | ✅ 包含 |

**⚠️ 差异发现**: 
- getChannelMetrics/Expanded: `event_type NOT IN (2, 5)` - 排除自定义事件和性能事件
- getRevenueMetrics: 基于 revenue 表，通过 event_id 关联 event_type = 2 的自定义事件（收入事件）

#### 10.3.2 去重统计方式

| 统计维度 | getChannelMetrics | getChannelExpandedMetrics | getRevenueMetrics |
|---------|:-----------------:|:-------------------------:|:-----------------:|
| visitors (session) | `count(distinct session_id)` | `count(distinct session_id)` | 按 session_id 聚合后求和 |
| visits | ❌ 不统计 | `count(distinct visit_id)` | ❌ 不统计 |
| pageviews | ❌ 不统计 | `count(*)` | ❌ 不统计 |
| 收入金额 | ❌ 不统计 | ❌ 不统计 | `sum(revenue)` |

#### 10.3.3 首事件归因逻辑（getRevenueMetrics 特有）

getRevenueMetrics 使用**首事件归因**，即：

```sql
WITH revenue_data AS (
  select
    e.session_id,
    e.value,
    we.min_date as created_at  -- 取该 session 的首个事件时间
  from events e
  join (
    select session_id, min(created_at) as min_date
    from website_event
    group by session_id
  ) we on we.session_id = e.session_id
)
-- 用首个事件的属性来判断渠道
```

**影响**: 收入的渠道归属是基于用户会话中**第一个事件**的来源属性，而不是产生收入的那个事件的属性。

---

## 十一、Session ID 生成路径与 DistinctId 影响

### 11.1 Session ID 生成逻辑

**文件位置**: `src/app/api/send/route.ts`, `src/lib/crypto.ts`

#### 11.1.1 核心生成代码

```typescript
// src/app/api/send/route.ts:147
const sessionId = id ? uuid(sourceId, id) : uuid(sourceId, ip, userAgent, sessionSalt);
```

```typescript
// src/lib/crypto.ts:60-66
export function uuid(...args: any) {
  if (args.length) {
    // 确定性 UUID v5: 基于输入参数的哈希值生成
    return v5(hash(...args, secret()), v5.DNS);
  }
  // 随机 UUID v4 或 v7
  return process.env.USE_UUIDV7 ? v7() : v4();
}
```

#### 11.1.2 两种生成模式对比

| 模式 | 触发条件 | 输入参数 | 稳定性 |
|------|---------|---------|--------|
| **DistinctId 模式** | 提供了 `id` 参数（来自 `umami.identify()`） | `sourceId` + `distinctId` | ✅ 跨设备/跨网络稳定 |
| **默认模式** | 未提供 `id` 参数 | `sourceId` + `ip` + `userAgent` + `sessionSalt` | ⚠️ IP/UserAgent 变化时改变 |

---

### 11.2 DistinctId 存在时的影响

#### 11.2.1 会话归并行为

**当调用 `umami.identify('user123')` 后：**

```javascript
// 前端调用
umami.identify('user123', { name: 'John' });
```

**效果**:
1. `identity` 变量被设置为 `'user123'`
2. 后续所有事件上报时，`payload.id = 'user123'`
3. 服务端使用 `uuid(sourceId, 'user123')` 生成 session_id
4. **同一 distinctId 的所有事件会归并到同一个 session_id**

#### 11.2.2 盐值轮换的影响

```typescript
// src/lib/crypto.ts:72-78
export function getSalt(saltRotation: string, createdAt: Date): string {
  return hash(
    (saltRotation === 'day' ? startOfDay : saltRotation === 'week' ? startOfWeek : startOfMonth)(
      createdAt,
    ).toUTCString(),
  );
}
```

| 配置 | 默认值 | 影响 |
|------|--------|------|
| `SALT_ROTATION` | `'month'` | 每月重置盐值 |
| 重置周期 | 月/周/天 | 盐值变化后，相同 IP+UA 会生成不同 session_id |

**注意**: DistinctId 模式**不受盐值轮换影响**，因为它不使用 sessionSalt！

---

### 11.3 Session 归并对比矩阵

| 场景 | 默认模式 (无 distinctId) | DistinctId 模式 |
|------|:-----------------------:|:---------------:|
| 同一浏览器，同一 IP | ✅ 同一 session_id | ✅ 同一 session_id |
| 同一浏览器，IP 变化 | ❌ 不同 session_id | ✅ 同一 session_id |
| 不同浏览器，同一用户 | ❌ 不同 session_id | ✅ 同一 session_id |
| 跨设备用户 | ❌ 不同 session_id | ✅ 同一 session_id |
| 盐值轮换后 | ❌ 不同 session_id | ✅ 同一 session_id |
| 隐私模式浏览 | ❌ 不同 session_id | ✅ 同一 session_id（需登录） |

---

### 11.4 对渠道统计的影响

#### 11.4.1 默认模式的问题

**问题场景**: 用户在上班时用公司网络访问，下班回家用家庭网络继续访问

```
上午 (公司网络):
  IP: 203.0.113.1
  UserAgent: Chrome/Windows
  → session_id: ABC123
  → 渠道: organicSearch

晚上 (家庭网络):
  IP: 198.51.100.2
  UserAgent: Chrome/Windows
  → session_id: DEF456  (不同！)
  → 渠道: direct
```

**统计结果**: 被算作 2 个访客，渠道被拆分

#### 11.4.2 DistinctId 模式的解决

**使用 identify 后**:

```
上午:
  distinctId: user123
  → session_id: UUID-user123
  → 渠道: organicSearch

晚上:
  distinctId: user123
  → session_id: UUID-user123  (相同！)
  → 渠道: organicSearch (沿用首事件属性)
```

**统计结果**: 被算作 1 个访客，渠道归属一致

---

## 十二、校验结果与证据汇总

### 12.1 一致性校验结论（已修正）

| 校验项 | 结论 | 证据位置 |
|-------|------|---------|
| 渠道分类优先级 | ✅ 完全一致 | getChannelMetrics:54-65<br>getChannelExpandedMetrics:85-96<br>getRevenueMetrics:207-218 |
| 付费前缀判断 | ❌ 三种实现语义不同，存在跨路径+跨引擎双重不一致 | getChannelMetrics:34 `LIKE 'p%'`<br>getRevenueMetrics:185 `ilike '%cp%'`<br>ClickHouse 全部:96 `multiSearchAny(…,['cp',…])` |
| 渠道 CASE 分支逻辑 | ✅ 完全一致（同一组常量、同一优先级） | 三文件使用相同的 SEARCH/SOCIAL/EMAIL/SHOPPING/VIDEO_DOMAINS |
| 未匹配处理 | ❌ 不一致（空 vs Unknown） | getChannelMetrics:74<br>getRevenueMetrics:219 |
| 事件类型过滤 | ❌ 不一致（设计意图不同） | getChannelMetrics:49 `NOT IN (2,5)`<br>getRevenueMetrics:46-56 关联收入事件 |
| Session 去重逻辑 | ⚠️ 统计维度不同（设计意图差异） | getChannelMetrics:67 `count(distinct session_id)`<br>getChannelExpandedMetrics:108-109 session+visit<br>getRevenueMetrics:107 按 session 聚合求和 |

### 12.2 关键发现清单

1. **收入归因采用首事件模式**: getRevenueMetrics 使用 session 中第一个事件的来源属性来判断渠道，而不是收入事件本身的属性
2. **DistinctId 可跨设备归并**: 使用 identify() 后，相同用户在不同设备/网络下会被归并为同一会话
3. **盐值轮换不影响 DistinctId**: DistinctId 模式不使用 sessionSalt，因此盐值轮换不会拆分会话
4. **Unknown 渠道仅在收入报表出现**: 普通渠道报表会过滤未匹配记录，收入报表会显示为 Unknown

---

## 十三、核对清单

### 13.1 采集端核对

- [ ] `document.referrer` 同源检测是否正确执行
- [ ] URL 标准化函数是否处理了相对 URL
- [ ] excludeSearch/excludeHash 配置是否生效
- [ ] Tracker 上报的 payload 中是否包含 referrer 字段

### 13.2 服务端处理核对

- [ ] referrer 是否被正确解析为 domain/path/query
- [ ] 域名是否正确移除 www. 前缀
- [ ] UTM 参数是否从目标 URL（而非 referrer）提取
- [ ] Click ID 参数（gclid/fbclid 等）是否正确提取
- [ ] session_id 生成逻辑是否符合预期（带/不带 distinctId）
- [ ] 盐值轮换配置是否正确应用

### 13.3 渠道映射核对

- [ ] 11 级优先级顺序是否正确执行
- [ ] Direct 条件是否为（domain 空 AND query 空）
- [ ] Paid Ads 参数列表是否完整
- [ ] 域名分类数组是否包含所有需要的域名
- [ ] 付费/自然前缀判断逻辑是否正确
- [ ] 站内跳转是否被正确排除

### 13.4 三条查询路径核对

**getChannelMetrics**:
- [ ] 事件过滤: `event_type NOT IN (2, 5)`
- [ ] 去重方式: `count(distinct session_id)`
- [ ] 空值处理: `where x != ''`

**getChannelExpandedMetrics**:
- [ ] 事件过滤: `event_type NOT IN (2, 5)`
- [ ] visitors: `count(distinct session_id)`
- [ ] visits: `count(distinct visit_id)`
- [ ] 空值处理: `where name != ''`

**getRevenueMetrics**:
- [ ] 首事件归因: 使用 min(created_at) 关联
- [ ] 未匹配处理: 返回 'Unknown'
- [ ] 收入聚合: `sum(revenue)`

### 13.5 Session 归并核对

- [ ] 无 distinctId 时，IP 变化是否产生新 session
- [ ] 有 distinctId 时，跨设备是否归并
- [ ] 盐值轮换后，默认模式是否产生新 session
- [ ] 盐值轮换后，distinctId 模式是否保持一致
- [ ] visit_id 30 分钟超时逻辑是否正确

### 13.6 过滤器核对

- [ ] referrer 过滤是否自动排除站内域名
- [ ] 时间窗口是否正确应用时区
- [ ] 过滤操作符是否正确映射
- [ ] AND/OR 逻辑组合是否正确

---

## 十四、相关文件速查（扩展版）

| 功能 | 文件路径 |
|------|---------|
| Tracker 采集 | `src/tracker/index.js` |
| 数据接收 | `src/app/api/send/route.ts` |
| 常量配置 | `src/lib/constants.ts` |
| 加密与 UUID | `src/lib/crypto.ts` |
| 事件保存 | `src/queries/sql/events/saveEvent.ts` |
| 渠道统计 | `src/queries/sql/getChannelMetrics.ts` |
| 渠道详情 | `src/queries/sql/getChannelExpandedMetrics.ts` |
| 收入渠道统计 | `src/queries/sql/reports/getRevenueMetrics.ts` |
| Prisma 过滤器 | `src/lib/prisma.ts` |
| ClickHouse 过滤器 | `src/lib/clickhouse.ts` |
| 参数解析 | `src/lib/params.ts` |

---

## 十五、已修正结论：付费前缀判断的精确分析

> 本节修正了此前文档中的错误论断。原文声称 `LIKE 'p%'` 能匹配 'cpc'（"因为 'p%' 匹配 'cpc' 的第二个字符开始"），这是对 SQL LIKE 语法的根本误解。下面给出精确推演。

### 15.1 三种实现的精确语义

#### 15.1.1 PostgreSQL `LIKE 'p%'`（getChannelMetrics / getChannelExpandedMetrics）

```sql
-- 来源: getChannelMetrics.ts:34, getChannelExpandedMetrics.ts:53
utm_medium LIKE 'p%'
```

**语义**：匹配以小写字母 'p' **开头**的字符串。PostgreSQL 的 LIKE 区分大小写。

- `%` 在 SQL LIKE 中是通配符，代表"零个或多个任意字符"
- `'p%'` = 字面量 'p' 后跟任意字符序列
- **不是**"任意位置包含 p"

#### 15.1.2 PostgreSQL `ilike '%cp%'`（getRevenueMetrics）

```sql
-- 来源: getRevenueMetrics.ts:185
we.utm_medium ilike '%cp%'
```

**语义**：匹配**任意位置包含**子串 'cp' 的字符串，不区分大小写。

- `%cp%` = 前面任意字符 + 'cp' + 后面任意字符
- ilike = 大小写不敏感的 LIKE

#### 15.1.3 ClickHouse `multiSearchAny(lower(…), ['cp',…])`（全部三条路径）

```sql
-- 来源: getChannelMetrics.ts:96, getChannelExpandedMetrics.ts:143, getRevenueMetrics.ts:396
multiSearchAny(lower(utm_medium), ['cp', 'ppc', 'retargeting', 'paid']) != 0
```

**语义**：将 utm_medium 转小写后，检查是否**包含** 'cp'、'ppc'、'retargeting'、'paid' 中任意一个子串。

- `multiSearchAny` 是子串搜索函数，等价于多个 `position(…) > 0` 的 OR
- `lower()` 保证大小写不敏感

---

### 15.2 逐值推演：哪些 utm_medium 值会产生分歧

#### 15.2.1 核心分歧表

| utm_medium | PG ChannelMetrics `LIKE 'p%'` | PG RevenueMetrics `ilike '%cp%'` | CH 全部 `multiSearchAny(['cp',…])` | 是否存在分歧 |
|-----------|:-----------------------------:|:--------------------------------:|:----------------------------------:|:----------:|
| `'cpc'` | ❌ organic（以 'c' 开头） | ✅ paid（含 'cp'） | ✅ paid（含 'cp'） | **❌ 三方不一致** |
| `'CPC'` | ❌ organic（LIKE 区分大小写） | ✅ paid（ilike 不区分） | ✅ paid（lower→'cpc'含 'cp'） | **❌ 三方不一致** |
| `'CPA'` | ❌ organic | ✅ paid（含 'CP'→ilike 匹配） | ✅ paid（lower→'cpa'含 'cp'） | **❌ 三方不一致** |
| `'paid'` | ✅ paid（以 'p' 开头） | ✅ paid（含 'paid'） | ✅ paid（含 'paid'） | ✅ 一致 |
| `'Paid'` | ❌ organic（大写 P，LIKE 区分大小写） | ✅ paid（ilike 不区分） | ✅ paid（lower→'paid'） | **❌ PG ChannelMetrics 不一致** |
| `'PPC'` | ❌ organic（大写 P） | ✅ paid（ilike '%ppc%'匹配） | ✅ paid（lower→'ppc'） | **❌ PG ChannelMetrics 不一致** |
| `'ppc'` | ✅ paid（以 'p' 开头） | ✅ paid（含 'ppc'） | ✅ paid（含 'ppc'） | ✅ 一致 |
| `'retargeting'` | ❌ organic（以 'r' 开头，但 LIKE '%retargeting%' 单独匹配） | ✅ paid | ✅ paid | ✅ 一致（靠其他条件） |
| `'promo'` | ✅ paid（以 'p' 开头 ⚠️误判） | ❌ organic（不含 'cp'） | ❌ organic（不含 'cp'） | **❌ PG ChannelMetrics 误判为 paid** |
| `'product'` | ✅ paid（以 'p' 开头 ⚠️误判） | ❌ organic | ❌ organic | **❌ PG ChannelMetrics 误判为 paid** |
| `'partnership'` | ✅ paid（以 'p' 开头 ⚠️误判） | ❌ organic | ❌ organic | **❌ PG ChannelMetrics 误判为 paid** |
| `'press'` | ✅ paid（以 'p' 开头 ⚠️误判） | ❌ organic | ❌ organic | **❌ PG ChannelMetrics 误判为 paid** |
| `'print'` | ✅ paid（以 'p' 开头 ⚠️误判） | ❌ organic | ❌ organic | **❌ PG ChannelMetrics 误判为 paid** |
| `'programmatic'` | ✅ paid（以 'p' 开头 ⚠️误判） | ❌ organic | ❌ organic | **❌ PG ChannelMetrics 误判为 paid** |

#### 15.2.2 分歧分类总结

**分歧类型 A — `LIKE 'p%'` 过度匹配（误判为 paid）**：

仅 PostgreSQL ChannelMetrics/ExpandedMetrics 会将非付费的 'p' 开头 utm_medium 误判为 paid：
- `'promo'` → 实际为推广，非付费广告
- `'product'` → 产品相关
- `'partnership'` → 合作伙伴
- `'press'` → 媒体/新闻
- `'print'` → 平面媒体
- `'programmatic'` → 程序化投放（可能是付费，也可能不是）

**分歧类型 B — `LIKE 'p%'` 漏匹配（漏判为 organic）**：

PostgreSQL ChannelMetrics/ExpandedMetrics 无法识别以下付费媒介：
- `'cpc'` → 标准 Google Ads 付费媒介（cost-per-click），被漏判为 organic
- `'CPC'` → 同上大写形式
- `'CPA'` → cost-per-action 付费模式
- `'cpv'` → cost-per-view 付费模式

**分歧类型 C — 大小写敏感导致漏匹配**：

PostgreSQL ChannelMetrics/ExpandedMetrics 使用区分大小写的 LIKE：
- `'Paid'` → 被漏判为 organic（大写 P 不匹配 `LIKE 'p%'`）
- `'PPC'` → 被漏判为 organic（但被 `LIKE '%ppc%'` 漏掉后，`LIKE 'p%'` 也因大写而失败）

---

### 15.3 ClickHouse 三条路径的一致性确认

ClickHouse 的三条查询路径使用**完全相同**的前缀判断逻辑：

```sql
-- getChannelMetrics.ts:96
-- getChannelExpandedMetrics.ts:143
-- getRevenueMetrics.ts:396
-- 三者代码完全一致
case when multiSearchAny(lower(utm_medium), ['cp', 'ppc', 'retargeting', 'paid']) != 0
  then 'paid' else 'organic' end prefix
```

**结论：ClickHouse 引擎下，三条路径的付费前缀判断完全一致，不存在跨路径分歧。**

---

### 15.4 PostgreSQL 两条渠道路径的一致性确认

getChannelMetrics 和 getChannelExpandedMetrics 使用**完全相同**的前缀判断逻辑：

```sql
-- getChannelMetrics.ts:34
-- getChannelExpandedMetrics.ts:53
-- 二者代码完全一致
case when website_event.utm_medium LIKE 'p%' OR
    website_event.utm_medium LIKE '%ppc%' OR
    website_event.utm_medium LIKE '%retargeting%' OR
    website_event.utm_medium LIKE '%paid%' then 'paid' else 'organic' end prefix
```

**结论：PostgreSQL 引擎下，getChannelMetrics 与 getChannelExpandedMetrics 的付费前缀判断完全一致。**

---

### 15.5 真正存在的不一致维度

| 不一致维度 | 涉及路径 | 根因 |
|-----------|---------|------|
| **PG ChannelMetrics ↔ PG RevenueMetrics** | 跨路径 | `LIKE 'p%'` vs `ilike '%cp%'` 语义不同 |
| **PG ChannelMetrics ↔ CH ChannelMetrics** | 跨引擎 | `LIKE 'p%'` vs `multiSearchAny(['cp',…])` 语义不同 |
| **PG ChannelMetrics 大小写** | 单路径内 | `LIKE` 区分大小写，漏判 'Paid'/'PPC'/'CPC' |
| **PG ChannelMetrics 过度匹配** | 单路径内 | `LIKE 'p%'` 误判 'promo'/'product'/'press' 为 paid |

**不存在的不一致**：
- ~~PG ChannelMetrics ↔ PG ExpandedMetrics~~：两者代码完全相同
- ~~CH 三条路径之间~~：三者代码完全相同

---

### 15.6 修正前错误声明与修正后结论对照

| 编号 | 修正前声明 | 修正后结论 |
|------|----------|----------|
| E1 | "`LIKE 'p%'` 能匹配 'cpc'，因为 'p%' 匹配 'cpc' 的第二个字符" | ❌ **错误**。`LIKE 'p%'` 只匹配以 'p' **开头**的字符串。'cpc' 以 'c' 开头，不匹配。`%` 在 LIKE 中不是"任意位置"通配符，只有 `%pattern%` 才是。 |
| E2 | "三条路径的实际效果可能一致，只是实现方式不同" | ❌ **错误**。实际效果在 'cpc'、'CPC'、'promo' 等场景下**完全不同**。 |
| E3 | "付费前缀判断是细微差异" | ❌ **错误**。这是**语义级差异**，影响面远超"细微"。`LIKE 'p%'` 存在过度匹配和漏匹配双重问题。 |
| E4 | "ClickHouse 三条路径一致" | ✅ **确认正确**。三条路径使用完全相同的 `multiSearchAny` 表达式。 |
| E5 | "PG ChannelMetrics 和 PG ExpandedMetrics 有细微差异" | ❌ **错误**。两者代码完全相同，无任何差异。 |

---

## 十六、核对清单（修订版）

### 16.1 付费前缀判断核对

- [ ] `LIKE 'p%'` 是否将 'promo'/'product'/'press' 误判为 paid（PG ChannelMetrics/ExpandedMetrics）
- [ ] `LIKE 'p%'` 是否漏判 'cpc' 为 organic（PG ChannelMetrics/ExpandedMetrics）
- [ ] `LIKE 'p%'` 是否因大小写敏感漏判 'Paid'/'PPC'（PG ChannelMetrics/ExpandedMetrics）
- [ ] `ilike '%cp%'` 是否正确匹配 'cpc'/'CPC'/'CPA'（PG RevenueMetrics）
- [ ] `multiSearchAny` 是否正确匹配所有付费媒介变体（CH 全部路径）
- [ ] 同一 utm_medium='cpc' 在 PG 和 CH 下是否产生不同渠道分类
- [ ] 同一 utm_medium='promo' 在 PG ChannelMetrics 和 PG RevenueMetrics 下是否产生不同渠道分类

### 16.2 渠道 CASE 分支一致性核对

- [ ] 11 级优先级顺序在三条路径中是否完全一致
- [ ] SEARCH_DOMAINS / SOCIAL_DOMAINS / EMAIL_DOMAINS / SHOPPING_DOMAINS / VIDEO_DOMAINS 常量引用是否一致
- [ ] PAID_AD_PARAMS 常量引用是否一致
- [ ] `toPostgresLikeClause` 和 `toPostgresPositionClause` 生成的 SQL 是否等价（答案：是，均生成 `column ilike '%value%'`）

### 16.3 跨引擎一致性核对

- [ ] PG `LIKE 'p%'` 与 CH `multiSearchAny(['cp',…])` 对 'cpc' 的判断是否不同（答案：是）
- [ ] PG `LIKE 'p%'` 与 CH `multiSearchAny(['cp',…])` 对 'promo' 的判断是否不同（答案：是）
- [ ] PG `ilike '%cp%'` 与 CH `multiSearchAny(['cp',…])` 对 'cpc' 的判断是否一致（答案：是，均判定为 paid）

### 16.4 未匹配处理核对

- [ ] getChannelMetrics/getChannelExpandedMetrics 是否过滤空字符串（`where x != ''`/`where name != ''`）
- [ ] getRevenueMetrics 是否将未匹配记录归为 'Unknown' 并保留统计
- [ ] 同一用户在渠道概览表和收入渠道表中是否可能出现渠道名不一致

### 16.5 事件类型核对

- [ ] getChannelMetrics/ExpandedMetrics 排除 event_type IN (2, 5)
- [ ] getRevenueMetrics 关联 revenue 表（基于 event_type=2 的自定义事件）
- [ ] 收入报表渠道归属使用首事件属性（min(created_at) 关联）

### 16.6 Session 归并核对

- [ ] 无 distinctId 时，IP 变化产生新 session_id
- [ ] 有 distinctId 时，跨设备归并为同一 session_id
- [ ] 盐值轮换后，默认模式产生新 session_id
- [ ] 盐值轮换后，distinctId 模式保持一致
- [ ] visit_id 30 分钟超时逻辑

### 16.7 采集端核对

- [ ] `document.referrer` 同源检测
- [ ] URL 标准化函数（excludeSearch/excludeHash）
- [ ] referrer 在 payload 中正确传递

### 16.8 过滤器核对

- [ ] referrer 过滤自动排除站内域名
- [ ] 时间窗口正确应用时区
- [ ] 过滤操作符正确映射
- [ ] AND/OR 逻辑组合正确
