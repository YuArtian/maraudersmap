#Python #struct #二进制 #字节

# struct 二进制处理

## TL;DR

**`struct` 模块在 `bytes` 和 C 风格的整数/浮点之间互转，用一个**格式字符串**描述每个字段。** 核心要点：

- `struct.pack(fmt, *values)` → bytes；`struct.unpack(fmt, data)` → tuple
- 格式串第一个字符是**字节序**：`>` big-endian（网络序）、`<` little-endian、`!` 网络序、`=` 本机
- 字段类型用 `i` (4-byte int)、`H` (2-byte uint)、`f` (float)、`d` (double)、`s` (字符串) 等
- 不指定字节序时会按本机+对齐——**跨平台代码必须显式指定**
- 适合处理二进制协议、文件头（如 BMP/PNG）、Socket 字节流；用不着时优先 JSON / msgpack

---

## 1. 为什么需要 struct

Python 的 `int` 是任意精度，没有固定字节数。要把 32 位整数变成 4 字节 `bytes`，手算位运算非常痛苦：

```python
n = 10240099
b1 = (n & 0xff000000) >> 24
b2 = (n & 0x00ff0000) >> 16
b3 = (n & 0x0000ff00) >> 8
b4 =  n & 0x000000ff
bs = bytes([b1, b2, b3, b4])
# b'\x00\x9c@c'
```

浮点数就更没辙了。`struct` 一行搞定：

```python
import struct

struct.pack('>I', 10240099)
# b'\x00\x9c@c'
```

```text
'>I' 拆解:
 >   big-endian (网络字节序)
 I   4 字节无符号整数
```

---

## 2. `pack` / `unpack`

### `pack(fmt, *values)`：值 → bytes

```python
struct.pack('>I', 10240099)           # b'\x00\x9c@c'
struct.pack('>IH', 12345, 9)          # 一次 pack 多个: int + short
struct.pack('>2I', 100, 200)          # 2I = II，两个 int
```

### `unpack(fmt, data)`：bytes → tuple

```python
struct.unpack('>IH', b'\xf0\xf0\xf0\xf0\x80\x80')
# (4042322160, 32896)
```

返回**总是 tuple**，单字段也要 `(val,) = unpack(...)` 解包。

### 长度计算

`struct.calcsize(fmt)` 告诉你这个格式需要多少字节：

```python
struct.calcsize('>IH')           # 6 = 4 + 2
struct.calcsize('>IHHHII')       # 24
```

---

## 3. 字节序：`>` `<` `=` `!`

第一个字符指定字节序，**强烈建议永远显式写**：

| 前缀 | 字节序 | 对齐 |
|---|---|---|
| `>` | big-endian | 不填充 |
| `<` | little-endian | 不填充 |
| `!` | 网络序（同 big-endian）| 不填充 |
| `=` | 本机字节序 | 不填充 |
| `@`（默认）| 本机字节序 | C 编译器对齐 |
| 无前缀 | 同 `@` | 有填充 ⚠️ |

```text
大端 (big-endian): 高位字节在前
  0x12345678  →  12 34 56 78

小端 (little-endian): 低位字节在前  ← x86/x64 默认
  0x12345678  →  78 56 34 12
```

**跨平台代码或网络协议必须用 `>` / `<` / `!`**——否则同一段代码在 x86 和 ARM 上行为不同。

---

## 4. 字段类型表

| 格式 | C 类型 | Python 类型 | 字节数 |
|---|---|---|---|
| `c` | char | bytes (len 1) | 1 |
| `b` / `B` | signed/unsigned char | int | 1 |
| `h` / `H` | short / unsigned short | int | 2 |
| `i` / `I` | int / unsigned int | int | 4 |
| `l` / `L` | long / unsigned long | int | 4 |
| `q` / `Q` | long long / unsigned long long | int | 8 |
| `f` | float | float | 4 |
| `d` | double | float | 8 |
| `s` | char[] | bytes | N（前面带数字）|
| `?` | bool | bool | 1 |
| `x` | padding | — | 1 |

**记忆口诀**：小写有符号、大写无符号；`hilq` 长度 2/4/4/8。

### 字符串字段 `Ns`

字符串字段长度写在前面：

```python
struct.pack('>4s2s', b'abcd', b'12')          # b'abcd12'
struct.unpack('>4s2s', b'abcd12')              # (b'abcd', b'12')
```

不够长会用 `\x00` 补齐，过长会截断——务必检查实际长度。

---

## 5. 实战：解析 BMP 文件头

BMP 是经典的二进制格式，文件开头有固定结构：

```text
偏移  长度  字段
0     2    'BM' 或 'BA'（魔数）
2     4    文件大小
6     4    保留位（0）
10    4    实际图像偏移量
14    4    Header 字节数
18    4    图像宽度
22    4    图像高度
26    2    总是 1
28    2    每像素位数
```

读前 30 字节解析：

