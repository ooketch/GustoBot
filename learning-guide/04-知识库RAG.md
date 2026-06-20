# 阶段 4：知识库 RAG

> **目标**：能讲清楚 RAG 在项目里的完整流程——从文档入库到检索到生成答案。

### 阅读顺序

1. [kb_ingest/](../kb_ingest/) — 知识入库服务，看文档怎么切 chunk、怎么写入向量库
2. [knowledge_service.py](../gustobot/infrastructure/knowledge/knowledge_service.py) — `KnowledgeService` 类，重点看 `search()` 方法
3. [vector_store.py](../gustobot/infrastructure/knowledge/vector_store.py) — Milvus 向量检索实现
4. [reranker.py](../gustobot/infrastructure/knowledge/reranker.py) — 多 Provider Reranker
5. [kb_tools/node.py](../gustobot/application/agents/kb_tools/node.py) — KB 子图节点实现

### 动手任务

找 `KnowledgeService.search()` 方法，标注出每一步做了什么、用了什么阈值。

### 要理解的问题

- RAG 的完整流程是什么？
- chunk size 和 overlap 怎么影响效果？
- embedding 检索和 rerank 的区别是什么？
- 为什么不能只靠向量相似度？
- 检索不到答案时怎么兜底？

---

## 4.1 架构总览

GustoBot 的 RAG 系统分为**两条数据通路**：

```
┌─────────────────────────────────────────────────────────┐
│  通路 A：文档入库（kb_ingest 服务）                        │
│  Excel/MySQL → DataProcessor → LLM改写 → Embedding      │
│              → VectorStoreWriter → PostgreSQL (pgvector)  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  通路 B：查询检索（gustobot 主服务）                        │
│  用户问题 → Guardrails → Router → local_search           │
│    ├─ 优先 PostgreSQL (pgvector)                          │
│    ├─ 兜底 Milvus 向量库                                   │
│    └─ 可选 外部检索                                        │
│  → Reranker → LLM 生成答案 → 带 sources 输出              │
└─────────────────────────────────────────────────────────┘
```

系统同时维护两套向量存储：
- **PostgreSQL + pgvector**（主存储）：通过 `kb_ingest` 服务写入，存放结构化表格数据、Excel 导入的枚举字段
- **Milvus**（兜底存储）：通过 `gustobot` 主服务的 `KnowledgeService` 写入，存放长文本、文章、典故故事等非结构化内容

---

## 4.2 文档入库流程（kb_ingest）

入库服务是一个独立的 Python 包，位于 `kb_ingest/`，提供 CLI 和 FastAPI 两种入口。

### 4.2.1 入口

**CLI 入口** (`kb_ingest/main.py`)：

```python
from kb_service.cli import main
# 支持子命令：process-excel, search, ingest-mysql
```

**FastAPI 入口** (`kb_ingest/kb_service/api/routes.py`)：

| 端点 | 方法 | 说明 |
|------|------|------|
| `POST /ingest/excel` | 同步处理 Excel 文件路径 | 后台异步执行 |
| `POST /ingest/excel/upload` | 上传 Excel 文件并处理 | 自动清理临时文件 |
| `POST /ingest/mysql` | 从 MySQL 拉取数据 | 后台异步执行 |
| `POST /search` | pgvector 相似度搜索 | 同步返回 |
| `POST /search/hybrid` | 向量检索 + Rerank 精排 | 混合召回 |

### 4.2.2 DataProcessor 处理管线

核心类 `DataProcessor`（`kb_ingest/kb_service/services/processor.py`）负责完整的入库管线：

**第一步：读取 Excel**

```python
def process_excel(self) -> None:
    xls = pd.ExcelFile(self.config.excel_file_path)
    for sheet_name in xls.sheet_names:
        self._process_sheet(xls, sheet_name)
    self._save_summary()
    if self.config.db:
        self._embed_and_store(incremental=self.incremental)
```

