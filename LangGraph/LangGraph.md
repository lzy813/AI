# 一、简介

- LangGraph 是一个<font color="red">**用于组织和执行复杂图计算流程的框架**</font>。其底层运行逻辑基于 Google 发表的 Pregel 计算模型，核心思想是<font color="red">**将 Agent 的执行过程抽象为一张由节点和边组成的有向图**</font>。

- 在实际开发中，大语言模型本身已经非常成熟，开发者需要做的是<font color="red">**将整个计算流程编排好**</font>——先调用谁、后调用谁、数据如何流转。LangGraph 就是用来解决这个编排问题的



## 1、定位

- LangChain 1.x 的定位发生了明显变化：旧式 Chain、Retriever、Memory 等组件被迁移至 langchain‑classic；LCEL 保留为 langchain‑core 的底层组合机制，但不再是主要开发入口。
- LangChain 1.x 现在聚焦于 Agent 开发，核心入口是 create_agent，围绕 Agent 提供模型、工具、消息、结构化输出、Middleware 和记忆管理等高层抽象。
- LangGraph 是更底层的编排框架与 Agent Runtime，负责复杂工作流和有状态 Agent 的执行，提供持久化、流式输出、Durable Execution、Human‑in‑the‑loop 等运行时能力。create_agent 底层基于 LangGraph 实现

- LangGraph 与 LangChain 的关系

  - 两者并非对立，而是<font color="red">**协同工作**</font>的关系：

  | 框架      | 定位             | 适用场景                         |
  | --------- | ---------------- | -------------------------------- |
  | LangChain | 顶层应用层设计   | 快速上手、简单直观的 Agent 构建  |
  | LangGraph | 底层源码架构编排 | 复杂逻辑、持久化检查点、人机协同 |

  - LangGraph 与 LangChain 的对比


| 对比维度 | LangChain             | LangGraph                                |
| -------- | --------------------- | ---------------------------------------- |
| 定位     | Agent 高层开发框架    | 底层编排框架 & Agent Runtime             |
| 核心入口 | create_agent          | StateGraph / @entrypoint                 |
| 适用场景 | 结构直接的 Agent 应用 | 复杂工作流、持久化、长时间运行、人工介入 |
| 流程控制 | Agent 循环自动管理    | 精细控制节点、边、条件分支               |
| 学习成本 | 较低                  | 较高（但能力上限更高）                   |

  - <font color="red">**LangChain 提供易于使用的 Agent 高层抽象，LangGraph 提供可靠、可持久化且可精细控制的底层执行能力**</font>
  - 对于大多数 Agent 项目，从 LangChain 的 create_agent 开始即可；需要复杂工作流编排、确定性步骤与 Agent 步骤混合、长时间运行或底层状态控制时，再引入 LangGraph


![LangChain和LangGraph](图片/LangChain和LangGraph.png)



## 2、构成图的三要素

- 构成 LangGraph 图的三个基本要素：<font color="red">**State（状态）、Node（节点）、Edge（边）**</font>

- <font color="red">**State（状态）**</font>

  - State 是 LangGraph 中最核心的概念，它表示<font color="red">**图运行过程中的所有数据**</font>
  - <font color="red">**关键理解**</font>：不要被"状态"这个中文翻译误导，它本质上就是<font color="red">**数据容器**</font>

  - State 承载的内容包括：

    - <font color="red">**上下文信息**</font>：当前运行到哪一步、接下来要执行什么

    - <font color="red">**中间结果**</font>：某个节点执行完毕后的输出数据

    - <font color="red">**后续节点需要读取的数据**</font>：传递给下游节点的输入

  - 一句话总结：在 LangGraph 运行过程中，所有数据都存储在 State 里面。

- <font color="red">**Node（节点）**</font>

  - 节点是图中的<font color="red">**一个执行单元**</font>，通常实现为一个函数
  - 节点会读取当前 State，执行相应的业务逻辑，并返回对 State 的局部更新。节点本身并不直接修改全局状态，状态的合并与提交由运行时统一完成。

  - 如果一个 Agent 只需要调用一次大模型，那就不需要构建图。正因为 Agent 内部的逻辑和流程非常复杂，才需要多个节点组合起来构成图。<font color="red">**每个节点负责完成一个具体的操作**</font>，比如调用某个大模型、处理数据、执行工具等。

- <font color="red">**Edge（边）**</font>

  - 边表示<font color="red">**节点与节点之间的流转关系**</font>，定义了执行的先后顺序。
  - Edge 可以是固定流转，也可以根据当前 State 进行条件判断，从而实现分支、循环等复杂控制流程。

  - 最简单的图结构：START -> Node 1 -> Node 2 -> END，其中每一条箭头就是一条边。边不仅定义了顺序，后续还可以包含条件判断逻辑，实现分支和并行。

![三要素](图片/三要素.png)



## 3、图运行过程

- 超步（SuperStep）：LangGraph 的图在运行时，被拆分成一个一个的<font color="red">**SuperStep（超步）**</font>。<font color="red">**超步是图运行的最小组成单元**</font>，每个超步处理一个节点，由三个阶段组成。

- 超步的三个阶段

  - 阶段1：路由（Routing）：当一个节点执行完成后，路由阶段决定<font color="red">**接下来要执行哪个节点**</font>
    - 对于简单线性结构：直接指向下一个节点
    - 对于复杂结构：可能有多个候选节点（并行执行）或条件判断（选择执行）
    - 边的逻辑在这一阶段被触发，确定下一步的执行目标

  ![运行图1](图片/运行图1.png)

  - 阶段2：执行（Execution）：确定目标节点后，开始<font color="red">**执行该节点的代码逻辑**</font>

    - **输入**：读取当前的 State 快照作为参数
    - **处理**：运行节点内部的函数逻辑
    - **输出**：产生局部更新（此时不提交到全局状态）

    - 关键点：执行阶段产生的更新是<font color="red">**局部的**</font>，不会立即影响全局状态。

  ![运行图2](图片/运行图2.png)

  - 阶段3：提交（Commit）：将节点执行过程中产生的数据更新<font color="red">**合并到全局状态**</font>中
    - 将日志、执行内容、当前执行的节点 ID 等信息统一提交
    - 更新全局 State，供后续节点读取
    - 这一步完成后，一个超步才算真正结束

  ![运行图3](图片/运行图3.png)

- 开始与结束的例外
  - **开始**：从 START 标记直接进入第一个节点，不经过完整超步
  - **结束**：当节点连接到 END 标记时，图运行结束

- 检查点（Checkpoint）：检查点是图运行过程中的<font color="red">**状态保存机制**</font>，类似于在不同位置拍照记录当前状态。它用于：

  - 持久化保存运行过程中的状态数据

  - 支持中断恢复和状态回溯

  - 后续课程会详细讲解



## 4、API分类

- LangGraph 提供了两种不同的 API 来构建运行图：<font color="red">**Graph API（图式 API）**</font> 和 <font color="red">**Functional API（函数式 API）**</font>。这两种 API 共享相同的底层运行时，可以在同一应用程序中协同使用，但它们针对不同的使用场景和开发偏好而设计。



### 3.1 图式 API

- Graph API 采用声明式方式构建工作流。开发者需要显式定义 State、Node 和 Edge，<font color="red">**将业务流程组织成一个可可视化的图结构**</font>

- 当流程中存在较复杂的分支、多个节点之间共享状态、并行执行、结果汇聚，或者需要通过图结构帮助调试和团队协作时，更适合使用 Graph API。官方文档也明确建议，在需要复杂流程可视化、显式状态管理、多条件分支、并行路径以及团队协作时，优先选择 Graph API。

- 总之：<font color="red">**Graph API 更适合构建结构清晰、节点关系复杂、需要长期维护的工作流**</font>

- 典型场景包括：

| 场景               | 说明                                               |
| ------------------ | -------------------------------------------------- |
| 多节点复杂流程     | 流程中存在多个处理节点，需要清晰表达节点之间的关系 |
| 条件分支较多       | 根据 State 中的不同字段决定后续执行路径            |
| 并行执行与结果汇聚 | 多个节点并行运行，之后汇总结果                     |
| 多组件共享状态     | 多个节点都需要读写同一个全局 State                 |
| 需要图结构展示     | 便于调试、讲解、文档化和团队协作                   |



### 3.2 Functional API

- Functional API 采用<font color="red">**命令式方式构建工作流**</font>，更贴近普通 Python 函数调用。开发者可以使用 @entrypoint 定义工作流入口，使用 @task 定义可做检查点记录的任务，然后在函数内部直接使用普通的 if/else、循环和函数调用来编排流程。官方文档指出：当已有过程式代码尽量少改动、流程以线性为主、分支逻辑简单、希望快速做原型验证时，优先 Functional API。

- 可以这样理解：Functional API 更适合在普通 Python 函数流程里，低成本接入 LangGraph 的持久化、中断恢复、任务检查点能力。

- 典型场景包括：

| 场景           | 说明                                          |
| -------------- | --------------------------------------------- |
| 现有代码改造   | 已有函数式 / 过程式代码，不想完整重构成图结构 |
| 线性流程       | 主要按 A → B → C 顺序执行                     |
| 简单分支       | 只有少量 if/else 判断                         |
| 快速原型验证   | 减少样板代码，快速跑通业务逻辑                |
| 局部任务持久化 | 希望部分函数作为独立 task 做检查点记录        |



### 3.3 二者的核心区别

| 对比项     | Graph API                      | Functional API                 |
| ---------- | ------------------------------ | ------------------------------ |
| 编程风格   | 声明式图结构                   | 命令式函数流程                 |
| 核心抽象   | State、Node、Edge              | entrypoint、task               |
| 状态管理   | 显式定义全局 State             | 更多依靠函数参数、返回值       |
| 流程表达   | 通过节点和边来表达             | 用原生 Python 控制流表达       |
| 可视化能力 | 强，天然支持绘图调试           | 弱，和普通代码一样             |
| 适合场景   | 复杂工作流、多分支、多节点协作 | 简单流程、快速原型、旧代码迁移 |
| 学习成本   | 相对更高                       | 相对更低                       |



# 二、图的基本构建与运行

## 1、入门案例

### 1.1 项目依赖

- 创建项目，导入依赖

~~~bash
# 创建项目
uv init LangGraphDemo
# 添加依赖
uv add langgraph
~~~



### 1.2 定义状态图步骤

- 状态图的定义可以分为以下几个步骤：
  - <font color="red">**1. 定义全局状态**</font> 定义整个运行图共享的全局状态（实际上，除了全局状态，LangGraph还支持其它类型的状态，下文详述）
  - <font color="red">**2. 创建状态图**</font> 创建 StateGraph实例，将状态和状态图绑定
  - <font color="red">**3. 定义图结构**</font> 定义节点和边并添加到状态图中



#### 1.2.1 定义全局状态

- <font color="red">**全局状态是 LangGraph 运行图每个节点都可以访问的公共对象**</font>

~~~python
class OverAllState(TypedDict):
    logs: Annotated[list[str], add]
    cur_id: str
~~~



#### 1.2.2 创建状态图

~~~python
builder = StateGraph(state_schema=OverAllState)
~~~



#### 1.2.3 定义图结构

~~~python
def node_1(state: OverAllState) -> OverAllState:
    pre_id = state["cur_id"]
    return {
        "logs": ["node_1 运行完毕"],
        "cur_id": pre_id + ", node_1"
    }

def node_2(state: OverAllState) -> OverAllState:
    pre_id = state["cur_id"]
    return {
        "logs": ["node_2 运行完毕"],
        "cur_id": pre_id + ", node_2"
    }
~~~



#### 1.2.4 添加节点

~~~python
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
~~~



#### 1.2.5 添加边

~~~python
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", END)
~~~



#### 1.2.6 编译状态图

- 将上一步得到的状态图编译为可执行的运行图

~~~python
graph = builder.compile()
~~~



#### 1.2.7 调用状态图

- 可以通过invoke调用计算图

~~~python
print(graph.invoke({"cur_id": "start"}))
~~~



### 1.3 总体代码

~~~python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from operator import add


# ============================================================
# 第一步：定义状态
# ============================================================
class OverAllState(TypedDict):
    # 日志字段：类型为 list[str]，更新方式为 add（追加）
    logs: Annotated[list[str], add]
    # 当前流程位置字段：类型为 str，默认覆盖更新
    cur_id: str


# ============================================================
# 第二步：定义节点
# ============================================================
def node_1(state: OverAllState) -> OverAllState:
    """节点1：在日志中追加记录，拼接当前ID"""
    pre_id = state["cur_id"]
    return {
        "logs": ["node_1 运行完毕"],
        "cur_id": pre_id + ", node_1"
    }

def node_2(state: OverAllState) -> OverAllState:
    """节点2：在日志中追加记录，拼接当前ID"""
    pre_id = state["cur_id"]
    return {
        "logs": ["node_2 运行完毕"],
        "cur_id": pre_id + ", node_2"
    }


# ============================================================
# 第三步：定义边（构建图）
# ============================================================
# 3.1 创建图的建造者
builder = StateGraph(state_schema=OverAllState)

# 3.2 添加节点
builder.add_node(node_1)
builder.add_node(node_2)

# 3.3 添加边（定义执行顺序）
builder.add_edge(START, "node_1")      # 开始 -> 节点1
builder.add_edge("node_1", "node_2")   # 节点1 -> 节点2
builder.add_edge("node_2", END)        # 节点2 -> 结束

# ============================================================
# 第四步：编译图
# ============================================================
graph = builder.compile()

# ============================================================
# 第五步：运行图
# ============================================================
result = graph.invoke({"cur_id": "start"})
print(result)
# 输出: {'logs': ['node_1 运行完毕', 'node_2 运行完毕'], 'cur_id': 'start, node_1, node_2'}
~~~

- **代码解读**：

  - **状态定义部分**：

    - TypedDict：Python 内置的类型化字典，LangGraph 官方推荐的状态定义方式

    - Annotated[list[str], add]：第一个参数 list[str] 是字段的数据类型，第二个参数 add 是 reducer 函数，表示该字段更新时采用追加方式而非覆盖

    - cur_id: str：普通字符串字段，没有指定 reducer，更新时默认覆盖

  - **节点定义部分**：

    - 节点本质上是普通 Python 函数

    - 参数 state 是当前状态的快照，类型为 OverAllState

    - 返回值是字典形式，只包含需要更新的字段（部分更新）

    - logs 字段因为配置了 add reducer，返回值会被追加到已有列表中

    - cur_id 字段没有配置 reducer，返回值会覆盖旧值

  - **边定义部分**：

    - 使用建造者模式：先创建 StateGraph 获取 builder

    - add_node 添加节点到图中

    - add_edge 定义节点间的连接关系，参数为字符串形式的节点名称

    - START 和 END 是 LangGraph 内置的特殊标记

  - **运行结果分析**：

    - logs 字段：['node_1 运行完毕', 'node_2 运行完毕'] — 两个节点的日志被追加合并

    - cur_id 字段：'start, node_1, node_2' — 手动拼接的结果



## 2、图结构可视化

### 2.1 Mermaid方式

- mermaid是一种生成图表和流程图的文本标记语言，它可以把简单的文本描述直接渲染成可视化图形。
  - 通过以下代码可以用mermaid语法表达图结构，并打印mermaid源码

~~~python
# 获取 Mermaid 语法的图结构源码
raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- 完整代码

~~~python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from operator import add


# ============================================================
# 第一步：定义状态
# ============================================================
class OverAllState(TypedDict):
    # 日志字段：类型为 list[str]，更新方式为 add（追加）
    logs: Annotated[list[str], add]
    # 当前流程位置字段：类型为 str，默认覆盖更新
    cur_id: str


# ============================================================
# 第二步：定义节点
# ============================================================
def node_1(state: OverAllState) -> OverAllState:
    """节点1：在日志中追加记录，拼接当前ID"""
    pre_id = state["cur_id"]
    return {
        "logs": ["node_1 运行完毕"],
        "cur_id": pre_id + ", node_1"
    }

def node_2(state: OverAllState) -> OverAllState:
    """节点2：在日志中追加记录，拼接当前ID"""
    pre_id = state["cur_id"]
    return {
        "logs": ["node_2 运行完毕"],
        "cur_id": pre_id + ", node_2"
    }


# ============================================================
# 第三步：定义边（构建图）
# ============================================================
# 3.1 创建图的建造者
builder = StateGraph(state_schema=OverAllState)

# 3.2 添加节点
builder.add_node(node_1)
builder.add_node(node_2)

# 3.3 添加边（定义执行顺序）
builder.add_edge(START, "node_1")      # 开始 -> 节点1
builder.add_edge("node_1", "node_2")   # 节点1 -> 节点2
builder.add_edge("node_2", END)        # 节点2 -> 结束

# ============================================================
# 第四步：编译图
# ============================================================
graph = builder.compile()

# ============================================================
# 第五步：运行图
# ============================================================
result = graph.invoke({"cur_id": "start"})
print(result)
# 输出: {'logs': ['node_1 运行完毕', 'node_2 运行完毕'], 'cur_id': 'start, node_1, node_2'}


# ============================================================
# 第六步：获取 Mermaid 语法的图结构源码
# ============================================================
raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- **代码解读**：graph.get_graph() 获取图结构对象，.draw_mermaid() 将其转换为 Mermaid 语法字符串。Mermaid 是一种文本格式的图表描述语言，支持在 Markdown 文档中直接渲染
- 输出源码

~~~bash
---
config:
  flowchart:
    curve: linear
---
graph TD;
	__start__([<p>__start__</p>]):::first
	node_1(node_1)
	node_2(node_2)
	__end__([<p>__end__</p>]):::last
	__start__ --> node_1;
	node_1 --> node_2;
	node_2 --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 将 graph TD 节点粘贴为代码，格式为mermaid

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_1(node_1)
	node_2(node_2)
	__end__([<p>__end__</p>]):::last
	__start__ --> node_1;
	node_1 --> node_2;
	node_2 --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~



### 2.2 保存本地

- 将图保存为本地 PNG 文件

~~~python
# 将图保存为本地 PNG 文件
png_bytes = graph.get_graph().draw_mermaid_png()
png_filename = 'D:/kn/AI/first_demo_graph.png'
with open(png_filename, "wb") as f:
    f.write(png_bytes)
~~~

- 全部代码

~~~python
from tkinter import Image

from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from operator import add


# ============================================================
# 第一步：定义状态
# ============================================================
class OverAllState(TypedDict):
    # 日志字段：类型为 list[str]，更新方式为 add（追加）
    logs: Annotated[list[str], add]
    # 当前流程位置字段：类型为 str，默认覆盖更新
    cur_id: str


# ============================================================
# 第二步：定义节点
# ============================================================
def node_1(state: OverAllState) -> OverAllState:
    """节点1：在日志中追加记录，拼接当前ID"""
    pre_id = state["cur_id"]
    return {
        "logs": ["node_1 运行完毕"],
        "cur_id": pre_id + ", node_1"
    }

def node_2(state: OverAllState) -> OverAllState:
    """节点2：在日志中追加记录，拼接当前ID"""
    pre_id = state["cur_id"]
    return {
        "logs": ["node_2 运行完毕"],
        "cur_id": pre_id + ", node_2"
    }


# ============================================================
# 第三步：定义边（构建图）
# ============================================================
# 3.1 创建图的建造者
builder = StateGraph(state_schema=OverAllState)

# 3.2 添加节点
builder.add_node(node_1)
builder.add_node(node_2)

# 3.3 添加边（定义执行顺序）
builder.add_edge(START, "node_1")      # 开始 -> 节点1
builder.add_edge("node_1", "node_2")   # 节点1 -> 节点2
builder.add_edge("node_2", END)        # 节点2 -> 结束

# ============================================================
# 第四步：编译图
# ============================================================
graph = builder.compile()

# ============================================================
# 第五步：运行图
# ============================================================
result = graph.invoke({"cur_id": "start"})
print(result)
# 输出: {'logs': ['node_1 运行完毕', 'node_2 运行完毕'], 'cur_id': 'start, node_1, node_2'}

# ============================================================
# 将图保存为本地 PNG 文件
# ============================================================
png_bytes = graph.get_graph().draw_mermaid_png()
png_filename = 'D:/kn/AI/first_demo_graph.png'
with open(png_filename, "wb") as f:
    f.write(png_bytes)
~~~



# 三、状态(State)管理

##  1、状态定义

- 状态的定义实际上是在声明状态的 Schema，后者是状态字段的完整描述。 
- 官方推荐了三种定义 Schema 的方式：TypedDict、dataclass、Pydantic



### 1.1 TypedDict方式

- TypedDict 是 Python 内置的类型化字典，使用字典语法访问字段。LangGraph 官方推荐的首选方式

~~~python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from operator import add


class OverAllState(TypedDict):
    logs: Annotated[list[str], add]
    cur_id: str


def node_1(state: OverAllState) -> OverAllState:
    pre_id = state["cur_id"]
    return {
        "logs": ["node_1 运行完毕"],
        "cur_id": pre_id + ", node_1"
    }


def node_2(state: OverAllState) -> OverAllState:
    pre_id = state["cur_id"]
    return {
        "logs": ["node_2 运行完毕"],
        "cur_id": pre_id + ", node_2"
    }


builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", END)

graph = builder.compile()
print(graph.invoke({"cur_id": "start"}))
~~~

- 输出：{'logs': ['node_1 运行完毕', 'node_2 运行完毕'], 'cur_id': 'start, node_1, node_2'}



### 1.2 dataclass方式

- 属性调用方式由 ['字段名'] 变为 <font color='red'>**.字段名**</font>

~~~python
from langgraph.graph import StateGraph, START, END
from typing import Annotated
from dataclasses import dataclass
from operator import add


@dataclass
class OverAllState:
    logs: Annotated[list[str], add]
    cur_id: str


