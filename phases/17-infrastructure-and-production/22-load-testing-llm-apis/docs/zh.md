# LLM API 负载测试 — 为什么 k6 和 Locust 会说谎

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 传统负载测试器不是为流式响应、可变的输出长度、token 级指标或 GPU 饱和设计的。两个陷阱会咬中大多数团队。GIL 陷阱：Locust 的 token 级测量在 Python GIL 之下做分词，高并发时和请求生成抢同一把锁；分词积压随后抬高所报告的 token 间延迟 — 瓶颈是你的客户端，不是服务器。提示词均一性陷阱：循环里的相同提示词只测试 token 分布上的一个点；真实流量长度可变，前缀匹配也更多样。LLMPerf 用 `--mean-input-tokens` + `--stddev-input-tokens` 修这一点。2026 年的工具映射：LLM 专用工具（GenAI-Perf、LLMPerf、LLM-Locust、guidellm）用来拿到 token 级准确度；**k6 v2026.1.0** + **k6 Operator 1.0 GA（2025 年 9 月）** — 感知流式，经由 TestRun/PrivateLoadZone CRD 做 Kubernetes 原生的分布式，最适合 CI/CD 门禁；Vegeta 用来做 Go 的恒定速率饱和；Locust 2.43.3 只有配上 LLM-Locust 扩展才适合流式。负载模式：稳态、爬坡、尖峰（自动扩缩测试）、浸泡（内存泄漏）。

**Type:** 动手做
**Languages:** Python（标准库，玩具级真实提示词生成器 + 延迟采集器）
**Prerequisites:** Phase 17 · 08（推理指标）、Phase 17 · 03（GPU 自动扩缩）
**Time:** ~75 分钟

## 学习目标

- 解释让通用负载测试器在 LLM API 上说谎的两种反模式（GIL 陷阱、提示词均一性陷阱）。
- 按给定目的挑工具：LLMPerf（基准运行）、k6 + 流式扩展（CI 门禁）、guidellm（大规模合成）、GenAI-Perf（NVIDIA 参考）。
- 设计四种负载模式（稳态、爬坡、尖峰、浸泡），并说出每种抓住的故障模式。
- 用输入 token 的均值 + 标准差构建真实的提示词分布，而不是固定长度。

## 问题

你用 k6 以 500 个并发用户测了 LLM 端点。它撑住了。你就发出去了。到了生产，只有 200 个真实用户，服务就垮了 — P99 TTFT 爆炸，GPU 打满。

发生了两件事。第一，k6 发了 500 条相同的提示词 — 你的请求合并和前缀缓存让局面看起来像在处理 500 个并发解码，实际上你只在处理一个。第二，k6 并不按眼睛所体验的方式去跟踪流式响应上的 token 间延迟；它看到的是一条 HTTP 连接，不是以变化的间隔到达的 500 个 token。

给 LLM 做负载测试是一门自己的学科。

## 概念

### GIL 陷阱（Locust）

Locust 用 Python，并在客户端于 GIL 之下做分词。高并发时，分词器排在请求生成后面。所报告的 token 间延迟把客户端的分词积压也算了进去。你以为服务器慢；慢的是测试工具。

修法：LLM-Locust 扩展把分词挪到单独的进程，或者用编译型语言的测试工具（k6，以及使用 tokenizers.rs 的 LLMPerf）。

### 提示词均一性陷阱

所有已知的负载测试器都让你配置一条提示词。10,000 次迭代的循环测试里，每次发出去的都是完全相同的那一条。服务器每次都看到同一个前缀 — 前缀缓存命中率接近 100%，吞吐量看起来好极了。

修法：从提示词分布里采样。LLMPerf 使用 `--mean-input-tokens 500 --stddev-input-tokens 150` — 长度多样，内容也多样。

### 四种负载模式

1. **稳态** — 恒定 RPS，持续 30–60 分钟。抓住的是：基线性能回退。
2. **爬坡** — 在 15 分钟内把 RPS 从 0 线性加到目标。抓住的是：容量拐点、预热异常。
3. **尖峰** — 突然升到 3–10 倍 RPS，持续 2 分钟再回到原状。抓住的是：自动扩缩延迟、队列饱和、冷启动的影响。
4. **浸泡** — 以稳态跑 4–8 小时。抓住的是：内存泄漏、连接池漂移、可观测性溢出。

### 2026 年的工具映射

**LLMPerf**（Anyscale）— 用 Python 写，但分词由 Rust 支撑。提示词按均值/标准差采样。感知流式。性能运行的最佳默认选择。

**NVIDIA GenAI-Perf** — NVIDIA 的参考。使用 Triton 客户端；指标覆盖全面。注意它的 ITL 不含 TTFT；LLMPerf 的包含 TTFT。两个工具对同一台服务器会得出不同的 TPOT。

