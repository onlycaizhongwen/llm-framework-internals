# LangChain 源码分享讲解稿

这份文档用于向别人介绍 LangChain 源码。建议按“先讲全局，再讲设计思想，再讲主流程，最后讲源码入口”的顺序。

## 1. 一句话定位

LangChain 现在不是一个单体包，而是一个多包 monorepo。它的核心思路是：用 `langchain-core` 定义通用协议，再用 `langchain` 主包组织 Agent、模型初始化和应用入口，最后通过 `libs/partners` 接入各个模型、向量库和工具供应商。

可以这样开场：

> 我们看 LangChain 源码时，不要把它当成一个包看，而要当成一个分层生态看。`langchain-core` 负责定义统一协议，`langchain` 主包负责把这些协议组织成 Agent 和模型入口，`partners` 负责接入外部供应商。当前版本最重要的主线有两条：一条是 `create_agent` 基于 LangGraph 编译 Agent 流程，另一条是 RAG 场景下 Document、Embeddings、VectorStore、Retriever 的组合链路。

## 2. 目录分工

| 目录 | 包名 | 讲解口径 |
| --- | --- | --- |
| `libs/core` | `langchain-core` | 底层协议层，定义 `Runnable`、`BaseChatModel`、`BaseTool`、`BaseRetriever`、`VectorStore`、`Embeddings` |
| `libs/langchain_v1` | `langchain` | 当前主包，也就是现在 `pip install langchain` 的核心入口 |
| `libs/langchain` | `langchain-classic` | classic 兼容分支，主要是旧 chains、memory、classic agents |
| `libs/partners` | `langchain-openai` 等 | 供应商集成，比如 OpenAI、Anthropic、Qdrant |
| `libs/text-splitters` | `langchain-text-splitters` | 文本切分，常用于 RAG 前处理 |
| `libs/standard-tests` | `langchain-tests` | 集成包标准测试，用来约束供应商实现质量 |

## 3. 核心设计思想

LangChain 的总架构不算复杂，真正值得讲的是它背后的几个设计思想。

### 3.1 统一执行协议：把一切都抽象成 Runnable

LangChain 很核心的一个设计是：不管是模型、工具、检索器，还是组合链路，都尽量变成“可调用、可批量、可流式、可组合”的运行单元。

源码证据：`libs/core/langchain_core/runnables/base.py:125`

```python
class Runnable(ABC, Generic[Input, Output]):
    """A unit of work that can be invoked, batched, streamed, transformed and composed.
```

讲解口径：

> `Runnable` 是 LangChain 的统一执行协议。它让不同组件看起来像同一种东西：都可以 `invoke`，都可以批处理、流式输出，也都可以组合。这就是 LangChain 能把 prompt、model、tool、retriever 串起来的基础。

### 3.2 面向接口编程：core 定义协议，partners 做实现

LangChain 不把 OpenAI、Anthropic、Qdrant 等供应商逻辑写死在主包里，而是在 `core` 里定义接口，在 `partners` 里做具体适配。

源码证据：`libs/core/langchain_core/language_models/chat_models.py:270`

```python
class BaseChatModel(BaseLanguageModel[AIMessage], ABC):
    r"""Base class for chat models.
```

源码证据：`libs/partners/openai/langchain_openai/chat_models/base.py:581`

```python
class BaseChatOpenAI(BaseChatModel):
    """Base wrapper around OpenAI large language models for chat.
```

源码证据：`libs/partners/openai/langchain_openai/chat_models/base.py:2534`

```python
class ChatOpenAI(BaseChatOpenAI):
    r"""Interface to OpenAI chat model APIs.
```

讲解口径：

> 这里是典型的接口和实现分离。`BaseChatModel` 是 LangChain 的模型协议，`ChatOpenAI` 是 OpenAI 对这个协议的实现。换供应商时，应用侧尽量面对统一接口，而不是直接面对每个 SDK 的细节。

### 3.3 组合优于继承：复杂流程由小组件拼出来

RAG 不是一个单独的大类，Agent 也不是一个巨型执行器。LangChain 更倾向于把流程拆成小协议，再组合起来。

