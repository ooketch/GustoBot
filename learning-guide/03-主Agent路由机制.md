# 阶段 3：主 Agent 路由机制 ⭐

> **目标**：能讲清楚系统怎么判断一个问题该走哪个能力。这是整个项目的核心设计。

### 阅读顺序

1. [docs/agent_routing_quick_reference.md](../docs/agent_routing_quick_reference.md) — 路由速查表，先建立整体认知
2. [lg_states.py](../gustobot/application/agents/lg_states.py) — `Router` Pydantic 模型（7 种路由类型）、`InputState`、`AgentState`
3. [lg_builder.py](../gustobot/application/agents/lg_builder.py) — 重点看三个部分：
   - `analyze_and_route_query`（第 103-236 行）：LLM 分类 + `_heuristic_router` 关键词兜底
   - `route_query`（第 197-236 行）：条件路由函数
   - 图的构建（第 958-973 行）：`StateGraph` 怎么组装
4. [lg_prompts.py](../gustobot/application/agents/lg_prompts.py) — `ROUTER_SYSTEM_PROMPT`，看它怎么引导 LLM 做分类

### 动手任务

画出完整的路由决策树：

```
用户问题
  → analyze_and_route_query (LLM structured_output + 关键词兜底)
  → route_query 条件分发
    ├─ general-query    → respond_to_general_query (纯 LLM 闲聊)
    ├─ additional-query → get_additional_info (Guardrails 检查 + 引导补充)
    ├─ kb-query         → create_kb_query (知识库子图)
    ├─ graphrag-query   → create_research_plan (KG 子图)
    ├─ text2sql-query   → create_research_plan (KG 子图)
    ├─ image-query      → create_image_query (图片识别/生成)
    └─ file-query       → create_file_query (文件解析)
```

### 要理解的问题

- 路由靠规则还是 LLM？
- 为什么要有启发式路由和 LLM 路由双保险？
- "红烧肉怎么做" 和 "数据库里有多少道川菜" 分别怎么路由？
- 如果路由错了，怎么调试和优化？

---

## 本文目标

本文将带你深入理解 GustoBot 的核心设计——**路由机制**。学完之后，你应该能讲清楚以下三个问题：

1. 系统怎么判断一个问题该走哪条路？
2. 判断的依据是什么？有几层保障？
3. 如果路由出错，怎么排查？

---

## 1. 从一个用户问题说起

假设用户说："红烧肉怎么做？"

系统需要回答的核心问题是：**这个问题应该交给谁来处理？**

- 如果是"你好"这种闲聊，直接用 LLM 回复就够了
- 如果是"红烧肉怎么做"，需要去知识库检索
- 如果是"统计有多少道川菜"，需要去数据库执行 SQL
- 如果是"生成一张红烧肉的图片"，需要调用图片生成 API

这个"交给谁"的决策过程，就是**路由**。

---

## 2. 七种路由类型详解

GustoBot 定义了 7 种路由类型，每种类型对应不同的处理能力。这些类型在两个地方同时定义，必须保持一致：

- **状态模型**：`lg_states.py` 第 12-19 行的 `Router.type` 字段
- **路由函数**：`lg_builder.py` 第 199-200 行的 `route_query` 返回值签名

### 2.1 general-query（闲聊/通用）

**处理什么**：问候、感谢、情绪表达、与菜谱无关的简短对话。

**走哪个节点**：`respond_to_general_query`（`lg_builder.py` 第 239-265 行）

**处理方式**：纯 LLM 生成，不调用任何外部服务。使用 `GENERAL_QUERY_SYSTEM_PROMPT` 模板，注入路由分类理由（`logic` 字段），让 LLM 用亲切语气回复。

**典型问题**：
- "你好"、"早上好"、"谢谢"
- "今天天气真好"
- "讲个笑话"

### 2.2 additional-query（补充信息）

