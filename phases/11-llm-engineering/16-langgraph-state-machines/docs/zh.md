# 智能体状态机——图、节点、检查点

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 手写的 ReAct 循环是一个 `while True`。把同一个循环写成显式的图，你就可以做检查点、中断、分支，并在其中做时间旅行。智能体没有变。变的是围着它的运行壳。

**Type:** 动手做
**Languages:** Python
**Prerequisites:** 第 11 阶段 · 09（工具调用），第 11 阶段 · 14（模型上下文协议）
**Time:** ~75 分钟

## 问题

你交付了一个工具调用智能体。它在三轮之内能工作，然后出了问题：模型试了一个返回 500 的工具，用户在任务中途改了主意，或者智能体决定退一笔订单，却没有人类签字。`while True:` 循环没有钩子。你不能暂停它，不能倒回它，也不能分叉去问「如果模型选了另一个工具会怎样」。一旦你把这个演示交付出去，智能体就变成一个黑盒，要么成了，要么没成。

下一步一旦看见就很明显。智能体本来就是一台状态机——系统提示词加上消息历史加上待处理的工具调用加上下一步动作。把状态机显式化：节点是「模型思考」「工具运行」「人类批准」，边是它们之间的条件转移。图一旦显式，运行壳免费得到四样东西：检查点（在步骤之间保存状态）、中断（暂停给人类）、流式（流式输出 token 和中间事件），以及时间旅行（倒回到先前状态，试另一条分支）。

这种抽象的参考实现是 LangGraph。它不是 LangChain 那种意义上的智能体框架（「这是一个 AgentExecutor，祝你好运」）。它是一个把状态、持久化和中断都当作一等公民的图运行时。智能体循环是你画出来的，不是你手写出来的。

## 概念

![LangGraph StateGraph：节点、边与 checkpointer](../assets/langgraph-stategraph.svg)

一个 `StateGraph` 有三样东西。

1. **状态。** 一个带类型的 dict（TypedDict 或 Pydantic 模型），在图中流动。每个节点收到完整状态，并返回一份部分更新，LangGraph 用每个字段的 *reducer* 合并它——应当累积的列表用 `operator.add`，默认则是覆盖。
2. **节点。** Python 函数 `state -> partial_state`。每一个都是离散的一步：「调用模型」「运行工具」「做摘要」。
3. **边。** 节点之间的转移。静态边只去一个地方。条件边接受一个路由函数 `state -> next_node_name`，使图可以根据模型输出分支。

你编译这张图。编译绑定拓扑，挂上一个 checkpointer（可选，但对生产必不可少），并返回一个可运行对象。你用初始状态和一个 `thread_id` 调用它。执行的每一步都持久化一个检查点，键是 `(thread_id, checkpoint_id)`。

### 四种超能力

**检查点。** 每一次节点转移都把新状态写入存储（测试用内存，生产用 Postgres / Redis / SQLite）。用同一个 `thread_id` 再次调用图即可恢复。图从它暂停的地方继续。

**中断。** 用 `interrupt_before=["human_review"]` 标记一个节点，执行会在该节点运行之前停下。状态被持久化。你的 API 用「等待批准」回应用户。之后对同一个 `thread_id` 发出带 `Command(resume=...)` 的请求，执行继续。

**流式。** `graph.stream(state, mode="updates")` 在状态增量发生时产出它们。`mode="messages"` 流式输出模型节点内部的 LLM token。`mode="values"` 产出完整快照。你选择在 UI 里呈现什么。

**时间旅行。** `graph.get_state_history(thread_id)` 返回完整的检查点日志。把任何一个先前的 `checkpoint_id` 传给 `graph.invoke`，你就从那一点分叉。这对调试很好（「如果模型选了工具 B 会怎样？」），也对重放生产轨迹的回归测试很好。

### reducer 才是要点

每个状态字段都有一个 reducer。大多数默认值没问题——新值覆盖旧值。但消息列表需要 `operator.add`，这样新消息是追加而不是替换。并行边通过 reducer 合并它们的更新。如果两个节点都更新 `messages`，而你忘了 `Annotated[list, add_messages]`，第二个会静默获胜，你丢掉半轮。reducer 是这个库里唯一微妙的东西；把它做对，其余的就能组合起来。

### 四个节点里的 ReAct 图

一个生产级 ReAct 智能体是四个节点和两条边：

