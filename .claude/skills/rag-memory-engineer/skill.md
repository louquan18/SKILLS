---
skill: rag-memory-engineer
description: RAG 与记忆系统工程师，负责上下文管理、向量检索、结构化记忆和上下文压缩
tags: [rag, memory, context, vector-search, compression, retrieval]
---

# RAG & Memory Engineer Skill

我是 RAG 与记忆系统工程师，专注于：

## 职责范围

### 1. RAG 系统设计
- 文档分块（Chunking）策略
- 向量索引构建
- 检索策略（向量/关键词/混合）
- 上下文窗口管理

### 2. 结构化记忆设计
- 短期记忆（对话历史）
- 长期记忆（持久化知识）
- 工作记忆（当前任务上下文）
- 实体关系图谱

### 3. 上下文压缩
- 摘要生成
- 关键信息提取
- 压缩率优化（目标 40%+）
- 信息丢失控制

### 4. 检索优化
- 向量检索
- 关键词检索（BM25）
- 混合检索策略
- 重排序（Reranking）
- 响应时间优化（目标 < 2s）

## RAG 系统架构

```
用户查询
    ↓
Query Processing (查询处理)
    ↓
Retrieval (检索)
    ├─ Vector Search (向量检索)
    ├─ Keyword Search (关键词检索)
    └─ Hybrid Search (混合检索)
    ↓
Reranking (重排序)
    ↓
Context Assembly (上下文组装)
    ↓
LLM Generation (生成)
```

## 文档分块策略

### 1. 固定大小分块

```python
def fixed_size_chunking(text: str, chunk_size: int = 512, overlap: int = 50):
    """固定大小分块"""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start = end - overlap
    return chunks
```

### 2. 语义分块

```python
async def semantic_chunking(text: str):
    """基于语义的分块（按段落/章节）"""
    # 按段落分割
    paragraphs = text.split("\n\n")

    # 合并过短的段落
    chunks = []
    current_chunk = ""
    for para in paragraphs:
        if len(current_chunk) + len(para) < 512:
            current_chunk += "\n\n" + para
        else:
            if current_chunk:
                chunks.append(current_chunk.strip())
            current_chunk = para

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks
```

### 3. 递归分块

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=["\n\n", "\n", "。", "！", "？", ".", "!", "?", " "]
)

chunks = splitter.split_text(document)
```

## 向量索引

### 构建向量索引

```python
from langchain.vectorstores import Chroma, FAISS
from langchain.embeddings import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

# Chroma（持久化）
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./chroma_db"
)

# FAISS（高性能）
vectorstore = FAISS.from_documents(
    documents=chunks,
    embedding=embeddings
)
```

### 添加文档

```python
async def add_documents(vectorstore, documents: List[str]):
    """添加文档到向量索引"""
    # 分块
    chunks = []
    for doc in documents:
        chunks.extend(semantic_chunking(doc))

    # 添加到索引
    vectorstore.add_documents(chunks)
```

## 检索策略

### 1. 向量检索

```python
async def vector_search(query: str, vectorstore, k: int = 5):
    """向量相似度检索"""
    results = vectorstore.similarity_search_with_score(
        query=query,
        k=k
    )
    return [(doc, score) for doc, score in results]
```

### 2. 关键词检索（BM25）

```python
from rank_bm25 import BM25Okapi

def bm25_search(query: str, corpus: List[str], k: int = 5):
    """BM25 关键词检索"""
    # 分词
    tokenized_corpus = [list(doc) for doc in corpus]
    bm25 = BM25Okapi(tokenized_corpus)

    # 查询
    tokenized_query = list(query)
    scores = bm25.get_scores(tokenized_query)

    # 返回 top-k
    top_indices = scores.argsort()[-k:][::-1]
    return [(corpus[i], scores[i]) for i in top_indices]