**处理什么**：用户提问模糊、缺少关键信息，需要反问用户。

**走哪个节点**：`get_additional_info`（`lg_builder.py` 第 268-379 行）

**处理方式**：这个节点有一个特殊的 **Guardrails 检查**（守护栏）。流程如下：

```
用户问题 → Guardrails 检查 → 是否在业务范围内？
                                  ├─ "proceed" → 用 LLM 生成引导式提问
                                  └─ "end" → 返回礼貌拒绝
```

Guardrails 检查通过 `GUARDRAILS_SYSTEM_PROMPT`（`lg_prompts.py` 第 192-215 行）指导 LLM 判断问题是否与菜谱业务相关。输出只有两种：`"continue"` 或 `"end"`。

**典型问题**：
- "我想做菜"（没说什么菜）→ proceed，询问具体菜名
- "今天天气怎么样" → end，礼貌拒绝

### 2.3 kb-query（知识库查询）

**处理什么**：需要从向量知识库检索长文本内容的问题，包括菜谱做法、历史文化、烹饪技巧等。这是**最常见的路由类型**。

**走哪个节点**：`create_kb_query`（`lg_builder.py` 第 711-801 行）

**处理方式**：调用 `create_kb_multi_tool_workflow` 创建多工具工作流，可能涉及 Milvus 向量检索、pgvector 检索、外部搜索，以及 Reranker 重排序。如果多工具工作流不可用，会降级到直接知识库查询（fallback，第 783-801 行）。

**典型问题**：
- "宫保鸡丁的历史典故"
- "川菜的特点"
- "西兰花的营养价值"

### 2.4 graphrag-query（图谱推理查询）

**处理什么**：需要查询知识图谱（Neo4j）进行结构化关系推理的问题。

**走哪个节点**：`create_research_plan`（`lg_builder.py` 第 804-910 行）

**处理方式**：创建多工具工作流，由 Planner（规划器）决定使用哪些工具组合：
- `predefined_cypher`：预定义的 Cypher 查询模板（高频场景）
- `cypher_query`：LLM 动态生成 Cypher 查询
- `microsoft_graphrag_query`：LightRAG 图推理
- `text2sql_query`：MySQL 结构化查询

**典型问题**：
- "红烧肉怎么做"（做法步骤）
- "宫保鸡丁需要哪些食材"
- "怎么判断鱼熟了"（烹饪技巧推理）

**重要说明**：`graphrag-query` 和 `text2sql-query` 在路由函数中合并到同一个节点。参见 `route_query` 第 227 行：

```python
elif _type in ("graphrag-query", "text2sql-query"):  # 图查询或结构化问数
    return "create_research_plan"
```

这意味着这两种类型共用 `create_research_plan` 节点，由节点内部的 Planner 进一步决定具体使用哪些工具。

### 2.5 text2sql-query（结构化数据查询）

**处理什么**：需要对数据库进行统计、计数、排名、聚合分析的问题。

**走哪个节点**：`create_research_plan`（与 graphrag-query 相同）

**处理方式**：由 Planner 选择 `text2sql_query` 工具，LLM 生成 SQL 语句，执行 MySQL 查询。

**典型问题**：
- "数据库里有多少道菜"
- "哪个菜系菜谱最多"
- "统计每个口味的数量"

### 2.6 image-query（图片处理）

**处理什么**：用户上传图片请求识别，或请求生成图片。

**走哪个节点**：`create_image_query`（`lg_builder.py` 第 478-619 行）

**处理方式**：根据请求类型分为两种模式：
- **识别模式**：调用 Vision API 分析图片，生成描述，再用 LLM 结合描述回答
- **生成模式**：LLM 优化 prompt，调用 CogView-4 API 生成图片

判断依据是关键词匹配（第 495-496 行）：
```python
generation_keywords = ["生成", "画", "创建", "制作图片", "做一张", "给我一张", "来一张"]
is_generation = any(keyword in user_query for keyword in generation_keywords)
```

