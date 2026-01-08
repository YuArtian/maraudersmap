#database #AI #RAG #embedding #design-pattern

# Embedding Service（嵌入服务设计）

> 参考 holywell 项目的 `utils_embedding.py` 设计模式

## TL;DR

**Embedding Service** 是将文本转换为向量的服务，是 [[RAG]] 和 [[Vector Database]] 的前置依赖：

- **核心能力**: 文本 → 向量（一组数字）
- **设计模式**: Provider 抽象 + Factory 工厂 + Fallback 降级
- **常用 Provider**: OpenAI、硅基流动、本地模型（sentence-transformers）
- **最佳实践**: 统一接口 + 多后端支持 + 自动降级

---

## 1. 架构设计

### 1.1 整体结构

```text
┌────────────────────────────────────────────────────────────┐
│                    EmbeddingService                        │
│                    (服务管理器)                             │
│                         │                                  │
│                         ▼                                  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              EmbeddingProvider (抽象基类)            │  │
│  │                                                     │  │
│  │   get_embedding(text) → List[float]                 │  │
│  │   get_embeddings_batch(texts) → List[List[float]]   │  │
│  │                                                     │  │
│  └──────────────────────┬──────────────────────────────┘  │
│                         │                                  │
│         ┌───────────────┼───────────────┐                  │
│         │               │               │                  │
│         ▼               ▼               ▼                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│  │  OpenAI    │  │ SiliconFlow│  │   Local    │           │
│  │  Provider  │  │  Provider  │  │  Provider  │           │
│  └────────────┘  └────────────┘  └────────────┘           │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 1.2 设计模式

| 模式 | 作用 |
|------|------|
| **Strategy Pattern** | 不同 Provider 实现相同接口 |
| **Factory Pattern** | 根据配置创建对应 Provider |
| **Fallback Pattern** | API 失败时自动切换本地模型 |

---

## 2. 代码实现

### 2.1 抽象基类

```python
from abc import ABC, abstractmethod
from typing import List, Optional

class EmbeddingProvider(ABC):
    """Embedding 提供商抽象基类"""

    @abstractmethod
    async def get_embedding(self, text: str) -> Optional[List[float]]:
        """获取单个文本的 embedding"""
        pass

    @abstractmethod
    async def get_embeddings_batch(self, texts: List[str]) -> List[Optional[List[float]]]:
        """批量获取 embeddings"""
        pass
```

### 2.2 OpenAI 实现

```python
import aiohttp

class OpenAIEmbeddingProvider(EmbeddingProvider):
    """OpenAI Embedding 提供商"""

    def __init__(
        self,
        api_key: str,
        model: str = "text-embedding-3-small",
        base_url: str = "https://api.openai.com/v1"
    ):
        self.api_key = api_key
        self.model = model
        self.base_url = base_url
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        }

    async def get_embedding(self, text: str) -> Optional[List[float]]:
        async with aiohttp.ClientSession() as session:
            payload = {"model": self.model, "input": text}
            async with session.post(
                f"{self.base_url}/embeddings",
                headers=self.headers,
                json=payload
            ) as response:
                if response.status == 200:
                    data = await response.json()
                    return data["data"][0]["embedding"]
                return None

    async def get_embeddings_batch(self, texts: List[str]) -> List[Optional[List[float]]]:
        async with aiohttp.ClientSession() as session:
            payload = {"model": self.model, "input": texts}
            async with session.post(
                f"{self.base_url}/embeddings",
                headers=self.headers,
                json=payload
            ) as response:
                if response.status == 200:
                    data = await response.json()
                    return [item["embedding"] for item in data["data"]]
                return [None] * len(texts)
```

### 2.3 本地模型实现（Fallback）

```python
class LocalEmbeddingProvider(EmbeddingProvider):
    """本地 Embedding 提供商（sentence-transformers）"""

    def __init__(self, model_name: str = "all-MiniLM-L6-v2"):
        self.model_name = model_name
        self.model = None

    async def _load_model(self):
        if self.model is None:
            from sentence_transformers import SentenceTransformer
            self.model = SentenceTransformer(self.model_name)

    async def get_embedding(self, text: str) -> Optional[List[float]]:
        await self._load_model()
        embedding = self.model.encode([text])[0]
        return embedding.tolist()

    async def get_embeddings_batch(self, texts: List[str]) -> List[Optional[List[float]]]:
        await self._load_model()
        embeddings = self.model.encode(texts)
        return [emb.tolist() for emb in embeddings]
```

### 2.4 服务管理器

```python
class EmbeddingService:
    """Embedding 服务管理器"""

    def __init__(self, provider: EmbeddingProvider):
        self.provider = provider

    async def get_text_embedding(self, text: str) -> Optional[List[float]]:
        if not text or not text.strip():
            return None
        return await self.provider.get_embedding(text.strip())

    async def get_texts_embeddings(self, texts: List[str]) -> List[Optional[List[float]]]:
        if not texts:
            return []
        
        # 过滤空文本，保持索引对应
        valid_texts = []
        valid_indices = []
        for i, text in enumerate(texts):
            if text and text.strip():
                valid_texts.append(text.strip())
                valid_indices.append(i)
        
        if not valid_texts:
            return [None] * len(texts)
        
        valid_embeddings = await self.provider.get_embeddings_batch(valid_texts)
        
        # 重建完整结果
        results = [None] * len(texts)
        for i, embedding in enumerate(valid_embeddings):
            if i < len(valid_indices):
                results[valid_indices[i]] = embedding
        
        return results