```

### 3. 混合检索

```python
async def hybrid_search(
    query: str,
    vectorstore,
    corpus: List[str],
    k: int = 5,
    alpha: float = 0.7
):
    """
    混合检索（向量 + 关键词）

    alpha: 向量检索权重（0-1）
    """
    # 向量检索
    vector_results = await vector_search(query, vectorstore, k=k*2)

    # 关键词检索
    bm25_results = bm25_search(query, corpus, k=k*2)

    # 归一化分数
    vector_scores = normalize_scores([s for _, s in vector_results])
    bm25_scores = normalize_scores([s for _, s in bm25_results])

    # 合并分数
    combined = {}
    for (doc, _), score in zip(vector_results, vector_scores):
        combined[doc.page_content] = alpha * score

    for (doc, _), score in zip(bm25_results, bm25_scores):
        if doc in combined:
            combined[doc] += (1 - alpha) * score
        else:
            combined[doc] = (1 - alpha) * score

    # 排序返回
    sorted_results = sorted(combined.items(), key=lambda x: x[1], reverse=True)
    return sorted_results[:k]
```

### 4. 重排序（Reranking）

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank(query: str, documents: List[str], top_k: int = 3):
    """重排序"""
    pairs = [(query, doc) for doc in documents]
    scores = reranker.predict(pairs)

    # 按分数排序
    sorted_indices = scores.argsort()[::-1][:top_k]
    return [documents[i] for i in sorted_indices]
```

## 记忆系统设计

### 1. 短期记忆（对话历史）

```python
from collections import deque

class ShortTermMemory:
    """短期记忆（对话历史）"""

    def __init__(self, max_turns: int = 10):
        self.history = deque(maxlen=max_turns)

    def add(self, role: str, content: str):
        self.history.append({"role": role, "content": content})

    def get_context(self) -> str:
        return "\n".join(
            f"{msg['role']}: {msg['content']}"
            for msg in self.history
        )
```

### 2. 长期记忆（持久化知识）

```python
class LongTermMemory:
    """长期记忆（持久化知识）"""

    def __init__(self, vectorstore):
        self.vectorstore = vectorstore

    async def store(self, key: str, content: str, metadata: dict = None):
        """存储知识"""
        doc = Document(
            page_content=content,
            metadata={"key": key, **(metadata or {})}
        )
        self.vectorstore.add_documents([doc])

    async def recall(self, query: str, k: int = 5):
        """检索相关知识"""
        return self.vectorstore.similarity_search(query, k=k)
```

### 3. 工作记忆（当前任务上下文）

```python
class WorkingMemory:
    """工作记忆（当前任务上下文）"""

    def __init__(self):
        self.data = {}

    def set(self, key: str, value: Any):
        self.data[key] = value

    def get(self, key: str, default=None):
        return self.data.get(key, default)

    def clear(self):
        self.data.clear()

    def to_context(self) -> str:
        return "\n".join(f"{k}: {v}" for k, v in self.data.items())
```

### 4. 实体关系图谱

```python
import networkx as nx

class EntityGraph:
    """实体关系图谱"""

    def __init__(self):
        self.graph = nx.DiGraph()

    def add_entity(self, name: str, attributes: dict = None):
        """添加实体"""
        self.graph.add_node(name, **(attributes or {}))

    def add_relation(self, from_entity: str, to_entity: str, relation: str, **attrs):
        """添加关系"""
        self.graph.add_edge(from_entity, to_entity, relation=relation, **attrs)

    def get_related(self, entity: str, relation: str = None):
        """获取相关实体"""
        edges = self.graph.edges(entity, data=True)
        if relation:
            return [(to, data) for _, to, data in edges if data.get("relation") == relation]
        return [(to, data) for _, to, data in edges]

    def find_path(self, from_entity: str, to_entity: str):
        """查找路径"""
        try:
            return nx.shortest_path(self.graph, from_entity, to_entity)
        except nx.NetworkXNoPath:
            return None
```

## 上下文压缩

### 压缩流程

```python
async def compress_context(context: str) -> Dict[str, Any]:
    """
    压缩上下文

    流程：
    1. 提取关键信息
    2. 生成摘要
    3. 结构化存储
    """

    # 1. 关键信息提取
    key_info = await extract_key_info(context)

    # 2. 生成摘要
    summary = await generate_summary(context)

    # 3. 结构化存储
    compressed = {
        "summary": summary,
        "key_info": key_info,
        "original_length": len(context),
        "compressed_length": len(summary) + len(str(key_info))
    }

    return compressed
```

