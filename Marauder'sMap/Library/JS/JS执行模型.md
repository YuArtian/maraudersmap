# JavaScript 执行模型

> 参考文档: <https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Execution_model>

## TLDR

JavaScript 执行模型的核心架构：

- **代理（Agent）**: 自主执行单元，包含：
  - **堆（Heap）**: 存储对象
  - **队列（Queue）**: 实现事件循环的异步执行
  - **栈（Stack）**: 管理函数调用的执行上下文
- **域（Realm）**: 每个代理拥有一个或多个域，包含：
  - 固有对象（如 `Array`、`Array.prototype`）
  - 全局变量和 `globalThis`
- **执行上下文**: 跟踪：
  - 变量环境（Variable Environment）
  - 词法环境（Lexical Environment）
  - `this` 绑定
- **内存共享**: 多个代理可通过 `SharedArrayBuffer` 共享内存

## 引擎和宿主

JavaScript 的执行需要两个软件的配合：**JavaScript 引擎**和**宿主环境**。

- **JavaScript 引擎**: 实现了 ECMAScript（JavaScript）语言，提供了核心功能。它接收源代码，对其进行解析并执行。
- **宿主环境**: 为了与外部世界交互，例如产生任何有意义的输出、与外部资源接口，或实现与安全或性能相关的机制，我们需要由宿主环境提供额外的特定环境机制。

**示例**:

- 在 web 浏览器中执行 JavaScript 时，HTML DOM 就是宿主环境
- Node.js 是另一种允许 JavaScript 在服务器端运行的宿主环境

虽然我们在本参考文献中主要关注的是 ECMAScript 中定义的机制，但我们偶尔也会讨论 HTML 规范中定义的机制，这些机制通常会被 Node.js 或 Deno 等其他宿主环境所模仿。

## 代理执行模型

在 JavaScript 规范中，JavaScript 的每个自主执行器都被称为**代理**（Agent），它维护着自己的代码执行设施：
![[runtime-environment-diagram.svg]]


> 上图展示了由两个代理组成的执行模式：一个 HTML 页面和一个 Worker。每个代理都有自己的栈（包含执行上下文）、堆（包含对象）和队列（包含作业）。两个代理通过 SharedArrayBuffer 共享内存。

### 代理的组成部分

1. **（对象）堆（Heap）**
   - 这只是一个名称，用来表示内存中的一个大区域（大多是非结构化的）
   - 当程序中创建对象时，它就会被填充
   - 在共享内存的情况下，每个代理都有自己的堆，每个堆都有自己版本的 SharedArrayBuffer 对象，但缓冲区所代表的底层内存是共享的

2. **（作业）队列（Queue）**
   - 这在 HTML 中（通常）被称为**事件循环**（event loop）
   - 它可以在 JavaScript 中实现异步编程，同时又是单线程的
   - 之所以称其为队列，是因为它通常是先入先出（FIFO）：先执行的工作在后执行的工作之前

3. **（执行上下文）栈（Stack）**
   - 这就是所谓的**调用栈**（call stack）
   - 允许通过进入和退出执行上下文（如函数）来传输控制流
   - 之所以称为栈，是因为它是后进先出（LIFO）
   - 每个任务进入时都会向（空）栈中推入一个新帧，退出时则会清空栈

> 这是三种不同的数据结构，用于跟踪不同的数据。要了解堆内存如何分配和释放，请参阅内存管理。

### 代理的特性

- 每个代理都类似于一个线程（注意，底层实现可能是也可能不是实际的操作系统线程）
- 每个代理可以拥有多个域（与全局对象一一对应），这些代理可以同步访问彼此，因此需要在单个执行线程中运行
- 一个代理也有一个单一的内存模型，表明它是否是小端序的、是否可以同步阻塞、原子操作是否无锁等

### Web 上的代理类型

web 上的代理可以是以下之一：

