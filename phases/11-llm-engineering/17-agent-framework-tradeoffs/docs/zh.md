# 智能体框架的取舍 — 图、角色与 Actor 编排

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 每个框架都在卖同一个演示（研究智能体做一份报告），也藏着同一个 bug（状态 schema 和编排层打架）。选择抽象与你的问题形状匹配的框架；其余的都是你要写两遍的胶水。

**Type:** 理解
**Languages:** Python
**Prerequisites:** Phase 11 · 09（函数调用）, Phase 11 · 16（LangGraph）
**Time:** ~45 分钟

## 问题

你有一个需要不止一次 LLM 调用的任务。也许是研究工作流（规划、搜索、总结、引用）。也许是代码审查流水线（解析 diff、批评、打补丁、验证）。也许是一个多轮助手，订机票、写邮件、提交报销。你选了一个框架。

三天后，你发现框架的抽象在泄漏。CrewAI 给你角色，但当 “researcher” 需要把一份结构化计划交给 “writer” 时，它跟你对着干。AutoGen 给你智能体之间的聊天，但没有一等的状态，于是你的检查点是一份对话日志的 pickle。LangGraph 给你一个状态图，但迫使你在知道智能体会做什么之前就命名每一次转移。Agno 给你单智能体抽象，当你试图扇出到三个并发 worker 时它会尖叫。

解决办法不是“选最好的框架”。而是让框架的核心抽象匹配你问题的形状。本课画出这张地图。

## 概念

![智能体框架矩阵：核心抽象与问题形状](../assets/framework-matrix.svg)

四个框架主导 2026 年的格局。它们的核心抽象并不相同。

| 框架 | 核心抽象 | 最适合 | 最不适合 |
|-----------|------------------|----------|-----------|
| **LangGraph** | `StateGraph` — 带类型的状态、节点、条件边、checkpointer。 | 具有显式状态和人在回路中断的工作流；需要时间旅行调试的生产智能体。 | 拓扑未知的、松散的、由角色驱动的头脑风暴。 |
| **CrewAI** | `Crew` — 角色（目标、背景故事）、任务、流程（顺序或层级）。 | 带短线性/层级计划的角色扮演或人格驱动工作流。 | 超出 crew 回合历史的任何有状态需求；复杂分支。 |
| **AutoGen** | `ConversableAgent` 对 — 两个或更多智能体轮流发言，直到退出条件。 | 多智能体*对话*（教师-学生、提议者-批评者、行动者-审阅者），思考从聊天中涌现。 | 已知 DAG 的确定性工作流；任何需要跨重启持久状态的东西。 |
| **Agno** | `Agent` — 单个 LLM + 工具 + 记忆，可组合成团队。 | 快速构建的单智能体和轻量团队；强多模态和内置存储驱动。 | 带自定义 reducer（归约器）的、深的、显式分支的图。 |

### “抽象”实际意味着什么

框架的核心抽象，是你在白板上推销架构时画的那个东西。

- **LangGraph** → 你画一张图。节点是步骤，边是转移，每一点上的状态对象都有类型。心智模型是状态机。
- **CrewAI** → 你画一张组织架构图。每个角色有职位描述，经理路由任务。心智模型是一小队专家。
- **AutoGen** → 你画一段 Slack 私信。两个智能体互相发消息；如果需要主持人，第三个加入。心智模型是聊天。
- **Agno** → 你画一个挂着工具的单框。把框并排放就是一个团队。心智模型是“自带电池的智能体”。

### 状态问题

状态是大多数框架选择在生产中崩溃的地方。

- **LangGraph。** 带类型的状态（`TypedDict` 或 Pydantic 模型）、按字段的 reducer、一等的 checkpointer（SQLite/Postgres/Redis）。恢复、中断和时间旅行是免费的。*（见 Phase 11 · 16。）*
- **CrewAI。** 状态通过 `context` 字段在任务之间以字符串流动，或通过 `output_pydantic` 结构化。开箱没有持久的 per-crew 存储；如果 crew 必须挺过一次重启，你得自己接上。
- **AutoGen。** 状态是聊天历史和任何用户定义的 `context`。对话记录会持久化；任意工作流状态不会，除非你写适配器。
- **Agno。** 内置存储驱动（SQLite、Postgres、Mongo、Redis、DynamoDB）通过 `storage=` 挂到一个 `Agent` 上——对话会话和用户记忆自动持久化。不是完整的图 checkpointer；是一个会话存储。

