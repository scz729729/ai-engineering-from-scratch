# ReWOO 与 Plan-and-Execute：把规划拆开

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> ReAct 把思考和行动交织在同一条流里。ReWOO 把它们分开：先给出一份大计划，再执行。token 少 5 倍，HotpotQA 上准确率 +4%，而且可以把规划器蒸馏进一个 7B 模型。Plan-and-Execute 把它推广成一种模式；Plan-and-Act 把它扩展到了网页导航。

**Type:** 动手做
**Languages:** Python（标准库）
**Prerequisites:** 第 14 阶段 · 01（智能体循环）
**Time:** ~60 分钟

## 学习目标

- 解释为什么 ReWOO 的 Planner / Worker / Solver 拆分能省 token，并且比 ReAct 的交织循环更稳健。
- 实现一份计划 DAG、一个按依赖顺序执行的执行器，以及一个把工作者输出组合起来的 solver——全部使用标准库。
- 用 2026 年「五种工作流模式」的框架（Anthropic）判断一项任务应当先计划再执行，还是用交织的 ReAct。
- 识别长程网页或移动任务何时需要 Plan-and-Act 的合成计划数据。

## 问题

ReAct 交织的思考–行动–观察循环简单、灵活，但每一次工具调用都得带上全部先前上下文——包括之前的每一次思考。token 用量随深度二次增长。更糟的是：循环中途某个工具失败时，模型必须从错误观察里重新推导整份计划。

ReWOO（Xu 等人，arXiv:2305.18323，2023 年 5 月）注意到这一点，并打了一个赌：事先把整件事计划好，并行取证，最后再组合答案。一次 LLM 调用做计划，N 次工具调用取证（可以并行），一次 LLM 调用求解。交换条件是灵活性变少（计划是静态的），换来好得多的 token 效率和更清楚的失败模式。

## 概念

### 三种角色

```
Planner:  user_question -> [plan_dag]
Workers:  [plan_dag]     -> [evidence]        (tool calls, possibly parallel)
Solver:   user_question, plan_dag, evidence -> final_answer
```

Planner 产出一份 DAG。每个节点点名一个工具、它的参数，以及它依赖哪些更早的节点（像 `#E1`、`#E2` 这样的引用）。Worker 按拓扑顺序执行节点。Solver 把一切缝合起来。

### 为什么 token 少 5 倍

ReAct 的提示词长度随步数线性增长。到第 10 步，提示词里含有思考 1 加行动 1 加观察 1，再加上思考 2 加行动 2 加观察 2，以此类推。每一个中间步骤还冗余地包含原始提示词。

ReWOO 付出一次规划器提示词（大），N 次很小的工作者提示词（每次只是工具调用，没有链条），以及一次 solver 提示词。在 HotpotQA 上，论文测得 token 大约少 5 倍，同时绝对准确率高 4 个百分点。

### 为什么它更稳健

如果在 ReAct 里工作者 3 失败，循环必须在流的中途从错误里推理出来。在 ReWOO 里，工作者 3 返回一个错误字符串；solver 在原始计划的上下文里看到它，可以优雅降级。失败定位是按节点，不是按步骤。

### 规划器蒸馏

论文的第二个结果：因为规划器看不到观察，你可以用 175B 教师的规划器输出微调一个 7B 模型。小模型负责规划；推理时不需要大模型。这现在已经是标准做法——2026 年的许多生产智能体使用小规划器加大执行器，或者反过来。

### Plan-and-Execute（2023）

LangChain 团队 2023 年 8 月的文章把 ReWOO 推广成一个模式名称：Plan-and-Execute。事先的规划器发出步骤列表，执行器运行每一步，一个可选的 replanner 可以在观察到结果之后修订。这比 ReWOO 更接近 ReAct（replanner 把观察带回到规划里），但保留了 token 节省。

### Plan-and-Act（Erdogan 等人，arXiv:2503.09572，ICML 2025）

Plan-and-Act 把这种模式扩展到长程网页和移动智能体。关键贡献是合成计划数据：一个带标签的轨迹生成器产出计划被显式写出来的训练数据。用来微调规划器模型，使它们在类似 WebArena 的任务上越过 30–50 步之后仍然能工作；在那些任务上，单条 ReAct 轨迹会失去连贯性。

### 何时选哪一种