RAG 组合链路：

```text
TextSplitter -> Document -> Embeddings -> VectorStore -> Retriever -> ChatModel
```

对应源码：

- `libs/text-splitters/langchain_text_splitters/base.py`
- `libs/core/langchain_core/embeddings/embeddings.py`
- `libs/core/langchain_core/vectorstores/base.py`
- `libs/core/langchain_core/retrievers.py`
- `libs/core/langchain_core/language_models/chat_models.py`

讲解口径：

> LangChain 不是给每种场景都写一个巨型模块，而是定义一组可以拼接的零件。RAG 就是文本切分、嵌入、向量库、检索器、模型的组合。

### 3.4 图编排范式：Agent 不是链，而是状态图

当前版本的 Agent 主流程不是简单 chain，而是通过 LangGraph 编译成 `StateGraph`。这让 Agent 可以循环调用工具，也可以接入中间件、持久化、流式和中断恢复。

源码证据：`libs/langchain_v1/langchain/agents/factory.py:697`

```python
def create_agent(
    model: str | BaseChatModel,
    tools: Sequence[BaseTool | Callable[..., Any] | dict[str, Any]] | None = None,
```

源码证据：`libs/langchain_v1/langchain/agents/factory.py:1050`

```python
graph = StateGraph(
    state_schema=resolved_state_schema,
    input_schema=input_schema,
    output_schema=output_schema,
```

源码证据：`libs/langchain_v1/langchain/agents/factory.py:1386`

```python
graph.add_node("model", RunnableCallable(model_node, amodel_node, trace=False))
```

源码证据：`libs/langchain_v1/langchain/agents/factory.py:1390`

```python
graph.add_node("tools", tool_node)
```

源码证据：`libs/langchain_v1/langchain/agents/factory.py:1671`

```python
return graph.compile(
```

讲解口径：

> `create_agent` 的本质是图构建器。它把模型节点、工具节点、中间件节点拼成图，然后编译。模型需要工具时进入 tools 节点，工具结果再回到 model 节点，这就是 Agent 循环。

### 3.5 中间件范式：用 hook 扩展主流程

Agent 的可扩展点不是散落在各处，而是通过 `AgentMiddleware` 集中表达。

源码证据：`libs/langchain_v1/langchain/agents/middleware/types.py:383`

```python
class AgentMiddleware(Generic[StateT, ContextT, ResponseT]):
    """Base middleware class for an agent.
```

源码证据：`libs/langchain_v1/langchain/agents/middleware/types.py:443`

```python
def before_model(self, state: StateT, runtime: Runtime[ContextT]) -> dict[str, Any] | None:
```

源码证据：`libs/langchain_v1/langchain/agents/middleware/types.py:491`

```python
def wrap_model_call(
    self,
    request: ModelRequest[ContextT],
    handler: Callable[[ModelRequest[ContextT]], ModelResponse[ResponseT]],
```

讲解口径：

> 这是典型的 middleware / hook 设计。主流程不需要为每个扩展场景改代码，而是暴露 `before_model`、`after_model`、`wrap_model_call`、`wrap_tool_call` 这些扩展点。重试、缓存、限流、模型兜底、工具拦截都可以放在 middleware 里。

### 3.6 适配器范式：供应商包把外部 SDK 翻译成统一接口

`libs/partners` 的角色可以理解成 Adapter。它们不改变 LangChain 的核心协议，而是把外部供应商 SDK 的请求、响应、认证、流式输出和工具调用格式翻译成 LangChain 的统一抽象。

以 OpenAI 为例：

```text
OpenAI SDK / API
        ↓
langchain_openai.ChatOpenAI
        ↓
BaseChatModel
        ↓
LangChain Agent / Runnable / 应用代码
```

讲解口径：

> `partners` 不是业务主流程，而是适配层。它的价值是屏蔽不同供应商 SDK 的差异，让上层仍然面对 `BaseChatModel`、`Embeddings`、`VectorStore` 这些统一接口。

## 4. 两条主流程

