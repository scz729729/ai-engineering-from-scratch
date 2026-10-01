# 定制课：带评测的工具型 RAG Agent

给已经会写 Java、Python、JavaScript，用过 Dify 和 n8n，做过提示词，并对大模型有基本概念的人。

上游仍是 [rohitg00/ai-engineering-from-scratch](https://github.com/rohitg00/ai-engineering-from-scratch) 的 523 节、20 个阶段。这份文件只规定你这 10 周学哪几节、跳过什么、最后交出什么。下表每节都有中文课文 `docs/zh.md`。代码、命令和标识符保持英文，以同目录 `docs/en.md` 和 `code/` 为准。

## 目标

10 周、每周 8–10 小时，合计约 **84 小时**。结束时你有一个自己写的 Agent，而不是一条 Dify / n8n 工作流。

它要能：

- 对你自己的文档做检索问答
- 通过 MCP 调用至少两个真实工具
- 用一组固定失败用例给自己打分
- 记下每次调用的 token 和费用
- 在文档里埋了间接提示注入时，不跟着注入走

第 0–10 阶段不进主线。数学、视觉、语音、从零预训练、强化学习都留在仓库里，卡住再查。LoRA 微调（第 11 阶段第 8 课）也不学，这条路径不改模型权重。

## 怎么学一节课

1. 打开下表里的链接，读 `docs/zh.md`。
2. 关键代码自己打，不要只看。
3. 在仓库根目录运行该课给出的命令。
4. 在 `notes/week-NN.md` 留下：命令、工作目录、退出码、一段有意义的输出、你改过或生成的文件。
5. 能口头讲清输出，并能做一个小改动，再进下一节。

提示词课（第 11 阶段第 1、2 课）默认跳过。你已经有这块经验。只有在结构化输出或检索答非所问时，才回去翻。

## 10 周课表

时间是上游 `ROADMAP.md` 里的估计，加上表中写明的练习。

### 第 1 周 · 从提示词走到检索（约 7.5 小时）

先准备语料：选一份你能公开的文档，不少于能切出 20 个块。不要放密钥、客户数据和内网地址。环境缺依赖时，只做 [第 0 阶段第 1 课](../phases/00-setup-and-tooling/01-dev-environment/docs/zh.md)，不要把第 0 阶段学完。

| 课 | 估计 | 这节要留下的东西 |
| --- | --- | --- |
| [11.03 结构化输出](../phases/11-llm-engineering/03-structured-outputs/docs/zh.md) | 75 分钟 | 一个会校验失败的 schema |
| [11.04 嵌入](../phases/11-llm-engineering/04-embeddings/docs/zh.md) | 75 分钟 | 你的语料里两句近义、两句无关的相似度 |
| [11.05 上下文工程](../phases/11-llm-engineering/05-context-engineering/docs/zh.md) | 75 分钟 | 一段你主动删掉的上下文，以及为什么删 |
| [11.06 RAG](../phases/11-llm-engineering/06-rag/docs/zh.md) | 75 分钟 | 能对你的语料回答一个问题 |
| [11.07 进阶 RAG](../phases/11-llm-engineering/07-advanced-rag/docs/zh.md) | 75 分钟 | 同问句改一种切块或重排后的对比 |

### 第 2 周 · 先会打分，再加功能（约 7.5 小时）

| 课 | 估计 | 这节要留下的东西 |
| --- | --- | --- |
| [11.10 评测](../phases/11-llm-engineering/10-evaluation/docs/zh.md) | 45 分钟 | 5 条问答的对错记录 |
| [11.11 缓存、限流、成本](../phases/11-llm-engineering/11-caching-cost/docs/zh.md) | 45 分钟 | 一次调用的 token 和估算费用 |
| [11.12 护栏](../phases/11-llm-engineering/12-guardrails/docs/zh.md) | 45 分钟 | 一条被拦住的输入和拦住它的规则 |
| [11.13 生产级 LLM 应用](../phases/11-llm-engineering/13-production-app/docs/zh.md) | 120 分钟 | 应用的目录结构，而不是笔记本 |
| [11.15 提示缓存](../phases/11-llm-engineering/15-prompt-caching/docs/zh.md) | 60 分钟 | 有缓存和无缓存的费用差 |
| [11.17 框架取舍](../phases/11-llm-engineering/17-agent-framework-tradeoffs/docs/zh.md) | 45 分钟 | 半页对照：这件事在 Dify 或 n8n 里是哪个节点，在代码里是哪一层 |
| [5.27 评测框架](../phases/05-nlp-foundations-to-advanced/27-llm-evaluation-frameworks/docs/zh.md) | 75 分钟 | faithfulness 和 answer relevance 各用一句话解释，并各举你语料里的一个例子 |

### 第 3 周 · 工具（约 7.5 小时）

第 11 阶段第 9 课和第 13 阶段前几课都在讲工具。按这个顺序走，避免同一概念学两遍却不写 schema。

| 课 | 估计 | 这节要留下的东西 |
| --- | --- | --- |
| [13.01 工具接口](../phases/13-tools-and-protocols/01-the-tool-interface/docs/zh.md) | 45 分钟 | 一个工具的输入、输出、失败三种形状 |
| [13.02 函数调用](../phases/13-tools-and-protocols/02-function-calling-deep-dive/docs/zh.md) | 75 分钟 | 一次真实的工具往返 |
| [11.09 函数调用与工具使用](../phases/11-llm-engineering/09-function-calling/docs/zh.md) | 75 分钟 | 模型选错工具时你怎么发现 |
| [13.03 并行和流式工具调用](../phases/13-tools-and-protocols/03-parallel-and-streaming-tool-calls/docs/zh.md) | 75 分钟 | 两个工具谁先返回，结果如何合并 |
| [13.05 工具 schema 设计](../phases/13-tools-and-protocols/05-tool-schema-design/docs/zh.md) | 45 分钟 | 毕业项目两个工具的草案 schema |
| [13.15 工具投毒](../phases/13-tools-and-protocols/15-mcp-security-tool-poisoning/docs/zh.md) | 60 分钟 | 一段会误导模型的工具描述，以及你拒绝它的规则 |
| 练习 | 75 分钟 | 把两个 schema 写成代码里的函数签名，先不接 MCP |

两个工具就定成：

1. `search_docs`：只查你的语料。
2. `get_cost_report`：读本地成本日志并返回汇总。它碰的是另一份状态，不是第二次搜索。

### 第 4 周 · MCP（约 9 小时）

服务器用这节课里的 Python 或 TypeScript 都可以。Agent 循环保持 Python。

| 课 | 估计 | 这节要留下的东西 |
| --- | --- | --- |
| [13.06 MCP 基础](../phases/13-tools-and-protocols/06-mcp-fundamentals/docs/zh.md) | 55 分钟 | 一次 JSON-RPC 请求和响应原文 |
| [13.07 写一个 MCP 服务器](../phases/13-tools-and-protocols/07-building-an-mcp-server/docs/zh.md) | 85 分钟 | 一个能列出工具的服务器 |
| [13.08 写一个 MCP 客户端](../phases/13-tools-and-protocols/08-building-an-mcp-client/docs/zh.md) | 85 分钟 | 客户端发现并调用其中一个工具 |
| [13.09 传输](../phases/13-tools-and-protocols/09-mcp-transports/docs/zh.md) | 65 分钟 | 你选定 stdio 或 HTTP 的理由 |
| [13.28 工具契约](../phases/13-tools-and-protocols/28-mcp-tool-contracts-and-content/docs/zh.md) | 120 分钟 | `search_docs` 和 `get_cost_report` 都挂在这个服务器上 |
| [13.29 可靠性](../phases/13-tools-and-protocols/29-mcp-reliability-cancellation-and-flow-control/docs/zh.md) | 120 分钟 | 工具超时或取消时的一条记录 |

第 13 阶段第 22–27 课是 Agent Skills 专线，第 16–18 课是 OAuth 和生产鉴权。这条毕业项目用本地 stdio，先不学。以后要接到别人的远程服务器，再回来。

### 第 5 周 · 循环（约 7.5 小时）

| 课 | 估计 | 这节要留下的东西 |
| --- | --- | --- |
| [14.01 Agent 循环](../phases/14-agent-engineering/01-the-agent-loop/docs/zh.md) | 60 分钟 | 一个没有框架的循环，能调用你的两个工具 |
| [14.02 先计划再执行](../phases/14-agent-engineering/02-rewoo-plan-and-execute/docs/zh.md) | 60 分钟 | 一个问题的计划步骤，以及哪一步被改掉了 |
| [14.06 工具使用](../phases/14-agent-engineering/06-tool-use-and-function-calling/docs/zh.md) | 60 分钟 | 循环里一次检索、一次费用查询的轨迹 |
| [14.12 工作流模式](../phases/14-agent-engineering/12-anthropic-workflow-patterns/docs/zh.md) | 60 分钟 | 标出你的场景是「工作流」还是「Agent」，以及为什么 |
| [11.16 状态机](../phases/11-llm-engineering/16-langgraph-state-machines/docs/zh.md) | 75 分钟 | 检索、工具、回答、拒绝四个状态 |
| [14.03 反思](../phases/14-agent-engineering/03-reflexion-verbal-rl/docs/zh.md) | 60 分钟 | 一次答错后，循环怎样改口 |
| [14.26 失败模式](../phases/14-agent-engineering/26-failure-modes-agentic/docs/zh.md) | 60 分钟 | 你的循环已经会犯的一种错 |

### 第 6 周 · 记忆、观测、用评测驱动（约 7 小时）

| 课 | 估计 | 这节要留下的东西 |
| --- | --- | --- |
| [14.07 记忆](../phases/14-agent-engineering/07-memory-virtual-context-memgpt/docs/zh.md) | 75 分钟 | 什么放进窗口，什么只留在检索里 |
| [14.23 OpenTelemetry](../phases/14-agent-engineering/23-otel-genai-conventions/docs/zh.md) | 60 分钟 | 一次调用的 span 里有哪些字段 |
| [14.24 可观测性](../phases/14-agent-engineering/24-agent-observability-platforms/docs/zh.md) | 45 分钟 | 你选一个本地日志或一个平台，并说明不选另一个的原因 |
| [14.27 提示注入](../phases/14-agent-engineering/27-prompt-injection-defense/docs/zh.md) | 75 分钟 | 一条注入样本和拦截结果 |
| [14.30 用评测驱动开发](../phases/14-agent-engineering/30-eval-driven-agent-development/docs/zh.md) | 60 分钟 | 先写失败用例，再改循环 |
| [14.38 验证闸门](../phases/14-agent-engineering/38-verification-gates/docs/zh.md) | 55 分钟 | 一个不过闸门就不能回答用户的条件 |
| [14.29 生产运行时](../phases/14-agent-engineering/29-production-runtimes/docs/zh.md) | 60 分钟 | 你的程序如何启动、如何停 |

### 第 7 周 · 费用和边界（约 6 小时）

不学 GPU、Kubernetes、量化和推测解码。你现在要为一次请求付钱，不是自建推理集群。

| 课 | 估计 | 这节要留下的东西 |
| --- | --- | --- |
| [17.08 推理指标](../phases/17-infrastructure-and-production/08-inference-metrics-goodput/docs/zh.md) | 60 分钟 | 一次请求的 TTFT 和总延迟 |
| [17.13 观测栈](../phases/17-infrastructure-and-production/13-llm-observability/docs/zh.md) | 60 分钟 | 你要看的三张图：延迟、费用、失败率 |
| [17.16 模型路由](../phases/17-infrastructure-and-production/16-model-routing/docs/zh.md) | 60 分钟 | 哪类问题走便宜模型，哪类必须走强模型 |
| [17.19 AI 网关](../phases/17-infrastructure-and-production/19-ai-gateways/docs/zh.md) | 60 分钟 | 密钥放在哪里，应用代码里不再出现密钥 |
| [17.25 密钥、PII、审计](../phases/17-infrastructure-and-production/25-security-secrets-audit/docs/zh.md) | 60 分钟 | 一条审计日志样例 |
| [17.27 FinOps](../phases/17-infrastructure-and-production/27-finops-llms/docs/zh.md) | 60 分钟 | 按请求累计的费用表 |

### 第 8 周 · 注入和评分尺（约 7 小时）

| 课 | 估计 | 这节要留下的东西 |
| --- | --- | --- |
| [18.15 间接提示注入](../phases/18-ethics-safety-alignment/15-indirect-prompt-injection/docs/zh.md) | 75 分钟 | 一段藏在文档里的指令 |
| [19.68 RAG 评测指标](../phases/19-capstone-projects/68-rag-eval-precision-recall/docs/zh.md) | 90 分钟 | 你的 5 条可回答问题上的命中情况 |
| [19.83 注入检测](../phases/19-capstone-projects/83-prompt-injection-detector/docs/zh.md) | 90 分钟 | 一个能标出注入的小检测器或规则 |
| [19.27 评测夹具](../phases/19-capstone-projects/27-eval-harness-fixture-tasks/docs/zh.md) | 90 分钟 | 一个能重复跑的用例文件 |
| 起草 | 60 分钟 | 毕业项目的 15 条用例初稿，先放进仓库，先不必全过 |

### 第 9–10 周 · 毕业项目（约 22 小时）

把前 8 周的零件收成一个程序。运行时是你的 Python 进程。Dify 和 n8n 不能当这个程序的引擎。最后用半页纸写对照：同一条流程在 Dify 或 n8n 里怎么拖，现在哪几层是你自己的代码。

必须有：

1. **语料。** 你自己的文档，切块后不少于 20 块。
2. **检索。** `search_docs` 只查这份语料。
3. **第二个工具。** `get_cost_report` 读本地成本日志。
4. **MCP。** 两个工具都由 MCP 服务器暴露，Agent 通过客户端调用。
5. **循环。** 无框架或只用你能讲清的一层。状态至少包括检索、工具、回答、拒绝。
6. **15 条固定用例。**
   - 5 条文档里答得出来
   - 3 条文档里没有，必须拒绝
   - 3 条必须调用费用工具
   - 2 条工具失败（超时或参数错误）
   - 2 条文档内的间接提示注入，且不能让费用工具或检索去执行注入里的指令
7. **费用。** 每次请求记下模型、输入 token、输出 token、估算金额。
8. **成绩。** 15 条里至少 12 条行为符合预期。剩下的 3 条写成已知失败，不要改用例去凑分。
9. **说明。** 项目目录里的 `README.md` 写清如何准备密钥、如何跑评测、最后的通过数。

建议的目录（放在本仓库之外，不要把密钥提交进这条 fork）：

```text
tool-rag-agent/
  README.md
  corpus/
  src/agent.py
  src/mcp_server.py
  eval/cases.jsonl
  eval/run_eval.py
  logs/cost.jsonl
```

22 小时可以这样分：第 9 周把检索、两个工具和循环跑通（约 10 小时），第 10 周补评测、注入、费用和 README（约 12 小时）。

## 时间合计

| 周 | 小时 |
| --- | --- |
| 1 检索 | 7.5 |
| 2 评测与成本 | 7.5 |
| 3 工具 | 7.5 |
| 4 MCP | 9 |
| 5 循环 | 7.5 |
| 6 记忆与观测 | 7 |
| 7 费用与边界 | 6 |
| 8 注入与评分尺 | 7 |
| 9–10 毕业项目 | 22 |
| **合计** | **81** |

表内取整后约 81 小时。某一课超时是正常的，总预算看到 84 小时。每周若稳定有 10 小时，多出来的时间用在下面的加选题，不要提前加第 0–10 阶段。

## 有余力再学

按这个顺序，加到大约 100 小时为止：

1. [13.30 MCP 注册与漂移](../phases/13-tools-and-protocols/30-mcp-registry-supply-chain-and-drift/docs/zh.md)（90 分钟）
2. [14.13 持久执行与检查点](../phases/14-agent-engineering/13-langgraph-stateful-graphs/docs/zh.md)（75 分钟）
3. [17.20 影子发布和金丝雀](../phases/17-infrastructure-and-production/20-shadow-canary-progressive/docs/zh.md)（60 分钟）
4. [17.22 压测](../phases/17-infrastructure-and-production/22-load-testing-llm-apis/docs/zh.md)（75 分钟）
5. [14.16 追踪与护栏 SDK](../phases/14-agent-engineering/16-openai-agents-sdk/docs/zh.md)（75 分钟），只用来对照你自己的循环

## 明确不学

| 范围 | 原因 |
| --- | --- |
| 第 0–10 阶段的完整课表 | 编程和模型概念已经够进入第 11 阶段 |
| 第 11 阶段第 1、2、8 课 | 提示词你会；这条路径不微调 |
| 第 12 阶段多模态 | 毕业项目不看图 |
| 第 13 阶段第 10–14、16–19、22–27 课 | 远程鉴权、A2A、Skills 专线，和这个本地项目无关 |
| 第 14 阶段第 8–11、14–15、19–22、31–54 课 | 编码工作台和产品判断专线，不是这条路径 |
| 第 15、16 阶段 | 自改进和群体系统，超出 10 周 |
| 第 17 阶段里的 GPU、集群、量化、推测解码 | 你调用的是模型 API |
| 第 19 阶段其余毕业项目 | 只做一个 |

## 和上游同步

这条 fork 的远程是 `origin`。上游可以另加：

```bash
git remote add upstream https://github.com/rohitg00/ai-engineering-from-scratch.git
git fetch upstream
git merge upstream/main
```

定制内容就是本文件，以及 README 顶部指向它的那一段。其余课保持上游原样。
