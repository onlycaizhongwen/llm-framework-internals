# LLM Framework Internals

这个仓库用于沉淀开源 LLM 框架的源码分析资料，目标是把不同框架的架构、核心设计思想、源码主流程和可分享材料整理成可复用文档。

## 当前内容

### LangChain

目录：`docs/langchain-source-analysis/`

- `index.html`：LangChain 源码架构分析主阅读页
- `index.md`：Markdown 版分析底稿
- `architecture.mmd`：总架构图
- `agent-flow.mmd`：Agent 主流程图
- `rag-flow.mmd`：RAG / 检索增强流程图
- `share-guide.html`：面向分享的讲解稿 HTML
- `share-guide.md`：面向分享的讲解稿 Markdown

## 仓库约定

- 本仓库保存分析文档，不直接提交上游框架源码。
- 上游源码可在本地克隆到 `*-src/` 目录，例如 `langchain-src/`，这些目录会被 `.gitignore` 排除。
- 每个框架建议单独建立一个 `docs/{framework}-source-analysis/` 目录。
- HTML 用于快速阅读和分享，Markdown 用于维护和二次整理，Mermaid 用于架构图和流程图。

## 后续计划

- 继续补充 LangChain 的专题分析：Agent middleware、provider adapter、RAG 细节、classic 迁移关系。
- 增加更多开源 LLM 框架的源码分析，例如 LlamaIndex、Haystack、LangGraph、vLLM 等。

