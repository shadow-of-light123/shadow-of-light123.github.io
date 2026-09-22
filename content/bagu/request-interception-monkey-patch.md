---
title: '全站请求拦截：Monkey Patch 的实现与工程实践'
date: 2026-09-19
draft: false
categories: ['前端']
tags: ['浏览器原理', '网络', '工程实践']
---

面试里有一类题，表面在问"你会不会改 API"，实际在考**工程判断力**：

> 如果让你做 fetch 的改写，你主要会做哪些事情？
>
> 如何拦截页面上所有请求？多个包都改写 XHR 会发生什么？

这类题的陷阱在于，很多人一上来就罗列"我采集 url、method、status、耗时"——那是功能清单，不是答案。面试官真正想听的是三层：**会不会把业务搞挂、有没有工程意识、有没有取舍能力**。

这篇文章把这道题从场景、术语、实现到数据落地完整拆一遍。

---

## 一、为什么要"拦截所有请求"

先别想技术手段，看三个具体的人遇到了什么问题。

### 场景一：给后台系统加监控

产品经理问："昨天下午有用户反馈系统很卡，能查到是哪个接口慢吗？"

你想让每个用户的每个请求都被自动记录。最直觉的做法是手动埋点：

```js
// 原来
axios.get('/api/user').then(...)

// 改成
const t = Date.now()
axios.get('/api/user').then(res => track('end', '/api/user', Date.now() - t, res.status))
```

但这走不通，因为项目里：

1. 有 80 多个发请求的地方，一个个改不现实
2. 三年前的老模块还在用 `new XMLHttpRequest()`，根本不走 axios
3. 集成的在线客服 SDK、地图 SDK，代码是压缩过的，**你改不了**
4. 埋点用的是 `<img src="/collect?id=123">`，压根没有 JS 调用

于是只剩一个办法：**不管上面的代码怎么写，在请求真正发出的"总闸门"上装个计数器。**

`XMLHttpRequest.prototype.send` 和 `window.fetch` 就是全站请求的总闸门——只要请求要发出去，最终都会走到这两个函数。把它们换成自己的版本，里面记一笔再调用原来的，就等于给所有请求装上了监控。

### 场景二：安全管控 / 数据防泄漏

公司内部系统注入了一个安全 SDK。某天员工把一批客户手机号粘进搜索框点了搜索：

```
GET /api/search?keyword=13800138000,13900139000,...
```

这是典型的数据外泄。安全 SDK 要做的事：在请求真正发出去之前，把 URL 和请求体读出来跑一遍敏感规则，命中就**阻断 + 上报**。

"在请求真正发出去之前"这个时机，只有改写 `send` 才拿得到。

### 场景三：基建统一注入

公司有几百个前端项目。链路追踪要求每个请求都带 `x-trace-id`，灰度发布要带灰度标记，多租户 SaaS 要带 `tenant-id`。

让每个业务团队自己加，一定有人漏、有人格式写错。基建团队的做法是：写一个 SDK，业务方入口引入一行，之后所有出站请求自动补齐。这个"自动"就是靠改写实现的。

### 三个场景的共同点

**他们都无法要求别人配合。**

- 做监控的，没法要求全公司 80 处请求都手动埋点
- 做安全的，没法要求每个业务方都记得校验参数
- 做基建的，没法保证每个团队都规范加 header

所以只能从**别人控制不到的地方**下手——浏览器提供的平台 API 这一层。这才是 Monkey Patch 存在的根本原因。

### 为什么不能用 axios 拦截器

请求出站的路径大致分三层：

```
① 业务代码层      axios 封装 · fetch 直调 · 老代码原生 XHR · 第三方 SDK 内部
       ↓
② 平台出口层      XMLHttpRequest · fetch · sendBeacon · WebSocket
       ↓
③ 网络栈层        浏览器网络 / Service Worker
```

axios 拦截器装在 ① 里"axios 封装"这一条支路上，只有走 axios 的请求会经过它：

| 请求从哪来           | axios 拦截器 | Monkey Patch（②层） | Service Worker（③层） |
| -------------------- | ------------ | ------------------- | --------------------- |
| 业务代码走 axios     | ✅           | ✅                  | ✅                    |
| 业务代码用 fetch     | ❌           | ✅                  | ✅                    |
| 老代码用原生 XHR     | ❌           | ✅                  | ✅                    |
| 第三方 SDK 内部发的  | ❌           | ✅                  | ✅                    |
| `navigator.sendBeacon` | ❌         | ✅                  | ✅                    |
| `<img>` 打点         | ❌           | ❌                  | ✅                    |
| iframe 里的请求      | ❌           | ❌                  | ❌                    |

