# JavaScript 知识地图：从加载到渲染

> JavaScript 在浏览器中的完整执行流程，从代码加载到页面绘制的全过程

## 核心流程概览

JavaScript 在浏览器中的执行流程可以概括为以下几个关键阶段：

1. **HTML 解析与脚本加载** - 浏览器解析 HTML，遇到 `<script>` 标签时加载 JavaScript 代码
2. **JavaScript 引擎初始化** - 创建代理（Agent）、域（Realm）和执行环境
3. **代码解析阶段** - 词法分析（Tokenization）和语法分析（Parsing）
4. **编译/解释执行** - 将代码转换为可执行的字节码或机器码
5. **执行上下文创建** - 创建全局执行上下文和函数执行上下文
6. **作用域链与闭包** - 变量查找机制和闭包的形成
7. **事件循环与异步执行** - 处理异步任务和回调函数
8. **DOM 操作与渲染** - JavaScript 操作 DOM 触发页面更新
9. **页面绘制** - 浏览器将 DOM 和样式渲染到屏幕上

---

## 1. HTML 解析与脚本加载

### 1.1 HTML 解析过程

当浏览器接收到 HTML 文档时，会启动 HTML 解析器：

- **解析顺序**: 从上到下，从左到右
- **遇到 `<script>` 标签**: 暂停 HTML 解析，开始处理 JavaScript
- **脚本类型**:
  - **同步脚本** (`<script>`): 阻塞 HTML 解析
  - **异步脚本** (`<script async>`): 并行下载，下载完成后立即执行
  - **延迟脚本** (`<script defer>`): 并行下载，等待 HTML 解析完成后执行
  - **模块脚本** (`<script type="module">`): ES6 模块，默认 defer

### 1.2 脚本加载机制

```javascript
// 同步脚本 - 阻塞解析
<script src="app.js"></script>

// 异步脚本 - 不阻塞解析，下载完立即执行
<script async src="app.js"></script>

// 延迟脚本 - 不阻塞解析，HTML 解析完再执行
<script defer src="app.js"></script>
```

### 1.3 资源加载优先级

- **关键资源**: 阻塞渲染的 CSS、同步 JavaScript
- **非关键资源**: 图片、异步脚本等
- **预加载**: `<link rel="preload">` 可以提前加载关键资源

---

## 2. JavaScript 引擎初始化

### 2.1 引擎与宿主环境

JavaScript 执行需要两个组件配合：

- **JavaScript 引擎**: 实现 ECMAScript 规范，解析和执行代码
  - V8 (Chrome, Edge)
  - SpiderMonkey (Firefox)
  - JavaScriptCore (Safari)
- **宿主环境**: 提供 Web API 和运行环境
  - 浏览器: 提供 DOM、BOM、Web API
  - Node.js: 提供文件系统、网络等 API

### 2.2 代理（Agent）创建

每个 JavaScript 执行环境都是一个**代理**，包含：

- **堆（Heap）**: 存储对象和复杂数据结构
- **队列（Queue）**: 事件循环的任务队列
- **栈（Stack）**: 执行上下文栈（调用栈）