每个工作表按 `config.batch_size`（默认 16）分批处理，逐行调用 `_process_row()`。

**第二步：LLM 改写或扁平化**

每行数据有两种处理策略：

1. **LLM 改写模式**（`config.use_llm=True`）：通过 `PromptManager` 构造 prompt，调用 `LLMClient.generate()` 将表格行改写为自然语言文本。失败时重试 `config.retry_times`（默认 3）次，最终失败可降级为扁平化（`llm_fallback_to_flatten`）。

2. **扁平化模式**（`config.use_llm=False`）：调用 `flatten_row()` 直接将键值对拼接为字符串。

**第三步：Embedding 与写入 pgvector**

`_embed_and_store()` 方法将处理好的数据交给 `VectorStoreWriter.upsert()`。

### 4.2.3 VectorStoreWriter 写入 pgvector

类 `VectorStoreWriter`（`kb_ingest/kb_service/services/vector_store.py`）负责：

1. 自动检测并创建 `searchable_documents` 表（如不存在）
2. 验证向量维度与 Embedding 模型一致
3. 按批次（batch_size=32）生成 Embedding 并 UPSERT

表结构：

```sql
CREATE TABLE searchable_documents (
    id            BIGSERIAL PRIMARY KEY,
    source_table  TEXT,
    source_id     TEXT,
    content       TEXT,
    embedding     vector({dimension}),
    company_name  TEXT,
    report_year   TEXT,
    credit_no     TEXT,
    origin_status TEXT,
    metadata      TEXT,
    content_hash  TEXT DEFAULT '',
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(source_table, source_id)
);
```

UPSERT 语句使用 `ON CONFLICT (source_table, source_id) DO UPDATE`，确保同一来源的记录可以被覆盖更新。

**增量模式**：当 `incremental=True` 时，`_filter_changed_items()` 方法通过 `content_hash` 字段对比，只处理内容发生变化的记录，跳过未变化的数据。

---

## 4.3 Embedding 生成

### 4.3.1 OpenAICompatibleEmbeddings

主服务使用的 Embedding 类（`gustobot/infrastructure/knowledge/embeddings.py`），基于 OpenAI 官方 SDK：

```python
class OpenAICompatibleEmbeddings:
    def __init__(self, *, model, api_key, base_url, dimension, max_batch_size=64, request_timeout=60.0):
```

**关键设计**：

- **兼容多供应商**：通过 `base_url` 切换 OpenAI、DashScope（Qwen）、Ollama 等任何兼容 `/embeddings` 端点的服务
- **分批请求**：`_embed()` 方法按 `max_batch_size`（默认 64）分批调用 API，避免单次请求过大
- **空值保护**：DashScope 对空字符串报错，代码用单空格 `" "` 替代
- **维度校验**：返回向量维度与预期不符时输出 warning
- **零向量兜底**：API 未返回某条结果时，用全零向量填充
- **代理容错**：检测到 `socks://` 等不支持的代理 scheme 时，自动创建 `trust_env=False` 的 httpx 客户端

两个公共方法：

| 方法 | 用途 |
|------|------|
| `embed_documents(texts)` | 批量嵌入文档文本（入库时用） |
| `embed_query(text)` | 嵌入单条查询文本（检索时用） |

### 4.3.2 kb_ingest 的 EmbeddingClient

入库服务使用独立的 `EmbeddingClient`（`kb_ingest/kb_service/clients/embedding.py`），同样兼容 OpenAI 协议，配置项以 `EMBEDDING_` 为前缀。

---

## 4.4 Milvus 向量检索

类 `VectorStore`（`gustobot/infrastructure/knowledge/vector_store.py`）封装了 Milvus 的连接、建表、索引和搜索。

### 4.4.1 集合 Schema