**越往下走，覆盖面越大，代价也越大。**

- **axios 拦截器**：只覆盖自己项目的一条支路，最规范可控，覆盖率最低
- **Monkey Patch**：覆盖大部分走 JS 的请求，但要写"脏"代码，容易踩坑
- **Service Worker**：位置最低，连 `<img>` 都能拦，但依赖 HTTPS + 注册激活，且首次加载时它还没接管

所以「拦截页面上所有请求」**本质是一个覆盖率需求，不是一个功能需求**。

---

## 二、术语澄清：这不叫"覆写"

很多人（包括我自己一开始）会说"覆写 XHR"、"重写 fetch"。严格说不准确。

| 术语               | 英文        | 语义                                   | 适用场景                       |
| ------------------ | ----------- | -------------------------------------- | ------------------------------ |
| 覆写 / 重写        | Override    | 子类重新定义父类方法，**需要继承关系** | 面向对象，`class Son extends Father` |
| 覆盖               | Overwrite   | 用新内容盖掉旧内容                     | 文件、变量、内存               |
| **改写 / 替换 / 劫持** | Monkey patch | 直接替换一个已存在的函数引用        | **改 `XMLHttpRequest.prototype.send`** |
| 复写               | —           | 复写纸式的重复抄写，**不是计算机术语** | 不建议用于编程语境             |

用代码看区别最清楚：

```js
// ① 覆写（Override）：需要继承关系
class MyXHR extends XMLHttpRequest {
  send(body) {
    track('send')
    return super.send(body)   // 调父类实现
  }
}
// 只有 new MyXHR() 的实例被改，别人的 new XMLHttpRequest() 完全不受影响


// ② Monkey Patch：没有继承关系，直接替换原型上的属性
XMLHttpRequest.prototype.send = function (body) { /* ... */ }
window.fetch = myFetch
// 全页面所有代码都被改了，包括第三方 SDK
```

覆写的必要条件是有继承关系（子类 extends 父类 + 同名方法 + 多态分发）。而 `XMLHttpRequest.prototype.send = fn` 从语言机制看，就是一次普通的属性赋值——把一个已有的函数引用换成新的，没有父子类，谈不上覆写。

**两者的真正分界线：**

|            | 覆写 Override            | Monkey Patch               |
| ---------- | ------------------------ | -------------------------- |
| 前提       | 存在继承关系             | 没有继承关系，直接改属性   |
| 合法与否   | 语言设计内的正常用法     | 越界改了不属于自己的代码   |
| 影响范围   | 只影响你自己的子类实例   | 影响整个运行环境           |
| 语感       | 正规、可维护             | hack、临时、不优雅         |

规则型答案：描述这个动作，说**改写** `window.fetch`、**劫持** XHR、**Monkey Patch 了 `XMLHttpRequest.prototype.send`**。面试时说"覆写"懂的人也能明白，但严格说不准确，容易被追问一句"你这是继承吗？"。

> 顺带区分两个容易混的英文词：**Override**（覆写方法）和 **Overwrite**（覆盖数据）是完全不同的词，中文翻译经常被混用，这是"覆写/覆盖"容易混淆的根源。

### 一个有用的推论

看上面 ① 那段代码就明白：**继承 + 覆写这套正规做法根本解决不了"拦截所有请求"**——因为别人的代码用的是 `new XMLHttpRequest()`，不是你的 `MyXHR`。

这就是为什么监控/安全 SDK 必须放弃正规手段，改用 Monkey Patch 去改全局原型：**需求本身就是"越界"的，所以手段也只能越界**。代价就是后面讲的那一堆坑。

---

## 三、改写哪些方法

分两层：网络请求类是必改的，周边能力类按场景加。

### 网络请求类（几乎所有 SDK 都会改）

| 目标 | 改写的方法                              | 能拿到什么                             |
| ---- | --------------------------------------- | -------------------------------------- |
| XHR  | `XMLHttpRequest.prototype.open`         | method、url                            |
| XHR  | `XMLHttpRequest.prototype.send`         | 请求体、发起时刻（计时起点）           |
| XHR  | `XMLHttpRequest.prototype.setRequestHeader` | 全部请求头（也是**注入** header 的位置） |
| XHR  | `XMLHttpRequest.prototype.abort`        | 用户取消行为                           |
| fetch | `window.fetch`                         | 一个函数收口，method/url/body/header 全在里面 |

