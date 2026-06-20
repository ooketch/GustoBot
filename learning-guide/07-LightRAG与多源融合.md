# 阶段 7：LightRAG 和多源融合

> **目标**：能讲清楚项目不只是单一路径回答，而是多源组合。

### 阅读顺序

1. [docs/lightrag_service_guide.md](../docs/lightrag_service_guide.md) — LightRAG 服务指南
2. [lightrag_service.py](../gustobot/application/services/lightrag_service.py) — `LightRAGService` 实现
3. [data/lightrag/](../data/lightrag/) — 预构建的图索引文件

### 要理解的问题

- LightRAG 和传统 RAG 有什么区别？
- 什么问题走 LightRAG 更合适？
- local/global/hybrid 三种模式的含义？
- 多源结果冲突时怎么处理？
- 最终答案怎么融合多个工具结果？

---

## 本阶段目标

GustoBot 不是单一路径回答问题的系统。当用户提问时，系统会从多个知识源中检索信息，再将结果汇聚成一份完整的回答。本阶段将带你拆解这套多源融合架构。

学完本阶段后，你应该能回答以下问题：

- LightRAG 和传统 RAG 有什么区别？
- 项目中有哪些检索工具，各自适合什么场景？
- 多个工具的结果如何汇总成一份回答？
- 多源信息冲突时怎么处理？

---

## 1. LightRAG 是什么：和传统 RAG 的区别

### 1.1 传统 RAG 的工作方式

传统 RAG（Retrieval-Augmented Generation）的流程是：

```
用户问题 → 向量检索 → 取回相关文本块 → 拼入 prompt → LLM 生成回答
```

它只做一件事：把文本切成块（chunk），计算 embedding，查询时用余弦相似度找到最相关的几块，然后交给 LLM 总结。这种方式简单直接，但有一个根本问题——它不理解文本之间的**关系**。

### 1.2 LightRAG 的工作方式

LightRAG 在传统 RAG 的基础上加了一层**知识图谱**。它的流程是：

```
文档 → LLM 提取实体和关系 → 构建知识图谱 + 向量索引
                              ↓
用户问题 → 关键词提取 → 图谱遍历 + 向量检索 → LLM 生成回答
```

核心区别在于：

| 维度 | 传统 RAG | LightRAG |
|------|----------|----------|
| 索引结构 | 只有向量索引 | 知识图谱（GraphML）+ 向量索引 + KV 存储 |
| 检索方式 | 纯语义相似度 | 图谱遍历 + 语义搜索，可组合 |
| 跨文档推理 | 弱，依赖 chunk 相似度 | 强，通过实体关系链接不同文档 |
| 适用场景 | "这段文字说了什么" | "这个概念和哪些知识有关联" |

### 1.3 在 GustoBot 中的角色

GustoBot 用 LightRAG 处理**非结构化菜谱文本**的问答，比如"红烧肉怎么做？"这类问题。而 Neo4j 知识图谱处理**结构化实体关系**查询，比如"找出所有包含五花肉的川菜"。两者互补。

---

## 2. 检索模式详解

LightRAG 支持六种检索模式，定义在 `lightrag_service.py` 的类型别名中：

```python
SearchMode = Literal["naive", "local", "global", "hybrid", "mix", "bypass"]
```

### 2.1 各模式对比

| 模式 | 原理 | 适用场景 | 速度 | 准确度 |
|------|------|----------|------|--------|
| **naive** | 直接语义搜索，跳过图谱 | 简单关键词查询 | 最快 | 较低 |
| **local** | 局部图谱检索，关注特定实体及其邻居 | "红烧肉的食材有哪些" | 快 | 中等 |
| **global** | 全局图谱检索，综合多个知识点 | "川菜的共同特点" | 慢 | 高 |
| **hybrid** | local + global 混合 | 通用场景（**推荐默认**） | 中等 | 高 |
| **mix** | naive + hybrid 混合 | 需要最大覆盖面 | 较慢 | 最高 |
| **bypass** | 跳过检索，直接返回 | 调试/测试 | 最快 | 无 |

### 2.2 代码中的模式使用

在 `LightRAGService.query()` 中，模式通过 `QueryParam` 传给 LightRAG：

```python
param = QueryParam(
    mode=retrieval_mode,   # "naive" / "local" / "global" / "hybrid"
    top_k=k,
    stream=stream,
)
response = await self.rag.aquery(query, param=param)
```

