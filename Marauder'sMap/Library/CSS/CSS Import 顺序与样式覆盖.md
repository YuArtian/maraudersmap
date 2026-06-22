#CSS #样式覆盖 #模块化 #Vite #Webpack

# CSS Import 顺序与样式覆盖

## TL;DR

**CSS 优先级 = 特异性（selector specificity）+ 加载顺序。特异性相同时，"后加载的覆盖先加载的"。打包工具按 JS import 顺序输出 CSS，所以"业务 CSS 想覆盖第三方库 CSS"必须保证业务 CSS 在 import 链路里更靠后。**

- CSS 选择器特异性相同时，**源代码顺序决定胜负**（后写覆盖先写）
- Vite/Webpack 把 CSS 当模块处理，按 JS `import` 链的执行顺序注入 `<style>` 标签
- 把第三方 CSS 在组件内部 import，会让它**晚于**入口 CSS 加载，导致库的默认样式覆盖你的 override
- 修复方法：在入口（main.tsx / main.jsx）里**先** import 第三方 CSS，**后** import 业务 CSS
- 进阶：用 CSS Modules / scoped CSS / 更高特异性选择器 / `!important`（不推荐）

---

## 1. CSS 优先级的两层规则

浏览器决定一条规则是否生效，按这个顺序：

```text
1. !important（不推荐使用）
2. 内联 style="..."
3. 选择器特异性 (specificity)
4. 源代码顺序（后写覆盖先写）
```

第 3 步**特异性公式**（简化版）：

| 选择器类型 | 权重 |
|---|---|
| inline style | 1000 |
| `#id` | 100 |
| `.class` / `[attr]` / `:hover` | 10 |
| `tag` / `::before` | 1 |

举例：`#main .btn` 特异性 = 100+10 = 110，`.container .btn` = 10+10 = 20，前者赢。

**关键陷阱**：当两条规则**特异性完全相同**，浏览器看的是"谁后加载/后定义"。

```css
/* a.css 先加载 */
.bar { background: blue; }

/* b.css 后加载 */
.bar { background: red; }
/* 最终：red 赢 */
```

## 2. 打包工具如何决定 CSS 加载顺序

现代前端工程里，CSS 不再是手动 `<link>` 在 HTML 里，而是被当作模块用 JS `import` 引入：

```js
import "./styles.css";
```

Vite / Webpack 看到这个 import，会：
1. 把 CSS 文件作为依赖加入模块图
2. 按 JS 模块**执行顺序**（深度优先），把 CSS 内容收集起来
3. dev 模式注入 `<style>` 标签，prod 模式合并打包

**CSS 在最终 HTML 里的顺序 = 模块图的执行顺序**。

```text
main.jsx
├── import "./index.css"          ← (1) 第一个加载
├── import App                     ← App 模块开始执行
│   ├── import Header              
│   │   └── import "./header.css"  ← (2) 第二个加载
│   └── import "./app.css"         ← (3) 第三个加载
└── ...

最终 <head> 里顺序：
<style>(1) index.css</style>
<style>(2) header.css</style>
<style>(3) app.css</style>
```

**(3) 会覆盖 (1) 中特异性相同的规则。**

## 3. 经典坑：第三方库 CSS 覆盖你的 override

很多 npm 包提供默认 CSS（NProgress、Swiper、ReactQuill 等）。typical 错误写法：

```jsx
// MyComponent.jsx
import NProgress from "nprogress";
import "nprogress/nprogress.css";  // ❌ 在组件内 import

NProgress.configure({ ... });
```

```css
/* index.css —— 入口最先加载 */
#nprogress .bar { background: navy; }  /* 想覆盖默认蓝 */
```

**实际加载顺序：**

```text
main.jsx
├── import "./index.css"                ← (1) 我的 override
├── import App
│   └── import MyComponent
│       └── import "nprogress/nprogress.css"  ← (2) 库默认色
```

特异性都是 `#nprogress .bar`（相同），**(2) 后加载 → 库的蓝色赢**，你的 navy override 无效。

控制台看 computed styles，会看到默认色生效，覆盖被划掉。

## 4. 三种修复方式

### 4.1 调整 import 顺序（推荐）

把第三方 CSS 提到入口最前面，**早于**业务 CSS：

```js
// main.jsx
import "nprogress/nprogress.css";  // ← 先
import "./index.css";              // ← 后，覆盖生效
import App from "./App";
```

简单、零运行时成本。**优先用这个。**

### 4.2 提高自己规则的特异性

```css
/* 加一层 ID 或 body 提高权重 */
body #nprogress .bar { background: navy; }
```

可行但不优雅，破坏选择器简洁性。

### 4.3 `!important`（不推荐）

```css
#nprogress .bar { background: navy !important; }
```

能解决问题但污染样式系统，未来要再覆盖就得用更高级别的 `!important` 链战。**最后手段。**

| 方法 | 优点 | 缺点 |
|---|---|---|
| 调整 import 顺序 | 简单、零侵入 | 需要知道这个机制 |
| 提高特异性 | 不依赖 import 顺序 | 选择器变长、不直观 |
| `!important` | 一定生效 | 污染样式系统，难维护 |

## 5. 为什么 CSS Modules / scoped CSS 不会有这个问题

```jsx
import styles from "./Button.module.css";
<button className={styles.btn} />
```

CSS Modules 在编译时把 `.btn` 改名成 `.Button_btn__xK3p9`（全局唯一），所以**完全不会和第三方 CSS 的 `.btn` 冲突**，选择器特异性也独立。

| 方案 | 全局命名冲突风险 | 是否怕加载顺序 |
|---|---|---|
| 全局 CSS | 有 | 怕 |
| CSS Modules | 无 | 不怕（选择器都是唯一的） |
| Vue scoped / Svelte | 无 | 不怕（自动加 hash 属性） |
| Tailwind | 有（utility 同名） | 怕（utility 顺序很重要） |
| styled-components / Emotion | 无 | 不怕（运行时生成） |

但**第三方库的 CSS 不在 CSS Modules 体系内**（它们用全局选择器），所以"override 第三方库"这个场景下还是要靠 import 顺序或特异性。

## 6. 实战 checklist

引入新的第三方库带 CSS 时：

```text
[ ] 把库的 CSS import 放到入口（main.tsx/main.jsx），不要散在组件里
[ ] 业务 CSS（index.css）import 放在第三方 CSS 之后
[ ] 如果想覆盖某条规则，先在 DevTools 看 computed styles，确认特异性
[ ] 特异性相同被覆盖 → 改 import 顺序
[ ] 特异性更低被覆盖 → 提高选择器权重
[ ] 实在搞不定再用 !important
```

## 相关文章

- [[React Router v7 路由懒加载与切页 UI]] - 实例：nprogress 主题色覆盖
- [[Container Queries]] - CSS 现代特性

## 参考资料

- MDN CSS Specificity: https://developer.mozilla.org/docs/Web/CSS/Specificity
- MDN The cascade: https://developer.mozilla.org/docs/Web/CSS/CSS_cascade/Cascade
- Vite CSS handling: https://vite.dev/guide/features.html#css
