#Python #JavaScript #Set #数据结构

# Set 集合

## TL;DR

**Python 的 set 按内容去重，JS 的 Set 按引用去重**。Python 因此禁止放入可变对象（list、dict），JS 什么都能放但内容相同的两个数组不会被当成重复。

---

## 1. 基本用法

```python
s = set()
s.add(1)
s.add(2)
s.add(1)       # 重复，不会加进去
print(s)       # {1, 2}
```

也可以从 list 创建：

```python
s = set([1, 2, 2, 3])   # {1, 2, 3}，自动去重
```

---

## 2. 不能放可变对象

```python
>>> s = set()
>>> s.add([1, 2, 3])
TypeError: unhashable type: 'list'
```

原因：set 内部用**哈希表**存数据，需要对元素算哈希值来快速查找和去重。list 是可变的，如果允许放进去：

```text
1. 放入 [1, 2, 3]，根据内容算哈希值，存到位置 A
2. 之后修改成 [1, 2, 999]
3. 内容变了，但还存在位置 A → 查找时按新内容算哈希，去了位置 B → 找不到了
```

所以 Python 直接禁止可变对象放进 set。

### 能放 vs 不能放

| 能放进 set | 不能放进 set |
|-----------|-------------|
| `int`, `float`, `str` | `list` |
| `tuple`（元素也不可变时） | `dict` |
| `bool`, `None` | `set` 本身 |

dict 也是同样的限制——key 必须是不可变对象。

---

## 3. Python set vs JS Set

JS 的 Set 可以放任何东西，包括数组和对象：

```javascript
const s = new Set()
s.add([1, 2, 3])    // 没问题
s.add({a: 1})       // 也没问题
```

但去重方式完全不同：

```javascript
const s = new Set()
s.add([1, 2, 3])
s.add([1, 2, 3])
s.size  // 2，不会去重！因为是两个不同的引用
```

| | Python set | JS Set |
|---|---|---|
| 去重依据 | 哈希值（按**内容**判断） | 引用（按**是不是同一个对象**判断） |
| 能放可变对象 | 不能 | 能，但内容相同也不去重 |

```javascript
// JS：同一个引用才算重复
const arr = [1, 2, 3]
s.add(arr)
s.add(arr)
s.size  // 1，同一个引用，去重了
```

Python 选择了"按内容去重"，所以必须禁止可变对象（内容会变，哈希就乱了）。JS 选择了"按引用去重"，所以什么都能放，但两个内容一样的数组不会被当成重复。各有取舍。

---

## 相关文章

- [[数据类型基础]] - tuple、list 等基本类型
- [[Map]] - JS 的 Map 数据结构

---

## 参考资料

- 廖雪峰 Python 教程 - 使用 set
