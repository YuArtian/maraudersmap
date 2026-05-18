#React #Vue #渲染机制 #反应式 #VirtualDOM

# React vs Vue 渲染对比

## TL;DR

**机制层面两者一样：`<Child />` 都是数据描述，父函数都先于子函数 return。范式层面完全不同：React 是「显式 push（你告诉我变了）+ 子树整片重渲染」，Vue 是「自动 pull（Proxy 拦截读写）+ 细粒度按订阅重渲染」。**

- 两者都有「虚拟 DOM 描述对象」（React Element vs Vue VNode），都是先创建再执行
- 反应式模型截然相反：React 靠 `setState` 显式触发，Vue 靠 Proxy 拦截 get/set 自动追踪
- 重渲染粒度：React 默认整棵子树，Vue 默认只重渲染用到该数据的组件
- Vue 编译器做大量静态分析（hoisting、patch flags、block tree），React 直到 React Compiler 才补这课
- Svelte/Solid 走「无虚拟 DOM」路线，根本没有「重新执行组件」这回事

---

## 1. 相同：都是「先做菜谱，再做菜」

两者同属虚拟 DOM 派，架构思路一致：

```text
你写的代码
    │ 编译
    ▼
渲染函数 render()
    │ 执行
    ▼
虚拟 DOM 树（VNode / React Element）
    │ diff
    ▼
真实 DOM
```

### Vue 的 `<Child />` 也是数据

Vue 模板编译后：

```js
// <template><Child /></template> 编译为：
import { createVNode as _createVNode } from 'vue'

function render(_ctx) {
  return _createVNode(Child)        // 注意：Child 传引用，没加括号
}
```

`createVNode(Child)` 返回普通对象（VNode）：

```js
{
  __v_isVNode: true,
  type: Child,          // 函数引用
  props: {...},
  children: ...,
  patchFlag: 0,         // 编译期静态分析的标记
}
```

**和 React Element 概念完全一样。**

### 父组件 render 也是先 return 再轮到子

Vue 3 的渲染流程（简化）：

```text
渲染 App
   │
   ├─► App.setup() 跑一次（建立反应式订阅）
   ├─► App.render() ─► 返回 VNode tree（描述子组件占位，没实例化子）
   │                                ↓
   └─► 渲染器遍历 VNode tree
            │
            └─► 遇到子组件 VNode → 实例化 → setup() → render()
```

**和 React 一致**：父组件的 render 函数先 return，然后渲染器才去处理 VNode 里的子组件。父和子在调用栈上**不嵌套**，是先后关系。

详见 [[React 渲染机制]] 第 2 节。

---

## 2. 不同：反应式模型截然相反

这是两者最大的分水岭，决定了几乎所有差异。

### React：显式触发（push 模型）

```jsx
function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

- 状态存在 hooks 里（挂在 Fiber 节点上）
- **必须显式调用 setter 才会重渲染**
- React 不知道你「用了哪些状态」，只知道「你说要更新」

### Vue：自动追踪（pull 模型）

```vue
<script setup>
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

- `ref(0)` 返回 Proxy 对象
- 模板里**读** `count.value` 时，Vue 偷偷记下：「这个 effect 用到了 count」
- 当 `count.value = 1` 时，Vue 自动通知所有用到 count 的 effect 重跑
- **没有 setState**，赋值即更新

### Vue 反应式系统的极简实现

```js
const targetMap = new WeakMap()       // target → key → effects
let activeEffect = null

function reactive(obj) {
  return new Proxy(obj, {
    get(target, key) {
      track(target, key)              // 读：记下 activeEffect 依赖了这个 key
      return target[key]
    },
    set(target, key, value) {
      target[key] = value
      trigger(target, key)            // 写：通知所有依赖此 key 的 effect 重跑
    }
  })
}

function effect(fn) {
  activeEffect = fn
  fn()                                 // 跑一次，期间 track 会记录依赖
  activeEffect = null
}
```

组件的 render 函数本身就是一个 effect。**你用了什么就自动订阅什么。**

---

## 3. 最大差异：重渲染范围

这是体感最不同的一点。

### React：父变 → 子默认跟着变

```jsx
function Parent() {
  const [count, setCount] = useState(0)
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveChild />   {/* 即使没用到 count，也会重渲染 */}
    </>
  )
}
```

点按钮 → Parent 重渲染 → ExpensiveChild 也跟着重渲染。**除非用 `React.memo` 包一下。**

### Vue：只有用到变化数据的组件重渲染

```vue
<script setup>
import { ref } from 'vue'
import ExpensiveChild from './ExpensiveChild.vue'
const count = ref(0)
</script>

<template>
  <button @click="count++">{{ count }}</button>
  <ExpensiveChild />   <!-- 不会重渲染 -->
</template>
```

点按钮 → 只有 Parent 自己 render 重跑（它用到了 count）→ ExpensiveChild **完全不重渲染**。

**Vue 默认就有 React.memo 的效果**，不需要手动包。

### 但要注意 props

```vue
<ExpensiveChild :value="count" />
```

count 变化时 ExpensiveChild 会重渲染——**不是因为父变了，而是它自己的 props 是反应式依赖**。

---

## 4. 并排对比表

