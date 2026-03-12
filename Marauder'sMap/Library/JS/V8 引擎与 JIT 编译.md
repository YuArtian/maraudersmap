#JavaScript #V8 #JIT #引擎

# V8 引擎与 JIT 编译

## TL;DR

V8 是 Google 开发的 JavaScript 引擎（用于 Chrome 和 Node.js），用 C++ 编写。与 CPython 止步于解释字节码不同，V8 通过**多层 JIT 编译**将热点代码编译成原生机器码，性能接近 C。

- **执行流程**: 源码 → AST → 字节码（Ignition）→ JIT 机器码（TurboFan）
- **字节码对 CPython 是终点，对 V8 是起点**
- **隔离模型**: 每个线程（Worker）运行在独立的 V8 Isolate 中，不共享对象

---

## 1. 执行流程

```text
源码 (.js)
  → Parser → AST
  → Ignition（基线编译器）→ 字节码      ← 类似 CPython 到这里
  → Sparkplug → 轻量机器码（非优化）
  → TurboFan（优化编译器）→ 高度优化的机器码   ← CPython 没有这步
```

### 分层编译

| 层 | 名称 | 作用 |
|----|------|------|
| 1 | **Ignition** | 快速生成字节码，解释执行，同时收集类型反馈（profiling） |
| 2 | **Sparkplug** | 把字节码几乎 1:1 翻译成机器码，启动快但不做优化 |
| 3 | **TurboFan** | 对热点函数做激进优化——内联、逃逸分析、类型特化 |

### 一个直觉

同样一个循环：

```js
for (let i = 0; i < 1000000; i++) { sum += i; }
```

- **第 1 次**: Ignition 解释字节码（慢，但在收集信息："i 一直是整数"）
- **第 N 次**: TurboFan 介入，编译成相当于 `MOV`, `ADD`, `CMP`, `JMP` 几条机器指令（接近 C 的速度）

CPython 跑同样的循环，100 万次迭代**每次都要**：查字节码 → switch 分支 → 拆箱 PyObject → 做加法 → 装箱回 PyObject。

---

## 2. V8 的字节码

JS 也需要先编译成字节码。以 `a = 1 + 2` 为例，Ignition 字节码大致是：

```
LdaSmi [1]          // Lda = Load to Accumulator，把 1 加载到累加器
Add [2], [0]        // 累加器 += 2
Star r0             // Sta = Store from Accumulator to Register，存到寄存器 r0
```

### 基于寄存器 vs 基于栈

| | CPython | V8 Ignition |
|---|---|---|
| 虚拟机类型 | 基于**栈**（PUSH/POP） | 基于**寄存器**（累加器 + r0, r1...） |
| 同样的操作 | 4 条指令 | 3 条（少一次 PUSH/POP） |
| 指令长度 | 短（不需要编码寄存器号） | 长（要编码寄存器编号） |

基于寄存器的设计指令更少，因为不需要反复压栈弹栈。

---

## 3. 寄存器与累加器

### 3.1 物理寄存器

CPU 内部的**超小型存储单元**，速度最快，数量很少：

```text
存储层级（越往上越快、越小、越贵）:

寄存器    ~0.3ns    几十个     在 CPU 芯片里
L1 缓存   ~1ns     64KB
L2 缓存   ~4ns     256KB
L3 缓存   ~10ns    8MB
内存      ~100ns   16GB      内存条
硬盘      ~ms      1TB
```

寄存器比内存快几百倍。CPU 做运算时，必须先把数据搬到寄存器里：

```asm
mov rax, 1      ; 把 1 放进 rax 寄存器
add rax, 2      ; rax += 2，结果 3 在 rax 里
```

各架构通用寄存器数量对比：

| 架构 | 通用寄存器 | 说明 |
|------|-----------|------|
| x86-64（Intel/AMD） | 16 | 历史包袱，从 8086 的 8 个扩来的 |
| ARM64（Apple Silicon） | 31 | 后来设计，没有包袱 |
| RISC-V | 32 | 同上 |

寄存器不是越多越好——多了指令编码就要更多位来表示寄存器号，导致指令变长，指令缓存效率下降。

### 3.2 累加器

累加器是一个**特殊用途的寄存器**，专门存"当前运算结果"。V8 Ignition 中它是隐含的默认寄存器，大多数指令自动读写它：

```
LdaSmi [1]       // Load to Accumulator
Add [2], [0]     // 累加器 += 2
Star r0          // Store Accumulator to Register r0
```

好处是指令更短——不需要显式指定"从哪读、往哪存"。

### 3.3 物理 vs 虚拟寄存器

V8 Ignition 的 r0、r1 是**虚拟寄存器**——本质就是 V8 用 C++ 定义的数据结构（实际在内存中）。叫"寄存器"是因为整个虚拟机的设计思路是**模仿真实 CPU 的工作方式**：

- 真实 CPU：指令集 + 物理寄存器 → 执行机器码
- V8 Ignition：字节码指令集 + 虚拟寄存器 → 解释执行字节码

