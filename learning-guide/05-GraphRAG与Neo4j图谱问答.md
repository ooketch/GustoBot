# 阶段 5：GraphRAG / Neo4j 图谱问答 ⭐

> **目标**：能讲清楚为什么菜谱适合图谱、图谱怎么被 Agent 调用、Cypher 怎么生成。

### 阅读顺序

1. [data/recipe.json](../data/recipe.json)（看几条数据）— 理解菜谱数据长什么样
2. [data/neo4j/](../data/neo4j/) — 图谱 Schema 和导入数据
3. [recipe_kg/](../gustobot/infrastructure/knowledge/recipe_kg/) — `Neo4jQAService`
4. [multi_tool.py](../gustobot/application/agents/kg_sub_graph/agentic_rag_agents/workflows/multi_agent/multi_tool.py) — `create_multi_tool_workflow()`
5. [edges.py](../gustobot/application/agents/kg_sub_graph/agentic_rag_agents/workflows/multi_agent/edges.py) — Send 并行分发 + 工具路由
6. [guardrails/node.py](../gustobot/application/agents/kg_sub_graph/agentic_rag_agents/components/guardrails/node.py) — 安全防护
7. [planner/node.py](../gustobot/application/agents/kg_sub_graph/agentic_rag_agents/components/planner/node.py) — 任务分解
8. [tool_selection/node.py](../gustobot/application/agents/kg_sub_graph/agentic_rag_agents/components/tool_selection/node.py) — 工具选择
9. [text2cypher/](../gustobot/application/agents/kg_sub_graph/agentic_rag_agents/components/text2cypher/) — Cypher 生成 → 校验 → 执行
10. [predefined_cypher/cypher_dict.py](../gustobot/application/agents/kg_sub_graph/agentic_rag_agents/components/predefined_cypher/cypher_dict.py) — 预定义查询模板

### 动手任务

手画 KG 子图的完整流程图，标出每个节点的输入输出状态：

```
START → guardrails → planner → [Send 并行] → tool_selection × N
  → cypher_query / predefined_cypher / customer_tools / text2sql_query
  → summarize → final_answer → END
```

### 要理解的问题

- Neo4j 比 SQL 或向量库强在哪里？
- 什么场景适合 Cypher，什么场景适合 RAG？
- LLM 生成 Cypher 有什么风险？
- 如何防止生成危险或错误查询？
- predefined Cypher 和 dynamic Cypher 的区别是什么？

---

## 5.1 为什么菜谱适合用图谱？

### 5.1.1 关系型数据库的局限

假设用户问："鸡蛋可以做哪些口味偏咸鲜的菜？" 用关系型数据库需要 JOIN 多张表（菜品表、食材表、口味表），SQL 冗长且难以表达"食材共现"、"口味相似"等关系推理。菜谱数据天然具有**多对多关系**：一道菜有多个食材、一个食材出现在多道菜里、菜品有口味/工艺/类型等多种属性。这些关系恰好是图数据库的优势所在。

### 5.1.2 图谱的关系推理场景

在 GustoBot 中，图谱能高效回答以下类问题：

| 场景 | 示例问题 | 图谱优势 |
|------|---------|---------|
| 食材反查 | "鸡蛋能做什么菜？" | `(Ingredient)-[:HAS_MAIN_INGREDIENT]-(Dish)` 一跳完成 |
| 多条件筛选 | "有哪些炒的咸鲜菜？" | 沿 `HAS_FLAVOR` + `USES_METHOD` 双路径过滤 |
| 相似菜品推荐 | "和红烧豆腐做法类似的菜？" | 共享口味/工艺节点的图遍历 |
| 营养功效查询 | "百合有什么功效？" | `(Ingredient)-[:HAS_HEALTH_BENEFIT]->(HealthBenefit)` |
| 统计分析 | "最常见的烹饪方法是什么？" | `GROUP BY` 节点聚合 |

---

