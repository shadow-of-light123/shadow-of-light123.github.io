---
title: '字节抖音直播前端日常实习一面'
date: 2026-07-23
draft: false
categories: ['面经']
tags: ['前端', '字节跳动', '手撕代码']
---

## 面试问题

### 1. 自我介绍、介绍项目

**回答思路**：自我介绍控制在 1 分钟左右，结构是「基本信息 + 技术方向 + 核心项目亮点」。

- 基本信息：学校、年级、实习意向
- 技术方向：React/Vue 技术栈、工程化能力
- 项目亮点：挑最有含金量的一两个讲，突出「背景 + 我的职责 + 难点 + 结果」，比如：AI 对话应用中流式输出（SSE）、语音转写（WebSocket）的落地，以及后台管理系统的 RBAC 权限体系
- 结尾引导面试官往自己熟悉的领域提问

### 2. 文本流式输出选用 SSE、语音转写选用 WebSocket 的原因

**回答**：

- **文本流式输出是纯单向场景**：只需要服务端把 LLM 逐 token 生成的文本推给浏览器，客户端不需要回传数据。SSE 基于普通 HTTP，天然适合这种「服务端 → 客户端」的单向推送，实现简单、兼容性好（`EventSource` 原生支持），还有自动重连、`Last-Event-ID` 断点续传等开箱即用的能力。
- **语音转写是双向实时场景**：浏览器要持续上传麦克风采集的音频流（客户端 → 服务端），同时接收转写文本（服务端 → 客户端）。SSE 做不到客户端向服务端推流，而 WebSocket 提供全双工通道，且支持二进制帧（音频数据是二进制的，SSE 只能传 UTF-8 文本），长连接通信开销也更小，延迟更低。

**一句话总结**：单向推文本用 SSE，双向传流用 WebSocket，按数据方向和数据格式选型。

### 3. SSE 和 WebSocket 两者区别

**回答**：

| 维度        | SSE                                                         | WebSocket                                     |
| ----------- | ----------------------------------------------------------- | --------------------------------------------- |
| 协议        | 基于普通 HTTP（`text/event-stream`）                        | 独立协议（`ws://` / `wss://`，HTTP 101 升级） |
| 通信方向    | 单向（服务端 → 客户端）                                     | 全双工双向                                    |
| 数据格式    | 仅文本（UTF-8）                                             | 文本 + 二进制                                 |
| 重连机制    | 浏览器自动重连，支持 `retry` 和 `Last-Event-ID` 续传        | 需要自己实现心跳 + 重连                       |
| 连接数限制  | HTTP/1.1 下同域名有 6 个连接数限制（HTTP/2 多路复用可解决） | 无此限制                                      |
| 代理/防火墙 | 走 HTTP 基础设施，穿透性更好                                | 部分代理对 Upgrade 支持不好                   |
| API         | `EventSource`，自带 `onmessage`                             | `WebSocket`，需自己封装心跳重连逻辑           |

**选型**：只需要服务端推送（LLM 流式输出、通知、行情）选 SSE；需要双向实时（语音、聊天室、协同编辑）选 WebSocket。

### 4. SSE 重连、WebSocket 心跳保活机制，项目落地方案

**回答**：

**SSE 重连**：

- 浏览器 `EventSource` 断线后**默认自动重连**，服务端可以通过 `retry: 3000` 字段告知重连间隔
- 服务端为每条消息设置 `id:` 字段，重连时浏览器自动带上 `Last-Event-ID` 请求头，服务端据此续传，避免丢消息
- 项目里对于 LLM 场景：回答中途断线就从头生成代价高，所以服务端把已生成内容按消息 id 缓存，重连后从断点继续推
- 兜底：`onerror` 里判断 `readyState`，用**指数退避**手动重连（`EventSource` 自动重连失效的场景，如 Nginx 缓冲导致连接假死，配合 `X-Accel-Buffering: no` 关闭代理缓冲）

**WebSocket 心跳保活**：

