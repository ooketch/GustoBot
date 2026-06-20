# 阶段 6：Text2SQL

> **目标**：能讲清楚自然语言怎么变成 SQL，以及怎么保证安全。

### 阅读顺序

1. [docs/TEXT2SQL_IMPLEMENTATION.md](../docs/TEXT2SQL_IMPLEMENTATION.md) — Text2SQL 设计文档
2. [text2sql/workflow.py](../gustobot/application/agents/text2sql/workflow.py) — 独立工作流（含重试逻辑）
3. [text2sql/](../gustobot/application/agents/text2sql/) — 各子模块

### 动手任务

画出 Text2SQL 的完整流水线：

```
用户问题
  → Schema Retrieval（获取数据库表结构）
  → Query Analysis（分析问题意图）
  → SQL Generation（LLM 生成 SQL）
  → SQL Validation（只允许 SELECT/WITH）
  → SQL Execution（执行查询）
  → 格式化 → 自然语言回答
```

### 要理解的问题

- Text2SQL 最大风险是什么？
- 如何防 SQL 注入或破坏性 SQL？
- LLM 不知道数据库 schema 怎么办？
- 如果生成 SQL 错了，系统怎么修正？

---

## 1. 设计动机：为什么需要自然语言转 SQL

在 GustoBot 的菜谱管理系统中，数据库里存着大量结构化数据——菜谱、食材、营养成分、烹饪步骤等。普通用户想查"哪些川菜的热量低于 500 千卡"时，他们不会写 SQL。Text2SQL 的目标就是：

**用户用自然语言提问，系统自动将其转化为可执行的 SQL 查询，返回结果并可视化。**

GustoBot 的 Text2SQL 系统参考了 ChatDB 的多智能体架构，但用 LangGraph StateGraph 重新实现，将原来的多个 AutoGen Agent 改造为 LangGraph 节点，形成一条清晰的流水线。

---

## 2. 完整流水线概览

整条流水线由 7 个节点串联而成，每个节点都是一个纯函数，接收 state 字典、返回 state 字典的更新：

```
用户提问
   |
   v
[Step 1] retrieve_schema        -- 从 MySQL INFORMATION_SCHEMA 读取表结构
   |
   v
[Step 2] analyze_query           -- LLM 结构化分析查询意图
   |
   v
[Step 3] generate_sql            -- LLM 生成 SQL 语句
   |
   v
[Step 4] validate_sql            -- 语法检查 + 安全检查
   |
   |  (条件边)
   |-- 验证通过 --> [Step 5] execute_sql
   |-- 验证失败且未超限 --> 回到 Step 3 重试
   |-- 验证失败且超限 --> 直接跳到 Step 7
   v
[Step 5] execute_sql             -- 执行只读查询，最多返回 1000 行
   |
   v
[Step 6] visualization_node      -- LLM 推荐图表类型
   |
   v
[Step 7] format_answer_node      -- 组装 Markdown 最终回答
   |
   v
END
```

### 工作流组装代码

核心入口在 `workflow.py` 中的 `create_text2sql_workflow` 函数：

```python
# 文件：gustobot/application/agents/text2sql/workflow.py

def create_text2sql_workflow(
    llm: BaseChatModel,
    neo4j_graph: Neo4jGraph,
    db_type: str = "MySQL",
    connection_string: str | None = None,
    max_retries: int = 3,
) -> StateGraph:
    # 创建各节点
    retrieve_schema = create_schema_retrieval_node(neo4j_graph)
    analyze_query = create_query_analysis_node(llm)
    generate_sql = create_sql_generation_node(llm, db_type)
    validate_sql = create_sql_validation_node(db_type)
    execute_sql = create_sql_execution_node(connection_string)
    recommend_viz = create_visualization_node(llm)
    format_answer = create_answer_formatter_node()

    # 组装 StateGraph
    builder = StateGraph(Text2SQLState, input=Text2SQLInputState, output=Text2SQLOutputState)

    builder.add_node("retrieve_schema", retrieve_schema)
    builder.add_node("analyze_query", analyze_query)
    builder.add_node("generate_sql", generate_sql)
    builder.add_node("validate_sql", validate_sql)
    builder.add_node("execute_sql", execute_sql)
    builder.add_node("visualization_node", recommend_viz)
    builder.add_node("format_answer_node", format_answer)

    # 线性边
    builder.add_edge(START, "retrieve_schema")
    builder.add_edge("retrieve_schema", "analyze_query")
    builder.add_edge("analyze_query", "generate_sql")
    builder.add_edge("generate_sql", "validate_sql")

    # 条件边：验证后决定执行、重试还是终止
    builder.add_conditional_edges(
        "validate_sql",
        _should_execute_or_retry,
        {"execute": "execute_sql", "retry": "generate_sql", "end": "format_answer_node"},
    )

    builder.add_edge("execute_sql", "visualization_node")
    builder.add_edge("visualization_node", "format_answer_node")
    builder.add_edge("format_answer_node", END)

    return builder.compile()
```