安全管控场景会多改两个（因为要审**响应**而不只是请求）：

```js
XMLHttpRequest.prototype.getAllResponseHeaders
XMLHttpRequest.prototype.getResponseHeader
```

### 周边能力类（按场景选）

| 场景         | 改写目标                                                   | 用途                                     |
| ------------ | ---------------------------------------------------------- | ---------------------------------------- |
| 卸载上报     | `navigator.sendBeacon`                                     | 页面关闭前的数据回传，漏了会丢埋点       |
| 实时通信     | `WebSocket` 构造函数 + `WebSocket.prototype.send`          | 监控 WS 连接、收发消息                   |
| 实时通信     | `EventSource` 构造函数                                     | SSE 连接监控                             |
| 表单提交     | `HTMLFormElement.prototype.submit` / `requestSubmit`       | 老式表单跳转也要覆盖                     |
| SPA 埋点     | `History.prototype.pushState` / `replaceState`             | 路由切换算 PV（不是网络，但总一起改）    |
| 错误采集     | `console.error` / `console.warn`                           | 捕获业务主动打的错误日志                 |
| 数据外泄防护 | `document.cookie` 的 setter、`localStorage.setItem`        | 防止敏感数据写本地                       |
| 性能监控     | `setTimeout` / `setInterval` / `requestAnimationFrame`     | 长任务、首屏、卡顿分析                   |
| 行为录制     | `window.addEventListener` 本身                              | 录制类 SDK 会包装全局事件绑定            |

### 选择原则：找"收口点"

决定改哪个方法的标准不是"这个方法重不重要"，而是**它是不是所有同类操作的必经之处**：

- `window.fetch` —— 一个函数就是全部 fetch 的收口，改一处全覆盖 ✅
- XHR —— 有两个收口（`open` / `send`），而且数据分散在两个方法里，所以必须都改
- `document.cookie` —— 是个 accessor，改 setter 就控住了所有 cookie 写入 ✅

**收口点越少、越底层，改一处覆盖面越大，但也越危险。**所以实践中按需选择，不会无脑全改——改得越多，踩坑和污染业务的风险越大。

### 改写 vs 监听

有些东西**不需要改写**，订阅就行，成本低得多：

| 目的             | 改写（侵入）           | 监听（推荐）                                          |
| ---------------- | ---------------------- | ----------------------------------------------------- |
| JS 运行时错误    | patch `console`        | `window.addEventListener('error')`                    |
| Promise 未处理拒绝 | —                    | `window.addEventListener('unhandledrejection')`       |
| 资源加载耗时     | patch `createElement`  | `PerformanceObserver`                                 |
| 路由变化         | patch `pushState`      | `popstate` 事件（注意：`pushState` 不会触发 `popstate`，所以这个还得改） |

原则：**能用标准监听 API 拿到的，就别去改原型。**

---

## 四、XHR 的实现

XHR 的方法挂在 `XMLHttpRequest.prototype` 上（不是实例属性），而且**一次请求的信息分散在两个方法里**：`open` 里有 method 和 url，`send` 里有 body。

```js
;(function patchXHR() {
  const proto = XMLHttpRequest.prototype
  if (proto.__xhrPatched) return          // 幂等：避免被引入两次时重复 patch
  proto.__xhrPatched = true

  const rawOpen = proto.open              // ← patch 那一刻读取，就是上一个包的包装
  const rawSend = proto.send
  const META = Symbol('meta')             // 用 Symbol 避免和业务自定义属性撞名

  proto.open = function (method, url) {
    // 状态挂"实例"上，不能用闭包变量
    this[META] = { method: String(method).toUpperCase(), url: String(url) }
    return rawOpen.apply(this, arguments)  // open 有 5 个参数，用 arguments 全透传
  }

  proto.send = function (body) {
    const meta = this[META] || { method: 'GET', url: '' }
    const start = Date.now()

    const done = () => {
      this.removeEventListener('loadend', done)
      report({
        url: meta.url,
        method: meta.method,
        status: this.status,
        duration: Date.now() - start,
        // responseType 不是 text 时读 responseText 会抛 InvalidStateError
        response: (this.responseType === '' || this.responseType === 'text')
          ? this.responseText
          : undefined,
      })
    }
    this.addEventListener('loadend', done)  // 用事件监听，不覆盖 onreadystatechange

    return rawSend.apply(this, arguments)
  }
})()

// patch 自身的上报出错，绝不能影响业务请求
function report(data) {
  try { /* 真正的上报逻辑 */ } catch (e) {}
}
```

