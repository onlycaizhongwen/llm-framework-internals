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
| Spring AI | `sources/spring-ai-main` | `main codeload snapshot, version 2.0.1-SNAPSHOT, commit SHA unavailable` | `docs/spring-ai-source-analysis/index.html` | `docs/spring-ai-source-analysis/index.md` | Java / Spring Boot AI 应用框架、ChatClient、Advisor、Model Provider Adapter、Tool Calling、RAG / VectorStore、ChatMemory、MCP、AutoConfiguration、Observation |
| Spring AI Alibaba | `sources/spring-ai-alibaba` | `4a823250415e7a42deb650410edf6948c35875bd` | `docs/spring-ai-alibaba-source-analysis/index.html` | `docs/spring-ai-alibaba-source-analysis/index.md` | Java / Spring Boot Agentic AI、Graph Core、ReactAgent、FlowAgent、Hook / Interceptor、Checkpoint、Nacos、A2A、MCP、Observation |
| Google ADK Python | `sources/adk-python` | `2de8a53525261a0dcf53588fbbb5ebd7d4c3a9c2` | `docs/adk-python-source-analysis/index.html` | `docs/adk-python-source-analysis/index.md` | Google Agent Development Kit 2.0、Agent / Workflow / Runner、Event / Session、Task API、HITL / Resume、ADK Web / API Server、Evaluation、Gemini / Vertex / Google Cloud 生态 |
| Microsoft Agent Framework | `sources/agent-framework` | `23bfa4957521c50326ad68daee93c9f5b251f38c` | `docs/agent-framework-source-analysis/index.html` | `docs/agent-framework-source-analysis/index.md` | Python / .NET 生产级 Agent / Workflow 框架、Agent / AIAgent、ChatClient / provider adapter、middleware、tools、Workflow / orchestration、Durable Agents、Azure Functions、OpenTelemetry、Evaluation |
| TaskWeaver | `sources/TaskWeaver-main` | `main codeload snapshot, version.json prod 0.0.12, commit SHA unavailable` | `docs/taskweaver-source-analysis/index.html` | `docs/taskweaver-source-analysis/index.md` | Microsoft code-first 数据分析 Agent：Planner / CodeInterpreter、Python 代码生成与验证、Jupyter / Docker 执行服务、Plugin、Memory、Event、artifact |
| Semantic Kernel | `sources/semantic-kernel` | `main codeload snapshot, short SHA 83bffe1, full SHA unavailable due API rate limit` | `docs/semantic-kernel-source-analysis/index.html` | `docs/semantic-kernel-source-analysis/index.md` | Microsoft 应用内 AI Kernel：Kernel / Plugin / KernelFunction / FunctionChoiceBehavior / AgentGroupChat / Process Framework / Vector Search，并说明与 Microsoft Agent Framework 的后继迁移关系 |
| CrewAI | `sources/crewai` | `2b90117e887ef68a22ccf9552a58ffaf96de1fc4` | `docs/crewai-source-analysis/index.html` | `docs/crewai-source-analysis/index.md` | Crew / Agent / Task 编排、Flow 流程、工具调用、与 LangGraph 对比 |
| AutoGen | `sources/autogen` | `027ecf0a379bcc1d09956d46d12d44a3ad9cee14` | `docs/autogen-source-analysis/index.html` | `docs/autogen-source-analysis/index.md` | AssistantAgent、消息运行时、GroupChat、团队协作范式、与 LangGraph 对比 |
| mem0 | `sources/mem0` | `cd79fa8914b5b1cf66daacc957d826065df57df8` | `docs/mem0-source-analysis/index.html` | `docs/mem0-source-analysis/index.md` | 记忆抽取、向量检索、图谱记忆、生命周期、与 LangGraph 组合方式 |
| Zep | `sources/zep` | `826c5492d9cc3a7caf92a9870529f29b5a8546e3` | `docs/zep-source-analysis/index.html` | `docs/zep-source-analysis/index.md` | 会话记忆、上下文组装、图检索、长期记忆服务化 |
| Graphiti | `sources/graphiti` | `62ff03ac5662d288ebd9f6aafb70d6ae4070c632` | `docs/graphiti-source-analysis/index.html` | `docs/graphiti-source-analysis/index.md` | 时序知识图谱、episode ingestion、实体关系抽取、搜索召回 |
| Headroom | `sources/headroom` | `48201345be16a8b5aad74e8c390850dce0f34ec4` | `docs/headroom-source-analysis/index.html` | `docs/headroom-source-analysis/index.md` | 上下文压缩、代理网关、MCP / CCR 适配、Rust 热路径、fail-open 设计 |
| Letta | `sources/letta` | `b76da9092518cbaa2d09042e52fdcbde69243e18` | `docs/letta-source-analysis/index.html` | `docs/letta-source-analysis/index.md` | legacy Letta server、AgentState、core/archival memory、tool-first loop、provider adapter、stream/run 管理 |
| LlamaIndex | `sources/llama_index` | `7fd33e00a8947183327e75aef14687c499d5c150` | `docs/llama-index-source-analysis/index.html` | `docs/llama-index-source-analysis/index.md` | RAG 数据框架、IngestionPipeline、VectorStoreIndex、RetrieverQueryEngine、StorageContext、AgentWorkflow |
| Haystack | `sources/haystack` | `08e8885794687bd60ea5a59a672a428e7528b48a` | `docs/haystack-source-analysis/index.html` | `docs/haystack-source-analysis/index.md` | Component、Pipeline、AsyncPipeline、DocumentStore、Hybrid RAG、Agent、ToolInvoker、Serialization、Breakpoint、Tracing |
| PydanticAI | `sources/pydantic-ai` | `6a6d83d95d9c1fc37fcd808b3633241d7d2656ce` | `docs/pydantic-ai-source-analysis/index.html` | `docs/pydantic-ai-source-analysis/index.md` | typed Agent、Pydantic schema、tool/function schema、provider adapter、pydantic_graph、pydantic_evals、MCP、Logfire/OTel |
| OpenAI Agents Python | `sources/openai-agents-python` | `158b2f489ecf2f9aeea7a84cb53cc03fe930daea` | `docs/openai-agents-python-source-analysis/index.html` | `docs/openai-agents-python-source-analysis/index.md` | 轻量多 Agent SDK、Agent / Runner、handoff、guardrail、tools、MCP、session、tracing、realtime、sandbox、OpenAI Responses / WebSocket |
| OpenAI Swarm | `sources/swarm` | `main codeload snapshot, short SHA 6af0b4c, full SHA unavailable` | `docs/swarm-source-analysis/index.html` | `docs/swarm-source-analysis/index.md` | experimental / educational 多 Agent 编排原型：Agent、function tools、handoff、context_variables、streaming、execute_tools=False evals，以及向 OpenAI Agents Python 的迁移关系 |
| Agno | `sources/agno` | `02f13bb182fe2afdf8a6ceea80b36b14d14b5f38` | `docs/agno-source-analysis/index.html` | `docs/agno-source-analysis/index.md` | Agent platform SDK、Agent / Team / Workflow / AgentOS、生产 API、MCP、审批、记忆、知识、调度、与 LangGraph / CrewAI / PydanticAI / Dify 对比 |
| AgentScope | `sources/agentscope` | `39efe8af2b56f5121e8ddaf5f4755e3ecfd723a7` | `docs/agentscope-source-analysis/index.html` | `docs/agentscope-source-analysis/index.md` | 生产级 Agent runtime / 服务框架：Agent loop、事件流、middleware、权限/HITL、workspace/sandbox、RAG、长期记忆、多会话服务 |
| n8n | `sources/n8n` | `b000bbd2773d402ce879434b42a5b9fb8f7f106a` | `docs/n8n-source-analysis/index.html` | `docs/n8n-source-analysis/index.md` | 可视化工作流平台、WorkflowExecute、ActiveWorkflowManager、WebhookService、Queue/Worker、节点和凭证生态 |
| Dify | `sources/dify` | `main codeload snapshot, version 1.15.0, commit SHA unavailable` | `docs/dify-source-analysis/index.html` | `docs/dify-source-analysis/index.md` | LLM 应用开发平台、AI Workflow、RAG Pipeline、Agent v2、Model Provider、Plugin、dify-agent、graphon runtime |
| Langflow | `sources/langflow-main` | `main codeload snapshot, version 1.10.2, commit SHA unavailable` | `docs/langflow-source-analysis/index.html` | `docs/langflow-source-analysis/index.md` | 可视化 AI Agent / Workflow 平台、React Flow 画布、Graph/Vertex 执行、Component/Template、API/Playground、MCP 发布、与 Dify/n8n/LangGraph 对比 |
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
- 新增 Google ADK Python 源码分析：ADK 2.0 Agent / Workflow / Runner 主链路、Event / Session 事件模型、Workflow 图式运行时、RequestInput HITL / Resume、Task API / AgentTool、ADK Web / API Server、Evaluation 评测闭环，以及与 LangGraph / OpenAI Agents Python / Agno / Dify 的边界。
- 新增 Microsoft Agent Framework 源码分析：Python / .NET 双语 Agent runtime、AIAgent / AgentResponse、ChatClient provider adapter、middleware/context/session、tool approval/security、Workflow / orchestration、Durable Agents / Azure Functions hosting、OpenTelemetry / Evaluation，并已融入横向总览。
- 新增 TaskWeaver 源码分析：Microsoft code-first 数据分析 Agent，覆盖 Planner / CodeInterpreter 主链路、Python 代码生成与 AST 验证、Jupyter / Docker 执行服务、Plugin YAML 到函数签名、Memory / Event 协议、真实数据分析案例，以及与 LangGraph / AgentScope / OpenAI Agents Python / Agno / RAG 框架的边界。
- 新增 Semantic Kernel 源码分析：Kernel 应用内 AI 容器、Plugin / KernelFunction 工具化、FunctionChoiceBehavior 自动工具调用、ChatCompletionAgent / AgentGroupChat、Process Framework、Vector Search / RAG 工具化，以及与 Microsoft Agent Framework 的后继迁移关系。
- 新增 Headroom 源码分析：上下文压缩、代理网关、MCP / CCR 适配、Rust 压缩核心、应用场景和对比分析。
- 新增 Letta 源码分析：legacy Agent server、AgentState、Memory 分层、工具调用、Provider Adapter、Streaming / Run / Step。
- 新增 LlamaIndex 源码分析：core / integrations 边界、ingestion、index / retriever / query engine、StorageContext、AgentWorkflow。
- 新增 Haystack 源码分析：Component / Pipeline typed socket、同步与异步运行、Hybrid RAG、DocumentStore、Agent / ToolInvoker、PipelineTool、serialization、breakpoint、snapshot、tracing，并已融入横向总览。
- 新增 n8n 源码分析：可视化 workflow graph、节点契约、执行引擎、webhook 激活、queue/worker 扩展、与 LangGraph 组合边界。
- 新增 Dify 源码分析：LLM 应用平台架构、Workflow App 主流程、graphon 节点运行时、RAG Pipeline、Agent v2 / dify-agent、Plugin / Model Provider 治理、真实客服场景和与 LangGraph / n8n 对比。
- 新增 Langflow 源码分析：可视化 AI Agent / Workflow 平台，覆盖 React Flow 画布、Flow data、Graph / Vertex 执行、Component / Template、Tool Mode、API / Playground、MCP 发布、客服 RAG 真实案例，以及与 Dify / n8n / LangGraph / LangChain / Flowise 的边界。
- 新增 Hermes Agent 项目工程专题：长期个人 Agent 产品、多入口复用、Provider Adapter、工具执行安全边界、SessionDB 检索、Gateway 长任务恢复、Plugin / Skill / MCP 边界，并补充分享总览图、阅读 checklist 和演讲稿。
- 新增 Agno 源码分析：Agent platform SDK、Agent / Team / Workflow / AgentOS 四层架构、Agent 主运行链路、Team 协作模式、Workflow HITL、AgentOS 服务化、审批/记忆/知识、真实应用场景，以及与 LangGraph / CrewAI / PydanticAI / Dify 的对比。
- 新增 OpenAI Agents Python 源码分析：Agent / Runner 主循环、handoff 工具化、guardrail tripwire、ToolExecutionPlan、MCP approval、session / compaction、tracing spans、RealtimeRunner、SandboxAgent，以及与 LangGraph / PydanticAI / Agno / Dify 的边界。
- 新增 OpenAI Swarm 源码分析：experimental / educational 多 Agent 编排原型、Agent / Result / Response、Chat Completions run loop、function tools、handoff as function result、context_variables、streaming、execute_tools=False evals，以及向 OpenAI Agents Python 的迁移关系。
- 新增 AgentScope 源码分析：生产级 Agent runtime / 服务框架，覆盖 Agent 主循环、事件/消息模型、middleware、工具权限与 HITL、workspace/sandbox、RAG、长期记忆、多会话服务，以及与 LangGraph / OpenAI Agents Python / Agno / Dify 的边界。
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
