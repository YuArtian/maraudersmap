#React #性能优化 #useEffect #ReactCompiler

# React Compiler 与 useEffect 依赖

## TL;DR

**React Compiler 自动做的记忆化是"渲染路径的优化"，不是"effect 触发次数的契约"——effect 依赖项是少数仍然值得手写 `useMemo`/`useCallback` 的地方。**

- 编译器目标是让 `React.memo` 和 reconciliation 跳过工作；`useEffect` 用 `Object.is` 严格比较依赖
- 这两个失效模型 90% 时候一致，剩下 10% 不一致时 **吃亏的是 effect**（多触发一次 = 真实副作用代价）
- 不稳定来源：反应式作用域**连坐失效** / 编译器**整段 bail out** / 上游引用不稳定 / 编译器**主动不缓存**便宜值 / 启发式随版本变化
- 实践规则：**effect 依赖项尽量是原始值；必须是对象/函数时，用 `useMemo`/`useCallback` 兜底**

---

## 1. 官方文档怎么说

React 官方文档（[react-compiler/introduction](https://zh-hans.react.dev/learn/react-compiler/introduction)）的原话：

> 然而，在某些情况下，开发者可能需要对记忆化进行更精细的控制。`useMemo` 和 `useCallback` Hooks 可以继续与 React 编译器一起使用，作为一种脱围机制，用于控制哪些值会被记忆化。**一个常见的用例是，如果记忆化的值被用作 effect 的依赖项，即使其依赖项没有实质性变化，effect 也不会反复触发。**

这一节翻译成大白话：**effect 的依赖项是少数几个值得保留 `useMemo` 的场景**，原因不是编译器做得差，而是这里你需要的是**契约**，而编译器只给**优化**。

---

## 2. 根本原因：两套不同的失效模型

| 维度 | React Compiler | useEffect |
|---|---|---|
| 目标 | 让 reconciliation 跳过工作 | 依赖变化时重跑 effect |
| 何时介入 | 渲染过程中的值复用 | 每次渲染后比较依赖 |
| 比较方式 | 编译器自己的 `if ($[i] !== ...)` 缓存逻辑 | 对依赖数组逐项 `Object.is` |
| 性质 | **优化**（best-effort） | **语义契约**（硬规则） |
| 失效代价 | 多算一次值/多渲染一次 | 多跑一次副作用（请求、订阅、写日志） |

编译器只能尽量让你的依赖值**引用稳定**，但稳定与否是它单方面努力的结果，**不是承诺**。

```text
            ┌─────────────────────┐
            │  Compiler 优化目标   │
            │   (避免重渲染)        │
            └──────────┬───────────┘
                       │ 让 dep 值稳定（best-effort）
                       ▼
       ┌──────────────────────────────┐
       │  useEffect 比较 (Object.is)  │
       │   (硬契约，编译器管不到)       │
       └──────────────────────────────┘
                       │ 不等 → 重跑副作用
                       ▼
                  网络请求 / 订阅 / 日志
```

---

## 3. 五种"编译器记忆化失效"的具体场景

### 3.1 反应式作用域的"连坐失效"

编译器会把若干语句聚成一个 **reactive scope**，共用一段缓存槽。如果聚得不够细，组里任意一个依赖变化都会让整组重算：

```jsx
function Panel({ userId, filter, theme }) {
  const query = { userId, filter };       // 期望依赖只有 userId/filter
  const styles = computeStyles(theme);    // 跟 query 无关

  useEffect(() => { fetchData(query); }, [query]);
}
```

编译器**有可能**把 `query` 和 `styles` 划到同一个 scope。结果是 `theme` 一变，`query` 也被重算成新对象，effect 跟着重跑——`theme` 和这个 effect 八竿子打不着，却产生了副作用。

### 3.2 编译器整段 bail out

编译器检测到任何它没把握的代码，就**整个函数放弃编译**。触发条件比想象中多：

- ref 被以它不喜欢的方式赋值/读取
- 某个变量被推断为"可能在渲染期被修改"
- 用了不认识的自定义 Hook（内部规则破了）
- 闭包跨越了它分析不了的边界

bail out 后整个组件回退到没编译的状态，所有内部对象/函数都是每次渲染新建。生产环境的回归很难察觉——effect 多跑几次通常没明显报错，只是慢、贵、或后端日志里多了请求。

### 3.3 上游引用不稳定

编译器只能保证**它当前在编译的函数内部**的稳定性。父组件传新引用过来，再怎么缓存也救不回：

```jsx
function Parent() {
  return <Child config={{ mode: 'live' }} />;   // 父组件没编译/bail out
}

function Child({ config }) {
  useEffect(() => { connect(config); }, [config]);   // config 每次都是新引用
}
```

### 3.4 编译器主动不缓存便宜的值

编译器有成本模型，认为"重新计算比比较+取槽还便宜"的值会被**故意留作每次新建**：

- 简单字面量对象 `{a: 1}`
- 单参数箭头函数

对渲染没问题（`React.memo` 比较的是更上层的 element），但塞进 effect 依赖就是每次新引用。

### 3.5 编译器版本/启发式会变

`_c(n)` 槽位的划分、scope 合并策略、bail out 触发条件——都是**实现细节**，没进 React 的语义契约。今天稳定的值，升级编译器后被划进更大的 scope，行为就变了。对渲染是性能波动，对 effect 是**行为波动**。

---

## 4. 实践规则与解决方案

### 4.1 优先级（从好到差）

| 情况 | 做法 | 为什么 |
|---|---|---|
| 依赖是原始值（string/number/boolean） | **什么都不用做** | `Object.is` 天然稳定 |
| 依赖是对象/数组/函数，**只为 effect 而生** | 挪进 effect 内部，依赖原始值 | 根本不需要稳定引用 |
| 依赖是对象/数组/函数，渲染作用域里**还有别处用** | `useMemo` / `useCallback` 兜底 | 把稳定性写成契约 |

### 4.2 方案 A（首选）：依赖原始值

```jsx
function Panel({ userId, filter, theme }) {
  const styles = computeStyles(theme);

  useEffect(() => {
    const query = { userId, filter };   // 移到 effect 里面
    fetchData(query);
  }, [userId, filter]);                 // 依赖项只看原始值
}
```

**经验法则**：只要一个对象/数组只是为了喂给 effect 当依赖而存在，就把它挪进 effect 体内。

### 4.3 方案 B：对象在外部有别的用处时，用 useMemo

```jsx
const query = useMemo(
  () => ({ userId, filter }),
  [userId, filter]
);

return (
  <>
    <Sidebar query={query} />
    <Results query={query} />
  </>
);

useEffect(() => { fetchData(query); }, [query]);
```

这就是 React 文档说的"`useMemo` 作为脱围机制"。无论编译器怎么分 scope，`theme` 变化都不会牵连。

### 4.4 方案 C：抽成自定义 Hook

模式重复时，把"原始值进、副作用出"封装起来：

```jsx
function useFetchData(userId, filter) {
  useEffect(() => {
    fetchData({ userId, filter });
  }, [userId, filter]);
}

function Panel({ userId, filter, theme }) {
  useFetchData(userId, filter);
  const styles = computeStyles(theme);
}
```

好处是把"原始值依赖"这条规则**关在 Hook 边界里**，调用方接触不到那个临时对象，不可能写错。

### 4.5 反模式（不要这么做）

| 做法 | 问题 |
|---|---|
| 拆组件来切 scope | 过度工程，为缓存问题改架构 |
| 用 ref 绕过依赖检查 | 引入 stale closure bug |
| `JSON.stringify(query)` 当依赖 | 每次渲染都序列化，对函数/循环引用不稳，反模式 |

---

## 5. 一句话规则

> **Effect 的依赖项应该尽量是原始值；当它必须是对象/函数时，用 `useMemo`/`useCallback` 兜底。**

不是"用了 `useMemo` 才安全"，而是"**别让非原始值在没有契约的情况下进依赖数组**"。原始值不需要契约；非原始值要么挪走、要么上 `useMemo`，二选一。

这条规则在有编译器和没编译器的世界里**完全一样**——编译器只是让"忘了这条规则"的代价从"立刻出 bug"变成了"在某个版本升级后悄悄出 bug"。

---

## 相关文章

- [[ReactCompiler]] - React Compiler 使用指南
- [[ReactCompiler实现原理]] - `_c(n)` 缓存槽机制详解
- [[React 渲染机制]] - 重渲染的触发与时机

---

## 参考资料

- [React 编译器介绍（中文）](https://zh-hans.react.dev/learn/react-compiler/introduction)
- [React Compiler — Useful Patterns](https://react.dev/reference/react-compiler)