### 状态定义

所有节点共享同一个 `Text2SQLState`，它是一个 TypedDict，定义了数据在整个流水线中如何流转：

```python
# 文件：gustobot/application/agents/text2sql/state.py

class Text2SQLState(TypedDict, total=False):
    # 输入
    question: str                    # 用户的自然语言问题
    connection_id: Optional[int]     # 数据库连接 ID
    db_type: Optional[str]           # 数据库类型，默认 MySQL

    # Schema 检索产出
    schema_context: Dict[str, Any]   # 表结构信息
    value_mappings: Dict[str, Dict[str, str]]  # 值映射
    mappings_str: str                # 值映射的文本表示

    # 查询分析产出
    analysis: Optional[Dict[str, Any]]   # 结构化分析结果
    analysis_text: Optional[str]         # Markdown 格式的分析摘要

    # SQL 生成与验证
    sql_statement: str               # 生成的 SQL 语句
    is_valid: bool                   # 验证是否通过
    validation_errors: Annotated[List[str], operator.add]  # 错误列表（累加）

    # 执行阶段
    execution_results: Optional[List[Dict[str, Any]]]  # 查询结果
    execution_error: Optional[str]   # 执行错误信息

    # 可视化
    visualization: Optional[Dict[str, Any]]  # 图表推荐

    # 工作流控制
    steps: Annotated[List[str], operator.add]  # 步骤记录（累加）
    retry_count: int                 # 当前重试次数
    max_retries: int                 # 最大重试次数

    # 最终输出
    answer: str                      # 最终 Markdown 回答
    visualization_config: Optional[Dict[str, Any]]  # 前端图表配置
```

注意 `validation_errors` 和 `steps` 使用了 `Annotated[List[str], operator.add]`，这意味着多个节点对这两个字段的写入会自动合并（累加），而不是覆盖。

---

## 3. 每个步骤的详细实现

### Step 1：Schema Retrieval（表结构检索）

**目标**：从 MySQL 的 `INFORMATION_SCHEMA` 中读取所有表的结构信息，包括表名、列名、数据类型、主键/外键标记，以及表间关系。

**实现文件**：`components/text2sql/schema_retrieval/node.py`

**核心逻辑**：

1. 查询 `INFORMATION_SCHEMA.TABLES` 获取所有表名和表注释。
2. 对每张表，查询 `INFORMATION_SCHEMA.COLUMNS` 获取列的详细信息。
3. 查询 `INFORMATION_SCHEMA.KEY_COLUMN_USAGE` 获取外键关系。
4. 用领域知识库（`domain_knowledge.py`）中的 `TABLE_DESCRIPTIONS` 和 `COLUMN_DESCRIPTIONS` 补充缺失的中文描述。
5. 根据用户问题提取关键词，对表打分排序，只返回最相关的最多 6 张表。

**关键词提取与打分**：

```python
_STOP_WORDS = {"the", "and", "for", "查询", "一下", "所有", "哪些", "什么", "数据", ...}

def _extract_keywords(question: str) -> List[str]:
    tokens = re.findall(r"[a-zA-Z0-9_]+", question.lower())
    return [t for t in tokens if t and t not in _STOP_WORDS]

def _score_table(table: Dict[str, Any], keywords: Iterable[str]) -> float:
    score = 0.0
    name = table.get("table_name", "").lower()
    description = table.get("description", "").lower()
    for kw in keywords:
        if kw in name:      score += 2.0   # 表名匹配权重高
        if kw in description: score += 1.0  # 描述匹配权重低
        # 列名和列描述也参与打分...
    return score
```