1. `agent`——用当前消息历史调用 LLM。返回助手消息（其中可能包含 tool_calls）。
2. `tools`——执行最后一条助手消息里的任何 tool_calls，把工具结果作为工具消息追加。
3. 一条来自 `agent` 的条件边：如果最后一条消息有 tool_calls，就路由到 `tools`，否则到 `END`。
4. 一条从 `tools` 回到 `agent` 的静态边。

就是这样。你得到完整的 ReAct 循环（Thought → Action → Observation → Thought → …），带检查点、中断和流式，大约 40 行代码。

### StateGraph 与 Send（扇出）

`Send(node_name, state)` 让一个节点分发并行子图。例子：智能体决定同时查询三个检索器。每个 `Send` 派生目标节点的一次并行执行；它们的输出通过状态 reducer 合并。这就是 LangGraph 表达编排器–工作者模式的方式，而不用线程原语。

### 子图

一张已编译的图可以是另一张图里的节点。外层图看到一个节点；内层图有自己的状态和自己的检查点。团队就是这样构建监督者–工作者智能体的：监督者图把用户意图路由到按领域划分的工作者子图。

```figure
l5-state-graph-ledger
```

## 动手做

### 第 1 步：状态与节点

```python
from typing import Annotated, TypedDict
from langchain_core.messages import AnyMessage, HumanMessage, AIMessage
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver

class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]

def agent_node(state: State) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

def should_continue(state: State) -> str:
    last = state["messages"][-1]
    return "tools" if getattr(last, "tool_calls", None) else END

tool_node = ToolNode(tools=[search_web, read_file])

graph = StateGraph(State)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)
graph.set_entry_point("agent")
graph.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
graph.add_edge("tools", "agent")

app = graph.compile(checkpointer=MemorySaver())
```

`add_messages` 是让消息列表累积而不是覆盖的 reducer。忘掉它是最常见的 LangGraph 缺陷。

### 第 2 步：带线程运行

```python
config = {"configurable": {"thread_id": "user-42"}}
for event in app.stream(
    {"messages": [HumanMessage("find the Anthropic headquarters address")]},
    config,
    stream_mode="updates",
):
    print(event)
```

每一次更新都是一个字典 `{node_name: state_delta}`。前端可以把这些流到 UI，让用户看到「智能体正在思考……正在调用 search_web……拿到结果……正在回答。」

### 第 3 步：加入人在回路中的中断

标记一个节点，使执行在它运行之前暂停。

```python
app = graph.compile(
    checkpointer=MemorySaver(),
    interrupt_before=["tools"],  # pause before every tool call
)

state = app.invoke({"messages": [HumanMessage("delete the production database")]}, config)
# state["__interrupt__"] is set. Inspect proposed tool calls.
# If approved:
from langgraph.types import Command
app.invoke(Command(resume=True), config)
# If denied: write a rejection message and resume
app.update_state(config, {"messages": [AIMessage("Blocked by human reviewer.")]})
```

状态、检查点和线程都跨中断持久化。除了执行期间，没有任何东西只活在内存里。

### 第 4 步：为调试做时间旅行

```python
history = list(app.get_state_history(config))
for snapshot in history:
    print(snapshot.values["messages"][-1].content[:80], snapshot.config)

# Fork from a prior checkpoint
target = history[3].config  # three steps back
for event in app.stream(None, target, stream_mode="values"):
    pass  # replay from that point forward
```

把 `None` 作为输入传入，会从给定的检查点重放；传入一个值，则会在恢复之前把它作为更新追加到该检查点的状态上。这就是你复现一次糟糕的智能体运行、而不必重跑整段对话的方式。

### 第 5 步：为生产换掉 checkpointer

```python
from langgraph.checkpoint.postgres import PostgresSaver

with PostgresSaver.from_conn_string("postgresql://...") as checkpointer:
    checkpointer.setup()
    app = graph.compile(checkpointer=checkpointer)
```

SQLite、Redis 和 Postgres 都已随库提供。`MemorySaver` 用于测试。任何要跨重启持久化的东西都需要真正的存储。

## 技能

> 你把智能体建成图，而不是 `while True` 循环。

在伸手去拿 LangGraph 之前，先做 60 秒的设计：