### 分支问题

每一个非平凡的智能体都会分支。谁来决定分支很重要。

- **LangGraph** — 你来决定，通过条件边。路由是一个带命名分支的 Python 函数。分支在编译后的图里是一等的；checkpointer 记录走了哪条分支。
- **CrewAI** — 层级模式下由经理决定；顺序模式下你在构建时决定。路由隐含在任务列表里；在经理的提示词之外没有一等的 “if”。
- **AutoGen** — 智能体通过聊天决定。分支从下一个谁发言中涌现。`GroupChatManager` 选择下一位发言者；你可以手写 `speaker_selection_method`，但默认是 LLM 驱动的。
- **Agno** — 智能体通过接下来调用哪个工具来决定。团队有协调者/路由器/协作者模式；超出这之外的分支是开发者的责任。

### 可观测性问题

- **LangGraph** — 通过 LangSmith 或任何 OTel exporter 使用 OpenTelemetry。每一次节点转移都是一个 trace span；检查点同时也是可重放的 trace。LangSmith 是第一方选项；Langfuse/Phoenix 也有适配器。
- **CrewAI** — 自 2025 年末起一等支持 OpenTelemetry；与 Langfuse、Phoenix、Opik、AgentOps 集成。
- **AutoGen** — 通过 `autogen-core` 集成 OpenTelemetry；AgentOps 和 Opik 有连接器。追踪粒度是每条智能体消息，而不是每个节点。
- **Agno** — 内置 `monitoring=True` 标志加上 OpenTelemetry exporter；与 Langfuse 的会话 trace 集成紧密。

### 成本与延迟

四个框架都会增加每次调用的开销（框架逻辑、校验、序列化）。开销大致递增顺序：Agno ≈ LangGraph < CrewAI ≈ AutoGen。差异主要由框架额外做了多少 LLM 路由主导。CrewAI 的层级经理花 token 决定下一个是谁；AutoGen 的 `GroupChatManager` 同样如此。LangGraph 只在你写 `llm.invoke` 的地方花 token。Agno 的单智能体路径很薄。

当每次运行的成本重要时，优先选择显式路由（LangGraph 的边、AutoGen 的 `speaker_selection_method`），而不是由 LLM 选择的路由。

### 互操作性

- **LangGraph** ↔ **LangChain** 的工具、检索器、LLM。一等的 MCP（模型上下文协议）适配器（工具以 MCP 服务器的形式导入）。
- **CrewAI** ↔ 工具继承自 `BaseTool`；LangChain 工具、LlamaIndex 工具和 MCP 工具都能适配进来。Crew 到 crew 的委托通过 `allow_delegation=True`。
- **AutoGen** → `FunctionTool` 包装任何 Python 可调用对象；有 MCP 适配器。与 AG2 生态紧耦合，用于智能体到智能体的模式。
- **Agno** → `@tool` 装饰器或 BaseTool 子类；MCP 适配器；工具可以在智能体和团队之间共享。

## 这项技能

> 你能用一句话解释，为什么某个框架适合某个智能体问题。

构建前检查清单：

1. **画出形状。** 这是一张图（带类型的状态、命名的转移）？一场角色扮演（专家交接工作）？一段聊天（智能体谈到完成为止）？一个带工具的单智能体？
2. **决定谁来分支。** 开发者决定的分支 → LangGraph。经理智能体决定 → CrewAI 层级。聊天中涌现 → AutoGen。由工具调用决定 → Agno。
3. **检查状态预算。** 你需要从检查点恢复吗？时间旅行？运行中途的人工中断？如果是，LangGraph 是默认选择；Agno 会话覆盖对话范围的状态。
4. **检查成本预算。** 由 LLM 选择的路由每一轮都额外花 token。如果智能体每天运行成千上万次，优先显式路由。
5. **为框架开销做预算。** 每个框架都是又一个依赖。如果任务是两次 LLM 调用加一个工具，写 30 行纯 Python；没有框架比任何框架都便宜。

在你能画出那张图、那张组织架构图、那段聊天或那个智能体框之前，拒绝伸手去拿框架。拒绝选择一个迫使你为了真正需要的东西去对抗它的状态模型的框架。

## 决策矩阵

