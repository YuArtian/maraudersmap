#Python #索引 #总览

# Python 教程总览

## TL;DR

**本目录是基于廖雪峰《Python 教程》（2025-06 版）+ 个人实践整理的知识库。** 按主题分组：

- **基础与工具链** — 解释器、环境、打包
- **语言核心** — 数据类型、字符串、流程控制、变量作用域
- **函数体系** — 参数、lambda、高阶函数、装饰器、闭包
- **面向对象** — 类、继承、定制类、`@property`、枚举、对象内省
- **错误与并发** — 异常处理、IO、多线程/多进程、连接池

下文按主题列出全部文章 + 一个学习路径建议。

> 本页是**目录式**索引（按主题分组、便于查找）。想看这些知识点怎么串成一条线，读场景串讲版 [[Python/Map|Python 知识地图]]——跑一个 40 行脚本，从敲下回车到进程退出，途经每个知识点。

---

## 1. 学习路径

> 针对 JS/TS 背景 + 公司 AI 技术栈（FastAPI / Pydantic / asyncio）的个性化路线见 [[Python 学习路线]]，含速通/精读分级和打卡清单。

下面这张图是新手到进阶的推荐顺序，箭头表示"先学这个再学下一个"：

```text
[基础]
解释型语言与代码发布
    ↓
CPython 解释器原理
    ↓
环境管理 ── uv ── 环境变量与配置
    ↓
打包与分发（wheel / 源码包）
    ↓
[语言核心]
数据类型基础 ── 字符编码 ── 字符串格式化 ── Set 集合
    ↓
表达式与语句 ── 变量与作用域
    ↓
列表生成式 ── 迭代与切片
    ↓
[函数]
函数参数 ── 函数进阶 ── 高阶函数 ── 装饰器与闭包
    ↓
模块与包
    ↓
[面向对象]
面向对象编程 ── 属性访问与限制 ── 对象信息获取
    ↓
定制类 ── 枚举类
    ↓
[健壮性]
异常处理 ── IO 编程 ── 并发编程 ── 资源池与连接池
```

---

## 2. 全部文章索引

### 2.1 基础与工具链

| 文章 | 主要内容 |
|---|---|
| [[0.03 解释型语言与代码发布]] | 编译 vs 解释、源码发布、WASM、代码保护 |
| [[5.15 CPython 解释器原理]] | 执行流程、字节码、GIL、C 扩展 |
| [[0.01 环境管理]] | Anaconda / Conda / pyenv / uv 三条路线、虚拟环境 |
| [[0.05 uv]] | 一个工具全包：解释器、环境、装包、锁文件；与 pyenv 的差别、作用域、边界 |
| [[0.02 环境变量与配置]] | `.env` 文件、12-Factor 配置加载 |
| [[0.04 打包与分发]] | wheel 与源码包、文件名五段标签、manylinux、tar.gz vs zip |
| [[PDM]] | Python 依赖管理工具（可被 [[0.05 uv\|uv]] 替代） |
| [[1.12 模块与包]] | `import`、`__init__.py`、包结构 |

### 2.2 语言核心

| 文章 | 主要内容 |
|---|---|
| [[1.01 数据类型基础]] | int / float / bool / None / str / tuple，与 JS 对比 |
| [[1.05 字符编码]] | ASCII / Unicode / UTF-8、Python 3 默认 UTF-8 |
| [[1.04 字符串格式化]] | `%`、`.format()`、f-string 三种写法 |
| [[1.06 Set 集合]] | Python set vs JS Set、哈希去重、可哈希要求 |
| [[1.02 表达式与语句]] | 表达式 vs 语句的区别、三元写法、海象运算符 `:=` |
| [[1.03 变量与作用域]] | LEGB 查找规则、`global` / `nonlocal` |
| [[1.08 列表生成式]] | 列表/字典/集合推导式、生成器表达式 |
| [[1.07 迭代与切片]] | 迭代器协议、`__iter__` / `__next__`、切片语法 |

