#Nextjs #AppRouter #状态管理 #Context #useReducer #PartialRendering

# Next.js App Router 状态管理

## TL;DR

**Next.js App Router 客户端运行时用 useReducer 管理路由 state，通过 3 个 Context 按变化频率分发给组件树。所有导航（Link 点击、router.push、popstate）都是 dispatch 一个 action。LayoutRouter（Next 自动注入）夹在每对用户 Layout 之间，负责切换子 segment 和缓存管理——这就是「切换页面但 Layout 不重渲染」的物理基础。**

- 客户端核心：一个 reducer state（tree / cache / canonicalUrl / prefetch / focus）+ 7 种 action
- 3 个 Context 不是容量限制，是按「变化频率」做的性能拆分（一个 Context 能被无数组件订阅）
- 选 useReducer + Context 是被三个约束逼出来的：框架不能强加依赖、路由是动作驱动状态机、需要从树任意位置订阅
- LayoutRouter 与用户 Layout 交替嵌套，URL 段数决定嵌套深度
- 切换同层 segment 时，外层 Layout 不重渲染——Layout 不订阅路由 Context + Router Cache 引用稳定 + Server Component 输出本就静态

---

## 1. 客户端核心：一个 useReducer

App Router 客户端运行时的**核心数据结构**是一个 reducer state：

```ts
type AppRouterState = {
  tree: FlightRouterState         // 当前完整的路由树
  cache: CacheNode                 // Router Cache（已访问 segment 的 RSC payload）
  prefetchCache: Map<string, ...>  // 预取过的 segment
  pushRef: { pendingPush, mpaNavigation, preserveCustomHistoryState }
  focusAndScrollRef: { apply, hashFragment, segmentPaths }
  canonicalUrl: string             // 当前地址栏 URL
  nextUrl: string | null
}
```

**整个客户端运行时的「真相之源」就这一个 state。** 怎么改？dispatch action：

```ts
type Action =
  | { type: 'NAVIGATE',  url, navigateType, shouldScroll, ... }
  | { type: 'SERVER_PATCH', flightData, previousTree, overrideCanonicalUrl }
  | { type: 'RESTORE', url, tree }            // popstate
  | { type: 'REFRESH', cache, mutable }        // router.refresh()
  | { type: 'FAST_REFRESH', ... }              // 开发模式 HMR
  | { type: 'PREFETCH', url, kind }            // <Link prefetch>
  | { type: 'HMR_REFRESH', ... }

function reducer(state, action) {
  switch (action.type) {
    case 'NAVIGATE':      return navigateReducer(state, action)
    case 'SERVER_PATCH':  return serverPatchReducer(state, action)
    case 'RESTORE':       return restoreReducer(state, action)
    // ...
  }
}
```

**每一种导航行为对应一个 action**，和 Redux 思路一样。

---

## 2. 为什么是 useReducer + Context（而不是别的）？

Next.js 选这个组合是被**三个硬约束**逼出来的：

### 约束 1：框架不能强加依赖

如果 Next 内部用 Redux/Zustand：
- 每个 Next 项目的 bundle 都被迫包含
- 版本冲突
- 生态绑架

所以**框架内部只能用 React 内置工具**：`useState`、`useReducer`、`useContext`、`useSyncExternalStore`、`useEffect`、`useTransition`。

### 约束 2：路由是「动作驱动的状态机」

路由不是「一个变量在变」，而是**「一组离散动作触发离散状态转移」**：

```text
NAVIGATE          切换 segment、拉 RSC、更新 URL、滚动复位
RESTORE           前进后退时恢复历史 state
REFRESH           清空当前 segment cache 重新拉
SERVER_PATCH      服务端推送的 RSC 更新（revalidate 触发）
PREFETCH          hover 时预取，不切换页面
FAST_REFRESH      开发模式 HMR
```

对比两种写法：

```js
// useState 版本：5 个 setter 调用 + 手动同步
function navigate(href) {
  setTree(newTree)
  setCache(newCache)
  setUrl(href)
  setPrefetch(...)
  setFocus(...)
  // 失败回滚要存 5 个旧值，测试要 mock 5 个 setState
}

// useReducer 版本：一行搞定
function navigate(href) {
  dispatch({ type: 'NAVIGATE', href })
  // 失败回滚 = dispatch 一个 ROLLBACK action
  // 测试只需要测 reducer(prevState, action) === expectedState
}
```

