# Umami Performance 报表技术分析报告

## 一、整体架构概览

Performance 报表的数据流从前端浏览器采集开始，经过数据上报、后端存储、查询计算、缓存层，最终在报表页面展示。整个链路分为 **5 个核心阶段**：

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  浏览器采集层   │────▶│  数据上报层     │────▶│  数据存储层     │
│  (Tracker)      │     │  (/api/send)    │     │  (ClickHouse)   │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                         │
                                                         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  页面展示层     │◀────│  查询计算层     │◀────│  数据聚合层     │
│  (Performance)  │     │  (React Query)  │     │  (SQL Queries)  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

---

## 二、采集指标详解

### 2.1 核心 Web Vitals 指标

Performance 报表采集 **5 个核心 Web Vitals 指标**，全部基于浏览器原生 Performance API：

| 指标 | 全称 | 单位 | 含义 |
|------|------|------|------|
| **LCP** | Largest Contentful Paint | ms | 最大内容绘制时间，衡量页面加载性能 |
| **INP** | Interaction to Next Paint | ms | 交互到下一帧绘制时间，衡量页面响应性 |
| **CLS** | Cumulative Layout Shift | - | 累积布局偏移，衡量页面视觉稳定性 |
| **FCP** | First Contentful Paint | ms | 首次内容绘制时间，衡量页面首次渲染 |
| **TTFB** | Time to First Byte | ms | 首字节时间，衡量服务器响应速度 |

> 源码位置：`src/tracker/index.js:231-383`

### 2.2 采集实现机制

**启动条件**：需要在 tracker script 标签上设置 `data-performance="true"` 才会启用性能采集。

```javascript
const perf = config('performance') === 'true';
if (perf) initPerformance();
```

#### 2.2.1 LCP 采集

```javascript
// src/tracker/index.js:263-266
observe('largest-contentful-paint', entry => {
  metrics.lcp = Math.max(entry.startTime - activationStart, 0);
});
```
- 使用 `PerformanceObserver` 监听 `largest-contentful-paint` 类型
- 减去 `activationStart` 处理页面预渲染/后台加载场景
- 持续更新，取最大的 startTime

#### 2.2.2 INP 采集

```javascript
// src/tracker/index.js:292-313
const observer = new PerformanceObserver(list => {
  list.getEntries().forEach(entry => {
    if (entry.interactionId) {
      const existing = interactions[entry.interactionId];
      if (!existing || entry.duration > existing) {
        interactions[entry.interactionId] = entry.duration;
      }
      const values = Object.values(interactions).sort((a, b) => b - a);
      const p98Index = Math.floor(Math.max(values.length, 10) * 0.02);
      metrics.inp = values[Math.min(p98Index, values.length - 1)];
    }
  });
});
observer.observe({ type: 'event', buffered: true, durationThreshold: 40 });
```
- 按 `interactionId` 分组，取每组最大 duration
- 对所有交互按 duration 降序排列
- 取 **98th 百分位** 的值作为 INP（至少采样 10 次）
- `durationThreshold: 40` 过滤掉小于 40ms 的交互

#### 2.2.3 CLS 采集

```javascript
// src/tracker/index.js:268-290
let clsSessionValue = 0;
let clsSessionEntries = [];
observe('layout-shift', entry => {
  if (!entry.hadRecentInput) {
    const lastEntry = clsSessionEntries[clsSessionEntries.length - 1];
    const firstEntry = clsSessionEntries[0];
    if (
      lastEntry &&
      entry.startTime - lastEntry.startTime - lastEntry.duration < 1000 &&
      entry.startTime - firstEntry.startTime < 5000
    ) {
      clsSessionValue += entry.value;
      clsSessionEntries.push(entry);
    } else {
      clsSessionValue = entry.value;
      clsSessionEntries = [entry];
    }
    if (clsSessionValue > (metrics.cls || 0)) {
      metrics.cls = clsSessionValue;
    }
  }
});
```
- **会话窗口算法**：
  - 相邻布局偏移间隔 < 1s 视为同一会话
  - 单个会话最长持续 5s
  - 每个会话累加所有布局偏移值
- 取所有会话中最大的累积偏移值
- 排除用户输入导致的布局偏移（`hadRecentInput`）

#### 2.2.4 FCP 采集

