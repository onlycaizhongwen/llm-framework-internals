# AutoGen 源码分享讲稿

这份文档用于分享 AutoGen 源码。建议按“版本状态 -> 四层架构 -> Runtime 精读 -> AssistantAgent 精读 -> GroupChat 精读 -> 设计范式和 LangGraph 对比”的顺序讲。

## 1. 开场定位

可以这样开场：

> AutoGen 是微软开源的多 Agent 框架，现在已经进入 maintenance mode。它仍然很值得读，因为它不是简单封装聊天模型，而是把多 Agent 系统拆成 Core Runtime、AgentChat、Extensions 和 Studio 几层，是典型的“消息运行时 + Agent Team”设计。

## 2. 目录怎么讲

| 目录 | 分享口径 | 精读入口 |
| --- | --- | --- |
| `python/packages/autogen-core` | 底层消息运行时 | `_agent_runtime.py`、`_single_threaded_agent_runtime.py`、`_routed_agent.py` |
| `python/packages/autogen-agentchat` | 用户常用 API | `agents/_assistant_agent.py`、`teams/_group_chat/*` |
| `python/packages/autogen-ext` | 模型、MCP、代码执行等生态适配 | `models/openai/_openai_client.py`、`tools/mcp/_workbench.py` |
| `python/packages/autogen-studio` | 可视化构建和演示入口 | 产品化入口说明，不是运行时主干 |

## 3. 主流程一句话

```text
Core Runtime:
send_message / publish_message -> envelope -> queue -> _process_next -> RoutedAgent handler

AssistantAgent:
run_stream -> on_messages_stream -> _call_llm -> _process_model_result -> tool loop -> TaskResult

GroupChat:
run_stream -> register participants/manager/subscriptions -> GroupChatStart -> select_speaker -> GroupChatTermination
```

讲源码时不要只说“它有 Agent 和 Team”。更准确的讲法是：Core 管消息投递，AssistantAgent 管模型和工具循环，GroupChat 把 Team API 映射到 Core Runtime。

## 4. 源码精读口径

### 4.1 Runtime 消息投递

证据链：

- `_agent_runtime.py:21-50` 定义 `send_message/publish_message`。
- `_single_threaded_agent_runtime.py:332`、`:387` 入队。
- `_single_threaded_agent_runtime.py:671` 由 `_process_next` 分派。
- `_single_threaded_agent_runtime.py:466`、`:557` 分别处理 send 和 publish。

```python
class AgentRuntime(Protocol):
    async def send_message(...)
    async def publish_message(...)
```

讲法：

> AutoGen 的底座更像 actor runtime。`send_message` 是点对点 RPC，`publish_message` 是 topic 广播，真正的执行由 runtime queue 和 handler 分派完成。

### 4.2 AssistantAgent 工具循环

证据链：

- `_assistant_agent.py:901` 进入 `on_messages_stream`。
- `_assistant_agent.py:1056` 调模型。
- `_assistant_agent.py:1118` 处理模型结果。
- `_assistant_agent.py:1196` 执行工具。
- `_assistant_agent.py:1409` 反思工具结果。

```python
class AssistantAgent(BaseChatAgent, Component[AssistantAgentConfig]):
    """An agent that provides assistance with tool use."""
```

讲法：

> AssistantAgent 是组合器，不是 provider。它把 model_context、memory、tools、handoff 和模型调用串起来；模型和工具分别下沉到 `ChatCompletionClient`、`Workbench`、`BaseTool`。

### 4.3 GroupChat 映射到 Core Runtime

证据链：

- `_base_group_chat.py:191-241` 注册参与者、manager 和订阅。
- `_base_group_chat.py:535` 发布 `GroupChatStart`。
- `_base_group_chat_manager.py:306` 抽象 `select_speaker`。
- `_round_robin_group_chat.py:72` 给出轮询策略。
- `_selector_group_chat.py:50-52` 说明可以用模型或自定义函数选 speaker。

```python
class BaseGroupChat(Team, ABC, ComponentBase[BaseModel]):
    """The base class for group chat teams."""
```

讲法：

> GroupChat 看起来像群聊，但底层是 runtime、topic、manager、participant container 的组合。Team 不直接 while 循环调用每个 Agent，而是把协作映射成消息流。

## 5. 设计思想怎么讲

| 设计思想 | 源码证据 | 一句话解释 |
| --- | --- | --- |
| 运行时优先 | `AgentRuntime`、`SingleThreadedAgentRuntime` | 先有消息运行时，再有聊天 Agent |
| Actor / Event | `send_message`、`publish_message` | Agent 通过点对点消息和 topic 事件协作 |
| 声明式路由 | `@message_handler`、`@event`、`@rpc` | 消息类型决定进入哪个 handler |
| Template Method | `BaseChatAgent.run_stream`、`on_messages_stream` | 基类固定协议，子类实现行为 |
| Strategy | RoundRobin / Selector / Swarm manager | 替换 speaker selection 策略实现不同 Team |
| Ports and Adapters | `ChatCompletionClient`、`BaseTool`、`Workbench` | Core 定义端口，Ext 实现 provider |

## 6. 应用场景和 LangGraph 对比

应用场景可以这样讲：

> AutoGen 适合多 Agent 研究协作、团队式聊天原型、人机协同、工具/代码执行任务、Studio 可视化演示，以及存量 AutoGen 系统维护。

| 问题 | AutoGen 更合适 | LangGraph 更合适 |
| --- | --- | --- |
| 我要表达什么？ | 多个 Agent 像群聊一样协作 | 一个状态图按节点和边精确流转 |
| 多 Agent 怎么组织？ | RoundRobin、Selector、Swarm 等 Team 模式 | 节点、子图、条件边和共享状态 |
| 控制要求 | 发言顺序、终止条件、工具调用和流式事件 | checkpoint、interrupt、恢复、审计和复杂分支 |
| 项目建议 | 源码学习、原型、存量维护 | 新项目里的复杂 Agent Runtime |

一句话选型：

> AutoGen 强在“消息驱动的 Agent 团队协作”，LangGraph 强在“状态驱动的可控执行流”。

## 7. 15 分钟分享节奏

1. 2 分钟：说明 maintenance mode 和源码学习价值。
2. 3 分钟：讲 Core、AgentChat、Extensions、Studio 四层。
3. 4 分钟：精读 Core Runtime 消息投递。
4. 3 分钟：精读 AssistantAgent 的模型和工具循环。
5. 3 分钟：精读 GroupChat 和设计范式，再对比 LangGraph。

## 8. 收束口

> AutoGen 源码可以浓缩成一句话：Core 提供消息运行时，AgentChat 提供聊天 Agent 和 Team，Extensions 接入模型和工具。读懂 Runtime、AssistantAgent、GroupChat 三条主线，就读懂了它的多 Agent 设计。
