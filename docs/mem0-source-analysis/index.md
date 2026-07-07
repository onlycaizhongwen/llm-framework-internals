# mem0 源码架构精读

分析对象：`sources/mem0`，固定源码提交 `cd79fa8914b5b1cf66daacc957d826065df57df8`。

本文参考 AutoGen 精读方式，但 mem0 的核心不在 Agent 协作运行时，而在“长期记忆层”：如何从对话中抽取事实，如何写入向量存储，如何用语义、关键词、实体和 reranker 找回相关记忆。

## 1. 总体结论

mem0 是一个面向 AI 应用和 Agent 的长期记忆框架。它的主线是：

- **Memory Core**：`Memory.add/search/update/delete`，负责记忆生命周期。
- **Provider Abstraction**：LLM、Embedding、VectorStore、Reranker 都通过抽象基类和 Factory 接入。
- **Storage / Retrieval**：向量库保存 memory，SQLite 保存 history 和最近消息，entity store 负责实体链接增强。
- **Product Entrypoints**：Python library、Cloud `MemoryClient`、Self-hosted FastAPI server、CLI、Vercel AI SDK integration。

分享口径：

> AutoGen 解决“多个 Agent 怎么协作”，mem0 解决“Agent 和应用应该记住什么、怎么检索回来”。

## 2. 最高层架构

| 层级 | 源码位置 | 精读重点 |
| --- | --- | --- |
| Memory Core | `mem0/memory/main.py` | `Memory.add`、`_add_to_vector_store`、`Memory.search`、`_search_vector_store` |
| Config / Factory | `mem0/configs`、`mem0/utils/factory.py` | 通过 `MemoryConfig` 和 Factory 装配 provider |
| LLM / Embedding | `mem0/llms`、`mem0/embeddings` | LLM 事实抽取、embedding 写入/搜索/update |
| Vector Store / Reranker | `mem0/vector_stores`、`mem0/reranker` | 向量检索、keyword search、可选 rerank |
| Entity / Scoring | `mem0/utils/entity_extraction.py`、`mem0/utils/scoring.py` | 实体抽取、实体增强、混合评分 |
| Product Entrypoints | `server`、`mem0/client`、`cli`、`integrations` | REST、自托管、Cloud SDK、命令行和前端生态 |

架构图见：[architecture.mmd](architecture.mmd)。

```mermaid
flowchart TB
    App["应用 / Agent / CLI / REST\n使用方只关心 add/search"] --> OSS["mem0.Memory\n长期记忆编排核心"]
    App --> Client["MemoryClient\nCloud API 入口"]
    App --> Server["Self-hosted FastAPI Server\n把 Memory 包成 REST 服务"]
    Server --> OSS
    OSS --> Config["MemoryConfig\n声明要用哪些 provider"]
    OSS --> Add["Memory.add\n把对话提炼成长期 facts"]
    OSS --> Search["Memory.search\n把长期 facts 找回来"]
    Add --> LLM["LLMBase\n负责事实抽取\n为什么：过滤聊天噪音"]
    Add --> Embedder["EmbeddingBase\n事实向量化\n为什么：支持语义召回"]
    Add --> Vector["VectorStoreBase\n写入 / 检索 / keyword_search\n为什么：统一不同向量库"]
    Add --> SQLite["SQLiteManager\nhistory + last messages\n为什么：保留审计和上下文"]
    Add --> Entity["Entity Store\n实体 -> memory_id 链接\n为什么：提高人/项目相关召回"]
    Search --> Embedder
    Search --> Vector
    Search --> Entity
    Search --> Scoring["score_and_rank\nsemantic + BM25 + entity boost\n为什么：多信号比单向量更稳"]
    Search --> Reranker["Reranker\n可选二次排序\n为什么：高质量场景再付成本"]
```

读图说明：