**LLM-Locust**（TrueFoundry）— 修好 GIL 陷阱的 Locust 扩展。熟悉的 Locust DSL，外加流式指标。

**guidellm** — 大规模合成基准。

**k6 v2026.1.0** + **k6 Operator 1.0 GA（2025 年 9 月）**：
- k6 本身（Go，编译型，没有 GIL）加上了感知流式的指标。
- k6 Operator 用 TestRun / PrivateLoadZone CRD 做 Kubernetes 原生的分布式测试。
- 最适合 CI/CD 门禁和 SLA 测试。

**Vegeta** — Go 编写，比 k6 更简单。恒定速率的 HTTP 饱和。不感知 LLM，但适合网关 / 速率限制测试。

**Locust 2.43.3 原版** — 在 LLM 场景下有 GIL 陷阱。只有配上 LLM-Locust 扩展才行。

### CI 里的 SLA 门禁

在 PR 上这样跑 k6：

- 在基线 RPS 下各跑 30–50 次迭代。
- 门禁：P50/P95 TTFT、5xx < 5%、TPOT 低于阈值。
- 违规时让构建失败。

### 真实的提示词分布

从真实流量样本来建（如果你有），或者从已发布的分布来建（例如聊天用 ShareGPT 提示词，代码用 HumanEval）。把均值 + 标准差喂给 LLMPerf。不惜一切代价避开单提示词循环。

### 你该记住的数字

- k6 Operator 1.0 GA：2025 年 9 月。
- k6 v2026.1.0：感知流式的指标。
- 典型的 LLMPerf 运行：并发度 X 下 100–1000 个请求。
- 典型的 CI 门禁：每个 PR 30–50 次迭代。
- 四种模式：稳态、爬坡、尖峰、浸泡。

```figure
load-pattern-waves
```

## 使用

`code/main.py` 用真实的提示词分布模拟一次负载测试，测量有效 TPOT，并演示提示词均一性陷阱。

## 交付

本课产出 `outputs/skill-load-test-plan.md`。给定工作负载和 SLA，挑选工具并设计这四种负载模式。

## 练习

1. 运行 `code/main.py`。比较均一分布和真实分布 — 差距在哪？
2. 给 CI 门禁写 k6 脚本：100 并发下 TTFT P95 < 800 ms，运行 5 分钟。
3. 你的浸泡测试显示内存每小时涨 50 MB。说出三个原因，以及用来在它们之间判别的插桩。
4. 尖峰测试从 10 RPS 做到 100 RPS。如果 Karpenter + vLLM production-stack 已经就位（Phase 17 · 03 + 18），预期恢复时间是多少？
5. GenAI-Perf 报告 TPOT=6ms；LLMPerf 在同一台服务器上报告 TPOT=11ms。解释一下。

## 关键术语

| 术语 | 人们怎么说 | 它实际的意思 |
|------|----------------|------------------------|
| LLMPerf | 「那个 LLM 测试工具」 | Anyscale 的基准工具，感知流式 |
| GenAI-Perf | 「NVIDIA 的工具」 | NVIDIA 的参考测试工具 |
| LLM-Locust | 「给 LLM 用的 Locust」 | 修好 GIL 陷阱的 Locust 扩展 |
| guidellm | 「合成基准」 | 大规模合成工具 |
| k6 Operator | 「K8s 上的 k6」 | 基于 CRD 的分布式 k6 |
| GIL 陷阱 | 「Python 客户端开销」 | 分词积压抬高了所报告的延迟 |
| 提示词均一性陷阱 | 「单提示词的谎言」 | 同一条提示词循环会命中缓存，从而抬高吞吐量 |
| 稳态 | 「恒定负载」 | 持续 N 分钟的平坦 RPS |
| 爬坡 | 「线性往上」 | 在一段时间里从 0 升到目标 |
| 尖峰 | 「突发测试」 | 突然乘上一个倍数，然后再恢复 |
| 浸泡 | 「长时间测试」 | 以小时计，用来检测泄漏 |

## 延伸阅读

- [TianPan — LLM 应用的负载测试](https://tianpan.co/blog/2026-03-19-load-testing-llm-applications)
- [PremAI — 2026 年的 LLM 负载测试](https://blog.premai.io/load-testing-llms-tools-metrics-realistic-traffic-simulation-2026/)
- [NVIDIA NIM — LLM 推理基准测试简介](https://docs.nvidia.com/nim/large-language-models/1.0.0/benchmarking.html)
- [TrueFoundry — LLM-Locust](https://www.truefoundry.com/blog/llm-locust-a-tool-for-benchmarking-llm-performance)
- [LLMPerf](https://github.com/ray-project/llmperf)
- [k6 Operator](https://github.com/grafana/k6-operator)