```python
fields = [
    FieldSchema(name="id",         dtype=DataType.VARCHAR,      is_primary=True, max_length=256),
    FieldSchema(name="embedding",  dtype=DataType.FLOAT_VECTOR, dim=self.dimension),
    FieldSchema(name="content",    dtype=DataType.VARCHAR,      max_length=65535),
    FieldSchema(name="recipe_id",  dtype=DataType.VARCHAR,      max_length=256),
    FieldSchema(name="name",       dtype=DataType.VARCHAR,      max_length=512),
    FieldSchema(name="category",   dtype=DataType.VARCHAR,      max_length=128),
    FieldSchema(name="difficulty", dtype=DataType.VARCHAR,      max_length=128),
]
```

### 4.4.2 索引类型与距离度量

默认配置（来自 `gustobot/config/settings.py`）：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `MILVUS_INDEX_TYPE` | `IVF_FLAT` | 倒排文件 + 平坦量化，精度最高 |
| `MILVUS_METRIC_TYPE` | `IP` | Inner Product（等价于归一化向量的余弦相似度） |
| `MILVUS_HOST` | `localhost` | |
| `MILVUS_PORT` | `19530` | |
| `MILVUS_COLLECTION` | `recipes` | 集合名称 |

索引创建逻辑：

```python
index_params = {
    "index_type": self.index_type,
    "metric_type": self.metric_type,
    "params": {"nlist": 128} if self.index_type == "IVF_FLAT" else {}
}
```

- `IVF_FLAT`：将向量空间划分为 128 个聚类（nlist），搜索时探查 `nprobe=10` 个聚类
- 支持切换为 `IVF_SQ8`（标量量化，省内存）或 `HNSW`（图索引，高召回）

### 4.4.3 search 方法

```python
def search(self, query_embedding, top_k=10, filter_expr=None) -> List[Dict]:
    search_params = {"metric_type": self.metric_type, "params": {"nprobe": 10}}
    results = self.collection.search(
        data=[query_embedding],
        anns_field="embedding",
        param=search_params,
        limit=top_k,
        expr=filter_expr,                       # 支持 Milvus 表达式过滤
        output_fields=["id", "content", "recipe_id", "name", "category", "difficulty"]
    )
```

返回格式：

```python
{
    "id": "recipe_001_0",
    "content": "菜名：宫保鸡丁\n分类：川菜...",
    "score": 0.87,
    "metadata": {"recipe_id": "001", "name": "宫保鸡丁", "category": "川菜", "difficulty": "中等"}
}
```

`filter_expr` 参数支持 Milvus 布尔表达式，例如 `"category == '家常菜'"` 或 `"difficulty != '困难'"`。

---

## 4.5 Reranker 重排序

Reranker 的作用是对向量检索返回的候选集进行精排，提高最终结果的相关性。系统在两个层面实现了 Reranker：

### 4.5.1 主服务 Reranker

类 `Reranker`（`gustobot/infrastructure/knowledge/reranker.py`）支持四种 Provider：

| Provider | 方法 | 默认模型 | API 地址 |
|----------|------|----------|----------|
| `cohere` | `_cohere_rerank()` | `rerank-english-v3.0` | Cohere 官方 |
| `jina` | `_jina_rerank()` | `jina-reranker-v1-base-en` | `https://api.jina.ai/v1/rerank` |
| `voyage` | `_voyage_rerank()` | `rerank-lite-1` | `https://api.voyageai.com/v1/rerank` |
| `custom` | `_custom_rerank()` | 配置指定 | DashScope/BGE 等自建服务 |

初始化逻辑：

```python
def __init__(self):
    self.enabled = settings.RERANK_ENABLED      # 默认 True
    self.provider = settings.RERANK_PROVIDER     # 默认 "custom"
    # 如果 enabled 但缺少 provider 或 api_key，自动降级为 disabled
```

### 4.5.2 双重阈值过滤

`KnowledgeService.search()` 中实现了两层过滤：

**第一层：向量相似度阈值**

```python
# 先用向量相似度过滤明显不相关的候选
candidates = [r for r in candidates if r.get("score", 0.0) >= similarity_threshold]
# 默认阈值: KB_SIMILARITY_THRESHOLD = 0.2
```