- 这张图要从上往下读。应用可以直接调用本地 `mem0.Memory`，也可以通过 self-hosted server 间接调用，或者用 `MemoryClient` 访问 Cloud API。
- 中间的 `Memory` 是 OSS 内核，真正的主流程只有两条：`add` 写入记忆、`search` 检索记忆。
- 为什么这么设计：长期记忆不应该绑定某个模型或向量库，所以 LLM、Embedding、VectorStore、Reranker 都放在抽象层后面，由 Factory 按配置创建。
- 讲解重点：不要把这张图讲成“模块清单”，要强调它是“入口层 -> Memory core -> provider 抽象 -> 存储/检索能力”的依赖方向。

## 3. 源码精读一：Memory 初始化和装配

`Memory` 是组合器。它不直接绑定某个模型或向量库，而是从 `MemoryConfig` 读取 provider，然后通过 Factory 创建实例。

源码证据：

- `mem0/memory/main.py:444` 定义 `class Memory(MemoryBase)`。
- `mem0/memory/main.py:445-489` 初始化 embedder、vector_store、llm、SQLite history、collection、reranker。
- `mem0/configs/base.py:29` 定义 `MemoryConfig`。
- `mem0/utils/factory.py` 定义 `LlmFactory`、`EmbedderFactory`、`VectorStoreFactory`、`RerankerFactory`。
- `mem0/__init__.py:6` 导出 `AsyncMemory` 和 `Memory`。

关键片段：

```python
class Memory(MemoryBase):
    def __init__(self, config: MemoryConfig = MemoryConfig()):
        self.embedding_model = EmbedderFactory.create(...)
        self.vector_store = VectorStoreFactory.create(...)
        self.llm = LlmFactory.create(...)
        self.db = SQLiteManager(...)
```

设计含义：这是典型 Ports and Adapters。Memory core 只依赖 LLM、Embedding、VectorStore、Reranker 的抽象协议，具体 OpenAI、Qdrant、PGVector、Cohere 等通过 Factory 接入。

## 4. 源码精读二：Memory.add 写入流程

`Memory.add` 是 mem0 最核心的写入路径。新版 README 提到的 ADD-only extraction，在源码里对应 `_add_to_vector_store()` 的 phased batch pipeline。

主线：

```text
Memory.add
  -> 校验 user_id / agent_id / run_id
  -> 标准化 messages
  -> _add_to_vector_store
  -> 读取 last messages + 检索 existing memories
  -> LLM 单次抽取 facts
  -> embed_batch
  -> hash 去重
  -> vector_store.insert
  -> SQLite history
  -> entity linking
```

源码证据：

- `mem0/memory/main.py:717` 定义 `Memory.add()`。
- `mem0/memory/main.py:759-776` 校验并构造 metadata/filters。
- `mem0/memory/main.py:789-807` 标准化 messages。
- `mem0/memory/main.py:831` 定义 `_add_to_vector_store()`。
- `mem0/memory/main.py:870-885` 查询最近消息和已有 memory。
- `mem0/memory/main.py:895-907` 生成 additive extraction prompt 并调用 LLM。
- `mem0/memory/main.py:932-944` 批量 embedding 抽取出的 facts。
- `mem0/memory/main.py:947-981` hash 去重并组装 payload。
- `mem0/memory/main.py:991-999` 批量写入 vector store。
- `mem0/memory/main.py:1012-1028` 批量写 history。
- `mem0/memory/main.py:1031-1145` 批量实体抽取和 entity store 链接。
- `mem0/configs/prompts.py:468` 定义 `ADDITIVE_EXTRACTION_PROMPT`。
- `mem0/configs/prompts.py:1016` 定义 `generate_additive_extraction_prompt()`。

流程图见：[add-flow.mmd](add-flow.mmd)。