只有当 TurboFan 把字节码编译成机器码时，才会尽量把虚拟寄存器映射到真正的 CPU 物理寄存器上——这也是 JIT 快的原因之一。

---

## 4. 为什么 JS 引擎有 JIT 而 CPython 没有

**浏览器性能军备竞赛**。JS 是浏览器里唯一的编程语言，用户体验直接取决于 JS 性能：

- 2008 年 Google 推出 V8，引发引擎性能竞赛
- Google → V8（Chrome）
- Mozilla → SpiderMonkey（Firefox）
- Apple → JavaScriptCore（Safari）

三家巨头互相卷，每家都做到了多层 JIT。Python 没有这种竞争压力。

另外 Python 的动态性比 JS 更极端（运行时可以修改类、元类、`__getattr__`），让 JIT 的推测优化更难做、更容易回退。加上 Python 生态依赖 C 扩展（NumPy 等）获得性能，纯 Python 代码 JIT 的收益被稀释。

---

## 5. 线程模型：隔离 vs 共享

### JS：从设计上禁止共享

浏览器里 JS 很长一段时间**只有一个线程，连创建线程的 API 都没有**。后来加了 Web Worker，故意设计成隔离的：

```js
// 主线程
const data = []
const worker = new Worker('task.js')
worker.postMessage(data)  // 深拷贝，不是共享引用

// task.js
onmessage = (e) => {
    e.data.push(1)  // 这是一份副本，不会影响主线程的 data
}
```

所谓"禁止"不是靠检查或锁，而是**压根没给共享的途径**：

- Worker 里访问不到主线程的变量（不同的全局作用域）
- `postMessage` 传数据做的是**深拷贝**（structured clone）
- Worker 里甚至访问不到 `document`、`window`

### 引擎层面的实现

V8 给每个 Worker 创建独立的 **Isolate**（隔离单元）：

```text
主线程 V8 Isolate          Worker V8 Isolate
┌──────────────┐          ┌──────────────┐
│  自己的堆     │          │  自己的堆     │
│  自己的 GC    │  ←消息→  │  自己的 GC    │
│  自己的字节码  │          │  自己的字节码  │
└──────────────┘          └──────────────┘
```

这是**语言规范 + 引擎实现**共同决定的：
- ECMAScript 规范定义了 Worker 通信方式是 `postMessage`（深拷贝）
- V8 引擎用独立的 Isolate（独立堆）来**落实**这个规定

唯一的例外是 `SharedArrayBuffer`——但那是原始字节数组，不是 JS 对象。

### 为什么 JS 选择隔离

JS 诞生在浏览器里，多线程操作 DOM 会是灾难：

```js
// 如果允许共享
线程A: document.body.removeChild(div)
线程B: div.style.color = 'red'   // div 已经没了
```

所以浏览器从第一天就规定只有一个线程能碰 DOM，整个语言围绕单线程 + 事件循环设计。

### 对比 Python

| | Python | JavaScript |
|---|---|---|
| 线程模型 | 共享内存（OS 真线程） | 消息传递（不共享） |
| 并发安全靠 | 锁（GIL + 用户锁） | 隔离（根本碰不到别人的数据） |
| 设计背景 | 沿用 C/Java/OS 的传统线程模型 | 为浏览器环境做了简化 |

Python 不是"故意"要共享——只是沿用了传统做法。JS 是刻意做了限制，用隔离换来了不需要锁的简单性。

---

## 6. Node.js 可以调用 C/C++

浏览器 JS 不能调用 C（沙箱安全），但 Node.js 可以，原理和 CPython 的 C 扩展一样：

```cpp
// addon.cc — Node.js C++ 扩展
#include <node.h>

void FastAdd(const v8::FunctionCallbackInfo<v8::Value>& args) {
    int a = args[0].As<v8::Int32>()->Value();
    int b = args[1].As<v8::Int32>()->Value();
    args.GetReturnValue().Set(a + b);
}
```

```js
const addon = require('./build/Release/addon')
addon.fastAdd(1, 2)  // 调用的是 C++ 代码
```

区别在运行环境，不在语言：

| 环境 | 能调用 C/C++ | 原因 |
|------|-------------|------|
| CPython | 能 | 跑在你的机器上 |
| Node.js | 能 | 跑在你的机器上 |
| 浏览器 JS | 不能 | 必须沙箱隔离（WebAssembly 是安全替代方案） |

---

## 7. 主流 JS 引擎对比

| 引擎 | 浏览器 | 语言 | JIT |
|------|--------|------|-----|
| **V8** | Chrome | C++ | Ignition + Sparkplug + TurboFan |
| **SpiderMonkey** | Firefox | C++ + Rust | 多层 JIT |
| **JavaScriptCore** | Safari | C++ | LLInt + Baseline + DFG + FTL |

---

## 相关概念

- [[CPython 解释器原理]] - Python 解释器的字节码、GIL、C 扩展
- [[JS执行模型]] - JavaScript 代理、域、事件循环、调用栈
