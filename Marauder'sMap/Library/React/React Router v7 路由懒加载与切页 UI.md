#React #ReactRouter #路由懒加载 #Suspense #代码分割

# React Router v7 路由懒加载与切页 UI

## TL;DR

**`React.lazy` + Suspense 用在路由层会导致"切页时整个旧页面消失变白屏"。react-router v7 Data Mode 的官方推荐是 route-level `lazy` 字段：导航期间 router 内部 `await` lazy，旧页面 UI 完整保留，新页面 ready 后无缝切换。pending 状态通过 `useNavigation()` 暴露，由应用代码驱动顶部进度条等反馈，而不是依赖 Suspense fallback。**

- `React.lazy` 触发 suspend 时，最近的 Suspense fallback 会**替换掉**整棵子树 → 旧 UI 闪没了
- React Transition 机制理论上能保留旧 UI，但需要导航被包在 transition 中 + Suspense boundary 已经 mount 过内容
- react-router v7 route-level `lazy: async () => ({ Component })` 是 Data Mode 推荐方式，**完全不走 React Suspense**
- 切页 pending UI 用 `useNavigation().state` 驱动（如 NProgress 顶部条），官方明确说不要依赖 Suspense fallback
- `HydrateFallback` 这个名字在 v7 被复用：client-only SPA 在 route lazy/loader 首次解析期间也叫"hydration"，需要兜底 UI

---

## 1. 切页白屏的根源：React.lazy + Suspense 在路由层的副作用

很常见的写法：

```jsx
// router/index.js
const LoginPage = lazy(() => import("./pages/Login"));
const DataPage  = lazy(() => import("./pages/Data"));

// RootLayout.jsx
<Suspense fallback={<Spinner />}>
  <Outlet />
</Suspense>
```

看起来很自然 —— 每个页面独立打 chunk，加载时给个 loading。但**切页时的体验是这样的**：

```text
用户点 LoginPage 上的按钮 → navigate("/data")
        │
        ▼
React 渲染新组件树，DataPage 还没 ready → throw Promise
        │
        ▼
最近的 Suspense 接管 → 渲染 <Spinner />
        │
        ▼
画面：整个 LoginPage 被抹掉，变成 spinner（白屏 + 转圈）
        │
        ▼
DataPage 加载完 → Suspense 切回 → 显示 DataPage
```

中间那一帧"整页白屏"就是问题。**已经渲染好的 Layout / Sidebar / 顶栏也会一起消失**，因为它们也在这个 Suspense boundary 内。

| 现象 | 原因 |
|---|---|
| 切页时整页白屏 | Suspense fallback 替换了整棵子树 |
| Sidebar 消失 | Sidebar 在 Suspense 的子树内 |
| 看到 spinner 但没必要 | 模块很小、本地网速很快，闪一下反而割裂 |

## 2. React Transition：本可以保留旧 UI 的机制

React 18+ 其实有解决方案 —— `startTransition`。它的核心规则：

> 如果一次状态更新被包在 transition 里，且过程中 suspend 了，**Suspense 不会显示 fallback，而是保留当前已渲染的内容**，直到新内容 ready。

写法：

```jsx
import { startTransition } from "react";

startTransition(() => {
  navigate("/data");  // 这次导航期间，旧 LoginPage UI 会被保留
});
```

react-router v7 内部确实把所有导航包在 `startTransition` 中。**那为什么我还是看到白屏？**

关键细节："保留旧 UI"成立有两个前提：

1. 触发 suspend 的更新**必须在 transition 内**
2. 该 Suspense boundary **必须已经 mount 过内容**（不是第一次 mount）

跨大区域跳转（比如 `/login` 跳 `/dashboard/index`），新树里的某个 Suspense 是第一次出现，那次 suspend 会显示 fallback，不会保留旧 UI。这是 React 行为的"漏点"，不是 bug 是设计。

```text
理论：transition 内 suspend → 保留旧 UI
实际：跨 Layout 切换时，新树里的 Suspense 是新 mount 的 → 仍走 fallback
```

所以 **React.lazy + Suspense 用在路由层做切页 UI，不是稳定可靠的方案**。

## 3. React Router v7 官方推荐：route-level `lazy`

react-router v7 (Data Mode, `createBrowserRouter`) 提供了 router 层的 `lazy` 字段，**完全绕开 React Suspense**：

```js
// 改前：React.lazy
const DataPage = lazy(() => import("./pages/Data"));
{ path: "/data", Component: DataPage }

// 改后：route-level lazy
{
  path: "/data",
  lazy: async () => {
    const m = await import("./pages/Data");
    return { Component: m.default };
  },
}
```

`lazy` 返回值可以包含 `{ Component, loader, action, ErrorBoundary, handle }` 等字段。

**机制对比：**

| 维度 | `React.lazy` | route-level `lazy` |
|---|---|---|
| 谁负责等待 | React Suspense | react-router 内部 `await` |
| 等待期间显示什么 | Suspense fallback（覆盖旧 UI） | **旧页面 UI 完整保留** |
| pending 状态在哪 | 隐式在 Suspense | 暴露在 `useNavigation().state` |
| 导航完成时机 | 新组件 render 时 | **新模块加载完成后才提交导航** |
| URL 变化时机 | 立即变 | 提交导航时才变（用户感知不到中间态） |