这样当用户问"有哪些川菜"时，`recipes` 和 `cuisines` 表会因为关键词匹配得到较高分数，被优先选入 Schema 上下文。

**输出**：将 `schema_context`（包含 tables 和 relationships）、`value_mappings`、`mappings_str` 写入 state。

### Step 2：Query Analysis（查询意图分析）

**目标**：用 LLM 对用户问题做结构化分析，提取出意图、需要的表和列、连接条件、筛选条件、聚合需求等。

**实现文件**：`components/text2sql/query_analysis/node.py` + `query_analysis/prompts.py`

**Pydantic 模型**（`models.py`）：

```python
class SQLAnalysis(BaseModel):
    query_intent: str                          # 查询意图描述
    required_tables: List[str]                 # 需要的表名列表
    required_columns: List[str]                # 需要的字段列表
    join_conditions: Optional[str] = None      # 表连接条件
    filter_conditions: Optional[str] = None    # 筛选条件
    aggregation: Optional[str] = None          # 聚合操作
    order_by: Optional[str] = None             # 排序要求
    notes: Optional[str] = None                # 额外说明
```

**提示词策略**：

系统提示词要求 LLM 扮演"资深数据库查询分析专家"，输出严格符合上述 Pydantic 模型的 JSON。提示词中注入了 `DOMAIN_SUMMARY`，告诉 LLM 数据库的真实背景——菜谱管理场景、各表之间的关系、字段取值约束等。

```python
analysis_chain = prompt | llm.with_structured_output(SQLAnalysis)
analysis: SQLAnalysis = await analysis_chain.ainvoke(inputs)
```

使用 `with_structured_output(SQLAnalysis)` 让 LLM 的输出直接被解析为 Pydantic 对象，确保结构化。

**输出**：`analysis`（字典形式）、`analysis_text`（Markdown 渲染的分析报告）。

分析报告的渲染由 `utils.py` 中的 `render_analysis_markdown` 函数完成，生成类似以下格式的 Markdown：

```markdown
## SQL 命令分析报告

### 1. 查询意图
查找热量低于 500 千卡的川菜菜谱

### 2. 涉及的表
- recipes
- cuisines

### 3. 关键字段
- recipes.name
- recipes.total_calories
- cuisines.name

### 4. 连接关系
recipes.cuisine_id 关联 cuisines.id

### 5. 筛选条件
cuisines.name = '川菜' AND recipes.total_calories < 500
```

### Step 3：SQL Generation（SQL 生成）

**目标**：结合 Schema、查询分析和用户问题，让 LLM 生成一条可执行的 SQL 语句。

**实现文件**：`components/text2sql/sql_generation/node.py` + `sql_generation/prompts.py`

**关键设计**：

- **低温度**：`llm.with_config(temperature=0.1)`，确保输出稳定、确定性高。
- **单语句约束**：提示词明确要求"输出必须是单条 SQL 语句，不能包含额外解释或多条语句"。
- **只读约束**：提示词要求"默认只生成只读查询（SELECT/CTE）"。
- **领域知识注入**：提示词中包含 `DOMAIN_SUMMARY`，避免 LLM 凭空编造表名或字段。

**提示词结构**：

```
System: 你是一名资深 SQL 开发工程师...（约束条件 + 领域背景）

Human:
## 数据库 Schema
（由 format_schema_as_text 生成的 CREATE TABLE 语句 + 关系描述）

## 值映射
（当前简化实现为空）

## 查询分析摘要
（Step 2 产出的 Markdown 分析报告）

## 用户问题
（原始自然语言问题）

请输出最终 SQL 语句。
```

**Schema 格式化**（`format_schema_as_text`）：将 Schema 上下文转成类似 SQL DDL 的文本格式，让 LLM 更容易理解表结构：

```sql
-- Table: recipes
-- Description: 菜谱主表，记录菜谱的基础信息...
CREATE TABLE recipes (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    total_time INT,
    total_calories DECIMAL,
    cuisine_id INT FOREIGN KEY,
    ...
);

-- Table Relationships
-- recipes.cuisine_id -> cuisines.id (many-to-one)
```

**输出清理**：`_clean_sql_statement` 函数移除 LLM 输出中可能包含的 Markdown 代码围栏（` ```sql ... ``` `），只保留纯 SQL 文本。