默认模式在 `settings.py` 中配置为 `hybrid`：

```python
LIGHTRAG_RETRIEVAL_MODE: str = Field(default="hybrid", ...)
```

---

## 3. LightRAGService 实现细节

### 3.1 整体架构

`LightRAGService`（`gustobot/application/services/lightrag_service.py`）是一个**单例服务**，负责管理 LightRAG 实例的生命周期：

```
get_lightrag_service()  ←  全局单例入口
    │
    ├── __init__()        ←  读取配置，设置工作目录
    ├── initialize()      ←  检查索引文件 → 推断 embedding 维度 → 创建 LightRAG 实例 → 加载索引
    ├── query()           ←  执行查询（流式/非流式）
    ├── query_structured()←  结构化查询（返回 Pydantic 模型）
    ├── insert_documents()←  增量插入文档
    ├── get_index_stats() ←  获取索引文件统计
    └── cleanup()         ←  释放资源
```

### 3.2 初始化流程

`initialize()` 方法的执行顺序：

```python
async def initialize(self) -> None:
    # 1. 检查工作目录是否存在，不存在则创建
    if not os.path.exists(self.working_dir):
        os.makedirs(self.working_dir, exist_ok=True)

    # 2. 检查 7 个必需的索引文件
    required_files = [
        "graph_chunk_entity_relation.graphml",
        "kv_store_doc_status.json",
        "kv_store_full_docs.json",
        "kv_store_text_chunks.json",
        "vdb_chunks.json",
        "vdb_entities.json",
        "vdb_relationships.json",
    ]

    # 3. 推断 embedding 维度（优先级：环境变量 > VDB 文件推断 > 默认值 1536）
    # 4. 创建 LightRAG 实例，注入 LLM 和 embedding 函数
    self.rag = LightRAG(
        working_dir=self.working_dir,
        llm_model_func=self._llm_model_func,
        embedding_func=EmbeddingFunc(
            embedding_dim=embedding_dim,
            max_token_size=settings.LIGHTRAG_MAX_TOKEN_SIZE,
            func=self._embedding_func,
        ),
    )

    # 5. 加载预生成的索引文件
    await self.rag.initialize_storages()
    await initialize_pipeline_status()
```

### 3.3 Embedding 维度推断

这是一个值得学习的设计。代码不会盲目使用配置值，而是尝试从预生成的索引文件中推断正确的维度：

```python
def _infer_embedding_dim_from_working_dir(working_dir: str) -> Optional[int]:
    for file_name in ("vdb_chunks.json", "vdb_entities.json", "vdb_relationships.json"):
        path = Path(working_dir) / file_name
        if not path.exists():
            continue
        with path.open("r", encoding="utf-8") as f:
            payload = json.load(f)
        dim = payload.get("embedding_dim") if isinstance(payload, dict) else None
        if isinstance(dim, int) and dim > 0:
            return dim
    return None
```

推断优先级为：`EMBEDDING_DIMENSION` 环境变量 > VDB 文件中的 `embedding_dim` 字段 > 默认值 1536。这样即使部署环境的配置不完整，只要索引文件存在就能正常工作。

### 3.4 LLM 和 Embedding 函数

LightRAG 需要两个外部函数：一个用于文本生成（LLM），一个用于向量化（Embedding）。`LightRAGService` 将它们封装为 OpenAI 兼容的调用：

```python
async def _llm_model_func(self, prompt, system_prompt=None, history_messages=[], **kwargs) -> str:
    return await openai_complete_if_cache(
        model=settings.OPENAI_MODEL,
        prompt=prompt,
        system_prompt=system_prompt,
        history_messages=history_messages,
        api_key=settings.OPENAI_API_KEY,
        base_url=settings.OPENAI_API_BASE,
        **kwargs,
    )

async def _embedding_func(self, texts: List[str]) -> np.ndarray:
    # 注意：使用 .func 获取底层函数，避免 LightRAG v1.4.x 的维度检查 bug
    embed_func = getattr(openai_embed, "func", openai_embed)
    return await embed_func(
        texts=texts,
        model=settings.EMBEDDING_MODEL,
        api_key=api_key,
        base_url=base_url,
    )
```

