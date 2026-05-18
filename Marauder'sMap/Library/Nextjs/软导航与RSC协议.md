#Nextjs #软导航 #RSC #ReactServerComponents #HistoryAPI #路由

# Next.js 软导航与 RSC 协议

## TL;DR

**软导航是 SPA 路由的实现策略，底层依赖 History API 同步地址栏，上层依赖框架的客户端路由器拦截点击、拉取 RSC payload、局部重渲染。Next.js App Router 在此之上多做了一步：服务端组件渲染好的 JSX 通过 React Flight 协议序列化传输，客户端不需要 API endpoint 就能拿到数据。**

- 软导航 ≠ History API，软导航 = History API + 数据获取 + 受控渲染
- 硬导航刷新整页（白屏一下），软导航只换需要变的 segment
- `_rsc=xxx` 查询参数告诉 Next 返回 RSC payload 而不是 HTML
- RSC payload 是 React Flight 协议（Server-side rendered React tree 的序列化），不是 JSON
- React Router 十年前就在做软导航，Next App Router = React Router + Remix loader + RSC

---

## 1. 硬导航 vs 软导航

区别在于**浏览器是否真的重新加载整个页面**。

| 维度 | 硬导航（Hard） | 软导航（Soft） |
|------|---------------|---------------|
| 触发方式 | `location.href = ...`、`<a href>`、F5、地址栏回车 | `<Link href>`、`router.push()` |
| 浏览器行为 | 关闭 JS 上下文，重新加载、解析、执行所有 JS | **不卸载页面**，只在内存里更新路由状态 |
| 请求内容 | 完整 HTML 文档 | 只请求 RSC payload（`?_rsc=xxx`） |
| 页面状态 | 全部丢失（state、滚动位置、定时器、WebSocket） | **全部保留**，只换变化的部分 |
| 重新挂载 | 整个 React tree 从零 mount | 只重渲染变化的 segment |
| 视觉效果 | 白屏一下 → 新页面 | 像 SPA 一样平滑切换 |

### 判断方法

打开 DevTools Network 面板：
- 完整的 HTML 文档响应（`Content-Type: text/html`）→ **硬导航**
- `?_rsc=xxx` 请求，响应是 `0:[...]` `1:[...]` 流式格式 → **软导航**

---

## 2. 软导航 = History API + 数据获取 + 受控渲染

容易误以为「软导航 = History API」。实际上 History API 只是软导航的**地址栏部分**。

```text
完整软导航的三件事：

1. History API           更新 URL（pushState/replaceState）
2. 数据/视图获取          fetch 新页面需要的数据（在 Next 里就是 _rsc 请求）
3. 受控渲染              把新数据塞进 React tree，diff 出要变的部分
```

### History API 本身能做什么

```js
history.pushState(state, '', '/zh/leaderboard')
// 历史栈塞一条新记录，地址栏更新，但【浏览器不发请求、不刷新】

history.replaceState(state, '', '/zh/about')
// 替换当前记录，不新增栈条目

window.addEventListener('popstate', (e) => { ... })
// 用户点前进/后退时触发，让 JS 知道「该重新渲染了」
```

光有 History API 你能做到的极限是：

```js
document.querySelector('a').onclick = (e) => {
  e.preventDefault()
  history.pushState({}, '', e.target.href)
  // 地址栏变了，但页面还是原来那个！
}
```

地址栏更新了，但**没有任何内容变化**——所以这不是「导航」，只是「改地址」。

### 完整软导航的伪代码

```js
async function softNavigate(url) {
  // 1. 拉取新页面的 RSC payload
  const payload = await fetch(url + '?_rsc=' + token).then(r => r.body)

  // 2. 解析 Flight 协议，得到新的 React tree
  const newTree = parseFlightStream(payload)

  // 3. 用 React 的并发渲染把新 tree diff 到当前 tree
  React.startTransition(() => {
    routerStore.setTree(newTree)
  })

  // 4. 最后更新地址栏
  history.pushState({}, '', url)
}

// 监听 popstate，前进后退也走同样逻辑
window.addEventListener('popstate', () => softNavigate(location.href))
```

---

## 3. RSC Payload 是什么

访问 `http://localhost:3000/zh/leaderboard?bench=all&_rsc=1yitt`，响应不是 HTML 也不是 JSON，而是这样：