## 5.2 菜谱数据结构分析

### 5.2.1 recipe.json 的字段结构

原始数据存储在 `data/recipe.json`，每条记录以菜品名为 key：

```json
{
  "香肠炒菜干": {
    "主食材": [["香肠", "2根"], ["菜干", "200g"]],
    "辅料": [["豆豉", "2匙"], ["蒜", "少许"], ["葱", "1颗"], ["酱油", "2匙"], ["蚝油", "1匙"], ["食用油", "适量"]],
    "耗时": "十分钟",
    "口味": "酱香",
    "工艺": "炒",
    "做法": "1:准备的食材。2:香肉肠切片。3:爆香蒜末、豆豉。...",
    "类型": "热菜"
  }
}
```

字段说明：

| 字段 | 含义 | 图谱映射 |
|------|------|---------|
| `主食材` | `[名称, 用量]` 二元组列表 | `Dish -[:HAS_MAIN_INGREDIENT {amount_text}]-> Ingredient` |
| `辅料` | `[名称, 用量]` 二元组列表 | `Dish -[:HAS_AUX_INGREDIENT {amount_text}]-> Ingredient` |
| `耗时` | 烹饪时长描述 | `Dish.cook_time` 属性 |
| `口味` | 可包含多种口味（顿号分隔） | `Dish -[:HAS_FLAVOR]-> Flavor` |
| `工艺` | 烹饪方法（炒、煮、炖等） | `Dish -[:USES_METHOD]-> CookingMethod` |
| `做法` | 编号步骤文本 | `Dish -[:HAS_STEP {order}]-> CookingStep` |
| `类型` | 菜品分类（热菜、凉菜等） | `Dish -[:BELONGS_TO_TYPE]-> DishType` |

### 5.2.2 excipients.json 辅料数据

`data/excipients.json` 存储食材的营养与功效信息：

```json
{
  "百合": {
    "营养价值": null,
    "食用功效": "1、润肺止咳：甘凉清润... 2、宁心安神：... 3、美容养颜：... 4、防癌抗癌：..."
  }
}
```

解析后映射为：

```
(Ingredient {name: "百合"})
  -[:HAS_NUTRITION_PROFILE]-> (NutritionProfile {description: "..."})
  -[:HAS_HEALTH_BENEFIT]->   (HealthBenefit {name: "润肺止咳"})
  -[:HAS_HEALTH_BENEFIT]->   (HealthBenefit {name: "宁心安神"})
```

### 5.2.3 图谱 Schema 总览

基于 `graph_importer_service.py` 中的 Cypher 导入语句，完整的图谱 Schema 如下：

```
节点类型（Labels）：
  Dish              -- 菜品，属性：name, cook_time, instructions
  Ingredient        -- 食材，属性：name
  Flavor            -- 口味，属性：name
  CookingMethod     -- 烹饪工艺，属性：name
  DishType          -- 菜品类型，属性：name
  CookingStep       -- 烹饪步骤，属性：dish_name, order, instruction
  NutritionProfile  -- 营养档案，属性：name, description
  HealthBenefit     -- 健康功效，属性：name

关系类型（Relationships）：
  Dish -[:HAS_MAIN_INGREDIENT {amount_text, role}]-> Ingredient
  Dish -[:HAS_AUX_INGREDIENT  {amount_text, role}]-> Ingredient
  Dish -[:HAS_FLAVOR]->    Flavor
  Dish -[:USES_METHOD]->   CookingMethod
  Dish -[:BELONGS_TO_TYPE]-> DishType
  Dish -[:HAS_STEP {order}]-> CookingStep
  Ingredient -[:HAS_NUTRITION_PROFILE]-> NutritionProfile
  Ingredient -[:HAS_HEALTH_BENEFIT]->    HealthBenefit
```

---

## 5.3 KG 子图完整流程

GustoBot 的图谱问答子图（KG Sub-Graph）基于 LangGraph 的 `StateGraph` 构建，整体流程如下：

