# Umami 埋点脚本构建与配置参数对应关系

## 一、整体架构概览

Umami 的埋点系统采用**构建时静态注入 + 运行时动态配置**相结合的方式。关键配置参数通过三层独立的机制传递，各层之间没有自动联动：

1. **构建时**：Rollup 插件静态替换环境变量到脚本代码中
2. **运行时（前端）**：HTML 脚本标签的 `data-*` 属性动态配置
3. **运行时（服务端）**：服务端环境变量配置

> **重要纠正**：站点配置中的 `Website.domain` 字段与前端脚本的 `data-domains` 属性**没有任何自动关联**，是两个独立的配置体系。

---

## 二、埋点脚本构建流程

### 2.1 构建配置

**构建配置文件**：`rollup.tracker.config.js:1-20`

```javascript
export default {
  input: 'src/tracker/index.js',
  output: {
    file: 'public/script.js',
    format: 'iife',
  },
  plugins: [
    replace({
      __COLLECT_API_HOST__: process.env.COLLECT_API_HOST || '',
      __COLLECT_API_ENDPOINT__: process.env.COLLECT_API_ENDPOINT || '/api/send',
      delimiters: ['', ''],
      preventAssignment: true,
    }),
    terser({ compress: { evaluate: false } }),
  ],
};
```

**构建命令**：`npm run build-tracker` → `rollup -c rollup.tracker.config.js`

### 2.2 构建时参数注入机制

使用 `@rollup/plugin-replace` 在构建阶段将环境变量值**静态替换**到脚本源码中：

| 占位符 | 环境变量 | 默认值 | 作用 |
|--------|---------|--------|------|
| `__COLLECT_API_HOST__` | `COLLECT_API_HOST` | `''` | 收集服务端主机地址 |
| `__COLLECT_API_ENDPOINT__` | `COLLECT_API_ENDPOINT` | `/api/send` | 数据收集接口路径 |

**替换原理**：在 Rollup 打包过程中，直接将源码中的字符串 `__COLLECT_API_HOST__` 替换为环境变量的实际值，属于**编译时静态替换**。

### 2.3 运行时二次更新脚本

**更新脚本**：`scripts/update-tracker.js:1-16`

在服务启动时（`npm run start-docker`），`update-tracker` 脚本会再次读取 `COLLECT_API_ENDPOINT` 环境变量，对已构建的 `public/script.js` 进行二次字符串替换：

```javascript
if (endPoint) {
  const file = path.resolve(process.cwd(), 'public/script.js');
  const tracker = fs.readFileSync(file);
  fs.writeFileSync(file, tracker.toString().replace(/\/api\/send/g, endPoint));
}
```

> **设计意图**：支持在 Docker 等无需重新构建的部署场景下，通过环境变量修改收集端点路径。

---

## 三、同源域名（Domains）配置机制

### 3.1 两个独立的域名配置体系

Umami 中存在两个独立的域名配置，**彼此之间没有自动关联**：

| 配置项 | 存储位置 | 作用范围 | 自动注入 |
|--------|---------|---------|---------|
| `Website.domain` | 数据库 | 后台管理展示、统计查询参考 | ❌ 不自动注入 |
| `data-domains` | 脚本标签属性 | 前端域名白名单校验 | ❌ 不自动生成 |

### 3.2 Website.domain 字段

**数据库存储**：`prisma/schema.prisma:69`

```prisma
model Website {
  id        String    @id() @map("website_id") @db.Uuid
  name      String    @db.VarChar(100)
  domain    String?   @db.VarChar(500)  // 仅用于展示，不自动注入
  // ...
}
```

**字段用途**：
- 在管理后台网站列表中展示（`WebsiteData.tsx`、`useBoardEntityBadgeProps.ts`）
- 仅作为元数据存储，数据收集时**不参与任何校验**
- 创建/更新时仅做格式校验（`DOMAIN_REGEX` 正则匹配，`src/lib/constants.ts:249`）

### 3.3 管理后台跟踪代码生成

**生成逻辑**：`src/app/(main)/websites/[websiteId]/settings/WebsiteTrackingCode.tsx:31`

```javascript
const code = `<script defer src="${url}" data-website-id="${websiteId}"></script>`;
```

> **关键发现**：管理后台自动生成的跟踪代码**只包含 `data-website-id` 属性**，**不包含** `data-domains` 属性。用户需要手动添加 `data-domains` 到脚本标签中。

### 3.4 前端 data-domains 配置

**脚本标签属性**：`src/tracker/index.js:37-41`

```javascript
const domain = config('domains') || '';
const domains = domain.split(',').map(n => n.trim());
```

通过脚本标签的 `data-domains` 属性传入，多个域名用**逗号分隔**，需要用户**手动添加**：

```html
<script async defer 
  src="http://umami.example.com/script.js"
  data-website-id="xxx"
  data-domains="example.com,sub.example.com"></script>
```

### 3.5 前端域名校验逻辑

**校验逻辑**：`src/tracker/index.js:156-161`

```javascript
const trackingDisabled = () =>
  disabled ||
  !website ||
  localStorage?.getItem('umami.disabled') ||
  (domain && !domains.includes(hostname)) ||  // 域名白名单校验
  (dnt && hasDoNotTrack());
```

**校验规则**：
- 如果设置了 `data-domains`，当前页面的 `hostname` 必须在域名列表中才会发送数据
- 如果未设置 `data-domains`，则不进行域名过滤（所有域名都可发送）
- 域名匹配是**精确字符串匹配**，不支持通配符
- 比较的是 `window.location.hostname`，自动排除端口号

---

## 四、收集端点（Endpoint）配置

### 4.1 端点计算逻辑

**端点组装**：`src/tracker/index.js:42-44`

```javascript
const host =
  hostUrl || '__COLLECT_API_HOST__' || currentScript.src.split('/').slice(0, -1).join('/');
const endpoint = `${host.replace(/\/$/, '')}__COLLECT_API_ENDPOINT__`;
```

**HOST 优先级**（从高到低）：
1. 脚本标签 `data-host-url` 属性（运行时动态配置）
2. 构建时注入的 `__COLLECT_API_HOST__`（编译时静态替换）
3. 从脚本 `src` URL 自动推断（脚本所在域名，运行时动态计算）