```javascript
// src/tracker/index.js:256-261
observe('paint', entry => {
  if (entry.name === 'first-contentful-paint') {
    metrics.fcp = Math.max(entry.startTime - activationStart, 0);
  }
});
```

#### 2.2.5 TTFB 采集

```javascript
// src/tracker/index.js:250-254
observe('navigation', entry => {
  activationStart = entry.activationStart || 0;
  metrics.ttfb = Math.max(entry.responseStart - activationStart, 0);
});
```

### 2.3 上报触发时机

性能数据在以下时机触发上报：
1. **延迟上报**：页面加载后 10 秒自动上报（`setTimeout(sendPerformance, 10000)`）
2. **页面隐藏**：`visibilitychange` 事件触发，当页面变为 hidden 时上报
3. **页面卸载**：`pagehide` 事件触发时上报
4. **路由跳转**：SPA 路由变化时（`pushState`/`replaceState`）先 flush 再重置

```javascript
// src/tracker/index.js:379-382
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') sendPerformance();
});
window.addEventListener('pagehide', sendPerformance);
```

### 2.4 回退机制

如果 PerformanceObserver 未能采集到数据，上报前会通过 `performance.getEntriesByType()` 尝试回退获取：

```javascript
// src/tracker/index.js:323-349
const applyFallbackMetrics = () => {
  if (!isInitialLoad) return;
  if (metrics.ttfb === undefined) {
    const navigation = getEntriesByType('navigation')?.[0];
    if (navigation) {
      metrics.ttfb = Math.max(navigation.responseStart - (navigation.activationStart || 0), 0);
    }
  }
  // FCP、LCP 同理...
};
```

---

## 三、数据上报与存储

### 3.1 上报接口

**接口路径**：`POST /api/send`

**上报类型**：`type: "performance"`

**Payload 结构**：

```typescript
{
  type: 'performance',
  payload: {
    website: string;        // 网站 UUID
    hostname: string;       // 主机名
    language: string;       // 语言
    screen: string;         // 屏幕分辨率
    title: string;          // 页面标题
    url: string;            // 页面 URL
    tag?: string;           // 自定义标签
    id?: string;            // 用户标识
    
    // 性能指标（全部可选）
    lcp?: number;           // 0 ~ 60000 ms
    inp?: number;           // 0 ~ 60000 ms
    cls?: number;           // 0 ~ 100
    fcp?: number;           // 0 ~ 60000 ms
    ttfb?: number;          // 0 ~ 60000 ms
  }
}
```

> 源码位置：`src/app/api/send/route.ts:47-51`

### 3.2 数据校验（Zod Schema）

```typescript
// src/app/api/send/route.ts:24-52
const schema = z.object({
  type: z.enum(['event', 'identify', 'performance']),
  payload: z.object({
    // ... 基础字段
    lcp: z.number().nonnegative().max(60000).optional(),
    inp: z.number().nonnegative().max(60000).optional(),
    cls: z.number().nonnegative().max(100).optional(),
    fcp: z.number().nonnegative().max(60000).optional(),
    ttfb: z.number().nonnegative().max(60000).optional(),
  })
});
```

### 3.3 存储结构

#### 3.3.1 ClickHouse 表结构

性能数据存储在 `website_event` 表中，`event_type = 5` 标识性能事件：

```sql
-- db/clickhouse/migrations/09_add_performance.sql
ALTER TABLE umami.website_event ADD COLUMN lcp Nullable(Decimal(10, 1)) AFTER twclid;
ALTER TABLE umami.website_event ADD COLUMN inp Nullable(Decimal(10, 1)) AFTER lcp;
ALTER TABLE umami.website_event ADD COLUMN cls Nullable(Decimal(10, 4)) AFTER inp;
ALTER TABLE umami.website_event ADD COLUMN fcp Nullable(Decimal(10, 1)) AFTER cls;
ALTER TABLE umami.website_event ADD COLUMN ttfb Nullable(Decimal(10, 1)) AFTER fcp;
```

#### 3.3.2 物化视图排除

性能事件不会计入常规页面浏览统计，物化视图中明确排除：

```sql
-- db/clickhouse/migrations/09_add_performance.sql:84
sumIf(1, event_type NOT IN (2, 5)) views
```

---

## 四、统计口径详解