```
用户问题
  |
  v
[Guardrails]  -- 安全护栏：判断问题是否在菜谱范围内
  |
  |--- "end" --> [Final Answer]（返回拒绝消息）
  |
  |--- "planner" --> [Planner]（任务分解）
                        |
                        v
                   [Map-Reduce: Send()]（并行分发子任务）
                        |
                        v
              +--> [Tool Selection] --+--> [Text2Cypher]
              |                       +--> [Predefined Cypher]
              |                       +--> [Customer Tools (GraphRAG)]
              |                       +--> [Text2SQL]
              v
         [Summarize]（聚合所有工具结果）
              |
              v
         [Final Answer]（输出最终回答）
```

对应代码位于 `multi_tool.py` 的 `create_multi_tool_workflow` 函数：

```python
# 1. 创建各节点
guardrails = create_guardrails_node(llm=llm, graph=graph, scope_description=scope_description)
planner = create_planner_node(llm=llm)
cypher_query = create_cypher_query_node()
predefined_cypher = create_predefined_cypher_node(graph=graph, predefined_cypher_dict=predefined_cypher_dict)
customer_tools = create_graphrag_query_node()
text2sql_query = create_text2sql_tool_node(graph)
tool_selection = create_tool_selection_node(llm=llm, tool_schemas=tool_schemas, ...)
summarize = create_summarization_node(llm=llm)
final_answer = create_final_answer_node()

# 2. 注册节点和边
main_graph_builder = StateGraph(OverallState, input=InputState, output=OutputState)
main_graph_builder.add_node(guardrails)
main_graph_builder.add_node(planner)
# ... 其他节点

# 3. 定义流转
main_graph_builder.add_edge(START, "guardrails")
main_graph_builder.add_conditional_edges("guardrails", guardrails_conditional_edge)
main_graph_builder.add_conditional_edges("planner", map_reduce_planner_to_tool_selection, ["tool_selection"])
main_graph_builder.add_edge("cypher_query", "summarize")
main_graph_builder.add_edge("summarize", "final_answer")
main_graph_builder.add_edge("final_answer", END)
```

---

## 5.4 Guardrails：安全护栏

### 5.4.1 作用

Guardrails 是工作流的第一道关卡，负责判断用户问题是否在菜谱知识图谱的范围内。它有**两层过滤**机制：

1. **启发式关键词匹配**：快速检查问题中是否包含菜谱领域关键词
2. **LLM 结构化判断**：用大模型做更精细的范围判定

### 5.4.2 关键词白名单

在 `guardrails/node.py` 中定义了一组启发式关键词：

```python
heuristics_keywords = [
    "菜", "菜谱", "食材", "烹饪", "做法", "步骤", "口味",
    "炒", "煮", "炖", "蒸", "统计", "多少", "用量", "营养", "功效",
]
```

如果用户问题包含这些关键词，直接跳过 LLM 判断，路由到 Planner。这样既节省了 Token，又加快了响应速度。

### 5.4.3 LLM 结构化输出

当关键词不命中时，调用 LLM 并要求输出结构化的 `GuardrailsOutput`：

```python
guardrails_chain = guardrails_prompt | llm.with_structured_output(GuardrailsOutput)
```

LLM 返回 `decision` 字段：
- `"planner"`：问题在范围内，继续处理
- `"end"`：问题超出范围，直接跳转 Final Answer 返回拒绝消息

### 5.4.4 兜底策略

即使 LLM 判断为 `"end"`，如果问题中包含领域关键词，系统会强制覆盖为 `"planner"`。这是一种宁可多处理、也不误拒的保守策略。

---

## 5.5 Planner：任务分解

### 5.5.1 为什么需要任务分解？

用户的一个问题可能隐含多个子查询。例如：

> "香肠炒菜干的做法是什么？用到了哪些食材？"