**PATH 来源**：
- 构建时注入的 `__COLLECT_API_ENDPOINT__`（默认 `/api/send`）
- 可通过 `scripts/update-tracker.js` 运行时二次替换

### 4.2 服务端接收端点

**主收集端点**：`src/app/api/send/route.ts:66-322`

支持三种收集类型：
- `event`：页面浏览和自定义事件
- `identify`：用户识别
- `performance`：性能指标（Core Web Vitals）

**短链追踪端点**：`src/app/(collect)/q/[slug]/route.ts`
**像素追踪端点**：`src/app/(collect)/p/[slug]/route.ts`

### 4.3 数据发送逻辑

**发送函数**：`src/tracker/index.js:163-195`

```javascript
const send = async (payload, type = 'event') => {
  if (trackingDisabled()) return;  // 先执行忽略规则检查

  const callback = window[beforeSend];
  if (typeof callback === 'function') {
    payload = await Promise.resolve(callback(type, payload));
  }
  if (!payload) return;

  const res = await fetch(endpoint, {
    keepalive: true,
    method: 'POST',
    body: JSON.stringify({ type, payload }),
    headers: {
      'Content-Type': 'application/json',
      ...(typeof cache !== 'undefined' && { 'x-umami-cache': cache }),
    },
    credentials,
  });
};
```

---

## 五、域名校验边界与前后端责任划分

### 5.1 前端校验（唯一防线）

**唯一校验点**：`src/tracker/index.js:160`

```javascript
(domain && !domains.includes(hostname))
```

**前端校验责任**：
- 读取 `data-domains` 属性，解析为域名列表
- 检查当前 `window.location.hostname` 是否在白名单中
- 如不在白名单中，阻止发送任何数据

**前端校验的局限性**：
- 可通过浏览器开发者工具绕过（修改 localStorage 或脚本属性）
- 依赖用户正确配置 `data-domains` 属性
- 不与数据库中的 `Website.domain` 关联

### 5.2 服务端处理（无域名校验）

**服务端数据处理**：`src/app/api/send/route.ts:177-184`

```javascript
const base = hostname ? `https://${hostname}` : 'https://localhost';
const currentUrl = new URL(url, base);
const urlDomain = currentUrl.hostname.replace(/^www./, '');
```

**存储逻辑**：`src/app/api/send/route.ts:234`

```javascript
hostname: hostname || urlDomain,
```

> **重要发现**：**服务端在数据收集阶段完全不校验 `hostname` 与 `Website.domain` 的匹配关系**。`Website.domain` 字段在 `/api/send` 端点中**从未被读取或使用**。

### 5.3 服务端查询时的引用域名过滤

**查询时过滤**：`src/queries/sql/getValues.ts:26`、`src/queries/sql/pageviews/getPageviewMetrics.ts:50` 等多处

```sql
and website_event.referrer_domain != regexp_replace(website_event.hostname, '^www.', '')
and website_event.referrer_domain != ''
```

**过滤逻辑说明**：
- 这是**统计查询阶段**的过滤，不是数据收集阶段的校验
- 目的是排除**网站自身的引用流量**（即用户从网站 A 页面跳转到网站 B 页面时，不将 A 页面算作外部来源）
- 比较的是同一条记录的 `referrer_domain` 和 `hostname` 字段
- 与 `Website.domain` 字段无关

### 5.4 校验责任边界总结表

| 校验环节 | 前端 | 服务端 | 说明 |
|---------|------|--------|------|
| 域名白名单校验 | ✅ 负责 | ❌ 不负责 | 前端唯一防线，基于 `data-domains` |
| Website.domain 读取 | ❌ 不读取 | ❌ 收集时不读取 | 仅后台展示用 |
| 自身引用流量过滤 | ❌ 不做 | ✅ 查询时做 | 统计查询时排除内部跳转 |
| 机器人检测 | ❌ 不做 | ✅ 负责 | 服务端 `isbot()` 检测 |
| IP 黑名单 | ❌ 不做 | ✅ 负责 | 服务端 `IGNORE_IP` 检测 |

---

## 六、忽略规则触发条件与执行顺序

### 6.1 忽略规则执行流程图

```
请求开始
    │
    ▼
┌─────────────────────────┐
│  前端：trackingDisabled()  │
└─────────┬───────────────┘
          │
    ┌─────┴─────┐
    │           │
    ▼           ▼
  禁用?     未禁用?
    │           │
    ▼           ▼
  终止       继续
              │
              ▼
┌─────────────────────────┐
│  服务端：忽略规则检查      │
└─────────┬───────────────┘
          │
    ┌─────┼──────────────────┐
    │     │                  │
    ▼     ▼                  ▼
  机器人  IP黑名单         其他校验
  检测    检测
    │     │                  │
    └─────┼──────────────────┘
          │
    ┌─────┴─────┐
    │           │
    ▼           ▼
  忽略?     未忽略?
    │           │
    ▼           ▼
  拒绝       保存数据
