# 智能体可观测性：Langfuse、Phoenix、Opik

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 2026 年有三个开源智能体可观测性平台占主导。Langfuse（MIT）— 每月 600 万+ 次安装，追踪 + 提示词管理 + 评测 + 会话回放。Arize Phoenix（Elastic 2.0）— 深入的智能体专用评测、检索增强生成相关性、OpenInference 自动插桩。Comet Opik（Apache 2.0）— 自动化提示词优化、护栏、LLM 评判的幻觉检测。

**Type:** 理解
**Languages:** Python（标准库）
**Prerequisites:** Phase 14 · 23（OTel GenAI）
**Time:** ~45 分钟

## 学习目标

- 说出三个顶级开源智能体可观测性平台及其许可证。
- 区分各自最强的地方：Langfuse（提示词管理 + 会话）、Phoenix（检索增强生成 + 自动插桩）、Opik（优化 + 护栏）。
- 解释为什么到 2026 年有 89% 的组织报告已经具备智能体可观测性。
- 实现一条带 LLM 评判评测的标准库追踪到仪表板流水线。

## 问题

OTel GenAI（第 23 课）给你 schema。你仍然需要一个平台来摄入 span、运行评测、存储提示词版本，并暴露回退。三个竞争者各自强调生命周期的不同部分。

## 概念

### Langfuse（MIT）

- 每月 600 万+ 次 SDK 安装，1.9 万+ GitHub star。
- 功能：追踪、带版本管理和 playground 的提示词管理、评测（LLM 作评判、用户反馈、自定义）、会话回放。
- 2025 年 6 月：原先的商业模块（LLM 作评判、标注队列、提示词实验、Playground）以 MIT 开源。
- 最强项：端到端可观测性，并带紧的提示词管理闭环。

### Arize Phoenix（Elastic License 2.0）

- 更深入的智能体专用评测：追踪聚类、异常检测、检索增强生成的检索相关性。
- 原生 OpenInference 自动插桩。
- 与托管的 Arize AX 搭配用于生产。
- 没有提示词版本管理 — 定位是与更广平台并列的漂移/行为回退工具。
- 最强项：检索增强生成相关性、行为漂移、异常检测。

### Comet Opik（Apache 2.0）

- 通过 A/B 实验做自动化提示词优化。
- 护栏（PII 脱敏、主题约束）。
- LLM 评判的幻觉检测。
- 来自 Comet 自身测量的基准：Opik 的日志 + 评测 23.44 秒，Langfuse 327.15 秒（约 14 倍差距）— 把厂商基准当作方向性参考。
- 最强项：优化闭环、自动化实验、护栏执行。

### 行业数据

据 Maxim（2026 实地分析）：89% 的组织已经具备智能体可观测性；质量问题是生产中的首要障碍（32% 的受访者提到了它）。

### 如何选择

| 需求 | 选择 |
|------|------|
| 带提示词管理的一体化 | Langfuse |
| 深入的检索增强生成评测 + 漂移 | Phoenix |
| 自动化优化 + 护栏 | Opik |
| 开放许可、不要 ELv2 | Langfuse（MIT）或 Opik（Apache 2.0） |
| Datadog / New Relic 集成 | 任意 — 它们都导出 OTel |

### 这个模式会在哪里出错

- **没有评测策略。** 没有评测的追踪只是昂贵的日志。
- **自研 LLM 评判却没有依据。** CRITIC 模式（第 05 课）适用 — 评判需要外部工具来做事实核验。
- **提示词版本没有绑到追踪上。** 生产回退时，你无法二分到导致问题的那条提示词。

```figure
wb-trace-ingest
```

## 动手做

`code/main.py` 实现一个标准库追踪收集器 + LLM 评判器：

- 摄入 GenAI 形状的 span。
- 按会话分组，给失败运行打标签（护栏触发、低置信评测）。
- 一个脚本化的 LLM 评判器，按量规给智能体回复打分。
- 一份类似仪表板的摘要：失败率、首要失败原因、评测分数分布。

运行：

```
python3 code/main.py
```

输出：按会话的评测分数和失败分类，与 Langfuse/Phoenix/Opik 会展示的内容一致。

## 使用

- **Langfuse** 自托管或云；通过 OTel 或它们的 SDK 接入。
- **Arize Phoenix** 自托管；用 OpenInference 自动插桩。
- **Comet Opik** 自托管或云；自动化优化闭环。
- **Datadog LLM Observability** 适合已经在跑 Datadog 的运维 + 机器学习混合团队。

## 交付

`outputs/skill-obs-platform-wiring.md` 选定一个平台，并把追踪 + 评测 + 提示词版本接进现有智能体。

## 练习

1. 把一周的 OTel 追踪导出到 Langfuse 云（免费层）。哪些会话失败了？为什么？
2. 为你的领域写一份 LLM 评判量规（事实正确性、语气、范围遵守）。在 50 条追踪上测试。
3. 比较 Langfuse 的提示词版本管理与 Phoenix 的追踪聚类。哪一个更快告诉你坏在哪里？
4. 阅读 Opik 的护栏文档。把一个 PII 脱敏护栏接到你的某次智能体运行上。
5. 在你自己的语料上给三者做基准。忽略厂商公布的数字；测量你自己的。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| 追踪 | “span 收集器” | 摄入 OTel / SDK span；按会话索引 |
| 提示词管理 | “提示词 CMS” | 绑到追踪上的版本化提示词 |
| LLM 作评判 | “自动化评测” | 另一个 LLM 按量规给智能体输出打分 |
| 会话回放 | “追踪回放” | 逐步走过过去的运行以便调试 |
| 检索增强生成相关性 | “检索质量” | 检索到的上下文是否匹配查询 |
| 追踪聚类 | “行为分组” | 把相似运行聚成簇以检测漂移 |
| 护栏执行 | “记录时的策略” | 对已记录内容做 PII/毒性/范围检查 |

## 延伸阅读

- [Langfuse 文档](https://langfuse.com/) — 追踪、评测、提示词管理
- [Arize Phoenix 文档](https://docs.arize.com/phoenix) — 自动插桩、漂移
- [Comet Opik](https://www.comet.com/site/products/opik/) — 优化 + 护栏
- [OpenTelemetry GenAI 语义约定](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — 三者都消费的 schema
