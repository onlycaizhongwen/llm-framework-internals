# LLM Framework Internals

这个仓库用于沉淀开源 LLM / Agent 框架的源码分析资料。目标不是简单罗列 API，而是把每个项目的架构边界、主流程、核心设计思想、真实使用场景和可分享材料整理成一套可复用的中文文档。

主要阅读入口是 HTML 页面；Markdown 用于维护，Mermaid 文件用于架构图和流程图。

## 内容导航

| 框架 | 源码目录 | 固定提交 | HTML 阅读页 | Markdown 底稿 | 分析重点 |
| --- | --- | --- | --- | --- | --- |
| LangChain | `sources/langchain` | `eb2dabb8b7102fbedb33016dcf10fe475efde88e` | `docs/langchain-source-analysis/index.html` | `docs/langchain-source-analysis/index.md` | `core`、`langchain_v1`、`partners`、`classic` 边界；Agent middleware、provider adapter、RAG、classic 迁移关系 |
| LangGraph | `sources/langgraph` | `d57a74f950b87bfb9cb51240cc8dccf34b5edfaa` | `docs/langgraph-source-analysis/index.html` | `docs/langgraph-source-analysis/index.md` | StateGraph、Pregel 执行模型、checkpoint、中断恢复、prebuilt Agent |
| CrewAI | `sources/crewai` | `2b90117e887ef68a22ccf9552a58ffaf96de1fc4` | `docs/crewai-source-analysis/index.html` | `docs/crewai-source-analysis/index.md` | Crew / Agent / Task 编排、Flow 流程、工具调用、与 LangGraph 对比 |
| AutoGen | `sources/autogen` | `027ecf0a379bcc1d09956d46d12d44a3ad9cee14` | `docs/autogen-source-analysis/index.html` | `docs/autogen-source-analysis/index.md` | AssistantAgent、消息运行时、GroupChat、团队协作范式、与 LangGraph 对比 |
| mem0 | `sources/mem0` | `cd79fa8914b5b1cf66daacc957d826065df57df8` | `docs/mem0-source-analysis/index.html` | `docs/mem0-source-analysis/index.md` | 记忆抽取、向量检索、图谱记忆、生命周期、与 LangGraph 组合方式 |
| Zep | `sources/zep` | `826c5492d9cc3a7caf92a9870529f29b5a8546e3` | `docs/zep-source-analysis/index.html` | `docs/zep-source-analysis/index.md` | 会话记忆、上下文组装、图检索、长期记忆服务化 |
| Graphiti | `sources/graphiti` | `62ff03ac5662d288ebd9f6aafb70d6ae4070c632` | `docs/graphiti-source-analysis/index.html` | `docs/graphiti-source-analysis/index.md` | 时序知识图谱、episode ingestion、实体关系抽取、搜索召回 |
| Headroom | `sources/headroom` | `48201345be16a8b5aad74e8c390850dce0f34ec4` | `docs/headroom-source-analysis/index.html` | `docs/headroom-source-analysis/index.md` | 上下文压缩、代理网关、MCP / CCR 适配、Rust 热路径、fail-open 设计 |

## 推荐阅读方式

1. 先打开对应框架的 `index.html`，按「架构 -> 主流程 -> 设计思想 -> 真实例子 -> 对比分析」阅读。
2. 需要演讲或分享时，优先看 `share-guide.html`；没有单独分享页的项目直接使用 `index.html`。
3. 想看图的来源，可以打开同目录下的 `.mmd` 文件，里面是 Mermaid 架构图和流程图源码。
4. 想核对源码证据，可以进入 `sources/{framework}`，对照 README 中记录的 upstream commit。

## 文档结构约定

每个框架尽量保持相同结构：

```text
docs/{framework}-source-analysis/
  index.html          # 主要阅读页，适合浏览器打开
  index.md            # Markdown 分析底稿
  share-guide.html    # 可选，分享讲稿 HTML
  share-guide.md      # 可选，分享讲稿 Markdown
  *.mmd               # Mermaid 架构图 / 流程图
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
- 新增 Headroom 源码分析：上下文压缩、代理网关、MCP / CCR 适配、Rust 压缩核心、应用场景和对比分析。
- README 更新为总导航页，方便直接从 GitHub 进入 HTML / Markdown / 源码目录。

## 分享建议

对外分享源码分析时，建议不要从文件列表开始讲，而是按问题链路叙述：

1. 这个框架解决什么问题，边界在哪里。
2. 一次真实请求或任务如何流过核心模块。
3. 源码里最关键的抽象是什么，为什么这么设计。
4. 它适合什么场景，不适合什么场景。
5. 它和 LangGraph、LangChain 或记忆系统如何组合。

这样听众会先建立「这个框架为什么存在」的理解，再进入源码细节，阅读成本会低很多。
