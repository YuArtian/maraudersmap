#Python #collections #数据结构

# collections 高级容器

## TL;DR

**`collections` 模块提供六种"加强版"容器，解决普通 list/dict/tuple 的常见痛点。** 速查：

| 类 | 解决什么问题 |
|---|---|
| `namedtuple` | tuple 字段没名字 → 让 `p.x` 代替 `p[0]` |
| `deque` | list 从头部插入慢（O(n)）→ 双端队列 O(1) |
| `defaultdict` | dict 访问不存在的 key 抛 KeyError → 自动建默认值 |
| `OrderedDict` | （Python 3.7 前 dict 无序）→ 保持插入顺序 |
| `ChainMap` | 多个 dict 按优先级查 → 配置层级覆盖（命令行 > 环境变量 > 默认） |
| `Counter` | 一行统计每个元素出现次数 |

---

## 1. `namedtuple`：有字段名的 tuple

普通 tuple 表示坐标，看不出是啥：

```python
p = (1, 2)        # 这是坐标？日期？版本号？谁知道
```

`namedtuple` 像轻量级 class，只有数据没行为：

```python
from collections import namedtuple

Point = namedtuple('Point', ['x', 'y'])
p = Point(1, 2)

p.x                # 1   -- 用属性访问
p.y                # 2
p[0]               # 1   -- 兼容下标
x, y = p           # 也能解包

isinstance(p, Point)    # True
isinstance(p, tuple)    # True  -- 它本质还是 tuple
```

| 对比 | namedtuple | 普通 class |
|---|---|---|
| 不可变 | ✓ | 需要手动加 `__slots__` 或冻结 |
| 自带 `__repr__` | ✓ | 要手写 |
| 行数 | 1 行 | 至少 4-5 行 |
| 想加方法 | 不方便 | class 更合适 |

**何时用**：只需要"打包几个字段"，不需要方法时——比纯 tuple 更可读，比 class 更省事。

更现代的替代：`dataclass`（Python 3.7+），可变、可加方法。

---

## 2. `deque`：双端高效队列

list 从尾部添加 O(1)，从头部添加是 O(n)（要移动所有元素）。`deque` 两端都是 O(1)：

```python
from collections import deque

q = deque(['a', 'b', 'c'])
q.append('x')                  # 尾部加：deque(['a','b','c','x'])
q.appendleft('y')              # 头部加：deque(['y','a','b','c','x'])
q.pop()                        # 'x'，尾部弹
q.popleft()                    # 'y'，头部弹
```

```text
list:    [a, b, c, ...]      ↑ 加在右边 O(1)
                              ↓ 加在左边 O(n) 要全部右移
deque:   ...  a  b  c  ...   ← 两端都 O(1)
```

**何时用**：
- 实现队列（FIFO）：`append` + `popleft`
- 实现栈（LIFO）：`append` + `pop`（其实 list 也行）
- 固定长度滑动窗口：`deque(maxlen=N)` 自动丢弃溢出的元素

---

## 3. `defaultdict`：自动建默认值

普通 dict 访问不存在的 key 会抛 `KeyError`：

```python
counts = {}
counts['apple'] += 1                # KeyError: 'apple'
```

`defaultdict(工厂函数)` 在 key 不存在时调用工厂函数创建默认值：

```python
from collections import defaultdict

counts = defaultdict(int)           # 默认值是 int() = 0
counts['apple'] += 1                # 自动从 0 开始，apple = 1
counts['apple'] += 1                # apple = 2
```

常用工厂：

| 工厂 | 默认值 |
|---|---|
| `int` | 0 |
| `list` | `[]` |
| `set` | `set()` |
| `dict` | `{}` |
| `lambda: 'N/A'` | 自定义任意值 |

经典场景——按 key 分组：

```python
data = [('a', 1), ('b', 2), ('a', 3), ('b', 4)]
groups = defaultdict(list)

for k, v in data:
    groups[k].append(v)             # 不需要先判断 key 是否存在

# defaultdict(<class 'list'>, {'a': [1, 3], 'b': [2, 4]})
```