注意 `_embedding_func` 中的 `getattr(openai_embed, "func", openai_embed)` 这一行——这是为了绕过 LightRAG v1.4.x 中 `openai_embed` 被 `EmbeddingFunc` 包装后产生的"dimension mismatch"误报。

### 3.5 流式响应的防御性处理

`query()` 方法中有一段值得注意的防御性代码：

```python
if stream:
    if inspect.isasyncgen(response):
        return response  # 正常情况：返回异步生成器

    # 防御：某些 LLM provider 即使 stream=True 也可能返回字符串
    async def one_shot() -> AsyncGenerator[str, None]:
        if response:
            yield str(response)
    return one_shot()
```

这保证了无论底层 LLM 返回什么类型，HTTP SSE 接口的契约始终稳定。

---

## 4. 预构建索引：文件结构解析

### 4.1 索引文件清单

Docker 构建时，`data/recipe.json` 中的菜谱数据通过 `LightRAG.ainsert()` 处理后，生成以下文件：

```
data/lightrag/
├── graph_chunk_entity_relation.graphml   (约 95 KB)  ← 知识图谱：实体-关系图
├── kv_store_doc_status.json              (约 28 KB)  ← 文档处理状态
├── kv_store_full_docs.json               (约 12 KB)  ← 原始文档全文
├── kv_store_text_chunks.json             (约 25 KB)  ← 文本分块
├── kv_store_full_entities.json           (约 15 KB)  ← 实体详细信息
├── kv_store_full_relations.json          (约 15 KB)  ← 关系详细信息
├── kv_store_entity_chunks.json           (约 30 KB)  ← 实体到 chunk 的映射
├── kv_store_relation_chunks.json         (约 23 KB)  ← 关系到 chunk 的映射
├── kv_store_llm_response_cache.json      (约 1.5 MB) ← LLM 调用缓存
├── vdb_chunks.json                       (约 400 KB) ← chunk 向量索引
├── vdb_entities.json                     (约 1.1 MB) ← 实体向量索引
└── vdb_relationships.json                (约 700 KB) ← 关系向量索引
```

### 4.2 三类文件的职责

**GraphML 文件**（`graph_chunk_entity_relation.graphml`）：这是 LightRAG 的核心——知识图谱。用 NetworkX 的 GraphML 格式存储，包含从菜谱文本中提取的实体（如"红烧肉"、"五花肉"、"酱油"）和它们之间的关系（如"红烧肉 --HAS_MAIN_INGREDIENT--> 五花肉"）。

**KV Store 文件**（`kv_store_*.json`）：键值对存储，保存原始数据和元信息。`kv_store_full_docs.json` 存原文，`kv_store_text_chunks.json` 存分块，`kv_store_llm_response_cache.json` 缓存 LLM 的提取结果避免重复调用。

**VDB 文件**（`vdb_*.json`）：向量数据库的 JSON 文件实现。分别存储 chunk、实体、关系的 embedding 向量，用于语义检索。

### 4.3 为什么选择"构建时索引"

项目选择在 Docker build 时生成索引，而非运行时动态构建，原因是：

1. **启动速度快**：运行时只需加载文件，不需要调用 LLM 提取实体
2. **成本可控**：LLM 调用只在构建时发生一次，不会随请求数增长
3. **数据稳定**：菜谱数据变化频率低，不需要实时更新

增量更新通过 `/api/v1/lightrag/insert` 端点支持，但这只适合小批量数据。

---

## 5. 多源融合策略

GustoBot 的检索架构不是单一管道，而是多个独立的知识源协同工作。

### 5.1 知识源全景

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户提问                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   Guardrails    │  安全护栏：过滤超出范围的问题
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │    Planner      │  任务分解：拆成多个子任务
                    └────────┬────────┘
                             │ (每个子任务独立路由)
              ┌──────────────┼──────────────┬──────────────┐
              │              │              │              │
     ┌────────▼───────┐ ┌───▼────┐ ┌───────▼──────┐ ┌────▼─────┐
     │ Text2Cypher    │ │Predef. │ │  LightRAG    │ │ Text2SQL │
     │ (Neo4j 图查询) │ │ Cypher │ │ (图谱+向量)  │ │ (MySQL)  │
     └────────┬───────┘ └───┬────┘ └───────┬──────┘ └────┬─────┘
              │              │              │              │
              └──────────────┼──────────────┼──────────────┘
                             │
                    ┌────────▼────────┐
                    │    Summarize    │  结果聚合
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Final Answer   │  生成最终回答
                    └─────────────────┘