**第二层：Rerank 分数阈值**

```python
# Reranker 精排后，再用 rerank_score 过滤
candidates = [
    r for r in candidates
    if r.get("rerank_score", 0.0) >= settings.KB_RERANK_SCORE_THRESHOLD
    # 默认阈值: 0.8
]
```

### 4.5.3 搜索流程中的召回策略

```python
async def search(self, query, top_k, similarity_threshold, filter_expr, filter_by_similarity=True):
    recall_k = top_k
    if self.reranker.enabled:
        recall_k = settings.RERANK_MAX_CANDIDATES  # 默认 20，召回更多候选

    # 1. 向量检索（召回 recall_k 条）
    embedding = self.embedder.embed_query(query)
    results = self.vector_store.search(embedding, recall_k, filter_expr)

    # 2. 相似度阈值过滤
    if filter_by_similarity:
        candidates = [r for r in results if r["score"] >= similarity_threshold]

    # 3. Reranker 精排
    if candidates and self.reranker.enabled:
        candidates = await self.reranker.rerank(query, candidates, top_k)

    # 4. Rerank 分数阈值过滤
    if self.reranker.enabled:
        candidates = [r for r in candidates if r["rerank_score"] >= KB_RERANK_SCORE_THRESHOLD]

    return candidates[:top_k]
```

核心思路：**召回多、精排少**。启用 Reranker 时，向量检索阶段多召回一些候选（`RERANK_MAX_CANDIDATES=20`），Reranker 精排后再截取 `top_k` 条。

### 4.5.4 kb_ingest 的 RerankerClient

入库服务使用独立的 `RerankerClient`（`kb_ingest/kb_service/services/reranker.py`），支持 `cohere` 和 `custom` 两种模式。特色功能：

- **带重试的 POST 请求**：`_post_with_retry()` 处理 429 限流，指数退避重试
- **Z-score 分数融合**：`zscore_normalize()` 方法将向量分数和 rerank 分数标准化后加权融合（`rerank_score_fusion_alpha` 控制权重）

---

## 4.6 PostgreSQL 优先策略

### 4.6.1 为什么先查 pgvector

在 KB 子图的 `local_search` 节点中，系统执行 **PostgreSQL 优先、Milvus 兜底** 的策略。设计原因：

1. **数据准确性**：PostgreSQL 存放结构化表格数据（Excel 导入），字段精确、查询可控
2. **查询速度**：pgvector 的 IVFFlat 索引在中等数据量下性能优异
3. **覆盖面广**：结构化数据覆盖了大部分常见查询场景

### 4.6.2 执行流程

`local_search` 节点（`multi_tool.py` 第 554-719 行）的完整逻辑：

```
Step 1: 检查 selected_tools 列表
   └─ 如果包含 "postgres" → 查询 PostgreSQL
       POST {INGEST_SERVICE_URL}/api/v1/knowledge/search
       payload: {"query": question, "top_k": N, "threshold": 0.5}

Step 2: PostgreSQL 结果评估
   ├─ 有结果（>= 1 条）→ 直接使用，跳过 Milvus
   └─ 无结果 → 进入 Step 3

Step 3: Milvus 兜底
   └─ 调用 knowledge_service.search(query, top_k, similarity_threshold, ...)
```

代码中的关键判断：

```python
if postgres_results and len(postgres_results) > 0:
    # PostgreSQL 有结果，直接使用，跳过 Milvus
    combined_results = postgres_results
else:
    # PostgreSQL 无结果，使用 Milvus 兜底
    docs = await knowledge_service.search(query=question, ...)
    combined_results = milvus_results
```

### 4.6.3 双重阈值过滤（PostgreSQL 侧）

PostgreSQL 结果同样经过 Reranker 处理和阈值过滤：

