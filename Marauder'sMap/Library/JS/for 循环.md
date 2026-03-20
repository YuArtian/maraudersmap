#JavaScript #循环 #迭代

# for 循环

## TL;DR

**JS 有多种 for 循环，核心区别在于遍历的是"键"还是"值"。** 核心要点：

- `for...of` 遍历**值**，用于数组、字符串、Map、Set 等可迭代对象
- `for...in` 遍历**键/索引**（属性名），会沿原型链查找，有坑
- 实践中遍历数组用 `for...of`，遍历对象用 `Object.keys()` + `for...of`

---

## 1. for...in vs for...of 对比

| | `for...in` | `for...of` |
|---|---|---|
| 遍历的是 | **键/索引**（属性名，字符串） | **值** |
| 适用对象 | 对象、数组 | 可迭代对象（数组、字符串、Map、Set） |
| 能遍历普通对象？ | 能 | 不能（普通对象不可迭代） |
| 会沿原型链查找？ | 会 | 不会 |

```javascript
const arr = ['a', 'b', 'c'];

for (const x in arr)  console.log(x);  // '0', '1', '2' -- 索引（字符串！）
for (const x of arr)  console.log(x);  // 'a', 'b', 'c' -- 值
```

---

## 2. for...in 的坑：原型链污染

`for...in` 会遍历**原型链上的可枚举属性**：

```javascript
Array.prototype.foo = 'bar';
const arr = [1, 2];

for (const x in arr) console.log(x);  // '0', '1', 'foo' -- 多出来了!
```

---

## 3. 最佳实践

| 场景 | 推荐写法 | 避免 |
|------|---------|------|
| 遍历数组 | `for (const x of arr)` | `for...in`（得到索引不是值，还有原型链问题） |
| 遍历字符串 | `for (const ch of str)` | `for...in` |
| 遍历 Map/Set | `for (const x of map)` | `for...in` |
| 遍历对象的 key | `for (const k of Object.keys(obj))` | `for...in`（原型链问题） |
| 遍历对象的 key+value | `for (const [k, v] of Object.entries(obj))` | `for...in` |

简单记：**几乎所有场景都用 `for...of`**，`for...in` 基本不用。

---

## 4. 与 Python 的对应关系

| Python | JavaScript | 遍历的是 |
|--------|-----------|---------|
| `for x in [1,2,3]` | `for (const x of [1,2,3])` | 值 |
| `for k in d` | `for (const k of Object.keys(obj))` | key |
| `for k, v in d.items()` | `for (const [k, v] of Object.entries(obj))` | key + value |
| `for i, x in enumerate(arr)` | `arr.forEach((x, i) => ...)` 或 `for (const [i, x] of arr.entries())` | 索引 + 值 |

Python 没有 JS `for...in`（遍历属性名）的对应物。

---

## 相关文章

- [[JS执行模型]] - JS 运行机制
- [[迭代与切片]] - Python 的迭代机制

---

## 参考资料

- MDN - for...of https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of
- MDN - for...in https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...in
