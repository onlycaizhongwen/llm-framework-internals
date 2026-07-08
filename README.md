# LLM Framework Internals

这个仓库用于沉淀开源 LLM / Agent 框架的源码分析资料。目标不是简单罗列 API，而是把每个项目的架构边界、主流程、核心设计思想、真实使用场景和可分享材料整理成一套可复用的中文文档。

主要阅读入口是 HTML 页面；Markdown 用于维护，Mermaid 文件用于架构图和流程图。

## 横向总览

| 主题 | HTML 阅读页 | Markdown 底稿 | 说明 |
| --- | --- | --- | --- |
| LLM / Agent 开源框架源码横向分析与选型指南 | `docs/cross-framework-analysis/index.html` | `docs/cross-framework-analysis/index.md` | 总览对比矩阵、框架选型指南、组合架构专题、源码设计范式、真实案例合集 |

## 内容导航

| 框架 | 源码目录 | 固定提交 | HTML 阅读页 | Markdown 底稿 | 分析重点 |
| --- | --- | --- | --- | --- | --- |
| LangChain | `sources/langchain` | `eb2dabb8b7102fbedb33016dcf10fe475efde88e` | `docs/langchain-source-analysis/index.html` | `docs/langchain-source-analysis/index.md` | `core`、`langchain_v1`、`partners`、`classic` 边界；Agent middleware、provider adapter、RAG、classic 迁移关系 |
| LangGraph | `sources/langgraph` | `d57a74f950b87bfb9cb51240cc8dccf34b5edfaa` | `docs/langgraph-source-analysis/index.html` | `docs/langgraph-source-analysis/index.md` | StateGraph、Pregel 执行模型、checkpoint、中断恢复、prebuilt Agent |
| Spring AI Alibaba | `sources/spring-ai-alibaba` | `4a823250415e7a42deb650410edf6948c35875bd` | `docs/spring-ai-alibaba-source-analysis/index.html` | `docs/spring-ai-alibaba-source-analysis/index.md` | Java / Spring Boot Agentic AI、Graph Core、ReactAgent、FlowAgent、Hook / Interceptor、Checkpoint、Nacos、A2A、MCP、Observation |
| CrewAI | `sources/crewai` | `2b90117e887ef68a22ccf9552a58ffaf96de1fc4` | `docs/crewai-source-analysis/index.html` | `docs/crewai-source-analysis/index.md` | Crew / Agent / Task 编排、Flow 流程、工具调用、与 LangGraph 对比 |
| AutoGen | `sources/autogen` | `027ecf0a379bcc1d09956d46d12d44a3ad9cee14` | `docs/autogen-source-analysis/index.html` | `docs/autogen-source-analysis/index.md` | AssistantAgent、消息运行时、GroupChat、团队协作范式、与 LangGraph 对比 |
| mem0 | `sources/mem0` | `cd79fa8914b5b1cf66daacc957d826065df57df8` | `docs/mem0-source-analysis/index.html` | `docs/mem0-source-analysis/index.md` | 记忆抽取、向量检索、图谱记忆、生命周期、与 LangGraph 组合方式 |
| Zep | `sources/zep` | `826c5492d9cc3a7caf92a9870529f29b5a8546e3` | `docs/zep-source-analysis/index.html` | `docs/zep-source-analysis/index.md` | 会话记忆、上下文组装、图检索、长期记忆服务化 |
| Graphiti | `sources/graphiti` | `62ff03ac5662d288ebd9f6aafb70d6ae4070c632` | `docs/graphiti-source-analysis/index.html` | `docs/graphiti-source-analysis/index.md` | 时序知识图谱、episode ingestion、实体关系抽取、搜索召回 |
| Headroom | `sources/headroom` | `48201345be16a8b5aad74e8c390850dce0f34ec4` | `docs/headroom-source-analysis/index.html` | `docs/headroom-source-analysis/index.md` | 上下文压缩、代理网关、MCP / CCR 适配、Rust 热路径、fail-open 设计 |
| Letta | `sources/letta` | `b76da9092518cbaa2d09042e52fdcbde69243e18` | `docs/letta-source-analysis/index.html` | `docs/letta-source-analysis/index.md` | legacy Letta server、AgentState、core/archival memory、tool-first loop、provider adapter、stream/run 管理 |
| LlamaIndex | `sources/llama_index` | `7fd33e00a8947183327e75aef14687c499d5c150` | `docs/llama-index-source-analysis/index.html` | `docs/llama-index-source-analysis/index.md` | RAG 数据框架、IngestionPipeline、VectorStoreIndex、RetrieverQueryEngine、StorageContext、AgentWorkflow |
| Haystack | `sources/haystack` | `08e8885794687bd60ea5a59a672a428e7528b48a` | `docs/haystack-source-analysis/index.html` | `docs/haystack-source-analysis/index.md` | Component、Pipeline、AsyncPipeline、DocumentStore、Hybrid RAG、Agent、ToolInvoker、Serialization、Breakpoint、Tracing |
| n8n | `sources/n8n` | `b000bbd2773d402ce879434b42a5b9fb8f7f106a` | `docs/n8n-source-analysis/index.html` | `docs/n8n-source-analysis/index.md` | 可视化工作流平台、WorkflowExecute、ActiveWorkflowManager、WebhookService、Queue/Worker、节点和凭证生态 |
| Dify | `sources/dify` | `main codeload snapshot, version 1.15.0, commit SHA unavailable` | `docs/dify-source-analysis/index.html` | `docs/dify-source-analysis/index.md` | LLM 应用开发平台、AI Workflow、RAG Pipeline、Agent v2、Model Provider、Plugin、dify-agent、graphon runtime |
| Hermes Agent | `sources/hermes-agent` | `f64e4f4f5768c18a53f44890747653bafcab2796` | `docs/hermes-agent-source-analysis/index.html` | `docs/hermes-agent-source-analysis/index.md` | 项目工程专题，不纳入横向总览；长期个人 Agent 产品、多入口复用、工具安全、SessionDB、Gateway 恢复、Plugin / Skill / MCP 边界、分享稿 |