- TCP 连接断了但没发 RST 时，两端都感知不到「假死」，需要心跳检测
- 方案：客户端每 30s 发一个 `ping` 帧或自定义心跳消息，服务端回 `pong`；客户端在收到任何消息时重置计时器，若超过阈值（如 10s）没收到回复，判定连接死亡，主动 `close` 并重连
- 重连同样用指数退避 + 抖动（避免服务端重启瞬间被重连风暴打垮）
- 项目里封装了一个 `ReconnectingWebSocket` 类：心跳定时器、重连退避、消息队列（断线期间的消息缓存重发）、连接状态事件回调

### 5. 项目里 Redux 和 TanStack Query 如何划分状态

**回答**：核心原则是「客户端状态」和「服务端状态」分治。

- **Redux（Pinia 同理）管客户端状态**：UI 全局状态、用户信息、权限数据、主题、侧边栏折叠状态——这些数据由前端自己拥有，不存在「过期」概念
- **TanStack Query 管服务端状态**：接口返回的数据本质是**服务端状态在客户端的缓存快照**，它天生有缓存过期（`staleTime`）、后台重新验证、请求去重、缓存失效（`invalidateQueries`）、乐观更新等需求，用 Redux 手写这套逻辑样板代码非常多

这样划分后 Redux 里不再存接口数据，只存真正的全局 UI 状态，代码量和心智负担都明显下降。

### 6. 哪些状态适合放在组件内部，举例说明

**回答**：判断标准是「**是否有其他组件需要共享或同步这份状态**」。不需要共享的都放组件内部：

- 弹窗/抽屉的开关：`const [open, setOpen] = useState(false)`
- 下拉框展开态、手风琴当前展开项
- 表单的草稿值（提交前本地维护，提交时才提全局或发请求）
- hover、loading 等瞬时 UI 状态
- 受控输入的临时值

反例：登录态、权限、主题这些多个页面/组件都要用的，才值得提到全局。

### 7. 事件循环代码输出结果并解释执行顺序

**回答**：典型例题：

```js
console.log('script start')

setTimeout(() => {
  console.log('setTimeout')
}, 0)

Promise.resolve()
  .then(() => {
    console.log('promise1')
  })
  .then(() => {
    console.log('promise2')
  })

console.log('script end')

// 输出顺序：
// script start
// script end
// promise1
// promise2
// setTimeout
```

**执行规则**：

1. 先执行同步代码（调用栈里的脚本本身相当于一个宏任务）
2. 每个宏任务执行完后，**清空整个微任务队列**（Promise.then 回调、`queueMicrotask`、MutationObserver），微任务执行中产生的新微任务也会在本轮清空
3. 然后取下一个宏任务（setTimeout、setInterval、I/O、UI 渲染），周而复始
4. 同级的微任务按入队顺序 FIFO 执行；`promise2` 是 `promise1` 的回调返回后才入队的，所以在后面

**关键点**：微任务优先级高于宏任务；Promise 构造函数里的代码是同步执行的，`.then` 回调才是微任务。

### 8. Vue3 响应式原理

**回答**：Vue3 用 **Proxy** 重写了响应式系统，三个核心角色：**effect（副作用）、track（依赖收集）、trigger（触发更新）**。

- `reactive(obj)`：返回 obj 的 Proxy，拦截 get/set/deleteProperty/has 等操作
- **依赖收集（get 拦截）**：组件渲染时会执行渲染函数（一个 effect），执行过程中访问到响应式数据，就把它存进 `targetMap`（结构：`WeakMap<target, Map<key, Set<effect>>>`），key 对应的所有 effect 就是它的依赖
- **触发更新（set 拦截）**：修改数据时，从 `targetMap` 里取出这个 key 的 effect 集合，逐个重新执行，组件由此重新渲染
- `ref(基本类型)`：因为 Proxy 只能代理对象，基本类型用「具备 getter/setter 的 RefImpl 类」实现，`.value` 访问时 track，赋值时 trigger
- `computed`：惰性求值 + 脏标记（dirty），依赖变化只置脏，真正读取时才重算
- 嵌套对象**惰性代理**：访问到内层对象时才递归 `reactive`，初始化开销比 Vue2 递归遍历小得多

### 9. Vue2 和 Vue3 响应式实现核心区别

**回答**：

