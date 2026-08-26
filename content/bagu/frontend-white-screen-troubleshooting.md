---
title: '线上白屏问题排查思路'
date: 2026-08-26
draft: false
categories: ['前端']
tags: ['性能优化', '故障排查']
---

白屏是线上最严重的前端故障之一——用户打开页面一片空白，什么都看不到，直接流失。对电商这类业务，白屏时间每增加 1 秒转化率显著下降，大面积白屏甚至等于业务瘫痪。

面试中被问到「线上出现白屏你怎么排查」，考察的不是背原因，而是**有没有系统化的排查方法论**：能不能先稳住故障、再沿渲染链路逐段收敛、最后给出监控防复发的完整闭环。

## 白屏的本质：渲染流水线断裂

浏览器渲染出一个页面，要经过一条完整链路：

```
DNS 解析 → 建立连接 → 请求 HTML → 解析 DOM → 加载 JS/CSS
        → 执行 JS → 构建 DOM 树 → 生成渲染树 → 绘制像素
```

**白屏 = 这条流水线上某一环断了。** 所以排查的本质，就是沿链路逐段验证，找到断裂点。理解了这一点，所有具体的排查手段都是这条主线的展开。

## 排查思路（面试主干答案）

### 第零步：先止血，再排查 ⭐

很多人会忽略这一步。如果白屏影响的是全量用户，**第一动作不是找根因，而是恢复服务**：

- **回滚版本**：通过 CI/CD 快速回滚到上一个稳定版本，绝大多数白屏都是新版本引入的
- **降级**：开启功能开关降级新功能，或切到兜底页面

止血和排查不冲突——回滚后拿着出错版本慢慢分析，用户侧已经恢复。这个意识在面试中是重要加分项：**故障处理的优先级永远是恢复 > 定位 > 复盘**。

### 第一步：确认影响范围，收敛排查方向

不要上来就翻代码，先花一分钟定位问题的影响面：

| 维度   | 排查点                       | 指向的结论                                   |
| ------ | ---------------------------- | -------------------------------------------- |
| 时间   | 刚发版后出现，还是一直存在？ | 发版后 → 大概率新版本代码问题                |
| 用户   | 全量还是部分用户？           | 全量 → 构建/资源问题；部分 → 环境/兼容性问题 |
| 浏览器 | 是否集中在低版本或特定内核？ | 集中低版本 → 语法兼容性问题                  |
| 页面   | 全站白屏还是某个路由？       | 单页面 → 该页面的代码/接口问题               |
| 地域   | 是否集中某个地区？           | 集中某地域 → CDN 节点故障、DNS 污染          |

这一步能直接砍掉一半的排查方向。

### 第二步：看监控，让数据说话

如果有前端监控（Sentry、Fundebug、自建埋点等），大概率能直接拿到答案：

- **JS 运行时错误**：`window.onerror` / `unhandledrejection` 上报的错误堆栈，配合 sourceMap 反解出源码位置，直接定位出错代码
- **资源加载错误**：捕获阶段监听 `error` 事件拿到的 script/css 加载失败
- **接口异常**：核心接口 5xx 或超时，首屏数据拿不到导致渲染中断
- **性能指标**：FCP 一直不触发或异常偏大，说明用户看到的一直是白屏

{{< alert icon="lightbulb" >}}
如果监控里干干净净什么都没有，反而是一个重要线索——说明错误发生在监控 SDK 初始化之前（比如入口 JS 就 404 了），或者报错方式超出了捕获范围（比如语法错误导致整个 JS 文件一行都不执行，监控代码也在里面）。
{{< /alert >}}

### 第三步：亲自复现，打开 DevTools

拿到具体案例后自己复现，按三个面板逐个看：

**1. Network 面板**

- HTML 文档：状态码多少？内容对吗？是不是网关错误页？
- JS/CSS：有没有 404 / 504？CDN 是否正常？
- 一个高频信号：**JS 文件的响应内容是 HTML**——通常是 Nginx 把所有路径都 fallback 到了 `index.html`，或构建路径配错
- 核心接口：有没有挂掉或长时间 pending？

**2. Console 面板**

- `Uncaught SyntaxError` → 语法/兼容性问题，整个文件不执行，杀伤力最大
- `Uncaught TypeError: Cannot read properties of undefined` → 运行时错误，常见于接口数据结构变更
- `ChunkLoadError` → 发版后旧 chunk 被删（下面详述）

