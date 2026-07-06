# AutoGen 源码分享讲解稿

这份文档用于分享 AutoGen 源码。建议按“版本状态 -> 四层架构 -> Core Runtime -> AgentChat -> GroupChat -> 设计思想和对比”的顺序讲。

## 1. 开场定位

可以这样开场：

> AutoGen 是微软开源的多 Agent 框架，当前已经进入 maintenance mode，新项目官方建议看 Microsoft Agent Framework。但从源码学习角度，AutoGen 仍然很有价值，因为它把多 Agent 系统拆成了 Core Runtime、AgentChat、Extensions 和 Studio 几层。

## 2. 目录怎么讲

| 目录 | 分享口径 |
| --- | --- |
| `python/packages/autogen-core` | 底层消息运行时，包含 AgentRuntime、RoutedAgent、模型和工具抽象 |
| `python/packages/autogen-agentchat` | 用户常用 API，包含 AssistantAgent、Team、GroupChat、终止条件 |
| `python/packages/autogen-ext` | 模型、MCP、代码执行、GraphRAG 等生态适配 |
| `python/packages/autogen-studio` | 可视化构建和运行多 Agent 应用 |
| `python/packages/agbench` | Agent 评测工具 |
| `dotnet`、`protos` | 跨语言和分布式运行时相关基础 |

## 3. Core Runtime 主线

```text
AgentRuntime
  -> register_factory / register_agent_instance
  -> send_message / publish_message
  -> SingleThreadedAgentRuntime
  -> queue envelope
  -> RoutedAgent
  -> @message_handler / @event / @rpc
```

讲解口径：

> Core 层不是聊天 UI，而是 Agent 消息运行时。它把 Agent 看成可以接收消息、发布消息、处理 topic 的 actor。AgentChat 的团队协作最终也会映射到这个 runtime。

源码证据：

```python
class AgentRuntime(Protocol):
    async def send_message(...)
    async def publish_message(...)
```

```python
class SingleThreadedAgentRuntime(AgentRuntime):
    """A single-threaded agent runtime that processes all messages using a single asyncio queue."""
```

## 4. AgentChat 主线

```text
BaseChatAgent.run()
  -> task 转成 TextMessage
  -> on_messages / on_messages_stream
  -> AssistantAgent
  -> ChatCompletionClient.create
  -> tool calls / workbench
  -> Response
  -> TaskResult
```

讲解口径：

> AgentChat 是给应用开发者用的高层 API。AssistantAgent 把模型 client、上下文、工具、memory、handoff 包起来；BaseChatAgent 统一提供 run 和 run_stream。

源码证据：

```python
class BaseChatAgent(ChatAgent, ABC, ComponentBase[BaseModel]):
    """Base class for a chat agent."""
```

```python
class AssistantAgent(BaseChatAgent, Component[AssistantAgentConfig]):
    """An agent that provides assistance with tool use."""
```

## 5. GroupChat 主线

```text
Team.run_stream()
  -> 初始化 runtime
  -> 注册 participants
  -> 注册 group chat manager
  -> 发布 GroupChatStart
  -> manager 选择 speaker
  -> participant 响应
  -> 终止条件 / max_turns
  -> TaskResult
```

讲解口径：

> GroupChat 是 AutoGen 多 Agent 协作的核心。RoundRobin 是轮流说话，Selector 是让模型或函数选择下一位，Swarm 则偏 handoff 交接。

源码证据：

```python
class BaseGroupChat(Team, ABC, ComponentBase[BaseModel]):
    """The base class for group chat teams."""
```

```python
class RoundRobinGroupChatManager(BaseGroupChatManager):
    """A group chat manager that selects the next speaker in a round-robin fashion."""
```

## 6. 核心设计思想

- **运行时优先**：先有消息、topic、runtime、handler，再有聊天 Agent。
- **分层 API**：Core 负责底座，AgentChat 负责易用入口，Extensions 负责生态，Studio 负责可视化。
- **消息驱动**：点对点 `send_message` 和广播 `publish_message` 是底层协作模型。
- **状态和流式输出**：Agent 是 stateful，`run_stream` 能输出中间事件和最终结果。
- **可插拔生态**：模型、工具、MCP、代码执行都通过抽象接口接入。

## 7. 和 LangChain / LangGraph / CrewAI 对比

| 框架 | 一句话定位 | 更适合 |
| --- | --- | --- |
| AutoGen | 多 Agent 消息运行时 + Chat Team | 学习多 Agent runtime、维护存量 AutoGen、做团队式聊天原型 |
| LangChain | LLM 应用组件库和 Runnable 生态 | RAG、工具调用、模型/向量库/加载器集成 |
| LangGraph | 状态图 Agent Runtime | 复杂分支、循环、checkpoint、可恢复执行 |
| CrewAI | 角色化 Agent 团队和任务编排 | 产品化多角色协作、研究报告、销售/运营自动化 |

一句话对比：

> AutoGen 更像“多 Agent 消息运行时”，LangGraph 更像“可控状态图”，CrewAI 更像“角色任务团队”，LangChain 更像“LLM 应用组件库”。

## 8. 15 分钟分享节奏

1. 2 分钟：说明 AutoGen 版本状态和学习价值。
2. 3 分钟：讲 Core、AgentChat、Extensions、Studio 四层。
3. 4 分钟：讲 Core Runtime 的消息投递。
4. 3 分钟：讲 AssistantAgent 的模型和工具循环。
5. 3 分钟：讲 GroupChat 团队编排和与其他框架对比。

## 9. 收束句

> AutoGen 源码可以浓缩成一句话：Core 提供消息运行时，AgentChat 提供聊天 Agent 和 Team，Extensions 接入模型和工具，Studio 提供可视化入口。理解这几层，就能看懂 AutoGen 的多 Agent 设计。
