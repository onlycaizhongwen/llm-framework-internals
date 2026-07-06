# LLM Framework Internals

这个仓库用于沉淀开源 LLM 框架的源码分析资料，目标是把不同框架的架构、核心设计思想、源码主流程和可分享材料整理成可复用文档。

## 当前内容

### LangChain

分析文档：`docs/langchain-source-analysis/`

源码入口：`sources/langchain/`

当前固定源码提交：

```text
eb2dabb8b7102fbedb33016dcf10fe475efde88e
```

- `index.html`：LangChain 源码架构分析主阅读页
- `index.md`：Markdown 版分析底稿
- `architecture.mmd`：总架构图
- `agent-flow.mmd`：Agent 主流程图
- `rag-flow.mmd`：RAG / 检索增强流程图
- `share-guide.html`：面向分享的讲解稿 HTML
- `share-guide.md`：面向分享的讲解稿 Markdown

### LangGraph

分析文档：`docs/langgraph-source-analysis/`

源码入口：`sources/langgraph/`

当前固定源码提交：

```text
d57a74f950b87bfb9cb51240cc8dccf34b5edfaa
```

- `index.html`：LangGraph 源码架构分析主阅读页
- `index.md`：Markdown 版分析底稿
- `architecture.mmd`：总架构图
- `execution-flow.mmd`：Pregel 执行流程图
- `agent-flow.mmd`：prebuilt Agent 流程图
- `share-guide.html`：面向分享的讲解稿 HTML
- `share-guide.md`：面向分享的讲解稿 Markdown

### CrewAI

分析文档：`docs/crewai-source-analysis/`

源码入口：`sources/crewai/`

当前固定源码提交：

```text
2b90117e887ef68a22ccf9552a58ffaf96de1fc4
```

- `index.html`：CrewAI 源码架构分析主阅读页
- `index.md`：Markdown 版分析底稿
- `architecture.mmd`：总架构图
- `crew-flow.mmd`：Crew 执行流程图
- `flow-flow.mmd`：Flow 执行流程图
- `share-guide.html`：面向分享的讲解稿 HTML
- `share-guide.md`：面向分享的讲解稿 Markdown

### AutoGen

分析文档：`docs/autogen-source-analysis/`

源码入口：`sources/autogen/`

当前固定源码提交：

```text
027ecf0a379bcc1d09956d46d12d44a3ad9cee14
```

- `index.html`：AutoGen 源码架构分析主阅读页
- `index.md`：Markdown 版分析底稿
- `architecture.mmd`：总架构图
- `agent-flow.mmd`：AssistantAgent 执行流程图
- `groupchat-flow.mmd`：GroupChat 团队编排流程图
- `share-guide.html`：面向分享的讲解稿 HTML
- `share-guide.md`：面向分享的讲解稿 Markdown

## 仓库约定

- 本仓库优先保存分析文档，并通过 `sources/` 下的 Git submodule 固定上游源码版本。
- 每个源码入口都应记录对应的 upstream commit，方便复现分析结论和对比版本差异。
- 本地临时源码快照仍可克隆到 `*-src/` 目录，例如 `langchain-src/`，这些目录会被 `.gitignore` 排除。
- 每个框架建议单独建立一个 `docs/{framework}-source-analysis/` 目录。
- HTML 用于快速阅读和分享，Markdown 用于维护和二次整理，Mermaid 用于架构图和流程图。

## 克隆方式

如果需要连同源码入口一起拉取：

```bash
git clone --recurse-submodules https://github.com/onlycaizhongwen/llm-framework-internals.git
```

如果已经普通克隆，可以在仓库根目录执行：

```bash
git submodule update --init --recursive
```

## 后续计划

- 继续补充 LangChain 的专题分析：Agent middleware、provider adapter、RAG 细节、classic 迁移关系。
- 增加更多开源 LLM 框架的源码分析，例如 LlamaIndex、Haystack、LangGraph、vLLM 等。
