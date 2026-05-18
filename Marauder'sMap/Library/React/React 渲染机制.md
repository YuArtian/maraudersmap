#React #渲染机制 #JSX #Fiber #VirtualDOM

# React 渲染机制：从 JSX 到 DOM

## TL;DR

**React 渲染分两阶段：render（父先子后，计算虚拟 DOM）+ commit（子先父后，操作真实 DOM）。`<Child />` 不调用 Child，它只是一个描述对象，由 React 在更外层的循环里决定何时调用。**

- JSX 编译后是 `React.createElement(Child, ...)`，返回普通对象（虚拟 DOM 节点），**不执行** Child 函数
- 父组件函数 return 之后，才轮到 React 调用子组件函数——它们在调用栈上是「先后」不是「嵌套」
- 触发重渲染只有 6 种方式（setState / useReducer / Context / props / 外部 store / key 变化），全部围绕「state 变化」
- React 渲染严格自顶向下传播，子组件状态变化不会向上影响父组件
- mount/updated 这类「事后型」生命周期是**子先父后**，因为父的 DOM 必须等子装好才算 mount 完成

---

## 1. JSX 是数据描述，不是函数调用

最关键的认知：`<Child />` 不会执行 Child 函数。它只是创建一个对象。

JSX 是语法糖，编译器把它转成 `React.createElement` 调用：

```js
// 你写的
function Parent() {
  return <Child name="hi" />
}

// 编译后
function Parent() {
  return React.createElement(Child, { name: "hi" })
  //                          ^^^^^
  //                          传函数引用，注意没加括号！
}
```

`createElement` 返回的是个普通对象（React Element）：

```js
{
  $$typeof: Symbol.for('react.element'),
  type: Child,         // ← Child 函数引用存这里
  props: { name: "hi" },
  key: null,
  ref: null
}
```

| 写法 | 含义 |
|------|------|
| `Child(props)` | **立即执行** Child 函数 |
| `<Child />` 或 `createElement(Child, ...)` | **保存** Child 函数引用到对象里，等以后再用 |

类比：JSX 像建筑图纸，写「这里要有一面墙」不等于真砌墙。施工队（React）拿着图纸按顺序施工。

---

## 2. 父函数先 return，子函数后执行

理解 React 渲染最重要的心智模型：

```text
function Parent() {
  console.log('1. Parent 开始')
  const result = <Child />             // 创建对象，没调 Child
  console.log('2. Parent 结束')
  return result
}

function Child() {
  console.log('3. Child 开始')         // 在 Parent return 之后
  return <div>hi</div>
}

输出：
  1. Parent 开始
  2. Parent 结束
  3. Child 开始
```

**为什么？** React 内部大致这样工作：

```text
ReactDOM.render(<App />, root)
   │
   ├─► React 拿到 <App />，type 是 App 函数
   ├─► 调用 App() ────► App 返回 { type: Parent, ... }
   ├─► 调用 Parent() ─► Parent 返回 { type: Child, ... }   ← Parent 出栈！
   ├─► 调用 Child() ──► Child 返回 { type: 'div', ... }
   └─► 创建真实 DOM
```

**Parent 函数早已 return、从调用栈消失**，然后 React 才拿着 Parent 返回的对象去找 Child 函数。两者在调用栈上是先后关系，**不是嵌套**。

DevTools Sources 给 Parent 和 Child 都打断点验证：调用栈分别是 `React → Parent` 和 `React → Child`，**没有** `React → Parent → Child`。

---

## 3. 为什么要分两步（描述 vs 执行）？

如果 `<Child />` 立即调用 Child，React 就失去对渲染过程的控制。延迟调用换来这些能力：

| 能力 | 原理 |
|------|------|
| Concurrent 模式可中断 | React 想暂停就暂停，只要还没开始调 Child |
| 优先级调度 | React 决定先调哪个 Child |
| Suspense | React 可决定「先不调 Child，显示 fallback」 |
| memo 跳过 | props 没变就不调 Child |
| 渲染顺序优化 | 批量、延后都可以 |
| Server Component | 某些组件在服务端调，某些在客户端调 |

**核心思想：把「描述要渲染什么」和「什么时候真的渲染」分开。** JSX 负责描述，React 负责执行。

---

## 4. 触发重渲染的 6 种方式

React 只有以下方式能触发组件函数重新执行：

| 方式 | 例子 | 备注 |
|------|------|------|
| useState setter | `setCount(1)` | `Object.is` 相等时 bail out |
| useReducer dispatch | `dispatch({ type: 'inc' })` | reducer 必须返回新引用 |
| 父组件重渲染 | 父变 → 子跟着变 | 除非 `React.memo` |
| Context value 变化 | Provider 的 value 引用变 | 所有 consumer 都重渲染 |
| 外部 store | `useSyncExternalStore` | Redux/Zustand 内部用这个 |
| key 变化 | `<Comp key={x} />` | 实际是重新 mount，不是 re-render |

**不会触发的常见误区：**

| 操作 | 为什么不触发 |
|------|------------|
| 普通变量 `let x = 1; x = 2` | React 不知道 |
| `useRef().current = ...` | ref 是「逃生舱」，专门不触发 |
| `array.push()` / `obj.prop = ...` | 引用没变，`Object.is` 相等 |
| `setState(sameValue)` | bail out 优化 |
| `localStorage.setItem(...)` | 不在 React 管辖范围 |
| 直接操作 DOM | React 会在下次渲染时覆盖你 |

---

## 5. 单向数据流：子组件不影响父组件

React 的渲染是**自顶向下**的，没有反向冒泡：