**典型问题**：
- "这是什么菜"（附图）→ 识别
- "生成一张红烧肉的图片" → 生成

### 2.7 file-query（文件上传）

**处理什么**：用户上传菜谱文档、Excel 等文件。

**走哪个节点**：`create_file_query`（`lg_builder.py` 第 622-709 行）

**处理方式**：根据文件类型分流：
- **Excel 文件**（.xlsx/.xls）：转发给外部 Ingest Service 处理
- **文本文件**（.txt/.md/.json/.csv/.log）：读取内容，写入知识库，再用知识库查询回答

**典型问题**：
- "分析这个菜谱文件"（附 .txt 文件）
- "导入这份食材清单"（附 .xlsx 文件）

---

## 3. 双重路由机制

GustoBot 的路由不是单层决策，而是有两层保障：**LLM structured_output 路由**（主）+ **启发式关键词路由**（备）。

### 3.1 第一层：LLM Structured Output 路由

这是路由的主路径。在 `analyze_and_route_query` 函数（`lg_builder.py` 第 103-194 行）中：

**Step 1 — 构建 Prompt**（第 131-133 行）：

```python
messages = [
    {"role": "system", "content": ROUTER_SYSTEM_PROMPT}
] + state.messages
```

将系统提示词（`ROUTER_SYSTEM_PROMPT`，定义在 `lg_prompts.py` 第 7-66 行）与用户对话历史拼接。注意这里包含了**完整的历史消息**，不是只看最新一条，所以多轮对话的上下文会影响路由判断。

**Step 2 — LLM Structured Output 调用**（第 156 行）：

```python
raw_response = await model.with_structured_output(Router).ainvoke(messages)
```

关键点：使用 `with_structured_output(Router)` 方法，强制 LLM 输出符合 `Router` Pydantic 模型的结构化数据。这意味着 LLM 必须返回包含 `type`、`logic` 等字段的 JSON，而不是自由文本。

**Step 3 — 校验与净化**（第 161-193 行）：

```python
response = raw_response if isinstance(raw_response, Router) else Router.model_validate(raw_response)
router_type = response.type
```

检查返回的 `type` 是否在允许列表中（第 145-153 行定义了 7 种合法类型）。如果不在列表中，走启发式 fallback。

### 3.2 第二层：启发式关键词路由（_heuristic_router）

定义在 `lg_builder.py` 第 986-1022 行。

这是一层**纯代码、无 LLM 调用**的兜底机制。当 LLM 路由失败（异常或返回非法类型）时启用。

**关键词列表**：

```python
text2sql_keywords = ["统计", "多少", "总数", "数量", "排名"]  # 优先级更高
graphrag_keywords = [
    "怎么做", "如何做", "做法", "步骤", "火候",
    "食材", "原料", "需要什么", "配料", "用什么",
]
```

**匹配逻辑**（注意优先级）：

```python
if any(keyword in lowered for keyword in text2sql_keywords):
    return Router(type="text2sql-query", ...)   # 先检查 text2sql

if any(keyword in lowered for keyword in graphrag_keywords):
    return Router(type="graphrag-query", ...)   # 再检查 graphrag
```

text2sql 关键词优先级高于 graphrag 关键词。如果都没有匹配到，返回 `None`。

### 3.3 两层机制的协作关系

```
用户问题
    │
    ▼
analyze_and_route_query()
    │
    ├─ Step 1: 计算启发式路由结果（作为 fallback 预备）
    │     └─ _heuristic_router() → 可能返回 Router 或 None
    │
    ├─ Step 2: 调用 LLM structured_output
    │     ├─ 成功且 type 合法 → 直接使用 LLM 结果
    │     ├─ 成功但 type 不合法 → 使用启发式结果（如果有的话）
    │     └─ 调用异常 → 使用启发式结果（如果有的话）
    │           └─ 启发式也没匹配 → 默认 kb-query
    │
    ▼
返回 Router 对象
```