**XHR 的五个坑：**

| 坑                             | 说明                                                                 |
| ------------------------------ | -------------------------------------------------------------------- |
| 状态不能用闭包变量             | 页面同时跑几十个 XHR 实例，闭包变量会串数据；必须挂 `this`           |
| `open` 有 5 个参数             | `open(method, url, async, user, password)`，只接 3 个会丢参数        |
| 不能覆盖 `onreadystatechange`  | 业务代码可能自己用了这个属性，覆盖会吃掉它的回调                     |
| `responseType` 决定能否读文本  | 设为 `blob`/`arraybuffer` 时读 `responseText` 会直接抛异常；大 blob 读出来是内存灾难，建议不读 |
| 上报函数要各自 try/catch       | 监控 SDK 自身挂了导致业务发不出请求，是最严重的生产事故              |

> 补充：如果业务直接调用 `send()` 而没先 `open()`（不合法但可能发生），`META` 会是 `undefined`，所以要给默认值兜底。

---

## 五、fetch 的实现

fetch 比 XHR 简单，因为入参和出参都集中在一个函数调用里，但代价是陷阱更隐蔽。

### 5.1 透明代理骨架

```js
;(function interceptFetch() {
  const rawFetch = window.fetch
  if (!rawFetch || rawFetch.__intercepted) return   // 幂等，防重复引入

  async function patchedFetch(input, init) {
    const info = describe(input, init)              // 采集入参（5.2）
    const start = Date.now()

    let res
    try {
      res = await rawFetch.apply(this, arguments)   // ① 透传 this + 全部参数
    } catch (err) {
      info.aborted = !!(err && err.name === 'AbortError')
      info.error = String(err)
      info.duration = Date.now() - start
      collect(info)
      throw err                                     // ② 必须重新抛
    }

    info.status = res.status
    info.duration = Date.now() - start
    collect(info)

    captureBody(res, info)                          // 读响应体（5.3）
    return res                                      // ③ 原样返回
  }

  patchedFetch.__intercepted = true
  window.fetch = patchedFetch
})()
```

| 标注 | 为什么                                                                                     |
| ---- | ------------------------------------------------------------------------------------------ |
| ①    | 有代码会写 `fetch.call(x, ...)`；用 `(input, init)` 重传会丢掉多余参数                      |
| ②    | 吞掉异常，业务侧的 `try/catch`、`.catch()` 会**永久挂起**，页面直接白屏                      |
| ③    | 不能 `new Response(res.body, res)`，会丢 `statusText`、`url`、类型信息，还会破坏流式响应    |

另外 `AbortError` 要单独标记：项目里用 `AbortController` 中止流式输出时，每次中止都会走到 `catch`。不区分就当成网络错误上报，监控平台会出现大量假错误。

### 5.2 入参解析：三种形态

```js
function describe(input, init) {
  let url, method

  if (typeof input === 'string') {
    url = input
    method = (init && init.method) || 'GET'
  } else if (input instanceof URL) {
    url = input.href
    method = (init && init.method) || 'GET'
  } else if (input instanceof Request) {
    url = input.url                      // Request 对象自带 method
    method = input.method
  } else {
    url = String(input)
    method = 'GET'
  }

  return {
    url: normalizeUrl(url),              // 归一化后的，用于聚合统计
    rawUrl: url,                         // 原始 URL，用于排查具体问题
    method: method.toUpperCase(),
    body: safeStringify(init && init.body),
  }
}
```

**这是 fetch 比 XHR 麻烦的地方**：第一个参数有三种可能。如果只判断 `typeof input === 'string'`，用 `new Request()` 发起的请求就会采成空 URL。

补充一个坑：如果第一个参数是 `Request` 对象，请求体在 `input.body` 上，而且是个 `ReadableStream`，只能通过 `input.clone().text()` 读——不能直接读，会消费掉业务要发的 body。

### 5.3 读响应体：clone 的正确用法