```

### 5.2 五种检索工具详解

#### (1) Text2Cypher — Neo4j 知识图谱查询

- **数据源**：Neo4j 图数据库
- **数据类型**：结构化实体关系（菜品、食材、口味、烹饪方法、营养档案等）
- **工作方式**：LLM 将自然语言转为 Cypher 查询语句，在 Neo4j 中执行
- **适合问题**："红烧肉的主料有哪些"、"哪些菜用到了蒸的烹饪方法"

代码位置：`components/cypher_tools/node.py`

```python
# 核心流程：生成 Cypher → 验证 → 执行
cypher_generation = create_text2cypher_generation_node(llm=model, graph=neo4j_graph, ...)
cypher_result = await cypher_generation(state)

validate_cypher = create_text2cypher_validation_node(llm=model, graph=neo4j_graph, ...)
execute_info = await validate_cypher(state)

execute_cypher = create_text2cypher_execution_node(graph=neo4j_graph, cypher=execute_info)
final_result = await execute_cypher(state)
```

#### (2) Predefined Cypher — 预定义模板查询

- **数据源**：Neo4j 图数据库
- **数据类型**：同 Text2Cypher，但使用预写好的查询模板
- **工作方式**：通过向量匹配找到最合适的预定义模板，用 LLM 提取参数后执行
- **适合问题**：高频、模式固定的查询

代码位置：`components/predefined_cypher/node.py`

```python
# 通过向量匹配找到最接近的预定义查询
matcher = create_vector_query_matcher(predefined_cypher_dict, QUERY_DESCRIPTIONS)
matches = matcher.match_query(question, top_k=1)
query_name = matches[0]["query_name"]

# 用 LLM 从问题中提取参数
parameters = matcher.extract_parameters(question, query_name, llm=chat_llm)

# 执行预定义的 Cypher
records = graph.query(query=statement, params=parameters)
```

#### (3) LightRAG — 图谱增强语义检索

- **数据源**：预构建的 LightRAG 索引（GraphML + VDB + KV Store）
- **数据类型**：非结构化菜谱文本
- **工作方式**：结合知识图谱遍历和向量语义搜索
- **适合问题**："红烧肉怎么做"、"宫保鸡丁的来历"

代码位置：`components/customer_tools/node.py`

```python
class LightRAGAPI:
    async def query(self, query: str, mode: Optional[str] = None) -> Dict[str, Any]:
        param = QueryParam(mode=retrieval_mode, top_k=self.top_k)
        response = await self.rag.aquery(query, param=param)
        return {"response": response, "mode": retrieval_mode, "query": query}
```

注意：`customer_tools/node.py` 文件末尾有两行别名：

```python
create_graphrag_query_node = create_lightrag_query_node
GraphRAGAPI = LightRAGAPI
```

这说明项目从 Microsoft GraphRAG 迁移到了 LightRAG，但保留了旧名称的兼容性。

#### (4) Text2SQL — MySQL 结构化查询

- **数据源**：MySQL 关系数据库
- **数据类型**：结构化表格数据（统计、报表）
- **工作方式**：LLM 将自然语言转为 SQL，在 MySQL 中执行
- **适合问题**："菜谱数量统计"、"每种菜系有多少道菜"

代码位置：`components/text2cypher/text2sql_tool.py`

```python
workflow = create_text2sql_workflow(
    llm=text2sql_llm,
    neo4j_graph=graph,       # 用于获取 schema 信息
    db_type=db_type,         # 默认 MySQL
    connection_string=connection_string,
    max_retries=max_retries,
)
result = await workflow.ainvoke(input_state)
```

#### (5) External Search — 外部检索

- **数据源**：外部 HTTP API
- **数据类型**：外部知识库或网络搜索结果
- **工作方式**：将问题发送到外部 API，获取补充信息
- **适合问题**：本地知识库无法覆盖的查询

这在 KB（知识库）工作流中实现（`multi_tool.py` 中的 `create_kb_multi_tool_workflow`）。

### 5.3 工具选择机制

工具选择在 `components/tool_selection/node.py` 中实现，采用**启发式规则 + LLM 决策**的两层策略：

**第一层：启发式快速路径**

```python
# 描述类关键词 → 直接走 LightRAG
DESCRIPTIVE_KEYWORDS = ["口味", "特色", "风味", "营养", "功效", ...]
if any(keyword in question_text for keyword in DESCRIPTIVE_KEYWORDS):
    return _make_command("customer_tools", ...)  # → LightRAG