```mermaid
flowchart TD
    Start([Memory.add\n写入长期记忆]) --> Validate["校验 user_id / agent_id / run_id\n为什么：避免不同用户/Agent 记忆串线"]
    Validate --> Normalize["messages 标准化\n为什么：统一输入形态"]
    Normalize --> Context["last messages + existing memories\n为什么：让 LLM 知道上下文和已有 facts"]
    Context --> LLM["LLM 单次事实抽取\n为什么：把聊天噪音变成稳定事实"]
    LLM --> Embed["embed_batch 新 facts\n为什么：批量向量化更高效"]
    Embed --> Dedup["hash 去重\n为什么：避免重复 facts 污染记忆"]
    Dedup --> Insert["vector_store.insert\n写入可语义检索的长期 facts"]
    Insert --> History["SQLite history\n为什么：审计记忆变化"]
    History --> Entity["entity linking\n为什么：后续按人/项目增强召回"]
```

读图说明：

- 这张图展示的是“从对话到长期记忆”的写入路径。
- 输入不是直接进入向量库，而是先被标准化为 messages，再结合最近对话和已有记忆，让 LLM 做一次事实抽取，最后才进入 embedding、去重、写库、历史记录和实体链接。
- 为什么这么设计：聊天原文通常有寒暄、重复、临时上下文，直接存会污染长期记忆；先抽取 facts 可以让 memory 更稳定、更可检索。
- 为什么要查 existing memories：抽取 prompt 会带已有记忆，目的是减少重复事实，并让 LLM 知道哪些内容已经沉淀过。
- 为什么要 entity linking：长期记忆查询常围绕人、地点、项目、偏好展开，实体链接能在后续检索时给相关 memory 加分。

设计含义：mem0 的写入不是简单“把文本 embed 后存起来”。它先让 LLM 抽取可长期保存的事实，再做去重、历史记录、实体链接。这让 memory 更像知识沉淀层，而不是普通聊天日志。

## 5. 源码精读三：Memory.search 混合检索

`Memory.search` 是读取主路径。它先校验作用域，再进入 `_search_vector_store()`。检索不是单纯 vector search，而是 semantic + BM25 keyword + entity boost 的融合。

主线：

```text
Memory.search
  -> 校验 query / filters / top_k / threshold
  -> 处理高级 metadata filters
  -> lemmatize query + extract entities
  -> embedding semantic search
  -> keyword_search 取 BM25 分
  -> entity store 计算 entity_boost
  -> score_and_rank
  -> 可选 reranker
```

源码证据：

- `mem0/memory/main.py:1331` 定义 `Memory.search()`。
- `mem0/memory/main.py:1413-1431` 校验 filters，要求至少有 `user_id`、`agent_id`、`run_id` 之一。
- `mem0/memory/main.py:1438-1454` 处理高级 metadata filters。
- `mem0/memory/main.py:1456-1465` 调 `_search_vector_store()`。
- `mem0/memory/main.py:1469-1474` 可选 reranker。
- `mem0/memory/main.py:1580` 定义 `_search_vector_store()`。
- `mem0/memory/main.py:1586-1600` lemmatize query、extract entities、生成 query embedding。
- `mem0/memory/main.py:1602-1612` semantic search 和 keyword search。
- `mem0/memory/main.py:1622-1624` 计算 entity boosts。
- `mem0/memory/main.py:1638-1645` 调 `score_and_rank()`。
- `mem0/utils/scoring.py:60` 定义 `score_and_rank()`。

流程图见：[search-flow.mmd](search-flow.mmd)。

```mermaid
flowchart TD
    Start([Memory.search\n检索长期记忆]) --> Validate["校验 query / filters\n为什么：控制作用域和质量下限"]
    Validate --> Prep["lemmatize + extract_entities + embed\n为什么：同时准备语义、关键词、实体信号"]
    Prep --> Semantic["semantic vector search\n找语义相近 memory"]
    Prep --> Keyword["keyword BM25\n补足精确关键词"]
    Prep --> Entity["entity boost\n同一实体相关内容加分"]
    Semantic --> Score["score_and_rank\n融合三路信号"]
    Keyword --> Score
    Entity --> Score
    Score --> Rerank["optional reranker\n高质量排序场景再启用"]
```

读图说明：

