# mem0 源码分享讲稿

这份讲稿用于分享 `mem0ai/mem0` 源码。建议按“项目定位 -> 四层架构 -> add 写入精读 -> search 检索精读 -> 抽象边界 -> 和 AutoGen/LangGraph 对比”的顺序讲。

## 1. 开场定位

可以这样开场：

> mem0 不是 Agent 编排框架，而是给 AI 应用和 Agent 提供长期记忆层。它的主线不是“谁来发言”，而是“如何从对话里抽取事实、存成可检索记忆，并在后续查询里用语义、关键词、实体等多信号找回来”。

## 2. 目录怎么讲

| 目录 | 分享口径 | 精读入口 |
| --- | --- | --- |
| `mem0/memory` | OSS 本地记忆核心 | `main.py`、`storage.py`、`base.py` |
| `mem0/llms` | LLM provider 抽象和实现 | `base.py`、`openai.py`、`*_structured.py` |
| `mem0/embeddings` | embedding provider 抽象和实现 | `base.py`、`openai.py`、`huggingface.py` |
| `mem0/vector_stores` | 向量库适配层 | `base.py`、`qdrant.py`、`pgvector.py` 等 |
| `mem0/reranker` | 可选 rerank 层 | `base.py`、`llm_reranker.py` 等 |
| `server` / `client` / `cli` | 产品入口 | FastAPI 自托管、Cloud SDK、命令行 |

## 3. 主流程一句话

```text
Memory.add:
messages -> LLM 单次事实抽取 -> embed_batch -> vector_store.insert -> history/entity linking

Memory.search:
query -> embedding semantic search + keyword BM25 + entity boost -> score_and_rank -> optional reranker

产品入口:
Library Memory / FastAPI Server / MemoryClient Cloud SDK / CLI
```

## 4. 源码精读口径

### 4.1 Memory 初始化

证据链：

- `mem0/memory/main.py:444` 定义 `class Memory`。
- `mem0/memory/main.py:445-489` 初始化 embedder、vector_store、llm、SQLite history、reranker。
- `mem0/configs/base.py:29` 定义 `MemoryConfig`。
- `mem0/utils/factory.py` 定义 `LlmFactory`、`EmbedderFactory`、`VectorStoreFactory`、`RerankerFactory`。

讲法：

> Memory 是组合器。它不自己实现模型、向量库和 rerank，而是通过配置和 Factory 创建 provider，然后把这些能力组织成 add/search 两条主流程。

### 4.2 add 写入流程

证据链：

- `mem0/memory/main.py:717` 定义 `Memory.add()`。
- `mem0/memory/main.py:831` 进入 `_add_to_vector_store()`。
- `mem0/memory/main.py:870-885` 查询已有 memory，给 LLM 抽取提供上下文。
- `mem0/memory/main.py:895-907` 组装 additive extraction prompt 并调用 LLM。
- `mem0/memory/main.py:932-944` 批量 embedding 新事实。
- `mem0/memory/main.py:947-981` hash 去重并组装 payload。
- `mem0/memory/main.py:991-999` 批量写入 vector store。
- `mem0/memory/main.py:1012-1028` 写 history。
- `mem0/memory/main.py:1031-1145` 批量实体抽取和 entity store 链接。

讲法：

> 新版 mem0 是 ADD-only 事实抽取：LLM 一次性抽出应该新增的 facts，然后用 embedding 批量写入向量库。它不在主流程里让 LLM 决定 UPDATE/DELETE，而是通过 hash、上下文检索和实体链接减少重复，提高检索质量。

### 4.3 search 检索流程

证据链：