# SQL 类关键词 + 路由类型匹配 → 直接走 Text2SQL
if _looks_like_sql_question(question_text) and route_type == "text2sql-query":
    return _make_command("text2sql_query", ...)
```

**第二层：LLM 结构化决策**

如果启发式规则没有命中，则让 LLM 从可用工具中选择：

```python
tool_selection_chain = (
    tool_selection_prompt
    | llm.bind_tools(tools=tool_schemas)
    | PydanticToolsParser(tools=tool_schemas, first_tool_only=True)
)
tool_selection_output = await tool_selection_chain.ainvoke({"question": question_text})
```

LLM 的系统提示（`kg_prompts.py`）明确了工具优先级：

> - `cypher_query`：需要动态生成 Cypher 时使用，涵盖绝大多数图谱问答
> - `predefined_cypher`：当问题命中预设模板时直接复用
> - `text2sql_query`：当用户提出统计、报表类问题时选择
> - 其他自定义工具（如 LightRAG）仅在问题明确需要长文档推理或外部知识时使用

**兜底策略**：如果 LLM 也没有给出明确选择，默认回退到 Text2Cypher：

```python
if default_to_text2cypher:
    return go_to_text2cypher
```

---

## 6. Summarize 节点：多工具结果聚合

### 6.1 聚合逻辑

不管用户的问题被路由到了哪个工具，所有工具的输出都会汇聚到 `summarize` 节点。看图中的边定义：

```python
main_graph_builder.add_edge("cypher_query", "summarize")
main_graph_builder.add_edge("predefined_cypher", "summarize")
main_graph_builder.add_edge("customer_tools", "summarize")
main_graph_builder.add_edge("text2sql_query", "summarize")
```

四条边，四个工具，全部指向同一个 `summarize` 节点。

### 6.2 Summarize 节点的工作方式

`summarize` 节点（`components/summarize/node.py`）不做检索，它只负责**整理**。核心逻辑：

```python
async def summarize(state: OverallState) -> Dict[str, Any]:
    tasks = state.get("tasks", [])
    cypher_entries = state.get("cyphers", [])

    narrative_sections: List[str] = []   # 叙述性结果
    metric_sections: List[str] = []      # 数据统计结果
    error_sections: List[str] = []       # 错误信息

    for idx, cypher in enumerate(cypher_entries):
        records = data.get("records") or {}
        errors = data.get("errors") or []

        if errors:
            error_sections.append(...)
            continue

        # 处理不同类型的 records
        if isinstance(records, dict):
            result = records.get("result")      # LightRAG 返回的文本
            answer = records.get("answer")      # Text2SQL 返回的答案
            rows = records.get("rows")          # Cypher 查询返回的行数据

    # 组装最终 summary
    sections = []
    if narrative_sections:
        sections.append("### 川菜概览\n" + "\n\n".join(narrative_sections))
    if metric_sections:
        sections.append("### 数据统计\n" + "\n\n".join(metric_sections))
    if error_sections:
        sections.append("### 查询提示\n" + "\n".join(...))
```

关键点在于 `_format_rows()` 函数，它能识别不同类型的数据并格式化：

```python
def _format_rows(rows: List[dict]) -> str:
    # 检测是否是烹饪步骤（含"步骤序号"和"步骤说明"）
    is_cooking_steps = all("步骤序号" in row and "步骤说明" in row for row in rows)

    # 检测是否是食材列表（含"食材"和"用量"）
    is_ingredients = all("食材" in row and "用量" in row for row in rows)

    # 根据类型选择不同的格式化策略
