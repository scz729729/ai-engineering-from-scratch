# 推理指标 — TTFT（首 token 时间）、TPOT、ITL、Goodput、P99

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 四个指标决定一次推理部署是否在工作。TTFT 是预填充加上排队加上网络。TPOT（等价于 ITL）是每个 token 的、受内存约束的解码成本。端到端延迟是 TTFT 加上 TPOT 乘以输出长度。吞吐量是整个机群汇总的每秒 token 数。但对产品真正要紧的是 goodput — 同时满足每一条 SLO 的请求所占的比例。高吞吐量配上低 goodput，意味着你在处理那些永远无法按时到达用户的 token。2026 年 Llama-3.1-8B-Instruct 在 TRT-LLM 上的参考数字：平均 TTFT 162 ms，平均 TPOT 7.33 ms，平均端到端 1,093 ms。始终报告 P50、P90、P99 — 绝不要只报均值。还要当心测量陷阱：GenAI-Perf 在计算 ITL 时排除 TTFT，LLMPerf 把它包括进去；两个工具对同一次运行的 TPOT 意见不一。

**Type:** 理解
**Languages:** Python（标准库，玩具百分位计算器与 goodput 报告器）
**Prerequisites:** Phase 17 · 04（服务引擎内部）
**Time:** ~60 分钟

## 学习目标

- 精确定义 TTFT、TPOT、ITL、E2E、吞吐量和 goodput，并说出每一个测量的是哪个组成部分。
- 解释为什么均值是 LLM 服务的错误统计量，以及如何阅读 P50/P90/P99。
- 构造一条 SLO 多约束（例如 TTFT<500 ms 且 TPOT<15 ms 且 E2E<2 s），并据此计算 goodput。
- 说出两个对同一次运行的 TPOT 意见不一的基准工具，并解释原因。

## 问题

“我们的吞吐量是每秒 15,000 token。”那又怎样？如果 40% 的请求端到端超过了 2 秒，用户就放弃了会话。单靠吞吐量并不能告诉你产品是否在工作。

推理有多条延迟轴，每一条以不同方式失败。预填充（prefill）受计算约束，随提示词长度扩展。解码（decode）受内存约束，随批大小扩展。排队延迟是运维问题。网络是物理距离问题。你需要为每一项用不同的指标，你需要百分位，你还需要一个单一综合量来说“用户是否得到了他们期望的东西” — 那就是 goodput。

## 概念

### TTFT — 首 token 时间

`TTFT = queue_time + network_request + prefill_time`

提示词长时，预填充占主导。在 H100 上以 FP8 跑 Llama-3.3-70B，一条 32k 提示词需要约 800 ms 的纯预填充。排队时间是负载下的调度器行为。网络请求是包括 TLS 在内的线路时间。TTFT 是用户在任何内容流回来之前看到的延迟。

### TPOT / ITL — token 间延迟

一个量有很多名字。`TPOT`（每个输出 token 的时间）、`ITL`（token 间延迟）、`decode latency per token` — 全都一样。它是第一个 token 之后，连续流式 token 之间的时间。

`TPOT = (decode_forward_time + scheduler_overhead) / tokens_produced`

在同一套带分块 prefill 的 Llama-3.3-70B H100 栈上，TPOT 均值约 7 ms。没有分块 prefill 时，邻近序列正在做长预填充，TPOT 可以尖峰到 50 ms。看 P99，不要看均值。

### E2E 延迟

`E2E = TTFT + TPOT * output_tokens + network_response`

对长输出（>500 token），E2E 由 TPOT 主导。对短输出配长提示词，E2E 由 TTFT 主导。报告以输出长度为条件的 E2E。

### 吞吐量

`throughput = total_output_tokens / elapsed_time`

汇总指标。它告诉你机群效率。它不告诉你单个请求的健康状况。

### Goodput — 你真正关心的指标

`goodput = fraction of requests meeting (TTFT <= a) AND (TPOT <= b) AND (E2E <= c)`

SLO 是一条多约束。一个请求只有在每一条约束都成立时才是“好的”。Goodput 是这个份额。60% goodput 下的高吞吐量是失败。99% goodput 下的较低吞吐量才是目标。

2026 年，goodput 是 MLPerf Inference v6.0 提交所用的指标，也是 AI 平台提供商内部 SLA 跟踪所用的指标。

### 为什么均值是错误的统计量

LLM 延迟分布是右偏的。一个解码批次里如果有一个长预填充邻居，可以送出 500 个 TPOT 约 7 ms 的 token，以及 20 个 TPOT 约 60 ms 的 token。平均 TPOT 是 9 ms。P99 TPOT 是 65 ms。用户会经常撞上 P99 — 这就是他们离开的原因。

始终报告三元组（P50、P90、P99）。对用户体验而言，你要优化的是 P99。

### 参考数字 — 2026 年 TRT-LLM 上的 Llama-3.1-8B-Instruct

