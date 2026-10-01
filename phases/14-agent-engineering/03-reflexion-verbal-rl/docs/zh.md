# Reflexion：言语强化学习

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 基于梯度的强化学习要修好一种失败模式，需要成千上万次试验和一个 GPU 集群。Reflexion（Shinn 等人，NeurIPS 2023）用自然语言来做：每次失败的试验之后，智能体写下一份反思，存进情景记忆，并让下一次试验以这段记忆为条件。这就是 Letta 的 sleep-time compute、Claude Code 的 CLAUDE.md 学习记录，以及 pro-workflow 的 learn-rule 背后的模式。

**Type:** 动手做
**Languages:** Python（标准库）
**Prerequisites:** 第 14 阶段 · 01（智能体循环），第 14 阶段 · 02（ReWOO）
**Time:** ~60 分钟

## 学习目标

- 说出 Reflexion 的三个组件（Actor、Evaluator、Self-Reflector），以及情景记忆的角色。
- 用标准库实现一个 Reflexion 循环，带二元评测器、反思缓冲区和全新的重试。
- 为给定任务在标量、启发式和自我评测的反馈来源之间做选择。
- 解释为什么言语强化能抓住那些基于梯度的强化学习需要成千上万次试验才能修好的错误。

## 问题

一个智能体搞砸了一项任务。在标准强化学习里，你会再跑成千上万次试验，计算梯度，更新权重。昂贵、缓慢，而且大多数生产智能体没有为每一次失败准备的训练预算。

Reflexion（Shinn 等人，arXiv:2303.11366）问了一个不同的问题：如果智能体只是想想自己为什么失败，然后把这个想法放进提示词里再试一次呢？没有权重更新。没有梯度。只是试验之间存储的自然语言。

结果：在 ALFWorld 上它打败了 ReAct 和其他未微调的基线。在 HotpotQA 上它比 ReAct 更好。在代码生成（HumanEval / MBPP）上，它在当时达到了最高水平。全程没有一步梯度。

## 概念

### 三个组件

```
Actor         : generates a trajectory (ReAct-style loop)
Evaluator     : scores the trajectory — binary, heuristic, or self-eval
Self-Reflector: writes a natural-language reflection on the failure
```

再加上一个数据结构：

```
Episodic memory: list of prior reflections, prepended to the next trial's prompt
```

一次试验运行 Actor。Evaluator 给它打分。如果分数低，Self-Reflector 产出一份反思（「我选错了工具，因为我把问题误读成在问 X，而它问的是 Y」）。反思进入情景记忆。下一次试验从头开始，但看得到这份反思。

### 三种评测器

1. **标量**——一个外部的二元信号。ALFWorld 成功或失败。HumanEval 测试通过或失败。最简单，信号最强。
2. **启发式**——预定义的失败签名。「如果智能体连续两次产生同一个动作，就标为卡住。」「如果轨迹超过 50 步，就标为低效。」
3. **自我评测**——LLM 给自己的轨迹打分。没有真值可用时需要它。信号较弱；和基于工具的核实配合得很好（第 05 课——CRITIC）。

2026 年的默认是混合：有标量就用标量，没有就用自我评测，启发式当作安全栏杆。

### 为什么这能推广

Reflexion 与其说是一种新算法，不如说是一个被命名的模式。几乎每一个生产环境里「自我修复」的智能体都在跑某种变体：

- Letta 的 sleep-time compute（第 08 课）：一个单独的智能体反思过去的对话，并写入记忆块。
- Claude Code 的 `CLAUDE.md` / 「保存记忆」模式：反思被捕捉为学习记录，加到未来会话的前面。
- pro-workflow 的 `/learn-rule` 命令：纠正被捕捉为显式规则。
- LangGraph 的反思节点：一个给输出打分、并在需要时路由去精炼的节点。

它们都来自同一个洞察：自然语言是足够丰富的介质，可以在多次运行之间携带「我从失败中学到了什么」。

### 它何时有效，何时无效

Reflexion 在这些时候有效：

- 有清楚的失败信号（测试失败、工具错误、错误答案）。
- 任务类别可复现（同一类问题可以再问一次）。
- 反思有改进轨迹的空间（动作预算足够）。

Reflexion 在这些时候没有帮助：