- 这张图展示的是“从问题到相关记忆”的检索路径。
- mem0 先做 query 清洗和作用域校验，再并行取得三类信号：语义向量召回、BM25 关键词匹配、实体链接增强，然后交给 `score_and_rank` 做融合排序。
- 为什么不是只做向量检索：长期记忆里常有姓名、产品、地点、具体偏好，仅靠语义可能把精确实体打散；BM25 和 entity boost 用来补足精确匹配。
- 为什么 threshold 先过滤 semantic：源码里先用 semantic threshold 过滤候选，再叠加 BM25/entity，避免关键词把语义完全不相关的内容强行推上来。
- 为什么 reranker 是可选：rerank 成本更高，适合对排序质量要求高的场景；普通场景用融合分数已经能覆盖主要需求。

设计含义：长期记忆检索比普通 RAG 更强调“人和事的一致性”。semantic 负责语义召回，BM25 负责精确关键词，entity boost 负责实体关联，reranker 负责最后排序增强。

## 6. 源码精读四：Entity linking 和二级向量集合

mem0 维护一个 entity store。写入 memory 时抽取实体，把实体和 memory_id 建立链接；查询时抽取 query entities，搜索 entity store，再给 linked memories 加分。

源码证据：

- `mem0/utils/entity_extraction.py:751` 定义 `extract_entities()`。
- `mem0/utils/entity_extraction.py:761` 定义 `extract_entities_batch()`。
- `mem0/memory/main.py:540` 规范化 entity text。
- `mem0/memory/main.py:562` 定义 `_upsert_entity()`。
- `mem0/memory/main.py:664` 定义 `_link_entities_for_memory()`。
- `mem0/memory/main.py:1685` 定义 `_compute_entity_boosts()`。
- `mem0/utils/scoring.py:57` 定义 `ENTITY_BOOST_WEIGHT = 0.5`。

关键片段：

```python
ENTITY_BOOST_WEIGHT = 0.5

def score_and_rank(...):
    raw_combined = semantic_score + bm25_score + entity_boost
```

设计含义：实体不是直接替代向量检索，而是作为 boost 信号参与最终排序。这是更稳的方式：语义仍是召回主线，实体负责把“同一个人/地点/主题”的记忆推上来。

## 7. 源码精读五：产品入口

mem0 同时提供本地库、Cloud SDK、自托管 REST 和 CLI。

源码证据：

- `mem0/client/main.py:71` 定义 `MemoryClient`。
- `mem0/client/main.py:173` 定义 Cloud SDK 的 `add()`。
- `mem0/client/main.py:289` 定义 Cloud SDK 的 `search()`。
- `mem0/client/main.py:964` 定义 `AsyncMemoryClient`。
- `server/main.py:366` 定义自托管 `/memories` 创建接口。
- `server/main.py:451` 定义自托管 `/search` 检索接口。
- `server/main.py:321-331` 定义配置查询和配置更新接口。

设计含义：`mem0.Memory` 是 OSS 内核，`server` 把它包成 REST，`MemoryClient` 面向 Cloud API，CLI 和 integrations 是使用层。分析源码时应先看 Memory core，再看产品入口。

## 8. 核心设计思想和范式

| 设计思想 | 源码证据 | 解释 |
| --- | --- | --- |
| Memory Layer 优先 | `Memory.add/search` | 框架核心是记忆生命周期，不是 Agent 流程编排 |
| Pipeline 架构 | `_add_to_vector_store`、`_search_vector_store` | 写入和检索拆成阶段，便于优化和替换 |
| Ports and Adapters | `LLMBase`、`EmbeddingBase`、`VectorStoreBase` | Core 依赖抽象端口，provider 是适配器 |
| Factory Method | `LlmFactory`、`EmbedderFactory`、`VectorStoreFactory` | 通过 provider name 动态创建实现 |
| Hybrid Retrieval | `score_and_rank` | semantic、BM25、entity boost 融合 |
| Session-scoped Memory | `_build_filters_and_metadata` | user/agent/run 作用域隔离，避免记忆串线 |
| Audit Trail | `SQLiteManager.add_history/batch_add_history` | 写入、更新、删除有历史记录 |

## 9. 应用场景和框架对比

