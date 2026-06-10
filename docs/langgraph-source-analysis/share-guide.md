# LangGraph 源码分享讲解稿

这份文档用于对外分享 LangGraph 源码。建议按“定位 -> 架构 -> 主流程 -> 设计思想 -> 阅读路径”的顺序讲。

## 1. 一句话定位

LangGraph 是一个低层状态图编排框架，用来构建长时间运行、可中断、可恢复、有状态的 Agent 或工作流。

可以这样开场：

> 看 LangGraph 源码时，不要先把它理解成一个 Agent 框架，而要理解成一个状态图运行时。它让我们用节点和边描述流程，用 state 描述上下文，用 checkpoint 保存每一步状态。Agent、工具调用、人类介入、恢复执行，都只是这个图运行时上的不同模式。

## 2. 目录怎么讲

| 目录 | 分享口径 |
| --- | --- |
| `libs/langgraph/langgraph/graph` | 面向用户的建图 API，核心是 `StateGraph` |
| `libs/langgraph/langgraph/pregel` | 真正的执行引擎，按 Pregel/BSP step 模型运行 |
| `libs/langgraph/langgraph/channels` | 状态传播和合并规则 |
| `libs/checkpoint` | checkpoint 抽象，支撑恢复、回放、中断继续 |
| `libs/checkpoint-postgres` / `libs/checkpoint-sqlite` | checkpoint 的存储实现 |
| `libs/prebuilt` | 预构建 Agent 和工具节点，例如 `create_react_agent`、`ToolNode` |
| `libs/cli`、`libs/sdk-py`、`libs/sdk-js` | 服务化和远程调用工具链 |

## 3. 主流程怎么讲

主流程可以浓缩成一条线：

```text
StateGraph 声明图
    -> add_node / add_edge / add_conditional_edges
    -> compile()
    -> CompiledStateGraph
    -> Pregel.stream() / invoke()
    -> loop.tick() 按 step 推进
    -> runner.tick() 执行节点
    -> channel 更新 + checkpoint 保存
```

讲解口径：

> LangGraph 的源码脉络很像一个小型编译执行系统。`StateGraph` 是声明阶段，`compile()` 是编译阶段，`Pregel` 是运行时。用户写的是节点函数和边，运行时负责调度、并发、流式输出、中断恢复和状态持久化。

## 4. 核心设计思想

### 4.1 状态图，而不是线性链

源码证据：`libs/langgraph/langgraph/graph/state.py:130`

```python
class StateGraph(Generic[StateT, ContextT, InputT, OutputT]):
    """A graph whose nodes communicate by reading and writing to a shared state.

    The signature of each node is `State -> Partial<State>`.
    """
```

讲解口径：

> 节点之间不是直接互相调用，而是通过共享 state 协作。每个节点输入完整或局部状态，输出状态增量。这样复杂流程就能被拆成节点和边。

### 4.2 Builder 和 Runtime 分离

源码证据：`libs/langgraph/langgraph/graph/state.py:1164`

```python
def compile(...) -> CompiledStateGraph[StateT, ContextT, InputT, OutputT]:
    """Compiles the `StateGraph` into a `CompiledStateGraph` object.

    The compiled graph implements the `Runnable` interface and can be invoked,
    streamed, batched, and run asynchronously.
    """
```

讲解口径：

> `StateGraph` 不能直接执行，必须 compile。compile 之后才得到支持 invoke、stream、异步和批处理的运行对象。这个分离让建图 API 简单，运行时能力统一。

### 4.3 Pregel/BSP 回合制执行

源码证据：`libs/langgraph/langgraph/pregel/main.py:449`

```python
class Pregel(...):
    """Pregel manages the runtime behavior for LangGraph applications.

    Pregel combines actors and channels into a single application.
    Actors read data from channels and write data to channels.
    """
```

源码证据：`libs/langgraph/langgraph/pregel/main.py:2979`

```python
while loop.tick():
    for _ in runner.tick(...):
        yield from _output(...)
    loop.after_tick()
```

讲解口径：

> Pregel 的思想可以讲成“回合制执行”。每一轮先选出要运行的节点，再执行它们，最后统一更新 channel。这样并发节点不会互相看到半成品状态，执行边界很清楚。

### 4.4 Channel 决定状态怎么合并

讲解口径：

> LangGraph 的 state 看起来像 TypedDict，但运行时真正处理的是 channel。channel 决定这个状态 key 是保存最后值、累积多个值，还是用 reducer 聚合。这个设计解决了图里多节点写同一状态的问题。

### 4.5 Checkpoint 是版本化状态快照

源码证据：`libs/checkpoint/langgraph/checkpoint/base/__init__.py:92`

```python
class Checkpoint(TypedDict):
    """State snapshot at a given point in time."""

    channel_values: dict[str, Any]
    channel_versions: ChannelVersions
    versions_seen: dict[str, ChannelVersions]
```

讲解口径：

> checkpoint 不是简单保存最终结果，而是保存 channel 的值、版本、节点看过哪些版本。因为有这些信息，LangGraph 才能从某个 thread_id 恢复、回放，或者在人类中断后继续执行。

### 4.6 Prebuilt Agent 只是图的组合

源码证据：`libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py:862`

```python
workflow = StateGraph(
    state_schema=state_schema or AgentState, context_schema=context_schema
)

workflow.add_node("agent", RunnableCallable(call_model, acall_model))
workflow.add_node("tools", tool_node)
workflow.add_conditional_edges("agent", should_continue, path_map=agent_paths)
workflow.add_edge("tools", entrypoint)

return workflow.compile(...)
```

讲解口径：

> `create_react_agent` 并没有绕过 LangGraph 的核心机制。它也是创建 `StateGraph`，添加 agent 节点和 tools 节点，再加条件边，最后 compile。也就是说，Agent 是图运行时上的一个模板。

## 5. 分享时的脉络

可以按 15 分钟分享来组织：

1. 2 分钟：定位 LangGraph，说明它和 LangChain 的区别。
2. 3 分钟：看 monorepo 目录，讲 graph、pregel、checkpoint、prebuilt。
3. 4 分钟：讲主流程，从 `StateGraph` 到 `Pregel.stream()`。
4. 4 分钟：讲 checkpoint 和 human-in-the-loop 为什么能成立。
5. 2 分钟：讲 prebuilt agent，说明 Agent 只是图模板。

## 6. 最后怎么收束

可以这样总结：

> LangGraph 的源码不复杂，关键是抓住分层。`graph` 是声明图，`pregel` 是执行图，`checkpoint` 是保存图状态，`prebuilt` 是用图搭好的 Agent 模板。理解这条线，再看中断、恢复、流式、人类介入，就都能落回同一个模型：状态图的版本化执行。
