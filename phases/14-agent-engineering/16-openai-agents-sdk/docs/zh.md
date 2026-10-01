# OpenAI Agents SDK：交接、护栏、追踪

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> OpenAI Agents SDK 是建在 Responses API 之上的轻量多智能体框架。五个原语：Agent、Handoff、Guardrail、Session、Tracing。交接是名为 `transfer_to_<agent>` 的工具。护栏在输入或输出上触发。追踪默认开启。

**Type:** 理解 + 动手做
**Languages:** Python（标准库）
**Prerequisites:** Phase 14 · 01（智能体循环）、Phase 14 · 06（工具使用）
**Time:** ~75 分钟

## 学习目标

- 说出 OpenAI Agents SDK 的五个原语。
- 解释交接：为什么它们被建模成工具、模型看到的名字是什么形状，以及上下文如何转移。
- 区分输入护栏、输出护栏和工具护栏；解释 `run_in_parallel` 与阻塞模式。
- 用标准库实现一个带交接、护栏和 span 风格追踪的运行时。

## 问题

不能干净委托的智能体，最后会把所有东西都塞进一个提示词。没有护栏的智能体会把 PII、违反策略的输出发出去，或者永远循环。OpenAI 的 SDK 把那三个让多智能体变得可做的原语写成了规范。

## 概念

### 五个原语

1. **Agent。** LLM + 指令 + 工具 + 交接。
2. **Handoff。** 委托给另一个智能体。对模型来说，它是一个名为 `transfer_to_<agent_name>` 的工具。
3. **Guardrail。** 对输入（仅第一个智能体）、输出（仅最后一个智能体）或工具调用（按每个函数工具）做校验。
4. **Session。** 跨回合的自动对话历史。
5. **Tracing。** 针对 LLM 生成、工具调用、交接、护栏的内置 span。

### 交接即工具

模型在自己的工具列表里看到 `transfer_to_billing_agent`。调用它，就是通知运行时去做这三件事：

1. 复制对话上下文（或经由 `nest_handoff_history` beta 把它折叠起来）。
2. 用目标智能体自己的指令把它初始化好。
3. 由目标智能体把这次运行继续下去。

这就是监督者模式（第 13 课 / 第 28 课）的产品化。

### 护栏

三种：

- **输入护栏。** 跑在第一个智能体的输入上。在任何 LLM 调用之前，拒绝不安全或超出范围的请求。
- **输出护栏。** 跑在最后一个智能体的输出上。抓住 PII 泄漏、策略违反、格式错误的响应。
- **工具护栏。** 按每个函数工具来跑。校验参数、检查权限、审计执行。

模式：

- **并行**（默认）。护栏 LLM 和主 LLM 一起跑。尾延迟更低。一旦触发，主 LLM 已经做的工作会被丢掉（token 浪费掉）。
- **阻塞**（`run_in_parallel=False`）。护栏 LLM 先跑。一旦触发，主调用上不会浪费 token。

触发线会抛出 `InputGuardrailTripwireTriggered` / `OutputGuardrailTripwireTriggered`。

### 追踪

默认开启。每次 LLM 生成、工具调用、交接和护栏都会发出一个 span。`OPENAI_AGENTS_DISABLE_TRACING=1` 用来关掉它。`add_trace_processor(processor)` 把 span 扇出到你自己的后端，和 OpenAI 的并行。

### 会话

`Session` 把对话历史存在某个后端里（SQLite、Redis、自定义）。`Runner.run(agent, input, session=session)` 会自动加载并追加。

### 这个模式会在哪里出错

- **交接漂移。** 智能体 A 交给智能体 B，B 又交回给 A。加一个跳数计数器。
- **护栏绕过。** 工具护栏只在函数工具上触发；内置工具（文件读取器、网页抓取）需要单独的策略。
- **追踪过度。** span 里有敏感内容。和 OTel GenAI 内容捕获规则（第 23 课）配在一起 — 存到外部，用 ID 引用。

```figure
ae-agent-handoff
```

## 动手做

`code/main.py` 用标准库实现这一 SDK 形状：

- `Agent`、`FunctionTool`、`Handoff`（作为一个带转移语义的函数工具）。
- 带输入/输出/工具护栏、交接分派和跳数计数器的 `Runner`。
- 一个简单的 span 发射器，用来展示追踪的形状。
- 一个分诊智能体，按用户的查询交给账单或支持；护栏会在其中一个输入上触发。

运行：

```
python3 code/main.py
```

追踪显示两次成功的交接、一次输入护栏触发，以及一棵镜像真实 SDK 所发出内容的 span 树。

## 使用

- **OpenAI Agents SDK**，用于 OpenAI 优先的产品。
- **Claude Agent SDK**（第 17 课），用于 Claude 优先的产品。
- **LangGraph**（第 13 课），当你要显式状态和可持久恢复时。
- **自定义**，当你要精确控制时（语音、多提供商、联邦式部署）。

## 交付

`outputs/skill-agents-sdk-scaffold.md` 为一套 Agents SDK 应用搭脚手架：分诊智能体、交接、输入/输出/工具护栏、会话存储，以及一个追踪处理器。

## 练习

1. 加一个交接跳数计数器：转移超过 N 次就拒绝。把这个行为追出来。
2. 把 `nest_handoff_history` 做成一个选项 — 转移之前，把先前的消息折叠成一条摘要。
3. 写一个阻塞式输出护栏。比较会触发它的提示词和能通过的提示词，延迟差在哪。
4. 把 `add_trace_processor` 接到一个 JSON 日志器。它为每个 span 发出的是什么形状？
5. 读 SDK 文档。把你的标准库玩具移植到 `openai-agents-python`。你哪里建模错了？

## 关键术语

| 术语 | 人们怎么说 | 它实际的意思 |
|------|----------------|------------------------|
| Agent | 「LLM + 指令」 | SDK 里的 Agent 类型；拥有工具和交接 |
| Handoff | 「转移」 | 模型调用它，把工作委托给另一个智能体 |
| Guardrail | 「策略检查」 | 对输入 / 输出 / 工具调用的校验 |
| Tripwire | 「护栏触发」 | 护栏拒绝时抛出的异常 |
| Session | 「历史存储」 | 在多次运行之间持久化的对话记忆 |
| Tracing | 「span」 | 覆盖 LLM、工具、交接和护栏的内置可观测性 |
| 阻塞护栏 | 「顺序检查」 | 护栏先跑；触发时不浪费 token |
| 并行护栏 | 「并发检查」 | 护栏并行跑；延迟更低，触发时浪费 token |

## 延伸阅读

- [OpenAI Agents SDK 文档](https://openai.github.io/openai-agents-python/) — 原语、交接、护栏、追踪
- [Claude Agent SDK 概览](https://platform.claude.com/docs/en/agent-sdk/overview) — Claude 风味的对应物
- [Anthropic，构建有效的智能体](https://www.anthropic.com/research/building-effective-agents) — 究竟何时才该用交接
- [OpenTelemetry GenAI 语义约定](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — Agents SDK 的 span 所映射到的标准