mem0 更适合：

- **个性化 AI 助手**：记住用户偏好、长期目标、习惯。
- **客服和销售助手**：记住历史问题、客户背景、沟通偏好。
- **Agent 长期任务**：跨 session 保留任务上下文和执行偏好。
- **学习 / 健康 / 生产力应用**：长期积累用户状态和进展。
- **已有 Agent 框架补能力**：给 LangGraph、AutoGen、CrewAI、LangChain 应用补长期记忆层。

| 维度 | mem0 | AutoGen | LangGraph | LangChain |
| --- | --- | --- | --- | --- |
| 核心定位 | 长期记忆层 | 多 Agent 消息运行时 | 状态图 Agent Runtime | LLM 应用组件库 |
| 主流程 | add/search memory | send/publish + Team chat | node/edge/checkpoint | runnable/tool/retriever |
| 状态模型 | user/agent/run scoped memories | Agent/Team runtime state | 显式全局 state | 组件自带状态或外部状态 |
| 适用场景 | 个性化、偏好、长期上下文 | 多 Agent 对话协作 | 可恢复、可审计复杂流程 | RAG、工具调用、集成组合 |
| 选型判断 | 问“要记住什么” | 问“谁和谁协作” | 问“状态怎么流转” | 问“LLM 组件怎么搭” |

## 10. 真实例子：从一句话到可检索记忆

假设用户在一个代码助手里说：

> 以后帮我分析源码时，尽量用中文说明，先讲架构再讲主流程。我最近在准备 mem0 和 LangGraph 的源码分享。

如果直接把整句话存进向量库，后面会混入“以后”“最近”“帮我”等聊天噪音。mem0 的写入链路更像下面这样：

```text
原始消息
  -> LLM 抽取长期 facts
  -> embed_batch
  -> hash 去重
  -> vector_store.insert
  -> history + entity linking
```

可能沉淀出的 facts：

```json
[
  {"memory": "用户偏好用中文解释源码。"},
  {"memory": "用户偏好源码分享先讲架构，再讲主流程。"},
  {"memory": "用户正在准备 mem0 和 LangGraph 的源码分享。"}
]
```

后续用户问“我上次准备的框架分享讲到哪了？”时，`Memory.search` 不只靠向量相似度，还会结合关键词和实体增强：

- semantic search 能召回“源码分享”“框架分析”语义相关 memory。
- BM25 能强化 `mem0`、`LangGraph` 这类精确词。
- entity boost 能把同一项目、同一框架相关的记忆往前推。

分享时可以强调：这个例子说明 mem0 的核心价值不是“存文本”，而是“把对话中的长期事实变成可管理、可检索、可解释的 memory”。

### 10.1 更真实的产品例子：源码分享偏好记忆

场景：用户连续几天都在做源码分析分享，他告诉系统：

> 以后源码分析文档请用中文，先讲仓库边界，再讲主流程；HTML 要浅色背景，图里要有中文说明。

mem0 的写入可以理解为：

```python
memory.add(
    messages=[{"role": "user", "content": "以后源码分析文档请用中文，先讲仓库边界，再讲主流程；HTML 要浅色背景，图里要有中文说明。"}],
    user_id="caizw",
)
```

它可能抽出几条长期 facts：

| 抽出的 memory | 后续怎么用 |
| --- | --- |
| 用户偏好中文源码分析文档 | 生成新框架分析时默认中文。 |
| 用户希望先讲仓库边界再讲主流程 | 写文档结构时自动排序。 |
| 用户主要看 HTML | 优先打开和优化 HTML 页面。 |
| 用户不喜欢黑色背景 | 前端样式使用浅色高对比。 |
| 用户要求图中有中文说明 | Mermaid 节点写中文解释和“为什么”。 |

后续当用户只说“分析 Graphiti，参考 mem0”时，应用可以先执行：

```python
memory.search("源码分析文档偏好", user_id="caizw")
```

然后把检索到的偏好放进系统提示词里。这样模型不需要用户每次重复说明“中文、HTML、浅色背景、图中说明”。

