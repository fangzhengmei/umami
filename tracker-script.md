# Umami 埋点脚本构建与配置参数对应关系

## 一、整体架构概览

Umami 的埋点系统采用**构建时注入 + 运行时配置**相结合的方式，将服务端配置参数嵌入到分发的 `script.js` 中。关键配置参数通过三层机制传递：

1. **构建时**：Rollup 插件静态替换
2. **运行时（脚本标签**：HTML 标签 `data-*` 属性
3. **运行时（服务端）**：环境变量配置

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
    terser({ compress: { evaluate: false }),
  ],
};
```

**构建命令**：`npm run build-tracker` → `rollup -c rollup.tracker.config.js`

### 2.2 构建时参数注入

使用 `@rollup/plugin-replace` 在构建阶段将环境变量值**静态替换**到脚本中：

| 占位符 | 环境变量 | 默认值 | 作用 |
|--------|---------|--------|------|
| `__COLLECT_API_HOST__` | `COLLECT_API_HOST` | `''` | 收集服务端主机地址 |
| `__COLLECT_API_ENDPOINT__` | `COLLECT_API_ENDPOINT` | `/api/send` | 数据收集接口路径 |

### 2.3 运行时更新脚本

**更新脚本**：`scripts/update-tracker.js:1-16`

在服务启动时，`update-tracker` 脚本会再次读取 `COLLECT_API_ENDPOINT` 环境变量，对已构建的 `public/script.js` 进行二次替换：

```javascript
if (endPoint) {
  const tracker = fs.readFileSync(file);
  fs.writeFileSync(file, tracker.toString().replace(/\/api\/send/g, endPoint);
}
```

> **设计意图**：支持在不重新构建的情况下，通过环境变量修改收集端点。

---

## 三、同源域名（Domains）配置

### 3.1 数据库存储

**数据模型**：`prisma/schema.prisma:69`

```prisma
model Website {
  id        String    @id() @map("website_id") @db.Uuid
  name      String    @db.VarChar(100)
  domain    String?   @db.VarChar(500)  // 同源域名配置
  // ...
}
```

### 3.2 前端脚本嵌入机制

**脚本标签属性**：`src/tracker/index.js:37`

```javascript
const domain = config('domains') || '';
const domains = domain.split(',').map(n => n.trim());
```

通过脚本标签的 `data-domains` 属性传入，多个域名用**逗号分隔**：

```html
<script async defer 
  src="http://umami.example.com/script.js"
  data-website-id="xxx"
  data-domains="example.com,sub.example.com"></script>
```

### 3.3 前端域名校验

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
- 如果设置了 `data-domains`，只有当前页面的 `hostname` 必须在域名列表中才会发送数据
- 如果未设置，则不进行域名过滤
- 域名匹配是**精确匹配**，不支持通配符

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
1. 脚本标签 `data-host-url` 属性
2. 构建时注入的 `__COLLECT_API_HOST__`
3. 从脚本 `src` URL 自动推断（脚本所在域名）

**PATH 来源**：
- 构建时注入的 `__COLLECT_API_ENDPOINT__`（默认 `/api/send`）

### 4.2 服务端接收端点

**主收集端点**：`src/app/api/send/route.ts:66-322`

支持三种收集类型：
- `event`：页面浏览和自定义事件
- `identify`：用户识别
- `performance`：性能指标

**短链追踪端点**：`src/app/(collect)/q/[slug]/route.ts
**像素追踪端点**：`src/app/(collect)/p/[slug]/route.ts`

### 4.3 数据发送逻辑

**发送函数**：`src/tracker/index.js:163-195`

```javascript
const send = async (payload, type = 'event') => {
  if (trackingDisabled()) return;
  // ... beforeSend 回调处理
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

## 五、域名校验（Domain Validation）

### 5.1 前端校验（主要防线）

如 3.3 节所述，前端在 `trackingDisabled()` 函数中进行域名白名单校验。

### 5.2 服务端处理

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

> **重要发现**：**服务端没有强制校验 `hostname` 与 `website.domain` 的匹配关系。域名校验主要依赖前端实现。

### 5.3 引用域名过滤

**查询时过滤**：`src/lib/clickhouse.ts:125-127

在数据查询时，会过滤掉 `referrer_domain == hostname` 的流量（即网站自身的引用。

---

## 六、忽略规则（Ignore Rules）

### 6.1 前端忽略规则

| 规则 | 实现位置 | 说明 |
|-----|---------|------|
| 本地禁用标记 | `src/tracker/index.js:159` | `localStorage.getItem('umami.disabled')` |
| 域名白名单 | `src/tracker/index.js:160` | `domain && !domains.includes(hostname)` |
| DNT 尊重 | `src/tracker/index.js:161` | `dnt && hasDoNotTrack()` |
| 自动追踪禁用 | `src/tracker/index.js:33` | `data-auto-track="false"` |

### 6.2 服务端忽略规则

#### 6.2.1 IP 忽略

**实现**：`src/lib/detect.ts:140-170`

```javascript
export function hasBlockedIp(clientIp: string) {
  const ignoreIps = process.env.IGNORE_IP;
  if (ignoreIps) {
    const ips = ignoreIps.split(',').map(n => n.trim());
    return ips.find(ip => {
      if (ip === clientIp) return true;
      // 支持 CIDR 格式
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

**配置方式**：环境变量 `IGNORE_IP=192.168.1.1,10.0.0.0/8

**调用位置**：`src/app/api/send/route.ts:136-138

```javascript
if (hasBlockedIp(ip)) {
  return forbidden();
}
```

#### 6.2.2 机器人检测

**实现**：`src/app/api/send/route.ts:131-133`

```javascript
if (!process.env.DISABLE_BOT_CHECK && isbot(userAgent)) {
  return json({ beep: 'boop' });
}
```

使用 `isbot` 库检测爬虫 User-Agent，可通过 `DISABLE_BOT_CHECK=true` 禁用。

#### 6.2.3 本地 IP 忽略

**实现**：`src/lib/detect.ts:81`

```javascript
if (!ip || (await isLocalhost(ip))) {
  return null;
}
```

本地 IP（localhost、内网 IP 等）不收集地理位置信息。

#### 6.2.4 其他忽略配置

| 环境变量 | 作用 |
|---------|------|
| `DISABLE_BOT_CHECK` | 禁用机器人检测 |
| `SKIP_LOCATION_HEADERS` | 跳过从 HTTP 头解析地理位置 |
| `IGNORE_IP` | 忽略指定 IP 地址（支持 CIDR） |

---

## 七、脚本完整参数列表

### 7.1 脚本标签 `data-*` 属性

| 属性 | 变量 | 类型 | 默认值 | 说明 |
|-----|------|------|--------|------|
| `data-website-id` | `website` | string | - | **必填**，网站 UUID |
| `data-host-url` | `hostUrl` | string | - | 收集服务端 URL |
| `data-domains` | `domain` | string | - | 允许的域名列表，逗号分隔 |
| `data-auto-track` | `autoTrack` | boolean | `true` | 是否自动追踪页面浏览 |
| `data-do-not-track` | `dnt` | boolean | `false` | 是否遵守浏览器 DNT 设置 |
| `data-exclude-search` | `excludeSearch` | boolean | `false` | 是否排除 URL 搜索参数 |
| `data-exclude-hash` | `excludeHash` | boolean | `false` | 是否排除 URL hash |
| `data-fetch-credentials` | `credentials` | string | `'omit'` | fetch credentials 选项 |
| `data-before-send` | `beforeSend` | string | - | 发送前回调函数名 |
| `data-tag` | `tag` | string | - | 事件标签 |
| `data-performance` | `perf` | boolean | `false` | 是否启用性能监控 |

**示例：

```html
<script async defer
  src="https://umami.example.com/script.js"
  data-website-id="b59e9c65-ae32-47f1-8400-119fcf4861c4"
  data-domains="example.com,www.example.com"
  data-exclude-search="true"
  data-do-not-track="true"></script>
```

### 7.2 环境变量配置

| 变量 | 作用 | 生效时机 | 说明 |
|-----|------|---------|------|
| `COLLECT_API_HOST` | 收集服务端 HOST | 构建时 + 运行时 | 替换 `__COLLECT_API_HOST__` |
| `COLLECT_API_ENDPOINT` | 收集端点路径 | 构建时 + 运行时 | 替换 `__COLLECT_API_ENDPOINT__` |
| `IGNORE_IP` | 忽略 IP 列表 | 运行时（服务端） | 逗号分隔，支持 CIDR |
| `DISABLE_BOT_CHECK` | 禁用机器人检测 | 运行时（服务端） | `true` 时禁用 |
| `SKIP_LOCATION_HEADERS` | 跳过地理位置头 | 运行时（服务端） | `true` 时跳过 |

---

## 八、参数嵌入机制总结

| 配置项 | 嵌入时机 | 嵌入方式 | 可覆盖方式 |
|--------|---------|---------|-----------|
| **收集端点 HOST | 构建时 + 运行时 | Rollup replace 替换 `__COLLECT_API_HOST__` | `data-host-url` 属性优先级最高 |
| **收集端点 PATH | 构建时 + 运行时 | Rollup replace 替换 `__COLLECT_API_ENDPOINT__` | `update-tracker.js` 二次替换 |
| **同源域名** | 运行时（前端） | 读取 `data-domains` 属性 | 脚本标签属性动态配置 |
| **域名校验** | 运行时（前端） | `domains.includes(hostname)` 检查 | 服务端无强制校验，依赖前端 |
| **IP 忽略** | 运行时（服务端） | 读取 `IGNORE_IP` 环境变量 | 服务端环境变量配置 |
| **机器人忽略** | 运行时（服务端） | `isbot()` 检测 + `DISABLE_BOT_CHECK` | 环境变量开关 |

---

## 九、关键设计特点

1. **前端为主的校验机制**：域名白名单、DNT 尊重等主要在前端完成，减少服务端压力

2. **灵活的端点配置**：支持构建时、运行时、脚本标签三层配置，适应不同部署场景

3. **渐进式忽略**：从前端到服务端多层过滤，确保数据质量

4. **无状态设计**：通过 JWT token（`x-umami-cache` 头）缓存会话信息，服务端无状态

5. **隐私优先**：默认不依赖 Cookie，使用 IP + User-Agent + Salt 生成会话 ID

---

## 十、代码溯源索引

| 功能 | 文件位置 |
|-----|--------|
| 构建配置 | `rollup.tracker.config.js` |
| 追踪脚本源码 | `src/tracker/index.js` |
| 收集端点 | `src/app/api/send/route.ts` |
| IP 检测与忽略 | `src/lib/detect.ts` |
| 网站数据加载 | `src/lib/load.ts` |
| 数据库模型 | `prisma/schema.prisma` |
| 运行时更新脚本 | `scripts/update-tracker.js` |
