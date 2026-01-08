#database #AI #RAG #vectordb

# Vector Database（向量数据库）

> 向量数据库是 [[RAG]] 系统的核心组件之一，负责存储和检索文本的向量表示。

## TL;DR

**向量数据库** 是专门用于存储和检索高维向量的数据库系统：

- **核心能力**: 相似度搜索（找到最相似的向量）
- **与传统数据库的区别**: 传统数据库精确匹配，向量数据库近似匹配
- **典型流程**: `文本 → Embedding → 向量 → 存储 → 相似度检索`
- **选型关键**: 数据规模、是否需要服务、性能要求、运维成本

---

## 1. 什么是向量？

### 1.1 向量的概念

文本通过 Embedding Model 转换为一组数字（向量），这组数字捕捉了文本的**语义含义**：

```text
"今天天气真好" → [0.12, -0.34, 0.56, ..., 0.78]  # 1536个数字
"今日阳光明媚" → [0.11, -0.33, 0.55, ..., 0.77]  # 很相似！
"我要买手机"   → [-0.45, 0.22, -0.11, ..., 0.33]  # 完全不同
```

### 1.2 相似度计算

| 方法 | 描述 | 适用场景 |
|------|------|----------|
| **Cosine Similarity** | 计算向量夹角的余弦值 | 最常用，对向量长度不敏感 |
| **Dot Product** | 向量点积 | 归一化向量时等同于余弦相似度 |
| **Euclidean Distance** | 欧氏距离 | 关注绝对距离 |

---

## 2. 数据库运行模式

### 2.1 嵌入式 vs 客户端-服务端

```text
┌─────────────────────────────────────────────────────────────────┐
│                     嵌入式数据库 (Embedded)                       │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │                   你的程序 (Python)                      │    │
│  │                                                        │    │
│  │   ┌──────────────────────────────────────────────┐    │    │
│  │   │        数据库（作为库嵌入程序中）                 │    │    │
│  │   │        直接读写本地文件                         │    │    │
│  │   └──────────────────────────────────────────────┘    │    │
│  │                                                        │    │
│  └────────────────────────────────────────────────────────┘    │
│                              ↓                                  │
│                       ./data/ (本地文件)                        │
│                                                                 │
│  ✅ 无需启动服务    ✅ 部署简单    ❌ 单进程    ❌ 无法远程访问    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                 客户端-服务端数据库 (Client-Server)               │
│                                                                 │
│  ┌─────────────┐          网络           ┌─────────────────┐   │
│  │  你的程序    │  ────────────────────→  │   数据库服务      │   │
│  │  (客户端)    │  ←────────────────────  │   (独立进程)      │   │
│  └─────────────┘                         │   port: 19530    │   │
│                                          └─────────────────┘   │
│                                                                 │
│  ✅ 支持多客户端    ✅ 可远程访问    ❌ 需要运维    ❌ 配置复杂     │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 模式对比

| 特性 | 嵌入式 | 客户端-服务端 |
|------|--------|--------------|
| 启动方式 | 直接 `python main.py` | 先启动服务，再运行程序 |
| 部署复杂度 | ⭐ 简单 | ⭐⭐⭐ 复杂 |
| 并发访问 | 单进程 | 支持多客户端 |
| 远程访问 | ❌ 不支持 | ✅ 支持 |
| 数据位置 | 本地文件 | 服务端 |
| 适合场景 | 个人项目、本地应用 | 团队协作、生产环境 |

---

## 3. 主流向量数据库对比

### 3.1 嵌入式数据库

#### ChromaDB

> <https://www.trychroma.com/>

```python
import chromadb

# 零配置，直接使用
client = chromadb.PersistentClient(path="./chroma_data")
collection = client.get_or_create_collection("my_kb")

# 添加数据
collection.add(
    documents=["今天天气真好"],
    embeddings=[[0.12, -0.34, ...]],
    ids=["doc1"]
)

# 搜索
results = collection.query(
    query_embeddings=[[0.13, -0.32, ...]],
    n_results=5
)
```

| 优点 | 缺点 |
|------|------|
| ✅ 零配置，pip install 即用 | ❌ 数据量大时性能下降 |
| ✅ Python 友好 | ❌ 不支持分布式 |
| ✅ 自带持久化 | ❌ 单进程锁定 |

**适合**: 快速原型、个人项目、< 10万条数据

#### LanceDB

> <https://lancedb.com/>

```python
import lancedb

db = lancedb.connect("./lancedb")
table = db.create_table("documents", data=[
    {"text": "今天天气真好", "vector": [0.12, -0.34, ...]}
])

results = table.search([0.13, -0.32, ...]).limit(5).to_list()
```

| 优点 | 缺点 |
|------|------|
| ✅ 嵌入式，零配置 | ❌ 较新，社区小 |
| ✅ 支持多模态 | ❌ 功能还在完善 |
| ✅ 与 pandas 兼容好 | |

**适合**: 快速原型、多模态场景、数据分析

#### SQLite + sqlite-vss

> <https://github.com/asg017/sqlite-vss>

```python
import sqlite3
import sqlite_vss

db = sqlite3.connect("index.db")
db.enable_load_extension(True)
sqlite_vss.load(db)