def node_1(state: OverAllState) -> OverAllState:
    pre_id = state.cur_id
    return OverAllState(
        logs=state.logs + ["node_1 运行完毕"],
        cur_id=pre_id + ", node_1"
    )


def node_2(state: OverAllState) -> OverAllState:
    pre_id = state.cur_id
    return OverAllState(
        logs=state.logs + ["node_2 运行完毕"],
        cur_id=pre_id + ", node_2"
    )


builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", END)


graph = builder.compile()
print(graph.invoke({"cur_id": "start"}))
~~~

- @dataclass 是 Python 的装饰器，用于自动生成 __init__、__repr__ 等方法
- 使用点号（.）方式访问属性，而非字典的中括号方式
- 返回时可以返回字典，LangGraph 会自动处理转换
- 也可以返回 dataclass 实例



### 1.3 Pydantic方式

- Pydantic 的 BaseModel 提供数据校验能力，字段访问方式与 dataclass 相同（点号访问）

~~~python
from langgraph.graph import StateGraph, START, END
from typing import Annotated
from pydantic import BaseModel
from operator import add

# 1. 定义状态（使用 Pydantic BaseModel）
class OverAllState(BaseModel):
    logs: Annotated[list[str], add]
    cur_id: str

# 2. 定义节点
def node_1(state: OverAllState) -> OverAllState:
    pre_id = state.cur_id
    return {
        "logs": ["node_1 运行完毕"],
        "cur_id": pre_id + ", node_1"
    }

def node_2(state: OverAllState) -> OverAllState:
    pre_id = state.cur_id
    return {
        "cur_id": pre_id + ", node_2"
    }

# 3. 定义边
builder = StateGraph(state_schema=OverAllState)
builder.add_node(node_1)
builder.add_node(node_2)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", END)

graph = builder.compile()

# 4. 运行（可以传字典，Pydantic 会自动解析）
result = graph.invoke({"cur_id": "start"})
print(result)
# 输出: {'logs': ['node_1 运行完毕'], 'cur_id': 'start, node_1, node_2'}
~~~

- 使用 BaseModel 作为状态类的父类
- 字段访问使用点号方式 state.cur_id
- invoke 时传入字典，Pydantic 会自动解析为模型对象
- 注意输出中 logs 只有 node_1 的记录，因为 node_2 没有返回 logs 字段，而 Pydantic 不会保留未更新的字段默认值



### 1.4 总结

- 在 LangGraph 中，三种方式都有字段校验，但校验机制不同：

| 方式          | 校验行为                             | 异常类型        |
| ------------- | ------------------------------------ | --------------- |
| **TypedDict** | 将字段视为字典 Key，Key 不匹配抛异常 | KeyError        |
| **dataclass** | 将字段视为类属性，属性不匹配抛异常   | TypeError       |
| **Pydantic**  | 字段不匹配抛异常                     | ValidationError |

- **重要区别**：在 LangChain 中只有 Pydantic 有严格校验；但在 LangGraph 中，三种方式都会进行校验。

- 节点返回字段与状态字段不匹配时
  - 如果节点返回的字段名与状态定义的字段名不匹配，三种方式的行为是统一的：**该更新会被忽略**，不会影响全局状态。

- 推荐使用 TypedDict 的原因

  - **功能无差异**：三种方式在 LangGraph 中都会进行字段校验

  - **更贴近状态更新机制**：LangGraph 的状态更新本质上是字典更新，TypedDict 最自然

  - **轻量级**：不引入额外的数据校验开销

  - **官方首选**：大多数官方文档和示例都使用 TypedDict

  - **简洁清晰**：写法最简洁，结构最清晰
  - 总结：相比普通 dict，TypedDict 可以提供更明确的字段约束和类型提示；相比 dataclass，TypedDict 更贴近 LangGraph 中状态的更新方式，因为节点通常返回的是表示“部分状态更新”的字典，而不是完整对象；相比 Pydantic BaseModel，它又更加轻量，不会引入额外的数据校验开销。因此，在没有复杂校验需求的情况下，TypedDict 是定义 LangGraph State Schema 的首选方式



## 2、状态合并

### 2.1 什么是Reducer

- State Reducer 是 LangGraph 中用于<font color="red">**合并状态更新的核心机制**</font>。在 LangGraph 的 stateGraph 中，每个节点可以读取和写入共享状态，而 Reducer 定义了如何<font color="red">**将多个节点对同一状态键的更新合并**</font>

- **Reducer 的核心特征：**

  - <font color="red">**函数签名**</font>：(value, value) -> value，接收当前值和更新值，返回合并后的新值
  - <font color="red">**注解定义**</font>：通过 Annotated[Type, reducer_function] 为状态键指定 Reducer

  - <font color="red">**默认行为**</font>：未指定 Reducer 的状态键使用覆盖策略（Last‑Write‑Wins）



### 2.2 如何定义Reducer

#### 2.2.1 定义Reducer

- Reducer 本质上是一个二元合并函数，用于定义当同一个字段产生多个更新值时，LangGraph 应该如何将这些值合并为一个最终结果

- 函数签名：(Value, Value) -> Value
- 示例代码如下：

~~~python
def my_reducer(left: list[str], right: list[str]) -> list[str]:
    return left + right

left = ['a', 'b']
right = ['c']
print(my_reducer(left, right))
~~~

- 其中，my_reducer 用于处理 list[str] 类型的数据。它接收两个列表参数：

  - left：当前已累计的状态值；

  - right：本次待合并的新值。

- 函数内部通过 left + right 将两个列表合并，并返回合并后的结果。

- 因此，该 Reducer 的作用是：<font color='red'>**当某个状态字段存在多次列表更新时，将这些列表内容追加合并，而不是直接覆盖原值。**</font>

- 运行结果如下

~~~bash
['a', 'b', 'c']
~~~



#### 2.2.2 将Reducer和状态字段关联

- 在 LangGraph 中，Reducer 通常通过 Python 的 typing.Annotated 与状态字段进行关联。

- Annotated[] 是 Python 提供的一种类型注解扩展机制，<font color='red'>**用于在原始类型之外附加额外的元数据信息**</font>。需要注意的是，Annotated[] 本身并不规定这些元数据的具体含义，它只负责在类型注解中保留这些信息。

- 严格来说，Annotated 的第一个参数是被注解的原始类型，后续参数是附加的元数据。至于这些元数据表示什么、如何解析，则由使用它的框架或工具自行决定。

- 在 LangGraph 中，框架利用这一机制，将状态字段的类型和 Reducer 规则同时声明在字段定义中。其基本形式如下：

```python
Annotated[Type, reducer_function]
```

- 其中：

  - Type：表示状态字段的数据类型；

  - reducer_function：表示该字段对应的 Reducer 函数。

- 示例代码如下：

```python
from typing import TypedDict, Annotated

class OverAllState(TypedDict):
    logs: Annotated[list[str], my_reducer]
    cur_id: str
```

- 在上述代码中：

  - logs 字段的类型是 list[str]；

  - my_reducer 是与 logs 字段关联的 Reducer 函数；

  - 当多个节点同时更新 logs 字段时，LangGraph 会使用 my_reducer 将多个列表合并；

  - cur_id 字段没有指定 Reducer，因此采用默认更新规则。



#### 2.2.3 常用内置Reducer函数

- ###### <font color="red">**operator.add**</font>

  - operator.add 是 Python 内置的加法操作函数，底层由 C 实现

  - 它接收两个参数，等价于 a（第一个参数）+ b（第二个参数）

  - 代码如下

  ```python
  from operator import add
  
  print(f"{add(1,2) = }")
  print(f"{add([1,2], [3,4]) = }")
  print(f"{add(['a','b'], ['c']) = }")
  ```

  - 输出如下

  ```bash
  add(1,2) = 3
  add([1,2], [3,4]) = [1, 2, 3, 4]
  add(['a','b'], ['c']) = ['a', 'b', 'c']
  ```

- <font color="red">**langgraph.graph.message.add_messages**</font>

  - add_messages 是 LangGraph 中专用于合并消息列表的 Reducer 函数，常用于维护对话历史类的状态字段。其函数签名如下

  ~~~python
  def add_messages(
      left: Messages,
      right: Messages,
      *,
      format: Literal["langchain-openai"] | None = None,
  ) -> Messages:
      ...
      return merged
  ~~~

  - 参数说明：
    - left：状态中已有的消息列表；
    - right：当前节点返回的消息更新值；
    - format：可选参数，用于指定返回消息的格式，通常无需手动设置

  - <font color="red">**left 与 right 的类型均为 Messages**</font>。Messages 可以理解为 LangChain 消息对象的列表，其中每个元素都是 BaseMessage 或其子类的实例，常见子类包括：

    - HumanMessage：用户的输入消息；
    - AIMessage：AI 的回复消息；
    - SystemMessage：系统提示消息；
    - ToolMessage：工具调用的结果消息。

  - <font color="red">**add_messages 处理的是对话消息序列，而非普通的字符串列表**</font>

  - BaseMessage 包含一个可选的 id 属性，用于唯一标识一条消息。add_messages 在合并 left 与 right 时，不是简单地执行列表拼接，而是依据消息的 id 进行合并：

    - 若 right 中的某条消息的 id 在 left 中不存在，则将该消息追加到结果列表末尾；
    - 若 right 中的某条消息的 id 与 left 中已有消息的 id 相同，则使用 right 中的新消息替换 left 中的旧消息。

  - 因此，add_messages 的作用可以概括为：<font color="red">**在保留历史消息的基础上追加新消息，并允许通过相同的消息 id 覆盖已有消息**</font>。

    - 需要特别说明，add_messages 并非简单地对 left 与 right 求“并集”。更准确地说，它是一个基于消息 id 的消息列表合并函数：既支持追加新消息，也支持更新已有消息。
    - 可以理解为：

    ~~~python
    merged = left + right
    ~~~

    - 但若 right 中存在与 left 相同 id 的消息，则最终结果中不会出现重复消息，而是用 right 中的消息覆盖 left 中对应的旧消息

  - 示例代码如下

  ~~~python
  from langgraph.graph.message import add_messages
  from langchain.messages import HumanMessage, AIMessage, SystemMessage
  
  left = [
      SystemMessage(content="你是个善解人意的助手", id='1'),
      HumanMessage(content="你好", id='2'),
      AIMessage(content="你好~", id='3'),
  ]
  
  right = [
      HumanMessage(content="我是老王，你是小王", id='2'),
      AIMessage(content="好的，我记住啦", id='3'),
      HumanMessage(content="你是谁？", id='4'),
      AIMessage(content="我是小王", id='5'),
  ]
  
  merged = add_messages(left, right)
  
  for msg in merged:
      print(msg)
  ~~~

  - 输出如下

  ~~~bash
  content='你是个善解人意的助手' additional_kwargs={} response_metadata={} id='1'
  content='我是老王，你是小王' additional_kwargs={} response_metadata={} id='2'
  content='好的，我记住啦' additional_kwargs={} response_metadata={} id='3' tool_calls=[] invalid_tool_calls=[]
  content='你是谁？' additional_kwargs={} response_metadata={} id='4'
  content='我是小王' additional_kwargs={} response_metadata={} id='5' tool_calls=[] invalid_tool_calls=[]
  ~~~



#### 2.2.4 默认行为

- 如果某个 State 字段没有显式定义 Reducer，LangGraph 会使用默认的状态更新行为：<font color='red'>**后一次更新值会覆盖该字段原有的状态值。**</font>

- 换句话说，当节点返回的更新结果中包含某个字段时，如果该字段没有配置 Reducer，LangGraph 不会对新旧值进行追加、合并或累加，<font color='red'>**而是直接使用本次返回的新值替换原来的旧值。**</font>

- 示例代码如下：

~~~python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class OverAllState(TypedDict):
    logs: list[str]
    id: str

def node_a(state: OverAllState):
    return {
        "logs": ["node_a"],
        "id": "node_a"
    }

def node_b(state: OverAllState):
    return {
        "logs": ["node_b"],
        "id": "node_b"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)

graph = builder.compile()
result = graph.invoke({"logs": ["START"], "id": "start"})
print('=' * 30, '-> result <-', '=' * 30)
print(result)
~~~

- 输出结果

~~~bash
{'logs': ['node_b'], 'id': 'node_b'}
~~~

- 可以看到，logs字段和id字段都没有定义Reducer，因此，节点返回的新值会覆盖初始状态中的旧值，图运行结果中的状态值和最后一次更新保持一致



## 3、节点中访问State

### 3.1  图节点中读取State

- 在 LangGraph 中，节点本质上是一个可调用对象，通常定义为普通 Python 函数。节点函数被执行时，LangGraph 会自动将当前图运行到该节点时的 State 传入节点函数。

- 节点函数的第一个参数通常是当前运行图的状态对象，也就是 State。

~~~python
def node(state: StateSchema):
    ...
~~~

- 其中，state 表示当前节点执行时可以访问到的全局状态快照。节点可以通过读取 state 中的字段获取上游节点写入的数据，并基于这些数据完成当前节点的业务逻辑。
- 示例代码如下：

~~~python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from operator import add

class OverAllState(TypedDict):
    logs: Annotated[list[str], add]
    id: str

def node_a(state: OverAllState):
    for k, v in state.items():
        print(f"k: {k}, v: {v}")

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", END)

graph = builder.compile()
result = graph.invoke({"logs": ["START"], "id": "start"})
~~~

- 输出如下：

~~~bash
k: logs, v: ['START']
k: id, v: start
~~~



### 3.2 图节点更新State

- 在 LangGraph 中，节点函数通常不需要返回更新后的完整状态，只需要返回本节点对状态的局部更新。

- 也就是说，<font color='red'>**节点的返回值可以只包含需要修改的状态字段**</font>

- 对于节点没有返回的字段，LangGraph 会保留其原有状态值；
- 对于节点返回的字段，<font color='red'>**LangGraph 会根据该字段是否配置了 Reducer 来决定如何合并更新值**</font>
  - 如果字段配置了 Reducer，则使用对应的 Reducer 函数将旧值和新值合并；
  - 如果字段没有配置 Reducer，则按照默认规则使用节点返回的新值覆盖原值。

- LangGraph运行时会按照状态字段的Reducer函数将其与当前的最新状态合并。

- 示例代码如下：

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from operator import add

class OverAllState(TypedDict):
    logs: Annotated[list[str], add]
    id: str