reducer 的核心优势：

| 优势 | 说明 |
|------|------|
| 多字段联动一次完成 | action 内 return 整个新 state，不会出现「切到一半」中间态 |
| 状态转移可预测 | `reducer(state, action) → newState` 是纯函数 |
| 易测试 | 不需要 mount 组件，直接调 reducer 测 |
| 易调试 | 所有变化都通过 action，可 log/时间旅行 |
| 易扩展 | 加新行为 = 加一个 action type 和一个 case |
| Concurrent 安全 | reducer 是纯函数，重放安全 |

### 约束 3：状态要能从组件树任意位置订阅

| 方案 | 能从组件树订阅 | Next 能不能用 |
|------|--------------|--------------|
| 模块级变量 | 否 | 不能 |
| 全局事件总线 | 能但脆弱 | 不能 |
| **React Context** | 能（原生订阅、自动清理） | **能** |
| **useSyncExternalStore + 自建 store** | 能 | **能** |

Context 在 Next 的场景下更合适：
1. **天然的树状作用域**（每层 Layout 都有自己的 LayoutRouterContext）
2. **SSR/hydration 友好**
3. **Concurrent mode 兼容**（不会有 tearing）

代价是粗粒度订阅（value 变了所有订阅者重渲染），通过「拆 Context」缓解。

---

## 3. 3 个 Context：性能拆分而非容量限制

**常见误解**：「3 个 Context 够用吗？用户会写很多组件。」

**澄清**：Context 是 **1 对 N** 的——一个 Context 可以被**无限多个**组件订阅，没有数量上限。

```text
RouterContext (1 个) ────► CompA, CompB, CompC, ..., Comp10000  全部能订阅
```

类比：Context 像广播电台（一个台无数收音机），不是座位（不是「3 个座位坐 3 个人」）。

### 那为什么拆 3 个？性能！

Context 渲染规则：**只要 value 引用变化，所有 `useContext(它)` 的组件都重渲染——无论组件实际有没有用到 value 里的字段。**

假如 Next 只用一个大 Context：

```jsx
const RouterContext = createContext({
  push, replace, back, refresh,     // 路由方法（基本不变）
  tree,                              // 路由树（每次导航都变）
  cache,                             // 缓存（每次拉新 segment 都变）
  canonicalUrl,                      // 当前 URL（每次导航都变）
  focusAndScrollRef,                 // 焦点滚动状态（频繁变）
  prefetchCache,                     // 预取缓存（hover 就变）
})
```

后果：

```jsx
function MyButton() {
  const { push } = useContext(RouterContext)  // 只用 push！
  return <button onClick={() => push('/foo')}>Go</button>
}
```

按钮只用 push，但**任何一个字段变（哪怕 prefetch 缓存动了一下），按钮都重渲染**。如果页面里有 200 个按钮，每次状态变 200 个全部重渲染。

### Next 的拆法：按变化频率

```ts
// 1. 路由方法（push/replace/refresh…）
// 特点：在 router 实例化时就稳定了，几乎永远不变
const AppRouterContext = createContext<AppRouterInstance>(null)

// 2. 当前 segment 在路由树里的位置
// 特点：本层 layout 切 segment 时变，别层切换时不变
const LayoutRouterContext = createContext({ tree, url, ... })

// 3. 全局路由状态
// 特点：任何导航都变
const GlobalLayoutRouterContext = createContext({ tree, changeByServerResponse, ... })
```

| 用户写的代码 | 实际订阅 | 重渲染时机 |
|------|---------|----------|
| `useRouter()` | AppRouterContext | **几乎不重渲染**（router 方法引用稳定） |
| `usePathname()` / `useSearchParams()` | GlobalLayoutRouterContext | 导航时重渲染 |
| Layout 内部 outlet | LayoutRouterContext | 本层 segment 变时重渲染 |

收益：

```jsx
function MyButton() {
  const router = useRouter()   // 订阅 AppRouterContext
  return <button onClick={() => router.push('/foo')}>Go</button>
}
// 现在点别的链接、切换路由、prefetch——按钮一次都不重渲染！
```

