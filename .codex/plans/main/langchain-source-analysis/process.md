# LangChain 源码架构分析文档

## 恢复胶囊

- 任务需求：基于 `langchain-src` 当前最新快照做源码分析，先整理总架构，再按分支/模块展开，产出 Markdown、HTML、架构图、流程图。
- 关键决策：文档产物放在 `docs/langchain-source-analysis/`；HTML 是主要阅读入口；Markdown 是结构化底稿；图使用 Mermaid 并另存 `.mmd`。
- 当前阶段：已完成第一版交付。
- 已完成产物：`docs/langchain-source-analysis/index.md`、`index.html`、`architecture.mmd`、`agent-flow.mmd`、`rag-flow.mmd`。
- 剩余工作：如需继续深入，可按 provider、agent middleware、classic 迁移、RAG 实现细节分别扩展专题文档。
- 重要发现：仓库是多包 monorepo；`libs/core` 是协议底座；`libs/langchain_v1` 是当前 `langchain` 主包；`create_agent` 基于 LangGraph `StateGraph`；`libs/langchain` 是 `langchain-classic` 兼容分支。

## 步骤列表

- [v] 建立任务记录和产物目录。
- [v] 扫描 LangChain 源码结构、核心抽象、入口链路和测试/文档佐证。
  - 当前产物：`process.md`
  - 下一步：读取 `langchain-src` 下 README、pyproject、libs 结构、核心模块与典型流程源码。
  - 涉及文件：`langchain-src/`, `docs/langchain-source-analysis/`
- [v] 生成 Markdown 分析文档。
- [v] 生成 HTML 阅读文档。
- [v] 生成 Mermaid 架构图和流程图。
- [v] 验证产物完整性与源码引用。

## 研究发现

- `libs/core/README.md` 与 `libs/core/pyproject.toml` 证明 `langchain-core` 是基础抽象包。
- `libs/langchain_v1/pyproject.toml` 证明当前 `langchain` 主包依赖 `langchain-core` 与 `langgraph`。
- `libs/langchain_v1/langchain/agents/factory.py` 证明 `create_agent` 创建并编译 LangGraph `StateGraph`。
- `libs/langchain/README.md` 与 `libs/langchain/pyproject.toml` 证明 classic 分支发布为 `langchain-classic`。
- `libs/partners/openai` 和 `libs/partners/qdrant` 证明 partner 包通过实现 core 接口适配 provider SDK。

## 错误记录

- 外层目录不是 git 仓库，`git status --short` 不适用于外层工作区；已改用文件存在、HTML section、Markdown 引用和源码路径存在性验证。