这需要同时查询做法（`d.instructions`）和食材列表（`HAS_MAIN_INGREDIENT` + `HAS_AUX_INGREDIENT`）。Planner 的职责就是把复合问题拆成独立的、不相互依赖的子任务。

### 5.5.2 分解规则

Planner 的 Prompt 中明确规定了四条规则：

```
* 确保任务不会返回重复或相似的信息。
* 确保任务不依赖于从其他任务收集的信息！
* 相互依赖的任务应该合并为单个问题。
* 返回相同信息的任务应该合并为单个问题。
```

这些规则保证子任务之间是**并行独立**的，不存在执行顺序依赖。

### 5.5.3 输出结构

Planner 使用 `PlannerOutput` 模型输出结构化结果：

```python
class PlannerOutput(BaseModel):
    tasks: list[Task]  # 子任务列表

class Task(BaseModel):
    question: str        # 子任务的问题
    parent_task: str     # 原始父问题
    requires_visualization: bool = False
```

如果 LLM 认为问题无需拆分，会返回一个只包含原始问题的单元素列表。在代码中还有一个 fallback：

```python
tasks = planner_output.tasks or [
    Task(question=state.get("question", ""), parent_task=state.get("question", ""))
]
```

---

## 5.6 Map-Reduce 并行分发：Send() 的使用

### 5.6.1 Send() 是什么？

`Send()` 是 LangGraph 提供的**动态并行分发机制**。当 Planner 输出 N 个子任务时，`map_reduce_planner_to_tool_selection` 边函数会为每个子任务创建一个 `Send()`：

```python
def map_reduce_planner_to_tool_selection(state: OverallState) -> List[Send]:
    return [
        Send(
            "tool_selection",
            {
                "question": task.question,
                "parent_task": task.parent_task,
                "context": {"route_type": state.get("route_type")},
            },
        )
        for task in state.get("tasks", list())
    ]
```

### 5.6.2 并行执行原理

`Send("tool_selection", payload)` 的含义是：**为每个 payload 创建一个独立的 `tool_selection` 节点实例**，所有实例并行执行。每个实例有自己的状态副本，互不干扰。

例如，Planner 将问题拆成 2 个子任务，系统会同时启动 2 个 `tool_selection` 节点，各自选择工具并执行。

### 5.6.3 结果合并：add reducer

并行结果如何合并回主状态？关键在于 `state.py` 中 `tasks` 和 `cyphers` 字段的 reducer 定义：

```python
class OverallState(TypedDict, total=False):
    tasks: Annotated[List[Task], add]
    cyphers: Annotated[List[CypherOutputState], add]
    steps: Annotated[List[str], add]
```

`Annotated[List[...], add]` 表示当多个并行节点各自返回一个列表时，LangGraph 会自动用 `add` 操作（即列表拼接）将它们合并。例如：

- 节点 A 返回 `cyphers: [{task: "做法", ...}]`
- 节点 B 返回 `cyphers: [{task: "食材", ...}]`
- 合并后：`cyphers: [{task: "做法", ...}, {task: "食材", ...}]`

这是 Map-Reduce 模式中 **Reduce** 阶段的核心机制。

---

## 5.7 Tool Selection：工具选择

### 5.7.1 可用工具列表

GustoBot 的图谱子图支持 4 种工具：

| 工具名 | 说明 | 适用场景 |
|--------|------|---------|
| `cypher_query` | Text2Cypher，LLM 动态生成 Cypher | 复杂、非模板化的图查询 |
| `predefined_cypher` | 预定义 Cypher 模板 | 高频、标准化的查询 |
| `customer_tools` | GraphRAG / LightRAG 查询 | 描述类、知识类问题 |
| `text2sql_query` | Text2SQL | 结构化统计分析 |

### 5.7.2 选择策略：启发式 + LLM

工具选择采用**两阶段策略**：

**第一阶段：启发式快速路径**

