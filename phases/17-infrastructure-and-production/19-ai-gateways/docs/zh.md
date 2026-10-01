# AI 网关 — LiteLLM、Portkey、Kong AI Gateway、Bifrost

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 网关坐在你的应用和模型提供方之间。核心功能是提供方路由、回退、重试、速率限制、密钥引用、可观测性、护栏。2026 年的市场分化：**LiteLLM** 是 MIT 开源，100+ 提供方，兼容 OpenAI，但在大约 2000 RPS 附近垮掉（8 GB 内存，已发表基准中的级联失败）；最适合 Python、<500 RPS、开发/原型。**Portkey** 定位为控制面（护栏、PII 脱敏、越狱检测、审计轨迹），2026 年 3 月以 Apache 2.0 开源，20–40 ms 延迟开销，生产层 $49/月。**Kong AI Gateway** 建在 Kong Gateway 之上 — Kong 自己在同样 12 个 CPU 上的基准：比 Portkey 快 228%，比 LiteLLM 快 859%；定价 $100/模型/月（Plus 层最多 5 个）；如果你已经在用 Kong，就适合企业。**Bifrost**（Maxim AI）— 带可配置退避的自动重试，OpenAI 429 时回退到 Anthropic。**Cloudflare / Vercel AI Gateways** — 托管、零运维、基本重试。数据驻留驱动自托管决策；Portkey 和 Kong 以开源 + 可选托管居中。

**Type:** 理解
**Languages:** Python（标准库，玩具网关路由模拟器）
**Prerequisites:** Phase 17 · 01（托管 LLM 平台）、Phase 17 · 16（模型路由）
**Time:** ~60 分钟

## 学习目标

- 枚举六项核心网关功能（路由、回退、重试、速率限制、密钥、可观测性、护栏）。
- 把四个 2026 网关（LiteLLM、Portkey、Kong AI、Bifrost）映射到规模上限和用例。
- 引用 Kong 基准（相对 Portkey 228%，相对 LiteLLM 859%），并解释为什么它对 >500 RPS 重要。
- 在给定数据驻留和运维预算时，选择自托管还是托管。

## 问题

你的产品调用 OpenAI、Anthropic，以及一个自托管的 Llama。每个提供方有不同的 SDK、错误模型、速率限制和认证方案。你想要故障转移（如果 OpenAI 返回 429，就试 Anthropic）、一个单一的凭据存储、统一的可观测性，以及按租户的速率限制。

在应用层重造这些，会把每个服务耦合到每个提供方。网关层把它收进一个进程、一个 API（通常兼容 OpenAI），再扇出到各个提供方。

## 概念

### 六项核心功能

1. **提供方路由** — OpenAI、Anthropic、Gemini、自托管等，藏在一个 API 后面。
2. **回退** — 遇到 429、5xx 或质量失败时，到别处重试。
3. **重试** — 指数退避，有界尝试次数。
4. **速率限制** — 按租户、按密钥、按模型。
5. **密钥引用** — 运行时从保险库拉取凭据（绝不放在应用里）。
6. **可观测性** — OTel + GenAI 属性（Phase 17 · 13）+ 成本归因。
7. **护栏** — PII 脱敏、越狱检测、允许主题过滤器。

### LiteLLM — MIT 开源，Python

- 100+ 提供方，兼容 OpenAI，router 配置，回退，基本可观测性。
- 在 Kong 的基准里大约 2000 RPS 附近垮掉；8 GB 内存占用，持续负载下级联失败。
- 最适合：Python 应用、<500 RPS、开发/预发网关、实验性路由。
- 成本：开源 $0；存在云免费层。

### Portkey — 控制面定位

- 截至 2026 年 3 月为 Apache 2.0 开源。护栏、PII 脱敏、越狱检测、审计轨迹。
- 每请求 20–40 ms 延迟开销。
- 生产层 $49/月，带留存 + SLA。
- 最适合：需要把护栏 + 可观测性捆在一起的受监管行业。

### Kong AI Gateway — 规模打法

- 建在 Kong Gateway 上（成熟的 API 网关产品，lua+OpenResty）。
- Kong 自己在等价于 12 CPU 上的基准：比 Portkey 快 228%，比 LiteLLM 快 859%。
- 定价：$100/模型/月，Plus 层最多 5 个。
- 最适合：已经在用 Kong；>1000 RPS；愿意购买许可。

### Bifrost（Maxim AI）

- 带可配置退避的自动重试。
- OpenAI 429 时回退到 Anthropic 是一条典型配方。
- 较新的进入者；商业产品。

### Cloudflare AI Gateway / Vercel AI Gateway

