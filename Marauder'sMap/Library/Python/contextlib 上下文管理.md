#Python #contextlib #with #资源管理

# contextlib 上下文管理

## TL;DR

**`with` 语句保证资源进入和退出时都执行特定代码——这就是上下文管理。** 核心要点：

- 任何实现了 `__enter__` 和 `__exit__` 的对象都能用在 `with` 里
- `@contextmanager` 装饰器 + `yield` 让你**用函数**写上下文管理器，省去定义类
- `closing(obj)` 把任何"有 `close()` 方法但没实现 `__exit__`"的对象包装成可 `with`
- `suppress(Exc)` 替代"try/except/pass"忽略特定异常
- `with` 不仅管文件——锁、事务、计时、临时改全局状态都是上下文场景

---

## 1. 为什么需要上下文管理

资源（文件、连接、锁）的典型用法是"用完必须释放"。手写 `try/finally` 很啰嗦：

```python
# 啰嗦的写法
try:
    f = open('/path', 'r')
    data = f.read()
finally:
    if f:
        f.close()
```

`with` 一行搞定，无论中间是否出错，`f.close()` 必然执行：

```python
with open('/path', 'r') as f:
    data = f.read()
# 出了 with 块，f 自动 close，不管有没有异常
```

---

## 2. 上下文管理协议：`__enter__` / `__exit__`

任何实现了这两个 dunder 的类都能用 `with`：

```python
class Query:
    def __init__(self, name):
        self.name = name

    def __enter__(self):
        print('Begin')                # 进入 with 时执行
        return self                   # as 后面拿到的就是这个

    def __exit__(self, exc_type, exc_value, traceback):
        if exc_type:
            print('Error:', exc_value)
        else:
            print('End')
        # 默认返回 None == False，异常会继续抛出
        # 返回 True 则吞掉异常（少用）

    def query(self):
        print('Query info about %s' % self.name)

with Query('Bob') as q:
    q.query()
# Begin
# Query info about Bob
# End
```

```text
with Query('Bob') as q:           ┐
    q.query()                     │  正常路径
                                  │
                  ┌─ __enter__()  ┘  返回 self → 赋给 q
       异常 ──────┤
                  └─ __exit__(异常信息)
                        ↓
                     False/None → 异常继续抛
                     True       → 吞掉异常
```

`__exit__` 的三个参数：异常类型、异常对象、traceback。没异常时全是 None。

---

## 3. `@contextmanager`：用函数写上下文管理器

写类太重，`@contextmanager` 装饰器 + 一个 `yield` 就够：

```python
from contextlib import contextmanager

@contextmanager
def tag(name):
    print('<%s>' % name)         # yield 前 = __enter__
    yield                         # yield 出去的值 = as 后面拿到的
    print('</%s>' % name)        # yield 后 = __exit__

with tag('h1'):
    print('hello')
    print('world')

# <h1>
# hello
# world
# </h1>
```

```text
@contextmanager
def my_cm():
    setup()         ┐
    yield value     │ ──── 这一行 yield 把 value 给 as 后面
    cleanup()       ┘

执行顺序：
  setup()
  ↓
  with 块里的代码（拿 value 用）
  ↓
  cleanup()        ← 不管有没有异常都会跑（要保证用 try/finally）
```

**异常感知版**（更稳健）：

```python
@contextmanager
def transaction(conn):
    try:
        yield conn
        conn.commit()             # 没出异常 → 提交
    except:
        conn.rollback()           # 出异常 → 回滚
        raise                     # 然后重新抛出
```

---

## 4. `closing`：把"有 close()" 的对象变成 with-able

很多老库的资源类只有 `close()` 方法但没实现 `__exit__`：

```python
from contextlib import closing
from urllib.request import urlopen

with closing(urlopen('https://www.python.org')) as page:
    for line in page:
        print(line)
# closing 自动在退出时调 page.close()
```

`closing` 的实现就是一个最简单的 contextmanager：

```python
@contextmanager
def closing(thing):
    try:
        yield thing
    finally:
        thing.close()
```

