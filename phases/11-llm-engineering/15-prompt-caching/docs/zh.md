# 提示词缓存与上下文缓存

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 你的系统提示词有 4,000 个 token。你的检索增强生成（RAG）上下文有 20,000 个 token。你每次请求都把两者一起发送。你也每次都为两者付费。提示词缓存让提供商在他们那边把这个前缀保持温热，并在复用时按正常费率的 10% 计费。用对了，它能把推理成本削减 50–90%，把首 token 延迟削减 40–85%。

**Type:** 动手做
**Languages:** Python
**Prerequisites:** Phase 11 · 01（提示词工程）, Phase 11 · 05（上下文工程）, Phase 11 · 11（缓存与成本）
**Time:** ~60 分钟

## 问题

一个编程智能体在对话的每一轮都把同一份 15,000 token 的系统提示词发给 Claude。按每百万输入 token $3 计算，二十轮光输入成本就是 $0.90——还没算用户的任何实际消息。乘以每天 10,000 段对话，账单会达到每天 $9,000，而那些文本从未改变。

你不能在不伤害质量的情况下缩小提示词。你也不能避免发送它——模型每一轮都需要它。唯一的办法是不再为提供商已经见过的前缀支付全价。

这个办法就是提示词缓存。Anthropic 在 2024 年 8 月发布了它（2025 年又有了 1 小时的延长 TTL（存活时间）变体），OpenAI 在同年晚些时候把它自动化了，Google 在 Gemini 1.5 旁边发布了显式上下文缓存，现在三家都在其前沿模型上把它作为一等特性提供。

## 概念

![提示词缓存：写一次，读起来便宜](../assets/prompt-caching.svg)

**机制。** 当一次请求的前缀与最近一次请求的前缀匹配时，提供商直接提供上一次运行的 KV-cache（键值缓存），而不是重新编码这些 token。你第一次支付一小笔写入溢价，之后每一次都享受很大的读取折扣。

**2026 年的三种提供商形态。**

| 提供商 | API 风格 | 命中折扣 | 写入溢价 | 默认 TTL | 最小可缓存长度 |
|---------|-----------|--------------|---------------|-------------|---------------|
| Anthropic | 内容块上的显式 `cache_control` 标记 | 输入费用减免 90% | 25% 附加费 | 5 分钟（可延长到 1 小时） | 1,024 token（Sonnet/Opus），2,048（Haiku） |
| OpenAI | 自动前缀检测 | 输入费用减免 50% | 无 | 最长 1 小时（尽力而为） | 1,024 token |
| Google (Gemini) | 显式 `CachedContent` API | 按存储计费；读取约为正常价的 ~25% | 每 token·小时的存储费 | 用户设定（默认 1 小时） | 4,096 token（Flash），32,768（Pro） |

**不变量。** 三家都只缓存前缀。如果请求之间有任何一个 token 不同，第一个不同 token 之后的一切都是未命中。把*稳定*的部分放在顶部，把*可变*的部分放在底部。

### 对缓存友好的布局

```
[system prompt]          <-- cache this
[tool definitions]       <-- cache this
[few-shot examples]      <-- cache this
[retrieved documents]    <-- cache if reused, else don't
[conversation history]   <-- cache up to last turn
[current user message]   <-- never cache (different every time)
```

打乱顺序——把用户消息放在系统提示词上面，或在 few-shot 示例之间穿插动态检索——缓存就永远不会命中。

### 盈亏平衡计算

Anthropic 25% 的写入溢价意味着，一个被缓存的块至少要被读取两次才能净省钱。1 次写入 + 1 次读取，平均每次请求成本为 0.675 倍（省 32%）；1 次写入 + 10 次读取，平均为 0.205 倍（省 80%）。经验法则：缓存任何你预期在 TTL 内至少复用 3 次的内容。

```figure
prompt-cache-hit
```

## 动手做

### 第 1 步：带显式标记的 Anthropic 提示词缓存

```python
import anthropic

client = anthropic.Anthropic()

SYSTEM = [
    {
        "type": "text",
        "text": "You are a senior Python reviewer. Follow the rubric exactly.\n\n" + RUBRIC_15K_TOKENS,
        "cache_control": {"type": "ephemeral"},
    }
]

def review(code: str):
    return client.messages.create(
        model="claude-opus-4-7",
        max_tokens=1024,
        system=SYSTEM,
        messages=[{"role": "user", "content": code}],
    )
```

`cache_control` 标记告诉 Anthropic 把该块存储 5 分钟。在该窗口内复用即命中；过期后再复用会重新写入。

**响应的 usage 字段：**