```python
# 描述类关键词 -> 直接走 GraphRAG
DESCRIPTIVE_KEYWORDS = ["口味", "特色", "风味", "营养", "功效", "健康", "介绍", ...]
if any(keyword in question_text for keyword in DESCRIPTIVE_KEYWORDS):
    return _make_command("customer_tools", {...})

# SQL 类关键词 + route_type 匹配 -> 直接走 Text2SQL
if _looks_like_sql_question(question_text) and route_type == "text2sql-query":
    return _make_command("text2sql_query", {...})
```

**第二阶段：LLM bind_tools 动态选择**

当启发式规则不命中时，使用 LLM 的工具调用能力：

```python
tool_selection_chain = (
    tool_selection_prompt
    | llm.bind_tools(tools=tool_schemas)
    | PydanticToolsParser(tools=tool_schemas, first_tool_only=True)
)
```

`bind_tools` 将工具的 JSON Schema 注入到 LLM 的 system prompt 中，LLM 根据用户问题选择最合适的工具并输出参数。`PydanticToolsParser` 解析 LLM 的工具调用结果为 Pydantic 模型。

如果 LLM 未返回有效的工具调用，且 `default_to_text2cypher=True`，则默认回退到 `cypher_query`。

### 5.7.3 路由输出

工具选择节点返回 `Command` 对象，通过 `Send()` 将任务分发到对应的执行节点：

```python
return Command(goto=Send("cypher_query", {"task": question_text, ...}))
# 或
return Command(goto=Send("predefined_cypher", {"task": question_text, ...}))
```

---

## 5.8 Text2Cypher 完整流程

Text2Cypher 是图谱问答的核心能力——将自然语言问题翻译成 Cypher 查询语句。它本身是一个**子图**，包含 4 个节点：

```
[Generate] --> [Validate] --> [Execute]
                 |    ^
                 v    |
              [Correct]（错误重试循环）
```

### 5.8.1 Generation：Cypher 生成

在 `text2cypher/generation/node.py` 中：

```python
text2cypher_chain = generation_prompt | llm | StrOutputParser()

async def generate_cypher(state: CypherInputState) -> Dict[str, Any]:
    task = state.get("task", "")
    examples = cypher_example_retriever.get_examples(query=task, k=3)
    generated_cypher = await text2cypher_chain.ainvoke({
        "question": state.get("task", ""),
        "fewshot_examples": examples,
        "schema": graph.schema,
    })
    return {"statement": generated_cypher, "steps": ["generate_cypher"]}
```

关键要素：
- **schema**：图谱的完整 Schema（节点类型、关系类型、属性），让 LLM 知道图的结构
- **fewshot_examples**：从示例检索器中检索 k=3 个相似的 Cypher 示例，提供 few-shot 引导
- **question**：用户的自然语言问题

### 5.8.2 Validation：Cypher 校验

在 `text2cypher/validation/node.py` 中，校验分为 5 层：

```python
# 1. 语法检查：括号匹配、关键字使用等
syntax_error = validate_cypher_query_syntax(graph=graph, cypher_statement=...)

# 2. 写操作拦截：防止 LLM 生成 CREATE/DELETE/SET 等危险语句
write_errors = validate_no_writes_in_cypher_query(state.get("statement", ""))

# 3. 关系方向修正：自动修复错误的关系方向
corrected_cypher = correct_cypher_query_relationship_direction(graph=graph, ...)

# 4. LLM 语义验证：检查 Cypher 是否符合用户意图
if llm_validation:
    llm_errors = await validate_cypher_query_with_llm(...)

# 5. Schema 一致性检查：确保节点/关系/属性都存在
if not llm_validation:
    cypher_errors = validate_cypher_query_with_schema(graph=graph, ...)
```

校验后的路由逻辑：

```python
if (errors or mapping_errors) and GENERATION_ATTEMPT < max_attempts:
    next_action = "correct_cypher"    # 有错误且未超限 -> 去修正
elif GENERATION_ATTEMPT < max_attempts:
    next_action = "execute_cypher"    # 无错误 -> 去执行
else:
    next_action = "__end__"           # 超限 -> 终止
```