| 模式 | 何时 |
|---------|------|
| ReAct | 短任务、未知环境、需要反应式的异常处理 |
| ReWOO | 工具已知的结构化任务、对 token 敏感、证据可并行 |
| Plan-and-Execute | 像 ReWOO，但在部分执行之后会重新规划 |
| Plan-and-Act | 长程（>30 步）、网页 / 移动 / computer-use |
| Tree of Thoughts | 搜索值得为之付费时（第 04 课） |

Anthropic 2024 年 12 月的指导：从最简单的开始。如果任务是一次工具调用加一份摘要，不要构建 ReWOO。如果任务是一份 40 步的研究作业，不要只做 ReAct。

```figure
rewoo-plan
```

## 动手做

`code/main.py` 实现一个玩具 ReWOO：

- `Planner`——一段脚本化策略，从提示词发出一份计划 DAG。
- `Worker`——通过注册表分发每个节点的工具调用。
- `Solver`——脚本化的组合，读取证据并产出最终答案。
- 依赖解析——像 `#E1` 这样的引用被替换成更早的工作者输出。

演示回答「法国首都的人口是多少，四舍五入到百万？」用的是两步计划：（1）查出首都，（2）查出人口，然后求解。

运行：

```
python3 code/main.py
```

轨迹先展示完整计划，然后是工作者结果，然后是 solver 的组合。把 token 计数（我们打印一个粗略的字符数）和一次 ReAct 风格的交织运行比较——在这种结构化任务上，ReWOO 赢。

## 使用

LangGraph 把 Plan-and-Execute 作为一份配方交付（ReAct 用 `create_react_agent`，计划–执行用自定义图）。CrewAI 的 Flows 直接编码这种模式：你事先定义任务，Flow DAG 执行它们。Plan-and-Act 的合成数据方法大体上仍是研究；运行时模式（显式的计划 DAG）通过 LangGraph 和 CrewAI Flows 进入生产。

## 交付

`outputs/skill-rewoo-planner.md` 根据工具目录，从用户请求生成一份 ReWOO 计划 DAG。它在交给执行器之前校验计划（无环、每个引用都已解析、每个工具都存在）。

## 练习

1. 为相互独立的计划节点并行化工作者执行。在一份有 2 个并行组的 6 节点 DAG 上，这能给你带来什么？
2. 增加一个 replanner 节点，在任何工作者返回错误时触发。把 ReWOO 变成 Plan-and-Execute 的最小改动是什么？
3. 用一个小模型（7B 级）替换 `Planner`，把 `Solver` 留在前沿模型上。比较端到端质量——这种拆分在哪里失败？
4. 阅读 ReWOO 论文第 4 节关于规划器蒸馏的部分。在概念上复现 175B → 7B 的结果：你需要什么训练数据，又如何给计划质量打分？
5. 把这个玩具移植到 Plan-and-Act 的轨迹形状：计划是序列，不是 DAG。哪些权衡变了？

## 关键术语

| 术语 | 人们怎么说 | 它实际的意思 |
|------|----------------|------------------------|
| ReWOO | 「没有观察的推理」 | 先计划，再并行取证，再求解——规划提示词里没有观察 |
| Plan-and-Execute | 「LangChain 的 plan-execute 模式」 | 带一个可选的执行后 replanner 节点的 ReWOO |
| Plan-and-Act | 「扩展后的 plan-execute」 | 显式的规划器 / 执行器拆分，并用合成计划训练数据应对长程任务 |
| 证据引用 | 「#E1、#E2、……」 | 计划节点占位符，在分发时被替换成先前的工作者输出 |
| 规划器蒸馏 | 「小规划器，大执行器」 | 用大教师的规划器轨迹微调一个小模型 |
| token 效率 | 「更少的往返」 | 论文中相对 ReAct，HotpotQA 上 token 少 5 倍 |
| DAG 执行器 | 「拓扑分发器」 | 按依赖顺序运行计划节点；每一层内部并行 |

## 延伸阅读

- [Xu et al., ReWOO: Decoupling Reasoning from Observations (arXiv:2305.18323)](https://arxiv.org/abs/2305.18323) — 经典论文
- [Erdogan et al., Plan-and-Act (arXiv:2503.09572)](https://arxiv.org/abs/2503.09572) — 带合成计划的扩展规划器–执行器
- [LangGraph Plan-and-Execute tutorial](https://docs.langchain.com/oss/python/langgraph/overview) — 框架配方
- [Anthropic, Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — 选择能工作的最简单模式