代码对应（第 138-143 行）：

```python
heuristic_router = _heuristic_router(question_text)
fallback_router: Router = heuristic_router or Router(
    type="kb-query",
    logic="fallback: default to knowledge base routing",
    question=question_text,
)
```

默认 fallback 是 `kb-query`，因为知识库查询是覆盖面最广、最安全的选择。

---

## 4. Router Pydantic 模型字段解析

定义在 `lg_states.py` 第 8-31 行：

```python
class Router(BaseModel):
    """Classify user query (同时兼容旧字段/属性访问)."""
    logic: str = ""                          # 分类理由，LLM 解释为什么这样分类
    type: Literal[                           # 路由类型，必须是 7 种之一
        "general-query",
        "additional-query",
        "kb-query",
        "graphrag-query",
        "image-query",
        "file-query",
        "text2sql-query",
    ] = "kb-query"                           # 默认值：kb-query（最安全的 fallback）
    question: str = ""                       # 用户原始问题
    decision: Optional[str] = None           # 可选：额外决策信息
    confidence: Optional[float] = None       # 可选：置信度（0-1）
    reasoning: Optional[str] = None          # 可选：推理过程
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | Literal[7种] | 是 | 路由类型，默认 `kb-query` |
| `logic` | str | 是 | LLM 给出的分类理由，用于日志和调试 |
| `question` | str | 是 | 用户原始问题文本 |
| `decision` | Optional[str] | 否 | 额外决策信息，当前未使用 |
| `confidence` | Optional[float] | 否 | 置信度，当前未使用 |
| `reasoning` | Optional[str] | 否 | 推理过程，当前未使用 |

**设计要点**：

1. **默认值 `kb-query`**（第 20 行）：如果 LLM 没有返回有效类型，默认走知识库查询，因为这是覆盖面最广的路径。
2. **字典风格访问兼容**（第 29-31 行）：`get()` 方法兼容旧代码中的字典访问方式。
3. **Config extra = "allow"**（第 27-28 行）：允许 LLM 返回额外字段，不会因为多余字段报错。

还有一个相关的 `RouteResult` 数据类（第 34-40 行）：

```python
@dataclass(kw_only=True)
class RouteResult:
    route: str
    confidence: float = 0.0
    next_node: str = ""
    metadata: Dict[str, Any] = field(default_factory=dict)
```

这是下游节点使用的路由结果封装，目前主要在多工具工作流内部使用。

---

## 5. route_query 条件路由的实现

`route_query` 函数（`lg_builder.py` 第 197-236 行）是 LangGraph 图中的**条件边函数**，负责将 `Router.type` 映射到具体的下游节点名称。

### 5.1 函数签名

```python
def route_query(
    state: AgentState,
) -> Literal[
    "respond_to_general_query",
    "get_additional_info",
    "create_research_plan",
    "create_image_query",
    "create_file_query",
    "create_kb_query"
]:
```

返回值是 6 个节点名称之一。注意：虽然有 7 种路由类型，但 `graphrag-query` 和 `text2sql-query` 都映射到 `create_research_plan`，所以只有 6 个目标节点。

### 5.2 路由映射表

| Router.type | 目标节点 | 说明 |
|-------------|---------|------|
| `general-query` | `respond_to_general_query` | 闲聊回复 |
| `additional-query` | `get_additional_info` | 补充信息 |
| `graphrag-query` | `create_research_plan` | 图谱推理 |
| `text2sql-query` | `create_research_plan` | 结构化查询 |
| `image-query` | `create_image_query` | 图片处理 |
| `file-query` | `create_file_query` | 文件处理 |
| `kb-query` | `create_kb_query` | 知识库查询 |

### 5.3 图片/文件的特殊优先级

`route_query` 中有一个重要的设计：**图片和文件路径的检测优先于 LLM 路由结果**（第 214-220 行）：

```python
if hasattr(state, "config") and state.config:
    cfg = state.config.get("configurable", {})
    if cfg.get("image_path"):
        logger.info("检测到图片路径，转为图片查询处理")
        return "create_image_query"
    if cfg.get("file_path"):
        logger.info("检测到文件路径，转为文件上传处理")
        return "create_file_query"