## 11. 局限性和使用边界

| 风险 / 边界 | 为什么存在 | 落地建议 |
| --- | --- | --- |
| LLM 事实抽取可能误抽或漏抽 | 写入质量依赖 prompt、模型能力和上下文 | 对关键业务记忆增加人工确认、置信度或回滚机制 |
| 不是所有聊天都应该进入 memory | 临时上下文、寒暄、敏感信息会污染长期记忆 | 只沉淀偏好、约束、长期状态、关键决策和客户历史 |
| 记忆需要作用域隔离 | 用户、Agent、run 混在一起会造成记忆串线 | 强制传 `user_id`、`agent_id` 或 `run_id`，并设计清晰租户边界 |
| 长期记忆涉及隐私治理 | memory 可能包含偏好、客户信息、健康/财务等敏感内容 | 提供删除、导出、审计、脱敏和保留周期策略 |
| 混合检索增加成本和复杂度 | semantic、BM25、entity、reranker 都有额外计算或存储成本 | 默认先开 semantic + keyword，关键场景再加 entity/reranker |
| 记忆过期和冲突需要治理 | 用户偏好和任务状态会变化，旧记忆可能不再正确 | 结合 update/delete、history 和时间字段处理过期事实 |

讲这部分不是为了削弱 mem0，而是为了让听众知道：长期记忆是一层“带治理责任的状态系统”，不是一个无脑追加的向量库。

## 12. 和 LangGraph 怎么组合

LangGraph 和 mem0 适合互补：LangGraph 管“这次任务怎么流转”，mem0 管“跨会话应该记住什么”。

组合图见：[langgraph-combo.mmd](langgraph-combo.mmd)。

```mermaid
flowchart LR
    User["用户请求\n例如：继续分析 mem0 源码"] --> Graph["LangGraph StateGraph\n负责流程状态和节点流转"]
    Graph --> Load["load_memory 节点\n调用 mem0.search\n为什么：进入任务前取回长期上下文"]
    Load --> Agent["agent / tool 节点\n结合 state + memories 做推理或执行"]
    Agent --> Decide{"流程是否结束?\nLangGraph 根据 state 分支"}
    Decide -->|未结束| Agent
    Decide -->|结束| Save["save_memory 节点\n调用 mem0.add\n为什么：把新偏好、决策、任务状态沉淀为长期 facts"]
    Save --> Store["mem0 Memory Layer\n按 user_id / agent_id / run_id 隔离保存"]
    Store --> Next["下次会话\nmem0.search 再把相关记忆取回"]
```

推荐讲法：

- LangGraph 的 `state` 是单次流程内的显式状态，适合控制节点、分支、checkpoint 和恢复。
- mem0 的 `memory` 是跨会话的长期事实，适合保存用户偏好、项目背景、关键决策和历史约束。
- 两者组合时，可以在图入口加一个 `load_memory` 节点，在图结束或关键节点后加一个 `save_memory` 节点。
- 不建议把所有 LangGraph state 都写入 mem0，只挑选“下次会话仍然有价值”的 facts。

## 13. 分享建议

建议分享顺序：

1. 先讲定位：mem0 是 memory layer，不是 Agent 编排框架。
2. 再讲架构：Memory core、provider 抽象、存储检索、产品入口。
3. 精读写入：`Memory.add -> _add_to_vector_store -> LLM extraction -> vector insert -> entity linking`。
4. 精读检索：`Memory.search -> _search_vector_store -> semantic/BM25/entity -> score_and_rank`。
5. 用真实例子串起写入和检索，让听众看到一条 memory 如何产生、如何被找回。
6. 最后讲设计范式、局限性、使用边界和 LangGraph 组合方式。

收束口：

> mem0 源码最值得看的不是某个 provider，而是它如何把“长期记忆”拆成事实抽取、向量写入、历史记录、实体链接、混合检索和可选 rerank。读懂 `Memory.add` 和 `Memory.search` 两条主线，就读懂了 mem0 的核心设计。
