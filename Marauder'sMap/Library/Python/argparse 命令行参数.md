#Python #argparse #CLI #命令行

# argparse 命令行参数

## TL;DR

**`argparse` 把命令行参数解析、类型转换、帮助文档、错误提示一次性搞定——别再手写 `sys.argv`。** 核心要点：

- 创建 `ArgumentParser` → `add_argument` 定义每个参数 → `parse_args()` 返回带属性的 namespace
- **位置参数**直接写名字（如 `outfile`），**关键字参数**带 `--`（如 `--host`）
- `type=int` 自动转类型，`required=True` 强制必填，`default=...` 给默认值
- `action='store_true'` 让"出现即真"的 flag（如 `--verbose`）
- 自动生成 `-h` / `--help` 帮助信息，参数错误时报告清晰的错误消息

实际项目通常用更现代的 `typer` 或 `click`——基于装饰器，但 argparse 是 stdlib 自带、无依赖。

---

## 1. 不用 argparse 的痛苦

```python
# copy.py
import sys
source = sys.argv[1]                # 没传？IndexError
target = sys.argv[2]                # 没指定类型，永远是 str
# TODO: 处理 --recursive、--filter '*.py' 等参数...
```

手写解析这些复杂参数能写一百行 if/else，而且没有自动 `-h`、没有错误提示——重新发明轮子。

---

## 2. 最简示例

```python
# backup.py
import argparse

def main():
    parser = argparse.ArgumentParser(
        prog='backup',                     # 程序名
        description='Backup MySQL database.',
        epilog='Copyright (c) 2024',       # 帮助末尾
    )

    # 位置参数：必填，无前缀
    parser.add_argument('outfile')

    # 关键字参数：带 -- 或 -
    parser.add_argument('--host', default='localhost')
    parser.add_argument('--port', default=3306, type=int)   # 强制 int
    parser.add_argument('-u', '--user', required=True)       # 必填
    parser.add_argument('-p', '--password', required=True)
    parser.add_argument('--database', required=True)

    # 布尔 flag：出现即 True
    parser.add_argument('-gz', '--gzcompress', action='store_true',
                        help='Compress backup files by gz.')

    args = parser.parse_args()             # 解析当前进程的 sys.argv

    # 直接当属性访问
    print(f'outfile = {args.outfile}')
    print(f'host = {args.host}')
    print(f'port = {args.port}')           # 已经是 int 了

if __name__ == '__main__':
    main()
```

调用：

```bash
$ ./backup.py -u root -p hello --database testdb backup.sql
outfile = backup.sql
host = localhost
port = 3306
user = root
database = testdb
gzcompress = False
```

参数错或缺失，argparse 自动报错并退出：

```bash
$ ./backup.py --database testdb backup.sql
usage: backup [-h] [--host HOST] [--port PORT] -u USER -p PASSWORD --database DATABASE outfile
backup: error: the following arguments are required: -u/--user, -p/--password
```

自动支持 `-h`：

```bash
$ ./backup.py -h
usage: backup [-h] [--host HOST] [--port PORT] -u USER -p PASSWORD --database DATABASE outfile

Backup MySQL database.

positional arguments:
  outfile

optional arguments:
  -h, --help              show this help message and exit
  --host HOST
  --port PORT
  -u USER, --user USER
  -p PASSWORD, --password PASSWORD
  --database DATABASE
  -gz, --gzcompress       Compress backup files by gz.

Copyright (c) 2024
```

---

## 3. `add_argument` 参数详解

```python
parser.add_argument(
    name_or_flags,       # 'outfile' 位置参数，'--host' 关键字参数
    type=int,            # 自动转类型: int, float, str (默认), 自定义函数
    default='localhost', # 默认值
    required=True,       # 必填（仅对关键字参数）
    nargs='+',           # 接收多个值
    choices=['a', 'b'],  # 限定取值
    action='store_true', # 行为：store/store_true/store_false/append/count
    help='说明文本',     # -h 显示
    metavar='HOST',      # -h 里用什么名字显示参数（默认大写参数名）
    dest='host_name',    # 解析后 namespace 上叫什么名
)
```

### 几个常用 `action`

| action | 作用 |
|---|---|
| `'store'`（默认）| 把后面的值存起来 |
| `'store_true'` | 出现即 `True`，不出现即 `False`，**不需要值** |
| `'store_false'` | 反过来 |
| `'append'` | 多次出现就合并成 list，比如 `-x 1 -x 2` → `[1, 2]` |
| `'count'` | 计数，比如 `-vvv` → `3`，常用于日志级别 |

```python
# 多次 -v 提升详细度
parser.add_argument('-v', '--verbose', action='count', default=0)

# 多次 -i 收集 list
parser.add_argument('-i', '--input', action='append')
# ./prog -i a -i b -i c  →  args.input = ['a', 'b', 'c']
```