### 4.1 百分位统计

Performance 报表使用 **分位数统计** 而非平均值，这是 Web Vitals 的标准做法：

| 分位数 | 含义 | 适用场景 |
|--------|------|----------|
| **p50** | 中位数，50% 用户达到的性能 | 代表典型用户体验 |
| **p75** | 75% 用户达到的性能 | **默认展示值**，Google 推荐 |
| **p95** | 95% 用户达到的性能 | 衡量长尾性能，关注较差用户体验 |

### 4.2 数据库实现

#### ClickHouse 版本

```sql
-- src/queries/sql/reports/getPerformance.ts:142-144
select
  ${getDateSQL('created_at', unit, timezone)} t,
  quantile(0.5)(${metric}) as p50,
  quantile(0.75)(${metric}) as p75,
  quantile(0.95)(${metric}) as p95
from website_event
where website_event.website_id = {websiteId:UUID}
  and website_event.event_type = 5
  and website_event.created_at between {startDate:DateTime64} and {endDate:DateTime64}
group by t
order by t
```

#### PostgreSQL 版本

```sql
-- src/queries/sql/reports/getPerformance.ts:50-53
select
  ${getDateSQL('created_at', unit, timezone)} t,
  percentile_cont(0.5) within group (order by ${metric}) as p50,
  percentile_cont(0.75) within group (order by ${metric}) as p75,
  percentile_cont(0.95) within group (order by ${metric}) as p95
from website_event
group by t
order by t
```

### 4.3 汇总统计（Summary）

汇总查询返回每个指标在整个时间范围内的 p50/p75/p95 值：

```sql
-- src/queries/sql/reports/getPerformance.ts:159-175
select
  quantile(0.5)(lcp) as lcp_p50,
  quantile(0.75)(lcp) as lcp_p75,
  quantile(0.95)(lcp) as lcp_p95,
  -- inp, cls, fcp, ttfb 同理...
  count() as count
from website_event
where event_type = 5
```

返回数据结构：

```typescript
{
  summary: {
    lcp: { p50: number; p75: number; p95: number };
    inp: { p50: number; p75: number; p95: number };
    cls: { p50: number; p75: number; p95: number };
    fcp: { p50: number; p75: number; p95: number };
    ttfb: { p50: number; p75: number; p95: number };
    count: number;  // 样本总数
  };
}
```

### 4.4 维度下钻统计

支持按不同维度聚合性能指标：

| 维度 | 数据库字段 | 说明 |
|------|------------|------|
| 页面路径 | `url_path` | 按页面 URL 聚合 |
| 页面标题 | `page_title` | 按页面标题聚合 |
| 设备类型 | `device` | 按设备类型（mobile/desktop/tablet）聚合 |
| 浏览器 | `browser` | 按浏览器类型聚合 |

```sql
-- src/queries/sql/reports/getPerformanceMetrics.ts:81-96
select
  ${column} as "name",
  quantile(0.5)(${metric}) as p50,
  quantile(0.75)(${metric}) as p75,
  quantile(0.95)(${metric}) as p95,
  count() as count
from website_event
where event_type = 5
group by ${column}
order by p75 desc  -- 按 p75 降序排列，优先展示性能较差的
limit ${limit}
```

---

## 五、查询缓存机制

Umami Performance 报表涉及 **两种完全独立的缓存体系**，分别服务于数据采集上报和报表查询展示两个不同场景，二者边界清晰，互不干扰。

### 5.1 两种缓存的边界对比

| 维度 | React Query 查询缓存 | /api/send 会话缓存（Token Cache） |
|------|---------------------|----------------------------------|
| **所属层级** | 前端报表查询层 | 前端采集上报层 |
| **缓存内容** | `/api/reports/performance` 返回的报表数据（chart、summary、pages 等） | 会话标识（sessionId、visitId、iat） |
| **存储位置** | 浏览器内存（QueryClient 实例） | Tracker 脚本内存变量 + JWT Token |
| **传输方式** | 不传输，纯前端内存 | 通过 `x-umami-cache` 请求头发送给后端 |
| **生命周期** | 60 秒 staleTime，页面刷新即丢失 | sessionId 按月盐值轮换，visitId 30 分钟过期 |
| **触发场景** | 打开/切换报表页面时 | 上报性能数据 / 页面浏览事件时 |
| **缓存 Key** | queryKey 数组（所有查询参数序列化） | 无 Key，单用户单 Tracker 实例 |
| **失效条件** | 查询参数变化、60 秒过期、手动 refetch | 30 分钟过期、页面关闭、Tracker 重置 |
| **代码路径** | `useResultQuery` → `/api/reports/performance` | Tracker `send()` → `/api/send` |