1. **给节点命名。** 每一个离散的决策或有副作用的动作都是一个节点。「智能体思考」「工具运行」「审阅者批准」「响应流式输出」。如果你列不出来，这个任务还不是智能体形状。
2. **声明状态。** 最小的 TypedDict，每个列表字段都有 reducer。不要把一切都塞进 `messages`；把任务专有字段（一份进行中的 `plan`、一个 `budget` 计数器、一个 `retrieved_docs` 列表）提升到顶层。
3. **画边。** 除非下一步取决于模型输出，否则用静态边。每条条件边都需要一个带命名分支的路由函数。
4. **事先选定 checkpointer。** 测试用 `MemorySaver`，其他一切用 Postgres / Redis / SQLite。不要没有它就交付——没有 checkpointer 就意味着不能恢复、不能中断、不能时间旅行。
5. **在工具运行之前决定中断，而不是之后。** 批准放在进入有副作用节点的边上，这样你可以在造成伤害之前取消；校验放在离开模型的边上，这样你可以便宜地拒绝坏调用。
6. **默认流式。** UI 用 `mode="updates"`，模型节点内部的 token 级流式用 `mode="messages"`，评测期间的完整快照用 `mode="values"`。

拒绝交付一个没有 checkpointer 的 LangGraph 智能体。拒绝交付一个在副作用 *之后* 才中断的智能体。拒绝交付一个 `messages` 字段却不用 `add_messages` 做 reducer 的智能体。

## 练习

1. **容易。** 用一个计算器工具和一个网页搜索工具实现上面的四节点 ReAct 图。验证 `list(app.get_state_history(config))` 对一段两轮对话至少返回四个检查点。
2. **中等。** 增加一个在 `agent` 之前运行的 `planner` 节点，把结构化的 `plan: list[str]` 写入状态。让 `agent` 把计划步骤标为完成。如果 `plan` 在检查点恢复之后丢失（用错了 reducer），就让测试失败。
3. **困难。** 构建一张监督者图，用 `Send` 在三个子图（`researcher`、`writer`、`reviewer`）之间路由。每个子图有自己的状态和 checkpointer。在外层图上加 `interrupt_before=["writer"]`，让人类可以批准研究简报。确认从先前检查点做时间旅行时，只重跑被分叉的那条分支。

## 关键术语

| 术语 | 人们怎么说 | 它实际的意思 |
|------|-----------------|-----------------------|
| StateGraph | 「LangGraph 的那张图」 | 你在编译之前往上添加节点和边的构建器对象。 |
| Reducer | 「字段如何合并」 | 节点为该字段返回更新时应用的函数 `(old, new) -> merged`；默认是覆盖，`add_messages` 则追加。 |
| 线程 | 「一段对话 ID」 | 一个 `thread_id` 字符串，限定一次会话的全部检查点。 |
| 检查点 | 「一个暂停的状态」 | 节点转移之后整张图状态的持久快照，键是 `(thread_id, checkpoint_id)`。 |
| 中断 | 「暂停给人类」 | `interrupt_before` / `interrupt_after` 在节点边界停止执行；用 `Command(resume=...)` 恢复。 |
| 时间旅行 | 「从先前一步分叉」 | `graph.invoke(None, config_with_old_checkpoint_id)` 从该检查点向前重放。 |
| Send | 「并行子图分发」 | 节点可以返回的构造器，用来派生目标节点的 N 次并行执行。 |
| 子图 | 「把已编译的图当作节点」 | 用作另一张图中节点的已编译 StateGraph；保留自己的状态作用域。 |

## 延伸阅读

- [LangGraph documentation](https://langchain-ai.github.io/langgraph/) — StateGraph、reducer、checkpointer 和中断的权威参考。
- [LangGraph concepts: state, reducers, checkpointers](https://langchain-ai.github.io/langgraph/concepts/low_level/) — 本课使用的心智模型，直接来自源头。
- [LangGraph Persistence and Checkpoints](https://langchain-ai.github.io/langgraph/concepts/persistence/) — Postgres / SQLite / Redis 存储、检查点命名空间和线程 ID 的细节。
- [LangGraph Human-in-the-loop](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/) — `interrupt_before`、`interrupt_after`、`Command(resume=...)`，以及编辑状态的模式。
- [Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models" (ICLR 2023)](https://arxiv.org/abs/2210.03629) — 每个 LangGraph 智能体都实现的模式；读它是为了推理轨迹的理由。
- [Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) — 偏好哪些图形状（链、路由器、编排器–工作者、评测器–优化器），以及何时使用。
- 第 11 阶段 · 09（工具调用）——每个 LangGraph 智能体节点都复用的工具调用原语。
- 第 11 阶段 · 14（模型上下文协议）——外部工具发现，经 MCP 适配器接入 LangGraph 的 `ToolNode`。
- 第 11 阶段 · 17（智能体框架取舍）——何时选 LangGraph，而不是 CrewAI、AutoGen 或 Agno。