**3. Elements 面板**

- `#app` / `#root` 容器到底有没有内容？
- 有 DOM 但不可见 → CSS 问题（样式没加载、`display: none`、容器高度塌陷）
- 完全没有 DOM → JS 执行中断，渲染函数根本没跑

## 常见原因清单（按出现频率排序）

### 1. 发版导致的 chunk 加载失败 ⭐最高频

发新版后，旧 hash 的 chunk 文件（`app.abc123.js`）被删除，但用户浏览器里还持有旧 HTML / 旧路由，按旧 hash 去请求资源就 404 了：

```
ChunkLoadError: Loading chunk 12 failed
```

**解决方案一：监听加载失败，自动刷新**（用 sessionStorage 防止无限刷新）：

```js
window.addEventListener(
  'error',
  (e) => {
    // 资源加载失败必须用捕获阶段才能拿到
    if (e.target?.tagName?.toLowerCase() === 'script') {
      if (!sessionStorage.getItem('reloaded')) {
        sessionStorage.setItem('reloaded', '1')
        location.reload() // 刷新后拿到最新 HTML 和最新 hash 的资源
      }
    }
  },
  true,
)
```

**解决方案二：HTTP 缓存策略**（根治）：HTML 不缓存（`Cache-Control: no-cache`），只缓存带 hash 的静态资源，保证用户永远先拿到最新 HTML。

### 2. JS 运行时错误中断渲染

SPA 首屏渲染是一次同步执行流，入口处任何一个未捕获异常都会让整棵组件树渲染不出来：

```js
// 后端返回的数据结构和预期不符
const name = res.data.user.name // TypeError: Cannot read properties of undefined
```

**解决方案**：核心链路 try/catch 兜底 + **ErrorBoundary（错误边界）** 把异常控制在局部——React 用类组件的 `getDerivedStateFromError` + `componentDidCatch` 实现，Vue 对应 `app.config.errorHandler`。效果是一个组件挂了展示兜底 UI，不再整页陪葬。

### 3. 浏览器兼容性问题

构建产物里包含目标浏览器不支持的语法（可选链 `?.`、`??`、顶层 await 等），旧浏览器直接 `SyntaxError`。这种错误发生在**解析阶段**，整个 JS 文件一行都不执行——包括监控代码，所以监控里往往是空白。

典型特征：**错误集中在低版本浏览器，且监控捕不到**。

**解决方案**：

- `browserslist` 圈定目标浏览器范围，配合 Babel 转译和 polyfill
- 确认构建产物在目标浏览器上能跑（用 BrowserStack 等工具验证）

### 4. 首屏接口阻塞或异常

页面渲染依赖首屏接口，接口 5xx、超时、返回结构变更都会导致渲染中断或一直 loading。和 JS 错误的区别：Network 里能看到接口异常，且错误发生在异步回调里，不一定中断整页。

### 5. CSS 加载失败 / 样式问题

- CSS 文件 404 → 内容都在但「裸奔」，移动端容器高度塌陷常被误报为白屏
- 关键 CSS 异步加载，首帧渲染时样式未到
- 字体文件阻塞渲染，缺 `font-display: swap`

### 6. HTML 本身就没正常返回

- 服务端 5xx / 网关超时（Nginx 502/504）
- CDN 证书未同步，部分用户请求被浏览器拦截
- 运营商劫持注入非法内容导致解析失败

验证方法：`curl -v` 直接看原始响应。

## 终极问题：怎么提前发现白屏？（监控方案）

排查是被动的，面试官大概率会追问「怎么防患于未然」。

### 采样点检测：主流方案

思路：页面加载后，在视口取多个采样点（比如横竖各 9 个点，十字分布），用 `elementsFromPoint` 查每个点下是什么元素——如果所有点下的都是「空白容器」（html / body / #app 本身），说明页面上没有任何实际内容，判定为白屏：

```js
function checkBlank() {
  let emptyPoints = 0
  for (let i = 1; i <= 9; i++) {
    // 横向 9 点 + 纵向 9 点
    const xEl = document.elementsFromPoint(
      (innerWidth * i) / 10,
      innerHeight / 2,
    )[0]
    const yEl = document.elementsFromPoint(
      innerWidth / 2,
      (innerHeight * i) / 10,
    )[0]
    // 采样点下的元素是 html/body/#app 这些空容器 → 记一次空白点
    if (isWrapper(xEl)) emptyPoints++
    if (isWrapper(yEl)) emptyPoints++
  }
  // 17 个点里 16 个以上都是空白 → 判定白屏，上报
  if (emptyPoints >= 16) reporter.send('blank_screen', { url: location.href })
}

window.addEventListener('load', checkBlank)
```