- **相似源 window 代理**: 包含各种 Window 对象，这些对象有可能直接或通过使用 `document.domain` 相互联系
- **专用 Worker 代理**: 包含 `DedicatedWorkerGlobalScope`
- **共享 Worker 代理**: 包含 `SharedWorkerGlobalScope`
- **Service worker 代理**: 包含 `ServiceWorkerGlobalScope`
- **Worklet 代理**: 包含 `WorkletGlobalScope`

换句话说，每个 Worker 创建自己的代理，而一个或多个窗口可能在同一个代理中，通常是一个主文档及其类似的源 iframe。在 Node.js 中，也有一个类似的概念，称为 worker 线程。

## 域（Realm）

每个代理都拥有一个或多个**域**。每一段 JavaScript 代码在加载时都会与一个域相关联，即使从另一个领域调用也不会改变。

### 域的组成

域由以下信息组成：

- **固有对象列表**: 如 `Array`、`Array.prototype` 等
- **全局声明的变量**: `globalThis` 的值以及全局对象
- **模板字面数组的缓存**: 因为对同一标记的模板字面表达式的求值总是会导致标记接收到相同的数组对象

### Web 上的域

在 web 上，域和全局对象是一一对应的。全局对象可以是 `Window`、`WorkerGlobalScope` 或 `WorkletGlobalScope`。因此，举例来说，每个 `iframe` 都在不同的域中执行，尽管它可能与父窗口在同一个代理中。

### 域的重要性

在讨论全局对象的身份时，通常会提到域。例如，我们需要 `Array.isArray()` 或 `Error.isError()` 这样的方法，因为在另一个域构建的数组的原型对象与当前域中的 `Array.prototype` 对象不同，因此 `instanceof Array` 将错误地返回 `false`。

## 栈与执行上下文

### 执行上下文（Execution Context）

我们首先考虑同步代码执行。每个作业通过调用相关的回调进入。回调中的代码可以创建变量、调用函数或退出。每个函数都需要跟踪自己的变量环境和返回位置。

**执行上下文**一般也称为**栈帧**（stack frame），是执行的最小单位。它跟踪以下信息：

- **代码评估状态**
- **包含此代码的模块或脚本、函数**（如适用）以及当前执行的生成器
- **当前的域**
- **绑定**（Bindings），包括：
  - 变量环境（Variable Environment）
  - 词法环境（Lexical Environment）
  - 函数环境（Function Environment）
- **this 绑定**
- **其他状态信息**

### 调用栈的工作原理

当函数被调用时：

1. 创建一个新的执行上下文
2. 将该上下文推入调用栈
3. 执行函数代码
4. 函数返回时，从栈中弹出该上下文

这是典型的**后进先出**（LIFO）栈结构。

## 代理集群与内存共享

### SharedArrayBuffer

多个代理可以通过 `SharedArrayBuffer` 共享内存。这使得不同代理之间可以进行高效的通信和数据共享。

### 注意事项

- 每个代理都有自己的堆，但可以通过 SharedArrayBuffer 共享底层内存
- 需要小心处理并发访问，可能需要使用原子操作来保证数据一致性

## 规范

JavaScript 执行模型在以下规范中定义：

- **ECMAScript 规范**: 定义了核心语言特性
- **HTML 规范**: 定义了 web 环境中的特定机制（如事件循环）

## 相关概念

- **内存管理**: 了解堆内存如何分配和释放
- **事件循环**: 深入了解作业队列和异步执行机制
- **作用域和闭包**: 理解词法环境和变量环境
- **this 绑定**: 理解执行上下文中的 this 值

## 总结

JavaScript 执行模型的核心概念：

1. **代理（Agent）**: 每个自主执行器，包含堆、队列和栈
2. **域（Realm）**: 代码执行的环境，包含固有对象和全局变量
3. **执行上下文（Execution Context）**: 代码执行的最小单位，跟踪变量、this 绑定等
4. **调用栈（Call Stack）**: 后进先出的栈结构，管理函数调用
5. **事件循环（Event Loop）**: 先入先出的队列，实现异步编程

理解这些概念对于深入理解 JavaScript 的执行机制、异步编程、作用域和闭包等高级特性至关重要。