| 维度          | Vue2                                            | Vue3                                       |
| ------------- | ----------------------------------------------- | ------------------------------------------ |
| API           | `Object.defineProperty`                         | `Proxy`                                    |
| 新增/删除属性 | 无法侦测，需 `Vue.set` / `Vue.delete`           | 原生支持（`set`、`deleteProperty` 拦截器） |
| 数组          | 下标和 length 修改无法侦测，靠重写 7 个数组方法 | 直接拦截，下标赋值、length 修改均可侦测    |
| Map/Set       | 不支持                                          | 通过集合处理器支持                         |
| 初始化时机    | 初始化时递归遍历所有属性定义 getter/setter      | 惰性代理，访问到才转化，性能更好           |
| 兼容性        | 支持 IE9+                                       | Proxy 无法 polyfill，不支持 IE             |

### 10. Vue2 直接通过数组下标赋值无法更新视图的原因

**回答**：两个层面：

1. `Object.defineProperty` 只能拦截**已存在属性**的 get/set。Vue2 对数组只在初始化时把每个**元素对象**做成响应式，但「通过下标赋值」这个动作是修改数组本身的结构，defineProperty 拦截不到（对数组用 defineProperty 逐个下标定义理论上可行，但性能开销大——大数组要循环定义、稀疏数组浪费严重，所以 Vue2 放弃了这条路）
2. Vue2 的兜底方案是**重写 7 个数组变异方法**（push/pop/shift/unshift/splice/sort/reverse）来手动派发更新，而 `arr[0] = x` 不经过这 7 个方法，所以视图不更新

### 11. Vue2 修改数组下标并触发视图更新的解决方案

**回答**：

1. **用重写过的变异方法**：`arr.splice(index, 1, newValue)`（最常用）
2. **`Vue.set(arr, index, value)`** / 实例上的 `this.$set(arr, index, value)`
3. **整个替换数组**（引用替换触发 setter）：`this.arr = this.arr.map(...)`、`[...this.arr]` 后赋值
4. 兜底：`this.$forceUpdate()` 强制重渲染（不推荐，治标不治本）

### 12. 后台平台动态路由、RBAC 权限整套实现流程

**回答**：整体流程串起来讲：

1. **登录**：账号密码 → 换取 token，存 localStorage/cookie
2. **拉取权限**：用 token 请求 `/user/info`，后端根据 RBAC 模型（用户 → 角色 → 权限点）返回用户的角色和权限标识列表（权限码、菜单树）
3. **路由匹配**：前端路由表全量定义（或后端下发菜单树），每条路由 `meta` 里声明所需权限/角色；用过滤器筛出该用户有权的路由，得到「动态路由表」
4. **注册路由**：遍历动态路由表，`router.addRoute()` 动态添加，并把路由表存进 Pinia，侧边栏菜单由它渲染
5. **路由守卫**：`router.beforeEach` 里校验目标路由所需权限，无权则重定向 403；未拉取过权限则先异步拉取再放行
6. **按钮级控制**：指令/组件根据权限码控制操作入口（详见第 14 题）

### 13. 页面刷新时如何恢复路由与权限信息

**回答**：刷新会清空内存，Pinia 里的用户信息和动态路由全部丢失，而路由守卫照常执行，处理方案：

- token 持久化在 localStorage/cookie，刷新后仍在
- **在 `router.beforeEach` 里判断「有 token 但 Pinia 里没有用户信息」**：说明是刷新场景，先 `await` 拉取用户信息 + 权限 → 重新 `addRoute` 注册动态路由 → 再用 `return next({ ...to, replace: true })` 重新进入目标路由（因为 addRoute 是异步生效的，必须重新导航一次才能匹配到新路由）
- 权限信息本身也可以用 `pinia-plugin-persistedstate` 持久化，但**每次刷新仍应以接口返回为准**重新校验，防止权限变更后本地缓存过期
- 用户退出/换账号时清空缓存并 `removeRoute` 重置

### 14. 按钮级别的前端权限如何实现

**回答**：