```js
function captureBody(res, info) {
  // ★ 关键：只采 JSON，必须排除流式响应
  const type = res.headers.get('content-type') || ''
  if (!type.includes('application/json')) return

  res.clone().text()                       // clone 出新流，不消耗原始的
    .then(text => {
      info.response = truncate(desensitize(text))
      collect(info)
    })
    .catch(() => {})                       // 读副本失败绝不能影响业务
}
```

**那个 `content-type` 过滤是整个实现里最容易出事的地方**，原因有两个：

1. **SSE 是 `text/event-stream`**。如果对它做 `res.clone().text()`，这个 `text()` 会一直等流结束才 resolve——而 SSE 是长连接，可能几分钟都不结束，等于往内存里挂了一个永不释放的缓冲。页面上开着 AI 对话流式输出的话，内存会持续膨胀。
2. **文件下载是 `application/octet-stream`**。`text()` 会把整个文件的字节当字符串读进来，一个 500MB 的下载直接 OOM。

所以过滤规则要反过来写更安全：**只采白名单类型**（`application/json`），其他一律跳过。

### 5.4 攒批、采样与防死循环

```js
const REPORT_URL = '/api/monitor/report'
const BATCH_SIZE = 20
const FLUSH_MS = 3000
const SAMPLE_RATE = 0.05        // 成功请求采样 5%
const MAX_QUEUE = 100

const queue = []
let timer = null

function collect(info) {
  // 采样：错误全量上报，成功请求按比例
  const isError = info.status >= 400 || info.error
  if (!isError && Math.random() > SAMPLE_RATE) return

  queue.push(info)
  if (queue.length > MAX_QUEUE) queue.shift()     // 队列限长，丢最旧的

  if (queue.length >= BATCH_SIZE) flush()
  else if (!timer) timer = setTimeout(flush, FLUSH_MS)
}

function flush() {
  clearTimeout(timer); timer = null
  if (!queue.length) return

  const batch = queue.splice(0, queue.length)

  // ★ 用 rawFetch 而不是 patchedFetch → 上报请求天然绕过自己，不会死循环
  rawFetch(REPORT_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(batch),
    keepalive: true,
  }).catch(() => {})                              // 上报失败静默，绝不抛给业务
}
```

**防死循环这里用了最干净的一种做法**：直接用闭包里保存的 `rawFetch` 发上报请求。因为 `patchedFetch` 只在 `window.fetch` 这个属性上，闭包里的 `rawFetch` 还是原生的，天然不会被自己拦到。

另一种常见做法是 URL 白名单（在 `describe` 里判断 `url.includes(REPORT_URL)` 就直接放行），但白名单要维护，容易漏。

### 5.5 页面关闭兜底

```js
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState !== 'hidden') return
  if (!queue.length) return

  // sendBeacon 是页面卸载时的唯一可靠通道
  navigator.sendBeacon(REPORT_URL, JSON.stringify(queue.splice(0, queue.length)))
})
```

为什么不用 `beforeunload`：在 `beforeunload` 里发 fetch/XHR，浏览器会直接杀掉这个请求。`sendBeacon` 是专门为这个场景设计的，请求交给浏览器后台发送，页面关了也能发出去。

注意 `sendBeacon` 内部也走网络栈——如果 SDK 同时改写了 `navigator.sendBeacon` 自己（监控 SDK 确实会这么做），这里就要用保存的原始引用来避免二次采集。

---

## 六、核心考点：多个包改写时的"装饰链"

三个包按 A → B → C 顺序加载，每次都是"保存当前引用 → 替换成自己的函数 → 内部调用保存的引用"，于是形成一条**装饰链（洋葱模型）**：

```
加载顺序：A → B → C（后加载者包装前一个的引用）

  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌──────────┐
  │ C 包装  │   │ B 包装  │   │ A 包装  │   │ 原生 XHR │
  │ 最后加载│   │ 中间加载│   │ 最先加载│   │ 底层实现 │
  └─────────┘   └─────────┘   └─────────┘   └──────────┘
   ───────────────────────────────────────────────────▶
          请求穿透：C → B → A → 原生

  ◀───────────────────────────────────────────────────
          响应回传：原生 → A → B → C
```

**结论：三个包的拦截逻辑都会执行，不是只过最后加载的 C。**

### 6.1 一个必须讲准的细节：同步阶段是洋葱，异步阶段不是

这是回答这个追问时最容易含糊的地方，分开说才准确。