```

这意味着：如果用户上传了图片或文件，无论 LLM 路由结果是什么，都会优先走对应的处理节点。这是一个实用的设计——用户说"这是什么菜"并附带图片时，即使 LLM 可能分类为 `kb-query`，系统也能正确路由到图片处理。

### 5.4 在 LangGraph 图中的注册

```python
builder.add_edge(START, "analyze_and_route_query")                          # 第 970 行
builder.add_conditional_edges("analyze_and_route_query", route_query)       # 第 971 行
```

`add_conditional_edges` 告诉 LangGraph：`analyze_and_route_query` 节点执行完毕后，调用 `route_query` 函数决定下一步走哪个节点。

---

## 6. 每个 Handler 节点的职责概览

| 节点名 | 函数 | 位置 | 核心职责 |
|--------|------|------|---------|
| `analyze_and_route_query` | `analyze_and_route_query()` | 第 103-194 行 | 意图识别：分析用户问题，输出 Router 分类结果 |
| `respond_to_general_query` | `respond_to_general_query()` | 第 239-265 行 | 闲聊回复：纯 LLM 生成，不调用外部服务 |
| `get_additional_info` | `get_additional_info()` | 第 268-379 行 | 补充信息：Guardrails 检查 + 引导式提问 |
| `create_research_plan` | `create_research_plan()` | 第 804-910 行 | 图谱/SQL 查询：多工具工作流，Planner 选工具 |
| `create_image_query` | `create_image_query()` | 第 478-619 行 | 图片处理：Vision 识别 或 CogView-4 生成 |
| `create_file_query` | `create_file_query()` | 第 622-709 行 | 文件处理：Excel 转外部服务，文本写入知识库 |
| `create_kb_query` | `create_kb_query()` | 第 711-801 行 | 知识库查询：多工具向量检索 + Reranker |

每个节点的详细实现将在后续阶段的文档中深入讲解。

---

## 7. 路由决策的完整流程图

```
用户发送消息
    │
    ▼
┌─────────────────────────────────┐
│  LangGraph: START               │
│  add_edge(START,                │
│    "analyze_and_route_query")   │
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│  analyze_and_route_query                    │
│                                             │
│  1. 提取用户最新消息                          │
│  2. 调用 _heuristic_router() 预计算         │
│  3. 拼接 system prompt + 历史消息            │
│  4. 调用 LLM with_structured_output(Router) │
│  5. 校验 type 是否在 7 种合法类型中           │
│     ├─ 合法 → 使用 LLM 结果                  │
│     └─ 非法 → 使用启发式结果或默认 kb-query   │
│  6. 返回 {"router": Router(...)}            │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│  route_query (条件路由)                      │
│                                             │
│  优先检查: image_path / file_path            │
│  然后根据 router.type 分发:                  │
│                                             │
│  general-query ──→ respond_to_general_query │
│  additional-query → get_additional_info     │
│  kb-query ────────→ create_kb_query         │
│  graphrag-query ──→ create_research_plan    │
│  text2sql-query ──→ create_research_plan    │
│  image-query ─────→ create_image_query      │
│  file-query ──────→ create_file_query       │
└─────────────────────────────────────────────┘
    │
    ▼ (根据路由结果，进入对应节点)
    │
    ├─→ respond_to_general_query → LLM 直接回复 → END
    │
    ├─→ get_additional_info
    │       ├─ Guardrails "end" → 拒绝回复 → END
    │       └─ Guardrails "proceed" → 引导提问 → END
    │
    ├─→ create_kb_query → 多工具向量检索 → END
    │       └─ (fallback: 直接知识库查询)
    │
    ├─→ create_research_plan → Planner 选工具 → 多工具执行 → END
    │       ├─ predefined_cypher
    │       ├─ cypher_query
    │       ├─ microsoft_graphrag_query
    │       └─ text2sql_query
    │
    ├─→ create_image_query → 识别/生成 → END
    │
    └─→ create_file_query → 读取/导入 → END