> **关键结论**：两种缓存位于完全独立的代码路径，服务于不同业务目的。报表查询缓存与数据上报缓存之间没有任何交互。

---

### 5.2 React Query 查询缓存（报表层）

**配置位置**：`src/app/Providers.tsx:11-19`

```typescript
const client = new QueryClient({
  defaultOptions: {
    queries: {
      retry: false,
      refetchOnWindowFocus: false,
      staleTime: 1000 * 60,  // 数据 60 秒内视为新鲜
    },
  },
});
```

**缓存 Key 构成（精确映射）**：

```typescript
// src/components/hooks/queries/useResultQuery.ts:17-29
queryKey: [
  'reports',  // 命名空间
  {
    type: 'performance',              // 报表类型，固定值
    websiteId,                        // 网站 UUID，来自 URL path
    startDate: '2024-01-01T00:00:00Z',  // 来自 useDateParameters
    endDate: '2024-01-02T00:00:00Z',    // 来自 useDateParameters
    timezone: 'UTC',                  // 来自 useDateParameters
    unit: 'day',                      // 来自 useDateParameters
    metric: 'lcp',                    // 来自组件内部状态 selectedMetric
    ...filters,                       // 来自 URL 查询参数（路径、浏览器等过滤）
  },
]
```

**缓存命中与失效规则**：

| 场景 | 行为 | 原因 |
|------|------|------|
| 切换日期范围（如 24h → 7d） | 缓存失效，重新请求 | startDate/endDate/unit 变化 → queryKey 变化 |
| 切换指标（如 LCP → INP） | 缓存失效，重新请求 | metric 参数变化 → queryKey 变化 |
| 添加过滤条件（如浏览器=Chrome） | 缓存失效，重新请求 | filters 变化 → queryKey 变化 |
| 切换百分位（p75 → p95） | **缓存命中**，不请求 | selectedPercentile 不参与 queryKey，仅前端过滤展示 |
| 60 秒内重复访问同一报表 | 缓存命中，不请求 | staleTime 内数据视为新鲜 |
| 页面刷新后重新访问 | 缓存失效，重新请求 | QueryClient 实例重建，内存缓存丢失 |
| 切换到其他报表再切回 | 60 秒内命中，否则重查 | 同一 QueryClient 实例内多报表缓存共存 |

**缓存生命周期**：
```
组件挂载 → useResultQuery 执行 → 检查 queryKey 对应缓存
    │
    ├─ 缓存存在且 < 60s → 直接返回缓存数据 ✅
    │
    └─ 缓存不存在或 ≥ 60s → 发起 POST /api/reports/performance 请求
                                    │
                                    ▼
                          数据返回 → 写入缓存（60s 有效期）
                                    │
                                    ▼
                          组件接收数据 → 渲染页面
```

---

### 5.3 /api/send 会话缓存（采集层）

**设计目的**：避免每次上报都重新计算 sessionId 和 visitId（涉及加密哈希和数据库查询），提升上报性能。

**缓存 Token 生成与传递流程**：

```
Tracker 脚本 (浏览器)                          /api/send (后端)
       │                                            │
       │ 首次上报（无 cache header）                 │
       │───────────────────────────────────────────▶│
       │                                            │ 计算 sessionId = hash(IP + UA + salt)
       │                                            │ 计算 visitId = hash(sessionId + hourSalt)
       │                                            │ 生成 JWT Token = sign({sessionId, visitId, iat})
       │                                            │
       │ 响应：{ cache: Token, sessionId, visitId } │
       │◀───────────────────────────────────────────│
       │                                            │
       │ 保存 Token 到内存变量 cache                │
       │                                            │
       │ 后续上报（携带 x-umami-cache: Token）      │
       │───────────────────────────────────────────▶│
       │                                            │ 验证 Token 签名
       │                                            │ 解析 sessionId、visitId、iat
       │                                            │ 检查 iat 是否在 30 分钟内
       │                                            │ 未过期 → 复用，无需重新计算
       │                                            │
```

