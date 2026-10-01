# LLM 评测 — RAGAS、DeepEval、G-Eval

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 精确匹配和 F1 会漏掉语义等价。人工审阅无法扩展。LLM-as-judge（以 LLM 为评判器）是生产中的答案——但要有足够的校准，才敢相信那个数字。

**Type:** 动手做
**Languages:** Python
**Prerequisites:** Phase 5 · 13（问答）, Phase 5 · 14（信息检索）
**Time:** ~75 分钟

## 问题

你的检索增强生成（RAG）系统回答：“June 29th, 2007.”
黄金参考是：“June 29, 2007.”
Exact Match 得分 0。F1 大约 75%。人类会打 100%。

现在乘以 10,000 个测试用例。再乘以对检索器、分块、提示词或模型的每一次改动。你需要一个评测器，它理解含义、能便宜地大规模运行、不会在回归上撒谎，并且能浮现正确的失败模式。

2026 年有三个框架主导这个问题。

- **RAGAS。** Retrieval-Augmented Generation ASsessment。四个 RAG 指标（忠实性、答案相关性、上下文精确率、上下文召回率），后端是 NLI（自然语言推理）+ LLM 评判器。有研究支撑，轻量。
- **DeepEval。** 面向 LLM 的 Pytest。G-Eval、任务完成度、幻觉、偏见指标。原生 CI/CD。
- **G-Eval。** 一种方法（也是一个 DeepEval 指标）：带思维链的 LLM-as-judge、自定义标准、0–1 分数。

三者都依赖 LLM-as-judge。本课为这种方法以及围绕它的信任层建立直觉。

## 概念

![四个评测维度，LLM-as-judge 架构](../assets/llm-evaluation.svg)

**LLM-as-judge。** 用一个按评分量表给输出打分的 LLM 替换静态指标。给定 `(query, context, answer)`，提示一个评判 LLM：“在忠实性上打 0–1 分。”返回分数。

为什么它有效：LLM 以人类成本的极小一部分近似人类判断。GPT-4o-mini 大约每个打分用例 $0.003，使得 1000 样本的回归评测运行不到 $5。

为什么它会静默失败：

1. **评判器偏差。** 评判器偏好更长的答案、来自它们自己模型家族的答案、以及匹配提示词风格的答案。
2. **JSON 解析失败。** 坏的 JSON → NaN 分数 → 被静默地排除出聚合。RAGAS 用户知道这种痛。用 try/except + 显式失败模式把关。
3. **随模型版本漂移。** 升级评判器会改变每一个指标。冻结评判器模型 + 版本。

**RAG 四指标。**

| 指标 | 问题 | 后端 |
|--------|----------|---------|
| 忠实性（Faithfulness） | 答案中的每一条主张都来自检索到的上下文吗？ | 基于 NLI 的蕴含 |
| 答案相关性（Answer relevance） | 答案回应了问题吗？ | 从答案生成假设问题；与真实问题比较 |
| 上下文精确率（Context precision） | 在检索到的块中，有多大比例是相关的？ | LLM 评判器 |
| 上下文召回率（Context recall） | 检索是否返回了所需的一切？ | 对照黄金答案的 LLM 评判器 |

**G-Eval。** 定义一条自定义标准：“答案是否引用了正确的来源？”框架自动展开成思维链评测步骤，然后打 0–1 分。适合 RAGAS 未覆盖的领域特定质量维度。

**校准。** 在你有了与人类标签的相关性之前，永远不要相信原始的评判器分数。跑 100 个人工标注的例子。画出评判器对人类。计算 Spearman rho。如果 rho < 0.7，你的评判器评分量表需要改进。

```figure
n5-judge-gauge
```

## 动手做

### 第 1 步：用 NLI 做忠实性（RAGAS 风格）