```

### 6.2 前端忽略规则（按检查顺序）

**检查入口**：`src/tracker/index.js:156-161`

| 序号 | 规则 | 触发条件 | 实现位置 |
|-----|------|---------|---------|
| 1 | 全局禁用标记 | `disabled === true`（由服务端返回设置） | `src/tracker/index.js:157` |
| 2 | Website ID 缺失 | `!website`（未配置 `data-website-id`） | `src/tracker/index.js:158` |
| 3 | 本地禁用标记 | `localStorage.getItem('umami.disabled')` 存在 | `src/tracker/index.js:159` |
| 4 | 域名白名单 | 设置了 `data-domains` 且当前 hostname 不在列表中 | `src/tracker/index.js:160` |
| 5 | DNT 尊重 | 设置了 `data-do-not-track="true"` 且浏览器开启了 DNT | `src/tracker/index.js:161` |

**附加前端忽略条件**：
- `data-auto-track="false"`：禁用自动页面浏览追踪，但自定义事件仍可手动触发
  - 实现位置：`src/tracker/index.js:33`、`404-410`

### 6.3 服务端忽略规则（按检查顺序）

**检查入口**：`src/app/api/send/route.ts:131-138`

| 序号 | 规则 | 触发条件 | 实现位置 |
|-----|------|---------|---------|
| 1 | 机器人检测 | `isbot(userAgent)` 返回 true，且 `DISABLE_BOT_CHECK` 未设置 | `src/app/api/send/route.ts:131-133` |
| 2 | IP 黑名单 | `IGNORE_IP` 环境变量包含请求 IP（支持精确匹配和 CIDR） | `src/app/api/send/route.ts:136-138` |
| 3 | 本地 IP 跳过定位 | IP 是 localhost 或内网 IP，跳过地理位置解析 | `src/lib/detect.ts:81` |

#### 6.3.1 IP 忽略规则详解

**实现**：`src/lib/detect.ts:140-170`

```javascript
export function hasBlockedIp(clientIp: string) {
  const ignoreIps = process.env.IGNORE_IP;
  if (ignoreIps) {
    const ips = ignoreIps.split(',').map(n => n.trim());
    return ips.find(ip => {
      if (ip === clientIp) return true;  // 精确匹配
      // CIDR 格式匹配
      if (ip.indexOf('/') > 0) {
        const addr = ipaddr.parse(clientIp);
        const range = ipaddr.parseCIDR(ip);
        if (addr.kind() === range[0].kind() && addr.match(range)) {
          return true;
        }
      }
      return false;
    });
  }
  return false;
}
```

**配置方式**：环境变量 `IGNORE_IP=192.168.1.1,10.0.0.0/8,172.16.0.0/12`

#### 6.3.2 机器人检测详解

**实现**：`src/app/api/send/route.ts:131-133`

```javascript
if (!process.env.DISABLE_BOT_CHECK && isbot(userAgent)) {
  return json({ beep: 'boop' });
}
```

- 使用 `isbot` 库检测爬虫 User-Agent
- 可通过 `DISABLE_BOT_CHECK=true` 完全禁用
- 返回 `{ beep: 'boop' }` 伪装成正常响应，避免被爬虫探测

---

## 七、完整的参数注入流程

### 7.1 参数注入全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                      构建阶段 (Build Time)                        │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
              ┌──────────────────────────────────┐
              │  rollup.tracker.config.js         │
              │  - __COLLECT_API_HOST__           │
              │  - __COLLECT_API_ENDPOINT__       │
              └─────────────┬────────────────────┘
                                  │ 静态替换
                                  ▼
              ┌──────────────────────────────────┐
              │  public/script.js (已构建)        │
              └─────────────┬────────────────────┘
                                  │
┌─────────────────────────────────────────────────────────────────┐
│                      运行阶段 (Runtime)                           │
└─────────────────────────────────────────────────────────────────┘
                                  │
              ┌──────────────────────────────────┐
              │  scripts/update-tracker.js        │
              │  （可选）二次替换 API_ENDPOINT    │
              └─────────────┬────────────────────┘
                                  │
                                  ▼
              ┌──────────────────────────────────┐
              │  HTML 脚本标签                     │
              │  ├─ src=".../script.js"           │
              │  ├─ data-website-id="..."         │
              │  ├─ data-host-url="..." (可选)    │
              │  ├─ data-domains="..." (可选)     │
              │  └─ ... 其他 data-* 属性          │
              └─────────────┬────────────────────┘
                                  │ 运行时读取
                                  ▼
              ┌──────────────────────────────────┐
              │  src/tracker/index.js             │
              │  执行时读取 data-* 属性           │
              └─────────────┬────────────────────┘
                                  │
                                  ▼
              ┌──────────────────────────────────┐
              │  服务端环境变量                    │
              │  ├─ COLLECT_API_HOST              │
              │  ├─ COLLECT_API_ENDPOINT          │
              │  ├─ IGNORE_IP                     │
              │  ├─ DISABLE_BOT_CHECK             │
              │  └─ SKIP_LOCATION_HEADERS         │
              └──────────────────────────────────┘
```

### 7.2 各配置项的注入路径总结

| 配置项 | 注入时机 | 注入方式 | 可覆盖方式 |
|--------|---------|---------|-----------|
| **收集端点 HOST** | 构建时 + 运行时 | Rollup 静态替换 `__COLLECT_API_HOST__` | `data-host-url` 属性优先级最高 |
| **收集端点 PATH** | 构建时 + 运行时 | Rollup 静态替换 `__COLLECT_API_ENDPOINT__` | `update-tracker.js` 二次替换 |
| **同源域名白名单** | 运行时（前端） | 手动添加 `data-domains` 属性 | 脚本标签属性动态配置，与 Website.domain 无关 |
| **域名校验** | 运行时（前端） | `domains.includes(hostname)` 检查 | 服务端无强制校验，完全依赖前端 |
| **IP 忽略** | 运行时（服务端） | 读取 `IGNORE_IP` 环境变量 | 服务端环境变量配置 |
| **机器人忽略** | 运行时（服务端） | `isbot()` 检测 + `DISABLE_BOT_CHECK` | 环境变量开关 |

---

## 八、脚本完整参数列表

### 8.1 脚本标签 `data-*` 属性

| 属性 | 变量 | 类型 | 默认值 | 说明 |
|-----|------|------|--------|------|
| `data-website-id` | `website` | string | - | **必填**，网站 UUID |
| `data-host-url` | `hostUrl` | string | - | 收集服务端 URL，优先级最高 |
| `data-domains` | `domain` | string | - | 允许的域名列表，逗号分隔，手动添加 |
| `data-auto-track` | `autoTrack` | boolean | `true` | 是否自动追踪页面浏览 |
| `data-do-not-track` | `dnt` | boolean | `false` | 是否遵守浏览器 DNT 设置 |
| `data-exclude-search` | `excludeSearch` | boolean | `false` | 是否排除 URL 搜索参数 |
| `data-exclude-hash` | `excludeHash` | boolean | `false` | 是否排除 URL hash |
| `data-fetch-credentials` | `credentials` | string | `'omit'` | fetch credentials 选项 |
| `data-before-send` | `beforeSend` | string | - | 发送前回调函数名 |
| `data-tag` | `tag` | string | - | 事件标签 |
| `data-performance` | `perf` | boolean | `false` | 是否启用性能监控 |

**完整配置示例**：

```html
<script async defer
  src="https://umami.example.com/script.js"
  data-website-id="b59e9c65-ae32-47f1-8400-119fcf4861c4"
  data-domains="example.com,www.example.com,app.example.com"
  data-host-url="https://umami-api.example.com"
  data-exclude-search="true"
  data-exclude-hash="false"
  data-do-not-track="true"
  data-performance="true"></script>
```