**Token 内容与验证**：

```typescript
// src/app/api/send/route.ts:100-112
interface Cache {
  websiteId: string;   // 网站 ID
  sessionId: string;   // 会话 ID（基于 IP + UA + 盐值哈希）
  visitId: string;     // 访问 ID（基于会话 ID + 小时盐值哈希）
  iat: number;         // 签发时间（Unix 时间戳）
}

// 后端验证逻辑
if (websiteId) {
  const cacheHeader = request.headers.get('x-umami-cache');
  if (cacheHeader) {
    const result = await parseToken(cacheHeader, secret());
    if (result) {
      cache = result;  // Token 验证通过，复用缓存值
    }
  }
}
```

**过期机制**：
- **Visit ID 过期**：`now - iat > 1800`（30 分钟），重新生成 visitId
- **Session ID 过期**：随盐值轮换周期（默认按月），与 `SALT_ROTATION` 环境变量相关
- **页面级失效**：Tracker 脚本随着页面刷新/关闭而销毁，内存中的 `cache` 变量丢失

> **注意**：此缓存仅存在于 Tracker 采集场景，与 Performance 报表查询完全无关。报表查询使用的是用户登录态 JWT（Authorization header），而非此 Tracker Token。

---

## 六、页面展示与数据衔接

### 6.1 报表页面结构

**页面路径**：`src/app/(main)/websites/[websiteId]/(reports)/performance/Performance.tsx`

页面分为以下区域：

```
┌─────────────────────────────────────────────────────────────┐
│  百分位选择器 (p50 / p75 / p95)                             │
├─────────────────────────────────────────────────────────────┤
│  性能指标卡片 (5 个：LCP / INP / CLS / FCP / TTFB)           │
├─────────────────────────────────────────────────────────────┤
│  趋势图表 (折线图，显示 p50/p75/p95 三条趋势线)              │
├─────────────────────────────────────────────────────────────┤
│  页面维度统计           │  环境维度统计                      │
│  ┌─────────┬─────────┐ │  ┌─────────┬─────────┐             │
│  │ 路径    │ 标题    │ │  │ 设备    │ 浏览器  │             │
│  └─────────┴─────────┘ │  └─────────┴─────────┘             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 数据获取流程

```
Performance.tsx
       │
       ▼
useResultQuery('performance', { websiteId, startDate, endDate, metric })
       │
       ▼
POST /api/reports/performance
       │
       ▼
并行执行 5 个查询：
  ├─ getPerformance() → chart + summary
  ├─ getPerformanceMetrics(url_path) → pages
  ├─ getPerformanceMetrics(page_title) → pageTitles
  ├─ getPerformanceMetrics(device) → devices
  └─ getPerformanceMetrics(browser) → browsers
       │
       ▼
