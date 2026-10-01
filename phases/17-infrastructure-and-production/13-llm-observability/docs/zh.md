# LLM 可观测性栈选型

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 2026 年的可观测性市场分成两类。开发平台（LangSmith、Langfuse、Comet Opik）把监控与评测、提示词管理、会话回放捆在一起。网关/插桩工具（Helicone、SigNoz、OpenLLMetry、Phoenix）聚焦遥测。Langfuse 的核心是 MIT 许可，开源平衡很好（云免费层每月 5 万次事件）。Phoenix 在 Elastic License 2.0 下原生支持 OpenTelemetry — 擅长漂移/检索增强生成可视化，不是持久的生产后端。Arize AX 用零拷贝的 Iceberg/Parquet 集成，声称比单体可观测性便宜 100 倍。LangSmith 在 LangChain/LangGraph 上领先，$39/用户/月，仅 Enterprise 可自托管。Helicone 基于代理，15–30 分钟即可搭好，每月 10 万次请求免费，但对智能体追踪的深度较浅。常见生产模式：网关（Helicone/Portkey）+ 评测平台（Phoenix/TruLens），用 OpenTelemetry 粘在一起。

**Type:** 理解
**Languages:** Python（标准库，玩具追踪采样模拟器）
**Prerequisites:** Phase 17 · 08（推理指标）、Phase 14（智能体工程）
**Time:** ~60 分钟

## 学习目标

- 区分开发平台（捆绑：评测 + 提示词 + 会话）与网关/遥测工具（只有追踪 + 指标）。
- 把六个主要工具（Langfuse、LangSmith、Phoenix、Arize AX、Helicone、Opik）映射到它们的许可、定价和最适合的用例。
- 解释 OpenTelemetry 粘合模式：它让你把一个网关工具和一个独立的评测平台组合起来。
- 说出 2026 年的成本差异点（Arize AX 的零拷贝方案相对单体摄入），并陈述大约 100 倍的乘数。

## 问题

你交付了一个 LLM 功能。它能工作。你看不见提示词失败、工具循环、延迟回退、成本尖峰，或提示词缓存命中率。你搜索“LLM observability”，得到八个工具，全都声称以三种不同价位解决同一个问题。

它们解决的不是同一个问题。LangSmith 回答“这次 LangGraph 运行为什么失败？”Phoenix 回答“我的检索增强生成流水线在漂移吗？”Helicone 回答“哪个应用在烧 token？”Langfuse 回答“我能把整套东西自托管吗？”不同的工具，不同的受众。

选择涉及四条轴：技术栈（LangChain？原始 SDK？多厂商？）、许可容忍度（只要 MIT？Elastic 可以？商业也可以？）、预算（免费层？$100/月？$1000/月？），以及自托管（必须？有了更好？绝不？）。

## 概念

### 两类

**开发平台**把可观测性与评测、提示词管理、数据集版本管理、会话回放捆在一起。你跑实验，看哪条提示词有效，用数据集回归把新提示词对照旧的胜者。LangSmith、Langfuse、Comet Opik。

**网关/遥测工具**给推理调用做插桩 — 提示词、响应、token、延迟、模型、成本。Helicone、SigNoz、OpenLLMetry、Phoenix。极简。可以通过 OpenTelemetry 与一个独立的评测工具组合。

### Langfuse — 开源平衡

- 核心 Apache / MIT 许可；通过 Docker 自托管。
- 云免费层：每月 5 万次事件。付费：团队 $29/月。
- 评测、提示词管理、追踪、数据集。对开发平台的四项功能覆盖都还合理。
- 最适合：你想要 LangSmith 级别的功能，但必须自托管或留在开源许可上。

### Phoenix（Arize）— 遥测优先，原生 OpenTelemetry

- Elastic License 2.0；自托管很轻松。
- 擅长检索增强生成和漂移可视化。嵌入空间散点图是一等功能。
- 不是为持久生产后端设计的 — 主要是开发期可观测性。
- 最适合：检索增强生成流水线开发、漂移调试，并与一个独立网关搭配用于生产。

### Arize AX — 规模打法

- 商业产品。通过 Iceberg/Parquet 做零拷贝数据湖集成。
- 声称在规模上比单体可观测性（Datadog 级别）便宜约 100 倍。算法是：你把追踪存在自己 S3 上的 Parquet 里；Arize 直接读。
- 最适合：每天 >1000 万条追踪、已有数据湖、想要 LLM 专用仪表板又不想付 Datadog 的价钱。

### LangSmith — LangChain/LangGraph 优先

- 商业产品，$39/用户/月。仅 Enterprise 可自托管。
- 对 LangChain 和 LangGraph 技术栈是同类最佳。如果你不在这两者上，它就不那么有吸引力。
- 最适合：承诺使用 LangChain、愿意付费的团队。

### Helicone — 基于代理的最小可行

