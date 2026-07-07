# Zep 源码分享讲稿

这份讲稿用于分享 `getzep/zep` 源码，参考 mem0 的分析风格，但开场必须先讲仓库边界。

## 1. 开场定位

可以这样开场：

> 今天讲的 `getzep/zep` 和 mem0 不太一样。mem0 仓库里能看到本地 memory core，而当前 Zep 仓库 README 明确说它不是 Zep 产品或服务端源码，它主要放 examples、framework integrations、MCP server、evaluation harness 和 deprecated legacy CE。所以我们这次不是分析“Zep Cloud 内部怎么实现图谱记忆”，而是分析“Zep 如何接入 Agent 生态”。

## 2. 一句话主线

```text
create user/thread
  -> thread.get_user_context 注入 Context Block
  -> 模型推理 / 工具调用
  -> thread.add_messages 保存本轮对话
  -> graph.search / graph.add 作为按需图谱记忆工具
```

讲解重点：

- `thread.get_user_context`：自动取长期记忆。
- `thread.add_messages`：把对话写回 Zep。
- `graph.search`：模型按需搜索 facts、entities、episodes。
- `graph.add`：写业务数据、文档、JSON 或非对话内容。

## 3. 目录怎么讲

| 目录 | 分享口径 |
| --- | --- |
| `integrations/` | 这次源码精读主角，多框架接入包 |
| `examples/` | 使用示例，不是核心实现 |
| `mcp/zep-mcp-server` | 把 Zep graph/thread 能力变成 MCP tools |
| `zep-eval-harness/` | 记忆效果评测工具链 |
| `ontology/` | 默认 ontology 示例 |
| `legacy/` | deprecated Community Edition，不作为当前主线 |

## 4. 图怎么讲

### 4.1 最高层架构

架构图见：[architecture.mmd](architecture.mmd)。

讲法：

> 先看仓库边界：当前仓库是 Zep Cloud 的接入层集合。中间的 `integrations` 是主角，它把同一套 Zep Cloud API 包成 LangGraph、AutoGen、CrewAI、Vercel AI SDK 等框架能理解的扩展点。MCP server 和 eval harness 是工具侧补充。

### 4.2 Context Block 流程

流程图见：[context-flow.mmd](context-flow.mmd)。

讲法：

> Zep 对 conversational agent 的默认接入方式是 Context Block。每轮开始，先用 `thread.get_user_context` 取回 prompt-ready 的长期记忆，再注入 system message；本轮结束后用 `thread.add_messages` 保存对话，让云端 user graph 异步更新。

### 4.3 Graph Search 流程

流程图见：[graph-search-flow.mmd](graph-search-flow.mmd)。

讲法：

> Context Block 是自动注入，Graph Search Tool 是模型主动查。模型可以按 query 搜 edges、nodes、episodes 或 auto context。edges 通常代表 facts/relationships，最适合作为 Agent 长期记忆结果。

## 5. 源码精读口径

### 5.1 仓库边界

证据链：

- `README.md:20`：仓库包含 example code、framework integrations、tools。
- `README.md:31`：真正 powering Zep 的开源 temporal knowledge graph framework 指向 Graphiti。
- `README.md:43`：`legacy/` 是 deprecated Community Edition。

讲法：

> 所以这份源码不能像 mem0 那样沿一个 `Memory.add/search` 读完整内核。它要按“接入层”来读。

### 5.2 LangGraph

证据链：

- `context.py:48`：`get_zep_context()`。
- `context.py:155`：`build_system_message()`。
- `persistence.py:197`：`persist_messages()`。
- `store.py:86`：`ZepStore(BaseStore)`。
- `tools.py:94`：`create_graph_search_tool()`。

讲法：

> LangGraph 包最完整，覆盖了四类接入：Context Block 注入、消息持久化、BaseStore 兼容和 graph search tool。`ZepStore` 尤其值得讲，因为它没有假装 Zep 是 KV store，而是 backing store 管 exact get/put，Zep graph 管 semantic search。

### 5.3 AutoGen / CrewAI

证据链：

