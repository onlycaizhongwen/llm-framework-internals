# Hermes Agent 源码分享稿 / 分享提纲

> 适用场景：团队内部源码分享、LLM Agent 产品工程复盘、和 LangGraph / CrewAI / AutoGen 等框架做定位对比。
> 推荐节奏：不强行压缩到 15 分钟，可以按 35-50 分钟展开；如果时间短，就只讲“开场、核心观点、三条主线、一个案例、总结”。

## 0. 开场

大家好，今天分享的是 `NousResearch/hermes-agent` 的源码分析。

先给这个项目定个位：Hermes Agent 不是 LangGraph、Dify、Haystack 那种“给开发者搭 Agent/RAG/Workflow 的框架”，它更像一个完整的个人 AI Agent 产品工程。

它关心的问题不是“怎么定义一个 graph 节点”或者“怎么把几个 Agent 组成 crew”，而是：

- 一个 Agent 如何同时跑在 CLI、Gateway、ACP、Cron、Desktop 这些入口里？
- 长期会话怎么恢复、怎么检索、怎么压缩？
- 工具执行怎么做审批和安全边界？
- Gateway 这种聊天平台里，用户连续发消息、Agent 长时间运行、进程重启时，系统怎么不丢任务？
- 插件、技能、MCP 这些扩展怎么接进来，同时不污染核心循环？

所以今天我不会按目录一行行讲，而是按一个长期运行 Agent 产品的生命周期讲：请求怎么进来，模型怎么跑，工具怎么控风险，状态怎么留下，结果怎么投递，重启后怎么恢复。

## 1. 核心观点

我的核心观点是：Hermes 的源码价值，不在于它抽象出一个漂亮的 Agent 框架，而在于它展示了一个长期在线、跨入口、可扩展、可恢复、要控制成本和风险的个人 Agent 产品，工程上到底需要补齐哪些层。

可以把 Hermes 理解成五层：

1. 多入口接入层：CLI、Gateway、ACP、Cron、Desktop。
2. Agent Core：`AIAgent + conversation_loop`。
3. 能力边缘：Tool Registry、Toolsets、Plugin、Skill、MCP。
4. 长期状态：SessionDB、FTS/trigram、memory、compression lineage。
5. 产品投递与恢复：DeliveryRouter、pending queue、resume_pending、Gateway restart recovery。

这里最重要的两个设计约束是：

- **Prompt cache first**：长会话里系统提示和工具 schema 不能频繁抖动，否则成本和稳定性都会变差。
- **Core narrow waist**：核心保持窄腰，能力长在边缘。模型供应商、工具、插件、平台接入都尽量不要污染主循环。

## 2. 三条主线

### 主线一：多入口最终回到同一个 Agent Core

第一条主线是入口复用。

Hermes 有很多入口：CLI、TUI、Messaging Gateway、ACP、Cron、Desktop。表面看很分散，但源码里它们最终都尽量回到同一个核心：`AIAgent + conversation_loop`。

CLI 入口负责解析参数，比如 `query`、`toolsets`、`skills`、`model`、`provider`、`worktree`。Gateway 负责把 Telegram、Discord、Slack 这类平台消息标准化成内部事件。ACP 负责把编辑器或客户端请求转换成 Agent 能理解的 content parts。Cron 负责定时触发，并收紧非交互场景下的工具边界。

但是这些入口不各自实现一套 Agent。真正的推理循环在 `agent/conversation_loop.py`，里面处理模型调用、工具分发、retry、fallback、compression、post-turn hooks、memory/skill review。

这说明 Hermes 的架构不是“每个入口写一套逻辑”，而是“入口处理协议差异，核心处理 Agent 生命周期”。

分享时可以强调一句：

> Hermes 的入口很多，但核心只有一个。入口负责把不同场景接进来，Agent Core 负责把一轮任务跑完整。

### 主线二：工具能力放在边缘，但安全边界必须前置

第二条主线是工具系统。

