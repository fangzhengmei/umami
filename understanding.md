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

## 十、相关文件速查

| 功能 | 文件路径 |
|------|---------|
| Tracker 采集 | `src/tracker/index.js` |
| 数据接收 | `src/app/api/send/route.ts` |
| 常量配置 | `src/lib/constants.ts` |
| 事件保存 | `src/queries/sql/events/saveEvent.ts` |
| 渠道统计 | `src/queries/sql/getChannelMetrics.ts` |
| 渠道详情 | `src/queries/sql/getChannelExpandedMetrics.ts` |
| Prisma 过滤器 | `src/lib/prisma.ts` |
| ClickHouse 过滤器 | `src/lib/clickhouse.ts` |
| 参数解析 | `src/lib/params.ts` |