- `zep_autogen/memory.py:25`：`ZepUserMemory` 实现 AutoGen Memory。
- `zep_autogen/graph_memory.py:26`：`ZepGraphMemory`。
- `zep_crewai/user_storage.py:18`：`ZepUserStorage`。
- `zep_crewai/graph_storage.py:17`：`ZepGraphStorage`。
- `zep_crewai/tools.py:36`：`ZepSearchTool`。

讲法：

> AutoGen 是 Memory 接口，CrewAI 是 storage/tool contract。Zep 没有强推一个统一抽象，而是按每个框架最自然的扩展点接入。

### 5.4 Vercel AI SDK

证据链：

- `middleware.ts:116`：`createZepMiddleware()`。
- `middleware.ts:126`：`transformParams` 注入 Context Block。
- `middleware.ts:135`：只在新 user turn 注入。
- `helpers.ts:215`：`createZepOnFinish()`。
- `tools.ts:155`、`:211`、`:289`：search、remember、context 三类 tools。

讲法：

> Vercel AI SDK 这块很好讲“为什么要拆开”：middleware 只做 context injection，`onFinish` 才保存最终 turn。否则 tool loop 每一步都写入，会把中间工具调用过程污染成长期记忆。

## 6. 真实例子怎么讲

可以用这个例子：

> 用户说：“以后源码分析请用中文，先讲仓库边界，再讲主流程。我正在比较 mem0、Zep 和 LangGraph。”

Zep 接入层做的不是本地抽取 facts，而是调用云端能力：

1. 新一轮开始：`thread.get_user_context(thread_id)` 取回用户长期偏好和项目背景。
2. Agent 生成回答：system message 中带有 Context Block。
3. 本轮结束：`thread.add_messages(thread_id, user + assistant)` 保存对话。
4. 如果模型主动需要查记忆：调用 `graph.search(query, scope="edges")`。

这能讲清楚 Zep 和 mem0 的差别：

- mem0 文档里我们看的是 memory engine 内核。
- Zep 当前仓库里我们看的是 cloud memory service 的 framework adapters。

## 7. 局限性和使用边界

主动讲这几点，会显得分享更稳：

1. 当前仓库不是 Zep Cloud 服务端核心，不能从这里证明云端图谱算法细节。
2. Context Block 的组装逻辑在云端，本仓库只能看到调用点和注入方式。
3. `graph.add` / `thread.add_messages` 后不保证同一 turn 立刻可 search。
4. 多框架包很多，分享时不要逐目录念，要抽共同范式。
5. 长期 memory 仍然要考虑 user/thread 隔离、隐私、删除、脱敏和审计。
6. API 有长度和批量限制，源码里的 truncate、chunk、limit clamp 都是工程防线。

## 8. 和 mem0 / LangGraph 的关系

可以这样收束：

> mem0 是本地 memory layer，适合读 add/search 内核；Zep 当前仓库是 Zep Cloud 接入层集合，适合读 framework adapters；LangGraph 是状态图 runtime，适合管理流程。实际落地时，LangGraph 可以负责 Agent 流程，Zep 负责云端长期记忆，mem0 则适合需要本地可控 memory layer 的场景。

## 9. 建议演讲结构

不需要压缩到 15 分钟，建议 45 到 70 分钟：

1. **5 分钟：边界澄清。** 当前仓库是什么，不是什么；为什么不能按服务端内核讲。
2. **8 分钟：最高层架构。** integrations、examples、MCP、eval harness、ontology、legacy。
3. **15 分钟：LangGraph 精读。** Context Block、persistence、ZepStore、graph search tool。
4. **10 分钟：其他框架接入。** AutoGen Memory、CrewAI Storage/Tool、Vercel middleware/onFinish。
5. **8 分钟：真实例子。** 一轮对话如何取 context、生成回答、保存 turn、下轮召回。
6. **8 分钟：设计思想和边界。** Cloud adapter、graceful degradation、async ingestion、PII-safe logging。
7. **5 分钟：和 mem0 / LangGraph 对比。** 帮听众建立选型地图。

## 10. 收束口

> Zep 当前仓库最值得看的不是一个本地 memory class，而是它如何把云端图谱记忆能力包装成各框架可理解的扩展点。读懂 LangGraph、AutoGen、CrewAI、Vercel AI SDK、MCP 这几条接入线，就能读懂 Zep 在 Agent 生态里的工程定位。