```python
s = b'\x42\x4d\x38\x8c\x0a\x00\x00\x00\x00\x00\x36\x00\x00\x00\x28\x00\x00\x00\x80\x02\x00\x00\x68\x01\x00\x00\x01\x00\x18\x00'

result = struct.unpack('<ccIIIIIIHH', s)
# (b'B', b'M', 691256, 0, 54, 40, 640, 360, 1, 24)
```

```text
'<ccIIIIIIHH' 拆解:
  <   小端 (BMP 用小端)
  cc  两个 char → 'B' 'M'
  I   4 字节无符号 → 文件大小 691256
  I   保留位 0
  I   图像偏移 54
  I   Header 字节数 40
  I   宽度 640
  I   高度 360
  H   1
  H   每像素 24 位
```

解读：BMP 文件、640×360、24 位色深。

### 包装成函数

```python
def parse_bmp(data: bytes) -> dict:
    if len(data) < 30:
        return None
    magic, _, size, _, _, _, width, height, _, color = struct.unpack('<ccIIIIIIHH', data[:30])
    if magic + _ not in (b'BM', b'BA'):
        return None
    return {'width': width, 'height': height, 'color': color}
```

---

## 6. 实战：处理网络协议

很多协议都是固定结构。比如简化的 TCP 头：

```python
# 模拟一个简单的 4 字节包头：1B 类型 + 1B 版本 + 2B 长度
def encode_header(msg_type: int, version: int, length: int) -> bytes:
    return struct.pack('!BBH', msg_type, version, length)
    # ! 网络序; B 1字节, B 1字节, H 2字节

def decode_header(data: bytes) -> tuple:
    return struct.unpack('!BBH', data[:4])

# 客户端发送
header = encode_header(1, 2, 1024)        # b'\x01\x02\x04\x00'
# 服务端解析
msg_type, version, length = decode_header(header)
# (1, 2, 1024)
```

---

## 7. `Struct` 类：预编译

如果同一格式反复用，预编译 `Struct` 对象：

```python
header_fmt = struct.Struct('!BBH')   # 编译一次

# 之后直接用，速度更快
data = header_fmt.pack(1, 2, 1024)
msg_type, ver, length = header_fmt.unpack(data)

# 也能告诉你长度
header_fmt.size                       # 4
```

类似 `re.compile` 之于正则——重复使用场景的标配。

---

## 8. `unpack_from` / `pack_into`：操作偏移

处理大文件时，不要切片整个 `bytes`，直接在原 buffer 上操作：

```python
# 从偏移 N 开始 unpack
header = struct.unpack_from('!BBH', data, offset=0)
body_len = header[2]
body = struct.unpack_from(f'!{body_len}s', data, offset=4)
```

```python
# 写入预分配的 buffer
buf = bytearray(64)
struct.pack_into('!BBH', buf, 0, 1, 2, 1024)    # 写入到 buf[0:4]
```

---

## 9. 什么时候用 struct？什么时候别用？

| 场景 | 用 struct？ |
|---|---|
| 解析二进制文件格式（BMP/PNG/PE/ELF）| ✓ 必备 |
| 实现网络协议（TCP/UDP 自定义包头）| ✓ 必备 |
| 嵌入式与硬件通信 | ✓ 必备 |
| 跨语言数据传输 | △ 双方约定好格式才行；通常用 protobuf/msgpack 更好 |
| 进程间传 Python 对象 | ✗ 用 [[序列化]] 里的 pickle |
| 给前端送 API 数据 | ✗ 用 JSON |
| 大数据列存 | ✗ 用 parquet/arrow |

**记忆口诀**：struct 是给**面向字节**的协议用的，不是给"对象序列化"用的。

---

## 10. 速查表

| 任务 | 写法 |
|---|---|
| int → 4 字节 big-endian | `struct.pack('>I', n)` |
| 4 字节 big-endian → int | `struct.unpack('>I', data)[0]` |
| 多字段一次性 | `struct.pack('>IHH', a, b, c)` |
| 计算大小 | `struct.calcsize(fmt)` |
| 预编译重用 | `s = struct.Struct(fmt)` |
| 不切片操作 | `pack_into` / `unpack_from` |
| 网络字节序 | `!` 前缀 |
| 跨平台 | 永远显式 `<` 或 `>`，**不要**省略 |

---

## 相关文章

- [[字符编码]] - `bytes` 和 `str` 的区别（struct 永远操作 bytes）
- [[IO 编程]] - 二进制文件读写 `open(path, 'rb')`
- [[密码学工具]] - 哈希算法的输入也是 bytes，常与 struct 配合
- [[序列化]] - 高层的对象 ↔ bytes 转换

---

## 参考资料

- 廖雪峰 Python 教程 · struct https://liaoxuefeng.com/books/python/built-in-modules/struct/
- Python 官方 struct https://docs.python.org/3/library/struct.html
