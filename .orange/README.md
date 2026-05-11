# LangGraph 项目分析文档索引

本目录包含 LangGraph 框架的详细分析文档。

## 文档目录

| 文件 | 内容 |
|------|------|
| [01-产品介绍.md](./01-产品介绍.md) | 项目背景、核心定位、与 LangChain 关系 |
| [02-功能结构.md](./02-功能结构.md) | StateGraph、检查点、流式输出、人机交互 |
| [03-技术架构.md](./03-技术架构.md) | Pregel 执行引擎、Channels、Checkpointer |
| [04-代码结构.md](./04-代码结构.md) | 目录组织、核心文件、关键类说明 |
| [05-数据流.md](./05-数据流.md) | 图构建流、执行流、流式输出流 |
| [06-Agent设计.md](./06-Agent设计.md) | create_agent、ToolNode、多 Agent 子图 |
| [07-部署运行.md](./07-部署运行.md) | 安装配置、LangGraph Platform、Docker |
| [08-记忆架构.md](./08-记忆架构.md) | Checkpointer 短期记忆 vs BaseStore 长期记忆 |
| [09-多轮会话管理.md](./09-多轮会话管理.md) | thread_id/checkpoint 会话模型 |
| [10-Lead-Sub-Agent.md](./10-Lead-Sub-Agent.md) | 子图嵌套、Send API、多 Agent 架构 |
| [11-AgentLoop.md](./11-AgentLoop.md) | Pregel 超步执行模型 vs 其他 Loop 实现 |
| [12-Thinking模式.md](./12-Thinking模式.md) | Extended Thinking 在 LangGraph 中的集成 |
| [13-SKILL管理.md](./13-SKILL管理.md) | ToolNode 工具系统 vs SKILL.md 技能系统 |
| [14-如何评测.md](./14-如何评测.md) | 单元测试、集成测试、基准测试 |

## 项目一句话

**LangGraph** 是 LangChain 公司开源的 AI Agent 编排框架，以 Pregel（Bulk Synchronous Parallel）执行引擎为核心，通过 StateGraph + Channels + Checkpointer 三要素实现有状态的、可持久化的、支持人机交互的 Agent 工作流，是 deer-flow、hermes-agent 等 Agent 项目的底层基础设施。