返回聚合数据给前端
```

**API 响应结构**：

```typescript
{
  chart: Array<{ t: string; p50: number; p75: number; p95: number }>;
  summary: {
    lcp: { p50: number; p75: number; p95: number };
    inp: { p50: number; p75: number; p95: number };
    cls: { p50: number; p75: number; p95: number };
    fcp: { p50: number; p75: number; p95: number };
    ttfb: { p50: number; p75: number; p95: number };
    count: number;
  };
  pages: Array<{ name: string; p50: number; p75: number; p95: number; count: number }>;
  pageTitles: Array<{ name: string; p50: number; p75: number; p95: number; count: number }>;
  devices: Array<{ name: string; p50: number; p75: number; p95: number; count: number }>;
  browsers: Array<{ name: string; p50: number; p75: number; p95: number; count: number }>;
}
```

### 6.3 指标卡片（PerformanceCard）

**组件位置**：`src/components/metrics/PerformanceCard.tsx`

**评级规则**（基于 Google Web Vitals 标准）：

```typescript
// src/lib/constants.ts:106-112
export const WEB_VITALS_THRESHOLDS = {
  lcp: { good: 2500, poor: 4000, unit: 'ms' },
  inp: { good: 200, poor: 500, unit: 'ms' },
  cls: { good: 0.1, poor: 0.25, unit: '' },
  fcp: { good: 1800, poor: 3000, unit: 'ms' },
  ttfb: { good: 800, poor: 1800, unit: 'ms' },
};
```

| 评级 | 条件 | 徽章样式 |
|------|------|----------|
| Good | value ≤ good 阈值 | 绿色 |
| Needs Improvement | good < value ≤ poor | 黄色 |
| Poor | value > poor | 红色 |

### 6.4 图表展示

**时间序列数据处理**：

```typescript
// src/app/.../performance/Performance.tsx:80-134
const chartData = {
  datasets: [
    {
      label: 'p50',
      data: generateTimeSeries(
        data.chart.map((d: any) => ({ x: d.t, y: Number(d.p50) })),
        startDate, endDate, unit, dateLocale,
      ),
      borderColor: '#2680eb',  // 蓝色
      type: 'line', tension: 0.3,
    },
    {
      label: 'p75',
      data: generateTimeSeries(...),
      borderColor: '#9256d9',  // 紫色
    },
    {
      label: 'p95',
      data: generateTimeSeries(...),
      borderColor: '#44b556',  // 绿色
    },
  ],
};
```

**缺失值处理**：`generateTimeSeries` 函数会补全时间范围内缺失的数据点，保证图表连续性。

### 6.5 列表展示（ListTable）

**组件位置**：`src/components/metrics/ListTable.tsx`

**数据过滤**：只展示有数据的记录（值 > 0），并按选中的百分位排序：

```typescript
// src/app/.../performance/Performance.tsx:216-227
data.pages
  ?.filter(
    ({ p50, p75, p95 }: any) =>
      Number({ p50, p75, p95 }[selectedPercentile]) > 0,
  )
  .slice(0, 20)  // 最多展示 20 条
  .map(({ name, p50, p75, p95 }: any) => ({
    label: name,
    count: Number({ p50, p75, p95 }[selectedPercentile]),
    percent: 0,
  }))
```

---

## 七、完整链路时序图

```
浏览器 (Tracker)                          后端 (API)                          数据库
      │                                       │                                   │
      │ 页面加载，script 标签注入              │                                   │
      │──────────────────────────────────────▶│                                   │
      │                                       │                                   │
      │ initPerformance() 启动观测            │                                   │
      │ ├─ 监听 navigation (TTFB)             │                                   │
      │ ├─ 监听 paint (FCP)                   │                                   │
      │ ├─ 监听 largest-contentful-paint      │                                   │
      │ ├─ 监听 layout-shift (CLS)            │                                   │
      │ └─ 监听 event (INP)                   │                                   │
      │                                       │                                   │
      │ 10s 延迟 / 页面隐藏 / 路由跳转        │                                   │
      │──────────────────────────────────────▶│                                   │
      │ POST /api/send { type: 'performance' }│                                   │
      │                                       │ 校验 payload                     │
      │                                       │ ├─ Zod schema 校验               │
      │                                       │ ├─ Bot 检测                       │
      │                                       │ └─ IP 封禁检测                    │
      │                                       │                                   │
      │                                       │ saveEvent()                       │
      │                                       │──────────────────────────────────▶│
      │                                       │  INSERT INTO website_event        │
      │                                       │  (event_type=5, lcp, inp, cls...) │
      │                                       │◀──────────────────────────────────│
      │                                       │                                   │
      │◀──────────────────────────────────────│                                   │
      │ 返回 { cache: token, sessionId }      │                                   │
      │                                       │                                   │
      │                                       │                                   │
┌─────┴───────────────────────────────────────┴───────────────────────────────────┴─────┐
│                                       用户访问报表页面                                    │
└─────┬───────────────────────────────────────┬───────────────────────────────────┬─────┘
      │                                       │                                   │
      │ 进入 Performance 页面                 │                                   │
      │──────────────────────────────────────▶│                                   │
      │ POST /api/reports/performance         │                                   │
      │                                       │                                   │
      │                                       │ 并行执行 5 个 SQL 查询             │
      │                                       │ ├─ getPerformance (chart+summary) │
      │                                       │ ├─ getPerformanceMetrics (path)   │
      │                                       │ ├─ getPerformanceMetrics (title)  │
      │                                       │ ├─ getPerformanceMetrics (device) │
      │                                       │ └─ getPerformanceMetrics (browser)│
      │                                       │──────────────────────────────────▶│
      │                                       │  SELECT quantile(0.5/0.75/0.95)   │
      │                                       │  FROM website_event                │
      │                                       │  WHERE event_type = 5              │
      │                                       │  GROUP BY time/column              │
      │                                       │◀──────────────────────────────────│
      │◀──────────────────────────────────────│                                   │
      │ 返回完整数据                            │                                   │
      │                                       │                                   │
      │ 渲染页面：卡片、图表、列表             │                                   │
      │  (React Query 缓存 60s)               │                                   │