1. **自定义指令**（主流方案）：`v-permission="'user:delete'"`，指令 `mounted` 钩子里从 Pinia 取权限码列表，不包含则 `el.parentNode.removeChild(el)` 把按钮从 DOM 移除
2. **函数式判断**：封装 `hasPermission('user:delete')` 工具函数，用于逻辑判断（如提交前校验）
3. **组件封装**：`<Auth code="user:delete"><el-button/></Auth>` 包裹层
4. 数据来源：登录后后端返回权限码集合，菜单和按钮统一由权限码驱动渲染

### 15. 只依靠前端权限控制是否足够，后端如何兜底

**回答**：**远远不够**。前端权限只是体验层的「隐藏入口」，请求可以绕过页面直接发（Postman、改代码、直接 curl），前端代码里甚至能看到所有路由和接口。

后端兜底：

1. **接口级鉴权**：每个请求校验 token 有效性 + 用户是否拥有该接口对应权限码（拦截器/注解 `@PreAuthorize`/网关统一鉴权），无权返回 403
2. **数据级权限**：同一接口不同角色可见的数据范围不同（只能看本部门的数据），在查询条件里做行级过滤
3. **敏感字段服务端脱敏**：不把无权看到的数据下发，而不是下发后靠前端隐藏
4. 前端隐藏 + 后端强校验，**纵深防御**，前端管体验、后端管安全

### 16. 项目的性能优化数据是如何统计的

**回答**：分「实验室数据」和「真实用户数据」两条线：

- **真实用户监控（RUM）**：接入 `web-vitals` 库采集 LCP / INP / CLS，在 `onLCP` 等回调里把指标连同页面 URL 一起上报到监控平台（如 Sentry 或自建埋点接口），按版本聚合看分位数（P75）
- **实验室数据**：Lighthouse CI 跑构建产物、Chrome DevTools Performance 面板分析长任务、Network 面板看资源体积和瀑布图
- **对比方法**：优化前后用同一环境、同一网络限速条件多次测量取均值，报告里给出「优化前 → 优化后」的具体数字（如 LCP 从 3.2s 降到 1.8s），有数据才有说服力

### 17. 路由懒加载为什么可以优化首屏加载、改善 LCP

**回答**：

- **不懒加载的坏处**：所有路由组件打进一个 bundle，首屏要下载并解析执行全部 JS，阻塞主线程；页面越大首包越臃肿
- **懒加载（`() => import('./view.vue')`）**：按路由拆成独立 chunk，进入该路由才加载。首屏只下载当前路由的代码，**关键资源体积大幅减小**
- **LCP 改善的链路**：JS 下载/解析/执行时间变短 → 首次内容渲染提前 → LCP 元素（大图或文本块）更早完成绘制；同时主线程空闲更早，图片等资源也能更早发起加载
- 附带收益：按需加载也让浏览器缓存命中更精准（改一个路由不影响其他 chunk 的缓存）

### 18. 打包分包拆分 chunk 的作用，如何结合浏览器缓存优化

**回答**：

**拆 chunk 的作用**：

- **vendor 分包**：把 `node_modules` 里变动频率极低的依赖（react、vue、antd）单独拆出，与业务代码解耦
- **公共模块复用**：多个路由共享的组件/工具抽成公共 chunk，避免重复打包
- **单文件体积控制**：利于并行下载、按需加载，也避免单文件过大解析阻塞

**结合浏览器缓存**：

1. 文件名带**内容 hash**（`app.a1b2c3.js`）：内容不变 hash 不变，内容一变 hash 必变
2. 对带 hash 的静态资源设置**强缓存**：`Cache-Control: max-age=31536000, immutable`，一年内二次访问直接走本地缓存，不发请求
3. 发版后只有内容变化的 chunk 换了新 hash，其余 chunk 缓存全部命中，用户只需增量下载
4. **runtime chunk 单独拆**：manifest/运行时代码单独抽出，避免因为它变动导致所有引用它的 chunk hash 连锁变化
5. `index.html` 不加 hash，用 `no-cache` 保证每次拿到最新入口，引用新的 chunk 文件名

### 19. CDN 加速原理以及项目中落地方式

**回答**：

**原理**：