**同步调用阶段（`open` / `send` 的调用栈）是真的洋葱**。因为是嵌套函数调用，控制权会一层层还回来：

```js
C_send(body) {
  cLogic()                    // ← C 的前置逻辑
  const ret = B_send(body)    // ← 进入 B（同步等待）
  cPostLogic()                // ← B 返回后才轮到 C 的后置逻辑
  return ret
}
```

对 **fetch** 来说这是完整的洋葱：`await` 链条会真的从原生 → A → B → C 一层层展开，前后置逻辑对称出现。

对 **XHR** 来说，`send()` 是 fire-and-forget 的同步调用，**响应阶段根本不走调用栈**——它是通过 `loadend` / `readystatechange` 事件通知的。而三个包各自 `addEventListener` 注册的监听器是**平级**的，按注册顺序触发（A 先注册先触发），彼此不存在嵌套。

所以严格说：**XHR 的"响应沿链返回"只在事件注册顺序这个意义上成立，不是真正的调用栈回退。**

> 面试时如果能把这一层讲出来（"`send` 是同步的、响应靠事件，所以 XHR 只有请求阶段是洋葱"），比背"洋葱模型"这个名词有说服力得多。

### 6.2 链是怎么断的

前提是每个包都**在 patch 那一刻读取并保存当前引用**，链才天然接上：

```js
// ✅ 正确：prev 拿到的就是上一个包的包装
const prev = XMLHttpRequest.prototype.send
XMLHttpRequest.prototype.send = function (body) {
  cLogic()
  return prev.apply(this, [body])
}
```

两种典型断法：

```js
// ❌ 断法一：保存了"更早的"原生引用（例如从干净的 iframe 里取、或模块初始化时提前缓存）
const nativeSend = iframe.contentWindow.XMLHttpRequest.prototype.send
XMLHttpRequest.prototype.send = function (body) {
  cLogic()
  return nativeSend.call(this, body)   // 直接跳到原生，A、B 全部失效
}

// ❌ 断法二：只管自己的逻辑，忘了调用上一个引用
XMLHttpRequest.prototype.send = function (body) {
  cLogic()                             // 请求压根发不出去
}
```

这就是**多个监控 / 安全 SDK 混用时最经典的事故**：两个 SDK 都想 patch，其中一个写法不严谨，另一个的埋点数据就整体静默丢失。而且现象是"数据少了"而不是"报错了"，排查极其痛苦。

**自检方法**：在 patch 里打一行日志，确认保存的引用不是当前原型上的值：

```js
console.log(rawSend === XMLHttpRequest.prototype.send)   // 应该是 false，否则链有问题
```

---

## 七、拿到数据之后要做什么

采集只是拿到原料，真正的工程工作在后面五步。

```
采集 → 加工 → 缓冲 → 上报 → 服务端消费
```

### 7.1 采集：只抄数据，不做加工

patch 处在**每个请求的关键路径**上，这里只允许做"把 url/body/status 抄一份"这种 O(1) 动作。序列化、正则匹配、字符串处理全部往后挪，否则会直接拖慢全站请求。

### 7.2 加工：这一步最容易出事

| 动作       | 做什么                                                   | 不做的后果                                             |
| ---------- | -------------------------------------------------------- | ------------------------------------------------------ |
| **脱敏**   | 打码手机号、身份证、银行卡、`token`、`password` 字段     | 敏感数据上云，合规事故                                 |
| **截断**   | body / response 限长（比如 8KB）                         | 一个文件上传的请求体几 MB，内存和带宽直接爆炸          |
| **过滤**   | 排除静态资源、sourcemap、HMR，以及**你自己的上报接口**   | 噪声淹没有效数据                                       |
| **聚合**   | URL 归一化：`/api/user/123` → `/api/user/:id`            | 数据是散的，算不出"这个接口平均耗时"                   |

聚合那条尤其关键。不做归一化，同一个接口因为 id 不同会变成几万条独立记录，监控看板直接废掉：

```js
function normalizeUrl(raw) {
  try {
    const u = new URL(raw, location.origin)
    const path = u.pathname
      .replace(/\/\d+(?=\/|$)/g, '/:id')                        // /user/123 → /user/:id
      .replace(/\/[0-9a-f]{8}-[0-9a-f-]{27,}/gi, '/:uuid')      // uuid
    return u.origin + path                                       // 丢掉 query
  } catch {
    return raw
  }
}
```

