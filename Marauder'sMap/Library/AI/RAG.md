#AI #LLM #RAG

# RAG (Retrieval-Augmented Generation)

> 检索增强生成

> 字节跳动 RAG 实践手册-1014
> https://docs.qq.com/doc/DSXJiaE5taUtaVGx6?_t=1767768738907&nlc=1

## TL;DR

**RAG（Retrieval-Augmented Generation，检索增强生成）** 是一种将 **信息检索** 与 **文本生成** 结合的 AI 架构模式：

- **核心思想**: 先从外部知识库中检索相关信息，再将检索到的内容作为上下文提供给 LLM 生成回答
- **解决的问题**:
  - 🧠 LLM 的知识截止日期问题（Knowledge Cutoff）
  - 🎭 减少幻觉（Hallucination）
  - 📚 支持私有/专业领域知识
  - 💰 降低微调成本
- **基本流程**: `用户提问 → 向量化 → 相似度检索 → 构建 Prompt → LLM 生成回答`
- **关键组件**: Embedding Model、Vector Database、Chunking Strategy、Retriever、LLM

---

## 1. 为什么需要 RAG？

### 1.1 LLM 的局限性

大型语言模型（LLM）虽然强大，但存在几个固有问题：

| 问题 | 描述 |
|------|------|
| **知识截止** | 模型只知道训练数据截止日期之前的信息 |
| **幻觉** | 模型可能会"编造"不存在的事实 |
| **无法访问私有数据** | 企业内部文档、最新数据等无法直接使用 |
| **微调成本高** | 为特定领域微调模型需要大量数据和计算资源 |

### 1.2 RAG 的解决方案

RAG 通过在推理时动态注入外部知识来解决这些问题：

```text
传统 LLM:  用户提问 ──────────────────────────────→ LLM ──→ 回答
                                                    ↑
                                            (仅依赖模型内部知识)

RAG:       用户提问 ──→ 检索相关文档 ──→ 构建增强Prompt ──→ LLM ──→ 回答
                            ↑                    ↑
                       (外部知识库)          (检索到的上下文)
```

---

## 2. RAG 的基本架构

### 2.1 整体流程

```text
┌────────────────────────────────────────────────────────────────┐
│                     RAG System Architecture                    │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────┐    ┌────────────┐    ┌────────────────┐        │
│  │  Documents │───→│  Chunking  │───→│  Vector Store  │        │
│  └────────────┘    └────────────┘    └───────┬────────┘        │
│                                              │                 │
│                   Offline Indexing           │                 │
│  ────────────────────────────────────────────┼──────────────   │
│                   Online Querying            │                 │
│                                              ↓                 │
│  ┌────────────┐    ┌────────────┐    ┌────────────────┐        │
│  │   Query    │───→│  Embedding │───→│   Retrieval    │        │
│  └────────────┘    └────────────┘    └───────┬────────┘        │
│                                              │                 │
│                                              ↓                 │
│  ┌────────────┐    ┌────────────┐    ┌────────────────┐        │
│  │   Answer   │←───│   LLM Gen  │←───│  Augmentation  │        │
│  └────────────┘    └────────────┘    └────────────────┘        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 两个核心阶段

#### 阶段一：Indexing（离线索引）

将知识库转化为可检索的形式：

1. **Load（加载）**: 从各种来源加载文档（PDF、网页、数据库等）
2. **Split（切分）**: 将长文档切分为适当大小的 chunks
3. **Embed（向量化）**: 使用 Embedding Model 将文本转为向量
4. **Store（存储）**: 将向量存入 Vector Database

#### 阶段二：Retrieval & Generation（在线查询）

处理用户查询并生成回答：

1. **Embed Query（查询向量化）**: 将用户问题转为向量
2. **Retrieve（检索）**: 在向量数据库中找到最相似的 chunks
3. **Augment（增强）**: 将检索到的内容注入 Prompt
4. **Generate（生成）**: LLM 基于增强的 Prompt 生成回答

---

## 3. 核心组件详解

### 3.1 Embedding Model（嵌入模型）

将文本转换为高维向量，使语义相似的文本在向量空间中距离更近。

**常用模型**:

- OpenAI: `text-embedding-3-small`, `text-embedding-3-large`
- 开源: `BGE`, `E5`, `GTE`, `Jina Embeddings`

**选择考虑因素**:

- 向量维度（影响存储和计算成本）
- 支持的最大 token 数
- 多语言支持
- 在目标领域的表现

### 3.2 Chunking Strategy（文本切分策略）

将长文档切分为适当大小的片段，是影响 RAG 质量的关键因素。

**常见策略**:

| 策略 | 描述 | 适用场景 |
|------|------|----------|
| **Fixed Size** | 按固定字符/token数切分 | 简单快速，适合结构化文档 |
| **Sentence** | 按句子边界切分 | 保持语义完整性 |
| **Paragraph** | 按段落切分 | 适合文章类内容 |
| **Recursive** | 递归切分，先按大单位再按小单位 | 通用性好，LangChain 默认 |
| **Semantic** | 基于语义相似度动态切分 | 质量最高，计算成本也高 |

**关键参数**:

- `chunk_size`: 每个片段的大小
- `chunk_overlap`: 相邻片段的重叠部分，避免信息丢失

```python
# Example: LangChain RecursiveCharacterTextSplitter
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", "。", ".", " ", ""]
)
chunks = splitter.split_text(document)
```

### 3.3 Vector Database（向量数据库）

专门存储和检索向量的数据库，支持高效的相似度搜索。

**主流选择**:

| 数据库 | 特点 |
|--------|------|
| **Pinecone** | 全托管，开箱即用，适合快速上手 |
| **Weaviate** | 开源，支持混合搜索 |
| **Milvus** | 开源，高性能，适合大规模场景 |
| **Chroma** | 轻量级，适合本地开发和原型 |
| **Qdrant** | 开源，Rust 编写，性能优秀 |
| **pgvector** | PostgreSQL 扩展，适合已有 PG 基础设施 |

### 3.4 Retriever（检索器）

从向量数据库中检索相关文档的组件。

**检索方法**:

```text
┌───────────────────────────────────────────────────────┐
│                  Retrieval Methods                    │
├───────────────────────────────────────────────────────┤
│                                                       │
│   Dense Retrieval                                     │
│   └── Cosine Similarity / Dot Product / Euclidean     │
│                                                       │
│   Sparse Retrieval                                    │
│   └── Keyword Matching (BM25, TF-IDF)                 │
│                                                       │
│   Hybrid Retrieval                                    │
│   └── Combines Dense + Sparse for better results      │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