db.execute("CREATE VIRTUAL TABLE vss_docs USING vss0(embedding(1536))")
```

| 优点 | 缺点 |
|------|------|
| ✅ 纯本地，一个文件 | ❌ 扩展安装可能有兼容性问题 |
| ✅ 无需服务 | ❌ 功能较少 |
| ✅ 数据便携 | ❌ 性能一般 |

**适合**: CLI 工具、嵌入式场景、< 5万条

---

### 3.2 客户端-服务端数据库

#### PostgreSQL + pgvector

> <https://github.com/pgvector/pgvector>

```sql
-- 启用扩展
CREATE EXTENSION vector;

-- 创建表
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)
);

-- 向量搜索（余弦距离）
SELECT content, embedding <=> '[0.12, -0.34, ...]' AS distance
FROM documents
ORDER BY distance
LIMIT 5;
```

| 优点 | 缺点 |
|------|------|
| ✅ 一个数据库搞定所有 | ❌ 需要安装 PostgreSQL |
| ✅ SQL + 向量混合查询 | ❌ 亿级数据性能不如专业向量库 |
| ✅ 事务支持，数据一致性好 | |
| ✅ 生态成熟 | |

**适合**: 生产环境、需要 SQL 查询、< 100万条

#### Milvus

> <https://milvus.io/>

```python
from pymilvus import connections, Collection

connections.connect(host="localhost", port="19530")
collection = Collection("my_collection")

results = collection.search(
    data=[[0.12, -0.34, ...]],
    anns_field="embedding",
    limit=5
)
```

| 优点 | 缺点 |
|------|------|
| ✅ 性能最强，亿级数据秒级响应 | ❌ 架构复杂（etcd + minio + milvus） |
| ✅ 支持多种索引算法 | ❌ 资源消耗大（至少 2GB 内存） |
| ✅ 分布式，可横向扩展 | ❌ 学习曲线陡峭 |

**适合**: 企业级应用、大规模数据、高并发场景

#### Qdrant

> <https://qdrant.tech/>

```python
from qdrant_client import QdrantClient

# 支持本地嵌入式模式
client = QdrantClient(path="./qdrant_data")

client.search(
    collection_name="my_collection",
    query_vector=[0.12, -0.34, ...],
    limit=5
)
```

| 优点 | 缺点 |
|------|------|
| ✅ 性能好（Rust 编写） | ❌ 生态不如 Milvus |
| ✅ 支持本地嵌入式模式 | |
| ✅ API 设计友好 | |
| ✅ 支持过滤 + 向量混合查询 | |

**适合**: 中大规模应用、追求性能

---

## 4. 综合对比表

| 方案 | 类型 | 复杂度 | 性能 | 数据规模 | 适用场景 |
|------|------|--------|------|----------|----------|
| **ChromaDB** | 嵌入式 | ⭐ | ⭐⭐ | < 10万 | 快速原型、个人项目 |
| **LanceDB** | 嵌入式 | ⭐ | ⭐⭐⭐ | < 50万 | 快速原型、多模态 |
| **SQLite + vss** | 嵌入式 | ⭐ | ⭐ | < 5万 | CLI 工具、嵌入式 |
| **PostgreSQL + pgvector** | 服务端 | ⭐⭐ | ⭐⭐⭐ | < 100万 | 生产环境、混合查询 |
| **Qdrant** | 服务端 | ⭐⭐⭐ | ⭐⭐⭐⭐ | < 1000万 | 中大规模、高性能 |
| **Milvus** | 服务端 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 亿级 | 企业级、大规模 |

---

## 5. 嵌入式数据库的局限性

### 5.1 为什么企业不用嵌入式数据库？

#### 单进程锁定

```text
场景：两个用户同时访问

┌─────────────┐     ┌─────────────┐
│  用户 A      │     │  用户 B      │
│  写入数据    │     │  写入数据    │
└──────┬──────┘     └──────┬──────┘
       │                   │
       ▼                   ▼
   ┌──────────────────────────┐
   │      ChromaDB 文件        │
   │      ❌ 冲突！锁定！       │
   └──────────────────────────┘
```

Web 应用通常有多个 worker 进程 → 数据冲突/死锁

#### 无法远程访问

微服务架构需要多个服务共享数据 → 嵌入式数据库做不到

#### 无法水平扩展

数据量增长只能换更大的机器（垂直扩展），成本高

#### 缺乏企业级特性

- 无访问控制（用户名/密码/权限）
- 无热备份
- 无高可用（主从复制）
- 无审计日志

---

## 6. 选型建议

### 6.1 演进路径

```text
阶段 1：MVP 验证
└── ChromaDB / LanceDB（快速开发）

阶段 2：小规模生产
└── PostgreSQL + pgvector（单机）

阶段 3：中等规模
└── Qdrant / PostgreSQL 主从

阶段 4：大规模
└── Milvus 集群 / Qdrant 集群
```

### 6.2 决策树

```text
Q: 是个人项目还是团队/生产？
│
├─ 个人项目
│  └─ Q: 数据量多大？
│     ├─ < 10万 → ChromaDB ✅
│     └─ < 50万 → LanceDB ✅
│
└─ 团队/生产
   └─ Q: 是否已有 PostgreSQL？
      ├─ 是 → pgvector ✅
      └─ 否 → Q: 数据量多大？
         ├─ < 100万 → Qdrant ✅
         └─ > 100万 → Milvus ✅
```

---

## 7. 参考资料

- [ChromaDB Documentation](https://docs.trychroma.com/)
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [Milvus Documentation](https://milvus.io/docs)
- [Qdrant Documentation](https://qdrant.tech/documentation/)
- [LanceDB Documentation](https://lancedb.github.io/lancedb/)
- [Vector Database Comparison (Benchmark)](https://ann-benchmarks.com/)