```python
from typing import Callable
from transformers import pipeline

nli = pipeline("text-classification",
               model="MoritzLaurer/DeBERTa-v3-large-mnli-fever-anli-ling-wanli",
               top_k=None)

# `llm` is any callable: prompt str -> generated str.
# Example: llm = lambda p: client.messages.create(model="claude-haiku-4-5", ...).content[0].text
LLM = Callable[[str], str]


def atomic_claims(answer: str, llm: LLM) -> list[str]:
    prompt = f"""Break this answer into simple factual claims (one per line):
{answer}
"""
    return llm(prompt).splitlines()


def faithfulness(answer: str, context: str, llm: LLM) -> float:
    claims = atomic_claims(answer, llm)
    if not claims:
        return 0.0
    supported = 0
    for claim in claims:
        result = nli({"text": context, "text_pair": claim})[0]
        entail = next((s for s in result if s["label"] == "entailment"), None)
        if entail and entail["score"] > 0.5:
            supported += 1
    return supported / len(claims)
```

把答案分解成原子主张。对每条主张相对检索到的上下文做 NLI 检查。忠实性 = 被支持的比例。

### 第 2 步：答案相关性

```python
import numpy as np
from sentence_transformers import SentenceTransformer

# encoder: any model implementing .encode(texts, normalize_embeddings=True) -> ndarray
# e.g., encoder = SentenceTransformer("BAAI/bge-small-en-v1.5")

def answer_relevance(question: str, answer: str, encoder, llm: LLM, n: int = 3) -> float:
    prompt = f"Write {n} questions this answer could be the answer to:\n{answer}"
    generated = [line for line in llm(prompt).splitlines() if line.strip()][:n]
    if not generated:
        return 0.0
    q_emb = np.asarray(encoder.encode([question], normalize_embeddings=True)[0])
    g_embs = np.asarray(encoder.encode(generated, normalize_embeddings=True))
    sims = [float(q_emb @ g_emb) for g_emb in g_embs]
    return sum(sims) / len(sims)
```

如果答案暗示的问题与所问的问题不同，相关性就会下降。

### 第 3 步：G-Eval 自定义指标

```python
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCaseParams, LLMTestCase

metric = GEval(
    name="Correctness",
    criteria="The answer should be factually accurate and match the expected output.",
    evaluation_steps=[
        "Read the expected output.",
        "Read the actual output.",
        "List factual claims in the actual output.",
        "For each claim, mark supported or unsupported by the expected output.",
        "Return score = fraction supported.",
    ],
    evaluation_params=[LLMTestCaseParams.INPUT, LLMTestCaseParams.ACTUAL_OUTPUT, LLMTestCaseParams.EXPECTED_OUTPUT],
)

test = LLMTestCase(input="When was the first iPhone released?",
                   actual_output="June 29th, 2007.",
                   expected_output="June 29, 2007.")
metric.measure(test)
print(metric.score, metric.reason)
```

评测步骤就是评分量表。显式步骤比隐含的“打 0–1 分”提示词更稳定。

### 第 4 步：CI 门禁

```python
import deepeval
from deepeval.metrics import FaithfulnessMetric, ContextualRelevancyMetric


def test_rag_system():
    cases = load_regression_cases()
    faith = FaithfulnessMetric(threshold=0.85)
    rel = ContextualRelevancyMetric(threshold=0.7)
    for case in cases:
        faith.measure(case)
        assert faith.score >= 0.85, f"faithfulness regression on {case.id}"
        rel.measure(case)
        assert rel.score >= 0.7, f"relevancy regression on {case.id}"
```

作为 pytest 文件交付。在每个 PR 上运行。回归时阻断合并。

### 第 5 步：从零做的玩具评测

见 `code/main.py`。只用标准库近似忠实性（答案主张与上下文的重叠）和相关性（答案 token 与问题 token 的重叠）。不是生产级。展示的是形状。

## 陷阱

- **没有校准。** 一个与人类标签相关性为 0.3 的评判器就是噪声。发布前要求一次校准运行。
- **自我评测。** 用同一个 LLM 既生成又评判，会把分数抬高 10–20%。评判器使用不同的模型家族。
- **成对评判中的位置偏差。** 评判器偏好先呈现的选项。始终随机化顺序并两边都跑。
- **原始聚合藏住失败。** 平均分 0.85 常常藏着 5% 的灾难性失败。始终检查底部的分位数。
- **黄金数据集腐烂。** 未版本化、随时间漂移的评测集会破坏纵向比较。每一次改动都给数据集打标签。
- **LLM 成本。** 在规模上，评判调用主导成本。使用能达到校准阈值的最便宜模型。GPT-4o-mini、Claude Haiku、Mistral-small。