def node_a(state: OverAllState):
    for k, v in state.items():
        print(f"k: {k}, v: {v}")
    return {
        "logs": ["node_a 更新状态"]
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", END)

graph = builder.compile()
result = graph.invoke({"logs": ["START"], "id": "start"})
print('=' * 30, '-> result <-', '=' * 30)
print(result)
```

- 输出如下

```bash
k: logs, v: ['START']
k: id, v: start
============================== -> result <- ==============================
{'logs': ['START', 'node_a 更新状态'], 'id': 'start'}
```

- 在上述示例中：
  - logs 字段通过 Annotated[list[str], add] 绑定了 Reducer 函数 operator.add，因此 LangGraph 会将原有的 logs 值和 node_a 返回的新 logs 值进行列表拼接：

```bash
["START"] + ["node_a 更新状态"]
```

- 最终得到：

```bash
["START", "node_a 更新状态"]
```

- id 字段没有出现在 node_a 的返回值中，因此该字段不会被更新。图运行结束后，输出状态中的 id 仍然保持输入时的值：

```bash
"id": "start"
```

- 因此，LangGraph 节点更新 State 的核心规则可以概括为：
- <font color='red'>**节点只返回需要更新的字段；未返回的字段保持不变；返回的字段根据是否配置 Reducer 决定是合并还是覆盖**</font>



### 3.3 Overwrite绕过Reducer

- 如果某个状态字段定义了 Reducer，那么节点返回该字段的更新值时，LangGraph 默认会通过对应的 Reducer 将新值与已有状态值进行合并。

- 在某些场景下，可能并不希望继续执行 Reducer 的聚合逻辑，而是希望本次更新直接覆盖旧值。这时可以使用 Overwrite。

- Overwrite 的作用是：<font color='red'>**告诉 LangGraph 本次状态更新不走该字段原本定义的 Reducer，而是直接用新值覆盖状态中的旧值**</font>

- 需要注意的是，<font color='red'>**Overwrite 只影响当前这一次更新，并不会修改状态字段本身的 Reducer 定义。后续节点如果继续正常返回该字段的更新值，仍然会按照原来的 Reducer 逻辑进行合并**</font>

- 示例代码如下：

~~~python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Overwrite
from typing import TypedDict, Annotated
from operator import add

class OverAllState(TypedDict):
    logs: Annotated[list[str], add]
    id: str

def node_a(state: OverAllState):
    return {
        "logs": ["node_a"],
        "id": "node_a"
    }

def node_b(state: OverAllState):
    return {
        "logs": Overwrite(["node_b"]),
        "id": "node_b"
    }

def node_c(state: OverAllState):
    return {
        "logs": ["node_c"],
        "id": "node_c"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("node_c", node_c)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", "node_c")
builder.add_edge("node_c", END)

graph = builder.compile()
result = graph.invoke({"logs": ["START"], "id": "start"})
print('=' * 30, '-> result <-', '=' * 30)
print(result)
~~~

- 输出如下

~~~bash
{'logs': ['node_b', 'node_c'], 'id': 'node_c'}
~~~

- 上述代码中：
  - logs 字段绑定了 operator.add() 函数
    - 如果没有 Overwrite，则最终输出的 logs 的值应为 ['START', 'node_a', 'node_b', 'node_c']
    - node_b 返回更新时，用 Overwrite 包裹了 logs 字段的值，那么当前状态的 logs 会被 ["node_b"] 覆盖，因此最终输出的 logs 字段值变成了 ['node_b', 'node_c']
  - id 字段按照默认行为，保留最后一次更新的值



## 4、Multi Schema用法

### 4.1 状态类型

- LangGraph 支持在一个图中使用多个状态 Schema，用于区分图的外部输入、外部输出、内部共享状态以及节点间的临时状态。

- 常见状态类型可以分为以下几类：
  - <font color='red'>**全局状态 / 内部状态**</font>：图内部主要使用的状态，创建 StateGraph 时传递给 state_schema 参数。它通常包含图运行过程中需要读写的大部分字段。
  - <font color='red'>**输入状态**</font>：图对外接收输入时使用的状态，创建 StateGraph 时传递给 input_schema 参数。它用于约束调用图时允许传入哪些字段。
  - <font color='red'>**输出状态**</font>：图最终对外返回结果时使用的状态，创建 StateGraph 时传递给 output_schema 参数。它用于约束图运行结束后只返回哪些字段。
  - <font color='red'>**私有状态**</font>：图内部节点之间传递的临时状态，通常不作为图的输入，也不作为图的最终输出。它可以通过节点函数的入参类型注解声明，并在节点返回值中写入。

- 需要注意，输入状态和输出状态主要面向图的边界，即“图如何接收外部输入”和“图如何返回外部结果”；而全局状态和私有状态主要面向图内部节点之间的数据传递



### 4.2 状态之间的关系

#### 4.2.1 设计规范

- 本节主要说明 LangGraph 状态设计中的规范。以下规则属于工程上的最佳实践，违反这些规范未必一定导致程序报错，但容易降低代码的可读性和可维护性。

  - 1、<font color='red'>**输入状态和输出状态通常应是全局状态的子集**</font>，输入状态描述图对外需要接收的数据，输出状态描述图最终需要返回的数据。通常情况下，<font color='red'>**它们都应该是全局状态的一部分**</font>。 如：

    - 其中，InputState 和 OutputState 的所有字段均存在于 OverAllState 中

    ~~~python
    class InputState(TypedDict):
        username: str
    
    class OutputState(TypedDict):
        graph_output: str
    
    class OverAllState(TypedDict):
        username: str
        nickname: str
        graph_output: str
    ~~~

  - 2、<font color='red'>**私有状态和全局状态应尽量避免字段重名**</font>，私有状态的定位是图内部某些节点之间传递的临时字段。如果私有状态字段和全局状态字段重名，虽然某些情况下程序仍然可以运行，但容易让人误以为该字段是全局共享字段，从而造成理解混乱。 因此，推荐让私有状态字段和全局状态字段保持清晰边界

  - 3、<font color='red'>**节点函数应明确声明入参状态类型和返回状态类型**</font>，节点函数的第一个参数通常是当前节点可读取的状态。通过类型注解声明该参数，可以明确表达该节点需要读取哪些字段。 同时，给节点函数声明返回状态类型，也可以帮助阅读者理解该节点会更新哪些字段。 例如

  ~~~python
  def node_1(state: InputState) -> OverAllState:
      return {
          "nickname": "Dear " + state["username"]
      }
  ~~~

  - 4、<font color='red'>**节点函数中不应该访问入参状态类型中不存在的字段**</font>， 节点实际接收到的状态会按照其入参类型进行裁剪。因此，如果节点入参声明为 InputState，就不应该在节点内部访问 InputState 中不存在的字段。

    - 例如：

    ~~~python
    def node_1(state: InputState) -> OverAllState:
        return {
            "nickname": state["username"]
        }
    ~~~

    - 如果在该函数中访问：

    ~~~bash
    state["nickname"]
    ~~~

    - 而 nickname 不属于 InputState，运行时就可能抛出 KeyError

  - 5、<font color='red'>**节点函数返回的字典应尽量和返回类型注解保持一致**</font>， 从 Python 类型注解的角度看，函数返回类型只是静态提示，运行时不会自动强制校验。 从 LangGraph 的运行机制看，节点返回的是对状态的部分更新，不是完整状态。只要返回字段已经被图记录为可用状态字段，LangGraph 就可以将其作为状态更新处理。 不过，从工程规范上讲，节点返回字典中的字段最好和函数返回类型注解保持一致，这样更利于阅读、调试和维护



  #### 4.2.2 源码层面的约束

##### 4.2.2.1 状态的记录

- LangGraph 的状态并不是简单保存在一个普通字典中，而是会被拆分成多个可读写的状态字段。每个状态字段在底层通常对应一个 Channel。

- 这些状态字段会在不同阶段被记录到状态图中。
  - StateGraph 记录状态字段的核心方法是 _add_schema()，_add_schema() 会解析传入的状态 Schema，并将其中声明的字段记录到图中，使这些字段成为图运行时可以读写的状态字段。
  
    - <font color='red'>**创建 StateGraph 时，会记录 state_schema、input_schema 和 output_schema 中的字段**</font> 当创建状态图时：
  
    ~~~python
    builder = StateGraph(
        OverAllState,
        input_schema=InputState,
        output_schema=OutputState
    )
    ~~~
  
    - LangGraph 会解析这些 Schema，并将其中涉及的字段加入图的状态管理体系。
  - <font color='red'>**调用 add_node() 添加节点时，也可能记录节点入参声明的状态**</font> Schema 当添加节点时，LangGraph 会根据节点函数第一个参数的类型注解推断该节点的输入状态类型。 如果这个输入状态类型之前没有被图记录过，LangGraph 也会通过 _add_schema() 将其加入图中。 这也是私有状态能够生效的原因。
  
    -  例如：
  
    ~~~python
    class PrivateState(TypedDict):
        greeting: str
    
    def node_3(state: PrivateState) -> OutputState:
        return {
            "graph_output": state["greeting"]
        }
    ~~~
  
    - 当 node_3 被添加到图中时，PrivateState 中的 greeting 字段会被记录到图中，从而成为图内部可以传递的状态字段。
  
- 总结

  - <font color='red'>**全局状态、输入状态、输出状态通常在创建 StateGraph 时被记录**</font>
  - <font color='red'>**私有状态通常在调用 add_node() 添加节点时，根据节点入参类型注解被记录**</font>
  - <font color='red'>**被记录后的状态字段，底层会成为图运行时可以读写的状态字段**</font>



##### 4.2.2.2 状态的访问

- 1、调用图时，输入会按照 input_schema 进行约束 当调用图时：

  ```python
  graph.invoke({"username": "小黄"})
  ```

  - 如果创建图时声明了 input_schema，那么外部输入会按照 input_schema 进行约束。 如果没有声明 input_schema，则通常按照 state_schema 作为图的输入 Schema。 因此，input_schema 的作用不是“只让第一个节点可见”，<font color="red">**而是约束图的外部输入结构**</font>。 此处的约束是指：按照 schema 裁剪输入，只保留 schema 中出现的状态字段

- 2、节点接收到的状态会按照节点入参类型进行裁剪 每个节点能读取哪些字段，主要取决于该节点第一个参数的类型注解。 例如：

  ```python
  def node_1(state: InputState) -> OverAllState:
      ...
  ```

  - 此时，node_1 接收到的 state 会按照 InputState 进行裁剪。即使图的全局状态中还有其他字段，node_1 也不应该访问不属于 InputState 的字段。 如果访问了入参状态中不存在的字段，例如：

  ~~~python
  state["nickname"]
  ~~~

  - 就可能抛出：

  ```python
  KeyError
  ```

- 3、节点返回的是状态更新，而不是完整状态 节点函数不需要返回完整状态，只需要返回本节点想要更新的字段。 例如：

  ```python
  def node_1(state: InputState) -> OverAllState:
      return {
          "nickname": "Dear " + state["username"]
      }
  4
  ```

  - 这里虽然返回类型注解是 OverAllState，但函数实际只返回了 nickname 一个字段。这是允许的，因为 LangGraph 会把节点返回值视为对状态的部分更新

- 4、节点返回值的应用主要由字段名称和图中已记录的状态字段决定 节点返回的字典会根据字段名称写入对应状态字段，并按照该字段的 Reducer 规则进行合并。 需要注意的是，函数返回类型注解主要用于表达代码意图，不是严格的运行时写入边界。 也就是说，如果某个字段已经被图记录为可用状态字段，那么节点即使没有在返回类型注解中声明该字段，也可能仍然可以返回并更新它。 不过，为了代码清晰，仍然推荐让节点的返回值和返回类型注解保持一致。

- 5、最终输出会按照 output_schema 进行裁剪 图运行完成后，最终返回给外部调用方的结果会按照 output_schema 进行裁剪。 因此，output_schema 的作用不是“只让最后一个节点可见”，<font color="red">**而是约束图最终对外暴露哪些字段**</font>。 例如，图内部状态中可能同时存在：

  ```python
  username
  nickname
  greeting
  graph_output
  ```

  - 但如果 output_schema 只包含：

  ```python
  graph_output
  ```

  - 那么最终 graph.invoke() 的返回结果就只会包含 graph_output。



##### 4.2.2.3 案例

~~~python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class InputState(TypedDict):
    username: str

class OutputState(TypedDict):
    graph_output: str

class OverAllState(TypedDict):
    nickname: str
    username: str
    graph_output: str

class PrivateState(TypedDict):
    greeting: str

def node_1(state: InputState) -> OverAllState:
    # 向全局状态写入数据
    return {
        "nickname": "Dear " + state["username"]
    }

def node_2(state: OverAllState) -> PrivateState:
    # 从全局状态读取数据，写入私有状态
    return {
        "greeting": state["nickname"] + ", 早上好~"
    }

def node_3(state: PrivateState) -> OutputState:
    # 从私有状态读取数据，写入输出状态
    return {
        "graph_output": state["greeting"] + " 很高兴认识你！"
    }

builder = StateGraph(OverAllState,input_schema=InputState,output_schema=OutputState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", "node_3")
builder.add_edge("node_3", END)

graph = builder.compile()
print(graph.invoke({"username":"小黄"}))
~~~

- 输出

~~~bash
{ "graph_output": "Dear 小黄, 早上好~ 很高兴认识你！" }
~~~



##### 4.2.2.4 案例分析

- 输入状态

  - <font color="red">**InputState 用于约束图的外部输入**</font>

  ~~~python
  class InputState(TypedDict):
      username: str
  ~~~

  - 因此调用图时，只需要传入：

  ~~~python
  {"username": "小黄"}
  ~~~

- 全局状态

  - <font color="red">**OverAllState 是图内部主要使用的状态 Schema**</font>

  ~~~python
  class OverAllState(TypedDict):
      username: str
      nickname: str
      graph_output: str
  ~~~

  - 其中：
    - username 来自图的输入；
    - nickname 由 node_1 写入；
    - graph_output 由 node_3 写入，并最终作为图的输出返回

- 私有状态

  - <font color="red">**PrivateState 用于节点之间传递临时数据**</font>

  ~~~python
  class PrivateState(TypedDict):
      greeting: str
  ~~~

  - 在本例中：node_2 写入了 greeting 字段

  ~~~python
  def node_2(state: OverAllState) -> PrivateState:
      return {
          "greeting": state["nickname"] + ", 早上好~"
      }
  ~~~

  - 随后：node_3 通过 PrivateState 读取 greeting 字段，并生成最终输出

  ~~~python
  def node_3(state: PrivateState) -> OutputState:
      return {
          "graph_output": state["greeting"] + " 很高兴认识你！"
      }
  ~~~

- 输出状态

  - <font color="red">**OutputState 用于约束图最终返回给外部调用方的数据**</font>

  ~~~python
  class OutputState(TypedDict):
      graph_output: str
  ~~~

  - 虽然图内部运行过程中还存在 username、nickname、greeting 等字段，但最终结果只返回

  ~~~python
  { "graph_output": "Dear 小黄, 早上好~ 很高兴认识你！" }
  ~~~

  - 这是因为图创建时声明了

  ~~~python
  output_schema=OutputState
  ~~~

  - 所以最终输出会按照 OutputState 进行裁剪



## 5、预定义状态

### 5.1 MessagesState

- LangGraph 构建的计算图通常会和 LLM 结合使用，而 LLM 在运行过程中通常需要维护一组消息列表。为了提升开发效率，<font color="red">**LangGraph 官方提供了一个预定义状态类型：langgraph.graph.message.MessagesState。**</font>

- 开发者可以直接继承该状态类型，并在其基础上扩展自定义状态字段。

- 源码如下：

```python
class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

- 由此可知，MessagesState 只有一个字段：messages。

- 该字段的类型是列表，元素类型为 AnyMessage；同时，它通过 Annotated 绑定了内置 Reducer 函数 add_messages。
  - add_messages 的完全限定名（英文全称 fully qualified name）
  - langgraph.graph.message.add_messages，正是上文介绍的内置 Reducer 函数

~~~python
import os

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)

API_KEY=os.getenv("API_KEY")
BASE_URL=os.getenv("BASE_URL")
AI_MODEL=os.getenv("AI_MODEL")

model = init_chat_model(
    base_url=BASE_URL,
    model_provider="openai",
    api_key=API_KEY,
    model=AI_MODEL
)

class OverAllState(MessagesState):
    username: str
    output: str

def node_a(state: OverAllState) -> OverAllState:
    return {
        "messages": [HumanMessage("你好，我是 " + state["username"])]
    }

def llm_node(state: OverAllState) -> OverAllState:
    res = model.invoke(state["messages"])

    return {
        "messages": [res],
        "output": res.content
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("llm_node", llm_node)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "llm_node")
builder.add_edge("llm_node", END)

graph = builder.compile()
response = graph.invoke({"username": "小黄"})
print(response)
~~~

- 输出

~~~json
{
    "messages": [
        HumanMessage(
            content="你好，我是 小黄",
            additional_kwargs={},
            response_metadata={},
            id="155e1ef4-5bbc-4250-b978-62a7a5918cef",
        ),
        AIMessage(
            content="你好呀，小黄！😊 我是DeepSeek，很高兴认识你！有什么我可以帮你的吗？无论是聊天、解答问题、帮你写作、编程，还是其他任何需要，尽管告诉我吧！你名字里的“黄”是哪个黄呀？😄",
            additional_kwargs={
                "refusal": "None",
            },
            response_metadata={
                "token_usage": {
                    "completion_tokens": 57,
                    "prompt_tokens": 10,
                    "total_tokens": 67,
                    "completion_tokens_details": "None",
                    "prompt_tokens_details": {
                        "audio_tokens": "None",
                        "cached_tokens": 0,
                    },
                    "prompt_cache_hit_tokens": 0,
                    "prompt_cache_miss_tokens": 10,
                },
                "model_provider": "deepseek",
                "model_name": "deepseek-v4-flash",
                "system_fingerprint": "fp_8b330d02d0_prod0820_fp8_kvcache_20260402",
                "id": "17d57633-2ffb-4af3-be87-b3d58f9acd2b",
                "finish_reason": "stop",
                "logprobs": "None",
            },
            id="lc_run--019e6890-4544-7212-b4f2-aa20fc079911-0",
            tool_calls=[],
            invalid_tool_calls=[],
            usage_metadata={
                "input_tokens": 10,
                "output_tokens": 57,
                "total_tokens": 67,
                "input_token_details": {
                    "cache_read": 0,
                },
                "output_token_details": {},
            },
        ),
    ],
    "username": "小黄",
    "output": "你好呀，小黄！😊 我是DeepSeek，很高兴认识你！有什么我可以帮你的吗？无论是聊天、解答问题、帮你写作、编程，还是其他任何需要，尽管告诉我吧！你名字里的“黄”是哪个黄呀？😄",
}
~~~

- 上述案例中，OverAllState 继承了 MessagesState，因此可用的状态字段为

  ```python
  messages
  username
  output
  ```

  - 只关注 messages 状态，执行流程如下：

    - node_a 返回一条 HumanMessage；

    - LangGraph 使用 add_messages 将该消息合并到 messages 状态字段中；

    - llm_node 从 state["messages"] 中读取完整消息列表，并调用模型；

    - llm_node 将模型生成的 AIMessage 作为状态更新返回；

    - LangGraph 再次通过 add_messages 将 AIMessage 合并到 messages 中；

    - 最终状态中包含完整消息列表。

  - <font color='red'>**MessagesState 帮开发者预先定义好了 messages 字段及其合并规则**</font>。在构建聊天机器人、Agent、工具调用流程、多轮对话流程时，它可以提升开发效率。



### 5.2 AgentState

- AgentState 是 LangChain Agent 内部使用的状态类型。由于 LangChain Agent 底层也是基于 LangGraph 运行图构建的，所以从技术上讲，开发者也可以将 AgentState 或其子类作为自定义 LangGraph 的状态类型。
- AgentState 的全类名，也可以称为类的完全限定名（英文全称： fully qualified class name）是：langchain.agents.middleware.types.AgentState

- 源码如下

```python
class AgentState(TypedDict, Generic[ResponseT]):
    """State schema for the agent."""

    messages: Required[Annotated[list[AnyMessage], add_messages]]
    jump_to: NotRequired[Annotated[JumpTo | None, EphemeralValue, PrivateStateAttr]]
    structured_response: NotRequired[Annotated[ResponseT, OmitFromInput]]
```

- 该状态中主要包含三个字段

  - <font color='red'>**messages：用于存储 Agent 运行过程中的消息列表**</font>

    ~~~python
    messages: Required[Annotated[list[AnyMessage], add_messages]]
    ~~~

    - 字段和 MessagesState 中的 messages 字段类似，也使用 add_messages 作为 Reducer

  - <font color='red'>**jump_to：LangChain Agent 内部使用的控制字段，主要服务于 Agent 中间件体系**</font>

    ~~~python
    jump_to: NotRequired[Annotated[JumpTo | None, EphemeralValue, PrivateStateAttr]]
    ~~~

    - 它通常用于表示运行流程的跳转意图，例如某些中间件希望影响 Agent 后续应该进入哪个节点。
    - 需要注意的是，jump_to 并不是普通 LangGraph 状态图中的通用跳转机制。
    - 在自定义 StateGraph 中，即使状态中定义了 jump_to 字段，LangGraph 也不会因为该字段的值自动跳转到某个节点。普通 LangGraph 运行图如果需要控制后续流向，通常应使用

    ~~~python
    Command(goto="node_name")
    ~~~

  - <font color='red'>**structured_response：用于存储 Agent 最终生成的结构化输出**</font>

    ~~~python
    structured_response: NotRequired[Annotated[ResponseT, OmitFromInput]]
    ~~~

    - 当使用 LangChain Agent 的结构化输出能力时，例如指定 response_format，Agent 最终生成的结构化结果通常会被写入该字段。
    - 其中，OmitFromInput 表示该字段不应作为外部输入字段暴露给调用方，而是由 Agent 运行过程中内部生成。
    - 总体来看，AgentState 是专门为 LangChain Agent 运行时设计的状态类型。
    - 因此，在普通自定义 LangGraph 项目中，一般不建议直接基于 AgentState 扩展图状态



# 四、控制流

## 1、顺序结构

### 1.1 add_edge

- <font color='red'>**add_edge 用于在两个节点之间添加一条有向边**</font>。边是图结构中最基本的元素之一，节点之间的执行顺序、分支跳转以及循环控制，最终都依赖节点和边共同表达。

- 因此，在 LangGraph 中，<font color='red'>**基础的控制流结构都可以通过 add_edge进行构建**</font>

~~~python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class OverAllState(TypedDict):
    username: str
    greeting: str
    output: str

def node_a(state: OverAllState) -> OverAllState:
    return {
        "greeting": "Dear " + state["username"]
    }

def node_b(state: OverAllState) -> OverAllState:
    return {
        "output": state["greeting"] + "，你好！"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)

graph = builder.compile()
res = graph.invoke({"username": "小黄"})
print(res)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- 输出

~~~python
{'username': '小黄', 'greeting': 'Dear 小黄', 'output': 'Dear 小黄，你好！'}
~~~

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_a(node_a)
	node_b(node_b)
	__end__([<p>__end__</p>]):::last
	__start__ --> node_a;
	node_a --> node_b;
	node_b --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 在上述示例中，图的执行流程如下：

  ```python
  START -> node_a -> node_b -> END
  ```

  - 其中：

    - **START -> node_a** 表示图从 node_a 开始执行；

    - **node_a -> node_b** 表示 node_a 执行完成后，继续执行 node_b；

    - **node_b -> END** 表示 node_b 执行完成后，图进入结束状态



### 1.2 add_sequence

- <font color='red'>**如果需要构建一组按顺序执行的节点**</font>，也可以使用 **add_sequence**。

- **add_sequence** 支持传入一个可执行对象列表。LangGraph 会按照列表顺序依次添加节点，并在相邻节点之间自动添加边。默认情况下，函数名会被用作节点名称。

- **示例如下：**

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class OverAllState(TypedDict):
    username: str
    greeting: str
    output: str

def node_a(state: OverAllState) -> OverAllState:
    return {
        "greeting": "Dear " + state["username"]
    }

