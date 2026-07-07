# Graphiti 源码分享讲稿

## 开场

今天这份源码分享讲 Graphiti，不是继续讲 Zep adapter。Zep 当前仓库更多是 Cloud SDK、framework integrations、MCP 和 eval；Graphiti 才是 Zep README 指向的开源 temporal context graph engine。

一句话：Graphiti 把 Agent memory 做成“有来源、有时间窗口、可增量更新、可混合检索的图谱”。

这句话可以展开成三层：

- 有来源：每条事实都能追溯到 episode。
- 有时间窗口：事实有 `valid_at`、`invalid_at`、`expired_at`。
- 可混合检索：不是只靠 embedding，而是 BM25、向量、BFS 和 rerank 组合。

## 先讲仓库边界

- `graphiti_core/`：核心引擎，重点精读。
- `server/`：FastAPI 服务包装。
- `mcp_server/`：MCP 工具服务。
- `examples/`：Neo4j、FalkorDB、Neptune、LangGraph agent 等示例。
- `tests/`：核心逻辑、driver、LLM client、MCP 的测试。

边界提醒：Graphiti 是自托管 OSS engine，不包含 Zep 托管平台的用户/线程/dashboard/SLA/治理能力。

## 架构图讲法

先展示 `architecture.mmd`。讲法是：

Graphiti 入口是 `Graphiti` 类，内部组合四类依赖：

- `LLMClient`：实体抽取、边抽取、去重、矛盾识别。
- `EmbedderClient`：实体名、fact、query 向量。
- `CrossEncoderClient`：更强 rerank。
- `GraphDriver`：Neo4j/FalkorDB/Neptune/Kuzu 等后端。

核心图模型是 episode、entity、fact edge。episode 保留原始证据，edge 保存事实和 temporal fields。

为什么这么设计：

- 把 LLM 抽取和图数据库访问拆开，是为了降低 provider 绑定。
- 把 episode、entity、fact edge 拆开，是为了同时保留原始证据和可检索结构。
- 把 FastAPI/MCP 放在外层，是为了让 Graphiti core 可以被服务、工具、应用多种入口复用。

## 写入流程讲法

展示 `ingestion-flow.mmd`，沿 `Graphiti.add_episode()` 走：

1. 用户输入 episode，带 `reference_time`。
2. Graphiti 取 previous episodes，给抽取提供上下文。
3. 保存 `EpisodicNode`。
4. `extract_nodes()` 抽实体。
5. `resolve_extracted_nodes()` 做语义候选、确定性相似和 LLM 去重。
6. `extract_edges()` 抽 fact / relation。
7. `resolve_extracted_edges()` 找重复和矛盾。
8. `resolve_edge_contradictions()` 给旧 fact 写 `invalid_at` / `expired_at`。
9. 保存 nodes、episodic edges、entity edges。

重点强调：Graphiti 的“记忆更新”不是覆盖，而是让旧事实失效，同时保留历史。

这里可以补源码细节：

- previous episodes 是为了让当前 episode 的抽取有上下文，避免省略主语导致抽错。
- `group_id` 是图分区，适合按用户、租户或业务域隔离。
- `entity_types` / `edge_types` 是 schema 约束，避免完全依赖 LLM 自由发挥。
- `saga` / `NEXT_EPISODE` 是连续事件链，适合长任务或多轮过程追踪。
- tracing span 记录 node、edge、invalidated_count 和 duration，说明这条链路是生产可观测的。

## 检索流程讲法

展示 `search-flow.mmd`：

- `Graphiti.search()` 是基础 fact search。
- `Graphiti.search_()` 是高级图谱搜索，默认 `COMBINED_HYBRID_SEARCH_CROSS_ENCODER`。
- 底层 search 同时跑 edges、nodes、episodes、communities。
- 每个 scope 可以组合 BM25、cosine similarity、BFS。
- rerank 支持 RRF、MMR、cross encoder、node distance、episode mentions。

这和 mem0 的差异是：mem0 更像记忆 item/fact 的检索层；Graphiti 把 fact 放在图边上，并利用图结构和时间窗口做更强的召回。

这里可以补为什么不是只用向量：

- BM25 适合专有名词、代码名、精确术语。
- 向量适合语义相近但字面不同的问题。
- BFS 适合从相关实体扩展邻居关系。
- RRF/MMR/cross encoder/node distance 负责把多路候选统一排序和去噪。

## 代码片段作证

`EntityEdge` 是最能说明设计思想的代码：

```python
class EntityEdge(Edge):
    fact: str
    episodes: list[str]
    expired_at: datetime | None
    valid_at: datetime | None
    invalid_at: datetime | None
```

`resolve_edge_contradictions()` 是 temporal memory 的关键：

```python
edge.invalid_at = resolved_edge.valid_at
edge.expired_at = edge.expired_at if edge.expired_at is not None else utc_now()
invalidated_edges.append(edge)
```

这两段说明 Graphiti 的 fact 不是普通文本，而是可追溯、可过期、可按时间查询的关系。

## 真实例子

例子：

用户先说：“源码分析先讲仓库边界，再讲主流程。”

Graphiti 写入：

```python
await graphiti.add_episode(
    name="preference",
    episode_body="用户希望源码分析先讲仓库边界，再讲主流程。",
    source_description="chat message",
    reference_time=now,
    group_id="share",
)
```

后续用户说：“分享时别先讲边界了，先讲应用场景。”

Graphiti 会让旧偏好成为历史事实，而不是简单删除。讲到这里，听众就能理解为什么它叫 temporal context graph。

## 和几个框架的对比口径

| 项目 | 讲法 |
| --- | --- |
| Graphiti | 开源 temporal graph memory engine，重在事实时间变化和 provenance。 |
| Zep | 托管 context graph 平台，Graphiti 是开源核心之一。 |
| mem0 | 轻量 memory layer，适合快速落地长期记忆。 |
| LangGraph | Agent workflow runtime，适合和 Graphiti 组合：LangGraph 管流程，Graphiti 管长期图谱记忆。 |

## 局限性

要主动讲局限性，这样分享更可信：

- LLM structured output 质量会影响抽取。
- 写入链路重，适合异步队列，不适合同步阻塞请求。
- 需要图数据库和索引，运维成本高于 mem0。
- 自托管 Graphiti 不等于 Zep 托管能力。
- Kuzu 已 deprecated，新项目优先 Neo4j/FalkorDB。

## 建议演讲节奏

- 10 分钟：Graphiti 是什么，和 Zep/mem0/LangGraph 的边界。
- 15 分钟：数据模型 episode/entity/fact edge。
- 25 分钟：`add_episode()` 写入主流程。
- 20 分钟：temporal invalidation 和代码证据。
- 15 分钟：`search_()` 混合检索。
- 10 分钟：真实例子、局限性、选型建议。

总时长 60-90 分钟都可以，不需要压缩到 15 分钟。