### Step 4：SQL Validation（SQL 验证）

**目标**：在执行前对生成的 SQL 做语法检查和安全检查。

**实现文件**：`components/text2sql/sql_validation/node.py` + `sql_validation/validators.py`

**两层检查**：

#### 语法检查（`validate_sql_syntax`）

```python
def validate_sql_syntax(sql: str, db_type: str = "MySQL") -> Tuple[bool, List[str]]:
    errors = []

    # 1. 必须以 SELECT 或 WITH 开头
    if not (leading_sql.startswith("SELECT") or leading_sql.startswith("WITH")):
        errors.append("仅支持以 SELECT 或 WITH 开头的只读查询")

    # 2. 括号匹配检查
    if sql.count("(") != sql.count(")"):
        errors.append("括号不匹配")

    # 3. 引号匹配检查
    if sql.count("'") % 2 != 0:
        errors.append("单引号不匹配")
    if sql.count('"') % 2 != 0:
        errors.append("双引号不匹配")

    return len(errors) == 0, errors
```

#### 安全检查（`validate_sql_security`）

```python
def validate_sql_security(sql: str) -> Tuple[bool, List[str]]:
    warnings = []
    sql_upper = sql.upper()

    dangerous_keywords = [
        "DROP TABLE", "DROP DATABASE", "TRUNCATE", "DELETE FROM",
        "INSERT INTO", "UPDATE ", "MERGE ", "ALTER TABLE",
        "CREATE TABLE", "CREATE DATABASE", "GRANT", "REVOKE",
        "CALL ", "EXEC ",
    ]

    for keyword in dangerous_keywords:
        if keyword in sql_upper:
            warnings.append(f"检测到危险操作: {keyword}")

    return len(warnings) == 0, warnings
```

**验证结果处理**：如果验证失败，节点会将 `is_valid` 设为 `False`，累加错误到 `validation_errors`，并将 `retry_count` 加 1。

### Step 5：SQL Execution（SQL 执行）

**目标**：将通过验证的 SQL 发送到 MySQL 执行，获取查询结果。

**实现文件**：`components/text2sql/sql_execution/node.py`

**安全措施——执行前再次检查只读**：

```python
def _is_read_only_query(sql: str) -> bool:
    statement = sql.strip()
    # 检查是否包含多条语句（;分隔）
    if ";" in statement.rstrip(";"):
        return False

    upper = statement.upper()
    readonly_keywords = ['SELECT ', 'WITH ', 'EXPLAIN ', 'SHOW ']
    dangerous_keywords = ['INSERT ', 'UPDATE ', 'DELETE ', 'DROP ', ...]

    starts_with_readonly = any(upper.startswith(kw) for kw in readonly_keywords)
    contains_dangerous = any(kw in upper for kw in dangerous_keywords)

    return starts_with_readonly and not contains_dangerous
```

这是第二道防线——即使验证节点遗漏了什么，执行节点也会再次检查。

**执行方式**：

- 使用 SQLAlchemy 创建引擎，在线程池中执行同步查询（`asyncio.to_thread`）。
- 使用 `stream_results=True` 流式获取结果，用 `fetchmany(max_rows)` 限制返回行数（默认 1000 行）。
- 结果转为 `List[Dict[str, Any]]`，每行是一个字典，key 是列名。

```python
def _run_query_sync(connection_string, sql, max_rows):
    engine = create_engine(connection_string, future=True)
    with engine.connect() as connection:
        result = connection.execution_options(stream_results=True).execute(text(sql))
        columns = list(result.keys())
        rows = result.fetchmany(max_rows)
        return [{column: row[idx] for idx, column in enumerate(columns)} for row in rows]
```

**连接管理**：`_get_connection_string` 直接返回配置中的 `DATABASE_URL`，统一使用 MySQL。

### Step 6：Visualization（可视化推荐）

**目标**：分析查询结果的数据特征，推荐最合适的图表类型。

**实现文件**：`components/text2sql/visualization/node.py` + `visualization/prompts.py`

**Pydantic 模型**：

```python
class VisualizationRecommendation(BaseModel):
    chart_type: Literal["bar", "line", "pie", "scatter", "table"]
    title: str
    x_axis: Optional[str] = None
    y_axis: Optional[str] = None
    series: Optional[List[str]] = None
    config: Optional[Dict] = None
```