```

---

## 8. 常见问题的路由走向分析

以下分析基于实际代码逻辑，帮助你建立直觉。

### 8.1 "你好"

- LLM 路由：`general-query`
- 启发式：无关键词匹配
- 最终路径：`respond_to_general_query` → LLM 直接回复
- 不调用任何外部服务

### 8.2 "红烧肉怎么做"

- LLM 路由：`graphrag-query`（当前 prompt 的定义）
- 启发式：匹配"怎么做" → `graphrag-query`
- 最终路径：`create_research_plan` → Planner 选择工具

**注意**：这是一个容易出错的场景。`ROUTER_PROMPT_FIX.md` 文件记录了历史问题——早期 prompt 将"怎么做"类问题错误地路由到 `kb-query`，导致查询 Milvus 向量库而非 Neo4j 图谱。当前 prompt 已修复，将这类问题正确路由到 `graphrag-query`。

### 8.3 "统计有多少道川菜"

- LLM 路由：`text2sql-query`（prompt 中明确要求识别"统计"关键词）
- 启发式：匹配"统计"和"多少" → `text2sql-query`（优先级最高）
- 最终路径：`create_research_plan` → Planner 选择 `text2sql_query`

### 8.4 "今天天气怎么样"

- LLM 路由：`general-query` 或 `additional-query`
- 启发式：无关键词匹配
- 如果路由到 `additional-query`：Guardrails 检查 → "end" → 礼貌拒绝
- 如果路由到 `general-query`：LLM 直接回复"不在能力范围内"

### 8.5 "我想做菜"

- LLM 路由：`additional-query`（缺少具体菜名）
- 启发式：无关键词匹配
- 最终路径：`get_additional_info` → Guardrails "proceed" → 引导用户说出具体菜名

### 8.6 "生成一张宫保鸡丁的图片"

- LLM 路由：`image-query`
- 启发式：无关键词匹配（启发式不含图片相关关键词）
- 最终路径：`create_image_query` → 检测到"生成"关键词 → 调用 CogView-4

### 8.7 用户上传了图片并说"这是什么菜"

- LLM 路由：可能是 `image-query` 或 `kb-query`
- 但 `route_query` 中优先检测 `image_path`（第 214-218 行）
- 最终路径：`create_image_query` → Vision API 识别 → LLM 回答

---

## 9. 如何调试路由错误

### 9.1 看日志关键字

路由相关日志在 `lg_builder.py` 中通过 `logger.info` 输出。关键日志：

```
# 路由开始
INFO - -----Analyze user query type-----
INFO - History messages: [...]

# 路由完成
INFO - Analyze user query type completed, result: {'type': '...', 'logic': '...'}

# Fallback 触发
WARN - Router LLM failed: ... Falling back to KB query.

