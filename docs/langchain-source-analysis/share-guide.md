# LangChain 源码分享讲解稿

这份讲解稿配合 `index.html` 使用，重点讲新增四个专题：Agent middleware、provider adapter、RAG 细节、classic 迁移关系。

## 1. 开场定位

可以这样开场：

> 现在读 LangChain 源码，不能再把它当成一个单体大包。它已经拆成协议层、应用入口层、供应商适配层和 classic 兼容层。当前 Agent 是基于 LangGraph 编译出来的状态图，middleware 是扩展点；RAG 则是 Document、Embedding、VectorStore、Retriever 和 ChatModel 的组合。

## 2. 讲解顺序

1. 包边界：`langchain-core` 定协议，`langchain` 做当前主入口，`partners` 接 provider，`langchain-classic` 保留旧 API。
2. Agent middleware：`create_agent` 编译 `StateGraph`，middleware 的 before/after hook 变成图节点，wrap hook 包住模型/工具调用。
3. Provider adapter：`init_chat_model("openai:gpt-5.5")` 只负责解析和选择 provider，真正 API 适配在 `langchain-openai` 等 partner 包里。
4. RAG：不是一个大类，而是 `TextSplitter -> Document -> Embeddings -> VectorStore -> Retriever -> Prompt -> ChatModel` 的组合链路。
5. Classic 迁移：旧 chains、memory、classic agents 留在 `langchain-classic`，新项目优先看 `langchain_v1` 和 `core`。

## 3. 四个专题怎么讲

### Agent middleware

核心话术：

> `create_agent` 的本质是图构建器。它把模型节点、工具节点和 middleware 节点拼成 LangGraph。middleware 分两类：一类是 before/after 生命周期 hook，会变成图节点；另一类是 wrap_model_call/wrap_tool_call，会包住模型或工具调用，用来做重试、限流、缓存、路由和审计。

源码锚点：

- `libs/langchain_v1/langchain/agents/factory.py:697-718`
- `libs/langchain_v1/langchain/agents/factory.py:1386-1474`
- `libs/langchain_v1/langchain/agents/middleware/types.py:383-503`
- `libs/langchain_v1/langchain/agents/middleware/types.py:662-730`

### Provider adapter

核心话术：

> LangChain 主包不直接绑定 OpenAI SDK。`init_chat_model` 负责把 `provider:model` 解析成具体模型实现；`langchain-openai` 这种 partner 包负责把 LangChain 的 Message、Tool、Callback、stream chunk 翻译成 OpenAI SDK 的请求和响应。

源码锚点：

- `libs/langchain_v1/langchain/chat_models/base.py:210-330`
- `libs/langchain_v1/langchain/chat_models/base.py:493-518`
- `libs/langchain_v1/langchain/chat_models/base.py:597-625`
- `libs/partners/openai/langchain_openai/chat_models/base.py:581-612`
- `libs/partners/openai/langchain_openai/chat_models/base.py:1624-1795`

### RAG 细节

核心话术：

> LangChain 的 RAG 不是一个神秘黑盒，而是一串协议对象。切分器把资料变成 Document；Embedding 把文档和查询变成向量；VectorStore 负责写入和相似度搜索；Retriever 把检索统一成 Runnable；最后把召回文档放进 prompt 给 ChatModel。

源码锚点：

- `libs/text-splitters/langchain_text_splitters/base.py:44-149`
- `libs/core/langchain_core/embeddings/embeddings.py:8-58`
- `libs/core/langchain_core/vectorstores/base.py:43-75`
- `libs/core/langchain_core/vectorstores/base.py:964-1058`
- `libs/core/langchain_core/retrievers.py:55-236`

### Classic 迁移

核心话术：

> `libs/langchain` 现在发布为 `langchain-classic`，它是旧 API 的兼容层，不是新项目的首选入口。新项目优先看 `libs/langchain_v1`，也就是当前 `langchain` 包；需要旧 chains、memory、classic agents 时，再回 classic 包看迁移背景。

源码锚点：

- `libs/langchain_v1/pyproject.toml:6-28`
- `libs/langchain/pyproject.toml:6-27`
- `libs/langchain/README.md:21-23`

## 4. 适合口头分享的一段总结

> LangChain 的源码主线可以压缩成三句话：第一，`langchain-core` 定义统一协议，所以模型、工具、检索器、向量库都能被组合；第二，当前主包 `langchain` 把这些协议组织成模型入口和基于 LangGraph 的 Agent；第三，外部 provider 都通过 partner 包适配，classic 包则保留旧 API。理解这三层之后，再看 middleware、RAG 和迁移关系就顺了。