```

### 2.5 工厂函数 + Fallback

```python
def create_embedding_service(provider_type: str, **kwargs) -> EmbeddingService:
    """工厂函数：创建 embedding 服务"""
    
    if provider_type.lower() == "openai":
        provider = OpenAIEmbeddingProvider(
            api_key=kwargs.get("api_key"),
            model=kwargs.get("model", "text-embedding-3-small"),
            base_url=kwargs.get("base_url", "https://api.openai.com/v1")
        )
    elif provider_type.lower() == "siliconflow":
        provider = OpenAIEmbeddingProvider(
            api_key=kwargs.get("api_key"),
            model=kwargs.get("model", "BAAI/bge-large-zh-v1.5"),
            base_url="https://api.siliconflow.cn/v1"  # 兼容 OpenAI 格式
        )
    elif provider_type.lower() == "local":
        provider = LocalEmbeddingProvider(
            model_name=kwargs.get("model_name", "all-MiniLM-L6-v2")
        )
    else:
        raise ValueError(f"不支持的 provider: {provider_type}")

    return EmbeddingService(provider)


async def get_default_embedding_service() -> EmbeddingService:
    """获取默认服务（带 Fallback）"""
    import os
    
    # 优先使用 OpenAI / SiliconFlow
    api_key = os.getenv("OPENAI_API_KEY") or os.getenv("SILICONFLOW_API_KEY")
    
    if api_key:
        try:
            base_url = "https://api.siliconflow.cn/v1" if os.getenv("SILICONFLOW_API_KEY") else "https://api.openai.com/v1"
            return create_embedding_service("openai", api_key=api_key, base_url=base_url)
        except Exception as e:
            print(f"API 服务创建失败: {e}, 回退到本地模型")
    
    # Fallback: 本地模型
    return create_embedding_service("local")
```

---

## 3. 常用 Embedding 模型

### 3.1 API 服务

| Provider | 模型 | 维度 | 特点 |
|----------|------|------|------|
| **OpenAI** | text-embedding-3-small | 1536 | 平衡性能和成本 |
| **OpenAI** | text-embedding-3-large | 3072 | 更高精度 |
| **硅基流动** | BAAI/bge-large-zh-v1.5 | 1024 | 中文效果好，免费额度 |
| **硅基流动** | BAAI/bge-m3 | 1024 | 多语言 |

### 3.2 本地模型（sentence-transformers）

| 模型 | 维度 | 特点 |
|------|------|------|
| `all-MiniLM-L6-v2` | 384 | 轻量，速度快 |
| `all-mpnet-base-v2` | 768 | 更高质量 |
| `paraphrase-multilingual-MiniLM-L12-v2` | 384 | 多语言 |
| `BAAI/bge-small-zh-v1.5` | 512 | 中文 |

---

## 4. 使用示例

```python
import asyncio

async def main():
    # 方式一：使用默认服务（自动 Fallback）
    service = await get_default_embedding_service()
    
    # 方式二：指定 Provider
    service = create_embedding_service(
        "siliconflow",
        api_key="your_api_key",
        model="BAAI/bge-large-zh-v1.5"
    )
    
    # 单文本
    embedding = await service.get_text_embedding("今天天气真好")
    print(f"维度: {len(embedding)}")
    
    # 批量
    texts = ["文本1", "文本2", "文本3"]
    embeddings = await service.get_texts_embeddings(texts)
    print(f"获取了 {len(embeddings)} 个向量")

asyncio.run(main())
```

---

## 5. 最佳实践

### 5.1 统一接口

所有 Provider 实现相同的 `EmbeddingProvider` 接口，方便切换

### 5.2 批量处理

- 尽量使用 `get_embeddings_batch` 而不是循环调用 `get_embedding`
- API 通常有批量限制（如每次 10-100 条），需要分批

### 5.3 错误处理与降级

```python
async def get_embedding_with_fallback(text: str) -> List[float]:
    try:
        # 优先使用 API
        service = create_embedding_service("openai", api_key=API_KEY)
        return await service.get_text_embedding(text)
    except Exception:
        # 降级到本地模型
        service = create_embedding_service("local")
        return await service.get_text_embedding(text)
```

### 5.4 缓存

对于重复文本，考虑缓存 embedding 结果，避免重复调用 API

---

## 6. 参考资料

- [OpenAI Embeddings](https://platform.openai.com/docs/guides/embeddings)
- [SiliconFlow API](https://docs.siliconflow.cn/)
- [sentence-transformers](https://www.sbert.net/)
- [MTEB Benchmark](https://huggingface.co/spaces/mteb/leaderboard) - Embedding 模型评测
