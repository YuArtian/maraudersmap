#Python #HTTP #urllib #requests

# urllib HTTP 客户端

## TL;DR

**`urllib` 是 Python 内置的 HTTP 客户端——能用但啰嗦，实际项目几乎都用第三方 `requests`。** 核心要点：

- `urlopen(url)` 发 GET 请求；返回 file-like 对象，`.read()` 拿 bytes
- POST：用 `Request` 对象 + `data=...`（必须是 `bytes`）
- 加 header（如 `User-Agent`）：`req.add_header(key, value)`
- 拼接 query string 用 `urllib.parse.urlencode(dict)`
- 处理代理/认证：`build_opener` + `ProxyHandler` / `ProxyBasicAuthHandler`
- **现代实战**：写新代码直接 `pip install requests`，stdlib `urllib` 只在不能装包时用

---

## 1. urllib vs requests 速对比

```python
# urllib：发个 GET，拼参数都麻烦
from urllib import request, parse
url = 'https://api.example.com/search?' + parse.urlencode({'q': 'python'})
with request.urlopen(url) as f:
    data = f.read().decode('utf-8')

# requests：一行
import requests
data = requests.get('https://api.example.com/search', params={'q': 'python'}).text
```

| 维度 | `urllib` (stdlib) | `requests` (第三方) |
|---|---|---|
| 安装 | 内置 | `pip install requests` |
| API 易用度 | 啰嗦、bytes/str 混 | 简洁、自动处理 |
| 默认编码 | 自己 `.decode()` | 自动检测 |
| JSON | 自己 `json.loads()` | `r.json()` |
| Session 复用 | `OpenerDirector`（难用） | `requests.Session()` |
| 实际推荐 | 仅在无法装包时 | 默认选择 |

下面正文先讲 `urllib`（书里的官方内容），再讲 `requests` 实战。

---

## 2. urllib 基础：GET 请求

```python
from urllib import request

with request.urlopen('https://api.douban.com/v2/book/2129650') as f:
    data = f.read()
    print('Status:', f.status, f.reason)
    for k, v in f.getheaders():
        print(f'{k}: {v}')
    print('Data:', data.decode('utf-8'))
```

```text
Status: 200 OK
Server: nginx
Content-Type: application/json; charset=utf-8
...
Data: {"rating": ..., "author": [...]}
```

`urlopen` 返回的对象类似文件——支持 `read()`, `getheaders()`, `status`, `reason`。

### 带 query 参数

```python
from urllib import request, parse

params = parse.urlencode({'q': 'python', 'cat': 1001})
# 'q=python&cat=1001'

url = f'https://api.example.com/search?{params}'
with request.urlopen(url) as f:
    data = f.read()
```

`parse.urlencode(dict)` 自动处理 URL 编码（中文、特殊字符）。

---

## 3. 添加 HTTP 头：模拟浏览器

很多网站会检查 `User-Agent`，没有就拒绝。要加 header 必须用 `Request` 对象：

```python
from urllib import request

req = request.Request('https://www.douban.com/')
req.add_header('User-Agent',
               'Mozilla/6.0 (iPhone; CPU iPhone OS 8_0 like Mac OS X) '
               'AppleWebKit/536.26 (KHTML, like Gecko) '
               'Version/8.0 Mobile/10A5376e Safari/8536.25')

with request.urlopen(req) as f:
    print(f.read().decode('utf-8'))
```

模拟 iPhone 的 UA，豆瓣返回手机版页面。

---

## 4. POST 请求

POST 必须传 `data=` 参数，**必须是 `bytes`**：

```python
from urllib import request, parse

login_data = parse.urlencode([
    ('username', 'bob'),
    ('password', '123456'),
])

req = request.Request('https://example.com/login')
req.add_header('Content-Type', 'application/x-www-form-urlencoded')

with request.urlopen(req, data=login_data.encode('utf-8')) as f:
    result = f.read().decode('utf-8')
```

关键点：

| 步骤 | 注意 |
|---|---|
| 拼参数 | `parse.urlencode([...])` 返回 `str` |
| 编码 | `.encode('utf-8')` 转 `bytes` |
| 传给 urlopen | `data=` 参数 |
| 有 data 就是 POST | 不用显式指定 method |

---

## 5. 通过代理

```python
import urllib.request

proxy_handler = urllib.request.ProxyHandler({
    'http': 'http://proxy.example.com:3128/',
    'https': 'http://proxy.example.com:3128/',
})

# 带认证的代理
proxy_auth = urllib.request.ProxyBasicAuthHandler()
proxy_auth.add_password('realm', 'host', 'username', 'password')

opener = urllib.request.build_opener(proxy_handler, proxy_auth)
with opener.open('http://www.example.com/') as f:
    data = f.read()
```

`build_opener` 把多个 handler 组合成一个"会处理代理 + 认证"的客户端。

---

## 6. 异常处理

`urllib` 用 `HTTPError` / `URLError` 表示请求失败：

```python
from urllib import request, error

try:
    with request.urlopen('https://nonexistent.example/') as f:
        data = f.read()
except error.HTTPError as e:
    print('HTTP error:', e.code, e.reason)   # 4xx / 5xx
except error.URLError as e:
    print('Network error:', e.reason)         # 连不上、DNS 失败等
```