```text
2:"$Sreact.fragment"
4:I["[project]/node_modules/.../segment-explorer-node.js", [...chunks], "SegmentViewNode"]
f:I["[project]/.../boundary-components.js", [...], "OutletBoundary"]
11:"$Sreact.suspense"
...
6:{"name":"LeaderboardPage","key":null,"env":"Server","props":{"params":"$@7","searchParams":"$@8"}}
0:{"b":"development","f":[["children",["locale","zh","d"],"children","leaderboard","children",...]]}
```

这叫 **React Flight 协议**，是 React Server Components 的传输格式。

### 协议格式

每行都是 `<hex_id>:<payload>`，根据**首字母前缀**有不同含义：

| 前缀 | 含义 |
|------|------|
| `I[...]` | **客户端组件引用**（Import）— 指向某个 chunk 里的 export，浏览器要 lazy load |
| `D{...}` | **Debug info**（开发模式才有，生产没有） |
| `J{...}` | **任务/Promise 元数据**（异步边界，比如 `await`） |
| `$<id>` | 引用前面定义过的某个 ID（去重 + 解决环引用） |
| `$L<id>` | **Lazy 引用**（指向尚未到达的 chunk，组件会 Suspense） |
| `$@<id>` | **Promise 引用**（要 await，后面会再 push 一行 resolve 它） |
| `$S<name>` | React 内置 Symbol（如 `$Sreact.fragment`、`$Sreact.suspense`） |
| `$E<code>` | 服务端 eval 的 JS 字符串（很少见） |

### 解读示例

```text
2:"$Sreact.fragment"
  └─ ID=2 的值 = React.Fragment

4:I["...segment-explorer-node.js", [...chunks], "SegmentViewNode"]
  └─ ID=4 = 客户端组件 SegmentViewNode，浏览器从这几个 chunk 加载

6:{"name":"LeaderboardPage", "env":"Server", "props":{...}}
  └─ ID=6 是 LeaderboardPage 这个服务端组件的描述
```

### 从 RSC payload 看渲染时间线

把 `D{"time": x}` 这些 debug 行按时间排序，能看到服务端 render 的完整过程：

```text
t=0.5ms     LeaderboardPage 服务端组件开始执行
t=0.86ms    await get requestLocale (next-intl 取 locale)
t=12.4ms    ← requestLocale resolve = "zh"
t=12.6ms    开始 generateMetadata (并行)
t=14.4ms    await params resolve = {bench: "all"}
t=23.0ms    await auth() resolve = {user: {...}}
t=27.8ms    getLeaderboard() 开始（fetch DB）
t=81.3ms    ← getLeaderboard() resolve（50ms 查 DB）
t=95ms      PageHeader + LeaderboardTable JSX 序列化完成
```

---

## 4. Server Component 的"无 API"特性

普通做法（要写 API endpoint）：

```ts
// 1. /api/leaderboard 路由
export async function GET() {
  const data = await db.query(...)
  return Response.json(data)
}

// 2. 客户端组件
const data = await fetch('/api/leaderboard').then(r => r.json())
```

RSC 做法（不用 API endpoint）：

```tsx
// page.tsx (Server Component)
export default async function LeaderboardPage() {
  const data = await getLeaderboard()        // 服务端直接查 DB
  return <LeaderboardTable initialData={data} />
}
```

`LeaderboardTable` 是 `"use client"` 组件，所以服务端把：
1. **组件引用**（chunk 路径）写成 `I[...]` 行
2. **props（包含数据）** 直接序列化到 stream 里

浏览器拿到后，下载 chunk → 用序列化的 props 实例化 LeaderboardTable，**不用再发 API 请求拿数据**。

DB 凭据、API 密钥都留在服务端，客户端 bundle 完全不知道数据来源。

---

## 5. 客户端路由的演进史

软导航不是 Next.js 发明的。React 社区做了十年。

| 阶段 | 代表 | 数据怎么来 | 路由定义 |
|------|------|-----------|---------|
| 古早 SPA（2014~） | React Router v3 | `componentDidMount` 里 `fetch` | JSX `<Route>` 配置 |
| Hooks 时代（2019~） | React Router v5 + SWR/React Query | `useEffect` / `useQuery` | JSX `<Route>` 配置 |
| Loader 时代（2021~） | **Remix** / React Router v6.4+ | **`loader` 函数，路由切换前预取** | 文件系统路由 + `loader` |
| RSC 时代（2023~） | **Next.js App Router** | **服务端组件直接 `await fetch`，走 Flight 协议** | 文件系统路由 + Server Component |