### 8.2 环境变量配置

| 变量 | 作用 | 生效时机 | 说明 |
|-----|------|---------|------|
| `COLLECT_API_HOST` | 收集服务端 HOST | 构建时 + 运行时 | 替换 `__COLLECT_API_HOST__` |
| `COLLECT_API_ENDPOINT` | 收集端点路径 | 构建时 + 运行时 | 替换 `__COLLECT_API_ENDPOINT__`，支持运行时二次替换 |
| `IGNORE_IP` | 忽略 IP 列表 | 运行时（服务端） | 逗号分隔，支持精确 IP 和 CIDR 格式 |
| `DISABLE_BOT_CHECK` | 禁用机器人检测 | 运行时（服务端） | `true` 时禁用 `isbot()` 检测 |
| `SKIP_LOCATION_HEADERS` | 跳过地理位置头 | 运行时（服务端） | `true` 时跳过从 HTTP 头解析地理位置 |

---

## 九、Hostname 处理逻辑深度分析

### 9.1 Hostname 三阶段处理流程总览

Umami 中 `hostname` 字段在三个关键阶段有不同的处理逻辑，且**三处的信任边界和归一化规则不一致**，存在统计偏差风险：

```
┌─────────────────────────┐    ┌─────────────────────────┐    ┌─────────────────────────┐
│   1. Tracker Payload    │    │   2. /api/send 入库      │    │   3. 查询时 referrer 过滤  │
│                         │    │                         │    │                         │
│  - 来源: window.location│    │  - Zod 校验: max(100)   │    │  - Prisma: 去 www 后比较 │
│  - 可被 track() 覆盖    │    │  - 优先用 payload 值    │    │  - ClickHouse: 直接比较 │
│  - 可被 beforeSend 修改 │    │  - 无合法性校验         │    │  - 无二次校验          │
└───────────┬─────────────┘    └───────────┬─────────────┘    └───────────┬─────────────┘
            │                            │                            │
            ▼                            ▼                            ▼
      完全信任客户端               完全信任 payload               完全信任存储值
      (可控性极高)                 (无任何校验)                 (无二次校验)
```

---

### 9.2 阶段一：Tracker Payload 中的 Hostname

#### 9.2.1 默认来源

**源码位置**：`src/tracker/index.js:14`、`src/tracker/index.js:69`

```javascript
// 第 14 行：从 window.location 解构
const { hostname, href, origin } = location;

// 第 64-74 行：getPayload() 中使用
const getPayload = () => ({
  website,
  screen,
  language,
  title: document.title,
  hostname,  // 直接使用 window.location.hostname
  url: currentUrl,
  referrer: currentRef,
  tag,
  id: identity ? identity : undefined,
});
```

**默认值**：`window.location.hostname`，即当前页面的域名（不含端口）。

#### 9.2.2 Hostname 的可控性（关键风险点）

`hostname` 并非只读字段，可以通过**三种方式被覆盖**，这是设计上允许的功能（类型定义中 `hostname` 是可选字段）：

**方式一：通过 `track()` 直接传入对象覆盖**
**源码位置**：`src/tracker/index.js:209`

```javascript
const track = (name, data) => {
  // ...
  if (typeof name === 'object') return send({ ...name });  // 完全覆盖 payload
  // ...
};
```

**示例**：
```javascript
// 可以完全替换 hostname 为任意值
umami.track({
  hostname: 'fake-website.com',  // 伪造域名
  url: '/page',
  title: 'Fake Page'
});
```

**方式二：通过 `track()` 函数式参数修改**
**源码位置**：`src/tracker/index.js:210`

```javascript
const track = (name, data) => {
  // ...
  if (typeof name === 'function') return send(name(getPayload()));  // 函数式修改
  // ...
};
```

**示例**：
```javascript
umami.track((payload) => ({
  ...payload,
  hostname: 'another-fake.com'  // 修改 hostname
}));
```

**方式三：通过 `beforeSend` 回调修改**
**源码位置**：`src/tracker/index.js:166-170`

```javascript
const send = async (payload, type = 'event') => {
  if (trackingDisabled()) return;
  const callback = window[beforeSend];
  if (typeof callback === 'function') {
    payload = await Promise.resolve(callback(type, payload));  // 发送前可修改任何字段
  }
  // ...
};
```

**类型定义确认**：`src/tracker/index.d.ts:137`
```typescript
(properties: PageViewProperties): Promise<string>;
// PageViewProperties 包含 hostname 字段
```

> **重要结论**：`payload.hostname` 完全可控，没有任何前端层面的保护。这是设计上的灵活性，但也带来了数据伪造风险。

---

### 9.3 阶段二：/api/send 入库时的 Hostname 处理

#### 9.3.1 Schema 校验

**源码位置**：`src/app/api/send/route.ts:32`

```javascript
const schema = z.object({
  // ...
  payload: z.object({
    // ...
    hostname: z.string().max(100).optional(),  // 仅限制长度 ≤ 100，不校验内容
    // ...
  }),
});
```

**校验边界**：
- ✅ 长度校验：最多 100 字符
- ❌ 格式校验：不校验是否为合法域名格式
- ❌ 范围校验：不校验是否在 `Website.domain` 列表中
- ❌ 绑定校验：不校验与 `websiteId` 的关联关系

#### 9.3.2 入库逻辑

**源码位置**：`src/app/api/send/route.ts:177-184`、`234`

