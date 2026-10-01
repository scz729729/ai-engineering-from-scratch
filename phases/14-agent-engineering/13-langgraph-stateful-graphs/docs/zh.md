# 有状态图编排 — 持久执行与检查点

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 智能体是一台状态机；节点是函数；边是转移；状态在每个节点之后做成检查点。从任意故障处，在最后一次成功的检查点恢复。LangGraph 是 2026 年这一套低层有状态编排模型的参考。

**Type:** 理解 + 动手做
**Languages:** Python（标准库）
**Prerequisites:** Phase 14 · 01（智能体循环）、Phase 14 · 12（工作流模式）
**Time:** ~75 分钟

## 学习目标

- 描述 LangGraph 的核心模型：带类型状态的状态机、函数节点、条件边，以及节点之后的检查点。
- 说出文档强调的四项能力：持久执行、流式输出、人在回路、全面的记忆。
- 解释 LangGraph 支持的三种编排拓扑：监督者、对等（swarm）、层次化（嵌套子图）。
- 用标准库实现一张状态图，带类型状态、条件边，以及一轮检查点 / 恢复循环。

## 问题

智能体和工作流共享一个问题：一次 40 步的运行在第 38 步失败时，你想从第 38 步恢复，而不是从头再来。二等的状态模型会逼着运维人员围着一个假定每次都是全新运行的库去拼重试。

LangGraph 的设计答案是：状态是一等的、带类型的对象，变更是显式的，检查点在每个节点之后落盘。恢复就是一次 `load_state(session_id)` 调用。

## 概念

### 图

一张图由这些东西定义：

- **状态类型。** 一个带类型的 dict（或 Pydantic 模型），每个节点都读它、改它。
- **节点。** 纯函数 `(state) -> state_update`。更新在返回之后被合并进状态。
- **边。** 节点之间的条件转移或直接转移。
- **入口和出口。** `START` 和 `END` 哨兵节点标出边界。

例子：一个带 `classify`、`refund`、`bug`、`sales`、`done` 节点的智能体 — 把路由工作流做成一张图。

### 持久执行

每个节点返回之后，运行时把状态序列化，写进检查点器（SQLite、Postgres、Redis、自定义）。第 N 步失败时，运行时可以 `resume(session_id)`，带着精确状态从第 N+1 步接着做。

LangGraph 文档明确点出了这件事要紧的生产用户：Klarna、Uber、J.P. Morgan。主张并不在图的形状；而在于图的形状加上检查点，让恢复变得便宜。

### 流式输出

每个节点都可以产出部分输出。图把逐节点增量事件流给调用方，界面会随着图的运行而更新。

### 人在回路

在节点之间检查并修改状态。做法是：在关键节点之前暂停，把状态摆到人面前，接受修改，然后恢复。检查点器让这件事容易，因为状态已经序列化好了。

### 记忆

短期（一次运行之内 — 状态里的对话历史）和长期（跨多次运行 — 经由检查点器，再加上单独的长期存储来持久化）。LangGraph 通过工具与外部记忆系统（Mem0、自定义）集成。

### 三种拓扑

1. **监督者（supervisor）。** 中心路由 LLM 把工作分派给专家子智能体。`langgraph-supervisor` 里的 `create_supervisor()`（不过 LangChain 团队在 2026 年建议直接用工具调用来做，以便对上下文有更多控制）。
2. **Swarm / 对等。** 智能体通过共享的工具表面直接交接。没有中心路由器。
3. **层次化。** 监督者管理子监督者，实现为嵌套子图。

### 这个模式会在哪里出错

- **检查点太小。** 只对对话回合做检查点，会让工具状态和记忆写入无法恢复。完整状态必须能序列化。
- **非确定性节点。** 恢复假定节点输入会产生相同的状态更新。随机种子、墙上时钟、外部 API 都必须被捕获。
- **条件边用得过多。** 每条边都是条件边的图，是一台无法被推理的状态机。优先用偶尔带分支的线性链。

```figure
langgraph-state
```

## 动手做

`code/main.py` 用标准库实现一张有状态图：

- `State` — 带 `messages`、`step`、`route`、`output`、`human_approval` 的带类型 dict。
- `Node` — 接受状态、返回更新 dict 的可调用对象。
- `StateGraph` — 节点 + 边 + 条件边 + 运行 + 恢复。
- `SQLiteCheckpointer`（内存里的假实现）— 每个节点之后序列化状态；`load(session_id)` 负责恢复。
- 一张演示图：classify -> 分支（refund / bug / sales）-> 人工门 -> 发送。

运行：

```
python3 code/main.py
```

轨迹显示第一次运行在人工门失败、状态被持久化，然后恢复并产出最终输出。

## 使用

- **LangGraph** — 参考实现，可以上生产。用 `create_react_agent`、`create_supervisor`，或自己建图。
- **AutoGen v0.4**（第 14 课）— 高并发场景下的 actor 模型替代。
- **Claude Agent SDK**（第 17 课）— 带内置会话存储的托管执行框架。
- **自定义** — 当你要精确控制状态形状或检查点器后端时。

## 交付

`outputs/skill-state-graph.md` 在任意目标运行时里生成一张 LangGraph 形状的状态图，检查点和恢复都接好。

## 练习

1. 当分类置信度低于阈值时，加一条从 `classify` 到 `end` 的条件边。人手工设置 `route` 之后，恢复这次运行。
2. 把类似 SQLite 的假实现换成真正的 SQLite 检查点器。测量每一步的序列化开销。
3. 实现并行边：两个节点并发运行，用自定义 reducer 合并。不可变状态在这里换来了什么？
4. 读 `langgraph-supervisor` 参考。把这个玩具移植到 `create_supervisor`。比较轨迹的形状。
5. 加上流式输出：每个节点在运行时产出部分状态。增量一到就打印。

## 关键术语

| 术语 | 人们怎么说 | 它实际的意思 |
|------|----------------|------------------------|
| 状态图 | 「智能体即状态机」 | 带类型的状态 + 节点 + 边 + reducer |
| 检查点器 | 「持久化后端」 | 每个节点之后序列化状态；使恢复成为可能 |
| Reducer | 「状态合并器」 | 把当前状态和节点更新组合起来的函数 |
| 条件边 | 「分支」 | 由状态的某个函数选出的边 |
| 子图 | 「嵌套图」 | 被当作另一个图里的节点来用的图 |
| 持久执行 | 「从故障恢复」 | 带着精确状态，从最后一次成功的节点重启 |
| 监督者 | 「路由 LLM」 | 专家子智能体的中心分派器 |
| Swarm | 「对等智能体」 | 智能体通过共享工具交接；没有中心路由器 |

## 延伸阅读

- [LangGraph 概览](https://docs.langchain.com/oss/python/langgraph/overview) — 参考文档
- [langgraph-supervisor 参考](https://reference.langchain.com/python/langgraph/supervisor/) — 监督者模式 API
- [AutoGen v0.4，Microsoft Research](https://www.microsoft.com/en-us/research/articles/autogen-v0-4-reimagining-the-foundation-of-agentic-ai-for-scale-extensibility-and-robustness/) — actor 模型替代
- [Claude Agent SDK 概览](https://platform.claude.com/docs/en/agent-sdk/overview) — 会话存储与子智能体
