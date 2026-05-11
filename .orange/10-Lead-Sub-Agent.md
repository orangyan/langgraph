# 10 Lead-Sub-Agent 架构

## LangGraph 的多 Agent 支持

LangGraph 提供两种原生多 Agent 机制：子图嵌套（静态结构）和 Send API（动态并行）。

---

## 10.1 子图嵌套（Subgraph Pattern）

```python
# 定义子 Agent
research_builder = StateGraph(ResearchState)
research_builder.add_node("search", search_fn)
research_builder.add_node("summarize", summarize_fn)
research_builder.add_edge("search", "summarize")
research_builder.add_edge("summarize", END)
research_graph = research_builder.compile()

# 在父图中使用
parent_builder = StateGraph(ParentState)

# 子图作为父图的一个节点
parent_builder.add_node("research", research_graph)
parent_builder.add_node("write", write_fn)
parent_builder.add_edge("research", "write")

parent_graph = parent_builder.compile(checkpointer=checkpointer)
```

**状态类型处理（父子图状态不同类型）:**
```python
# 方式一：子图使用父图状态的子集
# 父图 State: {messages, research_result, final_output}
# 子图 State: {messages, research_result}  ← 只包含需要的字段

# 方式二：显式转换函数
def transform_input(parent_state: ParentState) -> ResearchState:
    return {"query": parent_state["messages"][-1].content}

def transform_output(research_output: ResearchState, parent_state: ParentState) -> ParentState:
    return {"research_result": research_output["result"]}

parent_builder.add_node(
    "research",
    lambda s: transform_output(research_graph.invoke(transform_input(s)), s)
)
```

---

## 10.2 Send API（动态并行 Map-Reduce）

```python
from langgraph.types import Send

# Coordinator 节点动态生成并行任务
def coordinator(state: State) -> list[Send]:
    documents = state["documents"]
    return [
        Send("summarize_doc", {"doc_id": i, "content": doc})
        for i, doc in enumerate(documents)
    ]

# Worker 节点处理单个任务
def summarize_doc(state: dict) -> dict:
    summary = llm.invoke(f"Summarize: {state['content']}")
    return {"summaries": [summary.content]}

# Reducer 节点汇总结果
def reducer(state: State) -> dict:
    all_summaries = state["summaries"]  # BinaryOperatorAggregate 自动聚合
    final = llm.invoke(f"Combine these summaries: {all_summaries}")
    return {"final_summary": final.content}

# 图结构
graph.add_node("coordinator", coordinator)
graph.add_node("summarize_doc", summarize_doc)
graph.add_node("reducer", reducer)
graph.add_conditional_edges("coordinator", coordinator)  # 返回 Send 列表
graph.add_edge("summarize_doc", "reducer")
```

---

## 10.3 Supervisor 模式（LLM 协调器）

```python
class SupervisorState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    next_worker: str

# Worker agents
workers = {
    "researcher": research_graph,
    "writer": writing_graph,
    "reviewer": review_graph,
}

def supervisor(state: SupervisorState) -> dict:
    """LLM 决定下一步调用哪个 worker"""
    response = llm_with_structured_output.invoke(
        state["messages"],
        output_schema={"next_worker": Literal["researcher", "writer", "reviewer", "FINISH"]}
    )
    return {"next_worker": response["next_worker"]}

def route(state: SupervisorState) -> str:
    return state["next_worker"]  # "FINISH" → END

builder.add_conditional_edges("supervisor", route, {
    "researcher": "researcher",
    "writer": "writer",
    "reviewer": "reviewer",
    "FINISH": END,
})
```

---

## 10.4 与其他项目的 Sub-Agent 对比

| 维度 | LangGraph | deer-flow | hermes-agent | nanobot | gstack |
|------|---------|-----------|--------------|---------|--------|
| **派生机制** | 子图嵌套 + Send API | `delegate_task` 工具 | `task` 工具 | `spawn` 函数 | Conductor 并行工作区 |
| **任务分配** | 静态（图结构）+ 动态（Send） | 动态（LLM 决策） | 动态（LLM 决策） | 动态（代码调用） | 动态（用户决策） |
| **状态共享** | 共享图状态（Channels） | 事件总线 | 消息传递 | 直接引用 | 文件系统 |
| **并行度** | 任意（Send API） | 有限（事件总线） | 有限（asyncio） | 无限制 | 10-15（Conductor） |
| **子任务持久化** | ✅ (子图 checkpoint) | ✅ (LangGraph 继承) | ✅ (SQLite) | ❌ | ✅ (git) |
| **跨模型** | ⚠️ (需手动配置不同 LLM) | ❌ | ❌ | ❌ | ✅ (pair-agent) |
| **可视化** | ✅ (LangGraph Studio) | ❌ | ✅ (WandB) | ❌ | ❌ |

---

## 10.5 deer-flow 中的 LangGraph 多 Agent 实现

deer-flow 是基于 LangGraph 构建的，其 Lead-Sub-Agent 架构直接使用了 LangGraph 的子图特性：

```python
# deer-flow/backend/src/deerflow/agents/lead_agent/agent.py

# Lead Agent: 协调整体流程
lead_graph = StateGraph(LeadAgentState)
lead_graph.add_node("planner", planner_fn)
lead_graph.add_node("executor", sub_agent_graph)  # 子图作为节点
lead_graph.add_node("reporter", reporter_fn)

# Sub Agent: 执行具体任务
sub_agent_graph = StateGraph(SubAgentState)
sub_agent_graph.add_node("researcher", researcher_fn)
sub_agent_graph.add_node("coder", coder_fn)
```

这展示了 LangGraph 子图特性在实际产品中的应用。