```javascript
if (type === COLLECTION_TYPE.event) {
  // 第 178 行：使用 payload.hostname 作为 base URL
  const base = hostname ? `https://${hostname}` : 'https://localhost';
  const currentUrl = new URL(url, base);
  
  // ...
  
  // 第 184 行：从 URL 解析 hostname（去 www 前缀）
  const urlDomain = currentUrl.hostname.replace(/^www./, '');
  
  // ...
  
  // 第 234 行：入库时优先使用 payload.hostname
  hostname: hostname || urlDomain,
}
```

**处理逻辑分解**：

1. **优先级**：`payload.hostname` > 从 `url` 解析的 `urlDomain`
2. **归一化不一致**：
   - 对 `urlDomain` 做了 `replace(/^www./, '')` 归一化（去 `www.` 前缀）
   - 对 `payload.hostname` **没有做任何归一化**，直接存储
3. **Base URL 影响**：如果 `payload.hostname` 被伪造，`new URL(url, base)` 会基于伪造的域名解析，进一步影响 `urlDomain`

**极端场景**：
- 如果 `payload.hostname = 'evil.com'`，且 `url = '/page'`
- 则 `base = 'https://evil.com'`
- `currentUrl = new URL('/page', 'https://evil.com') = 'https://evil.com/page'`
- `urlDomain = 'evil.com'`（去 www 后）
- 最终入库 `hostname = 'evil.com'`（优先使用 payload 值）

> **重要发现**：服务端对 `payload.hostname` 完全信任，没有任何合法性校验。这是信任边界最宽松的环节。

---

### 9.4 阶段三：查询时 referrer 过滤中的 Hostname

#### 9.4.1 过滤逻辑

查询 referrer_domain 时，会自动过滤掉**网站自身的引用流量**（即用户从网站 A 页面跳转到 B 页面时，不将 A 算作外部来源）。

**Prisma（关系型数据库）版本**：
**源码位置**：`src/queries/sql/getValues.ts:26-27`

```sql
and website_event.referrer_domain != regexp_replace(website_event.hostname, '^www.', '')
and website_event.referrer_domain != ''
```

**ClickHouse 版本**：
**源码位置**：`src/queries/sql/getValues.ts:83`

```sql
and referrer_domain != hostname
and referrer_domain != ''
```

#### 9.4.2 三处归一化规则不一致

| 阶段 | 归一化规则 | 代码位置 |
|-----|-----------|---------|
| Tracker payload | 无（直接使用 `window.location.hostname`） | `src/tracker/index.js:69` |
| /api/send 入库 | `urlDomain` 去 `www.`，但 `payload.hostname` 不去 | `src/app/api/send/route.ts:184` |
| Prisma 查询 | 对 `hostname` 字段去 `www.` 后比较 | `src/queries/sql/getValues.ts:26` |
| ClickHouse 查询 | 直接比较，不去 `www.` | `src/queries/sql/getValues.ts:83` |

**不一致导致的问题**：
- 如果真实域名是 `www.example.com`，tracker 发送的是 `www.example.com`
- 入库时如果走 `urlDomain` 分支，会变成 `example.com`（去 www）
- Prisma 查询时会把存储的 `hostname` 再去一次 `www.`（但已经没有了）
- ClickHouse 查询时直接比较 `referrer_domain != hostname`

---

### 9.5 Hostname 可控性带来的统计偏差风险

#### 9.5.1 风险场景一：伪造 Hostname 使 referrer 过滤失效

**攻击目标**：让网站自身的内部跳转被算作外部来源，虚高引荐流量。

**攻击方式**：
```javascript
// 攻击者在页面上执行
umami.track({
  hostname: 'external-fake.com',  // 伪造为外部域名
  url: '/product-page',
  referrer: 'https://example.com/home'  // 真实来源是本网站
});
```

**结果**：
- 入库 `hostname = 'external-fake.com'`
- 入库 `referrer_domain = 'example.com'`（去 www 后）
- 查询时：`referrer_domain ('example.com') != hostname ('external-fake.com')` → **不会被过滤**
- 这条内部跳转流量被错误统计为外部来源

**统计影响**：
- Referrer Domain 统计中出现大量虚假的外部来源
- 直接来源（Direct）统计偏低
- 渠道归因分析完全失真

#### 9.5.2 风险场景二：伪造 Hostname 与 referrer 相同，过滤所有外部来源

**攻击目标**：让所有外部来源流量都被过滤掉，掩盖真实流量来源。

**攻击方式**：
```javascript
// 攻击者在页面上执行，统一将 hostname 设为与 referrer 相同
umami.track((payload) => ({
  ...payload,
  // 从 referrer 中提取域名作为 hostname
  hostname: new URL(payload.referrer || window.location.href).hostname
}));
```

**结果**：
- 如果用户来自 `https://google.com/search`，则 `referrer_domain = 'google.com'`
- 伪造 `hostname = 'google.com'`
- 查询时：`referrer_domain ('google.com') != hostname ('google.com')` → **条件不成立，被过滤**
- 所有外部来源都被过滤掉了

**统计影响**：
- Referrer Domain 统计为 0 或接近 0
- 直接来源（Direct）统计虚高（因为其他渠道都被过滤了）
- 无法判断真实的流量来源

#### 9.5.3 风险场景三：随机 Hostname 导致统计碎片化

**攻击方式**：
```javascript
// 每次请求生成随机 hostname
umami.track((payload) => ({
  ...payload,
  hostname: `random-${Math.random().toString(36).substring(7)}.com`
}));
```

**结果**：
- 每条记录的 hostname 都不同
- 查询时 `referrer_domain != hostname` 几乎总是成立
- 所有流量都被算作外部来源，但 hostname 毫无意义

**统计影响**：
- Hostname 维度的统计完全不可用
- Referrer Domain 统计包含大量噪音
- 无法按域名进行数据聚合

#### 9.5.4 非攻击场景：正常使用中的误配置

即使没有恶意攻击，正常使用中也可能出现问题：

```javascript
// 开发者在 SPA 路由切换时手动调用 track
router.on('routeChange', () => {
  umami.track({
    url: router.pathname,
    title: document.title
    // 漏掉了 hostname，会用 window.location.hostname（通常没问题）
  });
});

// 但如果开发者不小心传了错误的 hostname
umami.track({
  hostname: 'staging.example.com',  // 复制粘贴时忘记修改
  url: '/home'
});
```

---

### 9.6 风险总结表

| 风险点 | 影响程度 | 利用难度 | 说明 |
|-------|---------|---------|------|
| payload.hostname 可任意伪造 | ⭐⭐⭐⭐⭐ | ⭐⭐ | 任何访问者都可通过控制台执行 |
| 服务端无 hostname 校验 | ⭐⭐⭐⭐⭐ | ⭐ | 架构设计上完全信任客户端 |
| 三处归一化规则不一致 | ⭐⭐⭐ | ⭐ | 正常使用也可能出现统计偏差 |
| referrer 过滤依赖不可信字段 | ⭐⭐⭐⭐ | ⭐⭐ | 过滤逻辑建立在沙滩上 |
| 类型定义允许修改 hostname | ⭐⭐⭐ | ⭐⭐ | 设计上的灵活性被滥用 |