```

---

## 八、关键技术决策分析

### 8.1 为什么使用分位数而不是平均值？

1. **抗异常值**：性能数据通常呈长尾分布，平均值容易被极端值影响
2. **行业标准**：Google Web Vitals 推荐使用 p75 作为核心指标
3. **用户导向**：分位数能更好地反映真实用户体验，尤其是 p95 关注长尾用户

### 8.2 为什么性能数据独立存储为 event_type=5？

1. **隔离统计**：性能数据不应计入页面浏览量（PV）统计
2. **数据稀疏性**：性能数据上报频率低于页面浏览，单独过滤更高效
3. **可扩展性**：方便后续添加更多性能指标而不影响现有逻辑

### 8.3 为什么使用 PerformanceObserver 而不是直接读取 performance.timing？

1. **被动监听**：数据就绪时立即获取，无需轮询
2. **缓冲机制**：`buffered: true` 可以获取注册前已经发生的条目
3. **动态更新**：LCP 会多次触发，始终取最新（最大）值
4. **INP 支持**：只有 PerformanceObserver 才能获取交互事件的 interactionId

### 8.4 前端缓存 60 秒的权衡？

**优点**：
- 减少数据库压力，相同查询在 60 秒内直接走缓存
- 提升页面切换速度，报表间跳转无需重新加载

**缺点**：
- 最新数据最多有 60 秒延迟
- 对于实时监控场景可能不够及时

**适用场景**：性能报表本身是用于趋势分析，秒级实时性不是核心需求。

---

## 九、性能优化建议（基于代码分析）

### 9.1 采集端优化

**当前问题**：INP 采集在每次交互后都要排序所有交互，时间复杂度 O(n log n)

**优化建议**：维护一个固定大小的小根堆，只保留最大的几个值，将时间复杂度降为 O(n log k)

### 9.2 存储端优化

**当前问题**：性能数据直接写入 website_event 表，该表数据量巨大

**优化建议**：
- 考虑单独的 performance_event 表
- 按日期粒度做物化视图预聚合，进一步提升查询速度

### 9.3 查询端优化

**当前问题**：每次请求并行 5 个独立的 SQL 查询，扫描相同数据 5 次

**优化建议**：
- 合并为单次查询，在应用层拆分结果
- 或使用 ClickHouse 的 `GROUP BY WITH ROLLUP` 一次性返回多维度聚合结果

---

## 十、总结

Umami Performance 报表遵循了现代 Web 性能监控的最佳实践：

1. **采集标准**：严格遵循 Google Web Vitals 规范，使用原生 Performance API
2. **统计科学**：采用分位数统计，真实反映用户体验分布
3. **架构清晰**：采集、上报、存储、查询、展示各层职责明确
4. **缓存合理**：60 秒客户端缓存平衡了实时性和性能
5. **多维度分析**：支持按页面、设备、浏览器等维度下钻

整个系统设计兼顾了准确性、性能和可扩展性，是一个优秀的开源性能监控实现。

---

**核心文件索引**：

| 模块 | 文件路径 |
|------|----------|
| 前端采集 | `src/tracker/index.js` |
| 上报接口 | `src/app/api/send/route.ts` |
| 报表查询 | `src/app/api/reports/performance/route.ts` |
| 统计查询 | `src/queries/sql/reports/getPerformance.ts` |
| 维度统计 | `src/queries/sql/reports/getPerformanceMetrics.ts` |
| 页面组件 | `src/app/(main)/websites/[websiteId]/(reports)/performance/Performance.tsx` |
| 指标卡片 | `src/components/metrics/PerformanceCard.tsx` |
| 阈值定义 | `src/lib/constants.ts` |
| 数据库迁移 | `db/clickhouse/migrations/09_add_performance.sql` |
