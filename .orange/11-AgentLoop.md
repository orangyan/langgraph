# 11 AgentLoop

## LangGraph 的 Loop：Pregel 超步模型

LangGraph 的 AgentLoop 不是简单的 while 循环，而是基于 Google Pregel 论文的**超步（Superstep）模型**——这是它与其他框架最根本的区别。

---

## 11.1 Pregel 超步 vs 传统 While 循环

**传统 while 循环（hermes-agent, nanobot）:**
```python
while not done and turns < max_turns:
    response = llm(messages)
    if response.tool_calls:
        results = execute_tools(response.tool_calls)
        messages.extend(results)
    else:
        done = True
```

**LangGraph Pregel 超步:**
```
超步 1: {START → agent}
  ├── 触发: START channel 有值
  ├── 执行: agent_fn(state)
  └── 输出: 写入 messages channel

超步 2: {agent → tools 或 agent → END}
  ├── 触发: messages channel 有新值
  ├── 执行: route_fn(state) → "tools" 或 END
  └── 分支

超步 3 (如果选 tools): {tools → agent}
  ├── 触发: messages channel 有新工具调用
  ├── 执行: ToolNode(state)
  └── 输出: 写入 ToolMessage 到 messages

超步 4: {agent → ...}（继续循环）
```

**关键差异:**
- **并行性**: 同一超步内的多个节点并行执行（while 循环串行）
- **同步屏障**: 超步之间有屏障，确保所有并行节点完成后才开始下一超步
- **持久化点**: 每个超步后可以保存 Checkpoint（while 循环没有自然分界）

---

## 11.2 PregelLoop 实现

```python
# libs/langgraph/langgraph/pregel/_loop.py

class PregelLoop:
    status: str  # "pending" | "done" | "interrupt"

    def tick(self) -> bool:
        """执行一个超步"""
        if not self._should_stop():
            tasks = self._prepare_tasks()  # 确定哪些节点可触发
            if not tasks:
                self.status = "done"
                return False
            self._execute_tasks(tasks)     # 并行执行（asyncio.gather）
            self._commit(tasks)            # 写入 Channels + 保存 Checkpoint
            return True
        return False

    def _prepare_tasks(self) -> list[PregelTask]:
        """Pregel 计划阶段：确定触发节点"""
        tasks = []
        for node_name, node in self.nodes.items():
            # 比较节点的 versions_seen 和 Channel 当前版本
            if self._has_new_input(node_name):
                task = PregelTask(node=node_name, input=self._get_input(node_name))
                tasks.append(task)
        return tasks

class AsyncPregelLoop(PregelLoop):
    async def _execute_tasks(self, tasks):
        await asyncio.gather(*[
            self._run_task(task)
            for task in tasks
        ])
```

---

## 11.3 中间件支持（新版 langchain_v1）

虽然 LangGraph 核心没有"中间件"概念，但 `langchain_v1` 的 `create_agent` 在 LangGraph 之上添加了中间件层：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    model_call_limit,
    model_retry,
    tool_retry,
    summarization,
    human_in_the_loop,
)

agent = create_agent(
    model=llm,
    tools=tools,
    middleware=[
        model_call_limit(max_calls=100),
        model_retry(max_retries=3, retry_on=(APIError,)),
        tool_retry(max_retries=2),
        summarization(model=summarizer_llm, max_tokens=4096),
    ]
)
```

**8 个 Hook Points（AgentMiddleware 协议）:**
```python
class AgentMiddleware(Protocol):
    async def before_agent(self, state: AgentState) -> AgentState
    async def after_agent(self, state: AgentState) -> AgentState
    async def before_model(self, request: ModelRequest) -> ModelRequest
    def wrap_model_call(self, call: ModelCallFn) -> ModelCallFn
    async def after_model(self, state: AgentState) -> AgentState
    def wrap_tool_call(self, call: ToolCallFn) -> ToolCallFn
    async def before_tool(self, request: ToolCallRequest) -> ToolCallRequest
    async def after_tool(self, state: AgentState) -> AgentState
```

---

## 11.4 与其他项目的 AgentLoop 对比

| 维度 | LangGraph | deer-flow | hermes-agent | nanobot | gstack |
|------|---------|-----------|--------------|---------|--------|
| **实现方式** | Pregel 超步模型 | LangGraph（使用 LangGraph） | 手写 while 循环 | 异步事件总线 | Claude Code 原生 Loop |
| **主要文件** | `pregel/_loop.py` | `middleware.py` + LangGraph | `agent_loop.py` | `event_bus.py` | SKILL.md + claude binary |
| **中间件** | `langchain_v1` AgentMiddleware（新） | ✅ (Middleware Chain) | ✅ (函数列表) | ✅ (事件处理器) | ❌ (文字描述) |
| **并行工具执行** | ✅ (asyncio.gather) | ✅ (LangGraph 内置) | ✅ (asyncio.gather) | ❌ | ✅ (Claude Code 内置) |
| **流式输出** | ✅ (7 种 stream_mode) | ✅ (LangGraph stream) | ✅ (AsyncGenerator) | ❌ | ✅ (stream-json) |
| **持久化检查点** | ✅ (每超步) | ✅ (LangGraph 继承) | ✅ (SQLite) | ❌ | ✅ (git WIP) |
| **时间旅行** | ✅ | ✅ | ❌ | ❌ | ❌ |
| **可观察性** | ✅ (LangSmith + Studio) | ✅ (LangSmith) | ✅ (WandB) | ⚠️ | ✅ (NDJSON) |

---

## 11.5 Loop 终止条件

**正常终止:**
- 节点路由到 `END`
- 所有并行分支都到达 `END`
- `tick()` 返回 False（无可触发节点）

**异常终止:**
- `max_concurrency` 超限（Pregel 并发限制）
- `recursion_limit` 超限（默认 25 超步）
- 节点抛出未处理异常
- `interrupt_before/after` 中断点触发

**配置 recursion_limit:**
```python
graph = builder.compile()
config = {
    "recursion_limit": 50,  # 默认 25
    "configurable": {"thread_id": "..."}
}
await graph.ainvoke(input, config=config)
```