### 5.8.3 Correction：错误修正

当校验发现错误时，进入修正节点：

```python
corrected_cypher = await correct_cypher_chain.ainvoke({
    "question": state.get("task"),
    "errors": state.get("errors"),      # 校验发现的具体错误
    "cypher": state.get("statement"),    # 原始 Cypher
    "schema": graph.schema,              # 图谱 Schema
})
```

修正后回到 Validate 节点重新校验，形成一个**生成-校验-修正**的循环，最多执行 `max_attempts`（默认 3 次）。

### 5.8.4 Execution：Cypher 执行

通过校验后，调用 Neo4j 驱动执行 Cypher：

```python
records = graph.query(state.get("statement", ""))
```

执行结果封装为 `CypherOutputState`，包含任务描述、Cypher 语句、查询结果和执行步骤：

```python
CypherOutputState(
    task=state.get("task", []),
    statement=state.get("statement", ""),
    parameters=None,
    errors=state.get("errors", list()),
    records=records if records else NO_CYPHER_RESULTS,
    steps=steps,
)
```

---

## 5.9 Predefined Cypher：高频查询模板

### 5.9.1 为什么需要预定义模板？

Text2Cypher 灵活但不稳定——LLM 可能生成语法错误或语义偏差的 Cypher。对于高频、标准化的查询（如"某道菜的做法"、"某食材的功效"），使用预定义模板更可靠。

### 5.9.2 模板分类

在 `predefined_cypher/cypher_dict.py` 中定义了 8 大类、共 28 个查询模板：

| 分类 | 模板示例 | 用途 |
|------|---------|------|
| 菜品属性查询 | `dish_instructions`, `dish_cook_time`, `dish_flavor` | 查菜品基本信息 |
| 属性约束查询 | `dishes_by_flavor`, `dishes_by_method`, `dishes_by_type` | 按条件筛选菜品 |
| 关系约束查询 | `dishes_by_main_ingredient`, `ingredients_of_dish` | 食材与菜品关系 |
| 用量查询 | `ingredient_amount_in_dish`, `main_ingredient_amount` | 查询具体用量 |
| 烹饪步骤 | `cooking_steps`, `step_by_order` | 查询做法步骤 |
| 营养功效 | `ingredient_nutrition`, `ingredient_health_benefits` | 食材营养信息 |
| 统计分析 | `most_used_cooking_methods`, `most_popular_flavors` | 聚合统计 |
| 综合推荐 | `similar_dishes`, `dishes_with_ingredients` | 相似推荐 |

### 5.9.3 模板参数化

模板使用 `$variable` 占位符，运行时注入参数：

```cypher
// dish_instructions 模板
MATCH (d:Dish {name: $dish_name})
RETURN d.name AS 菜名, d.instructions AS 做法

// 执行时注入参数：{"dish_name": "香肠炒菜干"}
```

### 5.9.4 模板匹配机制

在 `predefined_cypher/node.py` 中，模板匹配采用**向量检索 + LLM 参数提取**：

```python
matcher = create_vector_query_matcher(predefined_cypher_dict, QUERY_DESCRIPTIONS)

# 1. 向量匹配：根据用户问题的语义相似度找到最匹配的模板
matches = matcher.match_query(question, top_k=1)

# 2. LLM 参数提取：从问题中提取模板所需的参数
parameters = matcher.extract_parameters(question, query_name, llm=chat_llm)
```

每个模板都有对应的自然语言描述（存储在 `descriptions.py`），用于向量匹配。例如：

```python
"dish_instructions": "查询指定菜品的完整做法文本, 适用于用户想了解整道菜的烹饪步骤和细节。"
"dishes_by_flavor": "根据口味标签筛选菜品, 适用于用户想找同一口味的多道菜。"
```