## 4. RAG 的进阶技术

### 4.1 Query Transformation（查询转换）

优化用户的原始查询，提高检索效果。

**常见方法**:

- **Query Rewriting**: 重写查询，使其更清晰、更适合检索
- **HyDE (Hypothetical Document Embeddings)**: 先让 LLM 生成假设答案，用假设答案去检索
- **Multi-Query**: 将一个查询扩展为多个不同角度的查询
- **Step-back Prompting**: 生成更抽象/概括的问题

```python
# HyDE Example
original_query = "What is RAG?"

# Step 1: Generate hypothetical answer
hypothetical_answer = llm.generate(
    f"Write a passage that answers: {original_query}"
)

# Step 2: Use hypothetical answer for retrieval
results = vector_store.similarity_search(hypothetical_answer)
```

### 4.2 Re-ranking（重排序）

对初步检索结果进行二次排序，提高相关性。

**工作流程**:

```text
Query → 初次检索 (Top-K) → Re-ranker → 重排序后的结果 (Top-N, N < K)
```

**常用 Re-ranker**:

- Cohere Rerank
- BGE Reranker
- Cross-Encoder models

### 4.3 Contextual Compression（上下文压缩）

压缩检索到的内容，只保留与查询最相关的部分。

**优点**:

- 减少 Prompt 长度，节省 token
- 过滤噪声信息
- 提高回答质量

### 4.4 Multi-Modal RAG（多模态 RAG）

支持图片、表格等非文本内容的检索和生成。

**实现方式**:

- 图片: 使用 CLIP 等模型生成图像向量
- 表格: 转换为文本描述或使用专门的表格 Embedding

---

## 5. RAG 评估指标

### 5.1 检索质量评估

| 指标 | 描述 |
|------|------|
| **Recall@K** | Top-K 结果中包含正确答案的比例 |
| **Precision@K** | Top-K 结果中相关文档的比例 |
| **MRR (Mean Reciprocal Rank)** | 第一个相关结果的排名倒数的平均值 |
| **NDCG** | 归一化折损累计增益，考虑排名位置 |

### 5.2 生成质量评估

| 指标 | 描述 |
|------|------|
| **Faithfulness** | 回答是否忠实于检索到的上下文 |
| **Answer Relevancy** | 回答与问题的相关程度 |
| **Context Relevancy** | 检索到的上下文与问题的相关程度 |

