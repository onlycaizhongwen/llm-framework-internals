# TASKS

## LangChain 源码架构分析文档
- 状态：已完成
- 摘要：基于 `langchain-src` 当前快照整理 LangChain 源码总架构、模块分支、核心流程，并产出 Markdown、HTML、架构图、流程图。
- 过程文件：`.codex/plans/main/langchain-source-analysis/process.md`
- 恢复提示：如需继续深入，读取 `process.md` 顶部恢复胶囊，按 provider、agent middleware、classic 迁移或 RAG 细节扩展专题文档。

## LangChain 上游源码入口
- 状态：已完成
- 摘要：以 Git submodule 方式把 `langchain-ai/langchain` 固定到 `sources/langchain`，让 GitHub 仓库同时具备分析文档和可追溯源码入口。
- 过程文件：无，短任务直接闭环。
- 恢复提示：后续新增框架时沿用 `sources/{framework}` submodule + `docs/{framework}-source-analysis/` 文档目录的结构。

## LangGraph 上游源码入口
- 状态：已完成
- 摘要：以 Git submodule 方式把 `langchain-ai/langgraph` 固定到 `sources/langgraph`，作为后续 LangGraph 源码分析的源码入口。
- 过程文件：无，短任务直接闭环。
- 恢复提示：下一步可基于 `sources/langgraph` 建立 `docs/langgraph-source-analysis/`，先整理架构、分支、核心流程、架构图和流程图。

## LangGraph 源码架构分析文档
- 状态：已完成
- 摘要：基于 `sources/langgraph` 固定提交整理 LangGraph 总架构、源码分支、Pregel 执行脉络、checkpoint 设计、prebuilt Agent 组合方式，并产出 Markdown、HTML、架构图、流程图和分享讲解稿。
- 过程文件：无，短任务直接闭环。
- 恢复提示：如需继续深入，可按 checkpoint 存储实现、interrupt/human-in-the-loop、stream transformer、LangGraph Server/SDK 四条专题扩展。