### 4.1 Agent 流程

可以这样讲：

> 用户调用 `create_agent` 后，LangChain 会把模型、工具和中间件编译成 LangGraph 的图。模型先回答，如果需要工具，就进入 tools 节点执行工具，再把工具结果喂回模型，直到模型不再调用工具。

对应源码：

- `libs/langchain_v1/langchain/agents/factory.py`
- `libs/langchain_v1/langchain/agents/middleware/types.py`
- `libs/core/langchain_core/tools/base.py`
- `libs/core/langchain_core/language_models/chat_models.py`

### 4.2 RAG 流程

可以这样讲：

> 原始文本先通过 `TextSplitter` 切成 `Document`，再用 `Embeddings` 向量化并写入 `VectorStore`。用户提问时，问题也会被向量化，然后通过相似度搜索取回相关文档，再交给模型生成答案。

对应源码：

- `libs/text-splitters/langchain_text_splitters/base.py`
- `libs/core/langchain_core/embeddings/embeddings.py`
- `libs/core/langchain_core/vectorstores/base.py`
- `libs/core/langchain_core/retrievers.py`
- `libs/core/langchain_core/documents/`

## 5. 推荐源码阅读路线

1. `libs/core/langchain_core/runnables/base.py`
   先理解 LangChain 的统一执行协议。

2. `libs/core/langchain_core/language_models/chat_models.py`
   理解 ChatModel 的输入输出、流式调用和工具绑定。

3. `libs/langchain_v1/langchain/chat_models/base.py`
   理解 `init_chat_model` 如何从字符串模型名解析到供应商模型。

4. `libs/langchain_v1/langchain/agents/factory.py`
   理解 `create_agent` 如何把模型、工具和中间件编译成 LangGraph。

5. `libs/langchain_v1/langchain/agents/middleware/types.py`
   理解 Agent 的可扩展点。

6. `libs/core/langchain_core/tools/base.py`
   理解工具如何作为 Runnable 参与 Agent 循环。

7. `libs/core/langchain_core/retrievers.py`
   理解检索器接口。

8. `libs/core/langchain_core/vectorstores/base.py`
   理解向量库接口。

9. `libs/partners/openai/langchain_openai/`
   用 OpenAI 集成包作为供应商适配样例。

## 6. 分享时的注意事项

- 不要一开始讲所有模块，先讲 `core -> langchain_v1 -> partners` 这条主线。
- 不要把 `libs/langchain` 当成当前主包讲，它现在是 `langchain-classic`。
- 不要把 RAG 讲成单独模块，它是多个协议组合出来的流程。
- 不要只讲概念，每讲一个概念都最好指到一个源码文件。
- 如果听众偏业务，重点讲 Agent 和 RAG 两条流程。
- 如果听众偏工程，重点讲 `Runnable`、接口抽象、图编排、中间件和供应商适配。

## 7. 适合口头分享的版本

可以直接这样讲：

> LangChain 当前源码可以按三层理解。第一层是 `langchain-core`，它定义模型、消息、工具、检索器、向量库和 Runnable 这些通用协议。第二层是当前主包 `langchain`，源码在 `libs/langchain_v1`，它负责把这些协议组织成用户能直接使用的模型入口和 Agent 入口。第三层是 `libs/partners`，负责把 OpenAI、Anthropic、Qdrant 等外部服务适配成 LangChain 的统一接口。
>
> 它的核心设计思想有几个：第一是统一执行协议，很多东西都尽量抽象成 Runnable；第二是面向接口编程，core 定义协议，partners 做实现；第三是组合优于继承，RAG 和 Agent 都是小组件拼出来的；第四是图编排，当前 Agent 是 StateGraph，不是简单 chain；第五是中间件扩展，很多增强能力都通过 hook 接入。
>
> 所以读源码时，先看 `Runnable`，再看 `BaseChatModel`，然后看 `init_chat_model` 和 `create_agent`。理解这几个入口之后，再去看 OpenAI 这种 partner 包，就能看清 LangChain 是怎么把外部模型接进统一生态的。
