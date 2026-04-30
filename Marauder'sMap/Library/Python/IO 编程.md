#Python #IO #文件

# IO 编程

## TL;DR

**IO 就是程序和外部（磁盘、网络）之间的数据交换，Python 默认用同步 IO。** 核心要点：

- 文件读写用 `with open()` 语句，自动关闭文件
- 同步 IO 阻塞当前线程直到完成，异步 IO 不等待、先去做别的
- `StringIO` / `BytesIO` 在内存中读写，接口和文件一样
- file-like Object：只要有 `read()` 方法就行（鸭子类型）

---

## 1. 同步 IO vs 异步 IO

| 模式 | 类比 | 特点 |
|------|------|------|
| 同步 IO | 在收银台前等汉堡做好 | 简单，但等待时什么都干不了 |
| 异步 IO | 先去逛商场，做好了通知你 | 性能高，但编程复杂 |

Python 日常写脚本用同步就够了，只有高并发 Web 服务才需要异步。

---

## 2. 文件读写

**永远用 `with` 语句**，自动关闭文件，即使出错也不会泄漏：

```python
# 读文件
with open('file.txt', 'r', encoding='utf-8') as f:
    content = f.read()

# 写文件（覆盖）
with open('file.txt', 'w', encoding='utf-8') as f:
    f.write('hello')

# 追加
with open('file.txt', 'a') as f:
    f.write('追加内容')
```

`with` 等价于 `try...finally` 加 `f.close()`，但更简洁。

### 读取方式选择

| 方法 | 适用场景 |
|------|---------|
| `f.read()` | 小文件，一次读完 |
| `f.read(size)` | 大文件，分块读 |
| `f.readline()` | 逐行读 |
| `f.readlines()` | 配置文件，按行返回 list |
| `for line in f:` | 大文件逐行（最省内存） |

### 打开模式

| 模式 | 含义 |
|------|------|
| `'r'` | 读文本 |
| `'w'` | 写文本（覆盖） |
| `'a'` | 追加文本 |
| `'rb'` | 读二进制（图片、视频） |
| `'wb'` | 写二进制 |

### 编码问题

```python
# 读 GBK 编码文件
with open('gbk.txt', 'r', encoding='gbk') as f:
    content = f.read()

# 遇到编码错误就忽略
with open('bad.txt', 'r', encoding='gbk', errors='ignore') as f:
    content = f.read()
```

---

## 3. open() 不会开新线程/进程

`open()` 和 `f.read()` 都在**当前线程**执行，向操作系统发系统调用：

```text
Python 程序（当前线程）
    │
    │  open('file.txt', 'r')
    ↓
操作系统内核
    │  打开文件，返回文件描述符
    ↓
返回文件对象给 Python
```

整个过程没有新线程/进程，当前线程同步等待操作系统完成。

---

## 4. StringIO 和 BytesIO

在**内存中**读写，接口和文件一样：

```python
from io import StringIO, BytesIO

# StringIO -- 内存中读写字符串
f = StringIO()
f.write('hello world')
f.getvalue()   # 'hello world'

# BytesIO -- 内存中读写二进制
f = BytesIO()
f.write('中文'.encode('utf-8'))
f.getvalue()   # b'\xe4\xb8\xad\xe6\x96\x87'
```

---

## 5. file-like Object

Python 不要求必须是真正的文件，只要有 `read()` 方法就算 file-like Object：

- 磁盘文件（`open()` 返回的）
- `StringIO` / `BytesIO`
- 网络流
- 自定义对象

这就是[[迭代与切片]]中提到的**鸭子类型**：不看你是什么类型，看你能做什么。

---

## 6. 对比 JS

```python
# Python
with open('file.txt', 'r', encoding='utf-8') as f:
    content = f.read()
```

```javascript
// Node.js（同步）
const content = fs.readFileSync('file.txt', 'utf-8')

// Node.js（异步，默认推荐）
const content = await fs.promises.readFile('file.txt', 'utf-8')
```

| | Python | Node.js |
|---|---|---|
| 默认 | 同步 | 异步 |
| 为什么 | 脚本场景，阻塞无所谓 | Web 服务场景，阻塞会卡所有用户 |

---

## 相关文章

- [[异常处理]] - 文件操作常见的 `FileNotFoundError`、`UnicodeDecodeError`
- [[字符编码]] - UTF-8 / GBK 等编码问题
- [[并发编程]] - 同步/异步的深入讨论

---

## 参考资料

- 廖雪峰 Python 教程 - IO 编程 https://liaoxuefeng.com/books/python/
