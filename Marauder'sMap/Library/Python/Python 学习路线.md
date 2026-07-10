#Python #索引 #学习路线

# Python 学习路线（JS/TS 背景 → 公司 AI 项目）

## TL;DR

**为「前端工程师转型 AI 项目开发」量身定制的 Python 学习路线。** 与 [[Python 教程总览]] 的通用路径不同，这条路线：

- **利用 JS/TS 基础加速**——每篇笔记标注了 🟢速通 / 🟡精读 / 🔴需新学，一半以上的内容可以对照 JS 扫过去
- **对齐公司技术栈**——终点不是「学会 Python」，而是能读懂并修改公司项目代码（FastAPI + Pydantic + LangChain + MCP + asyncio）
- **碎片时间友好**——按 6 个主题块编排，只给顺序和里程碑，不设日程表；每块用复选框打卡，随时中断随时续上

学完 Stage 3 即可进入 [[AI开发学习计划]] 的「LLM 应用基础」板块。

---

## 图例

| 标记 | 含义 | 学法 |
|---|---|---|
| 🟢 速通 | JS/TS 有直接对应物 | 扫一遍，重点看笔记末尾的 JS 对照表，记差异即可 |
| 🟡 精读 | Python 特有心智模型，JS 没有对应物或差异大 | 认真读 + 动手敲示例代码 |
| 🔴 需新学 | vault 尚无笔记，但公司项目必需 | 按下方资源表学，学完整理成 vault 新笔记 |

笔记文件名的前缀是「**阶段.序号**」（如 `1.08 列表生成式`），文件列表按名称排序即学习顺序。`3.01`–`3.04` 和 `4.06` 已预留给 Stage 3/4 待新建的 🔴 笔记；Stage 5 工具箱的编号只表示归属，不要求按序读。

---

## Stage 0 · 环境就绪

> 目标：本机能跑 Python，理解「解释执行」和 JS 运行时的异同。
> **里程碑：`uv init` 建一个项目，`uv run` 跑通 hello world。**

- [ ] 🟢 [[0.01 环境管理]] —— venv ≈ node_modules 的隔离思路，了解 Conda/pyenv 即可
- [ ] 🟢 [[0.02 环境变量与配置]] —— `.env` 加载，和前端 dotenv 一回事
- [ ] 🟢 [[0.03 解释型语言与代码发布]] —— 类比 V8：都是「源码 → 字节码 → 虚拟机」
- [ ] 🔴 **uv 包管理** —— 公司全部项目用 uv（≈ pnpm），vault 里的 [[PDM]] 可跳过。掌握 `uv init` / `uv add` / `uv run` / `uv sync` 四个命令即可上路

---

## Stage 1 · 语言核心速通

> 目标：把 JS 的语言直觉平移到 Python。全部 🟢，重点是「同一件事 Python 怎么写」。
> **里程碑：把一段你熟悉的 JS 工具函数（比如 debounce 或数组去重分组）徒手翻译成 Python。**

- [ ] 🟢 [[1.01 数据类型基础]] —— tuple 是新概念；`None` ≈ `null`，没有 `undefined`
- [ ] 🟢 [[1.02 表达式与语句]] —— 三元写法 `a if cond else b` 顺序和 JS 不同
- [ ] 🟢 [[1.03 变量与作用域]] —— LEGB 规则；**没有块级作用域**，`if`/`for` 内定义的变量外面可见
- [ ] 🟢 [[1.04 字符串格式化]] —— f-string ≈ 模板字符串，记这一种就够
- [ ] 🟢 [[1.05 字符编码]] —— str/bytes 之分是 JS 没有的，读文件、调 API 时会遇到
- [ ] 🟢 [[1.06 Set 集合]] —— 比 JS Set 好用：支持交并差运算符
- [ ] 🟢 [[1.07 迭代与切片]] —— 切片 `[1:5]`、`[::-1]` 是高频语法，必须熟
- [ ] 🟢 [[1.08 列表生成式]] —— ≈ `map`+`filter` 链式调用，公司代码里到处都是
- [ ] 🟢 [[1.09 函数参数]] —— `*args`/`**kwargs` ≈ rest/spread；**默认参数是可变对象的坑**必看
- [ ] 🟢 [[1.10 函数进阶]] —— lambda ≈ 箭头函数（但只能写一个表达式）
- [ ] 🟢 [[1.11 高阶函数]] —— `sorted(key=...)` 的 key 函数模式，Python 处处用
- [ ] 🟢 [[1.12 模块与包]] —— import ≈ ESM，`__init__.py` ≈ index.js