### `nargs`：接收多个值

| nargs | 含义 |
|---|---|
| 整数 N | 恰好 N 个值 |
| `'?'` | 0 个或 1 个 |
| `'*'` | 0 个或多个 |
| `'+'` | 1 个或多个 |

```python
parser.add_argument('files', nargs='+')   # 至少 1 个文件
# ./prog a.txt b.txt c.txt → args.files = ['a.txt', 'b.txt', 'c.txt']
```

---

## 4. 子命令：像 `git` 一样

```python
parser = argparse.ArgumentParser(prog='mytool')
subparsers = parser.add_subparsers(dest='command', required=True)

# git add
parser_add = subparsers.add_parser('add', help='Add files')
parser_add.add_argument('files', nargs='+')

# git commit
parser_commit = subparsers.add_parser('commit', help='Commit changes')
parser_commit.add_argument('-m', '--message', required=True)

args = parser.parse_args()

if args.command == 'add':
    do_add(args.files)
elif args.command == 'commit':
    do_commit(args.message)
```

```bash
$ mytool add a.txt b.txt
$ mytool commit -m "fix bug"
$ mytool -h                    # 看所有子命令
$ mytool commit -h             # 看 commit 子命令的参数
```

适合做一个工具内有多个独立动作的 CLI，比如 `docker run` / `docker build` / `docker pull`。

---

## 5. 类型转换：`type` 不只是 int/float

```python
parser.add_argument('--port', type=int)
parser.add_argument('--rate', type=float)

# 用任意可调用对象做转换
parser.add_argument('--cfg', type=open)          # 自动打开文件！
parser.add_argument('--date', type=lambda s: datetime.strptime(s, '%Y-%m-%d'))
```

转换失败 argparse 自动报错：

```bash
$ prog --port abc
error: argument --port: invalid int value: 'abc'
```

---

## 6. 实战模式

### 模式 1：从环境变量+命令行双来源

```python
import os, argparse

parser = argparse.ArgumentParser()
parser.add_argument('--api-key', default=os.environ.get('API_KEY'), required=True)
# 优先级: 命令行 > 环境变量 > 报错
```

### 模式 2：互斥参数组

```python
group = parser.add_mutually_exclusive_group()
group.add_argument('--verbose', action='store_true')
group.add_argument('--quiet', action='store_true')
# -v 和 -q 不能同时给
```

### 模式 3：分组显示更清晰

```python
group_conn = parser.add_argument_group('connection options')
group_conn.add_argument('--host')
group_conn.add_argument('--port', type=int)

group_auth = parser.add_argument_group('authentication')
group_auth.add_argument('--user')
group_auth.add_argument('--password')
# -h 输出按 "connection options" / "authentication" 分组显示
```

---

## 7. argparse vs 替代方案

| 工具 | 何时用 |
|---|---|
| **argparse** | 简单脚本、stdlib（无依赖）|
| **[typer](https://typer.tiangolo.com/)** | 现代项目，基于装饰器和类型注解，体验最好 |
| **[click](https://click.palletsprojects.com/)** | 复杂 CLI（子命令多、需要彩色输出），Flask 作者出品 |
| **[fire](https://github.com/google/python-fire)** | 极速原型——一行 `fire.Fire(my_func)` 把函数变 CLI |

### typer 风格预览

```python
import typer

def backup(outfile: str, host: str = 'localhost', port: int = 3306):
    """Backup MySQL database."""
    print(f'{outfile=}, {host=}, {port=}')

if __name__ == '__main__':
    typer.run(backup)
```

类型注解 = 自动转换 + 帮助生成，**比 argparse 简短 50%**。

---

## 8. 速查表

| 需求 | 写法 |
|---|---|
| 位置参数 | `add_argument('name')` |
| 关键字参数 | `add_argument('--name')` |
| 简写 | `add_argument('-n', '--name')` |
| 必填 | `required=True` |
| 默认值 | `default=X` |
| 类型转换 | `type=int` |
| Bool flag | `action='store_true'` |
| 多值参数 | `nargs='+'` 或 `nargs='*'` |
| 限定取值 | `choices=['a','b','c']` |
| 解析 | `args = parser.parse_args()` |
| 取值 | `args.name`（属性访问）|

---

## 相关文章

- [[模块与包]] - `if __name__ == '__main__'` 配合 CLI 入口
- [[环境变量与配置]] - 命令行 + 环境变量 + 配置文件三层组合
- [[异常处理]] - argparse 内部用 `SystemExit`，普通 `except` 抓不到
- [[collections 高级容器]] - `ChainMap` 实现配置优先级查找

---

## 参考资料

- 廖雪峰 Python 教程 · argparse https://liaoxuefeng.com/books/python/built-in-modules/argparse/
- Python 官方 argparse https://docs.python.org/3/library/argparse.html
- typer https://typer.tiangolo.com/
