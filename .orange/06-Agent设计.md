# 06 Agent 设计

## 6.1 create_react_agent（已废弃，但仍常用）

```python
# libs/langgraph/langgraph/prebuilt/chat_agent_executor.py

from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    model=ChatAnthropic(model="claude-sonnet-4-5"),
    tools=[search_tool, calculator_tool],
    state_schema=AgentState,
    state_modifier="You are a helpful assistant",
    checkpointer=InMemorySaver(),
)
```

**内部构建的图结构:**
```
START → "agent" → 条件路由
                    ├── 有工具调用 → "tools" → "agent"
                    └── 无工具调用 → END
```

**已废弃原因:** 不支持中间件（middleware），无法注入 AgentMiddleware 钩子。新版 `create_agent` 在 `langchain_v1` 中。

---

## 6.2 ToolNode 设计

```python
# libs/langgraph/langgraph/prebuilt/tool_node.py

class ToolNode:
    def __init__(self, tools: list[BaseTool | Callable]):
        self.tools_by_name = {t.name: t for t in tools}

    async def __call__(self, state: dict, config: RunnableConfig) -> dict:
        messages = state["messages"]
        last_message = messages[-1]

        # 并行执行所有工具调用（asyncio.gather）
        results = await asyncio.gather(*[
            self._arun_tool(tc, config)
            for tc in last_message.tool_calls
        ])

        return {"messages": results}

    async def _arun_tool(self, tool_call: ToolCall, config) -> ToolMessage:
        tool = self.tools_by_name[tool_call["name"]]
        try:
            output = await tool.ainvoke(tool_call["args"], config)
            return ToolMessage(
                content=str(output),
                name=tool_call["name"],
                tool_call_id=tool_call["id"],
            )
        except Exception as e:
            return ToolMessage(
                content=f"Error: {e}",
                name=tool_call["name"],
                tool_call_id=tool_call["id"],
                status="error",
            )
```

**ToolCallWrapper**: 对 ToolNode 中的单个工具调用进行封装，支持：
- 重试（retries 参数）
- 错误处理（fallback）
- 并行执行

---

## 6.3 多 Agent 架构

### 方式一：子图嵌套

```python
# 研究 Agent
research_graph = StateGraph(ResearchState)
research_graph.add_node(...)
research_compiled = research_graph.compile()

# 写作 Agent
writing_graph = StateGraph(WritingState)

# 父图将子图作为节点
parent_graph = StateGraph(ParentState)
parent_graph.add_node("research", research_compiled)
parent_graph.add_node("writing", writing_compiled)
parent_graph.add_edge("research", "writing")
```

**关键**: 子图有自己独立的 State 类型，通过"输入转换"和"输出提取"与父图通信。

### 方式二：Send API（动态并行）

```python
def coordinator(state: State) -> list[Send]:
    """基于状态动态决定并行任务"""
    tasks = state["pending_tasks"]
    return [
        Send("worker_agent", {"task": task, "worker_id": i})
        for i, task in enumerate(tasks)
    ]

graph.add_conditional_edges("coordinator", coordinator)
```

**使用场景**: Map-Reduce 模式，任务数量在编译时未知。

### 方式三：Supervisor 模式

```python
class SupervisorState(TypedDict):
    messages: Annotated[list, add_messages]
    next: str  # 下一个要执行的 worker

def supervisor(state: SupervisorState) -> dict:
    """LLM 决定下一个 worker"""
    response = llm_with_tools.invoke(state["messages"])
    return {"next": response.tool_calls[0]["name"]}

graph.add_conditional_edges("supervisor", lambda s: s["next"])
```

---

## 6.4 Runtime Context（只读依赖注入）

```python
from langgraph.types import Runtime
from dataclasses import dataclass

@dataclass
class MyContext:
    user_id: str
    permissions: list[str]

def agent_node(state: State, *, runtime: Runtime[MyContext]):
    # 只读访问，不能修改
    user_id = runtime.context.user_id

# 编译时声明
graph = builder.compile(context_schema=MyContext)

# 运行时注入
await graph.ainvoke(
    input,
    config={"context": MyContext(user_id="u123", permissions=["read"])}
)
```

**设计意图**: 避免将认证信息、数据库连接等放入状态（状态会被持久化），只读约束防止节点修改运行时配置。

---

## 6.5 Human-in-the-Loop 设计模式

### 审批模式

```python
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["execute_action"],  # 执行前等待人类审批
)

# 运行到审批点
await graph.ainvoke(input, config=config)
# → GraphInterrupt 抛出

# 人类审批
state = graph.get_state(config)
# state.next = ("execute_action",)

# 批准：直接继续
await graph.ainvoke(None, config=config)

# 拒绝：修改状态后继续
graph.update_state(config, {"action": "modified_action"})
await graph.ainvoke(None, config=config)
```

### 反馈注入模式

```python
# 执行后中断，注入人类反馈
graph = builder.compile(
    interrupt_after=["draft"],
)

# 草稿生成后暂停
await graph.ainvoke(input, config=config)

# 注入人类反馈
graph.update_state(
    config,
    {"messages": [HumanMessage(content="Make it more concise")]}
)

# 继续修改
await graph.ainvoke(None, config=config)
```

---

## 6.6 错误处理和重试

```python
from langgraph.pregel.retry import RetryPolicy

# 节点级别重试
graph.add_node(
    "flaky_api_call",
    flaky_fn,
    retry=RetryPolicy(max_attempts=3, backoff_factor=2.0)
)
```

**RetryPolicy 参数**:
- `max_attempts`: 最大重试次数
- `backoff_factor`: 指数退避系数
- `retry_on`: 哪些异常触发重试（默认所有异常）