---

## 5.10 Summarize：多工具结果聚合

### 5.10.1 作用

当多个子任务并行执行完毕后，各自的 Cypher 查询结果需要汇总成一个连贯的回答。Summarize 节点承担了这个职责。

### 5.10.2 聚合逻辑

在 `summarize/node.py` 中，结果被分为三类：

```python
narrative_sections: List[str] = []   # 叙述性结果（如 GraphRAG 返回的文本）
metric_sections: List[str] = []      # 结构化数据（如 Cypher 查询结果）
error_sections: List[str] = []       # 错误信息
```

对每个 `cyphers` 中的结果：
- 如果有 `errors` -> 放入 `error_sections`
- 如果 `records` 是 dict 类型，提取 `result`/`answer`/`rows` 字段
- 如果 `records` 是 list 类型，进行**智能格式化**

### 5.10.3 智能格式化

`_format_rows` 函数会根据数据特征自动选择格式：

```python
# 烹饪步骤：只显示序号和说明
if is_cooking_steps:
    lines.append(f"{step_num}. {step_desc}")

# 食材列表：主料用 ★ 标记
elif is_ingredients:
    marker = "★ " if "MAIN" in relation else "  "
    lines.append(f"{marker}{ingredient}：{amount}")

# 其他数据：显示所有字段
else:
    row_desc = ", ".join(f"{key}：{value}" for key, value in row.items())
```

### 5.10.4 最终输出格式

聚合结果按以下结构组织：

```
### 川菜概览
（叙述性文本）

### 数据统计
（结构化数据条目）

### 查询提示
- （错误信息）
```

---

## 5.11 Final Answer：最终答案

### 5.11.1 职责

Final Answer 节点非常简洁，主要做两件事：

1. 从 state 中提取 `summary` 作为最终回答
2. 将本轮问答的上下文写入 `history`，供后续对话参考

```python
async def final_answer(state: OverallState) -> dict[str, Any]:
    answer = state.get("summary", " ")
    history_record = {
        "question": state.get("question", ""),
        "answer": answer,
        "cyphers": [...],
    }
    return {"answer": answer, "steps": ["final_answer"], "history": [history_record]}
```

### 5.11.2 历史记录管理

`history` 字段使用自定义的 `update_history` reducer，保留最近 5 轮对话：

```python
def update_history(history, new):
    SIZE = 5
    history.extend(new)
    return history[-SIZE:]
```

这使得后续对话可以引用之前的查询结果（如"刚才那道菜的食材用量是多少？"）。

---

## 5.12 State 管理：add reducer 详解

### 5.12.1 三种 State 类型

| State | 用途 | 关键字段 |
|-------|------|---------|
| `InputState` | 子图输入 | `question`, `history`, `route_type` |
| `OverallState` | 子图内部全局状态 | `tasks`, `cyphers`, `next_action`, `summary`, `steps` |
| `OutputState` | 子图输出 | `answer`, `steps`, `cyphers`, `history` |

### 5.12.2 add reducer 的作用

在并行执行场景中，多个节点可能同时向同一个状态字段写入数据。`Annotated[List[T], add]` 告诉 LangGraph：当多个节点各自产出一个列表时，用**列表拼接**（而非覆盖）来合并。

```python
class OverallState(TypedDict, total=False):
    tasks: Annotated[List[Task], add]           # Planner 产出的子任务
    cyphers: Annotated[List[CypherOutputState], add]  # 各工具执行结果
    steps: Annotated[List[str], add]            # 执行步骤追踪
```

### 5.12.3 合并示例

假设用户问"鸡蛋可以做哪些菜？鸡蛋的营养是什么？"，Planner 拆成 2 个子任务并行执行：