- 平均 TTFT：162 ms
- 平均 TPOT：7.33 ms
- 平均 E2E：1,093 ms
- P99 TPOT：随分块 prefill 配置在 10–25 ms 之间变化。

这些是 NVIDIA 公布的参考点。它们会随模型大小（70B 会显示 3–5 倍）、硬件（H100 相对 B200 约 3 倍）和负载而变化。

### 测量陷阱

2026 年最常用的两个基准工具，对同一次运行的 TPOT 意见不一：

- **NVIDIA GenAI-Perf**：在 ITL 计算中排除 TTFT。ITL 从第 2 个 token 开始。
- **LLMPerf**：包括 TTFT。ITL 从第 1 个 token 开始。

对一次 TTFT 为 500 ms、100 个输出 token 的总解码时间为 700 ms 的请求，GenAI-Perf 报告 `ITL = 700/99 = 7.07 ms`，LLMPerf 报告 `ITL = 1200/100 = 12.00 ms`。工具选择会改变数字。

始终说明用的是哪个工具。始终公布定义。

### 构造 SLO

2026 年一个面向消费者的 70B 聊天模型的合理 SLO：

- TTFT P99 <= 800 ms。
- TPOT P99 <= 25 ms。
- 对 <300 token 的输出，E2E P99 <= 3 s。
- Goodput 目标 >= 99%。

企业 SLO 收紧 TTFT（200–400 ms），放宽 E2E。要点是把它们写下来，测量全部三项，并把 goodput 当作单一综合量来跟踪。

### 如何测量

- 跑真实流量或逼真的合成流量（LLMPerf，带 `--mean-input-tokens 800 --stddev-input-tokens 300 --mean-output-tokens 150`）。
- 基准运行的目标是峰值并发的 2 倍。
- 跑 30–50 次迭代，对合并后的样本取百分位。
- 公布时带上工具名、工具版本、模型、硬件、并发、提示词分布。

```figure
throughput-latency
```

## 使用

`code/main.py` 是一个玩具 goodput 计算器。生成一个合成延迟分布，施加一条 SLO，并计算 goodput。它也在同一条追踪上展示 GenAI-Perf 与 LLMPerf 的 TPOT 差异。

## 交付

本课产出 `outputs/skill-slo-goodput-gate.md`。给定工作负载和 SLO，它产出一份可用于 CI/CD 的基准配方，用 goodput 而不是吞吐量来把守部署。

## 练习

1. 运行 `code/main.py`。生成一个带 1% 尾部尖峰的分布。当你把 P99 TPOT 从 30 ms 收紧到 15 ms 时，goodput 如何变化？
2. 一家厂商报出“Llama 3.3 70B 在 H100 上 15,000 tok/s”。在相信它之前，说出要问的三个问题。
3. 为什么分块 prefill 保护的是 P99 TPOT，而不是平均 TPOT？
4. 为一个语音助手构造一条消费者 SLO（首 token 是被听到的，不是被读到的）。哪个指标对用户最可见？
5. 阅读 LLMPerf README 和 GenAI-Perf 文档。找出另外三个工具意见不一的指标。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| TTFT | “首 token 时间” | 排队 + 网络 + 预填充；长提示词时由预填充主导 |
| TPOT | “每个输出 token 的时间” | 第一个 token 之后、受内存约束的每 token 解码成本 |
| ITL | “token 间延迟” | 在大多数工具里与 TPOT 相同（并非全部 — 见 GenAI-Perf） |
| E2E | “端到端” | TTFT + TPOT * output_len；上面再加响应侧网络 |
| 吞吐量 | “tok/s” | 机群效率；没有延迟百分位就没有用 |
| Goodput | “SLO 达成率” | 同时满足每一条 SLO 约束的请求所占比例 |
| P99 | “尾部” | 百分之一最差延迟；用户体验指标 |
| SLO 多约束 | “联合条件” | 三条延迟界限的 AND；任何一条被违反，请求即失败 |
| GenAI-Perf 与 LLMPerf | “工具陷阱” | 工具对 ITL 是否包含 TTFT 意见不一 |

## 延伸阅读

- [NVIDIA NIM — LLM Benchmarking Metrics](https://docs.nvidia.com/nim/benchmarking/llm/latest/metrics.html) — TTFT、ITL、TPOT 的规范定义。
- [Anyscale — LLM Serving Benchmarking Metrics](https://docs.anyscale.com/llm/serving/benchmarking/metrics) — 另一种定义和测量配方。
- [BentoML — LLM Inference Metrics](https://bentoml.com/llm/inference-optimization/llm-inference-metrics) — 真实部署上的应用测量。
- [LLMPerf](https://github.com/ray-project/llmperf) — 基于 Ray 的开源基准。
- [GenAI-Perf](https://github.com/triton-inference-server/perf_analyzer/blob/main/genai-perf/README.md) — NVIDIA 的基准工具。
- [MLPerf Inference](https://mlcommons.org/benchmarks/inference-datacenter/) — 业界接受的、基于 goodput 的基准。