---

### 9.7 CORS 策略、参数校验与 URL 解析的共同影响

#### 9.7.1 CORS 全放开策略：风险可达的基础

**配置位置**：`next.config.ts:63-84`、`docker/proxy.ts:13-18`

```javascript
// next.config.ts 中 /api/:path* 的全局 CORS 配置
const apiHeaders = [
  {
    key: 'Access-Control-Allow-Origin',
    value: '*',  // 允许任意来源
  },
  {
    key: 'Access-Control-Allow-Headers',
    value: '*',  // 允许任意请求头
  },
  {
    key: 'Access-Control-Allow-Methods',
    value: 'GET, DELETE, POST, PUT',  // 允许所有常用方法
  },
];
```

**CORS 配置覆盖范围**：

| 端点 | CORS 配置 | 说明 |
|-----|----------|------|
| `/api/:path*` | `Origin: *` | 所有 API 端点，包括 `/api/send` |
| `/script.js` | `Origin: *` | Tracker 脚本本身也允许跨域加载 |
| 自定义 `COLLECT_API_ENDPOINT` | 继承 apiHeaders | 通过 middleware 重写后应用相同配置 |
| `/q/:slug`、`/p/:slug` | 继承默认 headers | 通过 `GET` 请求调用，不受 CORS 限制 |

**风险可达性分析**：

1. **任何网站都可直接调用**：由于 `Access-Control-Allow-Origin: *`，任何网页（包括恶意网站）都可以直接向 Umami 实例发送数据。

2. **无需凭证即可发送**：Tracker 脚本中 `credentials: 'omit'`（`src/tracker/index.js:38`），且 CORS 配置允许 `*`，因此不需要 Cookie 或认证。

3. **跨站伪造请求完全可行**：攻击者可以在任意页面构造 fetch 请求：
   ```javascript
   fetch('https://umami.example.com/api/send', {
     method: 'POST',
     headers: { 'Content-Type': 'application/json' },
     body: JSON.stringify({
       type: 'event',
       payload: {
         website: 'target-website-uuid',
         hostname: 'fake.com',
         url: '/page',
         referrer: 'https://example.com'
       }
     })
   });
   ```

4. **Collect 端点的特殊风险**：`/q/:slug` 和 `/p/:slug` 通过 `GET` 请求调用，可以通过 `<img>` 标签触发，甚至不需要 JavaScript 执行权限：
   ```html
   <img src="https://umami.example.com/q/tracking-pixel?url=https://fake.com/page" />
   ```

**对比 (collect) 路由与 /api/send 路由**：

| 特性 | `/api/send` | `/q/:slug`、`/p/:slug` |
|-----|------------|---------------------|
| 请求方法 | `POST` | `GET` |
| CORS 限制 | 无（`Origin: *`） | 无（GET 不受 CORS 限制） |
| Payload 可控性 | 完全可控（JSON body） | 部分可控（URL 查询参数） |
| Hostname 可控性 | ✅ 可传入任意值 | ❌ 不可直接传入（使用 request.url） |
| Referrer 可控性 | ✅ 可传入任意值 | ❌ 使用 `request.headers.get('referer')` |
| 触发方式 | 需要 fetch/XHR | `<img>` 标签即可 |

> **关键结论**：CORS 全放开策略使得**任何网站都可以向任何 Umami 实例发送伪造数据**，这是风险能够真正到达服务端的前提条件。

---

#### 9.7.2 urlOrPathParam 校验：看似严格，实则宽松

**校验定义**：`src/lib/schema.ts:92-104`

```typescript
export const urlOrPathParam = z.string().refine(
  value => {
    try {
      new URL(value, 'https://localhost');  // 关键：base 固定为 https://localhost
      return true;
    } catch {
      return false;
    }
  },
  { message: 'Invalid URL.' }
);
```

**校验边界分析**：

使用 `new URL(value, 'https://localhost')` 进行校验是一个**非常宽松**的校验：

| 输入值 | 校验结果 | 实际解析结果 |
|-------|---------|------------|
| `/page` | ✅ PASS | `https://localhost/page` |
| `page` | ✅ PASS | `https://localhost/page` |
| `//evil.com/page` | ✅ PASS | `https://evil.com/page`（协议相对 URL） |
| `https://real.com/page` | ✅ PASS | `https://real.com/page`（绝对 URL，忽略 base） |
| `?query=1` | ✅ PASS | `https://localhost/?query=1` |
| `#hash` | ✅ PASS | `https://localhost/#hash` |
| `../escape` | ✅ PASS | `https://localhost/../escape` → `https://localhost/escape` |
| `javascript:alert(1)` | ✅ PASS | `javascript:alert(1)`（特殊协议） |
| `data:text/html,test` | ✅ PASS | `data:text/html,test`（特殊协议） |
| `https://` | ❌ FAIL | `Invalid URL`（协议后缺少 host） |
| ` http://example.com` | ❌ FAIL | `Invalid URL`（开头有空格） |

**关键风险点**：

1. **协议相对 URL 可绕过**：`//evil.com/page` 会被解析为 `https://evil.com/page`，完全绕过 `localhost` base。

2. **绝对 URL 忽略 base**：`https://real.com/page` 会忽略 `https://localhost` base，直接使用自身的 hostname。

3. **特殊协议可通过**：`javascript:`、`data:` 等协议可通过校验，但入库后 hostname 为空字符串。

4. **无长度限制**：校验没有限制 URL 长度（虽然后续有其他字段限制）。

> **关键结论**：`urlOrPathParam` 校验本质上只验证了"字符串可以被 URL 构造器解析"，但没有限制**最终解析出的 hostname 是什么**。这为伪造 hostname 打开了大门。

---

#### 9.7.3 new URL 入库链路：伪造 hostname 的完整路径

**核心代码**：`src/app/api/send/route.ts:177-215`