### 压缩率计算

```python
def calculate_compression_rate(original: str, compressed: Dict) -> float:
    """计算压缩率"""
    original_size = len(original)
    compressed_size = compressed["compressed_length"]

    compression_rate = 1 - (compressed_size / original_size)
    return compression_rate  # 目标 > 0.4 (40%)
```

## 上下文窗口管理

### 滑动窗口

```python
class ContextWindow:
    """上下文窗口管理"""

    def __init__(self, max_tokens: int = 4096):
        self.max_tokens = max_tokens
        self.messages = []

    def add(self, message: str):
        self.messages.append(message)
        self._trim()

    def _trim(self):
        """裁剪到最大 token 数"""
        total = sum(len(m) for m in self.messages)
        while total > self.max_tokens and len(self.messages) > 1:
            removed = self.messages.pop(0)
            total -= len(removed)

    def get_context(self) -> str:
        return "\n".join(self.messages)
```

### 摘要压缩

```python
async def summarize_and_compress(messages: List[str], max_tokens: int):
    """对旧消息生成摘要以节省空间"""
    if len(messages) <= 3:
        return messages

    # 旧消息生成摘要
    old_messages = messages[:-2]
    summary = await generate_summary("\n".join(old_messages))

    # 保留最近消息
    recent_messages = messages[-2:]

    return [f"[历史摘要] {summary}"] + recent_messages
```

## Memory Schema

```python
from pydantic import BaseModel
from typing import List, Dict, Optional, Any

class MemoryEntry(BaseModel):
    """记忆条目"""
    id: str
    content: str
    metadata: Dict[str, Any] = {}
    importance: str = "medium"  # high | medium | low
    created_at: str = ""
    last_accessed: str = ""
    access_count: int = 0

class ConversationMemory(BaseModel):
    """对话记忆"""
    messages: List[Dict[str, str]] = []
    summary: Optional[str] = None

class KnowledgeMemory(BaseModel):
    """知识记忆"""
    entities: Dict[str, Dict[str, Any]] = {}
    relations: List[Dict[str, Any]] = []
    facts: List[str] = []

class TaskMemory(BaseModel):
    """任务记忆"""
    task_id: str
    goal: str
    progress: Dict[str, Any] = {}
    context: Dict[str, Any] = {}
    history: List[str] = []
```

## 性能优化

### 1. 缓存策略

```python
from functools import lru_cache

@lru_cache(maxsize=100)
def cached_search(query: str, k: int = 5):
    """缓存检索结果"""
    return vectorstore.similarity_search(query, k=k)
```

### 2. 批量加载

```python
async def batch_load_context(task_id: str):
    """批量加载上下文"""

    # 并行加载
    conversation_task = load_conversation(task_id)
    knowledge_task = load_knowledge(task_id)
    task_context_task = load_task_context(task_id)

    conversation, knowledge, task_context = await asyncio.gather(
        conversation_task,
        knowledge_task,
        task_context_task
    )

    return {
        "conversation": conversation,
        "knowledge": knowledge,
        "task_context": task_context
    }
```

### 3. 异步索引更新

```python
async def async_index_update(vectorstore, new_documents: List[str]):
    """异步更新索引"""
    # 后台任务更新索引
    asyncio.create_task(
        update_index(vectorstore, new_documents)
    )
```

## 最佳实践

1. **结构化优于全文** - 提取结构化信息而非保存全文
2. **增量更新** - 只更新变化的部分
3. **分层存储** - 热数据内存，冷数据数据库
4. **定期压缩** - 历史数据定期压缩
5. **向量索引** - 使用向量检索提高准确性
6. **监控压缩率** - 确保达到 40%+ 压缩率
7. **响应时间** - 检索响应时间 < 2s
8. **混合检索** - 结合向量和关键词检索
9. **重排序** - 使用 Reranker 提高相关性
10. **缓存热点** - 缓存高频查询结果

## 相关 Skills

- `/agent-workflow-architect` - 集成到 Agent 工作流
- `/python-backend` - 后端实现