参考: [[JS执行模型#代理执行模型]]

### 2.3 域（Realm）初始化

每个代理拥有一个或多个**域**，包含：

- **固有对象**: `Array`、`Object`、`Function` 等内置对象
- **全局对象**: `window`（浏览器）或 `global`（Node.js）
- **全局变量**: 全局作用域中声明的变量

参考: [[JS执行模型#域（Realm）]]

---

## 3. 代码解析阶段

### 3.1 词法分析（Tokenization/Lexical Analysis）

将源代码字符串分解为**词法单元（Token）**：

```javascript
// 源代码
const x = 10 + 20;

// 词法单元
[
  { type: 'keyword', value: 'const' },
  { type: 'identifier', value: 'x' },
  { type: 'operator', value: '=' },
  { type: 'number', value: '10' },
  { type: 'operator', value: '+' },
  { type: 'number', value: '20' },
  { type: 'punctuator', value: ';' }
]
```

### 3.2 语法分析（Parsing）

将词法单元转换为**抽象语法树（AST）**：

```javascript
// AST 结构示例
{
  type: 'VariableDeclaration',
  kind: 'const',
  declarations: [{
    type: 'VariableDeclarator',
    id: { type: 'Identifier', name: 'x' },
    init: {
      type: 'BinaryExpression',
      operator: '+',
      left: { type: 'Literal', value: 10 },
      right: { type: 'Literal', value: 20 }
    }
  }]
}
```

### 3.3 解析错误处理

- **语法错误**: 在解析阶段发现，代码不会执行
- **运行时错误**: 在执行阶段发现

---

## 4. 编译/解释执行

### 4.1 执行策略

现代 JavaScript 引擎采用**即时编译（JIT）**策略：

1. **解释执行**: 首次执行时快速转换为字节码并执行
2. **性能分析**: 监控代码执行频率（热点代码）
3. **优化编译**: 将热点代码编译为优化的机器码
4. **去优化**: 如果假设失效，回退到解释执行

### 4.2 V8 引擎执行流程

```
源代码 → 解析器（Parser）→ 抽象语法树（AST）
                                    ↓
解释器（Ignition）← 字节码生成器 ← AST
                                    ↓
优化编译器（TurboFan）← 性能分析器
```

### 4.3 代码优化技术

- **内联缓存（Inline Cache）**: 缓存对象属性访问
- **隐藏类（Hidden Class）**: 优化对象属性访问
- **内联函数**: 减少函数调用开销
- **死代码消除**: 移除未使用的代码

---

## 5. 执行上下文创建

### 5.1 执行上下文类型

- **全局执行上下文**: 代码首次执行时创建，只有一个
- **函数执行上下文**: 每次函数调用时创建
- **Eval 执行上下文**: 使用 `eval()` 时创建（不推荐）

### 5.2 执行上下文组成

每个执行上下文包含：

1. **变量环境（Variable Environment）**
   - `var` 声明的变量
   - 函数声明
   - `arguments` 对象（函数上下文）

2. **词法环境（Lexical Environment）**
   - `let`、`const` 声明的变量
   - 块级作用域
   - `this` 绑定

3. **外部环境引用（Outer Environment Reference）**
   - 形成作用域链

参考: [[JS执行模型#栈与执行上下文]]

### 5.3 执行上下文创建过程

```javascript
function outer() {
  var a = 1;
  function inner() {
    var b = 2;
    console.log(a + b);
  }
  inner();
}
outer();

// 执行上下文栈的变化：
// 1. [全局执行上下文]
// 2. [全局执行上下文, outer 执行上下文]
// 3. [全局执行上下文, outer 执行上下文, inner 执行上下文]
// 4. inner 执行完毕，弹出 inner 执行上下文
// 5. outer 执行完毕，弹出 outer 执行上下文
```

### 5.4 变量提升（Hoisting）

- **函数声明**: 完全提升，可以在声明前调用
- **var 变量**: 提升声明，但值为 `undefined`
- **let/const**: 提升但不初始化，形成"暂时性死区"

```javascript
// 函数声明提升
console.log(foo); // [Function: foo]
function foo() {}

// var 提升
console.log(x); // undefined
var x = 10;

// let/const 暂时性死区
console.log(y); // ReferenceError
let y = 20;
```

---

## 6. 作用域链与闭包

### 6.1 作用域链

作用域链决定了变量查找的顺序：

```javascript
var globalVar = 'global';

function outer() {
  var outerVar = 'outer';
  
  function inner() {
    var innerVar = 'inner';
    console.log(innerVar);  // 当前作用域
    console.log(outerVar);  // 外部作用域
    console.log(globalVar); // 全局作用域
  }
  
  inner();
}
```

**查找顺序**: 当前作用域 → 外部作用域 → ... → 全局作用域

### 6.2 闭包（Closure）

闭包是函数能够访问其外部作用域变量的机制：

```javascript
function createCounter() {
  let count = 0;
  
  return function() {
    count++;
    return count;
  };
}

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
```

**闭包的形成条件**:

- 函数嵌套
- 内部函数引用外部函数的变量
- 内部函数在外部函数外部被调用

### 6.3 闭包的应用场景

- **数据私有化**: 封装私有变量
- **函数工厂**: 创建具有特定行为的函数
- **模块模式**: 实现模块化编程
- **事件处理**: 保存事件处理器的上下文

---

## 7. 事件循环与异步执行

### 7.1 事件循环机制

JavaScript 是单线程的，通过**事件循环**实现异步执行：

```
┌─────────────────────────┐
│   调用栈（Call Stack）    │
└─────────────────────────┘
           ↑ ↓
┌─────────────────────────┐
│   事件循环（Event Loop）  │
└─────────────────────────┘
           ↑ ↓
┌─────────────────────────┐
│   任务队列（Task Queue）  │
└─────────────────────────┘
```

### 7.2 任务队列类型

1. **宏任务（Macro Task）**
   - `setTimeout`、`setInterval`
   - I/O 操作
   - UI 渲染

2. **微任务（Micro Task）**
   - `Promise.then()`、`Promise.catch()`
   - `queueMicrotask()`
   - `MutationObserver`

### 7.3 执行顺序

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

console.log('4');

// 输出: 1, 4, 3, 2
// 执行顺序: 同步代码 → 微任务 → 宏任务
```

### 7.4 Promise 执行机制

```javascript
Promise.resolve()
  .then(() => {
    console.log('Promise 1');
    return Promise.resolve();
  })
  .then(() => {
    console.log('Promise 2');
  });

queueMicrotask(() => {
  console.log('Microtask');
});

// 输出顺序: Promise 1, Promise 2, Microtask
```

参考: [[JS执行模型#代理执行模型]]

---

## 8. DOM 操作与渲染

### 8.1 DOM 树构建

浏览器解析 HTML 时构建 DOM 树：

```html
<html>
  <body>
    <div id="app">Hello</div>
  </body>
</html>
```

```
Document
└── html
    └── body
        └── div#app
            └── "Hello"
```

### 8.2 CSSOM 构建

浏览器解析 CSS 构建 CSSOM（CSS Object Model）：

```css
div {
  color: red;
  font-size: 16px;
}
```

### 8.3 渲染树（Render Tree）

结合 DOM 和 CSSOM 生成渲染树：

- 只包含需要渲染的节点（排除 `display: none`）
- 包含每个节点的样式信息

### 8.4 JavaScript 操作 DOM

```javascript
// 查询 DOM
const element = document.getElementById('app');

// 修改内容
element.textContent = 'Hello World';

// 修改样式
element.style.color = 'blue';

// 添加事件监听
element.addEventListener('click', () => {
  console.log('Clicked');
});
```

### 8.5 重排（Reflow）与重绘（Repaint）

- **重排**: 改变元素的几何属性（宽高、位置），需要重新计算布局
- **重绘**: 改变元素的视觉属性（颜色、背景），只需要重新绘制

**性能优化**:

- 批量 DOM 操作
- 使用 `DocumentFragment`
- 使用 `requestAnimationFrame`
- 避免频繁读取布局属性

---

## 9. 页面绘制

### 9.1 渲染流程

```
1. 解析 HTML → DOM 树
2. 解析 CSS → CSSOM 树
3. 合并 DOM + CSSOM → 渲染树
4. 布局（Layout/Reflow）→ 计算元素位置和大小
5. 绘制（Paint）→ 填充像素
6. 合成（Composite）→ 图层合成
```

### 9.2 图层合成

浏览器将页面分为多个图层：

- **主图层**: 页面主要内容
- **合成图层**: 使用 `transform`、`opacity` 等属性的元素

**优势**: 只更新变化的图层，提高性能

### 9.3 渲染优化

```javascript
// 使用 requestAnimationFrame 优化动画
function animate() {
  // 更新动画状态
  element.style.transform = `translateX(${x}px)`;
  
  requestAnimationFrame(animate);
}

// 使用虚拟滚动处理大量数据
// 使用 Web Workers 处理复杂计算
// 使用 Intersection Observer 实现懒加载
```

### 9.4 关键渲染路径（Critical Rendering Path）

优化首次渲染的关键步骤：

1. **减少关键资源数量**
2. **减小关键资源大小**
3. **缩短关键渲染路径长度**

---

## 总结

JavaScript 在浏览器中的完整执行流程：

```
HTML 解析
  ↓
脚本加载
  ↓
引擎初始化（代理、域）
  ↓
代码解析（词法分析、语法分析）
  ↓
编译/解释执行（JIT）
  ↓
执行上下文创建
  ↓
作用域链查找
  ↓
代码执行
  ↓
事件循环处理异步任务
  ↓
DOM 操作
  ↓
页面渲染
```

理解这个完整流程有助于：

- **性能优化**: 知道瓶颈在哪里
- **调试技巧**: 理解代码执行顺序
- **异步编程**: 掌握事件循环机制
- **内存管理**: 理解作用域和闭包

---

## 相关文档

- [[JS执行模型]] - JavaScript 执行模型的详细说明
- [[undefined == null]] - JavaScript 类型系统
- [[JS执行模型#事件循环与异步执行]] - 异步执行机制