```javascript
if (type === COLLECTION_TYPE.event) {
  // 第 178 行：base 完全由 payload.hostname 控制
  const base = hostname ? `https://${hostname}` : 'https://localhost';
  
  // 第 179 行：解析 url（url 已通过 urlOrPathParam 校验）
  const currentUrl = new URL(url, base);
  
  // 第 184 行：从解析结果提取 hostname（去 www 前缀）
  const urlDomain = currentUrl.hostname.replace(/^www./, '');
  
  // ...
  
  if (referrer) {
    // 第 210 行：解析 referrer 时复用同一个 base
    const referrerUrl = new URL(referrer, base);
    referrerDomain = referrerUrl.hostname.replace(/^www./, '');
  }
  
  // 第 234 行：入库时优先使用 payload.hostname
  hostname: hostname || urlDomain,
}
```

**完整风险链路分析**：

```
攻击者控制的输入
    │
    ├─► payload.hostname = 'evil.com'  ───────────────────┐
    │                                                      │
    ├─► payload.url = '/landing-page'  ────────┐          │
    │                                          ▼          ▼
    │                               new URL(url, base) = 'https://evil.com/landing-page'
    │                                          │
    │                                          ├─► urlDomain = 'evil.com'（去 www）
    │                                          │
    ├─► payload.referrer = 'https://example.com/home'  ───┐
    │                                          ▼          │
    │                               new URL(referrer, base) = 'https://example.com/home'
    │                                          │
    │                                          ├─► referrerDomain = 'example.com'（去 www）
    │                                          │
    └──────────────────────────────────────────┼──────────┘
                                               ▼
                          入库：hostname = 'evil.com'（优先使用 payload 值）
                               referrerDomain = 'example.com'
                                               │
                                               ▼
                    查询过滤：referrer_domain ('example.com') != hostname ('evil.com')
                                               │
                                               ▼
                                 ✅ 被统计为外部来源（伪造成功）
```

**四种典型攻击场景的完整路径**：

| 攻击目标 | payload.hostname | payload.url | payload.referrer | 最终效果 |
|---------|-----------------|------------|------------------|---------|
| 虚高外部来源 | `fake-external.com` | `/page` | `https://example.com/home` | 内部跳转变外部来源 |
| 掩盖真实来源 | `google.com` | `/page` | `https://google.com/search` | 真实外部来源被过滤 |
| 统计碎片化 | 随机字符串 | `/page` | 真实 referrer | hostname 维度完全不可用 |
| 协议相对 URL 攻击 | `undefined`（不传） | `//evil.com/page` | `https://example.com` | hostname 被篡改为 `evil.com` |

**场景 4 特别说明**：即使不传入 `payload.hostname`，也可以通过 `payload.url` 伪造 hostname：

```javascript
// payload: { hostname: undefined, url: '//evil.com/page', referrer: '...' }
const base = 'https://localhost';
const currentUrl = new URL('//evil.com/page', 'https://localhost');
// 解析结果: hostname = 'evil.com'
const urlDomain = 'evil.com';
// 入库: hostname = 'evil.com'（因为 hostname 为 undefined，使用 urlDomain）
```

> **关键结论**：`new URL(url, base)` 的解析行为，加上 `payload.hostname` 和 `payload.url` 的双重可控性，形成了完整的伪造路径。攻击者可以通过**两种独立方式**控制最终入库的 hostname。

---

#### 9.7.4 三者共同作用下的统计口径偏差矩阵

| CORS 放开 | urlOrPathParam 宽松 | new URL 可控 | 最终风险 | 统计影响 |
|----------|-------------------|-------------|---------|---------|
| ❌ 限制 | ✅ 严格 | ✅ 可控 | ⭐ | 仅本站可发送，风险可控 |
| ✅ 放开 | ❌ 严格 | ✅ 可控 | ⭐⭐ | 可跨站发送，但 URL 格式受限 |
| ✅ 放开 | ✅ 宽松 | ❌ 不可控 | ⭐⭐⭐ | 可跨站发送，但 hostname 由 URL 解析决定 |
| ✅ 放开 | ✅ 宽松 | ✅ 可控 | ⭐⭐⭐⭐⭐ | **完全失控**，任意伪造 |

**真实的风险可达路径（从攻击者视角）**：

```
1. 目标识别：获取目标网站的 website-id（可从公开页面的 script 标签获取）
2. 构造 payload：
   {
     type: 'event',
     payload: {
       website: 'target-uuid',
       hostname: 'fake.com',     // 伪造 hostname
       url: '/page',             // 相对路径，使用伪造的 base
       referrer: 'https://google.com/search'  // 真实或伪造的来源
     }
   }
3. 发送请求（从任意网站）：
   fetch('https://umami.example.com/api/send', {
     method: 'POST',
     headers: { 'Content-Type': 'application/json' },
     body: JSON.stringify(payload)
   })
4. 结果：伪造数据成功入库，统计口径被污染
```

**统计偏差的具体表现**：

| 统计维度 | 偏差表现 | 对业务决策的影响 |
|---------|---------|----------------|
| Hostname 统计 | 出现大量伪造域名 | 无法按域名聚合分析 |
| Referrer Domain | 虚高或虚低外部来源 | 渠道投放 ROI 计算错误 |
| Direct 流量 | 被掩盖或虚高 | 品牌流量评估失真 |
| UTM 参数 | 与伪造 hostname 混淆 | 营销活动归因错误 |
| 会话追踪 | hostname 频繁变化导致会话断裂 | 用户行为路径分析失效 |

---

### 9.8 缓解建议

> **说明**：以下建议仅为基于代码分析的技术探讨，不构成修改建议。

```typescript
// 假设要在服务端添加 hostname 校验的伪代码
// src/app/api/send/route.ts

// 1. 获取 website 配置
const website = await fetchWebsite(websiteId);

// 2. 从 payload 或解析获取 hostname
const effectiveHostname = hostname || urlDomain;

// 3. 如果 website 配置了 domain，则校验
if (website?.domain) {
  const allowedDomains = website.domain.split(',').map(d => d.trim().toLowerCase());
  const normalizedHostname = effectiveHostname.replace(/^www\./, '').toLowerCase();
  
  // 支持 Website.domain 中配置多个域名（当前设计是单值，可扩展）
  if (!allowedDomains.includes(normalizedHostname)) {
    // 拒绝或记录警告
  }
}

// 4. 统一归一化规则
const normalizedHostname = effectiveHostname.replace(/^www\./, '').toLowerCase();
```

**针对 CORS、参数校验和 URL 解析的补充建议**：