**提示词**：

LLM 扮演"数据可视化顾问"，根据查询意图、SQL 语句和结果样本（前 10 条）推荐图表类型。支持 5 种类型：`table`（表格）、`bar`（柱状图）、`line`（折线图）、`pie`（饼图）、`scatter`（散点图）。

**降级策略**：

- 如果查询结果为空，默认返回 `table` 类型。
- 如果 LLM 推荐失败（异常），也降级为 `table`。

### Step 7：Answer Formatting（答案格式化）

**目标**：将所有阶段的输出组装成用户可读的 Markdown 回答。

**实现文件**：`components/text2sql/formatting/node.py`

**格式化逻辑**：

```python
async def format_answer(state):
    if execution_error:
        answer = f"抱歉，执行 SQL 时出现错误：{execution_error}"
    else:
        answer_lines = []
        answer_lines.append(f"### 查询结果摘要\n问题：{question}")
        if analysis_text:
            answer_lines.append(analysis_text)        # 分析报告
        if results:
            preview = json.dumps(results[:5], ...)    # 前 5 条结果预览
            answer_lines.append(f"```json\n{preview}\n```")
        if visualization:
            answer_lines.append(f"- 类型：{chart_type}\n- 标题：{title}")
        answer = "\n".join(answer_lines)
```

**输出**：除了 `answer`（Markdown 文本），还保留 `sql_statement`、`execution_results`、`visualization`、`visualization_config` 在 state 中，方便前端直接读取并渲染图表。

---

## 4. 安全机制详解

Text2SQL 系统面临的核心安全风险是：LLM 生成的 SQL 可能包含破坏性操作。GustoBot 通过三层防护来应对：

### 第一层：提示词约束

SQL 生成的提示词明确要求：
- "默认只生成只读查询（SELECT/CTE）"
- "输出必须是单条 SQL 语句"

### 第二层：验证节点拦截

`validate_sql_syntax` 检查 SQL 必须以 `SELECT` 或 `WITH` 开头。`validate_sql_security` 检查是否包含以下危险关键词：

| 关键词 | 风险 |
|--------|------|
| `DROP TABLE` / `DROP DATABASE` | 删除表/数据库 |
| `TRUNCATE` | 清空表数据 |
| `DELETE FROM` | 删除行 |
| `INSERT INTO` | 插入数据 |
| `UPDATE` | 修改数据 |
| `ALTER TABLE` | 修改表结构 |
| `CREATE TABLE` / `CREATE DATABASE` | 创建对象 |
| `GRANT` / `REVOKE` | 权限操作 |
| `CALL` / `EXEC` | 执行存储过程 |

### 第三层：执行节点二次检查

`_is_read_only_query` 在执行前再次验证：
- 必须以 `SELECT`/`WITH`/`EXPLAIN`/`SHOW` 开头
- 不能包含任何危险关键词
- 不能包含多条语句（通过 `;` 检测）

```python
# 三层防护示意
SQL 生成 (提示词: 只生成 SELECT)
    |
    v
SQL 验证 (语法检查 + 安全关键词扫描)
    |
    v
SQL 执行 (再次检查只读 + 限制行数)
```

### 行数限制

执行节点默认限制返回 1000 行（`max_rows=1000`），使用 `fetchmany(max_rows)` 而非 `fetchall()`，避免一次性拉取大量数据导致内存溢出或响应过慢。

---

## 5. 重试机制

当 SQL 验证失败时，工作流不会直接报错，而是通过条件边回到 SQL 生成节点重新生成。

### 条件边决策函数

```python
def _should_execute_or_retry(state) -> Literal["execute", "retry", "end"]:
    is_valid = state.get("is_valid", False)
    retry_count = state.get("retry_count", 0)
    max_retries = state.get("max_retries", 3)

    if is_valid:
        return "execute"           # 验证通过 -> 执行
    if retry_count < max_retries:
        return "retry"             # 还有重试机会 -> 回到生成
    return "end"                   # 超出重试上限 -> 直接格式化错误回答
```

### 重试流程