```

### 6.3 Summarize 的 Prompt

`summarize` 节点使用 LLM 做最终的文本整理，prompt 定义在 `components/summarize/prompts.py`：

```python
ChatPromptTemplate.from_messages([
    ("system", "你是一位懂烹饪、语气亲和的菜谱指南助手..."),
    ("human", (
        "事实信息：{results}\n\n"
        "用户问题：{question}\n\n"
        "请根据上述事实信息生成菜谱解读..."
    )),
])
```

注意 prompt 中的约束："当事实信息不为空时，仅依据这些内容组织回答，**绝不编造**。"这是防止 LLM 幻觉的关键指令。

---

## 7. Final Answer：生成带来源标注的回答

### 7.1 Final Answer 节点

`final_answer` 节点（`components/final_answer/node.py`）非常简洁——它不调用 LLM，只是把 `summary` 传递出去，并记录历史：

```python
async def final_answer(state: OverallState) -> dict[str, Any]:
    answer = state.get("summary", " ")

    history_record = {
        "question": state.get("question", ""),
        "answer": answer,
        "cyphers": [
            {
                "task": c.task if hasattr(c, "task") else c.get("task", ""),
                "records": c.records if hasattr(c, "records") else c.get("records", {}),
            }
            for c in state.get("cyphers", list())
        ],
    }

    return {
        "answer": answer,
        "steps": ["final_answer"],
        "history": [history_record],
    }
```

### 7.2 来源追踪机制

来源信息通过 `cyphers` 字段传递。每个工具执行后都会将自己的查询记录写入 `state["cyphers"]`，格式为 `CypherOutputState`：

```python
class CypherOutputState(TypedDict):
    task: str           # 子任务描述
    statement: str      # 执行的查询语句（Cypher/SQL/LightRAG 模式）
    parameters: Any     # 查询参数
    errors: List[str]   # 错误信息
    records: Dict       # 查询结果
    steps: List[str]    # 执行步骤
```

在 KB 工作流中，来源追踪更加显式。`finalize` 节点会从所有检索结果中提取来源：

```python
sources = _collect_sources(milvus_results, postgres_results, external_results)
# 返回的 sources 列表包含所有检索到的文档来源
```

并且在最终 prompt 中要求 LLM "在结尾列出引用来源名称或编号"。

### 7.3 KB 工作流的多源融合

除了主 Agent 的多工具路由，项目还有一个独立的 KB（知识库）工作流（`multi_tool.py` 中的 `create_kb_multi_tool_workflow`），实现了更精细的多源融合：

```
用户问题
    │
    ▼
Guardrails（安全审查）
    │
    ▼
Router（路由决策：local / external / hybrid）
    │
    ├── local_search（本地检索）
    │   ├── PostgreSQL pgvector（优先，结构化数据）
    │   └── Milvus 向量库（兜底，非结构化文本）
    │
    ├── external_search（外部检索）
    │
    ▼
Finalize（融合三路结果生成回答）
```

这个工作流的融合策略在 `local_search` 节点中：

```python
# Step 1: 优先查询 PostgreSQL
if should_try_postgres:
    postgres_results = await search_postgres(question, ...)

# Step 2: PostgreSQL 有结果 → 直接使用，跳过 Milvus
if postgres_results and len(postgres_results) > 0:
    combined_results = postgres_results
else:
    # PostgreSQL 无结果 → Milvus 兜底
    if should_try_milvus:
        milvus_results = await knowledge_service.search(question, ...)
        combined_results = milvus_results
```

最终的 `finalize` 节点将三路结果分别格式化后交给 LLM：

```python
messages = final_prompt.format_messages(
    question=state.get("question", ""),
    milvus_context=milvus_context,       # Milvus 检索结果
    postgres_context=postgres_context,    # PostgreSQL 检索结果
    external_context=external_context,    # 外部检索结果
)
response = await llm.ainvoke(messages)
```

---

## 8. 冲突处理：多源信息不一致时怎么办

### 8.1 当前的冲突处理策略

GustoBot 采用的是**LLM 裁决**策略——不是用程序硬编码规则来解决冲突，而是把所有来源的结果都交给 LLM，由 LLM 在生成回答时自行判断如何融合。

具体体现在 KB 工作流的 `finalize` prompt 中：

```python
"你是菜谱文化讲解助手，需要依据给定检索结果作答。请遵循：\n"
"...\n"
"5. 区分并融合来自不同数据源的要点，避免重复叙述。\n"
"6. 在结尾列出引用来源名称或编号（如有）。"
```

### 8.2 优先级隐含在数据呈现顺序中

虽然 prompt 没有显式说"哪个来源优先"，但数据的呈现顺序暗示了优先级：

1. **Milvus 向量检索结果**（先展示）
2. **PostgreSQL 结构化检索结果**（其次）
3. **外部检索结果**（最后）

在主 Agent 的 `summarize` 节点中，数据按任务顺序排列，每个任务的结果带有明确的任务标签，LLM 可以据此判断信息的上下文。

### 8.3 信息不足时的兜底

当所有检索源都没有结果时，系统会返回预设的兜底消息：

```python
# KB 工作流
if not local_results and not external_results:
    fallback = "抱歉，菜谱文化知识库暂未找到相关记载，请尝试描述得更具体一些或稍后再试。"