- 智能体第一次尝试就已经成功。
- 失败是外部的（网络中断、工具坏了）——反思「网络中断了」对未来的运行没有帮助。
- 反思变成迷信——存储一段关于某次偶发不稳定运行的叙事。

2026 年的坑：记忆腐化。反思不断累积；有些过时或错误；情景缓冲区变大时，重跑变慢。缓解：周期性压缩（第 06 课）、反思上的 TTL，或一个单独的休眠期清理智能体（Letta）。

```figure
react-trace
```

## 动手做

`code/main.py` 在一个玩具谜题上实现 Reflexion：产出一个和为目标值的三元素列表。Actor 发出候选列表；Evaluator 检查和；Self-Reflector 写一行哪里错了。反思进入下一次试验的情景记忆。

组件：

- `Actor`——一段脚本化策略，看到反思时会改进。
- `Evaluator.binary()`——对目标和做通过 / 失败判断。
- `SelfReflector`——生成一行失败诊断。
- `EpisodicMemory`——一个带 TTL 语义的有界列表。

运行：

```
python3 code/main.py
```

轨迹展示三次试验。试验 1 失败，存下一份反思，试验 2 看到反思并有改进但仍然失败，试验 3 成功。和一次基线运行（没有反思）比较——它一直卡在试验 1 的答案上。

## 使用

LangGraph 把反思作为一种节点模式交付。Claude Code 的 `/memory` 命令和 pro-workflow 的 `/learn-rule` 把情景缓冲区外化成一个 markdown 文件。Letta 的 sleep-time compute 在空闲时运行 Self-Reflector，使主智能体保持受延迟约束。OpenAI Agents SDK 不直接交付 Reflexion；你用一个按分数拒绝轨迹的自定义 Guardrail，以及一个跨运行存活的记忆 `Session` 来构建它。

## 交付

`outputs/skill-reflexion-buffer.md` 创建并维护一个情景缓冲区，带反思捕捉、TTL 和去重。给定一个任务类别和一次失败，它发出一份真正能帮助下一次试验的反思（而不是泛泛的「再小心一点」）。

## 练习

1. 从二元评测器换成返回距离度量（离目标有多远）的标量评测器。它收敛得更快吗？
2. 给反思加上 10 次试验的 TTL。过了那个点之后，更老的反思是有害还是有帮助？
3. 实现启发式评测器：如果同一个动作重复，就把试验标为卡住。这和 Self-Reflector 如何相互作用？
4. 用一个忽略反思的对抗性 Actor 跑 Reflexion。迫使 Actor 注意到反思的最小提示词工程是什么？
5. 阅读 Reflexion 论文第 4 节关于 AlfWorld 的部分。在概念上复现 130% 的成功率提升：相对普通 ReAct 的关键增量是什么？

## 关键术语

| 术语 | 人们怎么说 | 它实际的意思 |
|------|----------------|------------------------|
| Reflexion | 「自我纠正」 | Shinn 等人 2023——Actor、Evaluator、Self-Reflector，加上情景记忆 |
| 言语强化 | 「没有梯度的学习」 | 加在下一次试验提示词前面的自然语言反思 |
| 情景记忆 | 「按任务的反思」 | 一个任务类别先前反思的有界缓冲区 |
| 标量评测器 | 「二元成功信号」 | 来自真值的通过 / 失败或数值分数 |
| 启发式评测器 | 「基于模式的检测器」 | 预定义的失败签名（例如卡住循环、步数过多） |
| 自我评测器 | 「用 LLM 当自己轨迹的裁判」 | 没有真值时信号较低的后备——与基于工具的核实配对 |
| 记忆腐化 | 「陈旧的反思」 | 情景缓冲区塞满过时条目；用压缩 / TTL 修复 |
| 休眠期反思 | 「异步自我反思」 | 把 Self-Reflector 放在热路径之外，使主智能体保持快 |

## 延伸阅读

- [Shinn et al., Reflexion: Language Agents with Verbal Reinforcement Learning (arXiv:2303.11366)](https://arxiv.org/abs/2303.11366) — 经典论文
- [Letta, Sleep-time Compute](https://www.letta.com/blog/sleep-time-compute) — 生产中的异步反思
- [Anthropic, Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — 把情景缓冲区当作上下文的一部分来管理
- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) — 反思节点模式