两个工程细节：

- **轮询修正**：初次判定白屏后定时复检几次，防止「页面只是加载慢」被误判
- **上报带上下文**：白屏发生时把最近的 JS 错误（提前用 `window.onerror` 存下来）一起上报，检测和归因一步到位

### 辅助手段

**错误捕获打底**（要在业务代码之前执行）：

```js
window.addEventListener('error', handler, true) // 捕获阶段才能拿到资源加载错误
window.addEventListener('unhandledrejection', handler) // 捕获未处理的 Promise 拒绝
```

**告警闭环**：白屏率 = 白屏 PV / 总 PV，接入告警（如 > 0.1% 告警），并关联发布记录——白屏率飙升的时间点如果刚好对应一次发布，基本就锁定了版本。

## 事后复盘：怎么让白屏不复发

- **缓存策略**：HTML 强制 `no-cache`，静态资源带 hash 长缓存
- **降级体验**：骨架屏 / SSR 首屏直出，即使 JS 挂了用户也能看到内容
- **发布流程**：灰度发布 + 发布后自动回归验证（打开核心页面截图比对）
- **CI 卡点**：构建产物上传失败阻断发布、browserslist 覆盖率检查

## 总结：一张图记住排查链路

```
白屏发生
  ├─ 第零步：止血 ──→ 影响全量？先回滚版本恢复服务
  ├─ 影响面分析 ────→ 全量/部分？发版后？特定浏览器/地域？
  ├─ 看监控 ────────→ JS 错误？资源失败？接口异常？FCP？
  ├─ 复现诊断 ──────→ 挂载点有 DOM 吗？（无 → JS；有但看不见 → CSS）
  │   ├─ Network ──→ HTML 到了吗？JS/CSS 200 吗？接口挂了吗？
  │   └─ Console ──→ SyntaxError（兼容性）？TypeError（运行时）？ChunkLoadError（发版）？
  └─ 事后 ────────→ 白屏率监控 + 告警闭环 + 复盘改进
```

{{< alert icon="lightbulb" >}}
回答这类问题的关键是展现「先恢复、再定位、后预防」的完整闭环，以及「沿渲染流水线收敛」的结构化思维，而不是一上来就罗列十几个可能原因。排查是收敛过程，不是发散过程。
{{< /alert >}}

## 附：面试参考回答

> 如果影响的是全量用户，我会先回滚版本恢复服务，再慢慢排查——故障处理永远是恢复优先于定位。
>
> 排查上，我会把白屏理解为**渲染流水线在某一步断了**：DNS、请求 HTML、加载 JS/CSS、执行 JS、渲染 DOM，逐段验证。
>
> 先确认影响范围缩小方向：是刚发版后出现的、还是全量用户都白？如果集中在低版本浏览器，大概率是兼容性语法问题；如果集中在某个地域，往 CDN 和 DNS 查。
>
> 然后看监控：`window.onerror` 和 `unhandledrejection` 的上报能直接给出错误堆栈，配合 sourceMap 定位到源码。有个细节是——如果监控里什么都没有，反而说明入口 JS 本身就没加载成功，或者语法错误导致整个文件没执行。
>
> 拿到案例后自己复现，看 DevTools：Network 里 HTML 和 JS/CSS 有没有 404，JS 响应内容是不是 HTML；Console 里 SyntaxError 指向兼容性、TypeError 指向运行时错误、ChunkLoadError 指向发版后旧 chunk 被删；Elements 里挂载点有 DOM 但看不见就是 CSS 问题，没 DOM 就是 JS 执行中断。
>
> 实际线上最高频的是发版导致的 chunk 加载失败——旧 hash 的文件被删了，用户还持着旧 HTML。解法是监听资源加载失败自动刷新一次，更根本的是 HTML 不缓存、只缓存带 hash 的资源。
>
> 最后是预防：用 `elementsFromPoint` 十字采样加轮询修正做白屏检测，配合 FCP 指标和白屏率告警，关联发布记录，白屏一飙就能定位到版本。代码层面用 ErrorBoundary 兜底，保证单个组件报错不会导致整页白屏。