```python
# 如果 Reranker 启用
if knowledge_service.reranker.enabled:
    postgres_results = await knowledge_service.reranker.rerank(question, postgres_results, effective_top_k)
    # 双重阈值：similarity >= KB_POSTGRES_SIMILARITY_THRESHOLD (0.5)
    #           rerank_score >= KB_POSTGRES_RERANK_THRESHOLD (0.8)
```

---

## 4.7 KB 子图的完整工作流

KB 子图由 `create_kb_multi_tool_workflow()` 函数（`multi_tool.py`）构建，是一个 LangGraph `StateGraph`。

### 4.7.1 状态定义

```python
class KBWorkflowState(TypedDict):
    question: str                    # 用户问题
    history: List[Dict[str, str]]    # 对话历史
    guardrails_decision: str         # "proceed" | "end"
    summary: str                     # guardrails 拒绝时的说明
    route: str                       # "local" | "external" | "hybrid"
    kb_tools: List[str]             # ["postgres", "milvus"] 等
    milvus_results: List[Dict]       # Milvus 检索结果
    postgres_results: List[Dict]     # PostgreSQL 检索结果
    local_results: List[Dict]        # 合并后的本地结果
    external_results: List[Dict]     # 外部检索结果
    answer: str                      # 最终答案
    steps: Annotated[List[str], add] # 步骤追踪（自动累加）
    sources: Annotated[List[str], add]  # 来源列表（自动累加）
```

### 4.7.2 图结构

```
START
  │
  ▼
guardrails ──┬── "proceed" ──→ kb_router
              │                    │
              │              ┌─────┴─────┐
              │              ▼           ▼
              │        local_search  external_search
              │              │           │
              │              └─────┬─────┘
              │                    ▼
              │               finalize
              │                    │
              │                    ▼
              │                   END
              │
              └── "end" ──→ finalize（直接返回拒绝消息）
```

### 4.7.3 节点详解

#### Guardrails 节点

使用 LLM 结构化输出（`KBGuardrailsDecision`）判断问题是否在服务范围内：

```python
class KBGuardrailsDecision(BaseModel):
    decision: Literal["proceed", "end"]
    summary: Optional[str] = None
    rationale: Optional[str] = None
```

Prompt 指定服务范围为"菜谱文化知识库"，包含历史渊源、命名来历、地域流派、典故故事等。超出范围的问题返回 `decision="end"`，流程直接跳到 `finalize` 返回拒绝消息。

#### Router 节点

使用 LLM 结构化输出（`KBRouteDecision`）决定检索路由：

```python
class KBRouteDecision(BaseModel):
    route: Literal["local", "external", "hybrid"]
    rationale: str
    tools: List[Literal["milvus", "postgres"]]
```

路由决策规则（在 Prompt 中硬编码）：

| 场景 | route | tools |
|------|-------|-------|
| 通用历史文化查询（默认） | `local` | `["postgres", "milvus"]` |
| 明确的结构化查询 | `local` | `["postgres"]` |
| 需要长文本叙事 | `local` | `["milvus"]` |
| 需要外网资料 | `hybrid` | `["milvus"]` |
| 超出范围 | `local` | `[]` |

如果 Router 未指定 tools，默认使用 `["postgres", "milvus"]`（PostgreSQL 优先 + Milvus 兜底）。

#### local_search 节点

即 4.6 节描述的 PostgreSQL 优先检索逻辑。如果本地检索全部为空且允许外部检索，自动将 route 切换为 `"external"`。

#### external_search 节点

通过 HTTP POST 调用外部检索 API（`KB_EXTERNAL_SEARCH_URL`），支持超时配置。如果外部检索 URL 与 PostgreSQL 搜索 URL 相同，自动跳过避免重复查询。

#### finalize 节点

汇总所有检索结果，调用 LLM 生成最终答案：