注意：**4xx/5xx 状态码会抛 `HTTPError`**——和 `requests` 默认不同（requests 不抛，要主动 `r.raise_for_status()`）。

---

## 7. 现代实战：用 requests

```python
import requests

# GET
r = requests.get('https://api.douban.com/v2/book/2129650')
r.status_code           # 200
r.text                  # 自动解码的字符串
r.content               # 原始 bytes
r.json()                # 自动 json.loads
r.headers               # dict-like

# 带 query
r = requests.get('https://api.example.com/search',
                 params={'q': 'python', 'cat': 1001})
r.url                   # 'https://api.example.com/search?q=python&cat=1001'

# 自定义 header
r = requests.get('https://www.douban.com/',
                 headers={'User-Agent': 'Mozilla/5.0 ...'})

# POST 表单
r = requests.post('https://example.com/login',
                  data={'username': 'bob', 'password': '123'})

# POST JSON
r = requests.post('https://api.example.com/users',
                  json={'name': 'Bob', 'age': 20})
# 自动序列化 + Content-Type: application/json

# 上传文件
r = requests.post('https://example.com/upload',
                  files={'file': open('report.pdf', 'rb')})

# 超时（生产代码必须设！）
r = requests.get('https://slow.example/', timeout=5)
```

### Session：复用连接 + 共享 Cookie

```python
session = requests.Session()
session.headers['Authorization'] = 'Bearer token123'

session.get('https://api.example.com/users/me')
session.post('https://api.example.com/items', json={...})
# 自动复用 TCP 连接，共享 cookies
```

跑爬虫、多次调同一个 API 必用 Session——比每次 `requests.get` 快很多。

### 错误处理

```python
try:
    r = requests.get(url, timeout=5)
    r.raise_for_status()              # 4xx/5xx 抛 HTTPError
    data = r.json()
except requests.Timeout:
    ...
except requests.HTTPError as e:
    print(e.response.status_code)
except requests.RequestException:
    ...   # 网络层错误
```

`requests` 默认**不**对 4xx/5xx 抛错——必须显式 `raise_for_status()`。这点跟 `urllib` 行为相反，是新人最容易踩的坑。

---

## 8. 现代异步：httpx

需要 `async/await` 异步并发 HTTP 时用 `httpx`（API 和 requests 几乎一样）：

```python
import httpx

# 同步：和 requests 一模一样
r = httpx.get('https://api.example.com/')

# 异步
async with httpx.AsyncClient() as client:
    r = await client.get('https://api.example.com/')
```

| 库 | 何时用 |
|---|---|
| `urllib` | 不能装第三方包的环境 |
| `requests` | **默认选择**，同步、生态最全 |
| `httpx` | 需要 async 或要 HTTP/2 |
| `aiohttp` | 纯 async，更贴近底层 |

---

## 9. 速查表

| 任务 | urllib | requests |
|---|---|---|
| GET | `urlopen(url)` | `requests.get(url)` |
| 带 query | 手动 `urlencode` 拼接 | `params={...}` |
| POST 表单 | `Request` + `data=bytes` | `data={...}` |
| POST JSON | 手动 `json.dumps` + `add_header` | `json={...}` |
| 上传文件 | 复杂，手写 multipart | `files={...}` |
| 自定义 Header | `req.add_header(k, v)` | `headers={...}` |
| 拿响应文本 | `f.read().decode('utf-8')` | `r.text` |
| 拿 JSON | `json.loads(f.read())` | `r.json()` |
| 看状态码 | `f.status` | `r.status_code` |
| 超时 | `urlopen(url, timeout=5)` | `requests.get(url, timeout=5)` |
| 复用连接 | `OpenerDirector`（难用）| `requests.Session()` |

---

## 10. 几条工程实践

| 建议 | 原因 |
|---|---|
| **永远设 timeout** | 不设就可能无限等，挂死整个程序 |
| **错误必须处理** | 网络永远可能挂——别假设 200 |
| **API 调用复用 Session** | TCP 握手很贵，单次握手 + 多次请求 >> 多次握手 |
| **生产用 requests/httpx** | urllib 啰嗦易错，stdlib 不是唯一选项 |
| **解析 HTML 用 BeautifulSoup/lxml** | requests 拿到 HTML，**别用正则**解析它 |
| **大文件用 stream** | `requests.get(url, stream=True)` + 迭代 `iter_content` |
| **HTTPS 证书** | 默认验证；`verify=False` 关验证仅限调试 |

---

## 相关文章

- [[序列化]] - `requests.get(...).json()` 背后是 `json.loads`
- [[字符编码]] - `r.text` 自动检测编码，`r.content.decode(...)` 手动
- [[XML与HTML解析]] - 拿到 HTML 后怎么解析
- [[并发编程]] - 多个 HTTP 请求并发（线程 / async）
- [[环境变量与配置]] - API key 应该放环境变量

---

## 参考资料

- 廖雪峰 Python 教程 · urllib https://liaoxuefeng.com/books/python/built-in-modules/urllib/
- 廖雪峰 Python 教程 · requests https://liaoxuefeng.com/books/python/third-party-modules/requests/
- Python 官方 urllib https://docs.python.org/3/library/urllib.html
- requests 文档 https://docs.python-requests.org/
- httpx 文档 https://www.python-httpx.org/