Hermes 的工具不是散落在主循环里的 if/else，而是通过 Tool Registry 管起来。工具模块 import 时注册 schema、handler、toolset 和校验逻辑。模型能看到哪些工具，由 `get_tool_definitions()` 根据 enabled/disabled toolsets 动态生成。

这样做有几个好处：

- 工具 schema、handler、toolset 分组有统一来源。
- CLI、Gateway、Cron 可以暴露不同工具集。
- MCP 和插件可以扩展工具，但仍然进入统一 registry。
- Gateway 长进程里工具 schema 可以缓存，registry generation 变化时再失效。

但工具系统最大的重点不是“能扩展”，而是“能控风险”。

Hermes 有专门的 `tools/approval.py` 管危险命令审批。它做几件事：

- 检测危险命令模式。
- 按 `session_key` 管理每会话审批状态。
- 支持 CLI 同步审批和 Gateway 异步审批。
- 对极危险命令做 hardline blocklist，即使 yolo 或 approval off 也不执行。
- Gateway 审批提示前做 secret redaction，避免把凭据发到聊天平台。

这里体现的是产品工程思维：长期在线 Agent 不能只依赖模型自觉。只要 Agent 能碰终端、文件、浏览器、MCP 外部服务，工具层就必须有 fail-closed 的安全边界。

分享时可以这样讲：

> Hermes 的工具系统不是单纯为了“让模型多做事”，而是为了“让模型在可控边界内做事”。

### 主线三：长期状态和 Gateway 恢复是产品复杂度的核心

第三条主线是长期状态。

如果只是一个 demo Agent，一轮对话结束就完了。但 Hermes 是长期个人 Agent，所以它必须解决：

- 会话怎么恢复？
- 历史怎么搜索？
- 中文和英文怎么检索？
- 长会话压缩后，旧事实怎么保留？
- Gateway 重启后，没跑完的任务怎么继续？

源码里 `hermes_state.py` 的 SessionDB 不只是消息表。`sessions` 存会话身份、模型配置、system prompt、父会话、token/cost 统计；`messages` 存消息、工具调用、reasoning、active/compacted 状态；FTS 和 trigram 分别支持全文检索和中文子串检索；`parent_session_id` 维护压缩 lineage。

Gateway 更能体现产品复杂度。聊天平台不是同步请求响应：

- 用户可能在 Agent 正在跑时继续发消息。
- Agent 可能正在等工具审批。
- 平台可能传图片、文件、语音。
- 进程可能重启。
- 多个会话不能互相污染。

所以 Gateway 里有 `_running_agents`、`_pending_messages`、overflow queue、`resume_pending`、run generation check、DeliveryRouter 等机制。重启或关闭前先标记 `resume_pending`，下次启动后 `_schedule_resume_pending_sessions()` 合成一个空事件，把原会话继续跑起来。

分享时可以强调：

> Hermes 和普通 Agent demo 的差异，不在模型调用，而在长期运行时的会话一致性、恢复、排队、审批和投递。

## 3. 真实案例

### 案例一：CLI 里让 Hermes 修复仓库 bug

假设用户运行：

```bash
hermes -w -q "检查当前仓库测试失败原因并修复"
```

这条链路可以这样讲：

1. CLI 解析 `-w`、query、model、provider、toolsets、skills。
2. `-w` 创建隔离 worktree，降低污染当前工作区的风险。
3. CLI 创建或恢复 `AIAgent`。
4. `AIAgent.run_conversation()` 转到 `agent.conversation_loop.run_conversation()`。
5. 模型看到 file/search/terminal 等工具 schema。
6. 模型发起工具调用，进入 `model_tools.handle_function_call()`。
7. 工具执行前经过 plugin hook、edit approval、dangerous command approval。
8. Tool Registry 分发 handler，结果回到模型。
9. 一轮结束后，turn finalizer 持久化会话、同步记忆、触发后台 review。

这个案例的重点不是“模型会写代码”，而是 Hermes 如何把代码任务变成一条可控的工具执行链。

### 案例二：Telegram 里创建定时总结任务

用户在 Telegram 里说：“每天早上总结 GitHub issue”。

链路是：