这是 React 社区的标准模式，叫 **Context Splitting**。React 官方文档专门有一节 [Scaling Up with Reducer and Context](https://react.dev/learn/scaling-up-with-reducer-and-context) 讲这个。

---

## 4. LayoutRouter：Next 偷偷塞的「分流开关」

访问 `/zh/leaderboard` 时**你写的代码**只有 3 个文件：

```text
app/
├── layout.tsx                       用户写的 RootLayout
└── [locale]/
    ├── layout.tsx                   用户写的 LocaleLayout
    └── leaderboard/
        └── page.tsx                 用户写的 LeaderboardPage
```

**但实际跑的组件树不止 3 层**——Next 在每对「用户组件」之间**自动塞了一个 LayoutRouter**：

| 名字 | 谁写的 | 干什么 |
|------|--------|--------|
| **Layout** | **用户** | UI 结构、导航栏、Provider、字体 |
| **LayoutRouter** | **Next 自动注入** | 决定这一层渲染哪个子 segment、管这一层的 cache |

**它俩总是成对出现，交替嵌套。**

### 完整组件树

```text
[Next 注入] AppRouterContext.Provider value={router}
[Next 注入]   GlobalLayoutRouterContext.Provider value={state}
[Next 注入]
[Next 注入]     LayoutRouter segment="root"
[用户]            RootLayout
[用户]              <html><body>{children}</body></html>
[Next 注入]
[Next 注入]           LayoutRouter segment="[locale]"
[用户]                  LocaleLayout
[用户]                    <NavBar /> {children}
[Next 注入]
[Next 注入]                 LayoutRouter segment="leaderboard"
[用户]                        LeaderboardPage
[Next 注入]                 </LayoutRouter>
[用户]
[用户]                  </LocaleLayout>
[Next 注入]           </LayoutRouter>
[用户]
[用户]       </RootLayout>
[Next 注入]     </LayoutRouter>
[Next 注入]
[Next 注入]   </GlobalLayoutRouterContext.Provider>
[Next 注入] </AppRouterContext.Provider>
```

**规律：URL 每一段对应一对「LayoutRouter + Layout/Page」三明治。** Layout 是面包（UI），LayoutRouter 是夹心（路由逻辑）。

### LayoutRouter 干的事

```tsx
function LayoutRouter({ segmentPath }) {
  const { tree, cache } = useContext(GlobalLayoutRouterContext)
  const childSegment = findChildSegment(tree, segmentPath)
  const childNode = cache.parallelRoutes.get(childSegment)

  return (
    <LayoutRouterContext.Provider value={{ tree: childSegment.tree, ... }}>
      {childNode.rsc}
    </LayoutRouterContext.Provider>
  )
}
```

| 职责 | 解释 |
|------|------|
| 路由分发 | 看 URL，决定本层渲染哪个子 segment |
| Cache 管理 | 本层的 RSC payload 缓存 |
| Suspense 边界 | 子 segment 还在拉的时候显示 `loading.tsx` |
| 错误边界 | 子 segment 报错时显示 `error.tsx` |
| 局部刷新隔离 | **切换本层 segment 时，只有这一层重渲染**，外层 Layout 不动 |

---

## 5. Router Cache：state 里的小型数据库

`state.cache`（CacheNode）是 App Router 的**客户端缓存层**，结构是一棵树，对应路由树：

```ts
type CacheNode = {
  lazyData: Promise<FlightData> | null        // 正在拉的 RSC payload
  rsc: ReactNode | null                        // 已渲染好的 server component 输出
  prefetchRsc: ReactNode | null
  parallelRoutes: Map<string, Map<string, CacheNode>>  // 子 segment
  loading: LoadingModuleData | null
  head: ReactNode | null
}
```

**作用：**
- 点 Link 进 `/leaderboard`，RSC payload 拉回来，存进对应 CacheNode
- 再点别处再点回 `/leaderboard`，**直接从 cache 拿**
- `router.refresh()` = 清空当前节点 cache 再重新拉
- `revalidatePath()` = 服务端通知客户端某些 cache 失效

这就是为什么 Next.js 页面切换感觉特别快。

---

## 6. Partial Rendering：切换页面 Layout 不重渲染

切换 `/zh/leaderboard` → `/zh/methodology` 时：

| 组件 | 是否重渲染 |
|------|----------|
| RootLayout | 否 |
| LocaleLayout | **否** |
| LeaderboardPage | 卸载（unmount） |
| MethodologyPage | 挂载（mount） |
| `[locale]` 层 LayoutRouter | 是（要换 child） |
| `root` 层 LayoutRouter | 可能 re-run 但 diff 后 skip |

**关键事实：用户写的 Layout 不重渲染，连里面的 useState 都保留。**

### 为什么不重渲染？三个机制叠加

#### 机制 1：Layout 没订阅路由 Context

```tsx
export default function LocaleLayout({ children }) {
  // 没有 useContext(GlobalLayoutRouterContext)
  // 没有 useRouter()
  return <NextIntlProvider><NavBar />{children}</NextIntlProvider>
}
```

**只有订阅了 Context 的组件才会因 value 变化而重渲染。** 你的 Layout 通常只是个壳，没 useContext 那个 Context。订阅 Context 的是 LayoutRouter（Next 自己的），不是你的 Layout。

#### 机制 2：React 的引用相等优化

> **如果父组件传给子组件的 children prop 是「同一个 React 元素引用」，React 会跳过子组件的重新执行。**

```tsx
function LayoutRouter({ parallelRouterKey }) {
  const { tree, cache } = useContext(GlobalLayoutRouterContext)
  const childNode = cache.parallelRoutes.get(parallelRouterKey).get(segment)
  // childNode.rsc 是【缓存的 React 元素】，引用稳定
  return (
    <LayoutRouterContext.Provider value={...}>
      {childNode.rsc}   {/* 同一个引用，没 new */}
    </LayoutRouterContext.Provider>
  )
}
```

切换 leaderboard ↔ methodology 时，`[locale]` 这层 LayoutRouter 重跑，但从 Router Cache 拿到的 `<LocaleLayout>...</LocaleLayout>` 元素引用还是老的。**React diff 看到 children 引用相等 → 跳过 LocaleLayout 整个子树。**

#### 机制 3：Server Component 输出本来就是缓存的

如果 Layout 是 Server Component（App Router 默认）：

- 它的 JSX 在**服务端跑了一次**，结果序列化成 RSC payload
- 客户端拿到 payload 后存进 Router Cache
- **客户端永远不会「执行」这个 Layout 函数**，它只是一段静态数据

切换路由时，服务端只重新渲染**变化的 segment**（methodology page），不会重渲染 layouts。

### 用一张图看清楚

```text
切换 /zh/leaderboard → /zh/methodology

Router Cache 树（客户端内存）

root segment
└─ RootLayout 输出（缓存）              复用，不重渲染
    └─ [locale] segment
        └─ LocaleLayout 输出（缓存）    复用，不重渲染
            └─ [leaderboard ✗→ methodology ✓]
                                        只换这一格
                                        拉新 RSC payload
                                        旧节点 unmount
                                        新节点 mount
```

### 怎么验证

```tsx
// 验证 1：加 console.log
export default function LocaleLayout({ children }) {
  console.log('LocaleLayout rendered')  // 服务端控制台只输出一次
  return <>...</>
}

// 验证 2：用 useState（要变成 client component）
'use client'
export default function LocaleLayout({ children }) {
  const [count, setCount] = useState(0)
  // 点几下让 count=5，然后切换路由 → count 还是 5（state 保留）
}

// 验证 3：React DevTools Profiler
// 录制一次导航，能看到 LocaleLayout 标 "did not render"
```

### 什么情况下 Layout 会重渲染？

| 场景 | 原因 |
|------|------|
| URL 切到**另一个完全独立的 layout segment** | 比如 `/(marketing)` 和 `/(app)` 跨组切换 |
| Layout 接收了变化的 params | `[locale]/layout.tsx` 读 `params.locale`，`/zh` → `/en` 时重渲染 |
| Layout 内部 useState 自己 setState | 自己的 state 变化 |
| `router.refresh()` | 主动告诉 Next 重拉 |
| `revalidatePath('/[locale]', 'layout')` | 服务端主动失效这层 cache |
| Layout 用了 `cookies()` / `headers()` | 标记为动态，每次请求都跑 |

---

## 7. Link 触发重渲染的完整链路

```text
用户点击 <Link href="/zh/leaderboard">
   │
   ▼
1. Link 的 onClick 拦截：e.preventDefault()  阻止浏览器默认硬导航
   │
   ▼
2. 调用 router.push('/zh/leaderboard')
   │
   ▼
3. 路由器内部：
   a. fetch('/zh/leaderboard?_rsc=xxx')
   b. 解析 Flight payload，得到新 React tree
   c. history.pushState({}, '', '/zh/leaderboard')
   d. dispatch({ type: 'NAVIGATE', tree: newTree })   关键！改 state
   │
   ▼
4. 路由器的 useReducer state 变了
   │
   ▼
5. Context.Provider 的 value 引用变了
   │
   ▼
6. 订阅这个 Context 的组件触发重渲染
   │
   ▼
7. React 用新 tree diff 旧 tree，卸载旧 page、挂载新 page
   │
   ▼
8. 新页面显示
```

**`<Link>` 本质上是一个会 dispatch 路由 state 的按钮。** 通过 `preventDefault` 截断浏览器硬导航，转而调用框架内部 router，router 改自己的 React state——改 state 就是 React 重渲染的标准触发条件。

详见 [[React 渲染机制]] 第 4 节。

---

## 8. 服务端那层（简略）

服务端没有「全局状态」概念，每个请求是独立 scope。Next 提供几个机制：

| 机制 | 作用 | 例子 |
|------|------|------|
| Request Memoization | 同请求内相同 fetch 自动去重 | 两个组件 `fetch('/api/x')` 只发一次 |
| Data Cache | 跨请求的 fetch 缓存 | `fetch(url, { next: { revalidate: 60 }})` |
| Full Route Cache | 整路由的 HTML+RSC 缓存 | 静态路由编译期生成 |
| Router Cache（客户端） | 上面讲过的 CacheNode | 浏览器内存 |
| `cache()` / `unstable_cache()` | 函数级缓存装饰器 | `const getUser = cache(async (id) => ...)` |
| Server Actions | `revalidatePath` / `revalidateTag` 主动失效 | 表单提交后让相关页面失效 |
| cookies() / headers() | 请求级状态（标记为动态） | 调用了就变 dynamic rendering |

服务端是「多层缓存系统」，和客户端 Router state 是**两套独立系统**，通过 RSC payload 衔接。

---

## 9. 想看源码？入口在这

```text
node_modules/next/dist/client/components/
├── app-router.js                                顶层 AppRouter 实现
├── router-reducer/
│   ├── router-reducer.js                        Reducer 主入口
│   └── reducers/
│       ├── navigate-reducer.js                  NAVIGATE action 处理
│       ├── server-patch-reducer.js
│       ├── restore-reducer.js
│       └── ...
└── ...
```

`grep -r "case 'NAVIGATE'"` 就能跳到核心逻辑。

---

## 10. 一句话总结

> **客户端：一个 useReducer 管路由 state + Router Cache，通过 3 个 Context 按变化频率分发给组件树，所有导航都是 dispatch 一个 action。LayoutRouter 与用户 Layout 交替嵌套，URL 段数决定嵌套深度。Layout 不订阅路由 Context + Router Cache 引用稳定 + Server Component 输出本就静态——三个机制叠加实现了「切换页面 Layout 不重渲染」的 Partial Rendering。**

---

## 相关文章

- [[软导航与RSC协议]] — RSC payload 是怎么产生和传输的
- [[React 渲染机制]] — useReducer/Context 触发重渲染的底层机制
- [[评测榜单系统设计]] — App Router 实战项目（hma-web）

---

## 参考资料

- [Next.js Docs: App Router](https://nextjs.org/docs/app)
- [Next.js Docs: Caching](https://nextjs.org/docs/app/building-your-application/caching)
- [React Docs: Scaling Up with Reducer and Context](https://react.dev/learn/scaling-up-with-reducer-and-context)
- Next.js 源码 `packages/next/src/client/components/router-reducer/`