---

## Stage 2 · Python 特有心智模型

> 目标：掌握 JS 里没有（或长得完全不一样）的概念——这是读公司代码的分水岭。全部 🟡 精读。
> **里程碑：随手打开公司项目一个文件，能说清 `@xxx`、`__init__`、`with` 各自在做什么。**

- [ ] 🟡 [[2.01 装饰器与闭包]] —— `@decorator` ≈ HOC，但语法内建；FastAPI 路由、pytest fixture 全靠它
- [ ] 🟡 [[2.02 异常处理]] —— Python 用异常做控制流比 JS 频繁得多，`try/except/finally` + 自定义异常
- [ ] 🟡 [[2.03 contextlib 上下文管理]] —— `with` 语句：JS 没有的资源管理语法，公司代码里开文件、开连接、开事务全是它
- [ ] 🟡 [[2.04 面向对象编程]] —— `self` 显式传递、多继承 MRO、MixIn；比 JS class 体系重得多
- [ ] 🟡 [[2.05 属性访问与限制]] —— `_name` 约定私有、`@property` ≈ getter/setter
- [ ] 🟡 [[2.06 定制类]] —— dunder 方法（`__str__`/`__call__`/`__getattr__`）：Python 的「协议」机制，Pydantic/LangChain 大量使用
- [ ] 🟡 [[2.07 枚举类]] —— ≈ TS enum，公司 schema 定义常用
- [ ] 🟡 [[2.08 对象信息获取]] —— `type`/`isinstance`/`getattr` 反射三件套，读框架源码必备

---

## Stage 3 · 现代 Python 工程 ⭐ 公司栈核心

> 目标：这 4 项 vault 里还没有笔记，但恰是公司项目的地基——本路线相对 [[Python 教程总览]] 最大的增量。全部 🔴。
> **里程碑：用 Pydantic 定义请求/响应模型 + asyncio 并发调用 LLM API，写一个小脚本跑通。**

- [ ] 🔴 **typing 类型提示** —— 你会 TS 就成功了 80%：`def foo(x: int) -> str` ≈ TS 函数签名；重点学 `Optional`/`Union`（`str | None`）、泛型 `list[str]`、`Protocol`（≈ TS interface 结构化类型）。学完建 [[3.01 typing 类型提示]]
- [ ] 🔴 **asyncio 异步编程** —— `async def`/`await` 语法和 JS 几乎一样，差异在运行时：JS 天生有事件循环，Python 要 `asyncio.run()` 显式启动；`asyncio.gather()` ≈ `Promise.all()`。公司全部服务都是 async 的。学完建 [[3.02 asyncio 异步编程]]
- [ ] 🔴 **Pydantic v2** —— ≈ Zod：声明式 schema + 运行时校验 + 序列化。`BaseModel`、`Field`、validator、Discriminated Union（公司评估框架的 schema 定义重度使用）。学完建 [[3.03 Pydantic]]
- [ ] 🔴 **pytest** —— 比 vault 里的 [[5.07 单元测试]]（unittest）现代：函数即测试、fixture 依赖注入 ≈ Jest 的 setup、`assert` 直接用。公司测试全是 pytest。学完建 [[3.04 pytest]]

---

## Stage 4 · 服务端与数据

> 目标：补齐后端视角——IO、并发、数据库，收尾于 FastAPI。
> **里程碑：读懂公司数据生成项目的 LLM 封装模块（一个几百行的多 Provider 封装，见《AI开发学习计划》12.1 的适配器模式）。**

- [ ] 🟡 [[4.01 IO 编程]] —— 文件读写 + `with`，注意同步 IO 会阻塞事件循环（和 Node 直觉相反，Python 同步 API 是默认）
- [ ] 🟢 [[4.02 序列化]] —— `json` 模块 ≈ `JSON.parse/stringify`，pickle 了解即可
- [ ] 🟡 [[4.03 并发编程]] —— 线程/进程/协程三选一的决策思维 + GIL：为什么 Python 多线程不能并行算 CPU
- [ ] 🟡 [[4.04 资源池与连接池]] —— 池化模式，公司数据库连接管理的底层逻辑
- [ ] 🟡 [[4.05 数据库访问]] —— DB-API → SQLAlchemy ORM（≈ Prisma），参数化查询防注入
- [ ] 🔴 **FastAPI** —— ≈ Express/Nest：装饰器路由 + Pydantic 自动校验 + 依赖注入 + 自动 OpenAPI 文档。官方 tutorial 过一遍即可，Stage 2/3 的知识在这里全部串起来。学完建 [[4.06 FastAPI]]