1. Telegram adapter 把平台消息转成统一 `MessageEvent`。
2. GatewayRunner 生成 `session_key`，找到或创建该会话对应的 AIAgent。
3. Agent 判断这是一个定时任务，调用 cronjob 相关工具。
4. scheduler 每 60 秒 tick，找到 due jobs。
5. Cron 执行时禁用 `cronjob`、`messaging`、`clarify` 等不适合非交互场景的工具。
6. 结果通过 DeliveryRouter 投递回 Telegram。

这个案例可以说明：Hermes 的复杂度来自“Agent 不是只在终端里跑一次”，而是会进入真实消息平台和后台任务系统。

### 案例三：Gateway 长任务执行中重启

用户在聊天平台发了一个长任务，Agent 正在执行工具。此时 Gateway 重启。

Hermes 的处理是：

1. Gateway 知道当前 session 有 running agent。
2. shutdown/restart 前先对这些 session 标记 `resume_pending`。
3. drain 超时后中断剩余 Agent。
4. 重启后 `_schedule_resume_pending_sessions()` 找到这些 session。
5. Gateway 合成一个空事件，让已有 resume_pending 分支负责恢复提示。
6. Agent 继续原会话，结束后再处理 pending queue 里的后续消息。

这个案例最好放在分享后半段，因为它能说明 Hermes 是产品工程，不只是调用模型和工具。

## 4. 和 LangGraph / CrewAI / AutoGen 的区别

可以用一句话区分：

> LangGraph 讲的是怎么编排 Agent，CrewAI 讲的是怎么分配角色和任务，AutoGen 讲的是怎么让多个 Agent 对话协作，而 Hermes 讲的是一个 Agent 真正长期在线以后，产品工程要补哪些层。

更具体一点：

| 项目 | 更适合讲什么 |
|---|---|
| LangGraph | 状态图、节点、边、checkpoint、中断恢复 |
| CrewAI | Role Agent、Task、Crew、Process，多角色协作 |
| AutoGen | 多 Agent 对话、消息路由、工具调用 runtime |
| Hermes Agent | 长期个人 Agent 产品，多入口、工具安全、状态恢复、平台投递 |

所以分享 Hermes 时，不要强行套 graph/node/reducer，也不要拿它和 Dify 那种平台型 workflow 做一一对齐。Hermes 的价值在于展示“个人 Agent 产品化”需要的完整工程层。

## 5. 总结

最后可以这样收束：

Hermes Agent 的源码可以给我们三个启发。

第一，Agent 产品的复杂度不只在模型调用。真正难的是多入口、多会话、长时间运行、工具安全、状态恢复和平台投递。

第二，核心要保持窄腰。`AIAgent + conversation_loop` 负责主生命周期，provider、tool、plugin、skill、MCP、platform adapter 都放在边缘扩展。

第三，长期在线 Agent 必须默认考虑风险。危险命令审批、hardline blocklist、session_key 隔离、secret redaction、resume_pending 都不是锦上添花，而是产品可用性的基础。

推荐结尾话术：

> Hermes Agent 的源码价值，不在于它给了我们一个新的 Agent 框架 API，而在于它把一个长期运行、跨入口、可恢复、可扩展、要控制成本和风险的个人 Agent 产品，拆成了可以学习的工程层次。

## 6. 现场讲述顺序

1. 开场定位：Hermes 是个人 Agent 产品工程，不是横向框架。
2. 核心观点：长期在线 Agent 需要补齐多入口、工具安全、状态恢复、投递。
3. 主线一：多入口如何回到同一个 Agent Core。
4. 主线二：工具系统如何扩展能力并控制风险。
5. 主线三：SessionDB 和 Gateway 恢复如何支撑长期运行。
6. 案例一：CLI 修复代码任务。
7. 案例二：Telegram 定时总结任务。
8. 案例三：Gateway 重启恢复长任务。
9. 对比：LangGraph / CrewAI / AutoGen 分别讲什么，Hermes 讲什么。
10. 总结：Hermes 的价值是产品化 Agent 的工程层次。
