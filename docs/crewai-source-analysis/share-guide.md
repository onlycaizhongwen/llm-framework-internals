# CrewAI 源码分享讲解稿

这份文档用于分享 CrewAI 源码。建议按“定位 -> 目录 -> Crews 主流程 -> Flows 主流程 -> 设计思想”的顺序讲。

## 1. 开场定位

可以这样开场：

> CrewAI 是一个多 Agent 自动化框架，但它不是只有 Agent。源码里最重要的是两条主线：Crews 负责角色化 Agent 协作，Flows 负责事件驱动流程控制。Crew 适合自治协作，Flow 适合确定性编排。

## 2. 目录怎么讲

| 目录 | 分享口径 |
| --- | --- |
| `lib/crewai` | 主框架包，包含 Agent、Task、Crew、Flow、LLM、memory、events |
| `lib/crewai-tools` | 工具生态和 RAG/外部集成 |
| `lib/crewai-files` | 文件输入、上传、解析和处理 |
| `lib/crewai-core` | CLI/平台相关公共能力 |
| `lib/cli` | 创建项目、运行 crew/flow、部署等命令行 |

## 3. Crews 主流程

```text
Crew.kickoff()
  -> prepare inputs
  -> process: sequential / hierarchical
  -> _execute_tasks()
  -> Task.execute_sync / execute_async
  -> Task._execute_core()
  -> Agent.execute_task()
  -> agent_executor.invoke()
  -> LLM.call() + tools
  -> TaskOutput / CrewOutput
```

讲解口径：

> Crews 不是图运行时，而是任务编排器。Crew 决定任务怎么排，Task 定义每个执行单元，Agent 负责拿角色、目标、上下文和工具去完成任务。

源码证据：

```python
class Crew(FlowTrackable, BaseModel):
    """
    Represents a group of agents, defining how they should collaborate and the
    tasks they should perform.
    """
```

```python
if self.process == Process.sequential:
    result = self._run_sequential_process()
elif self.process == Process.hierarchical:
    result = self._run_hierarchical_process()
```

```python
result = agent.execute_task(
    task=self,
    context=context,
    tools=tools,
)
```

## 4. Flows 主流程

```text
@start / @listen / @router
  -> Flow.kickoff()
  -> kickoff_async()
  -> _execute_start_method()
  -> _execute_method()
  -> _execute_listeners()
  -> final output
```

讲解口径：

> Flow 是 CrewAI 的确定性流程控制层。它用装饰器把普通 Python 方法声明成入口、监听器和路由节点。

源码证据：

```python
class Flow(BaseModel, Generic[T], metaclass=FlowMeta):
    """Base class for all flows."""
```

```python
def start(condition: FlowTrigger | None = None) -> FlowMethodDecorator:
    """Marks a method as a flow's starting point."""
```

```python
def listen(condition: FlowTrigger) -> FlowMethodDecorator:
    """Creates a listener that executes when specified conditions are met."""
```

## 5. 设计思想

### 5.1 业务语义优先

CrewAI 的抽象是 `Agent`、`Task`、`Crew`、`Flow`，非常接近业务叙述。分享时可以说：它不是先暴露底层图或 runnable，而是先让用户用业务语言描述自动化。

### 5.2 Crews + Flows 双主线

Crews 解决“多 Agent 如何协作”，Flows 解决“流程如何被事件驱动”。这也是 CrewAI 和 LangGraph/LangChain 不同的地方。

### 5.3 分层执行

Crew 编排 Task，Task 委派 Agent，Agent 通过 Executor 调用 LLM 和工具。每层职责比较清楚。

### 5.4 装饰器 DSL

Flow 用 `@start`、`@listen`、`@router` 把方法变成流程节点。这种 DSL 让工作流写起来像普通 Python 类。

### 5.5 事件总线

CrewAI 通过 `CrewAIEventsBus` 发出 Crew、Task、Agent、LLM、Flow 事件，方便 tracing、streaming、telemetry 和 hooks。

### 5.6 Provider / Tool 适配

LLM 层通过 provider 推断和 LiteLLM/native SDK 适配模型，Tool 层通过 `BaseTool` 和结构化工具把外部能力交给 Agent。

## 6. 15 分钟分享节奏

1. 2 分钟：定位 CrewAI，强调 Crews + Flows。
2. 3 分钟：讲目录结构和 workspace。
3. 4 分钟：讲 Crews 执行链路。
4. 3 分钟：讲 Flows 装饰器和事件流。
5. 3 分钟：讲设计思想和与 LangChain/LangGraph 的差异。

## 7. 收束句

> CrewAI 的源码主线可以浓缩成一句话：Crew 把角色化 Agent 和任务组织成协作团队，Flow 把自动化流程组织成事件驱动链路，LLM、tools、memory、events 都是围绕这两条主线服务的横切能力。