```
generate_sql --> validate_sql
                   |
                   |-- is_valid=True  --> execute_sql
                   |-- is_valid=False && retry_count < 3 --> generate_sql (重试)
                   |-- is_valid=False && retry_count >= 3 --> format_answer (放弃)
```

**重试时的状态变化**：

- `retry_count` 递增（每次验证失败 +1）
- `validation_errors` 累加（使用 `operator.add` 合并多次验证的错误列表）
- `sql_statement` 被新生成的 SQL 覆盖

**注意**：当前实现中，重试时 LLM 并不会收到之前的验证错误信息。它只看到原始的 Schema、分析结果和问题。这是一个可以改进的地方——将验证错误反馈给 LLM 可以提高重试成功率。

---

## 6. 数据库 Schema：MySQL 表结构设计

GustoBot 的 Text2SQL 系统操作的是一个菜谱管理数据库 `recipe_db`，包含 7 张核心表：

### ER 关系图

```
cuisines (菜系)
    |  1
    |
    v  N
recipes (菜谱) <-------------------------+
    |         |                          |
    | 1       | 1                        |
    |         |                          |
    v N       v N                        |
recipe_ingredients              recipe_steps (步骤)
(菜谱用料)                          |
    |         |                      | 1
    | N       |                      |
    |         |                      v N
    v 1       |              step_tools (步骤工具)
ingredients (食材)                  |
                                   | N
                                   |
                                   v 1
                           cooking_tools (烹饪工具)
```

### 各表字段说明

**cuisines（菜系表）**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INT PK | 主键 |
| name | VARCHAR(255) | 華系名称，唯一 |
| cooking_style | TEXT | 烹饪特点描述 |
| typical_tools | JSON | 常用工具列表 |

**recipes（菜谱主表）**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INT PK | 主键 |
| name | VARCHAR(255) | 菜谱名称 |
| description | TEXT | 菜谱简介 |
| total_time | INT | 总耗时（分钟） |
| servings | INT | 份量，默认 4 |
| difficulty | ENUM | easy/medium/hard |
| cuisine_id | INT FK | 关联 cuisines.id |
| total_calories | DECIMAL | 每份总热量 (kcal) |
| total_protein | DECIMAL | 每份蛋白质 (g) |
| total_carbs | DECIMAL | 每份碳水 (g) |
| total_fat | DECIMAL | 每份脂肪 (g) |

**ingredients（食材表）**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INT PK | 主键 |
| name | VARCHAR(255) | 食材名称，唯一 |
| category | VARCHAR(100) | 分类（如"肉类-原料"） |
| calories | DECIMAL | 每 100g 热量 |
| protein | DECIMAL | 每 100g 蛋白质 |
| storage_method | VARCHAR(100) | 存储方式 |
| shelf_life | INT | 保质期（天） |

**recipe_ingredients（菜谱用料关联表）**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INT PK | 主键 |
| recipe_id | INT FK | 关联 recipes.id |
| ingredient_id | INT FK | 关联 ingredients.id |
| quantity | VARCHAR(100) | 用量 |
| unit | VARCHAR(50) | 单位 |
| is_main | BOOLEAN | 是否主料 |
| ingredient_type | ENUM | main/auxiliary/seasoning |

**recipe_steps（菜谱步骤表）**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INT PK | 主键 |
| recipe_id | INT FK | 关联 recipes.id |
| step_number | INT | 步骤序号 |
| action | VARCHAR(100) | 烹饪动作（如"爆炒"） |
| instruction | TEXT | 详细说明 |
| duration | INT | 耗时（分钟） |
| temperature | VARCHAR(50) | 火候（如"大火"） |

**cooking_tools（烹饪工具表）** 和 **step_tools（步骤工具关联表）** 用于记录每一步使用了哪些工具。

### 领域知识库

`domain_knowledge.py` 中定义了三个关键常量，为 LLM 提供数据库的"背景知识"：

- `TABLE_DESCRIPTIONS`：每张表的中文描述
- `COLUMN_DESCRIPTIONS`：每个字段的中文描述（以 `(表名, 列名)` 为 key）
- `RELATIONSHIP_FACTS`：表间关系的自然语言描述
- `DOMAIN_SUMMARY`：一段综合性的领域背景描述，直接注入到 LLM 提示词中