```python
response = review(code_a)
response.usage
# InputTokensUsage(
#     input_tokens=120,
#     cache_creation_input_tokens=15023,   # paid at 1.25x
#     cache_read_input_tokens=0,
#     output_tokens=340,
# )

response_b = review(code_b)
response_b.usage
# cache_creation_input_tokens=0
# cache_read_input_tokens=15023           # paid at 0.1x
```

在 CI 里检查这两个字段——如果跨请求的 `cache_read_input_tokens` 一直为零，你的缓存键在漂移。

### 第 2 步：一小时延长 TTL

对长时间运行的批处理作业，5 分钟的默认值会在作业之间过期。设置 `ttl`：

```python
{"type": "text", "text": RUBRIC, "cache_control": {"type": "ephemeral", "ttl": "1h"}}
```

1 小时 TTL 的写入溢价是 2 倍（相对基线高出 50%，而不是 25%），但只要任何批次对前缀的复用超过 5 次，就能很快回本。

### 第 3 步：OpenAI 自动缓存

OpenAI 不给你任何可配置项。任何超过 1,024 token、且与最近一次请求匹配的前缀，都会自动获得 50% 折扣。

```python
from openai import OpenAI
client = OpenAI()

resp = client.chat.completions.create(
    model="gpt-5",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},   # long and stable
        {"role": "user", "content": user_msg},
    ],
)
resp.usage.prompt_tokens_details.cached_tokens  # the discounted portion
```

同样的对缓存友好的布局规则适用。有两件事会杀掉 OpenAI 的缓存、却不会杀掉 Anthropic 的：更改 `user` 字段（它被用作缓存键的组成部分），以及重排工具。

### 第 4 步：Gemini 显式上下文缓存

Gemini 把缓存当作你可以创建并命名的一等对象：

```python
from google import genai
from google.genai import types

client = genai.Client()

cache = client.caches.create(
    model="gemini-3.8-flash",
    config=types.CreateCachedContentConfig(
        display_name="rubric-v3",
        system_instruction=RUBRIC,
        contents=[FEW_SHOT_EXAMPLES],
        ttl="3600s",
    ),
)

resp = client.models.generate_content(
    model="gemini-3.8-flash",
    contents=["Review this code:\n" + code],
    config=types.GenerateContentConfig(cached_content=cache.name),
)
```

只要缓存还活着，Gemini 就按每 token·小时收取存储费，读取约为正常输入费率的 ~25%。当你在数天内的许多会话中复用同一份巨大提示词时，这是合适的形态。

### 第 5 步：在生产中测量命中率

参见 `code/main.py`，其中有一个模拟的三提供商记账器，追踪写入/读取/未命中次数，并计算每 1K 次请求的混合成本。用目标命中率把关部署——大多数生产中的 Anthropic 设置在预热后读取占比应看到 >80%。

## 2026 年仍会带进生产的陷阱

- **顶部的动态时间戳。** 系统提示词顶部写着 `"Current time: 2026-04-22 15:30:02"`。每次请求都未命中。把时间戳移到缓存断点之下。
- **工具重排。** 以稳定顺序序列化工具——部署之间一次字典重排就会打破每一次命中。
- **自由文本的近似重复。** “You are helpful.” 与 “You are a helpful assistant.”——差一个字节 = 完全未命中。
- **块太小。** Anthropic 强制 1,024 token 的下限（Haiku 为 2,048）。更小的块会静默地不缓存。
- **盲目的成本仪表盘。** 把“输入 token”拆成已缓存与未缓存。否则流量下降看起来像缓存赢了。

## 用起来

2026 年的缓存技术栈：

| 情形 | 选择 |
|-----------|------|
| 稳定的 10k+ 系统提示词、多轮的智能体 | 带 5 分钟 TTL 的 Anthropic `cache_control` |
| 复用前缀 30 分钟以上的批处理作业 | 带 `ttl: "1h"` 的 Anthropic |
| GPT-5 上的无服务器端点，没有自定义基础设施 | OpenAI 自动（只要让你的前缀稳定且足够长） |
| 对巨大代码/文档语料的多日复用 | Gemini 显式 `CachedContent` |
| 跨提供商回退 | 在各提供商之间保持可缓存前缀布局一致，这样任何命中都能工作 |

把它与语义缓存（Phase 11 · 11）组合，用于用户消息层：提示词缓存处理*token 完全相同*的复用，语义缓存处理*含义相同*的复用。

## 交付

保存 `outputs/skill-prompt-caching-planner.md`：