router 看到匹配的路由有 `lazy`，会先 `await` 它，期间不切换路由 —— React 树**完全不动**，旧页面继续渲染。等 `Component` 拿到了再提交导航，新组件接管。整个过程对 React 来说就是一次普通的同步 `Component` 切换，没有 suspend。

## 4. `useNavigation` 驱动 pending UI

route-level `lazy` 不走 Suspense，那 loading 反馈怎么做？v7 给了官方 hook：

```jsx
import { useNavigation } from "react-router";

const RootLayout = () => {
  const { state } = useNavigation();
  // state: "idle" | "loading" | "submitting"

  useEffect(() => {
    if (state === "loading") NProgress.start();
    else NProgress.done();
  }, [state]);

  return <Outlet />;
};
```

| state | 含义 |
|---|---|
| `idle` | 没有进行中的导航 |
| `loading` | 正在加载下一个路由（lazy / loader） |
| `submitting` | 正在提交（action） |

**官方明确：**
> Don't use Suspense fallback for navigation pending state. Use `useNavigation` instead.

为什么？因为 Suspense fallback 是"内容级"loading（替换内容区），而切页是"全局级"事件（顶部进度条更合适），把这两个 concern 分开更清晰。

## 5. HydrateFallback：为什么 client-only SPA 也有"hydration"

切到 route-level lazy 后，console 会出现一条警告：

```text
No `HydrateFallback` element provided to render during initial hydration
```

很多人困惑：**我没用 SSR，哪来的 hydration？**

react-router v7 把 "hydration" 这个词**扩展了语义**：

| 场景 | 传统 hydration | v7 "hydration" |
|---|---|---|
| SSR | server 已吐 HTML，client React 接管激活 | ✅ 是 |
| Client-only + 根路由有 `lazy` / `loader` | / | ✅ **被叫做 hydration** |

v7 的定义大致是：

> "Hydration" = router 从『未就绪』到『已就绪、可渲染』的过程

为什么这么命名？v7 把 framework mode（Remix-style SSR）和 data mode（client-only）的 API 统一了。两种场景的"激活动作"虽然不同，但**"路由初始化期间需要兜底 UI"这个需求是一样的**，于是复用了 `HydrateFallback` 这个名字。

社区也吐槽过命名误导。但官方优先 API 一致性 —— 同一份路由配置在两种 mode 下都能跑，名字保持一致。

**实际效果**：你的 client-only 项目首次访问 `/en/login`，router 在 await `lazy()` → 期间 React 树还没 render 任何路由内容 → 警告提示你应该提供 `HydrateFallback`。

简单处理：

```js
{
  Component: RootLayout,
  HydrateFallback: () => null,  // 或者一个全屏 spinner
  children: [...]
}
```

## 6. NProgress 集成实战

完整可用的代码：

```jsx
// RootLayout.jsx
import { useEffect } from "react";
import { Outlet, useNavigation } from "react-router";
import NProgress from "nprogress";

NProgress.configure({ showSpinner: false, trickleSpeed: 200, minimum: 0.1 });

const RootLayout = () => {
  const { state } = useNavigation();
  useEffect(() => {
    if (state === "loading") NProgress.start();
    else NProgress.done();
  }, [state]);
  return <Outlet />;
};
```

```css
/* index.css —— 主题色覆盖 */
#nprogress .bar {
  background: var(--color-accent);
  height: 2px;
}
#nprogress .peg {
  box-shadow:
    0 0 10px var(--color-accent),
    0 0 5px var(--color-accent);
}
```

**坑点：CSS 加载顺序。** 如果 `nprogress.css` 比 `index.css` 后加载，nprogress 的默认蓝色 `#29d` 会覆盖你的主题色 override。详见 [[CSS Import 顺序与样式覆盖]]。

正确做法：在入口 `main.jsx` 里**先** import nprogress.css，**后** import 业务 CSS：

```js
import "nprogress/nprogress.css";   // 先
import "./index.css";                // 后，覆盖默认色
```

## 7. 完整切页效果

迁移到 route-level `lazy` 后，用户体验：

```text
点按钮 navigate("/data")
        │
        ▼
react-router: await lazy() ← 期间 React 树不动，LoginPage 仍然渲染
        │
        ▼ useNavigation.state === "loading"
NProgress.start() → 顶部 2px 进度条出现
        │
        ▼ 模块加载完成
react-router 提交导航，URL 变化，DataPage 接管
        │
        ▼ useNavigation.state === "idle"
NProgress.done() → 进度条走完消失
```

**关键观感：**
- 没有白屏
- Sidebar / 全局布局完整保留
- 切换无 layout shift
- 顶部进度条提供加载反馈

## 相关文章

- [[React 渲染机制]] - render 阶段、commit 阶段、Suspense 的位置
- [[CSS Import 顺序与样式覆盖]] - nprogress 颜色被覆盖问题的通用规律
- [[软导航与RSC协议]] - Next.js App Router 的导航模型对比

## 参考资料

- react-router v7 docs: https://reactrouter.com/start/modes
- react-router v7 lazy route objects: https://reactrouter.com/start/data/route-object
- react-router v7 `useNavigation`: https://reactrouter.com/api/hooks/useNavigation
- React Suspense + Transition 行为: https://react.dev/reference/react/Suspense