```
子任务 A (Text2Cypher) -> cyphers: [{task: "鸡蛋做哪些菜", records: [...]}]
子任务 B (Predefined)  -> cyphers: [{task: "鸡蛋的营养", records: [...]}]

合并后 OverallState.cyphers = [
    {task: "鸡蛋做哪些菜", records: [...]},
    {task: "鸡蛋的营养", records: [...]}
]
```

---

## 5.13 底层基础设施：Neo4jQAService

### 5.13.1 架构分层

GustoBot 的图谱服务分为两层：

```
Neo4jQAService（对外暴露的统一服务）
  |-- Neo4jDatabase（底层驱动封装）
  |-- GraphCache（图谱缓存）
  |-- RecipeGraphImporter（数据导入）
  |-- Neo4jQAPipeline（简单问答管线）
```

### 5.13.2 数据导入流程

`RecipeGraphImporter` 负责将 JSON 数据导入 Neo4j：

1. `load_recipe_records()` 解析 `recipe.json`，提取菜品、食材、步骤等
2. `_create_recipe_nodes()` 创建 Dish 节点及 Flavor/CookingMethod/DishType 关系
3. `_create_relationships()` 创建食材关系（主料/辅料）和步骤关系
4. `_attach_ingredient_metadata()` 关联营养档案和功效信息

导入使用 `UNWIND + MERGE` 的批量写入模式，每批 200 条，避免单条事务开销。

### 5.13.3 简单 QA 管线

除了 LangGraph 子图，还有一套轻量级的 `Neo4jQAPipeline`：

```
QuestionClassifier（分类：recipe_property / property_constraint / relationship_constraint / relationship_query）
  -> QuestionParser（解析：提取菜名、食材名等参数）
  -> AnswerSearcher（执行：根据分类和参数生成并执行 Cypher）
```

这条管线适用于简单的单意图查询，不需要 LangGraph 的复杂编排。

---

## 5.14 学习者自检清单

### 基础概念

- [ ] 能画出 GustoBot 图谱的完整 Schema（节点、关系、属性）
- [ ] 能解释 `recipe.json` 中每个字段如何映射到图谱结构
- [ ] 能说出至少 3 个菜谱适合图谱而非关系型数据库的场景

### 流程理解

- [ ] 能描述 KG 子图的完整流转路径（Guardrails -> Planner -> Tool Selection -> 执行 -> Summarize -> Final Answer）
- [ ] 能解释 Guardrails 的两层过滤机制（关键词 + LLM）
- [ ] 能说明 Planner 任务分解的四条规则

### 并行机制

- [ ] 能解释 `Send()` 如何实现并行分发
- [ ] 能说明 `Annotated[List[T], add]` reducer 如何合并并行结果
- [ ] 能画出一个具体的 Map-Reduce 执行示例（2 个子任务并行）

### 工具选择

- [ ] 能说出 4 种可用工具及其适用场景
- [ ] 能解释启发式快速路径和 LLM bind_tools 的协作关系

### Text2Cypher

- [ ] 能描述 Text2Cypher 子图的 4 个节点及其职责
- [ ] 能解释 Cypher 校验的 5 层检查机制
- [ ] 能说明错误重试循环的工作原理（生成 -> 校验 -> 修正 -> 重新校验）

### 预定义模板

- [ ] 能说出预定义 Cypher 模板的 8 大分类
- [ ] 能解释模板参数化的实现方式（`$variable` 占位符）
- [ ] 能说明向量匹配 + LLM 参数提取的模板匹配机制

### 结果聚合

- [ ] 能解释 Summarize 节点如何区分叙述性/结构化/错误三类结果
- [ ] 能说明 `_format_rows` 的智能格式化逻辑（烹饪步骤 vs 食材列表 vs 通用数据）

### 代码实践

- [ ] 能在 `predefined_cypher_dict.py` 中添加一个新的查询模板
- [ ] 能在 `descriptions.py` 中为新模板添加语义描述
- [ ] 能修改 Guardrails 的关键词白名单
- [ ] 能调整 Planner 的 Prompt 以改变任务分解策略
