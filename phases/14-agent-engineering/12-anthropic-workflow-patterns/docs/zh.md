# Anthropic 的工作流模式：简单优于复杂

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> Schluntz 与 Zhang（Anthropic，2024 年 12 月）区分工作流（预定义路径）和智能体（动态的工具使用）。五种工作流模式覆盖大多数情况。从直接的 API 调用开始。只有当步骤无法预测时，才加入智能体。

**Type:** 理解 + 动手做
**Languages:** Python（标准库）
**Prerequisites:** 第 14 阶段 · 01（智能体循环）
**Time:** ~60 分钟

## 学习目标

- 说出 Anthropic 的五种工作流模式：prompt chaining（提示词链）、routing（路由）、parallelization（并行化）、orchestrator-workers（编排器–工作者）、evaluator-optimizer（评测器–优化器）。
- 解释智能体与工作流的区别，以及各自的工程成本。
- 识别何时选工作流而不是智能体（以及反过来）。
- 对照一个脚本化 LLM，用标准库实现全部五种模式。

## 问题

团队为那些只需要一次函数调用的问题去抓多智能体框架。成本是真实的：框架增加层次，遮住提示词，藏起控制流，并招来过早的复杂度。Schluntz 与 Zhang 2024 年 12 月的文章是被引用最多的行业回推：从简单开始，只有当复杂度配得上它的成本时才增加。

## 概念

### 工作流与智能体

- **工作流。** LLM 和工具通过预定义的代码路径编排。工程师拥有这张图。
- **智能体。** LLM 动态地指挥自己的工具，并走出自己的步骤。模型拥有这张图。

两者各有其位。工作流更便宜、更快、更容易调试。智能体打开开放式问题，但让失败模式更难推理。

### 增强 LLM

五种模式的基础：一个 LLM，接上三种能力——搜索（检索）、工具（动作）、记忆（持久化）。任何 API 调用都可以使用这些。

### 五种模式

1. **提示词链（prompt chaining）。** 调用 1 的输出是调用 2 的输入。当任务有干净的线性分解时使用。步骤之间可以有可选的程序化闸门。

2. **路由（routing）。** 一个分类器 LLM 挑选要调用哪个下游 LLM 或工具。当类别不同的输入需要不同处理时使用（一线支持 vs 退款 vs 缺陷 vs 销售）。

3. **并行化（parallelization）。** 并发运行 N 次 LLM 调用，再聚合结果。两种形状：分段（不同的块）和投票（同一提示词，N 次运行，多数或综合）。

4. **编排器–工作者（orchestrator-workers）。** 一个编排器 LLM 动态决定运行哪些工作者（也是 LLM），并综合它们的输出。和智能体循环相似，但编排器不会无限循环。

5. **评测器–优化器（evaluator-optimizer）。** 一个 LLM 提出答案，另一个 LLM 评测它。迭代直到评测器通过。这是推广后的 Self-Refine（第 05 课）。

### 工作流在哪里胜过智能体

- **可预测的任务。** 如果你能枚举步骤，你就应该枚举。
- **成本有界的任务。** 工作流的步数有上界；智能体可能螺旋上升。
- **合规有界的任务。** 审计员想读这张图，而不是从轨迹里推断它。

### 智能体在哪里胜过工作流

- **开放式研究。** 当下一步取决于上一步返回了什么。
- **长度可变的任务。** 几分钟到几小时的工作，步数未知。
- **新领域。** 当你还不知道正确的工作流时——先探索，以后再固化。

### 上下文工程这个伴生学科

「Effective context engineering for AI agents」（Anthropic 2025）把相邻的学科形式化了：20 万的窗口是预算，不是容器。包含什么、何时压缩、何时让上下文增长。本课程在第 14 阶段关于上下文压缩的课里有详细展开（重新编号之前，是第 14 阶段更早的第 06 课）。

```figure
workflow-chain
```

## 动手做

`code/main.py` 对照一个 `ScriptedLLM` 实现全部五种工作流模式：

- `prompt_chain(input, steps)`——顺序的。
- `route(input, classifier, handlers)`——分类加分发。
- `parallel_vote(prompt, n, aggregator)`——N 次运行，再聚合。
- `orchestrator_workers(task, workers)`——编排器挑选工作者。
- `evaluator_optimizer(task, proposer, evaluator, max_iter)`——循环直到通过。

运行：

```
python3 code/main.py
```

每种模式都打印自己的轨迹。每种模式的代码大约 10–15 行；框架的成本以千行计。

## 使用

- 大多数任务用直接的 API 调用。
- 只有当模式确实需要持久状态（LangGraph）、actor 模型并发（AutoGen v0.4）或角色模板（CrewAI）时，才用框架。
- 当你想要 Claude Code 的运行壳形状、又不想重建它时，去用 Claude Agent SDK。

## 交付

`outputs/skill-workflow-picker.md` 为给定的任务描述挑选正确的模式，包括决策理由，以及当工作流不够时重构到智能体的路径。

## 练习

1. 实现带置信度阈值的路由。低于阈值 → 升级给人类。一线支持用例的阈值落在哪里？
2. 给 `parallel_vote` 加超时。一次调用挂起时会发生什么？缺票时你如何聚合？
3. 把 `evaluator_optimizer` 变成一个 bandit：跨迭代保留前 2 个输出，这样后来的好结果不会被后来的坏结果覆盖。
4. 把提示词链和路由组合：一个路由器挑选三条链中的一条。相对单个大提示词方案，测量 token 成本。
5. 选你的一个生产功能。画出工作流图。数步骤。这里用智能体真的会更好吗？

## 关键术语

| 术语 | 人们怎么说 | 它实际的意思 |
|------|----------------|------------------------|
| 工作流 | 「预定义流程」 | 工程师拥有的 LLM 与工具调用图 |
| 智能体 | 「自主 AI」 | 模型拥有的图；动态的工具指挥 |
| 增强 LLM | 「带工具的 LLM」 | LLM + 搜索 + 工具 + 记忆；原子单元 |
| 提示词链 | 「顺序调用」 | 第 N 次调用的输出是第 N+1 次调用的输入 |
| 路由 | 「分类器分发」 | 挑选哪条链 / 哪个模型处理输入 |
| 并行化 | 「扇出」 | N 次并发调用；按分段或投票聚合 |
| 编排器–工作者 | 「分发智能体」 | 编排器 LLM 动态挑选专家 LLM |
| 评测器–优化器 | 「提议者 + 裁判」 | 迭代直到评测器通过；推广后的 Self-Refine |

## 延伸阅读

- [Anthropic, Building Effective Agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) — 五种工作流模式
- [Anthropic, Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — 伴生学科
- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) — 有状态的图何时配得上它们的成本
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) — 产品化的编排器–工作者模式