- 托管，零运维。基本的重试和可观测性。
- 最适合：跑在 Cloudflare/Vercel 上的边缘 JavaScript 应用。
- 在护栏和速率限制上比 Kong/Portkey 有限。

### 自托管与托管

数据驻留是强制函数。医疗和金融默认自托管（LiteLLM，或 Portkey 开源，或 Kong）。消费产品默认托管（Cloudflare AI Gateway）或中间层（Portkey 托管）。混合：受监管租户自托管，其他租户托管。

### 延迟预算

- LiteLLM：典型开销 5–15 ms。
- Portkey：开销 20–40 ms。
- Kong：开销 3–8 ms。
- Cloudflare/Vercel：开销 1–3 ms（边缘优势）。

网关延迟直接加到 TTFT（首 token 时间）上。对于 TTFT P99 < 100 ms 的 SLA，选 Kong 或 Cloudflare。对于 P99 < 500 ms，任意都可以。

### 速率限制的语义很重要

简单令牌桶在中等规模以下够用。多租户需要滑动窗口 + 突发余量 + 按租户分层。LiteLLM 带令牌桶；Kong 带滑动窗口；Portkey 带分层。

### 网关 + 可观测性 + 路由是组合在一起的

Phase 17 · 13（可观测性）+ 16（模型路由）+ 19（网关）在生产里是同一层。选一个覆盖全部三项的工具，或者小心地把它们接起来：大多数 2026 部署把 Helicone（可观测性）或 Portkey（护栏）与 Kong（规模）组合，角色分开。

### 你应该记住的数字

- LiteLLM：大约 2000 RPS 垮掉，8 GB 内存。
- Portkey：20–40 ms 开销；2026 年 3 月起 Apache 2.0。
- Kong：比 Portkey 快 228%，比 LiteLLM 快 859%。
- Kong 定价：$100/模型/月，Plus 层最多 5 个。
- Cloudflare/Vercel：边缘上 1–3 ms 开销。

```figure
mx-gateway-fallback
```

## 使用

`code/main.py` 模拟跨 3 个提供方、在注入 429/5xx 下带回退的网关路由。报告延迟、重试率和回退命中率。

## 交付

本课产出 `outputs/skill-gateway-picker.md`。给定规模、运维姿态、合规、延迟预算，选出一个网关。

## 练习

1. 运行 `code/main.py`。配置从 OpenAI→Anthropic→自托管的回退。在 5% 提供方错误率下，期望命中率是多少？
2. 你的 SLA 是在 300 ms 基线上 TTFT P99 < 200 ms。哪些网关仍在预算内？
3. 一个医疗客户要求自托管 + PII 脱敏 + 审计。在 Portkey 开源和 Kong 之间选择。
4. 比较 LiteLLM 与 Kong：团队应该在什么 RPS 上限迁移？
5. 为一个多租户 SaaS 设计速率限制策略：免费层、试用层、付费层。令牌桶还是滑动窗口？

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| 网关 | “API 代理” | 坐在应用和提供方之间的进程 |
| LiteLLM | “那个 MIT 的” | Python 开源，100+ 提供方，在 2K RPS 垮掉 |
| Portkey | “护栏网关” | 控制面 + 可观测性，Apache 2.0 |
| Kong AI Gateway | “那个规模的” | 建在 Kong Gateway 上，基准领先 |
| Bifrost | “Maxim 的网关” | 重试 + Anthropic 回退配方 |
| Cloudflare AI Gateway | “边缘托管” | 部署在边缘的托管网关，零运维 |
| PII 脱敏 | “数据清洗” | 发给模型之前用正则 + NER 做掩码 |
| 越狱检测 | “提示注入护栏” | 对用户输入的分类器 |
| 审计轨迹 | “受监管日志” | 每一次 LLM 调用的不可变记录 |
| 令牌桶 | “简单速率限制” | 基于补充的速率限制器 |
| 滑动窗口 | “精确速率限制” | 按时间窗口的速率限制器；公平性更好 |

## 延伸阅读

- [Kong AI Gateway 基准](https://konghq.com/blog/engineering/ai-gateway-benchmark-kong-ai-gateway-portkey-litellm)
- [TrueFoundry — AI Gateways 2026 Comparison](https://www.truefoundry.com/blog/a-definitive-guide-to-ai-gateways-in-2026-competitive-landscape-comparison)
- [Techsy — Top LLM Gateway Tools 2026](https://techsy.io/en/blog/best-llm-gateway-tools)
- [LiteLLM GitHub](https://github.com/BerriAI/litellm)
- [Portkey GitHub](https://github.com/Portkey-AI/gateway)
- [Kong AI Gateway 文档](https://docs.konghq.com/gateway/latest/ai-gateway/)