- 把 `OPENAI_API_BASE` 换成 Helicone 代理，15–30 分钟搭好。
- MIT 许可；每月 10 万次请求免费，付费 $20/月起。
- 包含故障转移、缓存、速率限制 — 也充当网关。
- 对智能体 / 多步追踪的深度较浅。
- 最适合：快速起步、单栈应用、需要网关 + 可观测性合而为一。

### Opik（Comet）— 开源开发平台

- Apache 2.0，完全开源。
- 功能集与 Langfuse 相似，带 Comet 的血统。
- 最适合：已经在用 Comet 的机器学习团队，想在同一块窗格里做 LLM 可观测性。

### SigNoz — OpenTelemetry 优先的完整 APM

- Apache 2.0。通过 OpenTelemetry 处理通用 APM 加上 LLM。
- 最适合：跨服务和 LLM 调用的统一可观测性。

### 粘合层：OpenTelemetry + GenAI 语义约定

OpenTelemetry 在 2025 年末发布了 GenAI 语义约定（`gen_ai.system`、`gen_ai.request.model`、`gen_ai.usage.input_tokens`）。消费 OTel 的工具可以互操作。正在出现的生产模式：

1. 每一次 LLM 调用都按 GenAI 约定发出 OTel。
2. 路由到网关（Helicone / Portkey）做日常。
3. 双写到评测平台（Phoenix / Langfuse）做回归。
4. 归档到数据湖（Iceberg），通过 Arize AX 或 DuckDB 做长期分析。

### 陷阱：在错误的层做插桩

在智能体框架内部做插桩（例如加上 LangSmith 追踪）会把你耦合到那个框架。在 HTTP/OpenAI-SDK 层做插桩（通过 OpenLLMetry 或你的网关）是可移植的。

### 采样 — 你没法全都留下

每天超过 100 万次请求时，全量追踪留存的成本会超过 LLM 调用本身。按规则采样：错误 100%，高成本 100%，成功 5%。聚合始终保留；原始数据留给长尾。

### 你应该记住的数字

- Langfuse 免费云：每月 5 万次事件。
- LangSmith：$39/用户/月。
- Helicone 免费：每月 10 万次请求。
- Arize AX 的声称：规模上比单体便宜约 100 倍。
- OpenTelemetry GenAI 约定：2025 年交付，2026 年广泛采用。

```figure
i4-otel-glue
```

## 使用

`code/main.py` 模拟一天 100 万条追踪，跨多种留存策略（100% 摄入、采样、采样 + 错误）。报告每种策略下的存储成本以及丢失了什么。

## 交付

本课产出 `outputs/skill-observability-stack.md`。给定技术栈、规模、预算、许可姿态，选出工具。

## 练习

1. 你的团队在 LangChain 上，想要开源自托管的可观测性。在 Langfuse 和 Opik 之间选择并给出理由。
2. 每天 500 万条追踪，Datadog 报价每月 $150K，计算 Arize AX 的盈亏平衡点。
3. 设计一套你的组织指南应当强制加在每一次 LLM 调用上的 OpenTelemetry GenAI 属性集。
4. 论证 Phoenix 单独是否足以用于生产。它在什么时候不够？
5. Helicone 有 20 ms 的代理开销。在 P99 TTFT（首 token 时间）为 300 ms 时，这可以接受吗？如果 SLA 是 100 ms 呢？

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| OpenLLMetry | “给 LLM 的 OTel” | 面向 LLM 的开源 OpenTelemetry 插桩 |
| GenAI 约定 | “OTel 属性” | LLM 调用的标准 OTel 属性名 |
| LangSmith | “LangChain 可观测性” | 与 LangChain 生态捆绑的商业平台 |
| Langfuse | “开源 LangSmith” | 功能集相似的 MIT 开源 |
| Phoenix | “Arize 开发工具” | 原生 OpenTelemetry 的开发/评测平台 |
| Arize AX | “规模可观测性” | 商业的零拷贝 Iceberg/Parquet 可观测性 |
| Helicone | “代理可观测性” | 收集 LLM 遥测并带网关功能的 HTTP 代理 |
| Opik | “Comet LLM” | 来自 Comet 的 Apache 2.0 开源开发平台 |
| 会话回放 | “追踪重跑” | 回放一次完整的智能体会话及其工具调用 |
| 评测 | “离线测试” | 在标注数据集上运行候选模型/提示词 |

## 延伸阅读

- [SigNoz — Top LLM Observability Tools 2026](https://signoz.io/comparisons/llm-observability-tools/)
- [Langfuse — Arize AX 替代方案分析](https://langfuse.com/faq/all/best-phoenix-arize-alternatives)
- [PremAI — Setting Up Langfuse, LangSmith, Helicone, Phoenix](https://blog.premai.io/llm-observability-setting-up-langfuse-langsmith-helicone-phoenix/)
- [OpenTelemetry GenAI 语义约定](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [Arize Phoenix 文档](https://docs.arize.com/phoenix)
- [Helicone 文档](https://docs.helicone.ai/)