- `mem0/memory/main.py:1331` 定义 `Memory.search()`。
- `mem0/memory/main.py:1413-1431` 校验 filters 和参数，要求至少有 `user_id`、`agent_id`、`run_id` 之一。
- `mem0/memory/main.py:1456-1465` 调 `_search_vector_store()`。
- `mem0/memory/main.py:1580` 定义 `_search_vector_store()`。
- `mem0/memory/main.py:1586-1600` lemmatize query、extract entities、生成 query embedding。
- `mem0/memory/main.py:1602-1612` semantic search 和 keyword search。
- `mem0/memory/main.py:1622-1624` 计算 entity boost。
- `mem0/memory/main.py:1638-1645` 调 `score_and_rank()`。
- `mem0/utils/scoring.py:60` 定义 `score_and_rank()`。

讲法：

> mem0 的 search 不是单纯向量相似度。它先用向量库召回语义候选，再叠加 BM25 keyword 分数和实体链接增强，最后可选 reranker。这个设计服务于“长期记忆”：同义表达、关键词精确匹配、实体一致性都要兼顾。

### 4.4 Entity linking

证据链：

- `mem0/utils/entity_extraction.py:751` 定义 `extract_entities()`。
- `mem0/utils/entity_extraction.py:761` 定义 `extract_entities_batch()`。
- `mem0/memory/main.py:562` 定义 `_upsert_entity()`。
- `mem0/memory/main.py:664` 定义 `_link_entities_for_memory()`。
- `mem0/memory/main.py:1685` 定义 `_compute_entity_boosts()`。

讲法：

> Entity store 是第二个向量集合，用实体文本关联 memory_id。写入时把 facts 中的实体抽出来，检索时从 query 抽实体并查 entity store，找到相关实体后给 linked memories 加分。

## 5. 设计思想怎么讲

| 设计思想 | 源码证据 | 一句话解释 |
| --- | --- | --- |
| Memory Layer 优先 | `Memory.add/search` | 框架核心是记忆写入和检索，不是 Agent 编排 |
| Pipeline 架构 | `_add_to_vector_store`、`_search_vector_store` | 写入和检索都被拆成稳定阶段 |
| Ports and Adapters | `LLMBase`、`EmbeddingBase`、`VectorStoreBase` | Core 依赖抽象，provider 通过 Factory 接入 |
| Hybrid Retrieval | `score_and_rank` | semantic、BM25、entity boost 多信号融合 |
| Session-scoped Memory | `_build_filters_and_metadata` | 记忆必须按 user/agent/run 作用域隔离 |
| History / Audit | `SQLiteManager` | 写入、更新、删除都有 history 记录 |

## 6. 和 AutoGen / LangGraph 对比

| 维度 | mem0 | AutoGen | LangGraph |
| --- | --- | --- | --- |
| 核心定位 | 长期记忆层 | 多 Agent 消息运行时 | 状态图 Agent Runtime |
| 主流程 | add/search memory | send/publish + Team chat | node/edge/checkpoint |
| 状态模型 | user/agent/run scoped memories | Agent/Team runtime state | 显式全局 state |
| 适用场景 | 个性化助手、客服历史、偏好记忆、长期上下文 | 多 Agent 对话协作 | 可恢复、可审计、复杂分支流程 |
| 工程价值 | 给任何 Agent/App 补长期记忆 | 学 actor/message multi-agent | 学状态机式 Agent 工作流 |

一句话选型：

> 如果问题是“Agent 要记住什么”，看 mem0；如果问题是“多个 Agent 怎么协作”，看 AutoGen；如果问题是“流程状态怎么可控流转”，看 LangGraph。

## 7. 15 分钟分享节奏

1. 2 分钟：讲 mem0 定位，强调它是 memory layer。
2. 3 分钟：讲架构分层：Memory core、provider 抽象、存储检索、产品入口。
3. 4 分钟：精读 `Memory.add` 写入流程。
4. 3 分钟：精读 `Memory.search` 混合检索流程。
5. 3 分钟：讲设计范式、应用场景和框架对比。

## 8. 收束口

> mem0 源码最值得看的不是某个 provider，而是它如何把“长期记忆”拆成事实抽取、向量写入、历史记录、实体链接、混合检索和可选 rerank。读懂 `Memory.add` 和 `Memory.search` 两条主线，就读懂了 mem0 的核心设计。