## 用起来

2026 年的技术栈：

| 用例 | 框架 |
|---------|-----------|
| RAG 质量监控 | RAGAS（4 个指标） |
| CI/CD 回归门禁 | DeepEval + pytest |
| 自定义领域标准 | DeepEval 内的 G-Eval |
| 在线实时流量监控 | 带无参考模式的 RAGAS |
| 人在回路中的抽查 | 带标注 UI 的 LangSmith 或 Phoenix |
| 红队 / 安全评测 | Promptfoo + DeepEval |

典型技术栈：RAGAS 做监控，DeepEval 做 CI，G-Eval 做新维度。三个都跑；它们的分歧是有用的。

## 交付

保存为 `outputs/skill-eval-architect.md`：

```markdown
---
name: eval-architect
description: Design an LLM evaluation plan with calibrated judge and CI gates.
version: 1.0.0
phase: 5
lesson: 27
tags: [nlp, evaluation, rag]
---

Given a use case (RAG / agent / generative task), output:

1. Metrics. Faithfulness / relevance / context-precision / context-recall + any custom G-Eval metrics with criteria.
2. Judge model. Named model + version, rationale for cost vs accuracy.
3. Calibration. Hand-labeled set size, target Spearman rho vs human > 0.7.
4. Dataset versioning. Tag strategy, change log, stratification.
5. CI gate. Thresholds per metric, regression-window logic, bottom-quantile alert.

Refuse to rely on a judge untested against ≥50 human-labeled examples. Refuse self-evaluation (same model generates + judges). Refuse aggregate-only reporting without bottom-10% surfacing. Flag any pipeline where judge upgrade lands without parallel baseline eval.
```

## 练习

1. **简单。** 在 10 个带已知幻觉的 RAG 例子上使用 RAGAS。验证忠实性指标抓住了每一个。
2. **中等。** 为正确性人工标注 50 个问答答案，分数 0–1。用 G-Eval 打分。测量评判器与人类之间的 Spearman rho。
3. **困难。** 用 DeepEval 构建一个 pytest CI 门禁。有意让检索器回归。验证门禁失败。通过对最低 10% 的阈值检查增加底部分位数告警。

## 关键术语

| 术语 | 人们怎么说 | 它实际的意思 |
|------|-----------------|-----------------------|
| LLM-as-judge | 用 LLM 打分 | 提示一个评判模型，按评分量表给输出打 0–1 分。 |
| RAGAS | RAG 指标库 | 带 4 个无参考 RAG 指标的开源评测框架。 |
| 忠实性（Faithfulness） | 答案是否锚定在检索内容上？ | 答案主张中被检索上下文所蕴含的比例。 |
| 上下文精确率（Context precision） | 检索到的块相关吗？ | 真正重要的 top-K 块所占的比例。 |
| 上下文召回率（Context recall） | 检索找到了一切吗？ | 被检索到的块所支持的黄金答案主张的比例。 |
| G-Eval | 自定义 LLM 评判器 | 评分量表 + 思维链评测步骤 + 0–1 分数。 |
| 校准（Calibration） | 信任但要验证 | 评判器分数与人类分数之间的 Spearman 相关。 |

## 延伸阅读

- [Es et al. (2023). RAGAS: Automated Evaluation of Retrieval Augmented Generation](https://arxiv.org/abs/2309.15217) — RAGAS 论文。
- [Liu et al. (2023). G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment](https://arxiv.org/abs/2303.16634) — G-Eval 论文。
- [DeepEval docs](https://deepeval.com/docs/metrics-introduction) — 开放的生产技术栈。
- [Zheng et al. (2023). Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena](https://arxiv.org/abs/2306.05685) — 偏差、校准、限度。
- [MLflow GenAI Scorer](https://mlflow.org/blog/third-party-scorers) — 整合 RAGAS、DeepEval、Phoenix 的统一框架。
