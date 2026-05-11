# 13 SKILL 管理

## LangGraph 的"工具"就是其他框架的"技能"

LangGraph 没有 SKILL.md 系统——它通过 `BaseTool`/`StructuredTool` 和 `ToolNode` 管理"技能"（工具）。

---

## 13.1 工具定义方式

**方式一：`@tool` 装饰器（最简单）**
```python
from langchain_core.tools import tool

@tool
def search_web(query: str) -> str:
    """Search the web for information. Use this for current events."""
    return web_search_api(query)

@tool
def execute_code(code: str, language: str = "python") -> str:
    """Execute code in a sandbox environment."""
    return sandbox.run(code, language)
```

**方式二：`StructuredTool`（更多控制）**
```python
from langchain_core.tools import StructuredTool
from pydantic import BaseModel

class SearchInput(BaseModel):
    query: str
    max_results: int = 5

search_tool = StructuredTool.from_function(
    func=search_web_fn,
    name="search_web",
    description="Search the web for information",
    args_schema=SearchInput,
    return_direct=False,
)
```

**方式三：继承 `BaseTool`（最灵活）**
```python
from langchain_core.tools import BaseTool

class CustomTool(BaseTool):
    name: str = "custom_tool"
    description: str = "A custom tool"

    def _run(self, query: str) -> str:
        return self._sync_implementation(query)

    async def _arun(self, query: str) -> str:
        return await self._async_implementation(query)
```

---

## 13.2 ToolNode：工具执行器

```python
from langgraph.prebuilt import ToolNode

tools = [search_web, execute_code, file_read]
tool_node = ToolNode(tools)

# 在图中使用
builder.add_node("tools", tool_node)
builder.add_edge("agent", "tools")
builder.add_edge("tools", "agent")
```

**ToolNode 并行执行机制:**
```python
# 如果 AI 调用了 3 个工具:
# AIMessage.tool_calls = [
#     {"name": "search_web", "id": "1", "args": {"query": "..."}},
#     {"name": "execute_code", "id": "2", "args": {"code": "..."}},
#     {"name": "file_read", "id": "3", "args": {"path": "..."}},
# ]

# ToolNode 并行执行所有工具:
results = await asyncio.gather(
    search_web.ainvoke({"query": "..."}),
    execute_code.ainvoke({"code": "..."}),
    file_read.ainvoke({"path": "..."}),
)
# 返回 3 个 ToolMessage
```

---

## 13.3 工具元数据（extras）

```python
from langchain_core.tools import BaseTool

class CacheFriendlyTool(BaseTool):
    extras: dict = {
        "cache_control": {"type": "ephemeral"},  # Anthropic 提示缓存
    }
```

`extras` 字段允许存储 vendor-specific 扩展，由底层 LLM provider 解析。

---

## 13.4 与其他项目的工具/技能系统对比

| 维度 | LangGraph/LangChain | deer-flow | hermes-agent | nanobot | gstack |
|------|---------------------|-----------|--------------|---------|--------|
| **技能格式** | Python 函数 + schema | Python 函数 + schema | Python 函数 + SQL | Python 函数 | SKILL.md (Markdown) |
| **执行方式** | ToolNode（并行） | ToolNode（并行） | async 函数调用 | 直接调用 | Claude 读取并决策 |
| **动态发现** | ❌ (编译时固定) | ❌ | ✅ (数据库查询) | ❌ | ✅ (文件系统) |
| **版本控制** | 代码版本 | 代码版本 | 数据库行 | 代码版本 | git (.tmpl + .md) |
| **自我更新** | ❌ | ❌ | ✅ (LLM 写入 DB) | ❌ | ⚠️ (学习 → 更新 tmpl) |
| **多 Host 支持** | ❌ (单一运行时) | ❌ | ❌ | ❌ | ✅ (10+ hosts) |
| **工具并行** | ✅ (asyncio.gather) | ✅ | ✅ | ❌ | ✅ (Claude Code 内置) |
| **验证机制** | Pydantic schema | Pydantic | SQL schema | 无 | frontmatter schema |

---

## 13.5 工具与 create_agent 的集成

```python
from langchain_v1.agents import create_agent

# create_agent 内部构建 LangGraph 图，ToolNode 是其核心节点
agent = create_agent(
    model=llm,
    tools=[search_web, execute_code, file_read],
    middleware=[
        tool_retry(max_retries=2),  # 工具失败时自动重试
    ]
)

# 等价于手动构建:
# builder.add_node("agent", llm_with_tools)
# builder.add_node("tools", ToolNode([search_web, execute_code, file_read]))
# builder.add_conditional_edges("agent", route_to_tools_or_end)
```

---

## 13.6 PreBuilt 工具（内置工具）

LangGraph 通过 `langchain-community` 提供大量预构建工具：

```python
from langchain_community.tools import (
    DuckDuckGoSearchRun,
    WikipediaQueryRun,
    PythonREPLTool,
    ReadFileTool,
    WriteFileTool,
    ShellTool,
)

tools = [
    DuckDuckGoSearchRun(),
    PythonREPLTool(),
    ReadFileTool(),
]
```

与 gstack 的 23+ 专家角色技能不同，LangGraph 工具更细粒度（每个工具只做一件事），由 Agent 自主组合。