| 问题形状 | 首选框架 | 为什么 |
|---------------|---------------------|-----|
| 带类型状态、人工审批、长时间运行的工作流 DAG | LangGraph | 一等状态、checkpointer、中断、时间旅行。 |
| 角色分明的研究 / 写作流水线 | CrewAI（顺序）或 LangGraph 子图 | 每任务一个角色在 CrewAI 里表达起来便宜；当分支变复杂时用 LangGraph 扩展。 |
| 提议者-批评者或教师-学生对话 | AutoGen | 双智能体聊天是它的原生形状。 |
| 带工具、会话、记忆的单智能体 | Agno | 最薄的搭建，内置存储和记忆。 |
| 带 reducer 的成千上万次并行扇出 | LangGraph + `Send` | 唯一具有一等并行分发 API 的。 |
| 快速原型，不承诺某个框架 | 纯 Python + 提供商 SDK | 没有框架就是最快的框架。 |

```figure
l5-framework-fit
```

## 练习

1. **简单。** 拿同一个任务——“研究 Anthropic 的总部，写一份 200 词的简报，引用来源”——分别用 LangGraph（四个节点：plan、search、write、cite）和 CrewAI（三个角色：researcher、writer、editor）实现。报告每次运行的 token 成本和代码行数。
2. **中等。** 用 AutoGen（researcher ↔ writer 聊天，editor 通过 `GroupChat` 加入）和 Agno（一个带 `search_tools` 和 `write_tools` 的单智能体，外加一个会话存储）构建同一任务。按 (a) 每次运行成本、(b) 崩溃后恢复的能力、(c) 在写入步骤前注入人工审批的能力，给四种实现排序。
3. **困难。** 构建一个决策树脚本 `pick_framework.py`，它接受一段简短的问题描述（JSON：`{has_typed_state, has_roles, has_dialogue, has_parallel_fanout, needs_resume}`）并返回带一句话理由的推荐。在你自己设计的六个案例上验证它。

## 关键术语

| 术语 | 人们怎么说 | 它实际的意思 |
|------|-----------------|-----------------------|
| 编排（Orchestration） | “智能体如何协调” | 决定下一个运行哪个节点/角色/智能体的那一层。 |
| 持久状态（Durable state） | “重启后恢复” | 在进程死亡后仍然存在、附着在检查点或会话存储上的状态。 |
| 由 LLM 选择的路由（LLM-selected routing） | “让模型决定” | 一个规划 LLM 每一轮挑选下一步；灵活，但每次决策都付 token。 |
| 显式路由（Explicit routing） | “开发者决定” | 一个 Python 函数或静态边挑选下一步；便宜且可审计。 |
| Crew | “一个 CrewAI 团队” | 角色 + 任务 + 流程（顺序或层级），绑定成一个可运行单元。 |
| GroupChat | “AutoGen 的多智能体聊天” | 由发言者选择器管理的 N 个智能体之间的对话。 |
| Team（Agno） | “多智能体的 Agno” | 在一组智能体上的路由 / 协调 / 协作模式。 |
| StateGraph | “LangGraph 的图” | 带类型状态、节点、条件边、checkpointer 的抽象。 |

## 延伸阅读

- [LangGraph documentation](https://langchain-ai.github.io/langgraph/) — StateGraph、checkpointer、中断、时间旅行。
- [CrewAI documentation](https://docs.crewai.com/) — Crew、Flow、Agent、Task、Process。
- [AutoGen documentation](https://microsoft.github.io/autogen/) — ConversableAgent、GroupChat、团队、工具。
- [Agno documentation](https://docs.agno.com/) — Agent、Team、Workflow、存储、记忆。
- [Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) — 与框架无关的模式库（提示词链、路由、并行化、编排者-工作者、评测者-优化者）。
- [Yao et al., "ReAct: Synergizing Reasoning and Acting" (ICLR 2023)](https://arxiv.org/abs/2210.03629) — 每个框架都在包装的那个循环。
- [Wu et al., "AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation" (2023)](https://arxiv.org/abs/2308.08155) — AutoGen 的设计论文。
- [Park et al., "Generative Agents: Interactive Simulacra of Human Behavior" (UIST 2023)](https://arxiv.org/abs/2304.03442) — CrewAI 风格的人格栈所建立于其上的角色扮演基础。
- Phase 11 · 16（LangGraph）— 本课用来对照的框架。
- Phase 11 · 19（Reflexion）— 一个能干净映射到 LangGraph、却别扭地映射到 CrewAI 的模式。
- Phase 11 · 22（生产可观测性）— 如何给你选定的任何框架装上仪器。