### 2.3 函数体系

| 文章 | 主要内容 |
|---|---|
| [[1.09 函数参数]] | 五种参数形态、默认参数的坑、解包 `*args` / `**kw` |
| [[1.10 函数进阶]] | lambda、`callable()`、函数类型、常量约定 |
| [[1.11 高阶函数]] | `map` / `reduce` / `filter` / `sorted` + key 函数 |
| [[2.01 装饰器与闭包]] | 闭包、`nonlocal`、装饰器两层/三层嵌套、`functools.wraps` / `partial` |

### 2.4 面向对象

| 文章 | 主要内容 |
|---|---|
| [[2.04 面向对象编程]] | 类、继承、多继承 MRO、MixIn、`__slots__`、`__getitem__`、metaclass |
| [[2.05 属性访问与限制]] | `_name` / `__name` 约定、name mangling、`@property` getter/setter、只读属性 |
| [[2.06 定制类]] | `__str__` / `__iter__` / `__getitem__` / `__getattr__` / `__call__` |
| [[2.07 枚举类]] | `Enum` / `@unique`、三种访问方式、带方法的枚举 |
| [[2.08 对象信息获取]] | `type` / `isinstance` / `dir` / `getattr` / `hasattr` |

### 2.5 错误、调试与测试

| 文章 | 主要内容 |
|---|---|
| [[2.02 异常处理]] | `try`/`except`/`finally`、自定义异常、`raise from` |
| [[5.06 调试技巧]] | print/assert/logging/pdb/IDE 断点选型与实战 |
| [[5.07 单元测试]] | `unittest.TestCase`、断言、`setUp`/`tearDown`、pytest 对比 |
| [[5.08 文档测试]] | doctest 写法、异常匹配、与 unittest 取舍 |

### 2.6 IO 与并发

| 文章 | 主要内容 |
|---|---|
| [[4.01 IO 编程]] | 文件读写、`with`、二进制文件、文件对象协议 |
| [[4.02 序列化]] | `pickle` vs `json`、自定义类 `default`/`object_hook`、`ensure_ascii` 坑 |
| [[4.03 并发编程]] | `threading` / `multiprocessing` / `asyncio`、GIL 影响 |
| [[4.04 资源池与连接池]] | 池化模式、数据库连接池、复用机制 |

### 2.7 常用内建模块

| 文章 | 主要内容 |
|---|---|
| [[5.01 正则表达式]] | `re` 模块、`\d`/`\w`/量词、分组、贪婪匹配、`re.compile`、`VERBOSE` |
| [[5.02 datetime 时间处理]] | 时间戳/时区/strptime-strftime/timedelta 加减 |
| [[5.03 collections 高级容器]] | namedtuple / deque / defaultdict / OrderedDict / ChainMap / Counter |
| [[5.04 itertools 迭代工具]] | count/cycle/chain/groupby/product/combinations |
| [[2.03 contextlib 上下文管理]] | `with` 协议、`@contextmanager`、`closing`、`suppress` |
| [[5.05 argparse 命令行参数]] | CLI 解析、位置/关键字参数、子命令、与 typer/click 对比 |
| [[5.09 密码学工具]] | base64 编码 / hashlib 哈希 / hmac 签名、salt、bcrypt 推荐 |
| [[5.12 struct 二进制处理]] | bytes ↔ int/float、字节序、网络协议头、BMP 解析 |
| [[5.10 urllib HTTP 客户端]] | urllib 基础、与 requests 对比、Session、异步选择 |
| [[5.11 XML 与 HTML 解析]] | SAX/ElementTree、HTMLParser、BeautifulSoup、lxml、JS 渲染应对 |

### 2.8 常用第三方模块

| 文章 | 主要内容 |
|---|---|
| [[5.14 常用第三方模块入门]] | Pillow 图像处理（缩放/滤镜/验证码）、chardet 编码检测、psutil 系统监控、pip 与 PyPI 速查 |