## 推荐阅读方式

1. 先打开横向总览 `docs/cross-framework-analysis/index.html`，建立“产品平台、状态图运行时、RAG 数据层、记忆系统、自动化集成”的分类视角。
2. 再打开对应框架的 `index.html`，按「架构 -> 主流程 -> 设计思想 -> 真实例子 -> 对比分析」阅读。
3. 需要演讲或分享时，优先看 `share-guide.html`；没有单独分享页的项目直接使用 `index.html`。
4. 想看图的来源，可以打开同目录下的 `.mmd` 文件，里面是 Mermaid 架构图和流程图源码。
5. 想核对源码证据，可以进入 `sources/{framework}`，对照 README 中记录的 upstream commit。

## 文档结构约定

每个框架尽量保持相同结构：

```text
docs/{framework}-source-analysis/
  index.html          # 主要阅读页，适合浏览器打开
  index.md            # Markdown 分析底稿
  share-guide.html    # 可选，分享讲稿 HTML
  share-guide.md      # 可选，分享讲稿 Markdown
  *.mmd               # Mermaid 架构图 / 流程图
docs/cross-framework-analysis/
  index.html          # 横向总览、选型指南、组合架构、设计范式、真实案例
  index.md
  *.mmd
sources/{framework}/  # 上游源码入口，通常以 Git submodule 固定版本
```

分析内容优先覆盖：

- 架构边界：包、模块、运行时、外部依赖分别负责什么。
- 主流程：一次典型调用从入口到执行结束经过哪些关键节点。
- 设计思想：为什么这么拆层、为什么这么抽象、适合解决什么问题。
- 真实例子：用一个可理解的业务或开发场景解释源码设计。
- 对比分析：和 LangGraph、LangChain、记忆系统或多 Agent 框架的关系。
- 局限性：哪些场景不适合，或者需要和其他框架组合。

## 克隆方式

推荐连同源码子模块一起拉取：

```bash
git clone --recurse-submodules https://github.com/onlycaizhongwen/llm-framework-internals.git
```

如果已经普通克隆，可以在仓库根目录执行：

```bash
git submodule update --init --recursive
```

## 本次重点更新

- 补充 LangChain 专题分析：Agent middleware、provider adapter、RAG 细节、classic 迁移关系。
- 新增 Spring AI Alibaba 源码分析：Java / Spring Boot Agentic AI 框架、Graph Core 状态图运行时、ReactAgent、FlowAgent、上下文工程 Hook / Interceptor、Checkpoint、Nacos Config、A2A、MCP 和 Observation，并已融入横向总览。
- 新增 Headroom 源码分析：上下文压缩、代理网关、MCP / CCR 适配、Rust 压缩核心、应用场景和对比分析。
- 新增 Letta 源码分析：legacy Agent server、AgentState、Memory 分层、工具调用、Provider Adapter、Streaming / Run / Step。
- 新增 LlamaIndex 源码分析：core / integrations 边界、ingestion、index / retriever / query engine、StorageContext、AgentWorkflow。
- 新增 Haystack 源码分析：Component / Pipeline typed socket、同步与异步运行、Hybrid RAG、DocumentStore、Agent / ToolInvoker、PipelineTool、serialization、breakpoint、snapshot、tracing，并已融入横向总览。
- 新增 n8n 源码分析：可视化 workflow graph、节点契约、执行引擎、webhook 激活、queue/worker 扩展、与 LangGraph 组合边界。
- 新增 Dify 源码分析：LLM 应用平台架构、Workflow App 主流程、graphon 节点运行时、RAG Pipeline、Agent v2 / dify-agent、Plugin / Model Provider 治理、真实客服场景和与 LangGraph / n8n 对比。
- 新增 Hermes Agent 项目工程专题：长期个人 Agent 产品、多入口复用、Provider Adapter、工具执行安全边界、SessionDB 检索、Gateway 长任务恢复、Plugin / Skill / MCP 边界，并补充分享总览图、阅读 checklist 和演讲稿。
- 新增横向总览材料：总览对比矩阵、框架选型指南、组合架构专题、源码设计范式、真实案例合集。
- README 更新为总导航页，方便直接从 GitHub 进入 HTML / Markdown / 源码目录。

## 分享建议

对外分享源码分析时，建议不要从文件列表开始讲，而是按问题链路叙述：

1. 这个框架解决什么问题，边界在哪里。
2. 一次真实请求或任务如何流过核心模块。
3. 源码里最关键的抽象是什么，为什么这么设计。
4. 它适合什么场景，不适合什么场景。
5. 它和 LangGraph、LangChain 或记忆系统如何组合。

这样听众会先建立「这个框架为什么存在」的理解，再进入源码细节，阅读成本会低很多。