```text
setState 发生在某组件
   │
   ▼
React 把这个 Fiber 标记 dirty
   │
   ▼
从 dirty 节点向【下】遍历子树（永远不向上）
   │
   ▼
重新调用每个组件函数（除非被 memo 跳过）
   │
   ▼
diff + commit DOM
```

**Parent 函数体在 Child 执行前就已经出栈了，物理上不可能被 Child 影响。** state 是组件私有的，父根本不知道子的 state 存在。

子要影响父，只能通过「Lifting State Up」模式：

```text
父持有 state → 通过 callback prop 下传 → 子调 callback
                                          → 触发父的 setState
                                          → 父重渲染（从父开始向下）
                                          → 子被牵连重渲染
```

数据流向永远是：**父 → 子 props，子 → 父 callback**。从来没有「子直接改父 state」。

### 常见误解澄清

| 误解 | 真相 |
|------|------|
| useContext 是反向影响父 | 不是。Provider 在上面，Consumer 订阅。Consumer 重渲染不影响 Provider |
| Portal 渲染到别处会影响那个 DOM 的父 | 不会。Portal 只改 DOM 层级，React 树层级不变 |
| useImperativeHandle 让父被子改 | 不是。是父主动调子暴露的方法，不是子改父 |

---

## 6. Render 阶段 vs Commit 阶段：方向相反

一次完整更新分两个阶段，方向相反：

```text
┌────── Render 阶段（计算）───────┐
│  Parent.render() ──► VNode tree │   方向：父 → 子
│         │                       │
│         └─► Child.render()      │
└─────────────────────────────────┘
              │
              ▼
┌────── Commit 阶段（DOM + 副作用）─────┐
│  1. 创建 Child 的 DOM                 │
│  2. 插入到 Parent 容器                │   方向：子 → 父
│  3. 触发 Child mounted               │
│  4. 把 Parent DOM 插入上一级          │
│  5. 触发 Parent mounted              │
└──────────────────────────────────────┘
```

### Render 阶段
- **输入**：state + props
- **输出**：虚拟 DOM 树
- **副作用**：禁止（不能 fetch、不能改 DOM、不能 setState）
- **可中断**：可以重试、可以丢弃（concurrent 模式利用这点）

### Commit 阶段
- **输入**：算出的虚拟树
- **输出**：真实 DOM 变化
- **副作用**：专门做副作用（DOM 操作、addEventListener、订阅）
- **不可中断**：一旦开始改 DOM 必须改完，否则界面撕裂

---

## 7. 生命周期触发顺序：一眼判断

**规律：「事前型」hook 是父先子后，「事后型」hook 是子先父后。**

| Hook 类型 | React (Hook) | 触发顺序 | 阶段 |
|----------|--------------|---------|------|
| 函数体执行 | 函数体本身 | 父先子后 | Render |
| 挂载完成 | `useEffect(fn, [])` | **子先父后** | Commit |
| 更新完成 | `useEffect(fn)` | **子先父后** | Commit |
| 卸载清理 | useEffect cleanup | **子先父后** | Commit |
| Layout 测量 | `useLayoutEffect` | **子先父后** | Commit（同步） |

**为什么 mount 是子先？** 物理原因：父的 DOM 必须包含子的 DOM。要让父「mount 完成」（DOM 在文档里），必须先把子的 DOM 装进父里。

```js
useEffect(() => {
  console.log(document.querySelector('.child').offsetHeight)
  // 如果父先 mounted，此时子还没插入，查不到
}, [])
```

所以框架必须等子全部 mount 完，父才能报 mount。

详见 [[React Fiber]] 中关于 Fiber 节点遍历顺序的实现细节。

---

## 8. 渲染 ≠ DOM 更新

容易混淆的两个概念：

```text
触发 setState
   │
   ▼
1. React 调用组件函数（render 阶段）    ← 可能因 memo/bail out 跳过
   │
   ▼
2. 生成新虚拟 DOM
   │
   ▼
3. diff 新旧虚拟 DOM
   │
   ▼
4. commit 阶段：把变化应用到真实 DOM    ← diff 没差异则 DOM 不动
   │
   ▼
5. useEffect / useLayoutEffect 运行
```

「重新渲染」严格说指**第 1 步**：组件函数被再次调用。即使函数被调用了，DOM 也可能完全没变（React 帮你优化掉了）。

---

## 9. 常见困惑速查

| 问题 | 答案 |
|------|------|
| `<Child />` 会立即执行 Child 吗？ | 不会，只创建对象 |
| 组件嵌套是函数嵌套调用吗？ | 不是，是先后调用 |
| useState 是局部变量吗？ | 不是，存在 Fiber 节点上，函数出栈后还在 |
| JSX 表达式 `{a + b}` 是延迟执行吗？ | 不是，立即求值。**只有组件本身被延迟** |
| 子 setState 父知道吗？ | 不知道，state 严格私有 |
| 子组件 mounted 时父 mounted 了吗？ | 还没。子先 mounted，父后 mounted |

---

## 相关文章

- [[React Fiber]] — Fiber 节点结构、遍历算法、可中断渲染的实现
- [[React vs Vue 渲染对比]] — 反应式模型与重渲染粒度的差异
- [[App Router 状态管理]] — Context Provider 嵌套和单向数据流的实战应用
- [[JS执行模型]] — JS 调用栈、事件循环（理解父子函数先后关系的基础）

---

## 参考资料

- [React 官方文档：Render and Commit](https://react.dev/learn/render-and-commit)
- [React 官方文档：State as a Snapshot](https://react.dev/learn/state-as-a-snapshot)
- [React 官方文档：Queueing a Series of State Updates](https://react.dev/learn/queueing-a-series-of-state-updates)
- React 源码 `packages/react-reconciler/src/ReactFiberBeginWork.js`
