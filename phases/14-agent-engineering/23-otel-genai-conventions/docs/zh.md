# OpenTelemetry GenAI 语义约定

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> OpenTelemetry 的 GenAI SIG（2024 年 4 月成立）定义了智能体遥测的标准 schema。span 名称、属性和内容捕获规则在各厂商之间收敛，使智能体追踪在 Datadog、Grafana、Jaeger 和 Honeycomb 里表示同一件事。

**Type:** 理解 + 动手做
**Languages:** Python（标准库）
**Prerequisites:** Phase 14 · 13（LangGraph）、Phase 14 · 24（可观测性平台）
**Time:** ~60 分钟

## 学习目标

- 说出 GenAI span 类别：model/client、agent、tool。
- 区分 `invoke_agent` 的 CLIENT 与 INTERNAL span，以及各自何时适用。
- 列出顶层 GenAI 属性：提供方名称、请求模型、数据源 ID。
- 解释内容捕获契约：默认关闭、`OTEL_SEMCONV_STABILITY_OPT_IN`、外部引用建议。

## 问题

每个厂商都发明自己的 span 名称。运维团队最终得为每个框架单独做仪表板。OpenTelemetry 的 GenAI SIG 通过定义一套整个生态都对准的标准来解决这个问题。

## 概念

### span 类别

1. **模型 / 客户端 span。** 覆盖原始 LLM 调用。由提供方 SDK（Anthropic、OpenAI、Bedrock）和框架的模型适配器发出。
2. **智能体 span。** `create_agent`（智能体被构造时）和 `invoke_agent`（智能体运行时）。
3. **工具 span。** 每次工具调用一个；通过父子关系连到智能体 span。

### 智能体 span 命名

- span 名称：若有名字则为 `invoke_agent {gen_ai.agent.name}`；否则回退为 `invoke_agent`。
- span 种类：
  - **CLIENT** — 用于远程智能体服务（OpenAI Assistants API、Bedrock Agents）。
  - **INTERNAL** — 用于进程内智能体框架（LangChain、CrewAI、本地 ReAct）。

### 关键属性

- `gen_ai.provider.name` — `anthropic`、`openai`、`aws.bedrock`、`google.vertex`。
- `gen_ai.request.model` — 模型 ID。
- `gen_ai.response.model` — 解析后的模型（因路由可能与请求不同）。
- `gen_ai.agent.name` — 智能体标识符。
- `gen_ai.operation.name` — `chat`、`completion`、`invoke_agent`、`tool_call`。
- `gen_ai.data_source.id` — 用于检索增强生成：查阅了哪个语料库或存储。

Anthropic、Azure AI Inference、AWS Bedrock、OpenAI 还有技术专用约定。

### 内容捕获

默认规则：插桩默认不应（SHOULD NOT）捕获输入/输出。捕获通过以下属性显式开启：

- `gen_ai.system_instructions`
- `gen_ai.input.messages`
- `gen_ai.output.messages`

推荐的生产模式：把内容存到外部（S3、你的日志存储），在 span 上记录引用（指针 ID，而不是正文）。这就是把第 27 课的内容投毒防御接进可观测性。

### 稳定性

截至 2026 年 3 月，大多数约定仍是实验性的。用下面的环境变量加入稳定预览：

```
OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental
```

Datadog v1.37+ 会把 GenAI 属性原生映射进它的 LLM Observability schema。其他后端（Grafana、Honeycomb、Jaeger）支持原始属性。

### 这个模式会在哪里出错

- **在 span 里捕获完整提示词。** 追踪里出现 PII、密钥、客户数据，运维可以读到。应存到外部。
- **没有 `gen_ai.provider.name`。** 归因缺失时，多提供方仪表板会坏掉。
- **span 没有父链接。** 工具 span 变成孤儿。始终传播上下文。
- **没有设置稳定性 opt-in。** 后端升级时你的属性可能被改名。

```figure
ae-genai-span-tree
```

## 动手做

`code/main.py` 实现一个符合 GenAI 约定的标准库 span 发射器：

- 带 GenAI 属性 schema 的 `Span`。
- 带 `start_span` 和嵌套上下文的 `Tracer`。
- 一次脚本化的智能体运行，发出：`create_agent`、`invoke_agent`（INTERNAL）、每个工具的 span，以及 LLM 调用的 `chat` span。
- 一种内容捕获模式：把提示词存到外部，在 span 上记录 ID。

运行：

```
python3 code/main.py
```

输出：一棵带齐所有必需 GenAI 属性的 span 树，以及一个展示 opt-in 内容引用的“外部存储”。

## 使用

- **Datadog LLM Observability**（v1.37+）原生映射这些属性。
- **Langfuse / Phoenix / Opik**（第 24 课）— 为生态做自动插桩。
- **Jaeger / Honeycomb / Grafana Tempo** — 原始 OTel 追踪；用 GenAI 属性做仪表板。
- **自托管** — 跑带 GenAI 处理器的 OTel Collector。

## 交付

`outputs/skill-otel-genai.md` 把 OTel GenAI span 接进现有智能体，并带上内容捕获默认值和外部引用存储。

## 练习

1. 用 `invoke_agent`（INTERNAL）和每个工具的 span 给你的第 01 课 ReAct 循环做插桩。发到一个 Jaeger 实例。
2. 以“仅引用”模式增加内容捕获：提示词进 SQLite，span 属性只带行 ID。
3. 阅读 `gen_ai.data_source.id` 的规范。把它接到你的第 09 课 Mem0 搜索上。
4. 设置 `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`，并验证收集器不会重命名你的属性。
5. 做一块仪表板：仅凭 GenAI 属性回答“哪些工具错误与哪些模型相关”。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| GenAI SIG | “OpenTelemetry GenAI 小组” | 定义该 schema 的 OTel 工作组 |
| invoke_agent | “智能体 span” | 表示一次智能体运行的 span 名称 |
| CLIENT span | “远程调用” | 调用远程智能体服务的 span |
| INTERNAL span | “进程内” | 进程内智能体运行的 span |
| gen_ai.provider.name | “提供方” | anthropic / openai / aws.bedrock / google.vertex |
| gen_ai.data_source.id | “检索增强生成来源” | 一次检索命中了哪个语料库/存储 |
| 内容捕获 | “提示词日志” | 对消息的 opt-in 捕获；生产中存到外部 |
| 稳定性 opt-in | “预览模式” | 用来钉住实验性约定的环境变量 |

## 延伸阅读

- [OpenTelemetry GenAI 语义约定](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — 规范
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) — 默认带 GenAI span
- [AutoGen v0.4（Microsoft Research）](https://www.microsoft.com/en-us/research/articles/autogen-v0-4-reimagining-the-foundation-of-agentic-ai-for-scale-extensibility-and-robustness/) — 内置 OTel span
- [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) — W3C 追踪上下文传播