# Guardrails 结果
INFO - -----Pass guardrails check-----
INFO - -----Fail to pass guardrails check-----
```

### 9.2 常见路由错误

**问题 1：LLM 返回了非法类型**

日志特征：
```
WARN - Router returned invalid type `xxx`; applying heuristic fallback.
```

原因：LLM 返回了 7 种合法类型之外的字符串。系统会尝试用启发式关键词匹配作为 fallback。如果启发式也没有匹配，最终默认到 `kb-query`。

**问题 2：LLM 调用异常**

日志特征：
```
WARN - Router LLM failed: <error message>. Falling back to KB query.
```

原因：网络超时、API key 无效、模型不可用等。直接使用预计算的启发式路由结果。

**问题 3：路由类型正确但结果不符合预期**

例如"红烧肉怎么做"被路由到 `kb-query` 而非 `graphrag-query`。

排查步骤：
1. 检查 `ROUTER_SYSTEM_PROMPT` 的内容（`lg_prompts.py` 第 7-66 行）
2. 确认 prompt 中对各类型的描述是否清晰
3. 查看 `logic` 字段了解 LLM 的分类理由

### 9.3 编程式调试

可以通过 Python 代码直接测试路由：

```python
from gustobot.application.agents.lg_builder import _heuristic_router, analyze_and_route_query
from gustobot.application.agents.lg_states import AgentState, InputState, Router
from langchain_core.messages import HumanMessage

# 测试启发式路由
result = _heuristic_router("统计有多少道菜")
print(result)  # Router(type="text2sql-query", ...)

result = _heuristic_router("红烧肉怎么做")
print(result)  # Router(type="graphrag-query", ...)

result = _heuristic_router("你好")
print(result)  # None（无关键词匹配）
```

---

## 10. 学习者自检清单

完成本阶段学习后，请确认你能回答以下问题：

- [ ] GustoBot 定义了哪 7 种路由类型？每种类型处理什么问题？
- [ ] `Router` Pydantic 模型有哪些字段？`type` 字段的默认值是什么？为什么选择这个默认值？
- [ ] `analyze_and_route_query` 函数的执行流程是什么？先做什么、后做什么？
- [ ] 什么是 `with_structured_output(Router)`？它如何保证 LLM 返回结构化数据？
- [ ] `_heuristic_router` 函数在什么情况下被使用？它匹配哪些关键词？text2sql 和 graphrag 的优先级关系是什么？
- [ ] `route_query` 函数为什么只有 6 个目标节点，而不是 7 个？哪两种类型合并了？
- [ ] 图片和文件路径为什么能覆盖 LLM 的路由结果？这个设计解决了什么问题？
- [ ] `get_additional_info` 节点中的 Guardrails 检查是什么？输出有几种？分别对应什么行为？
- [ ] 当 LLM 路由调用失败时，系统会怎样处理？最终 fallback 到哪种类型？
- [ ] 如果"红烧肉怎么做"被错误路由到 `kb-query`，你会怎么排查？查看哪个文件的哪个函数？

---

## 附录：关键代码位置索引

| 内容 | 文件 | 行号 |
|------|------|------|
| Router Pydantic 模型 | `lg_states.py` | 8-31 |
| RouteResult 数据类 | `lg_states.py` | 34-40 |
| AgentState（含 router 字段） | `lg_states.py` | 91-104 |
| analyze_and_route_query 函数 | `lg_builder.py` | 103-194 |
| _heuristic_router 函数 | `lg_builder.py` | 986-1022 |
| route_query 条件路由函数 | `lg_builder.py` | 197-236 |
| respond_to_general_query 节点 | `lg_builder.py` | 239-265 |
| get_additional_info 节点 | `lg_builder.py` | 268-379 |
| create_research_plan 节点 | `lg_builder.py` | 804-910 |
| create_image_query 节点 | `lg_builder.py` | 478-619 |
| create_file_query 节点 | `lg_builder.py` | 622-709 |
| create_kb_query 节点 | `lg_builder.py` | 711-801 |
| LangGraph 图构建与注册 | `lg_builder.py` | 958-973 |
| ROUTER_SYSTEM_PROMPT | `lg_prompts.py` | 7-66 |
| GUARDRAILS_SYSTEM_PROMPT | `lg_prompts.py` | 192-215 |
| GENERAL_QUERY_SYSTEM_PROMPT | `lg_prompts.py` | 69-116 |
| GET_ADDITIONAL_SYSTEM_PROMPT | `lg_prompts.py` | 120-138 |