- 核心是「**边缘缓存 + 就近访问**」：静态资源缓存到全国/全球各地的边缘节点，用户请求先经 DNS 智能调度（GSLB）解析到**地理上最近的边缘节点**
- 节点缓存命中 → 直接返回，RTT 和传输时间都大幅缩短；未命中 → 边缘节点回源拉取并缓存，供后续请求命中
- 同时分散了源站带宽与连接压力，还能抗流量峰值、隐藏源站 IP

**项目落地**：

- 构建产物（js/css/图片/字体）上传到对象存储，前置 CDN 加速域名，`vite.config` 里 `base`/`publicPath` 指向 CDN 域名
- **HTML 保留在源站**（不缓存或短缓存），保证发版即时生效；带 hash 的静态资源设长缓存
- 图片走 CDN 时开启压缩、WebP 自适应等处理，字体做子集化

### 20. 表格分页渲染能带来哪些优化收益

**回答**：

- **DOM 数量骤降**：一万行 × 每行若干单元格的 DOM 节点，会带来显著的内存占用和渲染开销；分页后一页 20 条，DOM 从几万节点降到几百
- **渲染性能**：浏览器 layout/paint 的工作量与 DOM 规模正相关，节点少了首屏渲染和交互（滚动、hover）更流畅，避免长任务阻塞主线程
- **数据传输**：配合后端分页（`page`/`pageSize` 参数），一次只传当前页数据，减少接口响应体和序列化开销
- **查询压力**：后端用 `LIMIT/OFFSET` 只查当前页，数据库压力小
- 补充：如果业务要求一页展示大量数据又不接受分页，可升级为**虚拟滚动**（只渲染可视区域）

### 21. 日常开发使用 AI 工具的场景与技巧

**回答**：

**场景**：

- 脚手架/样板代码生成（列表页、表单、类型定义）
- 单元测试生成、代码审查与重构建议
- 正则、SQL、Shell 命令的编写与解释
- 报错信息分析、不熟悉的第三方库 API 探索
- 技术方案调研、文档撰写

**技巧**：

- **给足上下文**：技术栈、版本、目录结构、约束条件（如「Vue3 组合式 API + TS」），产出质量天差地别
- **小步迭代**：先让它出方案，确认后再写代码，比一次生成一大坨再返工效率高
- **要求解释**：让它解释为什么这样写，把 AI 当导师而不只是代码机，顺便校验答案
- **沉淀规则**：在 Cursor 里维护 `.cursorrules` / 项目规范文件，让 AI 记住团队约定
- **永远验证**：AI 会编造 API（幻觉），跑测试、读文档确认后才合入

### 22. 谈谈你对 Skill、MCP 这类 AI 相关概念的理解

**回答**：

- **MCP（Model Context Protocol，模型上下文协议）**：一个**开放标准协议**，规范了「LLM 应用」和「外部工具/数据源」之间的连接方式。MCP Server 暴露三类能力——Tools（可调用的函数）、Resources（可读取的数据）、Prompts（提示模板）；宿主应用（Claude Desktop、Cursor 等）作为 Client 发现并调用它们。类比：**MCP 之于 AI 应用，就像 USB-C 之于外设**——统一接口后，任何模型接任何工具都不用重复造轮子
- **Skill（技能）**：把某类任务的**最佳实践打包成可复用的指令包**（提示词 + 脚本 + 资源），教会 AI「怎么做某件事」，比如「生成 PPT」「代码审查」。模型遇到匹配场景时自动加载对应 Skill，相当于给通用模型注入领域工作流
- **本质理解**：这些概念都在推动 AI 从「聊天对话」走向「Agent 工作流」——MCP 解决 AI **调用外部能力**的问题（手），Skill 解决 AI **掌握做事方法**的问题（经验），配合规划与记忆，AI 才能真正自主完成复杂任务

### 23. 开发过程中最有挑战性的功能是什么

**回答思路**（STAR 法则，以流式对话为例）：