```python
async def finalize(state: KBWorkflowState) -> KBOutputState:
    # 1. 如果 guardrails 拒绝，直接返回 summary
    # 2. 格式化各来源的上下文
    milvus_context = _format_milvus_results(milvus_results)
    postgres_context = _format_postgres_results(postgres_results)
    external_context = _format_external_results(external_results)
    # 3. 收集 sources
    sources = _collect_sources(milvus_results, postgres_results, external_results)
    # 4. 调用 LLM 生成答案
    messages = final_prompt.format_messages(...)
    response = await llm.ainvoke(messages)
```

---

## 4.8 答案生成：怎么带 sources

### 4.8.1 KB 子图的 finalize

finalize 的 Prompt 要求 LLM"在结尾列出引用来源名称或编号"。`_collect_sources()` 函数从每个检索结果中提取来源标识：

```python
def _collect_sources(*result_sets) -> List[str]:
    for dataset in result_sets:
        for doc in dataset or []:
            meta = doc.get("metadata") or {}
            candidate = (
                doc.get("source")
                or doc.get("source_table")
                or doc.get("document_id")
                or doc.get("source_id")
                or doc.get("id")
                or meta.get("source")
                or meta.get("url")
                or meta.get("title")
            )
            if candidate:
                collected.append(str(candidate))
    # 去重保留顺序
```

最终输出 `KBOutputState` 包含 `answer`、`sources`、`steps` 三个字段。

### 4.8.2 传统 KB 节点的 sources

`create_knowledge_query_node()`（`gustobot/application/agents/kb_tools/node.py`）使用类似的 `_collect_sources()`：

```python
def _collect_sources(documents):
    for doc in documents:
        meta = doc.get("metadata") or {}
        candidate = doc.get("source") or meta.get("source") or meta.get("url") or meta.get("id")
```

同时计算置信度：

```python
def _calculate_confidence(documents):
    scores = [doc.get("score", 0.0) for doc in documents]
    avg_score = sum(scores) / len(scores)
    return min(max(avg_score, 0.0), 1.0)
```

---

## 4.9 关键配置参数一览

所有配置项定义在 `gustobot/config/settings.py`，以环境变量方式注入。

### 入库相关

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `KB_CHUNK_SIZE` | `512` | 文本切块大小（字符数） |
| `KB_CHUNK_OVERLAP` | `80` | 切块重叠大小（字符数） |
| `KB_USE_LLM` | `True` | 入库时是否使用 LLM 改写 |
| `KB_LLM_MODEL` | `None` | 入库 LLM 模型名 |
| `KB_EMBEDDING_MODEL` | `None` | 入库 Embedding 模型名 |
| `KB_EMBEDDING_DIMENSION` | `1024` | 入库 Embedding 维度 |

### Milvus 相关

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `MILVUS_HOST` | `localhost` | Milvus 地址 |
| `MILVUS_PORT` | `19530` | Milvus 端口 |
| `MILVUS_COLLECTION` | `recipes` | 集合名称 |
| `MILVUS_INDEX_TYPE` | `IVF_FLAT` | 索引类型 |
| `MILVUS_METRIC_TYPE` | `IP` | 距离度量 |

### 检索相关

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `KB_TOP_K` | `5` | 最终返回结果数 |
| `KB_SIMILARITY_THRESHOLD` | `0.2` | 向量相似度最低阈值 |
| `KB_POSTGRES_SIMILARITY_THRESHOLD` | `0.5` | PostgreSQL 结果的相似度阈值 |
| `KB_POSTGRES_RERANK_THRESHOLD` | `0.8` | PostgreSQL 结果的 Rerank 分数阈值 |
| `KB_RERANK_SCORE_THRESHOLD` | `0.8` | Milvus 结果的 Rerank 分数阈值 |

### Reranker 相关

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `RERANK_ENABLED` | `True` | 是否启用 Reranker |
| `RERANK_PROVIDER` | `custom` | Provider：cohere/jina/voyage/custom |
| `RERANK_MODEL` | `bge-reranker-large` | Reranker 模型名 |
| `RERANK_MAX_CANDIDATES` | `20` | 向量检索阶段的召回数量 |
| `RERANK_TOP_N` | `6` | Reranker 返回数量 |
| `RERANK_TIMEOUT` | `30` | API 超时秒数 |
| `RERANK_SCORE_FUSION_ALPHA` | `None` | 向量分数与 Rerank 分数融合权重 |

