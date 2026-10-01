# 生产运行时：队列、事件、Cron

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 生产智能体跑在六种运行时形态上：请求-响应、流式、持久执行、基于队列的后台、事件驱动，以及定时。先选形态，再选框架。可观测性在每一种形态上都是承重的。

**Type:** 理解
**Languages:** Python（标准库）
**Prerequisites:** Phase 14 · 13（LangGraph）、Phase 14 · 22（语音）
**Time:** ~60 分钟

## 学习目标

- 说出六种生产运行时形态，并把每一种对应到一种框架 / 产品模式。
- 解释为什么持久执行（LangGraph）对长程任务重要。
- 描述事件驱动运行时，以及 Claude Managed Agents 何时合适。
- 解释对多步智能体而言“可观测性是承重的”这一主张。

## 问题

生产智能体的失败方式是 Jupyter notebook 不会暴露的：第 37 步网络超时、用户在语音通话中途挂断、cron 任务在机器重启时死掉、后台工作进程内存耗尽。运行时形态决定哪些失败是可存活的。

## 概念

### 请求-响应

- 同步 HTTP。用户等待完成。
- 只对短任务（<30 秒）可行。
- 技术栈：Agno（Python + FastAPI）、Mastra（TypeScript + Express/Hono/Fastify/Koa）。
- 可观测性：标准 HTTP 访问日志 + OTel span。

### 流式

- 用 SSE 或 WebSocket 做渐进输出。
- LiveKit 把它扩展到语音/视频的 WebRTC（第 22 课）。
- 技术栈：任何支持流式的框架 + 一个处理 SSE/WS 的前端。
- 可观测性：逐块计时、首 token 延迟、尾延迟。

### 持久执行

- 每一步之后把状态做成检查点；失败时自动恢复。
- AutoGen v0.4 的 actor 模型把失败隔离到一个智能体（第 14 课）。
- LangGraph 的核心差异点（第 13 课）。
- 当步数未知且恢复成本高时，这是必需的。

### 基于队列 / 后台

- 作业进入队列，工作进程领取，结果通过 webhook 或 pub/sub 流回。
- 对长程智能体必需（按 Anthropic 的 computer use 公告，每个任务几十到几百步）。
- 技术栈：Celery（Python）、BullMQ（Node）、SQS + Lambda（AWS）、自定义。
- 可观测性：队列深度、每作业延迟分布、死信队列（DLQ）大小。

### 事件驱动

- 智能体订阅触发器：新邮件、PR 打开、cron 触发。
- Claude Managed Agents 开箱覆盖这一点（第 17 课）。
- CrewAI Flows（第 15 课）把事件驱动的确定性工作流结构化。
- 可观测性：触发来源、事件到启动的延迟、智能体延迟。

### 定时

- cron 形态的智能体，周期性运行。
- 与持久执行结合，使失败的夜间运行在下一次节拍恢复。
- 技术栈：Kubernetes CronJob + 一个持久框架；托管（Render cron、Vercel cron）。

### 2026 部署模式

- **CrewAI Flows** 用于事件驱动的生产。
- **Agno** 无状态 FastAPI，用于 Python 微服务。
- **Mastra** 服务器适配器（Express、Hono、Fastify、Koa），用于嵌入。
- **Pipecat Cloud / LiveKit Cloud** 用于托管语音（第 22 课）。
- **Claude Managed Agents** 用于托管的长时间异步。

### 可观测性是承重的

没有 OpenTelemetry GenAI span（第 23 课）加上 Langfuse/Phoenix/Opik 后端（第 24 课），你就无法调试一个在第 40 步失败的多步智能体。这对生产不是可选项。它是“我们调试得快”和“我们带着更多日志从头回放”之间的差别。

### 生产运行时会在哪里失败

- **选错形态。** 给一个 5 分钟的任务选请求-响应。用户挂断；工作进程堆积；重试叠加。
- **没有 DLQ。** 队列工作进程没有死信。失败作业消失。
- **不透明的后台工作。** 后台智能体运行不导出追踪。失败直到用户报告才可见。
- **跳过持久状态。** 任何超过 30 秒、且你负担不起重启的运行，都需要持久执行。

```figure
wb-runtime-shapes
```

## 动手做

`code/main.py` 是一个标准库的多形态演示：

- 请求-响应端点（普通函数）。
- 流式处理器（生成器）。
- 带 DLQ 的基于队列的工作进程。
- 事件触发注册表。
- cron 形态的调度器。

运行：

```bash
python3 code/main.py
```

输出：五条追踪，展示同一任务上每种形态的行为。同一套智能体逻辑，不同的外壳。持久执行（第六种形态）有意放在第 13 课，用 LangGraph 检查点来讲。

## 使用

- **请求-响应** 用于聊天式体验。
- **流式** 用于渐进回复。
- **持久** 用于长程任务。
- **队列** 用于批处理 / 异步 / 长时间运行。
- **事件** 用于智能体反应性。
- **Cron** 用于内务（记忆合并、评测、成本报告）。

## 交付

`outputs/skill-runtime-shape.md` 为一个任务选定运行时形态，并接上可观测性要求。

## 练习

1. 把你的第 01 课 ReAct 循环移植到你技术栈里的全部六种形态。哪种形态适合哪个产品表面？
2. 给基于队列的演示加一个 DLQ。模拟 10% 作业失败；露出 DLQ 大小。
3. 写一个由 cron 触发的评测智能体，每晚针对当天排名前 20 的追踪运行。
4. 实现带背压的流式：如果客户端慢，就暂停智能体。这如何与回合预算交互？
5. 阅读 Claude Managed Agents 文档。你何时会把一个自托管的长程智能体迁到托管？

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| 请求-响应 | “同步” | 用户等待；只适合短任务 |
| 流式 | “SSE / WS” | 渐进输出；更好的体验；延迟可按块观察 |
| 持久执行 | “从失败处恢复” | 状态做成检查点；从最后一步重启 |
| 基于队列 | “后台作业” | 生产者 / 工作进程池 / DLQ |
| 事件驱动 | “基于触发” | 智能体对外部事件作出反应 |
| DLQ | “死信队列” | 失败作业的停车场 |
| Claude Managed Agents | “托管框架” | Anthropic 托管的长时间异步，带缓存 + 压缩 |

## 延伸阅读

- [LangGraph 概览](https://docs.langchain.com/oss/python/langgraph/overview) — 持久执行细节
- [Claude Managed Agents 概览](https://platform.claude.com/docs/en/managed-agents/overview) — 托管的长时间异步
- [Anthropic，Introducing computer use](https://www.anthropic.com/news/3-5-models-and-computer-use) — “每个任务几十到几百步”
- [AutoGen v0.4（Microsoft Research）](https://www.microsoft.com/en-us/research/articles/autogen-v0-4-reimagining-the-foundation-of-agentic-ai-for-scale-extensibility-and-robustness/) — actor 模型的故障隔离