---

## Stage 5 · 按需查阅工具箱

> 不排进主线，写代码遇到了再回来查。挑几篇和公司项目相关度高的优先。

| 笔记 | 什么时候查 |
|---|---|
| [[5.01 正则表达式]] | 处理文本/日志解析时 |
| [[5.02 datetime 时间处理]] | 处理健康数据时间序列时（公司高频） |
| [[5.03 collections 高级容器]] | 见到 `defaultdict`/`Counter` 时 |
| [[5.04 itertools 迭代工具]] | 见到 `chain`/`groupby` 时 |
| [[5.05 argparse 命令行参数]] | 读公司 CLI 脚本时 |
| [[5.06 调试技巧]] | 第一次需要断点调试时（建议早读） |
| [[5.07 单元测试]] / [[5.08 文档测试]] | 了解 unittest 遗留代码时 |
| [[5.09 密码学工具]] | 碰到 JWT/签名/哈希时 |
| [[5.10 urllib HTTP 客户端]] | 一般直接用 requests/httpx，了解即可 |
| [[5.11 XML 与 HTML 解析]] | 做文档解析时 |
| [[5.12 struct 二进制处理]] | 处理二进制文件格式时（少见） |
| [[5.13 网络编程]] | 想理解 socket 底层时 |
| [[5.14 常用第三方模块入门]] | 处理图像/编码检测时 |
| [[5.15 CPython 解释器原理]] | 兴趣向：想搞懂 GIL/字节码时 |

---

## 与《AI开发学习计划》的衔接

vault 根目录的 [[AI开发学习计划]] 是完整的 AI 开发知识大纲（8 个知识板块），本路线是其 **「Python 语言」+「现代 Python 工程」两个板块的展开版**：

```text
本路线 Stage 0-2、5  ≈  大纲「Python 语言」
       Stage 3       ≈  大纲「现代 Python 工程」
       Stage 4       ≈  大纲「Web 服务开发」的基础部分
       ────────────────────────────────────────
       Stage 3 完成后 → 进入「LLM 应用基础」
       Stage 4 完成后 → 进入「Agent 与工具调用」
```

碎片时间下不必追求「学完再上手」：**Stage 3 一过就去做大纲里的实战项目 1/2（CLI 聊天机器人、数据生成器），或直接上手公司的数据生成项目**，读不懂的地方回头补对应笔记，比线性刷完效率高。

---

## 🔴 缺口主题资源表

| 主题 | 资源 | 建议顺序 |
|---|---|---|
| uv | https://docs.astral.sh/uv/ | Stage 0 就学，后面所有练习用它建项目 |
| typing | https://docs.python.org/3/library/typing.html + https://mypy.readthedocs.io/ | 官方 typing 文档只看常用部分；mypy 装上让 IDE 报错即可 |
| asyncio | https://docs.python.org/3/library/asyncio-task.html | 只看 Coroutines & Tasks 一章：`run`/`gather`/`create_task`/`async with` |
| Pydantic v2 | https://docs.pydantic.dev/latest/ | Models → Fields → Validators → Discriminated Union，对照 Zod 学 |
| pytest | https://docs.pytest.org/en/stable/getting-started.html | Getting Started + fixture 一章 |
| FastAPI | https://fastapi.tiangolo.com/tutorial/ | 官方 tutorial 前半（路由/请求体/依赖注入），2 小时够 |

> 每学完一个 🔴 主题，整理成 vault 笔记并回 [[Python 教程总览]] 的 TODO 表打勾——让知识库跟着你一起成长。

---

## 参考资料

- [[Python 教程总览]] —— 全部 46 篇笔记的索引与通用学习路径
- [[AI开发学习计划]] —— 8 个知识板块大纲 + 实战项目阶梯 + 通用设计模式速览
- 廖雪峰 Python 教程 https://liaoxuefeng.com/books/python/ （vault 根目录有 2025-06 版 PDF）