def node_b(state: OverAllState) -> OverAllState:
    return {
        "output": state["greeting"] + "，你好！"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_edge(START, "node_a")
builder.add_sequence([node_a, node_b])
builder.add_edge("node_b", END)

graph = builder.compile()
res = graph.invoke({"username": "小黄"})
print(res)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
```

- 输出如下

```json
{
  "username": "小黄",
  "greeting": "Dear 小黄",
  "output": "Dear 小黄，你好！"
}
```

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_a(node_a)
	node_b(node_b)
	__end__([<p>__end__</p>]):::last
	__start__ --> node_a;
	node_a --> node_b;
	node_b --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 上述代码中：

```python
builder.add_sequence([node_a, node_b])
```

- 大致等价于：

```python
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge("node_a", "node_b")
```

- 因此，完整执行流程仍然是：

```python
START -> node_a -> node_b -> END
```

- 相比手动调用多次 **add_node** 和 **add_edge**，**add_sequence** 更适合用于构建简单的线性执行流程。



### 1.3 省略指向END的边

- 在 LangGraph 中，<font color='red'>**图的终止并不完全依赖某个真实执行的特殊节点**</font>

- 从运行机制上看，LangGraph 会在每个 **SuperStep** 开始时，根据当前发生更新的 **Channel** 以及节点之间的触发关系，计算本轮需要执行的任务列表。<font color='red'>**如果没有新的节点被激活，也就没有新的任务需要执行，图运行自然结束**</font>

- 需要注意的是，<font color='red'>**END 并不是运行阶段真正执行的节点**</font>。也就是说，指向 **END** 的边不会像指向普通节点的边那样，触发一个真实的节点任务。它更多用于表达图结构中的“终止语义”：当前路径执行到这里即可结束。

- 因此，在一些简单的线性流程中，即使省略指向 **END** 的边，最后一个节点执行完成后，如果没有后续节点被触发，图也可以正常结束。

- 示例如下：

```python
from langgraph.graph import StateGraph, START
from typing import TypedDict

class OverAllState(TypedDict):
    username: str
    greeting: str
    output: str

def node_a(state: OverAllState) -> OverAllState:
    return {
        "greeting": "Dear " + state["username"]
    }

def node_b(state: OverAllState) -> OverAllState:
    return {
        "output": state["greeting"] + "，你好！"
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")

graph = builder.compile()
res = graph.invoke({"username": "小黄"})
print(res)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
```

- 输出如下

```json
{
  "username": "小黄",
  "greeting": "Dear 小黄",
  "output": "Dear 小黄，你好！"
}
```

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_a(node_a)
	node_b(node_b)
	__end__([<p>__end__</p>]):::last
	__start__ --> node_a;
	node_a --> node_b;
	node_b --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 在这个例子中，虽然没有显式添加：

  ~~~python
  builder.add_edge("node_b", END)
  ~~~

  - 但是 **node_b** 执行完成后，没有新的节点被触发，图运行会自然结束。

  - 不过，<font color='red'>**可以省略指向 END 的边，并不表示 END 没有意义**</font>

  - 显式添加 **END** 边可以让图结构更加完整，也能更清晰地表达“流程到此结束”的语义。尤其是在分支、条件跳转、循环退出等场景中，显式指向 **END** 通常更利于阅读和维护。

  - 和 **END** 不同，<font color='red'>**START 通常不能省略**</font>。因为 **START** 不只是一个语义上的起点标记，它还用于告诉 LangGraph：图运行时应当从哪些节点开始执行。也就是说，下面这条边是启动图执行的关键：

  ~~~python
  builder.add_edge(START, "node_a")
  ~~~

  - 如果没有从 **START** 出发的边，或者没有通过其他方式指定入口节点，LangGraph 就无法确定图的初始执行节点。

- 因此，在实际开发中，建议保留指向 **END** 的边。虽然在某些简单流程中省略这些边也能正常运行，但显式添加它们，可以让图结构更加完整、语义更加清晰，便于后续维护。



## 2、分支结构

### 2.1 静态分布

- **定义**：节点的下游候选节点 <font color='red'>**在图编译阶段就完全确定**</font>，只是运行时根据条件选择哪条边执行。
- 特点：
  - 下游节点集合固定，数量、目标在编译时确定
  - 运行时可选择**一个或多个**下游目标
  - 可以用来做条件分支，但不生成新的节点

⚡ 核心判断：

> **编译期知道下游集合 → 静态分支**



#### 2.1.1 并行节点

- 并行节点是最简单的静态分支形式。

- 当多个节点都从同一个上游节点触发时，它们会在同一个**超步**被激活。典型写法如下：

~~~python
builder.add_edge(START, "node_a")
builder.add_edge(START, "node_b")
~~~

- 示例

~~~python
import os
from typing import TypedDict

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

class OverAllState(TypedDict):
    topic: str
    poem: str
    joke: str

def node_a(state: OverAllState) -> OverAllState:
    poem = model.invoke(
        [
            HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")
        ]
    ).content
    return {
        "poem": poem
    }

def node_b(state: OverAllState) -> OverAllState:
    joke = model.invoke(
        [
            HumanMessage(f"写一个关于 {state['topic']} 的笑话")
        ]
    ).content
    return {
        "joke": joke
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge(START, "node_b")

builder.add_edge("node_a", END)
builder.add_edge("node_b", END)

graph = builder.compile()
res = graph.invoke({"topic": "猫咪"})
print(res)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- 输出：

~~~json
{
  "topic": "猫咪",
  "poem": "《猫》\n花间小憩一身绒，\n忽逐飞虫入院中。\n夜半双眸明似月，\n轻身跳上屋檐东。",
  "joke": "我家猫最近总盯着我看，我以为它终于要表达爱意了。结果它伸出爪子，把我的水杯慢慢推到桌边，然后一脸平静地看着我。\n\n我赶紧接住水杯，它叹了口气，又推了一次。\n\n我问：“你到底想干嘛？”\n\n它说：“我在测试你。结果发现，你比我想的更适合当仆人。”"
}
~~~

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_a(node_a)
	node_b(node_b)
	__end__([<p>__end__</p>]):::last
	__start__ --> node_a;
	__start__ --> node_b;
	node_a --> __end__;
	node_b --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 上述案例中，**node_a** 和 **node_b** 都由 **START** 触发。图运行时，二者会在**同一个超步中**被调度，它们各自读取当前状态并独立执行。
- 需要注意的是：
  - 这里的“并行”主要指**调度语义上的并行**；两个节点之间没有先后依赖；
  - 它们的输出会在当前**超步**执行完成后统一合并到状态中；
  - 如果两个节点写入同一个状态字段，则该字段通常需要配置合适的 **Reducer**，否则可能出现状态更新冲突，抛出 **InvalidUpdateError** 异常



#### 2.1.2 条件分支

- **StateGraph** 提供了 **add_conditional_edges** 方法，用于从某个上游节点出发，根据运行时状态选择下游节点。

- 方法签名如下

```python
def add_conditional_edges(
    self,
    source: str,
    path: Callable[..., Hashable | Sequence[Hashable]]
    | Callable[..., Awaitable[Hashable | Sequence[Hashable]]]
    | Runnable[Any, Hashable | Sequence[Hashable]],
    path_map: dict[Hashable, str] | list[str] | None = None,
) -> Self:
```

- 不考虑 **self**，核心参数有三个：
  - <font color='red'>**source**</font>：条件分支的起始节点；
  - <font color='red'>**path**</font>：路由规则，是一个可执行对象，通常是函数
  - <font color='red'>**path_map**</font>：路由规则的返回值到真实节点名之间的映射关系。

- 其中，**path** 的返回值表示跳转的目标节点，可以是：
  - 字符串或特殊对象 **END** 表示的单个目标；
  - 字符串或特殊对象 **END** 表示的多个目标组成的序列；

- **path_map**

  - 可以省略，即取默认值 **None**，此时 **path** 返回值中出现的字符串必须是合法的节点名称。
  - 可以是字典，维护 **path** 返回值和真实节点的映射。
  - 也可以是列表，如下

  ~~~python
  path_map=["node_a", "node_b", "node_c"]
  ~~~

  - 相当于

  ```python
  path_map={
      "node_a": "node_a",
      "node_b": "node_b",
      "node_c": "node_c",
  }
  ```

  - 此时也要求 **path** 返回值中出现的字符串必须是合法的节点名称



##### 2.1.2.1 不使用path_map

- 如果不传 **path_map**，那么路由函数的返回值通常应当直接是图中的节点名称。如下

```python
def router(state: OverAllState) -> Literal["node_a", "node_b"]: 
    if "诗" in state["content_type"]: 
        return "node_a" 
    return "node_b"
```

- 此时，**router** 返回的 **"node_a"** 和 **"node_b"** 必须能够直接对应图中已经注册的节点名

~~~python
import os
from typing import TypedDict, Literal

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

class OverAllState(TypedDict):
    topic: str
    content_type: str
    poem: str
    joke: str

def node_a(state: OverAllState) -> OverAllState:
    poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")]).content

    return {
        "poem": poem
    }

def node_b(state: OverAllState) -> OverAllState:
    joke = model.invoke([HumanMessage(f"写一个关于 {state['topic']} 的笑话")]).content

    return {
        "joke": joke
    }

def router(state: OverAllState) -> Literal["node_a", "node_b"]:
    if "诗" in state["content_type"]:
        return "node_a"
    return "node_b"

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_conditional_edges(START, router)
builder.add_edge("node_a", END)
builder.add_edge("node_b", END)

graph = builder.compile()
poem_res = graph.invoke({"topic": "布偶狗", "content_type": "诗"})
joke_res = graph.invoke({"topic": "布偶狗", "content_type": "笑话"})

print('=' * 30, '-> poem_res <-', '=' * 30)
print(poem_res)
print('=' * 30, '-> joke_res <-', '=' * 30)
print(joke_res)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- 输出

~~~text
============================== -> poem_res <- ==============================
{
  "topic": "布偶狗",
  "content_type": "诗",
  "poem": "《布偶狗》\n绒身软耳坐窗台，不吠不奔待主来。\n夜半犹陪灯下影，忠心一片布中裁。"
}
============================== -> joke_res <- ==============================
{
  "topic": "布偶狗",
  "content_type": "笑话",
  "joke": "“你家布偶狗会握手吗？”\n“会，特别标准。”\n“这么厉害？”\n“是啊，就是每次握完，我都得拿针线把它的爪子缝回去。”"
}
~~~

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_a(node_a)
	node_b(node_b)
	__end__([<p>__end__</p>]):::last
	__start__ -.-> node_a;
	__start__ -.-> node_b;
	node_a --> __end__;
	node_b --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 观察图结构可以发现，<font color='red'>**start指向node_a和node_b的线是虚线，这表示节点的跳转是有条件的，不是固定的**</font>




##### 2.1.2.2 使用path_map

- 如果不希望路由函数直接返回节点名，而是返回业务语义更强的标识，可以使用 **path_map** 进行映射。如下：

```python
def router(state: OverAllState) -> Literal["a", "b"]: 
    if "诗" in state["content_type"]: 
        return "a" 
    return "b"

builder.add_conditional_edges( 
    START,
    router,
    path_map={ 
        "a": "node_a", 
        "b": "node_b", 
    } 
)
```

- 此时返回的 **"a"** 和 **"b"** 可以不是图中已注册的节点名称，但要通过 **path_map** 映射到正确的节点

- 这种写法的好处是：
  - 路由函数可以返回业务含义更清晰的标签；
  - 图节点名称可以保持工程化命名；
  - 渲染图结构时，边上可以显示路由标签，使图更容易理解。

- **示例如下**

~~~python
import os
from typing import TypedDict, Literal

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

class OverAllState(TypedDict):
    topic: str
    content_type: str
    poem: str
    ci_poem: str
    joke: str

def node_a(state: OverAllState) -> OverAllState:
    poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")]).content

    return {
        "poem": poem
    }

def node_b(state: OverAllState) -> OverAllState:
    joke = model.invoke([HumanMessage(f"写一个关于 {state['topic']} 的笑话")]).content

    return {
        "joke": joke
    }

def router(state: OverAllState) -> Literal["a", "b"]:
    if "诗" in state["content_type"]:
        return "a"
    return "b"

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_conditional_edges(
    START,
    router,
    path_map={
        "a": "node_a",
        "b": "node_b",
    }
)
builder.add_edge("node_a", END)
builder.add_edge("node_b", END)

graph = builder.compile()
poem_res = graph.invoke({"topic": "布偶狗", "content_type": "诗"})
joke_res = graph.invoke({"topic": "布偶狗", "content_type": "笑话"})

print('=' * 30, '-> poem_res <-', '=' * 30)
print(poem_res)
print('=' * 30, '-> joke_res <-', '=' * 30)
print(joke_res)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- 输出

~~~text
============================== -> poem_res <- ==============================
{
  "topic": "布偶狗",
  "content_type": "诗",
  "poem": "《布偶狗》绒身软耳坐窗台，不吠不奔待主来。夜半犹陪灯下影，忠心一片布中裁。"
}
============================== -> joke_res <- ==============================
{
  "topic": "布偶狗",
  "content_type": "笑话",
  "joke": "我家布偶狗特别会看家。昨天小偷一进门，它就冲了上去——“咬他了？”“没有，它从沙发上滚下来，躺平装死。小偷吓得当场跪下：‘大哥，我还没动手呢，你别碰瓷啊！’”"
}
~~~

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_a(node_a)
	node_b(node_b)
	__end__([<p>__end__</p>]):::last
	__start__ -. a .-> node_a;
	__start__ -. b .-> node_b;
	node_a --> __end__;
	node_b --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 观察图结构可以发现，__start__指向node_a和node_b的虚线上出现了文本a和b，它们是路由函数返回的名称，通过path_map映射到具体的下游节点




##### 2.1.2.3 同时路由至多个节点

- **add_conditional_edges** 也支持一次路由到多个下游节点。

- <font color='red'>**不用path_map**</font>

  ~~~python
  import os
  from typing import TypedDict, Literal, Sequence
  
  from langchain.chat_models import init_chat_model
  from langgraph.graph import StateGraph, START, END
  from langchain.messages import HumanMessage
  
  from dotenv import load_dotenv
  load_dotenv(override=True)
  
  CHAT_API_KEY=os.getenv("CHAT_API_KEY")
  CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
  CHAT_MODEL=os.getenv("CHAT_MODEL")
  
  model = init_chat_model(
      base_url=CHAT_BASE_URL,
      model_provider="openai",
      api_key=CHAT_API_KEY,
      model=CHAT_MODEL
  )
  
  class OverAllState(TypedDict):
      topic: str
      content_type: str
      poem: str
      ci_poem: str
      joke: str
  
  def node_a(state: OverAllState) -> OverAllState:
      poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")]).content
  
      return {
          "poem": poem
      }
  
  def node_b(state: OverAllState) -> OverAllState:
      joke = model.invoke([HumanMessage(f"写一个关于 {state['topic']} 的笑话")]).content
  
      return {
          "joke": joke
      }
  
  def node_c(state: OverAllState) -> OverAllState:
      ci_poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的词")]).content
  
      return {
          "ci_poem": ci_poem
      }
  
  def router(state: OverAllState) -> Sequence[Literal["node_a", "node_b", "node_c"]]:
      if "诗" in state["content_type"]:
          return ["node_a", "node_c"]
      return ["node_b", "node_c"]
  
  builder = StateGraph(state_schema=OverAllState)
  builder.add_node("node_a", node_a)
  builder.add_node("node_b", node_b)
  builder.add_node("node_c", node_c)
  builder.add_conditional_edges(
      START,
      router
  )
  builder.add_edge("node_a", END)
  builder.add_edge("node_b", END)
  builder.add_edge("node_c", END)
  
  graph = builder.compile()
  poem_res = graph.invoke({"topic": "布偶狗", "content_type": "诗"})
  joke_res = graph.invoke({"topic": "布偶狗", "content_type": "笑话"})
  
  print('=' * 30, '-> poem_res <-', '=' * 30)
  print(poem_res)
  print('=' * 30, '-> joke_res <-', '=' * 30)
  print(joke_res)
  
  raw_mermaid = graph.get_graph().draw_mermaid()
  print(raw_mermaid)
  ~~~

  - 输出

  ~~~python
  ============================== -> poem_res <- ==============================
  {
      "topic": "布偶狗",
      "content_type": "诗",
      "poem": "《咏布偶狗》\n铁骨棉裘本异俦，垂头摇尾学温柔。\n非关皮相失真性，恐向人间惹旧愁。\n\n赏析：这首作品以布偶狗为题，通过“铁骨棉裘”的悖论意象，巧妙揭示材质与形态的错位。后两句笔锋一转，从拟态之“学温柔”深入至本真之“失真性”，结句“恐向人间惹旧愁”更以物喻人，赋予玩偶以灵性，暗含对世情虚伪的讽喻，余韵悠长。",
      "ci_poem": "《捣练子·布偶狗》\n针线巧，布绒柔。静卧妆台伴月幽。未解铃铛摇夜语，却随清梦上云舟。\n\n注：我的仿作将布偶狗拟人化，通过“针线巧，布绒柔”展现其手工质感，“伴月幽”营造静谧氛围。末句“随清梦上云舟”暗合现代人渴望逃离现实的隐喻，既保留古典词牌的意境美，又赋予布偶狗超越物象的情感寄托。"
  }
  
  ============================== -> joke_res <- ==============================
  {
      "topic": "布偶狗",
      "content_type": "笑话",
      "ci_poem": "《鹧鸪天·布偶狗》\n绒目含星尾作虹，垂蹄静卧笑春风。\n无人自戏绒球转，得主频摇铃铎浓。\n\n偎暖日，扑帘栊。娇憨总在爪痕中。\n痴心不解浮生事，一抱虚温万事空。\n\n注：本词以布偶狗为意象，通过“绒目含星”、“垂蹄静卧”等细节摹写其柔顺之态。下阕“偎暖日”、“扑帘栊”拟人化笔法，寄寓物我相忘之趣。结句“一抱虚温”暗喻繁华终归寂灭之理，将玩偶之趣升华为对生命本真的哲思。",
      "joke": "小明带他的布偶狗去宠物医院，医生检查后说：“你这狗没心跳没呼吸，已经死了。”\n小明急了：“不可能！它刚刚还对我摇尾巴！”\n医生叹了口气：“那是你兜里的钥匙串，走起路来叮当响，它尾巴上的铃铛在共振。”\n小明沉默片刻，掏出手机拍了张照发朋友圈：“我的狗死了，但它的尾巴还在听我走路。”\n评论区炸了：“建议主人也去查查脑子，可能也共振坏了。”"
  }
  ~~~

  ~~~mermaid
  graph TD;
  	__start__(<p>__start__</p>)
  	node_a(node_a)
  	node_b(node_b)
  	node_c(node_c)
  	__end__(<p>__end__</p>)
  	__start__ -.-> __end__;
  	classDef default fill:#f2f0ff,line-height:1.2
  	classDef first fill-opacity:0
  	classDef last fill:#bfb6fc
  ~~~

  - 在这个例子中：
    - 当 **content_type** 包含 **"诗"** 时，同时触发 **node_a** 和 **node_c**；
    - 否则同时触发 **node_b** 和 **node_c**
  - 观察图结构可以发现，node_a、node_b和node_c独立于图结构之外。
  - 和上一节案例相比，router函数返回的是序列而非单个节点，渲染器无法推断节点间的映射关系

- <font color='red'>**使用path_map**</font>
  - 通过 **path_map** 显示声明映射关系，明确下游节点集合，在提升代码可读性的同时，也有助于渲染器正确展示条件边。

  - 当前场景下 **path** 返回值中的字符串就是合法的节点名称，**path_map** 可以是字典

    ```python
    path_map={
        "node_a": "node_a",
        "node_b": "node_b",
        "node_c": "node_c",
    }
    ```

    - 也可以是列表

    ```python
    path_map=["node_a", "node_b", "node_c"]
    ```

    - **完整案例如下**

    ```python
    import os
    from typing import TypedDict, Literal, Sequence
    
    from langchain.chat_models import init_chat_model
    from langgraph.graph import StateGraph, START, END
    from langchain.messages import HumanMessage
    
    from dotenv import load_dotenv
    load_dotenv(override=True)
    
    CHAT_API_KEY=os.getenv("CHAT_API_KEY")
    CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
    CHAT_MODEL=os.getenv("CHAT_MODEL")
    
    model = init_chat_model(
        base_url=CHAT_BASE_URL,
        model_provider="openai",
        api_key=CHAT_API_KEY,
        model=CHAT_MODEL
    )
    
    class OverAllState(TypedDict):
        topic: str
        content_type: str
        poem: str
        ci_poem: str
        joke: str
    
    def node_a(state: OverAllState) -> OverAllState:
        poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")]).content
    
        return {
            "poem": poem
        }
    
    def node_b(state: OverAllState) -> OverAllState:
        joke = model.invoke([HumanMessage(f"写一个关于 {state['topic']} 的笑话")]).content
    
        return {
            "joke": joke
        }
    
    def node_c(state: OverAllState) -> OverAllState:
        ci_poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的词")]).content
    
        return {
            "ci_poem": ci_poem
        }
    
    def router(state: OverAllState) -> Sequence[Literal["node_a", "node_b", "node_c"]]:
        if "诗" in state["content_type"]:
            return ["node_a", "node_c"]
        return ["node_b", "node_c"]
    
    builder = StateGraph(state_schema=OverAllState)
    builder.add_node("node_a", node_a)
    builder.add_node("node_b", node_b)
    builder.add_node("node_c", node_c)
    builder.add_conditional_edges(
        START,
        router,
        path_map={
            "node_a": "node_a",
            "node_b": "node_b",
            "node_c": "node_c",
        }
        # path_map=["node_a", "node_b", "node_c"]
    )
    builder.add_edge("node_a", END)
    builder.add_edge("node_b", END)
    builder.add_edge("node_c", END)
    
    graph = builder.compile()
    poem_res = graph.invoke({"topic": "布偶狗", "content_type": "诗"})
    joke_res = graph.invoke({"topic": "布偶狗", "content_type": "笑话"})
    
    print('=' * 30, '-> poem_res <-', '=' * 30)
    print(poem_res)
    print('=' * 30, '-> joke_res <-', '=' * 30)
    print(joke_res)
    
    raw_mermaid = graph.get_graph().draw_mermaid()
    print(raw_mermaid)
    ```

    - **输出如下**

    ```python
    ============================== -> poem_res <- ==============================
    
    {
        "topic": "布偶狗",
        "content_type": "诗",
        "poem": "《布偶狗》\n布偶憨然卧锦茵，牵绳随主步香尘。\n偶因客至微昂首，原是无声座上宾。\n\n注：我的创作思路是抓住布偶狗静中有动的特质。首句以“憨然卧锦茵”勾勒其静态萌态，次句“牵绳随主步香尘”暗喻宠物与人的陪伴关系。转句“偶因客至微昂首”捕捉其反应，既保留布偶的拟真特性，又暗示机械性动作。结句“原是无声座上宾”点明其非生物属性，与开篇“布偶”形成闭环。全诗通过动静、真假的微妙转化，赋予静物以灵性。",
        "ci_poem": "《醉花阴·布偶狗》\n绒耳垂垂星作眸，憨卧小窗幽。棉絮裹憨态，线描眉眼，缝就三分柔。\n旧尘不染春衫袖，伴我度春秋。虽无温舌语，月斜人静，也解倚床头。\n\n注：我以拟人笔法赋予布偶狗灵性，通过“绒耳垂垂”“棉絮裹憨态”等细节勾勒其柔软形态，又于“月斜人静”处暗藏陪伴深情。下阕“旧尘不染”暗喻其纯净守护，末句“解倚床头”更将静物写活，道出玩偶与主人之间无声的慰藉。"
    }
    
    ============================== -> joke_res <- ==============================
    
    {
        "topic": "布偶狗",
        "content_type": "笑话",
        "ci_poem": "《南歌子·布偶狗》\n绒耳垂春水，圆睛印月痕。\n棉絮缝成憨态真。\n不向朱门摇尾，卧蓬门。\n\n旧线牵云袖，新绒补雪身。\n线头散作泪斑纹。\n但把残年缝进，掌中温。\n\n注：我的布偶狗，是外婆用旧棉裤的棉花和二姐穿小的灯芯绒外套一针一线缝成的。二十年过去，棉絮从它破了洞的耳朵里露出，像吐着舌头。我模仿陆游“我与狸奴不出门”的意境，通过“棉絮缝春水”喻指时光在针脚里流逝，而“线头泪斑”则是故意缝补的裂痕——我们总要在破碎的事物里，重新认出圆满的形状。",
        "joke": "好的，这是一个关于布偶狗的笑话：\n\n有一只布偶狗，它特别特别懒，懒到什么程度呢？它的主人叫它去捡飞盘，它都懒得动。主人决定训练它，就扔出一个飞盘，大声喊：“布偶，快去捡！”\n\n布偶狗懒洋洋地抬头看了一眼飞盘落在远处，然后又趴下了。主人很生气，又扔了一个，它还是不理。\n\n主人无奈，只好采取了终极手段——假扮成一只会动的布偶狗，自己冲过去把飞盘叼了回来。\n\n布偶狗一看，浑身毛发都惊得炸了起来，它对旁边的一只小花狗低声说：“天哪，你看，那玩意儿居然能自己动！它还是个永动机！吓死狗了，好赖哦！”"
    }
    ```

    ~~~mermaid
    graph TD;
    	__start__([<p>__start__</p>]):::first
    	node_a(node_a)
    	node_b(node_b)
    	node_c(node_c)
    	__end__([<p>__end__</p>]):::last
    	__start__ -.-> node_a;
    	__start__ -.-> node_b;
    	__start__ -.-> node_c;
    	node_a --> __end__;
    	node_b --> __end__;
    	node_c --> __end__;
    	classDef default fill:#f2f0ff,line-height:1.2
    	classDef first fill-opacity:0
    	classDef last fill:#bfb6fc
    ~~~

    - 如图所示，图结构被正确渲染。



#### 2.1.3 defer node execution

- 某些情况下，我们希望<font color='red'>**在所有常规任务节点执行完毕后，再进行日志、审计等收尾工作**</font>
- 此时可以在添加节点时设置defer=True，如下：

~~~python
builder.add_node("audit_node", audit_node, defer=True)
~~~

- **defer=True** 的含义是：

  - 当前节点不会在其被触发后立即执行，<font color='red'>**而是被延迟到常规图运行流程结束后，再在额外的超步中触发执行**</font>

- 这类节点适合用于：

  - 日志记录；

  - 审计检查；

  - 结果汇总；

  - 收尾清理；

  - 统一校验前面节点是否已完成



##### 2.1.3.1 底层实现机制

![Defer延迟触发底层机制](图片/Defer延迟触发底层机制.png)

- ###### 1. 编译阶段：使用特殊 Channel
  - **LangGraph** 在编译状态图时，会为边创建对应的 **Channel**。
  - 此时会根据节点的 **defer** 属性创建不同类型的 **Channel**，如下。

  ```python
  self.channels[branch_channel] = (
      LastValueAfterFinish(Any)
      if node.defer
      else EphemeralValue(Any, guard=False)
  )
  ```

  - **defer** 默认值为 **False**

  - 对于普通节点，边对应的通道类型是 **EphemeralValue**，可以理解为普通临时通道；而对于 **defer=True** 的节点，边对应的通道类型是特殊的 **LastValueAfterFinish**。

- ###### 2. 常规运行阶段-写入但不触发
  - 在图运行过程中，每个节点执行完成后，会向其下游边对应的 **Channel** 写入数据。
  - 常规的 **Channel** 在运行开始后处于可用状态，被写入后记录在 **updated_channels** 列表中，从而在下一个超步中触发下游节点的执行。
  - 但是，**LastValueAfterFinish** 类型的通道起初是不可用的，首次被写入时不会添加到 **updated_channels** 列表中，下游节点自然不会被触发。

- ###### 3. 常规流程结束后：调用 finish() 唤醒延迟节点
  - **LangGraph** 底层用 **trigger_to_nodes** 维护了 **边的 Channel -> 节点** 的映射，是一个字典。
  - 在每个超步结束后，运行时会根据 **updated_channels** 判断是否还有新的节点需要被触发。
  - 如果 **updated_channels** 和 **trigger_to_nodes** 的 key 没有交集，说明当前没有新的普通节点需要继续执行，常规运行流程已结束。
  - 此时，**LangGraph** 运行时会调用所有 **Channel** 的 **finish()** 方法。
  - 对于普通 **Channel** ，**finish()** 通常不会产生新的触发效果；但对于 **LastValueAfterFinish** 类型通道，首次调用 **finish()** 时，会将内部的 **finished** 标记设置为 **True**，并返回 **True**。
  - 一旦 **finished=True**，该通道的 **is_available()** 就会变为 **True**。
  - 于是，原本被延迟的通道会被加入 **updated_channels**，从而在额外的超步中触发对应的 **defer** 节点。

- ###### 4. 总结
  - 因此，**defer=True** 的运行机制可以概括为：
  - <font color='red'>触发边的 **Channel** 为特殊类型，首次写入不触发；常规流程结束后，特殊通道 **finish()**；通道变为可用；从而触发延迟节点，后者在额外超步中执行。</font>



##### 2.1.3.2 案例

~~~python
import os
from typing import TypedDict

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage
from loguru import logger

from dotenv import load_dotenv
load_dotenv(override=True)

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

class OverAllState(TypedDict):
    topic: str
    poem: str
    joke: str

def node_a(state: OverAllState) -> OverAllState:
    poem = model.invoke([HumanMessage(f"写一首关于 {state['topic']} 的七言绝句")]).content

    return {
        "poem": poem
    }

def node_b(state: OverAllState) -> OverAllState:
    joke = model.invoke([HumanMessage(f"写一个关于 {state['topic']} 的笑话")]).content

    return {
        "joke": joke
    }

def audit_node(state: OverAllState) -> OverAllState:
    logger.info(f"任务节点全部执行完毕，诗 {'已生成' if state['poem'] else '未生成'}，笑话 {'已生成' if state['joke'] else '未生成'}")

builder = StateGraph(state_schema=OverAllState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_node("audit_node", audit_node, defer=True)
builder.add_edge(START, "node_a")
builder.add_edge(START, "node_b")
builder.add_edge(START, "audit_node")
builder.add_edge("node_a", END)
builder.add_edge("node_b", END)
builder.add_edge("audit_node", END)

graph = builder.compile()
res = graph.invoke({"topic": "布偶狗"})
print(res)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- 在这个图中：
  - **node_a** 和 **node_b** 是普通节点，在常规流程中被触发；
  - **audit_node** 虽然也由 **START** 触发，但由于设置了 **defer=True**，不会立即执行；
  - **node_a** 和 **node_b** 都执行完成后，常规运行流程结束，**audit_node** 才会在额外的超步中执行；
  - 因此，**audit_node** 中可以读取到 **node_a** 和 **node_b** 已经写入并提交后的状态。

- 输出

~~~bash
{'topic': '布偶狗', 'poem': '《布偶狗》\n布偶裁成小犬形，\n软耳垂垂眼点星。\n不吠门前迎客至，\n长陪枕畔梦初醒。', 'joke': '宠物店。\n我：“老板，这布偶狗会拆家吗？”\n老板：“不会。”\n我：“会乱叫吗？”\n老板：“不会。”\n我：“那它会干嘛？”\n老板：“会掉地上。”\n我：“……？”\n老板：“提醒你一下，它是个抱枕，睡觉别把它踢下去。”'}
~~~

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_a(node_a)
	node_b(node_b)
	audit_node(audit_node<hr/><small><em>defer = True</em></small>)
	__end__([<p>__end__</p>]):::last
	__start__ --> audit_node;
	__start__ --> node_a;
	__start__ --> node_b;
	node_a -.-> audit_node;
	node_b -.-> audit_node;
	audit_node --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~





##### 2.1.3.3 小结

- <font color='red'>**defer=True 适合“最后执行”的收尾节点**</font>

- 它的本质是通过特殊的 **Channel** 控制触发时机：
  - 编译阶段为延迟节点创建 **LastValueAfterFinish** 类型通道；
  - 常规运行阶段该通道可以被写入，但不可用，不会触发下游节点；
  - 常规流程结束后调用通道的 **finish()** 方法；
  - **finish()** 首次生效后将通道标记为可用；
  - 延迟节点在额外超步中被触发执行。

> - 因此， **defer=True** 将节点的触发信号延迟释放，使该节点在常规流程结束后执行。



### 2.2 动态分支

#### 2.2.1 定义

- **定义**：节点的后续执行路径在 <font color='red'>**运行时**</font> 才确定，可以根据当前状态、输入数据或中间结果，动态决定要触发哪些下游任务或跳转到哪个下游节点。

- 特点：

  - 下游执行目标可以在运行时选择；
  - 下游任务数量可以在运行时决定；
  - 可以为同一个下游节点动态创建多个执行任务；
  - 适合实现 Map-Reduce 式动态扇出、多任务并行处理、运行时条件跳转等场景。

  需要注意的是，所谓“动态”通常不是指运行时临时创建新的节点定义。节点本身一般仍需要在图编译前注册。动态性主要体现在：<font color='red'>**运行时决定触发哪些节点，以及为这些节点创建多少个执行任务**</font>

- 典型用法：

  - **Send**（动态扇出任务/数据）
  - **Command(goto=...)**（运行时跳转到下游节点）

⚡ 核心判断：

> **运行时决定后续执行目标或任务数量 → 动态分支**



#### 2.2.2 并行节点

- **Send** 结合 **add_conditional_edges()** 使用，可以用于动态扇出任务

- 所谓**扇出（Fan-out）**，是指一个上游节点像扇子一样，向外分发出多个下游任务。例如，根据一个主题，同时生成诗、词、笑话三个任务；又如，根据一个列表，为列表中的每个元素动态启动一个处理任务。

- 具体来说，路由函数可以返回一个 **Send** 实例序列。每个 **Send** 实例都描述了一次独立的任务分发：
  - 分发到哪个下游节点；
  - 给这个下游节点传入什么私有状态。

- 运行时，**LangGraph** 会根据返回的每个 **Send** 实例创建对应的任务。这些任务通常会在同一个超步中并行执行。

- **Send** 类构造器源码如下

```
def __init__(self, /, node: str, arg: Any) -> None:
    """
    Initialize a new instance of the Send class.

    Args:
        node: The name of the target node to send the message to.
        arg: The state or message to send to the target node.
    """
    self.node = node
    self.arg = arg
```

- 该构造器接收两个参数，并记录为实例属性：
  - node：待启动的下游节点名称，必须是已在状态图中注册的合法名称
  - arg：传递给下游节点的信息，仅对node指向的节点可见，通常应是私有状态。每个 Send 实例接收到的 arg 是独立的，互不相干。

**实例如下**

```python
import os
from typing import TypedDict, Literal
from collections.abc import Sequence

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)

CONTENT_TYPES = ["poem", "ci_poem", "joke"]

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

class OverAllState(TypedDict):
    topic: str
    poem: str
    ci_poem: str
    joke: str

class WorkerState(TypedDict):
    """
    私有状态，只对 Worker 节点可用
    """
    content_type: Literal["poem", "ci_poem", "joke"]
    prompt: str

class InputState(TypedDict):
    topic: str

class OutputState(TypedDict):
    poem: str
    ci_poem: str
    joke: str

def worker_node(state: WorkerState) -> OutputState:
    content_type = state["content_type"]
    prompt = state["prompt"]

    content = model.invoke([HumanMessage(prompt)]).content
    return {
        content_type: content
    }

def router(state: InputState) -> Sequence[Send]:
    prompt = "请生成关于 {} 的 {}"
    english2chinese = {
        "poem": "一首诗",
        "ci_poem": "一首词",
        "joke": "一个笑话"
    }

    topic = state["topic"]

    return [Send(
            "worker_node",
            {
                "content_type": content_type,
                "prompt": prompt.format(topic, english2chinese[content_type]),
            }
        ) for content_type in CONTENT_TYPES
    ]

builder = StateGraph(state_schema=OverAllState, input_schema=InputState, output_schema=OutputState)
builder.add_node("worker_node", worker_node)
builder.add_conditional_edges(
    START,
    router,
    path_map=["worker_node"]
)
builder.add_edge("worker_node", END)

graph = builder.compile()
res = graph.invoke({"topic": "布偶狗"})
print(res)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
```

- 在这个案例中，**router()** 并没有直接返回某一个固定的下游节点名称，而是返回了多个 **Send** 实例：

```json
[ 
    Send("worker_node", {"content_type": "poem", ...}), 
    Send("worker_node", {"content_type": "ci_poem", ...}), 
    Send("worker_node", {"content_type": "joke", ...}), 
]
```

- 因此，运行时会动态创建三个指向 **worker_node** 的任务。三个任务执行的是同一个节点函数，但它们接收到的私有状态不同，所以可以分别生成**诗、词和笑话**。

- **输出如下**

```json
{
  "poem": "《布偶狗》\n\n你是一团不会褪色的黄昏，\n棉花做的心，纽扣做的眼神。\n耳朵垂下来，接住我的叹息，\n缝线弯弯，像一条安静的小河。\n\n我出门时，你守着空房间，\n我归来时，你仍坐在原位。\n不会奔跑，却从不走远，\n不会吠叫，却懂得安慰。\n\n夜里把你抱在胸前，\n像抱住一小块柔软的人间。\n布偶狗，布偶狗，\n你用沉默，替我守住了童年。",
  "ci_poem": "《鹧鸪天·布偶狗》\n\n软絮裁成小犬形，双瞳纽扣线痕轻。  \n床头端坐听风雨，怀里依偎暖夜灯。  \n\n无吠影，有温情。不言不语守深更。  \n人间聚散何须问，只把春风抱到明。",
  "joke": "布偶狗为什么从来不拆家？\n\n因为它刚咬住沙发，自己先“啪嗒”一声散架了。"
}
```

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	worker_node(worker_node)
	__end__([<p>__end__</p>]):::last
	__start__ -.-> worker_node;
	worker_node --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 此处图结构能成功渲染的关键，是在 **path_map** 中显式声明了可能被路由到的下游节点：

```python
path_map=["worker_node"]
```

- 因为 **Send** 的目标和数量可以运行时动态决定，图渲染器无法仅靠运行时返回值提前知道图结构。通过 **path_map** 声明候选下游节点后，渲染图才能正确展示 **START** 到 **worker_node** 的条件边关系。

- 补充说明：如果多个并行任务写入同一个状态字段，通常需要为该字段定义 reducer，用于合并多个任务的输出。本例中三个任务分别写入 **poem**、**ci_poem**、**joke** 三个不同字段，因此不会发生同一字段的并发合并问题。

































































# 五、节点执行与容错机制

- 节点执行失败时，如果未做任何处理，可能导致整个运行图执行失败。<font color='red'>**为了提升运行图的稳定性，处理临时故障、超时、异常恢复以及重复计算等问题**</font>，LangGraph 提供了一系列节点执行与容错机制：重试、超时设置、异常处理和节点缓存。对应关系如下。

| 问题/场景    | 对应机制                    | 机制类型             | 说明                                                         |
| :----------- | :-------------------------- | :------------------- | :----------------------------------------------------------- |
| **临时故障** | **重试 Retry**              | **节点容错机制**     | 网络抖动、接口偶发失败、模型服务短暂不可用，可以通过重试机制再次执行节点。 |
| **超时**     | **超时设置 Timeout**        | **节点容错机制**     | 某个节点执行时间过长，可以通过超时限制避免图运行被长时间阻塞。 |
| **异常恢复** | **异常处理 Error Handling** | **节点容错机制**     | 节点抛出异常后，通过异常捕获、兜底逻辑、降级返回、路由到错误处理节点等方式恢复流程。 |
| **重复计算** | **缓存 Cache**              | **节点执行优化机制** | 相同输入反复触发耗时节点，可以通过节点缓存避免重复执行，提高性能。 |



## 1、LangGraph的节点容错机制

- **LangGraph** 提供了三种异常处理机制

  - <font color='red'>**retries**</font>：重试机制。节点执行失败时，根据重试策略自动重新执行。

  - <font color='red'>**Timeouts**</font>：超时控制。限制单次节点执行的最大等待时间。

  - <font color='red'>**Error Handling**</font>：错误处理。重试次数耗尽后，执行特定的异常处理逻辑。

- 三者的执行顺序如下：

  - 节点开始执行。

  - 如果节点执行超时或抛出异常，本次节点尝试失败。

  - 重试策略 **RetryPolicy** 判断该异常是否需要重试。

  - 如果满足重试条件，并且尚未达到最大尝试次数，则重新执行节点。

  - 如果重试次数耗尽，节点仍然失败，则进入 **error_handler** 错误处理逻辑。

  - 如果没有配置错误处理逻辑，则异常继续向外抛出，可能导致整个图运行失败。

- 需要注意：

  - **重试机制** 是当前版本中最常用、最基础的节点容错能力。

  - **超时控制** 和 **错误处理** 是较新版本引入的能力，要求 **langgraph>=1.2**。





## 2、重试机制

### 2.1 基本用法

- 重试机制用于处理临时性异常，例如网络抖动、远程服务短暂不可用、API 偶发失败等。

- 在添加节点时，可以通过 **retry_policy=** 配置节点的重试策略。

- **示例如下**

~~~python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import RetryPolicy

from requests.exceptions import HTTPError
from loguru import logger

class EmptyState(TypedDict):
    pass

def node_a(state: EmptyState) -> EmptyState:
    logger.info(f"node a 运行")
    raise HTTPError("网络连接超时...")

builder = StateGraph(state_schema=EmptyState)
builder.add_node(
    "node_a",
    node_a,
    retry_policy=RetryPolicy(
        max_attempts=3,
        jitter=False
    )
)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", END)

graph = builder.compile()

try:
    graph.invoke({})
except HTTPError as e:
    logger.info("重试次数耗尽: {}", e)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- 输出

~~~bash
2026-11-03 00:14:43.924 | INFO     | __main__:node_a:12 - node a 运行
2026-11-03 00:14:44.425 | INFO     | __main__:node_a:12 - node a 运行
2026-11-03 00:14:45.426 | INFO     | __main__:node_a:12 - node a 运行
2026-11-03 00:14:45.426 | INFO     | __main__:<module>:32 - 重试次数耗尽: 网络连接超时...
~~~

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_a(node_a)
	__end__([<p>__end__</p>]):::last
	__start__ --> node_a;
	node_a --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 本例中：
  - **max_attempts=3** 表示最多尝试 3 次。
  - 这 3 次包括首次执行，而不是首次失败后的 3 次重试。
  - 因此，节点总共会被调用 3 次。
  - 3 次全部失败后，异常继续向外抛出。
  - 外层的 **try...except** 捕获该异常，并打印日志



2.2 













































































































































# 六、持久化机制和可恢复执行

## 1、概述

### 1.1 什么是可恢复执行

- 在工作流、Agent、任务编排系统里，**可恢复执行 Durable Execution** 指的是：
- <font color='red'>**把一次任务执行过程中的关键进度、状态、结果保存到可靠存储中，使任务可以在中断、失败、等待外部输入后继续执行**</font>

- 它解决的是“执行过程能否恢复”的问题。

- 例如：

```
普通执行：
开始 -> A -> B -> C
如果执行到 B 后程序挂了，重启后可能只能从 A 重新开始。

可恢复执行：
开始 -> A   保存检查点
     -> B   保存检查点
     -> C
如果执行到 B 后挂了，恢复时可以从已保存的位置继续。
```

- 所以它关注的不是“数据长期保存”本身，而是：

> <font color='red'>**执行过程本身具有可恢复性。**</font>

- 它关心的是：系统能否记住“任务执行到了哪里、当前状态是什么、接下来应该继续执行什么”。



### 1.2 LangGraph的持久化机制

#### 1.2.1介绍

- **LangGraph** 的**持久化机制**（**Persistence**）可以理解为：
  - <font color='red'>**在图执行过程中，将每个关键阶段的图状态保存为检查点，并按照线程进行组织**</font>
  - 这里的“持久化”并不是简单地把某个变量保存到文件里，而是保存一次图执行所需的状态信息，使得未来可以基于检查点继续执行。

- **LangGraph** 的持久化机制围绕以下几个问题展开：

1. <font color='red'>**保存什么？**</font>

   保存图在某个时刻的状态快照，也就是 **Checkpoint**。

2. <font color='red'>**保存到哪里？**</font>

   保存到 **Checkpointer** 管理的存储后端中。 开发时可以使用基于内存的 **InMemorySaver**，生产环境中通常会使用数据库等可靠存储，如 **PostgresSaver**。

3. <font color='red'>**如何区分不同会话？**</font>

   通过 **thread_id** 区分不同的执行线程。 同一个 **thread_id** 表示同一条持久化执行线，也可以理解为同一个会话。

4. <font color='red'>**开发者看到的是什么？**</font>

   开发者通常不会直接操作底层 **Checkpoint**，而是通过 **get_state()** 和 **get_state_history()** 查看 **Checkpoint** 的开发者视图 **StateSnapshot**。

- 因此，本章后续会围绕四个核心问题展开：

```
如何启用可恢复执行？
如何配置持久化模式？
如何查看历史检查点？
如何利用检查点进行恢复、回放和分叉？
```



#### 1.2.2 核心组件

- **LangGraph** 的持久化机制依赖一系列核心组件，如下：

| 概念                   | 作用                                                         |
| ---------------------- | ------------------------------------------------------------ |
| **State**              | 用户定义的图状态结构，负责节点间信息传递。**LangGraph** 会将 **State** 转换为底层 **Channel**；运行时真正负责节点间通信的是 **Channel**。 |
| **Channel**            | **LangGraph** 用于节点间通信的底层机制，节点从 **Channel** 读数据、向 **Channel** 写更新；既包括承载 **State** 字段更新的状态通道，也包括用于分支、任务调度等行为的内部通道。 |
| **Checkpoint**         | 检查点，超步边界上的底层状态快照，保存 **Channel** 的值、版本等信息。 |
| **CheckpointMetadata** | 与 **Checkpoint** 关联的元数据，如超步编号 **step**、父检查点 **ID parents** 等。 |
| **Checkpointer**       | 检查点存储器，负责存储检查点、元数据、配置等信息。           |
| **thread**             | 此处的线程不同于操作系统的线程，是指 **LangGraph** 中一条逻辑上的、可持久化的执行线，也可以理解为会话。一个会话可以包含多次调用，并在执行过程中生成多个检查点。因此，要将多次调用组织在一个会话中，必须为该 **thread** 配置 **checkpointer**。 |
| **thread_id**          | **thread** 的唯一标识，用于区分不同的会话。复用同一个 **thread_id**，就表示多次调用共享同一条持久化执行线，也就是共享同一个会话历史。此处的 **thread/thread_id** 和 **LangChain Agent** 中提到的 **thread/thread_id** 是同一概念。 |
| **checkpoint_ns**      | 检查点命名空间 **namespace**，用于区分父图和子图的 **checkpoint**。根图通常是空字符串 **""**，子图会有自己的 **namespace**。这个字段在子图持久化时很重要，下文会有专门的篇幅讲解子图相关用法。 |
| **checkpoint_id**      | **Checkpoint** 的唯一标识。                                  |
| **StateSnapshot**      | 开发者通过 **get_state() / get_state_history()** 看到的状态快照对象。它不是原始 **Checkpoint**，而是 **LangGraph** 基于检查点和运行时信息封装出来的开发者视图，包含当前状态值、下一步待执行节点、任务、**config**、**metadata**、中断信息等内容。 |



### 1.3 LangGraph的可恢复执行

#### 1.3.1 概念

- **LangGraph** 的可恢复执行（**Durable Execution**）依托于其持久化机制（**Persistence**）。

- 可以简单理解为：<font color='red'>**持久化机制负责保存执行状态； 可恢复执行负责利用这些状态继续执行**</font>

- **Persistence 是基础，Durable Execution 是建立在 Persistence 之上的能力。**

- 在没有持久化机制时，一次图执行通常只存在于当前进程中。 如果程序退出、节点中断、人工审批暂停，无法恢复原来的执行进度。

- 而启用持久化之后，<font color='red'>**LangGraph 可以把执行过程中的状态保存为检查点。 后续只要使用相同的 thread_id，就可以回到会话对应的历史状态**</font>



#### 1.3.2 使用场景

- 从使用场景看，可恢复执行主要支持以下几类能力：

| 场景            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| **多轮会话**    | 同一个 **thread_id** 下的多次调用可以共享历史状态。          |
| **中断恢复**    | 图执行到人工确认等节点时可以中断，之后再继续。               |
| **失败恢复**    | 程序异常退出后，可以基于已经保存的检查点继续执行，避免重复计算。 |
| **Time Travel** | 可以回到某个历史检查点，**重放**或**分叉**执行。             |

- 因此，<font color='red'>**LangGraph 的可恢复执行是一组由持久化机制支撑的能力集合。**</font>



#### 1.3.3 总结

- <font color='red'>**LangGraph 会通过 Checkpointer 持久化保存图执行过程中产生的 Checkpoint，并通过 thread_id 找回同一会话的检查点历史，从而在已有状态基础上继续执行**</font>





## 2、启用可恢复运行

### 2.1 步骤

- **LangGraph** 内置了持久化机制，但对于本地运行的 **Graph API**，只有在编译图时传入 **checkpointer**，运行过程中的状态才会被记录下来，后续才能基于已有检查点在发生故障时恢复运行或进行检查点回溯。

- 启用可恢复执行通常需要两步：
  - <font color='red'>**在编译图时传入检查点存储器对象 checkpointer**</font>
  - <font color='red'>**在调用图时传递带有 thread_id 的配置对象**</font>

- 底层的持久化机制会按照 **thread_id** 将运行时检查点记录在编译时传入的 **checkpointer** 中。



### 2.2 检查点实现

- **LangGraph** 提供了检查点存储器基类：**langgraph.checkpoint.base.BaseCheckpointSaver**

- 还提供了一系列检查点存储器实现，采用了不同的检查点后端，它们都继承了基类，整体作用是一致的：负责存取检查点，并记录节点执行的中间结果。

- 可以将 **检查点后端** 理解为：<font color='red'>**负责保存检查点数据的存储介质**</font>

- 检查点后端必须基于某个检查点实现，**常见**检查点实现和后端的对应关系如下

| 类名              | 包名                              | 检查点后端            |
| ----------------- | --------------------------------- | --------------------- |
| **InMemorySaver** | **langgraph-checkpoint**          | 内存（**In-memory**） |
| **SqliteSaver**   | **langgraph-checkpoint-sqlite**   | **SQLite**            |
| **PostgresSaver** | **langgraph-checkpoint-postgres** | **PostgreSQL**        |
| **MongoDBSaver**  | **langgraph-checkpoint-mongodb**  | **MongoDB**           |
| **RedisSaver**    | **langgraph-checkpoint-redis**    | **Redis**             |

- 在这些实现中，**InMemorySaver** 更适合学习、调试和本地实验；**SQLite** 适合轻量级本地持久化；**PostgreSQL**、**MongoDB**、**Redis** 等数据库后端更适合需要跨进程、跨服务保存状态的场景。



### 2.3 基于内存的检查点存储器

- **LangGraph** 提供了基于内存的检查点存储器 **InMemorySaver**。它将检查点数据保存在当前 **Python** 进程的内存中，进程结束则数据丢失，适合快速开发、调试。

- **示例如下**

```python
import os

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langgraph.checkpoint.memory import InMemorySaver
from langchain.messages import HumanMessage


from dotenv import load_dotenv
load_dotenv(override=True)

CONTENT_TYPES = ["poem", "ci_poem", "joke"]

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

class OverAllState(MessagesState):
    output: str

def llm_node(state: OverAllState) -> OverAllState:
    messages = state["messages"]
    res = model.invoke(messages)
    return {
        "messages": [res]
    }

def output_node(state: OverAllState) -> OverAllState:
    return {
        "output": state["messages"][-1].content
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("llm_node", llm_node)
builder.add_node("output_node", output_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", "output_node")
builder.add_edge("output_node", END)

# 定义并在编译时传递 Checkpointer
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# 定义配置对象
config = {"configurable": {"thread_id": "chapter_6_6-2-2"}}
# 调用时传递
graph.invoke({"messages": [HumanMessage("你好，我是老王")]}, config=config)
graph.invoke({"messages": [HumanMessage("从现在开始，你是小王")]}, config=config)
res = graph.invoke({"messages": [HumanMessage("我是谁？你是谁？")]}, config=config)

print('=' * 30, '-> 完整消息列表 <-', '=' * 30)
for msg in res["messages"]:
    msg.pretty_print()

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
```

- **输出如下**

```
============================== -> 完整消息列表 <- ==============================
================================ Human Message =================================

你好，我是老王
================================== Ai Message ==================================

你好，老王！很高兴认识你。有什么我可以帮你的吗？
================================ Human Message =================================

从现在开始，你是小王
================================== Ai Message ==================================

好的，老王，从现在起我就是小王。有什么需要我做的，尽管说。
================================ Human Message =================================

我是谁？你是谁？
================================== Ai Message ==================================

你是老王；我是小王（按你刚才的设定）。本质上我是你的 AI 助手。
```

- 本例中，三次调用使用了相同的 **thread_id**，所以后两次调用能够读取前面已经保存的消息历史。最终模型可以知道“用户是老王”，也可以知道“自己被要求扮演小王”。

- 需要注意的是，如果重新执行完整代码，**InMemorySaver()** 实例会被重新创建，历史检查点会被清空。因此，消息列表不会继续累加，而是会基于新的空状态重新开始。



### 2.4 基于持久化数据库的检查点存储器

- 示例将 **PostgreSQL** 作为检查点后端，对应的 **Checkpointer** 实现为 **PostgresSaver**。

- 和 **InMemorySaver** 不同，**PostgresSaver** 会将检查点保存到 **PostgreSQL** 数据库中。只要数据库中的记录没有被删除，即使 **Python** 程序结束、连接对象重建，历史检查点也仍然存在。

- **示例如下**

```python
import os

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langgraph.checkpoint.postgres import PostgresSaver
from langchain_deepseek import ChatDeepSeek
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)


CONTENT_TYPES = ["poem", "ci_poem", "joke"]

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

class OverAllState(MessagesState):
    output: str

def llm_node(state: OverAllState) -> OverAllState:
    messages = state["messages"]
    res = model.invoke(messages)
    return {
        "messages": [res]
    }

def output_node(state: OverAllState) -> OverAllState:
    return {
        "output": state["messages"][-1].content
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("llm_node", llm_node)
builder.add_node("output_node", output_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", "output_node")
builder.add_edge("output_node", END)

# 定义并在编译时传递 Checkpointer
DB_URL = "postgresql://postgres:123456@172.20.1.36:5432/lzy?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URL) as checkpointer:
    # 示例中为了方便演示直接调用 setup()
    # 实际项目中通常建议把数据库初始化/迁移作为独立步骤处理
    checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer)

    # 定义配置对象
    config = {"configurable": {"thread_id": "chapter_6_6.2.4"}}
    # 调用时传递
    graph.invoke({"messages": [HumanMessage("你好，我是老王")]}, config=config)
    graph.invoke({"messages": [HumanMessage("从现在开始，你是小王")]}, config=config)
    res = graph.invoke({"messages": [HumanMessage("我是谁？你是谁？")]}, config=config)
    print(res["output"])

    print('=' * 30, '-> 完整消息列表 <-', '=' * 30)
    for msg in res["messages"]:
        msg.pretty_print()

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
```

- 运行结果如下

```
您是老王，我是小王。您刚才让我从现在开始当小王。
============================== -> 完整消息列表 <- ==============================
================================ Human Message =================================

你好，我是老王
================================== Ai Message ==================================

你好，老王！很高兴认识你。有什么我可以帮你的吗？
================================ Human Message =================================

从现在开始，你是小王
================================== Ai Message ==================================

好的老王，从现在开始我就是小王。有什么需要我帮忙的，您尽管说。
================================ Human Message =================================

我是谁？你是谁？
================================== Ai Message ==================================

您是老王，我是小王。您刚才让我从现在开始当小王。
```

- 如果多次运行这段代码，并且始终使用相同的 **thread_id**，那么历史消息会不断累加。这是因为检查点已经被保存到了 **PostgreSQL** 中，重新创建 **Python** 连接对象并不会清空数据库中的历史记录。

- 再次运行结果如下所示。

```
您是老王，我是小王。
============================== -> 完整消息列表 <- ==============================
================================ Human Message =================================

你好，我是老王
================================== Ai Message ==================================

你好，老王！很高兴认识你。有什么我可以帮你的吗？
================================ Human Message =================================

从现在开始，你是小王
================================== Ai Message ==================================

好的老王，从现在开始我就是小王。有什么需要我帮忙的，您尽管说。
================================ Human Message =================================

我是谁？你是谁？
================================== Ai Message ==================================

您是老王，我是小王。您刚才让我从现在开始当小王。
================================ Human Message =================================

你好，我是老王
================================== Ai Message ==================================

你好，老王！我是小王。有什么需要我帮忙的吗？
================================ Human Message =================================

从现在开始，你是小王
================================== Ai Message ==================================

好的，老王，我记住了：您是老王，我是小王。有什么吩咐您说。
================================ Human Message =================================

我是谁？你是谁？
================================== Ai Message ==================================

您是老王，我是小王。
```

- 因此，**PostgresSaver** 和 **InMemorySaver** 的关键区别在于：
  - **InMemorySaver**：<font color='red'>**状态保存在当前进程内存中，进程结束或对象重建后丢失**</font>
  - **PostgresSaver**：<font color='red'>**状态保存在 PostgreSQL 中，只要数据库记录存在，程序重启后仍可读取**</font>

- 所以，基于数据库的 **Checkpointer** 可靠性更高、更适合需要长期保存会话状态的场景



## 3、持久化模式

### 3.1 概述

- 启用 **Checkpointer** 后，**LangGraph** 会在执行过程中保存检查点。

- 检查点写入**越及时**，系统在异常中断后可恢复的状态<font color='red'>**越完整，容灾能力越强**</font>；但更及时的持久化通常也意味着<font color='red'>**更多中间写入**</font>，或**额外的**阻塞式**等待**，带来<font color='red'>**额外的性能开销和响应延迟**</font>（从调用者的角度考虑）。

- <font color='red'>**LangGraph 支持三种持久化模式**</font>，采用不同的检查点保存时机，在容灾能力和性能开销、响应时效性之间作取舍。
  - <font color='red'>**exit**</font>：退出模式。只在计算图正常结束、异常退出、被中断（如 **Human-In-The-Loop** 中断）时保存检查点。<font color='red'>**不能处理中途进程崩溃的场景**</font>。这种模式响应性能开销最小，但容灾能力最弱。
  - <font color='red'>**async**</font>：异步模式。默认模式，顾名思义，检查点在后台异步写入。
    - 它会在每个超步结束后写入<font color='red'>**完整检查点（主检查点）**</font>，并在图中任务执行完毕后记录**中间结果**。
    - 和 **exit** 模式相比，增加了性能开销，但写入操作发生在后台，不会引入明显的响应延迟，同时提升了容灾能力。
  - <font color='red'>**sync**</font>：同步模式。
    - 和异步模式唯一的区别在于，**LangGraph** 会在进入下一个超步之前等待当前主检查点的写入任务完成。
    - 它的容灾能力最强，但在 **async** 模式的基础上增加了响应延迟。

- **总结**

| 模式      | 写入时机                                                     | 性能开销 | 响应延迟 | 容灾能力 |
| --------- | ------------------------------------------------------------ | -------- | -------- | -------- |
| **exit**  | 运行退出时写入                                               | 低       | 最低     | 最弱     |
| **async** | 每个超步末尾写入主检查点，任务完成后写入中间结果，后台异步写入 | 高       | 较高     | 较强     |
| **sync**  | 和 **async** 区别在于，进入下一个超步之前等待当前主检查点的写入任务完成 | 高       | 最高     | 最强     |



### 3.2 示例

- 持久化模式影响的是检查点写入时机和故障恢复能力，通常**不会影响**正常情况下的**业务输出**。

- 也就是说，同一张图在正常执行完成时，使用 **exit**、**async** 或 **sync**，最终返回结果通常是一样的。真正的区别主要体现在**异常中断**或**进程崩溃**后**恢复执行**时。

> **durability 控制的是检查点写入策略，不改变图本身的执行逻辑。**

- 要真正看到不同模式的差异
  - 可以在学习**使用场景**后自行设计案例
  - 或调试源码，观察不同模式下的写入时机

~~~python
import os

from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langchain.messages import HumanMessage

from dotenv import load_dotenv
load_dotenv(override=True)


CONTENT_TYPES = ["poem", "ci_poem", "joke"]

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

class OverAllState(MessagesState):
    output: str

def llm_node(state: OverAllState) -> OverAllState:
    messages = state["messages"]
    res = model.invoke(messages)
    return {
        "messages": [res]
    }

def output_node(state: OverAllState) -> OverAllState:
    return {
        "output": state["messages"][-1].content
    }

builder = StateGraph(state_schema=OverAllState)
builder.add_node("llm_node", llm_node)
builder.add_node("output_node", output_node)
builder.add_edge(START, "llm_node")
builder.add_edge("llm_node", "output_node")
builder.add_edge("output_node", END)

# 定义并在编译时传递 Checkpointer
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# 定义配置对象
config = {"configurable": {"thread_id": "chapter_tmp"}}
# 调用时传递
res=graph.invoke(
    {"messages": [HumanMessage("你好")]},
    config=config,
    durability="async" # sync / exit
)
print(res["output"])

print('=' * 30, '-> 完整消息列表 <-', '=' * 30)
for msg in res["messages"]:
    msg.pretty_print()

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- 上述代码中，可以通过修改 **durability** 参数切换持久化模式：

~~~python
durability="exit"
durability="async"
durability="sync"
~~~

- 运行结果

~~~ bash
你好！很高兴见到你 😊

有什么我可以帮你的吗？无论是聊天、解答问题、写作、翻译，还是帮你分析资料、写代码，都可以告诉我。
============================== -> 完整消息列表 <- ==============================
================================ Human Message =================================

你好
================================== Ai Message ==================================

你好！很高兴见到你 😊

有什么我可以帮你的吗？无论是聊天、解答问题、写作、翻译，还是帮你分析资料、写代码，都可以告诉我。
~~~

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	llm_node(llm_node)
	output_node(output_node)
	__end__([<p>__end__</p>]):::last
	__start__ --> llm_node;
	llm_node --> output_node;
	output_node --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~



## 4、查看历史检查点

- **LangGraph** 在配置检查点存储器后，会把同一个 **thread_id** 下的执行过程保存为一组检查点。通过这些检查点，我们可以查看图运行的中间状态，也可以为后续的检查点回溯（重放、分叉）和失败恢复做准备。

- 本节主要介绍两个常用方法：
  - **graph.get_state_history(config)**：查看指定会话的完整历史检查点。
  - **graph.get_state(config)**：查看指定会话的**最新**检查点，或者查看某个**指定** **checkpoint_id** 对应的检查点。



### 4.1 查看完整历史检查点列表

- 核心代码

~~~python
history_checkpoints = list(graph.get_state_history(config=config))
print(history_checkpoints)
~~~

- 代码示例

~~~python
import os
from typing import TypedDict

from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage

from dotenv import load_dotenv
from loguru import logger

load_dotenv(override=True)


CONTENT_TYPES = ["poem", "ci_poem", "joke"]

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

class OverAllState(TypedDict):
    topic: str
    poem: str
    joke: str
    final_output: str

class InputState(TypedDict):
    topic: str

class OutputState(TypedDict):
    final_output: str

def node_poem(state: InputState) -> OverAllState:
    logger.info(f"node_poem 已执行")
    topic = state["topic"]
    poem = model.invoke([HumanMessage(f"写一首关于 {topic} 的七言绝句")]).content

    return {
        "poem": poem
    }

def node_joke(state: InputState) -> OverAllState:
    logger.info(f"node_joke 已执行")
    topic = state["topic"]
    joke = model.invoke([HumanMessage(f"写一个关于 {topic} 的笑话")]).content

    return {
        "joke": joke
    }

def node_output(state: OverAllState) -> OutputState:
    logger.info("node_output 已执行")
    topic = state["topic"]
    poem = state["poem"]
    joke = state["joke"]
    final_output = f"关于 {topic} 的七言绝句：\n{poem}\n笑话：\n{joke}"

    return {
        "final_output": final_output
    }

builder = StateGraph(state_schema=OverAllState, input_schema=InputState, output_schema=OutputState)
builder.add_node("node_poem", node_poem)
builder.add_node("node_joke", node_joke)
builder.add_node("node_output", node_output)
builder.add_edge(START, "node_poem")
builder.add_edge(START, "node_joke")
builder.add_edge("node_poem", "node_output")
builder.add_edge("node_joke", "node_output")
builder.add_edge("node_output", END)

checkpointer = InMemorySaver()
config = {"configurable": {"thread_id": "123"}}
graph = builder.compile(checkpointer=checkpointer)
res = graph.invoke({"topic": "猫咪"}, config=config)

print('=' * 30, '-> 运行结果 <-', '=' * 30)
print("res: {}", res)

print('=' * 30, '-> 历史检查点列表 <-', '=' * 30)
history_checkpoints = list(graph.get_state_history(config=config))
print(history_checkpoints)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- 运行结果
  - 由于 **node_poem** 和 **node_joke** 都从 **START** 出发，并且彼此没有依赖关系，所以它们会被调度到同一个超步中执行。实际日志顺序可能不同，例如可能先打印 **node_joke**，也可能先打印 **node_poem**

~~~bash
2026-11-03 15:25:19.928 | INFO     | __main__:node_joke:50 - node_joke 已执行
2026-11-03 15:25:19.928 | INFO     | __main__:node_poem:41 - node_poem 已执行
2026-11-03 15:25:34.594 | INFO     | __main__:node_output:59 - node_output 已执行
~~~

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_poem(node_poem)
	node_joke(node_joke)
	node_output(node_output)
	__end__([<p>__end__</p>]):::last
	__start__ --> node_joke;
	__start__ --> node_poem;
	node_joke --> node_output;
	node_poem --> node_output;
	node_output --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 最终结果和检查点列表

~~~json
============================== -> 运行结果 <- ==============================
res: {} {'final_output': '关于 猫咪 的七言绝句：\n《戏猫》\n狸奴酣卧小窗南，醉眼惺忪戏玉簪。\n忽跃雕檐追粉蝶，却衔明月入花龛。\n笑话：\n小猫问妈妈：“为什么人总说‘猫有九条命’？”  \n妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  \n小猫追问：“那为什么狗只有一条命？”  \n妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”'}
============================== -> 历史检查点列表 <- ==============================
[
    StateSnapshot(
        values={
            "topic": "猫咪",
            "poem": """
                        《戏猫》
                狸奴酣卧小窗南，醉眼惺忪戏玉簪。
                忽跃雕檐追粉蝶，却衔明月入花龛。
                """,
            "joke": """
                小猫问妈妈：“为什么人总说‘猫有九条命’？”  
                妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  
                小猫追问：“那为什么狗只有一条命？”  
                妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”
                """,
            "final_output": """
                关于 猫咪 的七言绝句：
                《戏猫》
                狸奴酣卧小窗南，醉眼惺忪戏玉簪。
                忽跃雕檐追粉蝶，却衔明月入花龛。
                笑话：
                小猫问妈妈：“为什么人总说‘猫有九条命’？”  
                妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  
                小猫追问：“那为什么狗只有一条命？”  
                妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”
                """
        },
        next=(),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-4cc7-61bd-8002-bcd7f63c3238"
            }
        },
        metadata={
            "source": "loop",
            "step": 2,
            "parents": {}
        },
        created_at="2026-06-09T09:36:08.368575+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-4cc3-6d96-8001-8314b5dbbfe3"
            }
        },
        tasks=(),
        interrupts=()
    ),

    StateSnapshot(
        values={
            "topic": "猫咪",
            "poem": """
                        《戏猫》
                狸奴酣卧小窗南，醉眼惺忪戏玉簪。
                忽跃雕檐追粉蝶，却衔明月入花龛。
                """,
            "joke": """
                小猫问妈妈：“为什么人总说‘猫有九条命’？”  
                妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  
                小猫追问：“那为什么狗只有一条命？”  
                妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”
                """
        },
        next=("node_output",),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-4cc3-6d96-8001-8314b5dbbfe3"
            }
        },
        metadata={
            "source": "loop",
            "step": 1,
            "parents": {}
        },
        created_at="2026-06-09T09:36:08.367236+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-393f-6082-8000-b9c0708bf03d"
            }
        },
        tasks=(
            PregelTask(
                id="91d81ad6-d2fb-ebd7-2dd1-8e08c2be5a07",
                name="node_output",
                path=("__pregel_pull", "node_output"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "final_output": """
                        关于 猫咪 的七言绝句：
                        《戏猫》
                        狸奴酣卧小窗南，醉眼惺忪戏玉簪。
                        忽跃雕檐追粉蝶，却衔明月入花龛。
                        笑话：
                        小猫问妈妈：“为什么人总说‘猫有九条命’？”  
                        妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  
                        小猫追问：“那为什么狗只有一条命？”  
                        妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”
                        """
                }
            ),
        ),
        interrupts=()
    ),

    StateSnapshot(
        values={
            "topic": "猫咪"
        },
        next=("node_poem", "node_joke"),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-393f-6082-8000-b9c0708bf03d"
            }
        },
        metadata={
            "source": "loop",
            "step": 0,
            "parents": {}
        },
        created_at="2026-06-09T09:36:06.320542+00:00",
        parent_config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-393d-6b9a-bfff-180e30c1d3bc"
            }
        },
        tasks=(
            PregelTask(
                id="80d8974c-3450-6faa-8b06-dcf8787d74fc",
                name="node_poem",
                path=("__pregel_pull", "node_poem"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "poem": """
                            《戏猫》
                    狸奴酣卧小窗南，醉眼惺忪戏玉簪。
                    忽跃雕檐追粉蝶，却衔明月入花龛。
                    """
                }
            ),
            PregelTask(
                id="2ca46996-dfc6-6de3-3662-cf1fe41bf987",
                name="node_joke",
                path=("__pregel_pull", "node_joke"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "joke": """
                    小猫问妈妈：“为什么人总说‘猫有九条命’？”  
                    妈妈答：“因为人类怕我们死一次就够他们内疚一辈子。”  
                    小猫追问：“那为什么狗只有一条命？”  
                    妈妈叹气：“因为狗犯错靠装可怜，而我们靠记仇。”
                    """
                }
            ),
        ),
        interrupts=()
    ),

    StateSnapshot(
        values={},
        next=("__start__",),
        config={
            "configurable": {
                "thread_id": "123",
                "checkpoint_ns": "",
                "checkpoint_id": "1f163e6a-393d-6b9a-bfff-180e30c1d3bc"
            }
        },
        metadata={
            "source": "input",
            "step": -1,
            "parents": {}
        },
        created_at="2026-06-09T09:36:06.320009+00:00",
        parent_config=None,
        tasks=(
            PregelTask(
                id="3aa3bc58-3567-5832-d823-4373a9ab94e5",
                name="__start__",
                path=("__pregel_pull", "__start__"),
                error=None,
                interrupts=(),
                state=None,
                result={
                    "topic": "猫咪"
                }
            ),
        ),
        interrupts=()
    )
]
~~~



#### 4.1.1 分析

##### 4.1.1.1 StateSnapshot

- **get_state_history()** 返回的是一个历史检查点迭代器。将其转换为列表后，可以看到一组 **StateSnapshot** 对象。

- **StateSnapshot** 是检查点的开发者视图，字段含义如下

  - **values**：当前检查点的状态值
  - **next**：从该检查点继续执行时，下一步将要执行的节点，即归属于下一个超步的节点
  - **config**：当前检查点配置。常见结构如下：

  ~~~python
  {
      "configurable": {
          "thread_id": "...",
          "checkpoint_ns": "...",
          "checkpoint_id": "..."
      }
  }
  ~~~

  - 其中

    - **configurable**：用于记录检查点的可配置信息，可能包含用户自定义字段，它一定包含以下三个字段：

      - **thread_id**：会话唯一标识
      - **checkpoint_ns**：检查点命名空间。根图的命名空间通常为空字符串 **""**；子图会使用非空命名空间。
      - **checkpoint_id**：检查点唯一标识

      这三个字段可以唯一标识一条检查点记录，在大多数检查点存储器实现中，写入检查点后端时都会将它们作为检查点的**唯一键**

  - **metadata**：检查点元数据，本节只需要关注 **step** 字段，后者是当前检查点对应的超步编号

  - **created_at**：检查点创建时间

  - **parent_config**：父检查点、即上一个检查点的配置

  - **tasks**：当前检查点关联的待执行任务信息，元素类型通常是 **PregelTask**。

    - **tasks** 通常和 **next** 对应，表示从当前检查点继续执行时，下一步将要运行的任务。

    - **需要注意的是**，**tasks** 中还可能包含这些任务已经成功（**result**）或失败（**error**）的任务记录，这是在下一个超步执行过程中记录的中间结果。在失败恢复场景中，可以避免重复执行已成功节点。

  - **interrupts**：当前图的中断信息，中断机制详见下文



##### 4.1.1.2 检查点主要信息说明

- 本例中检查点按照如下顺序返回

```python
Step 2 → Step 1 → Step 0 → Step -1
即:
最新检查点 → 最旧检查点
```

- 其中：

1. **step=-1**

   - 这是输入检查点，**metadata["source"] == "input"**，这表示该检查点是在运行时初始化时被记录的，尚未进入图计算循环。

   - 此时用户输入还没有真正变成普通业务节点可消费的状态，因此

   ```python
   values={}
   ```

   - 从这个检查点继续执行，下一步是内部启动节点 **__start__**，因此：

   ```python
   next=("__start__",)
   ```

2. **step=0**

   - 这是图计算循环的第一个检查点，**metadata["source"] == "loop"**，这表示检查点是在图计算循环中被写入的。

   - 此时，**__start__** 已经把用户输入写入状态，所以 **values** 中已经可以看到状态信息，如下：

   ```python
   values={
       "topic": "猫咪"
   }
   ```

   - 从这个检查点继续执行，**node_poem** 和 **node_joke** 将在下一个超步执行，因此：

   ```python
   next=("node_poem", "node_joke")
   ```

3. **step=1**

   - 这是图计算循环的第二个检查点，同样 **metadata["source"] == "loop"**。

   - 同时，它对应第一个执行图节点计算逻辑的超步。

   - 此时，**node_poem** 和 **node_joke** 的输出已经到状态中，因此：

   ```python
   values={
       "topic": "猫咪",
       "poem": "...",
       "joke": "..."
   }
   ```

   - 从这个检查点开始，下游的汇总节点 **node_output** 满足执行条件，因此：

   ```python
   next=("node_output",)
   ```

4. **step=2**

   - 这是图计算循环的第三个检查点，同样 **metadata["source"] == "loop"**

   - 此时，**node_output** 已经执行完成，最终输出也已经写入状态，因此：

   ```python
   values={
       "topic": "猫咪",
       "poem": "...",
       "joke": "...",
       "final_output": "..."
   }
   ```

   - 此时，图已经执行结束，所以：

   ```python
   next=()
   ```



##### 4.1.1.3 tasks字段说明

- **tasks** 是一个元组，其中的每个元素都是一个 **PregelTask** 实例

- **tasks** 表示：如果从当前检查点继续执行，下一步要运行哪些任务。比如在 **step=0** 的检查点中：

```python
next=("node_poem", "node_joke")
tasks=(
    PregelTask(name="node_poem", ...),
    PregelTask(name="node_joke", ...),
)
```

- 这表示从该检查点继续执行时，下一步会运行 **node_poem** 和 **node_joke** 节点，底层执行的是它们的同名任务。

- 不过，查看历史检查点时，**PregelTask** 实例的 **result** 字段可能非空，如：

```python
PregelTask(
    name="node_poem",
    result={
        "poem": "..."
    }
)
```

- 这看起来像是“上一个检查点中提前保存了下一个超步的执行结果”。实际上：
  - 完整的检查点（主检查点）是在超步末尾保存的
  - 但在一个超步内部，**LangGraph** 还会记录节点级别的计算结果
  - **checkpoint["id"]** 会在保存主检查点前由 **create_checkpoint()** 推进为新的 **ID**，然后用这个 **ID** 写入主检查点（假设当前超步为 **S1**）。下一轮（超步为 **S2**）计算循环中，任务执行完毕后的中间结果会用相同的 **ID** 写入。

  - 因此，下一轮（超步为 **S2**）任务的计算结果绑定到了“当前超步（超步为 **S1**）的主检查点”。
  - 从检查点后端检索时，会按照 **checkpoint_id** 将主检查点和中间结果组织起来
  - 因此，检索到的历史检查点列表中，主检查点（超步为 **S1**）和下一个超步（超步为 **S2**）的中间结果会被放在同一个 **StateSnapshot** 中
  - 如果同一个超步里有多个并行任务，其中部分任务已经成功，另一个任务失败，那么成功任务的结果可以被保存下来；
  - 后续恢复时，**LangGraph** 可以复用这些已成功任务的结果，避免重复执行已经成功的节点。



### 4.2 查看最新检查点

- 如果只想查看当前线程的最新检查点，可以使用：

~~~python
latest_history_checkpoint = graph.get_state(config=config)
print(latest_history_checkpoint)
~~~

- 示例代码

~~~python
import os
from typing import TypedDict

from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage

from dotenv import load_dotenv
from loguru import logger

load_dotenv(override=True)


CONTENT_TYPES = ["poem", "ci_poem", "joke"]

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

class OverAllState(TypedDict):
    topic: str
    poem: str
    joke: str
    final_output: str

class InputState(TypedDict):
    topic: str

class OutputState(TypedDict):
    final_output: str

def node_poem(state: InputState) -> OverAllState:
    logger.info(f"node_poem 已执行")
    topic = state["topic"]
    poem = model.invoke([HumanMessage(f"写一首关于 {topic} 的七言绝句，不要赏析，只给诗句即可")]).content

    return {
        "poem": poem
    }

def node_joke(state: InputState) -> OverAllState:
    logger.info(f"node_joke 已执行")
    topic = state["topic"]
    joke = model.invoke([HumanMessage(f"写一个关于 {topic} 的笑话，尽可能简单，不超过一百字")]).content

    return {
        "joke": joke
    }

def node_output(state: OverAllState) -> OutputState:
    logger.info("node_output 已执行")
    topic = state["topic"]
    poem = state["poem"]
    joke = state["joke"]
    final_output = f"关于 {topic} 的七言绝句：\n{poem}\n笑话：\n{joke}"

    return {
        "final_output": final_output
    }

builder = StateGraph(state_schema=OverAllState, input_schema=InputState, output_schema=OutputState)
builder.add_node("node_poem", node_poem)
builder.add_node("node_joke", node_joke)
builder.add_node("node_output", node_output)
builder.add_edge(START, "node_poem")
builder.add_edge(START, "node_joke")
builder.add_edge("node_poem", "node_output")
builder.add_edge("node_joke", "node_output")
builder.add_edge("node_output", END)

checkpointer = InMemorySaver()
config = {"configurable": {"thread_id": "123"}}
graph = builder.compile(checkpointer=checkpointer)
res = graph.invoke({"topic": "猫咪"}, config=config)

print('=' * 30, '-> 运行结果 <-', '=' * 30)
print("res: {}", res)

print('=' * 30, '-> 最新历史检查点 <-', '=' * 30)
latest_history_checkpoint = graph.get_state(config=config)
print(latest_history_checkpoint)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- 运行结果

~~~bash
2026-11-03 16:27:03.163 | INFO     | __main__:node_joke:50 - node_joke 已执行
2026-11-03 16:27:03.163 | INFO     | __main__:node_poem:41 - node_poem 已执行
2026-11-03 16:27:25.393 | INFO     | __main__:node_output:59 - node_output 已执行
~~~

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_poem(node_poem)
	node_joke(node_joke)
	node_output(node_output)
	__end__([<p>__end__</p>]):::last
	__start__ --> node_joke;
	__start__ --> node_poem;
	node_joke --> node_output;
	node_poem --> node_output;
	node_output --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 运行结果及检查点信息

~~~json
============================== -> 运行结果 <- ==============================
res: {} {'final_output': '关于 猫咪 的七言绝句：\n《戏题狸奴》\n昼眠锦褥夜巡檐，碧眼圆瞳映月纤。\n捕鼠本非真本领，得鱼时复近妆奁。\n笑话：\n小猫咪第一次照镜子，看到里面的自己，吓得炸毛：“你是谁？！” 它绕到镜子后面找了一圈，没找到，又回来对着镜子哈气。 最后它恍然大悟，对主人说：“我明白了，那是个隐藏的摄像头！”'}
============================== -> 最新历史检查点 <- ==============================
StateSnapshot(
    values={
        "topic": "猫咪",
        "poem": (
            "《戏题狸奴》\n"
            "昼眠锦褥夜巡檐，碧眼圆瞳映月纤。\n"
            "捕鼠本非真本领，得鱼时复近妆奁。"
        ),
        "joke": (
            "小猫咪第一次照镜子，看到里面的自己，吓得炸毛：“你是谁？！” "
            "它绕到镜子后面找了一圈，没找到，又回来对着镜子哈气。 "
            "最后它恍然大悟，对主人说：“我明白了，那是个隐藏的摄像头！”"
        ),
        "final_output": (
            "关于 猫咪 的七言绝句：\n"
            "《戏题狸奴》\n"
            "昼眠锦褥夜巡檐，碧眼圆瞳映月纤。\n"
            "捕鼠本非真本领，得鱼时复近妆奁。\n"
            "笑话：\n"
            "小猫咪第一次照镜子，看到里面的自己，吓得炸毛：“你是谁？！” "
            "它绕到镜子后面找了一圈，没找到，又回来对着镜子哈气。 "
            "最后它恍然大悟，对主人说：“我明白了，那是个隐藏的摄像头！”"
        ),
    },
    next=(),
    config={
        "configurable": {
            "thread_id": "123",
            "checkpoint_ns": "",
            "checkpoint_id": "1f1647bd-3512-631a-8002-4e48b5ecda6d",
        }
    },
    metadata={
        "source": "loop",
        "step": 2,
        "parents": {},
    },
    created_at="2026-06-10T03:24:01.426084+00:00",
    parent_config={
        "configurable": {
            "thread_id": "123",
            "checkpoint_ns": "",
            "checkpoint_id": "1f1647bd-350d-62c7-8001-34cd7fa012ed",
        }
    },
    tasks=(),
    interrupts=(),
)
~~~

- **graph.get_state(config=config)** 只返回当前会话的最新检查点。

  - 在本例中，图已经执行完成，所以最新检查点具有以下特征：

  ```python
  next=()
  tasks=()
  ```

  - 这表示没有后续节点需要继续执行



### 4.3 根据ID查看指定检查点

- 如果希望查看某个历史检查点，可以在 **configurable** 中额外传入 **checkpoint_id**：

```python
target_config = {
    "configurable": {
        "thread_id": "123",
        "checkpoint_id": "某个历史 checkpoint_id"
    }
}

snapshot = graph.get_state(config=target_config)
print(snapshot)
```

- 这样返回的就不是最新检查点，而是指定 **checkpoint_id** 对应的历史检查点

- 从完整历史检查点列表中获取一个**checkpoint_id**，然后调用**graph.get_state()** 查看该检查点

~~~python
import os
from typing import TypedDict

from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage

from dotenv import load_dotenv
from loguru import logger

load_dotenv(override=True)


CONTENT_TYPES = ["poem", "ci_poem", "joke"]

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

class OverAllState(TypedDict):
    topic: str
    poem: str
    joke: str
    final_output: str

class InputState(TypedDict):
    topic: str

class OutputState(TypedDict):
    final_output: str

def node_poem(state: InputState) -> OverAllState:
    logger.info(f"node_poem 已执行")
    topic = state["topic"]
    poem = model.invoke([HumanMessage(f"写一首关于 {topic} 的七言绝句")]).content

    return {
        "poem": poem
    }

def node_joke(state: InputState) -> OverAllState:
    logger.info(f"node_joke 已执行")
    topic = state["topic"]
    joke = model.invoke([HumanMessage(f"写一个关于 {topic} 的笑话")]).content

    return {
        "joke": joke
    }

def node_output(state: OverAllState) -> OutputState:
    logger.info("node_output 已执行")
    topic = state["topic"]
    poem = state["poem"]
    joke = state["joke"]
    final_output = f"关于 {topic} 的七言绝句：\n{poem}\n笑话：\n{joke}"

    return {
        "final_output": final_output
    }

builder = StateGraph(state_schema=OverAllState, input_schema=InputState, output_schema=OutputState)
builder.add_node("node_poem", node_poem)
builder.add_node("node_joke", node_joke)
builder.add_node("node_output", node_output)
builder.add_edge(START, "node_poem")
builder.add_edge(START, "node_joke")
builder.add_edge("node_poem", "node_output")
builder.add_edge("node_joke", "node_output")
builder.add_edge("node_output", END)

checkpointer = InMemorySaver()
config = {"configurable": {"thread_id": "123"}}
graph = builder.compile(checkpointer=checkpointer)
res = graph.invoke({"topic": "猫咪"}, config=config)

print('=' * 30, '-> 运行结果 <-', '=' * 30)
print("res: {}", res)

print('=' * 30, '-> 历史检查点列表 <-', '=' * 30)
history_checkpoints = list(graph.get_state_history(config=config))
print(history_checkpoints)

print('=' * 30, '-> 根据ID查看特定检查点 <-', '=' * 30)
checkpointe_id = history_checkpoints[3].config["configurable"]["checkpoint_id"]
print(f"checkpointe_id: {checkpointe_id}")

print('=' * 30, '-> 指定ID的检查点数据 <-', '=' * 30)
target_config = {
    "configurable": {
        "thread_id": config["configurable"]["thread_id"],
        "checkpoint_id": checkpointe_id
    }
}
specific_history_checkpoint = graph.get_state(config=target_config)
print(specific_history_checkpoint)

raw_mermaid = graph.get_graph().draw_mermaid()
print(raw_mermaid)
~~~

- 输出

~~~python
============================== -> 根据ID查看特定检查点 <- ==============================
checkpointe_id: 1f165804-9a90-6db5-bfff-d0eb857db9d8
StateSnapshot(
    values={},
    next=("__start__",),
    config={
        "configurable": {
            "thread_id": "123",
            "checkpoint_id": "1f165804-9a90-6db5-bfff-d0eb857db9d8",
        }
    },
    metadata={
        "source": "input",
        "step": -1,
        "parents": {},
    },
    created_at="2026-06-11T10:28:29.123104+00:00",
    parent_config=None,
    tasks=(
        PregelTask(
            id="63a24d9c-ea6d-0420-4b5a-b978c6eaeb90",
            name="__start__",
            path=("__pregel_pull", "__start__"),
            error=None,
            interrupts=(),
            state=None,
            result={
                "topic": "猫咪",
            },
        ),
    ),
    interrupts=(),
)
~~~

- 当前场景下，我们已经获取了完整检查点列表，然后从中取了某个检查点的 **checkpoint_id**，然后调用 **get_state()** 查到了它对应的快照。
- 这看起来有些“绕”，因为我们本来就已经拿到了完整的历史快照。但在实际场景中，这个方法仍然很有意义。
- 例如，在后续的 **检查点回溯-分叉** 场景中，我们经常会基于历史检查点创建新的检查点，拿到该检查点的、带有 **checkpoint_id** 的配置信息 **config**。此时就可以将 **config** 传递给 **graph.get_state()**，从而便捷而精确地查看该检查点的快照



## 5、使用场景

### 5.1 多轮对话

最基础的使用场景是多轮对话。前文示例已经实现了一个简单的多轮对话流程，这里不再展开。



### 5.2 中断恢复

中断恢复需要结合 **LangGraph** 的中断机制理解，见下文。



### 5.3 失败后恢复运行

#### 5.3.1 用法说明

- 任务失败的原因可能是偶发的外部因素，如网络波动、秘钥过期、第三方服务异常；也可能是程序自身的BUG。

- 如果是前者，故障发生后，修复外部问题，直接重试即可；如果是后者，需要更改代码修复BUG后，再基于历史检查点恢复运行。

- 要在失败后基于历史检查点恢复运行，需要满足以下条件：
  - 1、启用检查点存储器
    - 如果是 **InMemorySaver**，不要重建检查点存储器对象，否则历史检查点丢失，无法恢复
    - 如果希望跨进程、服务重启后仍可恢复，应使用基于 **SQLite、Postgres** 等持久化检查点后端的存储器
  - 2、再次运行时用 **None** 作为计算图的状态输入
  - 3、传递的配置信息应包含 **thread_id** 而不能包含 **checkpoint_id**

- 此时，**LangGraph** 会根据 **thread_id** 从 **checkpointer** 读取该会话的最新检查点，并从该检查点继续推进。

- 如果某个超步中有多个并行任务，其中一部分任务成功，另一部分任务失败，那么已经成功完成的任务结果会作为 **pending_writes** 被保存。恢复运行时，已经成功完成的任务不会重复执行



#### 5.3.2 模拟失败

- 本例使用如下结构：

~~~bash
START
  -> node_change_topic
      -> node_poem
      -> node_joke
  -> node_output
  -> END
~~~

- 运行时，**node_change_topic** 会把输入主题 **"猫咪"** 改写为 **"猫咪: 布偶猫"**，然后并行执行 **node_poem** 和 **node_joke**，最后由 **node_output** 汇总输出

~~~python
import os
import time
from typing import TypedDict
from rich import print as rprint

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage


from dotenv import load_dotenv
from loguru import logger

load_dotenv(override=True)


CONTENT_TYPES = ["poem", "ci_poem", "joke"]

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

#1. 声明状态
class OverAllState(TypedDict):
    topic:str
    poem:str
    joke:str
    final_output:str

#1.1 输入状态
class InputState(TypedDict):
    topic:str

#1.2 输出状态
class OutputState(TypedDict):
    final_output:str

topics = ["布偶猫","狸花猫","金渐层"]
topic_index = 0

#2. 定义节点
def node_change_topic(state:InputState)->OverAllState:
    global topic_index
    logger.info("topic_index:{}",topic_index)
    sub_topic = topics[topic_index]
    topic_index += 1
    topic_index %= len(topics)

    return {
        "topic":f"{state["topic"]}:{sub_topic}"
    }

#2.1 同一个超步的位置 成功运行的节点
def node_poem(state:OverAllState) -> OverAllState:
    logger.info("node_poem正在执行")
    topic = state["topic"]
    poem = model.invoke([HumanMessage(f"写一首关于{topic}主题的七言绝句")]).content
    return {
        "poem":poem
    }

#2.2 同一个超步的位置 运行失败的节点
def node_joke(state:OverAllState) -> OverAllState:
    logger.info("node_joke正在执行")
    topic = state["topic"]
    time.sleep(5)
    raise Exception("人为抛异常")
    joke = model.invoke([HumanMessage(f"写一首关于{topic}主题的笑话")]).content
    return {
        "joke":joke
    }

def node_output(state:OverAllState) -> OutputState:
    logger.info("node_output正在执行")
    topic = state["topic"]
    poem = state["poem"]
    joke = state["joke"]
    final_output = f"关于{topic}的七言绝句:{poem}\n 笑话:{joke}\n"
    return {
        "final_output":final_output
    }

#3. 构建图
builder = StateGraph(state_schema=OverAllState,input_schema=InputState,output_schema=OutputState)

#3.1 添加节点
builder.add_node("node_change_topic",node_change_topic)
builder.add_node("node_poem",node_poem)
builder.add_node("node_joke",node_joke)
builder.add_node("node_output",node_output)

#3.2 添加边
builder.add_edge(START,"node_change_topic")
builder.add_edge("node_change_topic","node_poem")
builder.add_edge("node_change_topic","node_joke")
builder.add_edge("node_poem","node_output")
builder.add_edge("node_joke","node_output")
builder.add_edge("node_output",END)

#4. 添加检查点后端
DB_URL = "postgresql://postgres:123456@172.20.1.36:5432/lzy?sslmode=disable"
from langgraph.checkpoint.postgres import PostgresSaver
with PostgresSaver.from_conn_string(DB_URL) as checkpointer:
    #5. 第一次使用PostgresSaver作为检查点 需要调用方法 setup()
    #checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer)

    config = {
        "configurable":{
            "thread_id":"chapter06"
        }
    }

    history_checkpoints = list(graph.get_state_history(config=config))
    rprint(history_checkpoints)

    raw_mermaid = graph.get_graph().draw_mermaid()
    print(raw_mermaid)

    res = graph.invoke({"topic":"猫"},config=config)
    print(res)

~~~

- 运行结果

~~~python
2026-06-10 16:17:43.544 | INFO     | __main__:node_change_topic:45 - topic_idx: 0
2026-06-10 16:17:43.545 | INFO     | __main__:node_joke:65 - node_joke 已执行
2026-06-10 16:17:43.548 | INFO     | __main__:node_poem:54 - node_poem 已执行
>Traceback...
Exception: 人为中断
During task with name 'node_joke' and id 'edb59019-4808-ed51-68eb-d5fba8696b13'
~~~

~~~mermaid
graph TD;
	__start__([<p>__start__</p>]):::first
	node_change_topic(node_change_topic)
	node_poem(node_poem)
	node_joke(node_joke)
	node_output(node_output)
	__end__([<p>__end__</p>]):::last
	__start__ --> node_change_topic;
	node_change_topic --> node_joke;
	node_change_topic --> node_poem;
	node_joke --> node_output;
	node_poem --> node_output;
	node_output --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
~~~

- 本例在 **node_joke** 中抛出异常，从而模拟计算图执行失败的场景。

  - 为了观察恢复执行时哪些节点会被重新执行，示例中额外定义了全局变量 **topics** 和 **topic_idx**。**node_change_topic** 每执行一次，都会从 **topics** 中取出一个新的子主题，作为状态字段 **topic** 的值。

  - 如果恢复运行时 **node_change_topic** 没有再次执行，则最终结果仍然会使用第一次生成的主题，即 **猫咪: 布偶猫**。如果该节点被重新执行，则主题可能会变成 **猫咪: 狸花猫** 或 **猫咪: 金渐层**

~~~json
[
    StateSnapshot(
        values={'topic': '猫:布偶猫'},
        next=('node_poem', 'node_joke'),
        config={
            'configurable': {
                'thread_id': 'chapter',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d7675-ce7e-6113-8001-7660b53c2005'
            }
        },
        metadata={'step': 1, 'source': 'loop', 'parents': {}},
        created_at='2026-11-03T09:17:16.549959+00:00',
        parent_config={
            'configurable': {
                'thread_id': 'chapter',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d7675-ce7a-6f12-8000-af78344d972d'
            }
        },
        tasks=(
            PregelTask(
                id='ea9aa29f-bd99-4ad2-8140-c68dda2b0849',
                name='node_poem',
                path=('__pregel_pull', 'node_poem'),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'poem': '《咏布偶猫》\n雪绒蓝眼态温纯，  \n软倚人怀不避身。
\n仿佛案头闲布偶，  \n却藏灵性惹人亲。'
                }
            ),
            PregelTask(
                id='2ae4f7af-b5f2-c0e8-2ab4-86b9d8601808',
                name='node_joke',
                path=('__pregel_pull', 'node_joke'),
                error="Exception('人为抛异常')",
                interrupts=(),
                state=None,
                result=None
            )
        ),
        interrupts=()
    ),
    StateSnapshot(
        values={'topic': '猫'},
        next=('node_change_topic',),
        config={
            'configurable': {
                'thread_id': 'chapter',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d7675-ce7a-6f12-8000-af78344d972d'
            }
        },
        metadata={'step': 0, 'source': 'loop', 'parents': {}},
        created_at='2026-11-03T09:17:16.548678+00:00',
        parent_config={
            'configurable': {
                'thread_id': 'chapter',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d7675-ce78-673e-bfff-a629e132545f'
            }
        },
        tasks=(
            PregelTask(
                id='50321714-b00c-439d-c77f-d2ae8839e578',
                name='node_change_topic',
                path=('__pregel_pull', 'node_change_topic'),
                error=None,
                interrupts=(),
                state=None,
                result={'topic': '猫:布偶猫'}
            ),
        ),
        interrupts=()
    ),
    StateSnapshot(
        values={},
        next=('__start__',),
        config={
            'configurable': {
                'thread_id': 'chapter',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d7675-ce78-673e-bfff-a629e132545f'
            }
        },
        metadata={'step': -1, 'source': 'input', 'parents': {}},
        created_at='2026-11-03T09:17:16.547660+00:00',
        parent_config=None,
        tasks=(
            PregelTask(
                id='70346cff-3f7f-51c1-3919-49f055332391',
                name='__start__',
                path=('__pregel_pull', '__start__'),
                error=None,
                interrupts=(),
                state=None,
                result={'topic': '猫'}
            ),
        ),
        interrupts=()
    )
]
~~~

- 可以看到，失败后的检查点中（**step=1**）记录了当前超步相关任务的执行情况：
  - **node_poem** 执行成功，结果记录在 **result** 字段中。
  - **node_joke** 执行失败，**result** 为空，异常信息记录在 **error** 字段中。
- 本例中，**node_poem** 和 **node_joke** 属于同一个超步。虽然 **node_joke** 失败了，但 **node_poem** 已成功，其计算结果会被持久化机制保存下来。因此，后续恢复运行时，**node_poem** 不会重复执行



#### 5.3.3 修复BUG之后恢复运行

~~~python
import os
import time
from typing import TypedDict
from rich import print as rprint

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langchain.messages import HumanMessage


from dotenv import load_dotenv
from loguru import logger

load_dotenv(override=True)


CONTENT_TYPES = ["poem", "ci_poem", "joke"]

CHAT_API_KEY=os.getenv("CHAT_API_KEY")
CHAT_BASE_URL=os.getenv("CHAT_BASE_URL")
CHAT_MODEL=os.getenv("CHAT_MODEL")

model = init_chat_model(
    base_url=CHAT_BASE_URL,
    model_provider="openai",
    api_key=CHAT_API_KEY,
    model=CHAT_MODEL
)

#1. 声明状态
class OverAllState(TypedDict):
    topic:str
    poem:str
    joke:str
    final_output:str

#1.1 输入状态
class InputState(TypedDict):
    topic:str

#1.2 输出状态
class OutputState(TypedDict):
    final_output:str

topics = ["布偶猫","狸花猫","金渐层"]
topic_index = 0

#2. 定义节点
def node_change_topic(state:InputState)->OverAllState:
    global topic_index
    logger.info("topic_index:{}",topic_index)
    sub_topic = topics[topic_index]
    topic_index += 1
    topic_index %= len(topics)

    return {
        "topic":f"{state["topic"]}:{sub_topic}"
    }

#2.1 同一个超步的位置 成功运行的节点
def node_poem(state:OverAllState) -> OverAllState:
    logger.info("node_poem正在执行")
    topic = state["topic"]
    poem = model.invoke([HumanMessage(f"写一首关于{topic}主题的七言绝句")]).content
    return {
        "poem":poem
    }

#2.2 同一个超步的位置 运行失败的节点
def node_joke(state: InputState) -> OverAllState:
    logger.info(f"node_joke 已执行")
    topic = state["topic"]

    joke = model.invoke([HumanMessage(f"写一个关于 {topic} 的笑话")]).content

    return {
        "joke": joke
    }

def node_output(state:OverAllState) -> OutputState:
    logger.info("node_output正在执行")
    topic = state["topic"]
    poem = state["poem"]
    joke = state["joke"]
    final_output = f"关于{topic}的七言绝句:{poem}\n 笑话:{joke}\n"
    return {
        "final_output":final_output
    }

#3. 构建图
builder = StateGraph(state_schema=OverAllState,input_schema=InputState,output_schema=OutputState)

#3.1 添加节点
builder.add_node("node_change_topic",node_change_topic)
builder.add_node("node_poem",node_poem)
builder.add_node("node_joke",node_joke)
builder.add_node("node_output",node_output)

#3.2 添加边
builder.add_edge(START,"node_change_topic")
builder.add_edge("node_change_topic","node_poem")
builder.add_edge("node_change_topic","node_joke")
builder.add_edge("node_poem","node_output")
builder.add_edge("node_joke","node_output")
builder.add_edge("node_output",END)

#4. 添加检查点后端
DB_URL = "postgresql://postgres:123456@172.20.1.36:5432/lzy?sslmode=disable"
from langgraph.checkpoint.postgres import PostgresSaver
with PostgresSaver.from_conn_string(DB_URL) as checkpointer:
    #5. 第一次使用PostgresSaver作为检查点 需要调用方法 setup()
    #checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer)

    config = {
        "configurable":{
            "thread_id":"1111111"
        }
    }

    raw_mermaid = graph.get_graph().draw_mermaid()
    print(raw_mermaid)

    res = graph.invoke(None, config=config)
    print(res)

    history_checkpoints = list(graph.get_state_history(config=config))
    rprint(history_checkpoints)

~~~

- 这里需要注意两点：
  1. <font color='red'>**重新编译计算图时，仍然使用原来的 checkpointer** 对象</font>
  2. <font color='red'>**恢复运行时，输入参数传入 None，并且配置中只传入 thread_id**</font>

- 输出

~~~python
2026-11-03 17:21:55.053 | INFO     | __main__:node_joke:71 - node_joke 已执行
2026-11-03 17:22:14.315 | INFO     | __main__:node_output:81 - node_output正在执行
{'final_output': '关于猫:布偶猫的七言绝句:《布偶猫》\n蓝眸似海雪绒身，软玉温香倚膝亲。\n莫道柔躯无傲骨，跃上花台也动人。\n 笑话:主人抱起布偶猫：“你怎么又躺平？一点猫样都没有！”\n\n布偶猫懒洋洋地说：“我这是敬业。品种说明书写了——布偶，布偶，随你摆布。”\n\n主人：“那你至少自己走两步？”\n\n布偶猫：“可以，但得你抱着我走。” 🐾\n'}
~~~

- 运行结果中可以观察到：

  - **node_joke** 被重新执行。
  - **node_output** 被执行。
  - **node_poem** 没有被重新执行。

  - 最终输出中的七言绝句和历史检查点中 **node_poem.result** 记录的内容一致，说明 **node_poem** 的执行结果被成功复用了

~~~json
[
    StateSnapshot(
        values={
            'topic': '猫:布偶猫',
            'poem': 
'《布偶猫》\n蓝眸似海雪绒身，软玉温香倚膝亲。\n莫道柔躯无傲骨，跃上花台也动人。
',
            'joke': 
'主人抱起布偶猫：“你怎么又躺平？一点猫样都没有！”\n\n布偶猫懒洋洋地说：“我这是
敬业。品种说明书写了——布偶，布偶，随你摆布。”\n\n主人：“那你至少自己走两步？”\n
\n布偶猫：“可以，但得你抱着我走。” 🐾',
            'final_output': 
'关于猫:布偶猫的七言绝句:《布偶猫》\n蓝眸似海雪绒身，软玉温香倚膝亲。\n莫道柔躯
无傲骨，跃上花台也动人。\n 
笑话:主人抱起布偶猫：“你怎么又躺平？一点猫样都没有！”\n\n布偶猫懒洋洋地说：“我
这是敬业。品种说明书写了——布偶，布偶，随你摆布。”\n\n主人：“那你至少自己走两步
？”\n\n布偶猫：“可以，但得你抱着我走。” 🐾\n'
        },
        next=(),
        config={
            'configurable': {
                'thread_id': '1111111',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d7680-e638-6cec-8003-5a2729436b0c'
            }
        },
        metadata={'step': 3, 'source': 'loop', 'parents': {}},
        created_at='2026-11-03T09:22:14.317169+00:00',
        parent_config={
            'configurable': {
                'thread_id': '1111111',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d7680-e632-6b5a-8002-f17d3fb205ce'
            }
        },
        tasks=(),
        interrupts=()
    ),
    StateSnapshot(
        values={
            'topic': '猫:布偶猫',
            'poem': 
'《布偶猫》\n蓝眸似海雪绒身，软玉温香倚膝亲。\n莫道柔躯无傲骨，跃上花台也动人。
',
            'joke': 
'主人抱起布偶猫：“你怎么又躺平？一点猫样都没有！”\n\n布偶猫懒洋洋地说：“我这是
敬业。品种说明书写了——布偶，布偶，随你摆布。”\n\n主人：“那你至少自己走两步？”\n
\n布偶猫：“可以，但得你抱着我走。” 🐾'
        },
        next=('node_output',),
        config={
            'configurable': {
                'thread_id': '1111111',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d7680-e632-6b5a-8002-f17d3fb205ce'
            }
        },
        metadata={'step': 2, 'source': 'loop', 'parents': {}},
        created_at='2026-11-03T09:22:14.314665+00:00',
        parent_config={
            'configurable': {
                'thread_id': '1111111',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d767f-16b9-60a3-8001-591363d4bae0'
            }
        },
        tasks=(
            PregelTask(
                id='3e35d3ab-15b9-6d1d-7f0e-14827f977911',
                name='node_output',
                path=('__pregel_pull', 'node_output'),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'final_output': 
'关于猫:布偶猫的七言绝句:《布偶猫》\n蓝眸似海雪绒身，软玉温香倚膝亲。\n莫道柔躯
无傲骨，跃上花台也动人。\n 
笑话:主人抱起布偶猫：“你怎么又躺平？一点猫样都没有！”\n\n布偶猫懒洋洋地说：“我
这是敬业。品种说明书写了——布偶，布偶，随你摆布。”\n\n主人：“那你至少自己走两步
？”\n\n布偶猫：“可以，但得你抱着我走。” 🐾\n'
                }
            ),
        ),
        interrupts=()
    ),
    StateSnapshot(
        values={'topic': '猫:布偶猫'},
        next=('node_poem', 'node_joke'),
        config={
            'configurable': {
                'thread_id': '1111111',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d767f-16b9-60a3-8001-591363d4bae0'
            }
        },
        metadata={'step': 1, 'source': 'loop', 'parents': {}},
        created_at='2026-11-03T09:21:25.715772+00:00',
        parent_config={
            'configurable': {
                'thread_id': '1111111',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d767f-16b6-6e01-8000-edf5b222e006'
            }
        },
        tasks=(
            PregelTask(
                id='d46918c6-f2ac-9ff6-0027-b4f768eedf20',
                name='node_poem',
                path=('__pregel_pull', 'node_poem'),
                error=None,
                interrupts=(),
                state=None,
                result={
                    'poem': 
'《布偶猫》\n蓝眸似海雪绒身，软玉温香倚膝亲。\n莫道柔躯无傲骨，跃上花台也动人。
'
                }
            ),
            PregelTask(
                id='3afa749b-50ff-4a07-78b6-d232623abcf5',
                name='node_joke',
                path=('__pregel_pull', 'node_joke'),
                error="Exception('人为抛异常')",
                interrupts=(),
                state=None,
                result={
                    'joke': 
'主人抱起布偶猫：“你怎么又躺平？一点猫样都没有！”\n\n布偶猫懒洋洋地说：“我这是
敬业。品种说明书写了——布偶，布偶，随你摆布。”\n\n主人：“那你至少自己走两步？”\n
\n布偶猫：“可以，但得你抱着我走。” 🐾'
                }
            )
        ),
        interrupts=()
    ),
    StateSnapshot(
        values={'topic': '猫'},
        next=('node_change_topic',),
        config={
            'configurable': {
                'thread_id': '1111111',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d767f-16b6-6e01-8000-edf5b222e006'
            }
        },
        metadata={'step': 0, 'source': 'loop', 'parents': {}},
        created_at='2026-11-03T09:21:25.714885+00:00',
        parent_config={
            'configurable': {
                'thread_id': '1111111',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d767f-16b4-6cae-bfff-8211ccb5e514'
            }
        },
        tasks=(
            PregelTask(
                id='43c6ccae-6930-52e7-683d-d93a6983b33e',
                name='node_change_topic',
                path=('__pregel_pull', 'node_change_topic'),
                error=None,
                interrupts=(),
                state=None,
                result={'topic': '猫:布偶猫'}
            ),
        ),
        interrupts=()
    ),
    StateSnapshot(
        values={},
        next=('__start__',),
        config={
            'configurable': {
                'thread_id': '1111111',
                'checkpoint_ns': '',
                'checkpoint_id': '1f1d767f-16b4-6cae-bfff-8211ccb5e514'
            }
        },
        metadata={'step': -1, 'source': 'input', 'parents': {}},
        created_at='2026-11-03T09:21:25.714034+00:00',
        parent_config=None,
        tasks=(
            PregelTask(
                id='71dc9548-473d-d562-a4c7-bd1a0828cc26',
                name='__start__',
                path=('__pregel_pull', '__start__'),
                error=None,
                interrupts=(),
                state=None,
                result={'topic': '猫'}
            ),
        ),
        interrupts=()
    )
]
~~~



### 5.4 Time Travel

- **Time Travel** 直译为 **时间旅行**，在当前场景下太过生硬，从技术实现和使用场景来看，译为 **检查点回溯** 更为合理。

- 所谓**检查点回溯**，是指基于某个历史检查点，重新执行后续流程，或者在该检查点基础上修改状态并创建新的执行分支。

- **检查点回溯**有两种形式，根据是否更改历史状态区分：
  - **Replay**：检查点重放，回到某个历史检查点，沿着原先的执行路径重新执行后续节点。
  - **Fork**：检查点分叉，回到某个历史检查点，修改状态，从该位置创建一条新的执行分支。

- 二者的共同点是：

> 检查点之前的节点不会重新执行，检查点之后的节点会重新执行。

- 二者的区别是：

> **Replay** 不修改历史状态；**Fork** 会基于历史检查点应用新的状态更新，并创建新的检查点分支。















































































































# 十一、流式执行

## 1、概述

### 1.1 什么是流式输出

- **流式执行（Streaming Execution）**，<font color='red'>**是指程序在任务尚未全部完成时，就将执行过程中已经产生的中间结果、状态变化或事件持续输出给调用方，而不是等待整个任务结束后一次性返回最终结果**</font>

- 简而言之：流式执行就是

> 边执行、边产生事件、边向调用方返回结果。



### 1.2. 什么是LangGraph的流式执行

- **LangGraph** 的流式执行是指，<font color='red'>**状态图计算过程中，将节点输出、状态更新、消息增量、自定义事件或调试信息等写入流式队列，并在特定的时机将流式队列中的信息返还给调用者**</font>

- 在同步执行中，这些数据最终进入内部的**同步流式队列**；在异步执行中，则进入对应的**异步队列**。调用方通过**迭代器**逐条消费这些数据，因此不必等到整张图执行完毕后再获得反馈



### 1.3. LangGraph的流式执行API

#### 1.3.1 stream/astream

- 获取 **LangGraph** 执行过程中的业务数据或运行时数据
  - **stream**：同步 **API**
  - **astream**：异步 **API**

- 调用方通过 **stream_mode** 选择需要消费的内容，例如状态、节点更新、模型消息、自定义数据、检查点或任务事件。这是本章的重点。

- **stream与astream的区别**

| 维度       | stream()                                | astream()                                      |
| ---------- | --------------------------------------- | ---------------------------------------------- |
| 调用方式   | 同步迭代 for chunk in graph.stream(...) | 异步迭代 async for chunk in graph.astream(...) |
| 运行环境   | 普通 Python 脚本                        | asyncio 事件循环（Jupyter、FastAPI 等）        |
| 参数与输出 | 相同 stream_mode，语义一致              | 相同 stream_mode，语义一致                     |
| 适用场景   | 同步批处理、CLI 工具                    | async 服务端、结合其他异步 IO                  |



#### 1.3.2 astream_events

- 获取图运行过程中产生的 **Runnable** 标准事件（组件生命周期、父子调用关系、输入输出等）。详见 **11.3. astream_events**



## 2、stream/astream

- **astream** 是 **stream** 的异步版本。二者使用相同的流模式，且在参数一致时，输出数据的语义和封装格式一致；<font color='red'>**区别主要在于调用方分别使用同步迭代和异步迭代进行消费**</font>

- 前面的章节都是通过 **invoke** 调用状态图。实际上，**invoke** 会在内部消费 **stream**，并把流式执行结果汇总为最终返回值；**ainvoke** 与 **astream** 的关系同理



### 2.1 输出格式版本说明

- 从 **LangGraph 1.1** 开始，**stream/astream** 支持两种输出格式版本：

- v1：当前默认格式。输出形态会随参数变化：
  - 单个流模式通常直接返回该模式的数据；
  - 多个流模式返回 **(mode, data)**；
  - 启用子图流式输出后，还会增加命名空间信息。
- **v2**：在 **v1** 基础上做了封装，统一返回 **StreamPart** 字典，固定包含 **type**、**ns** 和 **data** 三个字段，增强了输出的可读性，简化了解析难度、并补充了校验机制。

- 本节基于 **v1** 展开。示例刻意把 **stream_mode** 写成列表，即使每次只指定一种模式，也会得到 **(mode, data)** 二元组，便于观测模式名称，并且可以无缝扩展到指定多种模式的情况






































https://xbsheng.github.io/atguigu-note/langgraph/%E8%AF%BE%E4%BB%B6/01-LangGraph%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8#_3-2-3-%E9%BB%98%E8%AE%A4%E8%A1%8C%E4%B8%BA