### 2.9 网络与服务

| 文章 | 主要内容 |
|---|---|
| [[5.13 网络编程]] | TCP/IP 基础、socket API、TCP 客户端/多线程服务器、UDP echo、send/recv 字节流陷阱、TIME_WAIT |
| [[4.05 数据库访问]] | DB-API 标准、SQLite/MySQL/PostgreSQL、参数化查询防 SQL 注入、SQLAlchemy ORM、session_scope、连接池配置 |

### 2.10 练习与作业

| 文章 | 主要内容 |
|---|---|
| [[练习与作业]] | 做题记录：题目 + 自己的实现 + 踩到的坑 + 修正版，每条坑回链知识点 |

---

## 3. 核心概念地图

哪些概念彼此关联得最紧密：

```text
                  ┌─ 装饰器与闭包
       函数对象 ──┼─ 高阶函数
                  └─ @property (在 OOP 里)
                       │
                       ▼
                  ┌─ 定制类 (dunder)
       类对象 ────┼─ metaclass (在 OOP 里)
                  └─ 对象信息获取 (反射)
                       │
                       ▼
                  ┌─ 列表生成式
       迭代协议 ──┼─ 迭代器
                  └─ 生成器 (惰性序列)
                       │
                       ▼
                  ┌─ with 语句 / 上下文
       资源管理 ──┼─ 异常处理
                  └─ 连接池
```

---

## 4. 与其他语言对照（同一概念怎么写）

经常切换 JS/TS 和 Python 的人最容易混淆的点：

| 概念 | Python | JavaScript |
|---|---|---|
| 箭头函数 | `lambda x: x*2` | `x => x*2` |
| 解构赋值 | `a, b = 1, 2` | `let [a, b] = [1, 2]` |
| 展开运算 | `f(*args, **kw)` | `f(...args)` |
| 私有变量 | `_name` 约定 / `__name` 改名 | `#name` (真私有) |
| 继承构造 | `super().__init__()` | `super()` |
| 多继承 | 支持，按 MRO 解析 | 不支持，用 mixin 函数 |
| 异步 | `async def` / `await` | `async function` / `await` |
| 字符串模板 | f-string `f"{x}"` | 反引号 `` `${x}` `` |

详细对照见各篇文章末尾的对比表格。

---

## 5. 还没整理的章节（TODO）

| 章节 | 主题 | 备注 |
|---|---|---|
| 16.13 | venv | 已被 [[0.01 环境管理]] 覆盖，跳过 |
| 20 | 电子邮件（SMTP / POP3） |  |
| 22 | Web 开发（HTTP / WSGI / Flask） |  |
| 23 | 异步 IO（协程 / asyncio / aiohttp） | 见下方 [[3.02 asyncio 异步编程]] 占位 |
| — | typing 类型提示（Optional / 泛型 / Protocol） | 🔴 公司栈必需，见 [[Python 学习路线]] Stage 3 |
| — | asyncio 异步编程（async/await / gather） | 🔴 公司栈必需，见 [[Python 学习路线]] Stage 3 |
| — | Pydantic v2（BaseModel / validator） | 🔴 公司栈必需，见 [[Python 学习路线]] Stage 3 |
| — | pytest（fixture / 参数化） | 🔴 公司栈必需，见 [[Python 学习路线]] Stage 3 |
| — | FastAPI（路由 / 依赖注入） | 🔴 公司栈必需，见 [[Python 学习路线]] Stage 4 |
| ~~—~~ | ~~uv 包管理~~ | ✅ 已整理 → [[0.05 uv]] |

---

## 参考资料

- 廖雪峰 Python 教程 https://liaoxuefeng.com/books/python/
- Python 官方文档 https://docs.python.org/3/
- Python 数据模型 https://docs.python.org/3/reference/datamodel.html