### 7.3 缓冲：攒批 + 限长 + 采样

- **攒批**：攒够 N 条或等 T 毫秒发一次，别每条都发
- **限长**：队列设上限，写满后丢弃最旧的——和音频帧缓冲的环形队列是同一个思路
- **采样**：成功请求按比例采样（比如 5%），失败请求全量上报。不采样的话高流量页面一秒几百条，上报通道自己就先挂了

### 7.4 上报：分两个时机

```js
// 常态：批量 POST
flush()

// 页面要关了：必须用 sendBeacon
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') {
    navigator.sendBeacon('/api/monitor/report', JSON.stringify(queue))
  }
})
```

`visibilitychange` + `sendBeacon` 是页面关闭前的**唯一可靠通道**——`beforeunload` 里发 XHR/fetch 会被浏览器直接杀掉。

### 7.5 服务端消费：不同场景落脚点不同

| 场景         | 拿到数据后要产出什么                                       |
| ------------ | ---------------------------------------------------------- |
| 监控 / APM   | 成功率、P95/P99 耗时、慢接口排行；和 JS 错误、会话录像关联 |
| 安全 / DLP   | 规则命中告警、审计留痕（谁在什么时候传了什么数据）         |
| 基建注入     | 不需要消费，只是把 trace-id 串起来，交给链路追踪系统       |
| 业务埋点     | 转化漏斗、页面 PV/UV                                       |

### 7.6 四个必须踩住的工程点

**① 上报请求不能被自己拦截（死循环）**

最经典的坑。上报走 fetch，你又 patch 了 fetch，于是上报 → 采集 → 上报 → ……

两种解法：

```js
// 方案一：用闭包里的原生引用发上报（最干净，见 5.4）
rawFetch(REPORT_URL, { ... })

// 方案二：URL 白名单，采集前先放行
const REPORT_URL = '/api/monitor/report'
XMLHttpRequest.prototype.send = function (body) {
  const meta = this[META]
  if (meta && meta.url.indexOf(REPORT_URL) > -1) {
    return rawSend.apply(this, arguments)   // 上报请求直接放行，不采集
  }
  // ...正常采集逻辑
}
```

**② 加工和上报绝不能阻塞主线程**

重活丢给 `queueMicrotask` / `requestIdleCallback`，或者塞进 Web Worker。patch 里的同步代码越短越好。

**③ patch 内的异常绝不能冒泡**

监控 SDK 自身出错导致业务请求发不出去，是最严重的生产事故。每个自定义逻辑块都要独立 try/catch。

**④ 隐私合规**

脱敏、采样、用户告知，这三件事在数据出境相关的业务里是硬要求。安全场景还要额外做本地规则校验，避免敏感数据在传输和存储环节留下痕迹。

---

## 八、坑点速查表

| 位置       | 坑                                  | 后果                                        |
| ---------- | ----------------------------------- | ------------------------------------------- |
| 通用       | 用箭头函数写 patch                  | `this` 丢失，部分调用方式直接崩             |
| 通用       | 忘了 `apply(this, arguments)`       | 参数或 `this` 丢失                          |
| 通用       | 自定义逻辑没有 try/catch            | SDK 自身报错导致业务请求发不出去            |
| 通用       | 没有幂等标记                        | 重复引入时重复 patch，数据重复上报          |
| XHR        | 状态存闭包变量                      | 并发实例串数据                              |
| XHR        | 覆盖 `onreadystatechange`           | 吃掉业务注册的回调                          |
| XHR        | `responseType` 非 text 时读文本     | 抛 `InvalidStateError`；大 blob 导致 OOM    |
| fetch      | 只处理 string 入参                  | `Request` 对象的请求采成空                  |
| fetch      | 忘了 `throw err`                    | 业务 catch 永久挂起                         |
| fetch      | `new Response()` 包装返回           | 丢 statusText / url / 破坏流式响应          |
| fetch      | 没过滤 `content-type` 就读响应体    | SSE 内存泄漏、大文件 OOM                    |
| fetch      | 直接 `res.text()`                   | 业务读不到 body                             |
| 上报       | 用改写后的 fetch 发上报             | 死循环                                      |
| 上报       | 无采样 / 队列不限长                 | 上报通道被打爆、弱网下内存膨胀              |
| 兜底       | 用 `beforeunload` 发数据            | 关页面时丢数据                              |
| 装饰链     | 保存了更早的引用                    | 先加载的拦截器静默失效                      |