```markdown
---
name: prompt-caching-planner
description: Design a cache-friendly prompt layout and pick the right provider caching mode.
version: 1.0.0
phase: 11
lesson: 15
tags: [llm-engineering, caching, cost]
---

Given a prompt (system + tools + few-shot + retrieval + history + user) and a usage profile (requests per hour, TTL needed, provider), output:

1. Layout. Reordered sections with a single cache breakpoint marked; explain which sections are stable, which are volatile.
2. Provider mode. Anthropic cache_control, OpenAI automatic, or Gemini CachedContent. Justify from TTL and reuse pattern.
3. Break-even. Expected reads per write within TTL; net cost vs no-cache with math.
4. Verification plan. CI assertion that cache_read_input_tokens > 0 on the second identical request; dashboard split by cached vs uncached tokens.
5. Failure modes. List the three most likely reasons the cache will miss in this setup (dynamic timestamp, tool reorder, near-duplicate text) and how you will prevent each.

Refuse to ship a cache plan that places a dynamic field above the breakpoint. Refuse to enable 1h TTL without a reuse count that makes the 2x write premium pay back.
```

## 练习

1. **简单。** 拿一段对 Claude 的 10 轮对话，系统提示词为 5,000 token。先不带 `cache_control` 跑，再带上跑。分别报告输入 token 账单。
2. **中等。** 写一个测试工具：给定一个提示词模板和一份请求日志，计算各提供商的预期命中率和美元节省（Anthropic 5 分钟、Anthropic 1 小时、OpenAI 自动、Gemini 显式）。
3. **困难。** 构建一个布局优化器：给定一个提示词和一份标记了 `stable=True/False` 的字段列表，重写提示词，在不丢失信息的前提下把单个缓存断点放在对缓存最友好的最大位置。在真实的 Anthropic 端点上验证。

## 关键术语

| 术语 | 人们怎么说 | 它实际的意思 |
|------|-----------------|-----------------------|
| 提示词缓存（Prompt caching） | “让长提示词变便宜” | 为匹配的前缀复用提供商侧的 KV-cache；对重复的输入 token 给予 50–90% 的折扣。 |
| `cache_control` | “Anthropic 的标记” | 内容块属性，声明“到这里为止都可缓存”；`{"type": "ephemeral"}`。 |
| 缓存写入（Cache write） | “支付溢价” | 填充缓存的第一次请求；在 Anthropic 上按约 1.25 倍输入费率计费，在 OpenAI 上免费。 |
| 缓存读取（Cache read） | “折扣” | 匹配该前缀的后续请求；按 10%（Anthropic）、50%（OpenAI）、约 25%（Gemini）计费。 |
| TTL | “它能活多久” | 缓存保持温热的秒数；Anthropic 默认 5 分钟（可延长到 1 小时），OpenAI 尽力而为最长 1 小时，Gemini 由用户设定。 |
| 延长 TTL（Extended TTL） | “1 小时的 Anthropic 缓存” | `{"type": "ephemeral", "ttl": "1h"}`；2 倍写入溢价，但对批处理复用值得。 |
| 前缀匹配（Prefix match） | “我的缓存为什么未命中” | 只有从开头到断点的每一个 token 都字节级相同，缓存才会命中。 |
| 上下文缓存（Context caching，Gemini） | “显式的那一种” | Google 的具名、按存储计费的缓存对象；最适合对大型语料的多日复用。 |

## 延伸阅读

- [Anthropic — Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) — `cache_control`、1 小时 TTL、盈亏平衡表。
- [OpenAI — Prompt caching](https://platform.openai.com/docs/guides/prompt-caching) — 自动前缀匹配。
- [Google — Context caching](https://ai.google.dev/gemini-api/docs/caching) — `CachedContent` API 与存储定价。
- [Anthropic engineering — Prompt caching for long-context workloads](https://www.anthropic.com/news/prompt-caching) — 带延迟数字的原始发布帖。
- Phase 11 · 05（上下文工程）— 在哪里切开提示词，缓存才能落得下。
- Phase 11 · 11（缓存与成本）— 把提示词缓存与用户消息上的语义缓存配对。
- [Pope et al., "Efficiently Scaling Transformer Inference" (2022)](https://arxiv.org/abs/2211.05102) — 提示词缓存向用户暴露的 KV-cache 内存模型；解释了为什么重读一个已缓存前缀比重新计算大约便宜 10 倍。
- [Agrawal et al., "SARATHI: Efficient LLM Inference by Piggybacking Decodes with Chunked Prefills" (2023)](https://arxiv.org/abs/2308.16369) — prefill 是提示词缓存所短路的阶段；这篇论文解释了为什么缓存命中时 TTFT 大幅下降，而 TPOT 不受影响。
- [Leviathan et al., "Fast Inference from Transformers via Speculative Decoding" (2023)](https://arxiv.org/abs/2211.17192) — 提示词缓存与投机解码、Flash Attention 以及 MQA/GQA 并列，都是弯曲推理成本曲线的杠杆；读这篇是为了另外三个。