对比 `dict.setdefault`：

```python
# 等价但更啰嗦
groups = {}
for k, v in data:
    groups.setdefault(k, []).append(v)
```

---

## 4. `OrderedDict`：保持插入顺序

> Python 3.7 起，**普通 dict 也保持插入顺序**了——这是语言规范的一部分，不是实现细节。

但 `OrderedDict` 仍然有用：

```python
from collections import OrderedDict

od = OrderedDict()
od['z'] = 1
od['y'] = 2
od['x'] = 3
list(od.keys())                     # ['z', 'y', 'x']  -- 按插入顺序
```

**和普通 dict 的区别**：

| 特性 | 普通 dict (3.7+) | OrderedDict |
|---|---|---|
| 保持插入顺序 | ✓ | ✓ |
| `==` 比较考虑顺序 | ✗（只比键值对）| ✓（顺序不同就不等）|
| `popitem(last=False)` 弹出最早元素 | ✗ | ✓ |
| `move_to_end()` 重排 | ✗ | ✓ |

**何时还用 OrderedDict**：实现 LRU 缓存这种"顺序敏感"的数据结构。

---

## 5. `ChainMap`：分层配置查找

把多个 dict 串成一个逻辑 dict，按顺序查找：

```python
from collections import ChainMap

defaults = {'color': 'red', 'user': 'guest'}
env_args = {'user': 'admin'}            # 来自环境变量
cli_args = {'color': 'green'}           # 来自命令行

combined = ChainMap(cli_args, env_args, defaults)

combined['color']                       # 'green'  -- cli_args 优先
combined['user']                        # 'admin'  -- 命令行没有，落到环境变量
```

```text
查找顺序：
      cli_args    env_args    defaults
        ↓ no         ↓ yes
查 'user':     ────────→  'admin'   (env_args 有就用)

        ↓ yes
查 'color': 'green'                  (cli_args 有就用)
```

**典型场景**：应用配置——命令行 > 环境变量 > 配置文件 > 默认值，每层是一个 dict，用 ChainMap 一行串起来。

写操作只影响**第一个** dict：

```python
combined['size'] = 100      # 只加到 cli_args，不会污染 defaults
```

---

## 6. `Counter`：统计计数

最常用的 collections 类。一行统计每个元素出现次数：

```python
from collections import Counter

c = Counter('programming')
# Counter({'r': 2, 'g': 2, 'm': 2, 'p': 1, 'o': 1, 'a': 1, 'i': 1, 'n': 1})

c.update('hello')                # 累加另一个序列
c['o']                           # 2
c.most_common(3)                 # [('m', 2), ('o', 2), ('r', 2)]
```

应用：

```python
# 统计单词频率
words = open('article.txt').read().split()
Counter(words).most_common(10)

# 数组里出现次数最多的元素
Counter([1, 2, 3, 2, 1, 1, 4]).most_common(1)   # [(1, 3)]

# 算两个 multiset 的差
Counter('aabbc') - Counter('abc')   # Counter({'a': 1, 'b': 1})
```

`Counter` 是 `dict` 的子类，所以也能像 dict 一样用。**比手写 for 循环 + `+= 1` 优雅得多。**

---

## 7. 速查表

| 需求 | 类 | 替代 |
|---|---|---|
| 字段命名的 tuple | `namedtuple` | `dataclass` (3.7+) |
| 高效双端队列 | `deque` | — |
| key 不存在自动建 | `defaultdict` | `dict.setdefault` |
| 有序 dict | `OrderedDict` | 普通 dict (3.7+) |
| 多层配置查找 | `ChainMap` | 手写多个 if-elif |
| 元素计数 | `Counter` | 手写 for 循环 |

---

## 相关文章

- [[数据类型基础]] - list / dict / tuple 基础
- [[Set 集合]] - 另一种容器，可哈希去重
- [[迭代与切片]] - 所有容器都能迭代

---

## 参考资料

- 廖雪峰 Python 教程 · collections https://liaoxuefeng.com/books/python/built-in-modules/collections/
- Python 官方文档 https://docs.python.org/3/library/collections.html