# 主 Agent Guardrails
if decision == "end":
    summary = "抱歉，暂时没有关于该菜谱的消息，可以在问别的哦~"
```

### 8.4 Summarize 节点中的错误处理

当某个工具执行失败时，错误信息不会被丢弃，而是被归入"查询提示"板块：

```python
if errors:
    error_sections.append(f"{task_label}：{'；'.join(errors)}")

# 最终输出中包含错误信息
if error_sections:
    sections.append("### 查询提示\n" + "\n".join(f"- {msg}" for msg in error_sections))
```

这样 LLM 在生成回答时能感知到哪些工具失败了，从而在回答中做出适当说明（如"结构化数据暂不可用，以下信息来自文本检索"）。

---

## 9. 完整请求流程示例

以"红烧肉怎么做？有哪些主料？"为例，走一遍完整流程：

```
1. Guardrails 节点
   → 检测到"菜"关键词，命中启发式规则
   → next_action = "planner"

2. Planner 节点
   → LLM 将问题拆解为两个子任务：
     - Task A: "红烧肉的烹饪步骤是什么"
     - Task B: "红烧肉的主料有哪些"

3. Tool Selection（对每个子任务独立执行）

   Task A → 启发式：无描述类关键词
          → LLM 选择：LightRAG（需要长文档推理）
          → 路由到 customer_tools 节点

   Task B → 启发式：无描述类关键词
          → LLM 选择：cypher_query（结构化关系查询）
          → 路由到 cypher_query 节点

4. 并行执行
   customer_tools → LightRAG hybrid 搜索 → 返回烹饪步骤文本
   cypher_query   → 生成 Cypher → Neo4j 查询 → 返回食材列表

5. Summarize 节点
   → 汇总两个任务的结果
   → 检测到 Task B 的结果是食材列表，使用 ★ 标记主料
   → 调用 LLM 生成自然语言 summary

6. Final Answer 节点
   → 输出 answer = summary
   → 记录 history（含两个任务的查询记录和结果）
```

---

## 10. 学习者自检清单

完成本阶段学习后，请尝试回答以下问题：

**基础概念**

- [ ] LightRAG 的知识图谱是如何构建的？（提示：LLM 提取实体和关系）
- [ ] `hybrid` 模式为什么比单独的 `local` 或 `global` 更好？
- [ ] 预构建索引中的 `.graphml` 文件存储了什么？

**架构理解**

- [ ] GustoBot 中有哪些检索工具？各自的适用场景是什么？
- [ ] 工具选择的两层策略是什么？（启发式 + LLM 决策）
- [ ] 为什么所有工具的输出都汇聚到同一个 `summarize` 节点？

**代码细节**

- [ ] `LightRAGService` 如何推断 embedding 维度？为什么需要这个机制？
- [ ] `_embedding_func` 中的 `getattr(openai_embed, "func", openai_embed)` 是为了解决什么问题？
- [ ] `summarize` 节点的 `_format_rows()` 如何识别烹饪步骤和食材列表？

**多源融合**

- [ ] KB 工作流中，PostgreSQL 和 Milvus 的优先级关系是什么？
- [ ] 当多个来源的信息冲突时，系统采用什么策略处理？
- [ ] 来源追踪是如何实现的？最终回答中如何标注引用来源？

**动手练习**

- [ ] 尝试调用 `/api/v1/lightrag/test-modes?query=红烧肉怎么做` 对比四种模式的结果差异
- [ ] 阅读 `components/tool_selection/prompts.py` 中的 `TOOL_SELECTION_SYSTEM_PROMPT`，理解 LLM 如何选择工具
- [ ] 在 `summarize/node.py` 中添加日志，观察多任务结果是如何被分类为 `narrative_sections` 和 `metric_sections` 的