```typescript
// 1. 限制 CORS 来源（替换 Origin: *）
// next.config.ts
const apiHeaders = [
  {
    key: 'Access-Control-Allow-Origin',
    value: process.env.ALLOWED_ORIGINS || '*',  // 可配置白名单
  },
];

// 2. 强化 urlOrPathParam 校验，禁止协议相对 URL 和绝对 URL
// src/lib/schema.ts
export const urlOrPathParam = z.string().refine(
  value => {
    try {
      // 禁止以 // 开头的协议相对 URL
      if (value.startsWith('//')) return false;
      
      const u = new URL(value, 'https://localhost');
      // 禁止解析出的 hostname 不是 localhost
      if (u.hostname !== 'localhost') return false;
      // 禁止特殊协议
      if (u.protocol !== 'https:' && u.protocol !== 'http:') return false;
      
      return true;
    } catch {
      return false;
    }
  },
  { message: 'Invalid URL.' }
);

// 3. 在 new URL 解析前验证 payload.hostname
// src/app/api/send/route.ts
if (type === COLLECTION_TYPE.event) {
  // 校验 payload.hostname 是否在允许的域名列表中
  if (hostname && website?.domain) {
    const allowedDomains = website.domain.split(',').map(d => 
      d.trim().toLowerCase().replace(/^www\./, '')
    );
    const normalizedHostname = hostname.toLowerCase().replace(/^www\./, '');
    
    if (!allowedDomains.includes(normalizedHostname)) {
      return badRequest({ message: 'Invalid hostname.' });
    }
  }
  
  // 强制使用验证后的域名作为 base
  const safeHostname = hostname 
    ? hostname.toLowerCase().replace(/^www\./, '')
    : website?.domain?.split(',')[0]?.trim().toLowerCase().replace(/^www\./, '');
  
  const base = safeHostname ? `https://${safeHostname}` : 'https://localhost';
}
```

---

## 十、关键设计特点与安全边界

### 10.1 设计特点

1. **前端为主的校验机制**：域名白名单、DNT 尊重等主要在前端完成，减少服务端压力

2. **灵活的三层端点配置**：支持构建时、运行时脚本标签、运行时环境变量三层配置，适应不同部署场景

3. **渐进式忽略**：从前端到服务端多层过滤，确保数据质量

4. **无状态设计**：通过 JWT token（`x-umami-cache` 头）缓存会话信息，服务端无状态

5. **隐私优先**：默认不依赖 Cookie，使用 IP + User-Agent + Salt 生成会话 ID

### 10.2 安全边界与注意事项

> **⚠️ 重要安全提示**：
> 
> 1. **域名校验可绕过**：前端域名校验可通过浏览器开发者工具绕过，对于需要严格域名绑定的场景，需要在服务端自行添加校验逻辑。
> 
> 2. **Website.domain 仅供展示**：数据库中的 domain 字段不参与数据收集中的任何校验，不要误以为它能防止数据伪造。
> 
> 3. **data-domains 需手动配置**：管理后台生成的跟踪代码不包含 data-domains，用户必须手动添加才能启用域名白名单。
> 
> 4. **IP 检测依赖请求头**：IP 提取依赖 `X-Forwarded-For` 等请求头，在反向代理配置不当时可能不准确。
> 
> 5. **机器人检测可被欺骗**：`isbot()` 依赖 User-Agent 头，可被伪造。
> 
> 6. **CORS 全放开**：`/api/:path*` 配置了 `Access-Control-Allow-Origin: *`，任何网站都可以向 Umami 实例发送数据。
> 
> 7. **urlOrPathParam 校验宽松**：仅校验字符串可被 URL 解析，不限制最终 hostname，允许协议相对 URL 和绝对 URL。
> 
> 8. **Hostname 完全可控**：`payload.hostname` 可通过 `track()`、`beforeSend` 或 `payload.url`（协议相对 URL）任意修改，服务端无校验，Referrer Domain 统计存在严重伪造风险（详见第九章）。
> 
> 9. **归一化规则不一致**：三处 hostname 处理的归一化规则不一致，即使正常使用也可能出现统计偏差。
> 
> 10. **new URL 解析可被滥用**：`base` 完全由 `payload.hostname` 控制，结合 `urlOrPathParam` 的宽松校验，形成完整的伪造路径。

---

## 十一、代码溯源索引

| 功能 | 文件位置 |
|-----|--------|
| Tracker 构建配置 | `rollup.tracker.config.js:1-20` |
| Tracker 脚本源码 | `src/tracker/index.js:1-411` |
| 管理后台跟踪代码生成 | `src/app/(main)/websites/[websiteId]/settings/WebsiteTrackingCode.tsx:1-40` |
| 主收集端点 | `src/app/api/send/route.ts:66-322` |
| IP 检测与忽略 | `src/lib/detect.ts:140-170` |
| 机器人检测 | `src/app/api/send/route.ts:131-133` |
| 域名白名单校验 | `src/tracker/index.js:156-161` |
| 网站数据加载 | `src/lib/load.ts:6-20` |
| 数据库模型 | `prisma/schema.prisma:66-96` |
| 运行时更新脚本 | `scripts/update-tracker.js:1-16` |
| DOMAIN 正则定义 | `src/lib/constants.ts:249-250` |
| 查询时引用域名过滤 | `src/queries/sql/getValues.ts:26` |
| **Hostname 三阶段处理** | |
| Tracker payload.hostname | `src/tracker/index.js:14`、`69`、`209-210` |
| /api/send hostname 入库 | `src/app/api/send/route.ts:32`、`177-184`、`234` |
| 查询时 referrer 过滤 | `src/queries/sql/getValues.ts:26-27`、`83` |
| track() 函数重载定义 | `src/tracker/index.d.ts:128-147` |
| beforeSend 回调 | `src/tracker/index.js:166-170` |
| **CORS 与参数校验** | |
| /api/* CORS 配置 | `next.config.ts:63-84`、`113-123` |
| Docker 代理 CORS 配置 | `docker/proxy.ts:13-18` |
| Middleware 重写逻辑 | `docker/proxy.ts:26-37` |
| urlOrPathParam 校验 | `src/lib/schema.ts:92-104` |
| /api/send new URL 解析 | `src/app/api/send/route.ts:177-215` |
| (collect) /q/:slug 端点 | `src/app/(collect)/q/[slug]/route.ts:10-60` |
| (collect) /p/:slug 端点 | `src/app/(collect)/p/[slug]/route.ts:12-70` |