| 维度 | React | Vue 3 |
|------|-------|-------|
| 描述层（菜谱） | React Element | VNode |
| `<Child />` 立即调 Child？ | 否 | 否 |
| 父 return 先于子执行？ | 是 | 是 |
| 状态存哪 | Fiber 节点上的 hooks 链 | Proxy 对象 + 全局依赖图 |
| 触发更新方式 | 显式 `setState` | 直接赋值 `x.value = ...` |
| 依赖追踪 | 没有，靠开发者声明（hook deps、memo） | 自动（Proxy 拦截 get） |
| 重渲染默认范围 | **整棵子树** | **细粒度，只有用到该数据的组件** |
| 优化手段 | `memo` / `useMemo` / `useCallback` 手动 | 默认就细，**很少需要手动优化** |
| 编译期优化 | 几乎没有（JSX 太灵活）；React Compiler 在补 | 静态提升、Patch Flags、Block Tree |
| 重新执行整个组件函数 | 是，每次重渲染都重跑 | render 重跑，但 setup 只跑一次 |
| Server Component | 一等公民（App Router） | Nuxt 有类似概念，但反应式不跑到服务端 |
| 心智模型 | 数据变了我告诉你，你来重算 | 你只管描述依赖关系，我自动算 |

---

## 5. Vue 编译期魔法（React 没有的）

Vue 模板语法受限（只能写表达式不能写任意 JS），换来编译器可以**静态分析模板**做大量优化。

### 静态提升（Hoisting）

```vue
<template>
  <div>
    <p>这是静态文本</p>          <!-- 永远不变 -->
    <span>{{ count }}</span>
  </div>
</template>
```

编译后：

```js
const _hoisted_1 = createVNode('p', null, '这是静态文本')  // 模块顶层创建一次

function render() {
  return createVNode('div', null, [
    _hoisted_1,                                            // 直接复用
    createVNode('span', null, ctx.count)
  ])
}
```

每次重渲染时静态 VNode **完全不重新创建**，直接复用引用。**React 做不到**（因为 JSX 表达力太强，分析不出来）。

### Patch Flags

模板里哪些地方可能变、变的是什么类型，编译器都标好：

```js
createVNode('div', { class: dynamicClass }, text, 9 /* PROPS | TEXT */)
//                                                ^
//                                                只比较 class 和 text，跳过其他
```

diff 时只看标记位提示的部分，不像 React 要全方位 diff。

### Block Tree

把动态节点扁平化到一个数组，diff 时不用递归整棵树，直接遍历动态节点列表。

React 19 的 React Compiler 正在补这些课，但还在早期。详见 [[ReactCompiler]]。

---

## 6. Vue 2 vs Vue 3 的小坑

「细粒度重渲染」是 **Vue 3** 的特性。Vue 2 也基本是这样，但实现机制不同（`Object.defineProperty` 而非 Proxy），有些边界情况追踪不到：

| 操作 | Vue 2 | Vue 3 |
|------|-------|-------|
| 新增对象属性 | 追踪不到，需 `Vue.set` | 自动追踪 |
| 数组下标赋值 `arr[0] = x` | 追踪不到，需 `Vue.set` | 自动追踪 |
| 数组 length 修改 | 追踪不到 | 自动追踪 |
| Map / Set | 不支持 | 支持 |

Vue 3 用 Proxy 之后才真正「完美」自动追踪。

---

## 7. 还有第三类：Svelte / Solid（无虚拟 DOM）

视野放宽一点，还有第三类反应式：

| 框架 | 路线 | 组件函数 |
|------|------|---------|
| React | 虚拟 DOM + 显式 push | **每次重渲染都重跑** |
| Vue | 虚拟 DOM + 自动 pull | render 重跑，setup 只跑一次 |
| **Svelte** | **无虚拟 DOM**，编译期生成精准 DOM 操作代码 | 组件只编译，不运行 |
| **Solid** | **无虚拟 DOM**，组件只执行一次，反应式驱动 DOM 节点 | **只跑一次** |

Solid 长得像 React（JSX + hooks），但底层完全不同：
- `useState` 在 React 里每次重渲染都重新调用
- Solid 的 `createSignal` 全生命周期只调一次，靠 getter/setter 触发细粒度 DOM 更新

**React 是「全函数重跑」路线最激进的代表。**

---

## 8. 怎么选

| 场景 | 推荐 |
|------|------|
| 大型 SPA、生态成熟、企业级 | React（生态最大） |
| 中后台、模板友好、上手快 | Vue（学习曲线最缓） |
| 极致性能、首屏小 | Svelte / Solid |
| 全栈、SSR、React 团队 | Next.js (React) |
| 全栈、SSR、Vue 团队 | Nuxt (Vue) |

性能差异在大多数项目里**根本不是瓶颈**，选择更多看团队熟悉度和生态。

---

## 9. 一句话对照

| 问题 | React 答案 | Vue 答案 |
|------|-----------|---------|
| `<Child />` 是函数调用吗 | 不是 | 不是 |
| 父 return 先于子执行吗 | 是 | 是 |
| 子 setState 影响父吗 | 不影响 | 不影响 |
| 父变化影响所有子吗 | **默认影响（除非 memo）** | **默认不影响（除非订阅）** |
| 怎么知道哪些组件需要重渲染 | 子树整片，加 memo 优化 | 自动按依赖图精准触发 |

---

## 相关文章

- [[React 渲染机制]] — JSX 是数据、render/commit 双阶段、单向数据流
- [[React Fiber]] — React 的可中断渲染实现
- [[ReactCompiler]] — React 19 的编译期优化（补 Vue 的课）

---

## 参考资料

- [Vue 3 Reactivity Deep Dive](https://vuejs.org/guide/extras/reactivity-in-depth.html)
- [Vue 3 Rendering Mechanism](https://vuejs.org/guide/extras/rendering-mechanism.html)
- [React 官方：Render and Commit](https://react.dev/learn/render-and-commit)
- [Solid Reactivity](https://www.solidjs.com/guides/reactivity)
- Vue 源码 `packages/reactivity/src/effect.ts`
