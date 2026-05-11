# 12 Thinking 模式

## LangGraph 中的 Extended Thinking

LangGraph 框架本身不直接处理 Extended Thinking——它通过 LLM 集成传递 thinking 相关参数。

---

## 12.1 在 LangGraph 节点中启用 Extended Thinking

```python
from langchain_anthropic import ChatAnthropic

# 使用支持 Extended Thinking 的模型
llm = ChatAnthropic(
    model="claude-opus-4-5",
    thinking={"type": "enabled", "budget_tokens": 10000},
)

def agent_node(state: State):
    response = llm.invoke(state["messages"])
    # response 包含 thinking content blocks
    return {"messages": [response]}

graph = builder.compile()
```

---

## 12.2 Thinking Content 在状态中的流转

```python
class AgentState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]

# thinking content 作为 AIMessage 的一部分
# AIMessage.content = [
#     {"type": "thinking", "thinking": "Let me analyze..."},
#     {"type": "text", "text": "My answer is..."}
# ]

def agent_node(state: AgentState):
    response = llm_with_thinking.invoke(state["messages"])
    # 整个 AIMessage（含 thinking）存入状态
    return {"messages": [response]}
```

---

## 12.3 流式获取 Thinking Content

```python
async for chunk in graph.astream(
    input, config=config, stream_mode="messages"
):
    if hasattr(chunk, "type"):
        if chunk.type == "thinking":
            # Extended Thinking 内容（流式）
            print(f"[Thinking]: {chunk.thinking}")
        elif chunk.type == "text":
            # 最终回复（流式）
            print(chunk.text, end="")
```

---

## 12.4 与其他项目的 Thinking 模式对比

| 维度 | LangGraph | deer-flow | hermes-agent | nanobot | gstack |
|------|---------|-----------|--------------|---------|--------|
| **Extended Thinking** | ✅ (通过 LLM 参数) | ✅ (budget_tokens) | ✅ (reasoning_effort) | ✅ (thinking_content) | ❌ (文字指令替代) |
| **框架级控制** | ❌ (由 LLM 层处理) | ✅ (middleware 传入) | ✅ (agent_loop 传入) | ✅ (配置传入) | ❌ |
| **Thinking 持久化** | ✅ (存入 messages Channel) | ✅ (存入 messages) | ✅ (reasoning_per_turn) | ✅ (thinking_content) | ❌ |
| **流式 Thinking** | ✅ (stream_mode="messages") | ✅ | ✅ | ❌ | ❌ |
| **Budget 控制** | ✅ (budget_tokens 参数) | ✅ | ✅ | ✅ | ❌ |
| **Adaptive Thinking** | ✅ (type="auto") | ✅ | ❌ | ❌ | ❌ |

---

## 12.5 Adaptive Thinking（自适应）

```python
# 自适应 thinking：模型自己决定是否思考以及思考多少
llm = ChatAnthropic(
    model="claude-opus-4-5",
    thinking={"type": "auto"},  # 自动决定
)
```

与固定 `budget_tokens` 相比，`auto` 模式让模型自行判断每个请求的思考深度——复杂问题多思考，简单问题少思考，更经济。

---

## 12.6 Thinking 与 ToolNode 的交互

当 LLM 在思考后决定调用工具时，thinking content 会保留在消息历史中：

```
消息历史:
  HumanMessage: "What's 2+2?"
  AIMessage:
    content: [
      {"type": "thinking", "thinking": "Simple arithmetic..."},
      {"type": "tool_use", "name": "calculator", "input": {"expr": "2+2"}}
    ]
  ToolMessage: {"result": 4}
  AIMessage: "The answer is 4."
```

**Checkpointer 会保存整个消息历史**，包括 thinking content。这对于调试和分析很有价值，但会增加存储成本。

---

## 12.7 在 deer-flow 中的实际应用

deer-flow 基于 LangGraph，通过 Middleware 统一管理 thinking 配置：

```python
# deer-flow 中的 thinking middleware
class ThinkingMiddleware:
    def __init__(self, budget_tokens: int = 5000):
        self.budget_tokens = budget_tokens

    async def wrap_model_call(self, call: ModelCallFn) -> ModelCallFn:
        async def wrapped(messages, **kwargs):
            return await call(
                messages,
                thinking={"type": "enabled", "budget_tokens": self.budget_tokens},
                **kwargs
            )
        return wrapped
```

这展示了 LangGraph 的 thinking 集成在实际项目中的用法。