### Embedding 相关

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `EMBEDDING_MODEL` | `text-embedding-3-small` | 主服务 Embedding 模型 |
| `EMBEDDING_API_KEY` | `None` | API Key（为空时回退到 `LLM_API_KEY`） |
| `EMBEDDING_BASE_URL` | `None` | 自定义 API 地址 |
| `EMBEDDING_DIMENSION` | `1536` | 向量维度 |

---

## 4.10 核心代码索引

| 文件 | 核心类/函数 | 职责 |
|------|-------------|------|
| `kb_ingest/kb_service/services/processor.py` | `DataProcessor` | Excel/MySQL 数据处理管线 |
| `kb_ingest/kb_service/services/vector_store.py` | `VectorStoreWriter` | pgvector 写入与增量更新 |
| `kb_ingest/kb_service/services/search.py` | `VectorSearcher` | pgvector 搜索 + Rerank |
| `kb_ingest/kb_service/services/reranker.py` | `RerankerClient` | 入库侧 Reranker（Cohere/Custom） |
| `gustobot/infrastructure/knowledge/embeddings.py` | `OpenAICompatibleEmbeddings` | Embedding 生成 |
| `gustobot/infrastructure/knowledge/vector_store.py` | `VectorStore` | Milvus 封装 |
| `gustobot/infrastructure/knowledge/knowledge_service.py` | `KnowledgeService` | 主服务知识库门面 |
| `gustobot/infrastructure/knowledge/reranker.py` | `Reranker` | 主服务 Reranker（4 种 Provider） |
| `gustobot/application/agents/kb_tools/node.py` | `create_knowledge_query_node()` | 传统 KB 检索节点 |
| `gustobot/application/agents/kb_tools/prompts.py` | `build_knowledge_system_prompt()` | KB 答案生成 Prompt |
| `gustobot/application/agents/kg_sub_graph/agentic_rag_agents/workflows/multi_agent/multi_tool.py` | `create_kb_multi_tool_workflow()` | KB 子图完整工作流 |

---

## 4.11 学习者自检清单

读完本阶段后，试着回答以下问题：

1. **入库流程**：Excel 文件从读取到写入 pgvector，经过了哪几个步骤？LLM 改写失败时有什么降级策略？
2. **切块策略**：`RecursiveCharacterTextSplitter` 使用了哪些分隔符？`chunk_size=512` 和 `chunk_overlap=80` 意味着什么？
3. **Embedding**：`OpenAICompatibleEmbeddings` 如何兼容不同供应商？遇到空文本怎么处理？
4. **Milvus 索引**：`IVF_FLAT` 的 `nlist=128` 和 `nprobe=10` 分别控制什么？`IP` 度量在归一化向量上等价于什么？
5. **Reranker**：系统支持哪四种 Reranker Provider？双重阈值过滤分别过滤什么？
6. **PostgreSQL 优先**：为什么先查 pgvector？什么情况下才会用 Milvus 兜底？
7. **KB 子图工作流**：guardrails、router、local_search、finalize 四个节点各自的职责是什么？
8. **sources 收集**：`_collect_sources()` 从检索结果中提取哪些字段作为来源标识？
9. **增量入库**：`VectorStoreWriter` 如何通过 `content_hash` 实现增量更新？
10. **配置调优**：如果检索结果太多噪声，应该调整哪些参数？如果召回率太低呢？

---

## 4.12 延伸阅读

- [docs/crawler_guide.md](../docs/crawler_guide.md) — Wikipedia 爬虫 CLI 和 `data/recipe.json` 批量导入工具的使用方法，本阶段未覆盖的数据入库实操内容