**评估框架**:

- RAGAS
- TruLens
- LangSmith

---

## 6. 常见问题与优化

### 6.1 检索质量差

**可能原因及解决方案**:

| 问题 | 解决方案 |
|------|----------|
| Chunk 太大/太小 | 调整 chunk_size，测试不同值 |
| Embedding 模型不合适 | 尝试其他模型或领域特定模型 |
| 查询和文档表述差异大 | 使用 Query Transformation |
| 关键词匹配重要 | 使用 Hybrid Search |

### 6.2 生成质量差

**可能原因及解决方案**:

| 问题 | 解决方案 |
|------|----------|
| 上下文噪声多 | 使用 Re-ranking 或 Compression |
| Prompt 设计不佳 | 优化 Prompt Template |
| 上下文不足 | 增加 Top-K 或改进检索 |
| LLM 能力不足 | 使用更强的模型 |

### 6.3 幻觉问题

**缓解策略**:

- 在 Prompt 中明确要求只基于提供的上下文回答
- 要求模型在不确定时说明
- 添加引用/来源标注
- 使用 Faithfulness 检测

---

## 7. 实践示例

### 7.1 使用 LangChain 构建基础 RAG

```python
from langchain.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma
from langchain.chat_models import ChatOpenAI
from langchain.chains import RetrievalQA

# 1. Load documents
loader = PyPDFLoader("knowledge_base.pdf")
documents = loader.load()

# 2. Split into chunks
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200
)
chunks = text_splitter.split_documents(documents)

# 3. Create embeddings and store in vector DB
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(chunks, embeddings)

# 4. Create retriever
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 4}
)

# 5. Create RAG chain
llm = ChatOpenAI(model="gpt-4")
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=retriever
)

# 6. Query
response = qa_chain.invoke("What is the main topic of this document?")
print(response)
```

### 7.2 使用 LlamaIndex 构建 RAG

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# 1. Load documents
documents = SimpleDirectoryReader("./data").load_data()

# 2. Create index (handles chunking and embedding automatically)
index = VectorStoreIndex.from_documents(documents)

# 3. Create query engine
query_engine = index.as_query_engine()

# 4. Query
response = query_engine.query("What is RAG?")
print(response)
```

---

## 8. RAG vs Fine-tuning

| 维度 | RAG | Fine-tuning |
|------|-----|-------------|
| **知识更新** | 实时更新，只需更新知识库 | 需要重新训练 |
| **成本** | 推理成本稍高，但无训练成本 | 训练成本高 |
| **可解释性** | 高，可追溯信息来源 | 低，知识融入模型参数 |
| **适用场景** | 知识密集型、需要最新信息 | 风格迁移、特定任务优化 |
| **幻觉控制** | 较好，基于检索内容 | 较难控制 |

> 💡 **最佳实践**: RAG 和 Fine-tuning 并不互斥，可以结合使用。Fine-tuning 用于调整模型的行为/风格，RAG 用于注入领域知识。

---

## 9. 相关工具与框架

| 工具 | 描述 |
|------|------|
| **LangChain** | 最流行的 LLM 应用开发框架 |
| **LlamaIndex** | 专注于 RAG 的数据框架 |
| **Haystack** | 端到端 NLP 框架 |
| **Semantic Kernel** | 微软的 AI 编排框架 |
| **Vercel AI SDK** | 前端友好的 AI SDK |

---

## 10. 总结

RAG 是构建知识密集型 AI 应用的关键技术，它通过将检索与生成相结合，有效解决了 LLM 的知识局限性问题。

**核心要点**:

1. **理解本质**: RAG = Retrieval + Augmentation + Generation
2. **关注细节**: Chunking、Embedding、Retrieval 策略都会显著影响效果
3. **持续优化**: 使用评估指标量化效果，针对性优化
4. **适时进阶**: Query Transformation、Re-ranking 等技术能显著提升效果
5. **选择合适的工具**: 根据场景选择合适的框架和组件

---

## 参考资料

- [LangChain RAG Tutorial](https://python.langchain.com/docs/tutorials/rag/)
- [LlamaIndex Documentation](https://docs.llamaindex.ai/)
- [RAG Paper (Lewis et al., 2020)](https://arxiv.org/abs/2005.11401)
- [Pinecone Learning Center](https://www.pinecone.io/learn/)