这些领域知识的作用是弥补 `INFORMATION_SCHEMA` 中缺少的语义信息——MySQL 的元数据只有表名和列名，没有"这张表是干什么的"这样的业务描述。

---

## 7. 结果格式化：SQL 结果怎么转成自然语言

答案格式化节点（`formatting/node.py`）将流水线各阶段的输出组装成结构化的 Markdown 文档：

1. **查询结果摘要**：包含用户原始问题。
2. **分析报告**：Step 2 产出的 Markdown 格式分析（查询意图、涉及表、关键字段、连接关系、筛选条件等）。
3. **结果预览**：查询结果的前 5 条，以 JSON 格式展示。
4. **可视化建议**：图表类型和标题。

如果执行出错，则直接返回错误信息："抱歉，执行 SQL 时出现错误：{错误详情}"。

最终的 `answer` 字段是完整的 Markdown 文本，可以直接发送给用户或在前端渲染。

---

## 8. 可视化：查询结果怎么生成图表

可视化推荐节点（`visualization/node.py`）使用 LLM 分析查询结果的数据特征，推荐最合适的图表类型。

### 支持的图表类型

| chart_type | 适用场景 |
|------------|---------|
| `table` | 通用，默认选项 |
| `bar` | 分类对比（如各菜系的菜谱数量） |
| `line` | 趋势变化（如按时间的热量变化） |
| `pie` | 占比分布（如食材类别占比） |
| `scatter` | 相关性分析（如热量 vs 蛋白质） |

### 推荐流程

1. 取查询结果的前 10 条作为样本。
2. 将用户问题、分析摘要、SQL 语句和样本数据发送给 LLM。
3. LLM 返回结构化的 `VisualizationRecommendation`（chart_type、title、x_axis、y_axis、series、config）。
4. 推荐结果写入 state 的 `visualization` 字段。

### 前端渲染

格式化节点会将 `visualization` 中的 `config` 提取到 `visualization_config` 字段。前端可以读取这个字段，使用 ECharts、Chart.js 等库渲染图表。

---

## 9. 领域知识的作用

`domain_knowledge.py` 是 Text2SQL 系统中一个容易被忽视但非常关键的组件。它解决了 LLM 的一个根本问题：**LLM 不知道你的数据库长什么样**。

### 为什么需要领域知识

MySQL 的 `INFORMATION_SCHEMA` 只提供：
- 表名：`recipes`
- 列名：`total_calories`
- 数据类型：`DECIMAL(10,2)`

它不提供：
- "这是菜谱的总热量，单位是千卡"
- "difficulty 只能取 easy/medium/hard"
- "cuisines 表没有 code 字段，按菜系筛选要用 name"

### 领域知识的三个层次

1. **表描述**（`TABLE_DESCRIPTIONS`）：告诉 LLM 每张表是干什么的。
2. **列描述**（`COLUMN_DESCRIPTIONS`）：告诉 LLM 每个字段的业务含义、取值范围。
3. **关系事实**（`RELATIONSHIP_FACTS`）：告诉 LLM 表之间怎么连接，关系类型是什么。
4. **领域摘要**（`DOMAIN_SUMMARY`）：一段综合描述，直接注入到所有 LLM 提示词的系统消息中。

这些信息同时被 Schema 检索节点（补充 `INFORMATION_SCHEMA` 缺失的描述）和 SQL 生成/查询分析节点（作为提示词上下文）使用。

---

## 10. Pydantic 模型：结构化输出的保障

Text2SQL 系统大量使用 Pydantic 模型来约束 LLM 的输出格式：

```python
# 查询分析的输出格式
class SQLAnalysis(BaseModel):
    query_intent: str
    required_tables: List[str]
    required_columns: List[str]
    join_conditions: Optional[str]
    filter_conditions: Optional[str]
    aggregation: Optional[str]
    order_by: Optional[str]
    notes: Optional[str]

# 可视化推荐的输出格式
class VisualizationRecommendation(BaseModel):
    chart_type: Literal["bar", "line", "pie", "scatter", "table"]
    title: str
    x_axis: Optional[str]
    y_axis: Optional[str]
    series: Optional[List[str]]
    config: Optional[Dict]
```

通过 `llm.with_structured_output(Model)` 让 LangChain 自动处理 JSON 解析和验证，确保 LLM 的输出严格符合预期格式。如果 LLM 输出了不符合模型定义的 JSON，LangChain 会抛出异常，工作流会捕获并降级处理。