- **S 背景**：AI 对话应用中，LLM 回答要 10 秒以上才完整返回，用户体验极差
- **T 任务**：实现逐 token 流式输出的完整链路
- **A 行动**：对比 SSE 与 WebSocket 后选 SSE（单向、自动重连、轻量）；后端改造为流式转发 LLM 的 chunk；前端处理 Markdown 增量渲染的闪动问题（节流 + 整段重渲染）；设计 `Last-Event-ID` 断点续传；处理代理层缓冲导致的「假流式」（`X-Accel-Buffering: no`）；中断生成、多轮上下文管理
- **R 结果**：用户 300ms 内看到首字输出，感知响应速度提升一个量级

## 手撕代码

### 1. 实现 Promise 并发调度器

要求：限制最大并发数量，任务有序执行，任务失败不中断，收集所有结果。

```js
class Scheduler {
  constructor(limit) {
    this.limit = limit // 最大并发数
    this.queue = [] // 等待执行的任务
    this.active = 0 // 当前正在执行的任务数
  }

  add(task) {
    // 返回 Promise，外部可以拿到最终结果
    return new Promise((resolve, reject) => {
      this.queue.push({ task, resolve, reject })
      this._next()
    })
  }

  _next() {
    // 只要还有空位且队列非空，就取任务执行
    while (this.active < this.limit && this.queue.length) {
      const { task, resolve, reject } = this.queue.shift()
      this.active++
      Promise.resolve()
        .then(task)
        .then(resolve, reject)
        .finally(() => {
          this.active--
          this._next() // 空出位置，启动下一个任务
        })
    }
  }
}

// 任务有序执行 + 失败不中断 + 收集所有结果
async function runTasks(tasks, limit) {
  const scheduler = new Scheduler(limit)
  // 每个任务无论成败都包装成 fulfilled，Promise.all 不会被中断
  const wrapped = tasks.map((task) =>
    scheduler
      .add(task)
      .then((value) => ({ status: 'fulfilled', value }))
      .catch((reason) => ({ status: 'rejected', reason })),
  )
  // map 保序，结果与提交顺序一一对应
  return Promise.all(wrapped)
}

// 测试
const task =
  (name, delay, fail = false) =>
  () =>
    new Promise((resolve, reject) =>
      setTimeout(() => (fail ? reject(new Error(name)) : resolve(name)), delay),
    )

runTasks(
  [task('A', 1000), task('B', 500), task('C', 800, true), task('D', 300)],
  2,
).then(console.log)
// 约按提交顺序输出四个结果对象，C 为 rejected
```

**关键点**：

- `while` 循环保证一次补齐所有空位
- `finally` 里 `_next()` 保证任务结束立刻补位
- 失败不中断：单个任务的 reject 被 `catch` 包装，不影响其他任务
- 也可以直接用 `Promise.allSettled` 收集结果，语义等价

### 2. 统计数组元素出现频次，筛选频次 ≥ N 的元素

要求：挂在数组原型上，以 `arr.filterByFrequency(n)` 的方式调用。

```js
Array.prototype.filterByFrequency = function (n) {
  // 统计频次
  const freq = new Map()
  for (const item of this) {
    freq.set(item, (freq.get(item) || 0) + 1)
  }
  // 第二次遍历原数组：频次达标且未收集过才加入结果
  const res = []
  for (const item of this) {
    if (freq.get(item) >= n && !res.includes(item)) {
      res.push(item)
    }
  }
  return res
}

// 测试
console.log(['a', 'b', 'a', 'c', 'a', 'b'].filterByFrequency(2))
// ['a', 'b']

console.log([1, 1, 1, 2, 3, 3, 4].filterByFrequency(3))
// [1]
```

**关键点**：

- **this 指向**：方法挂在原型上时，`arr.filterByFrequency(n)` 的 `this` 指向 `arr`；实现必须用普通 `function`，箭头函数没有自己的 `this`
- 第二次遍历用 `!res.includes(item)` 去重，结果天然保持**元素首次出现的顺序**
- 用 `Map` 而不是普通对象，避免原型链 key（如 `'constructor'`）的干扰，且 key 可以是任意类型
- 时间复杂度 O(n)（`includes` 在结果数组很小的情况下近似常数）
- 变体追问：如果要按频次降序取前 K 个（LeetCode 347 Top K Frequent），统计后排序或用大小为 K 的最小堆