> 现代 `urllib.request.urlopen` 其实已经支持 `with`，例子是为了演示 `closing` 的用法。

---

## 5. `suppress`：优雅地忽略特定异常

```python
from contextlib import suppress

# 啰嗦写法
try:
    os.remove('/tmp/somefile')
except FileNotFoundError:
    pass

# 用 suppress
with suppress(FileNotFoundError):
    os.remove('/tmp/somefile')
```

意图更明显："我**知道**这个异常可能发生，我**故意**不管它。"

可以忽略多个异常类型：

```python
with suppress(FileNotFoundError, PermissionError):
    os.remove(path)
```

---

## 6. 其他常用工具

| 函数 | 用途 |
|---|---|
| `nullcontext()` | 占位的上下文管理器——啥也不做。需要"条件性 with" 时用 |
| `redirect_stdout(f)` | 把 print 输出重定向到 f |
| `redirect_stderr(f)` | 同上，针对 stderr |
| `ExitStack` | 在同一个 with 里动态注册多个清理函数 |

### `nullcontext` 的典型用法

```python
from contextlib import nullcontext

# 想要：有时持锁，有时不持
cm = lock if should_lock else nullcontext()

with cm:
    do_critical_section()
# 不用写两份代码
```

### `redirect_stdout`

```python
from contextlib import redirect_stdout
from io import StringIO

buf = StringIO()
with redirect_stdout(buf):
    print('hello')
    help(list)
buf.getvalue()                   # 'hello\n...help 输出...'
```

---

## 7. 实战：常见模式

### 计时器

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(name):
    start = time.time()
    yield
    print(f'{name}: {time.time() - start:.2f}s')

with timer('query'):
    db.execute('SELECT ...')
# query: 0.34s
```

### 临时改全局状态

```python
@contextmanager
def use_locale(loc):
    old = locale.getlocale()
    locale.setlocale(locale.LC_ALL, loc)
    try:
        yield
    finally:
        locale.setlocale(locale.LC_ALL, old)

with use_locale('en_US.UTF-8'):
    format_date(...)
# with 块外恢复原来的 locale
```

### 数据库事务

```python
@contextmanager
def transaction():
    conn = get_conn()
    try:
        yield conn
        conn.commit()
    except:
        conn.rollback()
        raise
    finally:
        conn.close()

with transaction() as conn:
    conn.execute('UPDATE ...')
```

---

## 8. 速查表

| 需求 | 用法 |
|---|---|
| 自动关闭文件 | `with open(...) as f:` |
| 自定义上下文（类风格）| 实现 `__enter__` / `__exit__` |
| 自定义上下文（函数风格）| `@contextmanager` + `yield` |
| 包装有 `close()` 的对象 | `with closing(obj) as o:` |
| 忽略特定异常 | `with suppress(Exc):` |
| 重定向 stdout | `with redirect_stdout(buf):` |
| 条件性 with | `with cm if cond else nullcontext():` |
| 动态多个 with | `with ExitStack() as stack: stack.enter_context(cm)` |

---

## 9. 与 JavaScript 对比

JS 没有 `with` 语法（旧的 `with(obj)` 是别的东西，已废弃）。资源管理靠 `try/finally`：

```javascript
const f = await open('/path');
try {
    await f.read();
} finally {
    await f.close();
}
```

将来 [Explicit Resource Management 提案](https://github.com/tc39/proposal-explicit-resource-management) 引入 `using` 语法时会更接近 Python：

```javascript
using f = await open('/path');     // 离开作用域自动 close
```

---

## 相关文章

- [[异常处理]] - `try` / `except` / `finally` 基础
- [[装饰器与闭包]] - `@contextmanager` 也是装饰器
- [[列表生成式]] - 生成器函数与 `yield`
- [[IO 编程]] - `with open()` 的标准用法
- [[定制类]] - `__enter__` / `__exit__` 也是 dunder

---

## 参考资料

- 廖雪峰 Python 教程 · contextlib https://liaoxuefeng.com/books/python/built-in-modules/contextlib/
- Python 官方文档 https://docs.python.org/3/library/contextlib.html