**Remix 是关键中转站**——Remix 的作者就是 React Router 的作者，他们把「路由切换时 loader 预取数据」这个理念带进 React Router 6.4，后来又被 Next.js 用 RSC 重新实现了一遍。

### 三者代码对比

**React Router v5（客户端 fetch）：**
```jsx
function Leaderboard() {
  const [data, setData] = useState(null)
  useEffect(() => {
    fetch('/api/leaderboard').then(r => r.json()).then(setData)
  }, [])
  if (!data) return <Loading />
  return <Table data={data} />
}
// 软导航流程：pushState → 渲染 Leaderboard → useEffect 触发 → fetch → setState 重渲染
// 用户感受：先白一下表格，loading 一会儿，才出数据
```

**React Router v6.4 / Remix（loader）：**
```jsx
const route = {
  path: '/leaderboard',
  loader: async () => fetch('/api/leaderboard').then(r => r.json()),
  Component: Leaderboard,
}
function Leaderboard() {
  const data = useLoaderData()
  return <Table data={data} />
}
// 软导航流程：pushState 之前先 await loader() → 数据齐了再切换 → 一次性渲染
```

**Next.js App Router：**
```jsx
// page.tsx — Server Component
export default async function LeaderboardPage() {
  const data = await getLeaderboard()
  return <LeaderboardTable initialData={data} />
}
// 软导航流程：fetch ?_rsc=xxx → 服务端跑 LeaderboardPage → 序列化的 JSX + props 流回
// 区别：loader 函数不见了，数据获取直接写在组件里；序列化协议从 JSON 变成 Flight
```

---

## 6. Prefetch：软导航的隐藏优化

Next.js 的 `<Link>` 还会在你**鼠标 hover 或链接进入视口时**，提前发一个 `_rsc` 请求把目标页面拉下来缓存。

```text
用户 hover 链接   ──► 后台静默 fetch ?_rsc=xxx
                         │
                         └─► 存进 Router Cache
                                       │
用户点击链接 ────────────────────► 直接用缓存，肉眼几乎瞬间切换
```

所以一个 `_rsc=xxx` 请求**未必是用户已经点击**，也可能只是 hover/prefetch。

---

## 7. 为什么要软导航？

| 好处 | 解释 |
|------|------|
| 保留状态 | 表单输入、模态框、滚动位置不丢 |
| 更快 | 不重下 JS chunk，不重 hydrate，不重连 WebSocket |
| 更小网络开销 | 只传变化的 segment，不传整页 HTML |
| 更平滑体验 | 没有白屏闪烁，像原生 app |

代价：**首次进入页面（或刷新）还是硬导航**——SPA 的本质：第一次必须完整加载，之后才能享受软导航。

---

## 8. RSC Payload 中的乱码问题

curl 看 RSC payload 会看到一堆乱码：

```text
"æ¦œå• | Health Memory Arena"
```

原文是 `"榜单 | Health Memory Arena"`。原因：UTF-8 字节被当 latin-1 显示。浏览器接收时按 UTF-8 解码就正常。

---

## 9. 一句话总结

> **软导航是 SPA 路由的实现策略：拦截 `<Link>` 点击 → 阻止浏览器硬导航 → 拉新数据 → 改 React 状态 → 局部重渲染 → 更新地址栏。本质和 React Router 十年前做的事一样，只是 Next.js App Router 把数据获取搬到了服务端，用 React Flight 协议把序列化的 JSX + props 流式传回客户端，不需要单独写 API endpoint。**

---

## 相关文章

- [[App Router 状态管理]] — 软导航如何触发路由 state 变化、Router Cache 机制
- [[React 渲染机制]] — Link 触发的重渲染如何在 React 内部传播
- [[React vs Vue 渲染对比]] — Vue 也有类似机制（Nuxt 的 nuxt-link）

---

## 参考资料

- [Next.js Docs: Linking and Navigating](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating)
- [React Server Components RFC](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)
- [React Flight Protocol](https://github.com/facebook/react/blob/main/packages/react-server/src/ReactFlightServer.js)
- [Remix Loaders](https://remix.run/docs/en/main/route/loader)
- [MDN: History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API)