---

## 九、面试怎么答（话术模板）

### 时间分配（约 2 分钟）

> fetch 改写的本质是做一个**透明代理**——在 `window.fetch` 外面包一层，业务代码完全感知不到。我会分三块考虑：
>
> **第一块是改写本身必须透明，这是底线。** 因为 fetch 是全局函数，改坏了整个页面的请求都挂。守四点：`this` 和全部参数用 `apply` 透传；第一个参数可能是 string、URL 或 Request 三种形态，都要处理；返回值必须是**原始的 Response 对象**，不能用 `new Response()` 包一层；异常必须重新 `throw`，否则业务侧的 catch 永远收不到。另外要读响应体的话必须 `clone()`，body 是一次性流，直接读会把业务要用的数据消费掉。
>
> **第二块是采集和加工。** 采集 url、method、status、耗时、请求头和响应体。但采到不能直接上报，要加工：一是**脱敏**，手机号、token、密码字段打码；二是**截断**，body 和响应体限长；三是 **URL 归一化**，把 `/api/user/123` 聚合成 `/api/user/:id`，否则同一个接口因为 id 不同会变成几万条记录，监控看板直接废掉。
>
> **第三块是上报通道。** 不能每条请求都发一次，要**缓冲攒批**——队列设上限，满了丢最旧的；成功请求**采样**上报，失败请求全量。页面关闭时用 `visibilitychange` 加 `sendBeacon` 兜底，因为 `beforeunload` 里发 fetch 会被浏览器杀掉。
>
> 最后有一个必须防的坑：**上报请求不能被自己拦截**，否则就是死循环，一般用闭包里的原生引用发上报，或者按上报接口 URL 白名单放行。

### 好在哪

| 面试官在考         | 答案里的对应内容                                |
| ------------------ | ----------------------------------------------- |
| 会不会把业务搞挂   | 第一块的透传 `this` / 参数 / 返回值 / 异常 + clone |
| 有没有工程意识     | 第二块的脱敏、截断、聚合                        |
| 懂不懂上报的工程约束 | 第三块的攒批、采样、sendBeacon                |
| 有没有实战经验     | 结尾的防死循环                                  |

关键差别：**别人答"我采集 url、method、status、耗时上报"，你答"我先保证不污染业务，再谈采集，采完还要加工，最后才谈上报"**——前者是功能清单，后者是工程判断。

### 两个加分动作

**① 结尾主动埋钩子。** 说完收尾加一句：

> 实现上还有两个点比较容易踩：一个是 `AbortError` 要单独处理，用户主动取消不能算错误上报；另一个是装饰链——如果页面上还有别的 SDK 也改写了 fetch，要保证保存的是 patch 那一刻的引用，不然先加载的拦截器会静默失效。

这句话把 AbortController 和装饰链都递到面试官手上，他大概率会顺着追问——而这两块都准备好了。

**② 补一句 XHR 的差异。** 面试官追问"那 XHR 呢"，一句话接住：

> 思路一样，只是 XHR 的请求数据分散在 `open` 和 `send` 两个方法里，而且响应靠 `loadend` 事件而不是 Promise，所以装饰链在响应阶段是各包监听器平级触发，不像 fetch 那样是真的洋葱。

### 一个反向提醒

如果面试官问"你**实际**做过 fetch 改写吗"，别硬编项目经历。诚实说"我在某个场景下做过/研究过"，或者"我了解实现方式和主要坑点，但没在生产环境落地过"。技术面试官对"讲得清原理但没做过"是能接受的，对"编一个项目然后被追问细节答不上来"是零容忍的。

---

## 十、把零散技能串起来

这道题有意思的地方在于，它几乎能串起前端工程的一大半常识：

- **环形缓冲丢最旧** —— 队列限长策略，和音频帧缓冲、请求去重队列同一套思路
- **幂等设计** —— patch 的幂等标记、接口幂等（Redis SETNX 去重）是同一个思维
- **AbortController** —— 取消语义要在监控里被正确识别，否则产生假错误
- **Service Worker** —— 拦截能力最强但有激活时序限制，和 Monkey Patch 是互补关系
- **流式响应的特殊性** —— SSE 是长连接，任何"读完整 body"的操作都会出事

所以答这道题时，如果能顺手把这些关联点带出来，展示的就是"体系化的理解"而不是"背过这道题"。