---

## 11. 学习者自检清单

读完本文档后，试着回答以下问题来检验你的理解：

### 基础概念

- [ ] Text2SQL 的核心目标是什么？它解决了什么用户痛点？
- [ ] GustoBot 的 Text2SQL 流水线包含哪 7 个步骤？每个步骤的输入和输出分别是什么？
- [ ] `Text2SQLState` 中的 `Annotated[List[str], operator.add]` 是什么意思？为什么需要它？

### Schema 检索

- [ ] Schema 检索节点从哪里获取表结构信息？为什么不直接把所有表都给 LLM？
- [ ] 关键词打分机制是如何工作的？表名匹配和描述匹配的权重有什么区别？
- [ ] 领域知识库（`domain_knowledge.py`）在 Schema 检索中起什么作用？

### 查询分析与 SQL 生成

- [ ] 查询分析节点使用什么方法确保 LLM 输出结构化数据？
- [ ] SQL 生成节点为什么使用 `temperature=0.1`？
- [ ] `format_schema_as_text` 函数将 Schema 转换成什么格式？为什么选择这种格式？

### 安全机制

- [ ] Text2SQL 系统有几层安全防护？分别在哪些节点？
- [ ] `validate_sql_security` 检查哪些危险关键词？
- [ ] 执行节点的 `_is_read_only_query` 为什么要检查分号（`;`）？
- [ ] 为什么执行节点要限制返回行数？默认限制是多少？

### 重试机制

- [ ] 当 SQL 验证失败时，工作流如何决定是重试还是放弃？
- [ ] 重试时 LLM 能看到之前的验证错误吗？这有什么潜在问题？
- [ ] `max_retries` 默认是多少？可以在哪里配置？

### 可视化与格式化

- [ ] 可视化推荐节点在什么情况下会降级为 `table` 类型？
- [ ] 最终的 `answer` 字段包含哪些内容？
- [ ] `visualization_config` 和 `visualization` 有什么区别？分别给谁用？

### 进阶思考

- [ ] 如果用户问"删除所有菜谱"，系统会怎么处理？走完整个流程后用户会看到什么？
- [ ] 如何改进重试机制，让 LLM 在重试时能参考之前的验证错误？
- [ ] 当前实现中 Schema 检索最多返回 6 张表，这个限制有什么好处和潜在问题？
- [ ] 如果要支持多轮对话（用户追问"那热量最高的呢？"），需要做哪些改造？

---

## 12. 关键源文件索引

| 文件路径 | 职责 |
|----------|------|
| `gustobot/application/agents/text2sql/workflow.py` | 工作流组装，条件边决策 |
| `gustobot/application/agents/text2sql/state.py` | 状态定义（TypedDict） |
| `gustobot/application/agents/text2sql/models.py` | Pydantic 模型（SQLAnalysis、VisualizationRecommendation 等） |
| `gustobot/application/agents/text2sql/utils.py` | 工具函数（Markdown 渲染） |
| `components/text2sql/schema_retrieval/node.py` | Schema 检索（MySQL INFORMATION_SCHEMA） |
| `components/text2sql/query_analysis/node.py` | 查询意图分析 |
| `components/text2sql/query_analysis/prompts.py` | 查询分析提示词 |
| `components/text2sql/sql_generation/node.py` | SQL 生成 |
| `components/text2sql/sql_generation/prompts.py` | SQL 生成提示词 + Schema 格式化 |
| `components/text2sql/sql_validation/node.py` | SQL 验证节点 |
| `components/text2sql/sql_validation/validators.py` | 语法检查 + 安全检查 |
| `components/text2sql/sql_execution/node.py` | SQL 执行（SQLAlchemy） |
| `components/text2sql/visualization/node.py` | 可视化推荐 |
| `components/text2sql/visualization/prompts.py` | 可视化提示词 |
| `components/text2sql/formatting/node.py` | 最终答案格式化 |
| `components/text2sql/domain_knowledge.py` | 领域知识库（表/列描述、关系、摘要） |
| `data/init_mysql.sql` | MySQL 表结构初始化脚本 |
| `data/insert_sample_data.sql` | 示例数据插入脚本 |
