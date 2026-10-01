# 并行工具调用，以及带工具的流式

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 三次彼此独立的天气查询如果串行，就是三次往返。并行跑它们，总时间会塌缩到最慢的那一次调用。如今每个前沿提供商都会在一轮里发出多次工具调用。收益是实在的；管道是微妙的。本课走完两半：并行扇出，以及流式参数的重组，重点是 id 关联陷阱。

**Type:** 动手做
**Languages:** Python（标准库，线程池 + 流式运行框架）
**Prerequisites:** 第 13 阶段 · 02（函数调用深入）
**Time:** ~75 分钟

## 学习目标

- 解释为什么存在 `parallel_tool_calls: true`，以及何时关掉它。
- 在并行扇出期间，把流式参数分块关联到正确的工具调用 id。
- 把不完整的 `arguments` 字符串重组成完整 JSON，而不要提前解析。
- 跑一个三城市天气基准，展示串行与并行的延迟差异。

## 问题

没有并行调用时，一个回答「Bengaluru、Tokyo 和 Zurich 天气如何」的智能体会这样做：

```
user -> LLM
LLM -> call get_weather(Bengaluru)
host -> run executor, reply with result
LLM -> call get_weather(Tokyo)
host -> run executor, reply with result
LLM -> call get_weather(Zurich)
host -> run executor, reply with result
LLM -> final text answer
```

三次 LLM 往返，每一次还要付执行器延迟。大约是理想墙钟时间的 4 倍。

有了并行调用：

```
user -> LLM
LLM -> call get_weather(Bengaluru); call get_weather(Tokyo); call get_weather(Zurich)
host -> run all three executors concurrently, reply with three results
LLM -> final text answer
```

一次 LLM 往返。执行器时间是三者的最大值，而不是总和。OpenAI、Anthropic 和 Gemini 上的生产基准显示，扇出工作负载的墙钟时间减少 60% 到 70%。

代价是关联复杂度。当三次调用乱序完成时，你的结果必须携带匹配的 `tool_call_id`，模型才能把它们对上。当结果以流式到达时，你必须先把部分参数片段组装成完整 JSON，然后才能执行。Gemini 3 增加唯一 id，部分是为了解决一个真实问题：对同一工具的两次并行调用此前无法区分。

## 概念

### 启用并行

- **OpenAI。** `parallel_tool_calls: true` 默认开启。设为 `false` 以强制串行。
- **Anthropic。** 通过 `disable_parallel_tool_use: false` 实现并行（Claude 3.5 及以上默认开启）。设为 `true` 则串行。
- **Gemini。** 始终具备并行能力；`tool_config.function_calling_config.mode = "AUTO"` 让模型决定。

当工具有顺序依赖（先 `create_file` 再 `write_file`）、当一次调用的输出要告知另一次的输入，或当速率限制器承受不了扇出时，关掉并行。

### id 关联

模型发出的每一次调用都有一个 `id`。宿主返回的每一个结果都必须带上同一个 id。没有它，结果就是含糊的。

- **OpenAI。** 每条 tool 角色消息上的 `tool_call_id`。
- **Anthropic。** 每个 `tool_result` 块上的 `tool_use_id`。
- **Gemini。** 每个 `functionResponse` 上的 `id`（Gemini 3 及以上；Gemini 2 按名称匹配，同名并行调用会因此坏掉）。

### 并发运行调用

宿主在各自的线程、协程或远程 worker 上运行每次调用的执行器。最简单的运行框架用线程池；生产环境用 asyncio 的 `asyncio.gather` 或结构化并发。完成顺序不可预测——id 才是标识符。

一个常见错误：按调用列表顺序回复结果，而不是按完成顺序。这通常能工作，因为模型只关心 `tool_call_id`，但如果某个结果被丢掉或重复，乱序提交会让调试更难。更宜按完成顺序回复，并带上显式 id。

### 流式工具调用

当模型流式输出时，`arguments` 一片一片到达。三次并行调用的三股分块流在线路上交错。你需要每个 id 一个累加器。

各提供商的形态：

- **OpenAI。** 每个分块是 `choices[0].delta.tool_calls[i].function.arguments`（部分字符串）。分块携带 `index`（在调用列表中的位置）。你按 index 累加，在 `id` 第一次出现时读它，并在 `finish_reason = "tool_calls"` 时解析 JSON。
- **Anthropic。** 流事件先是 `message_start`，然后每个块一个 `content_block_start`，类型为 `tool_use`（含 id、name、空的 input）。`content_block_delta` 事件携带 `input_json_delta` 分块。`content_block_stop` 关闭每个块。
- **Gemini。** `streamFunctionCallArguments`（Gemini 3 及以上）发出带 `functionCallId` 的分块，因此调用可以干净地交错。Gemini 3 之前，流式一次返回一个完整调用。

### 部分 JSON，以及过早解析陷阱

在 `arguments` 完整之前，你不能解析它。像 `{"city": "Beng` 这样的部分 JSON 不合法，会抛错。正确的闸门是提供商的调用结束信号：OpenAI 的 `finish_reason = "tool_calls"`、Anthropic 的 `content_block_stop`，或 Gemini 的流结束事件。只有那时才尝试 `json.loads`。更稳健的做法是用增量 JSON 解析器，在结构完成时产出事件；OpenAI 的流式指南建议这样做，以便 UX 能显示一个实时的「思考中」指示。花括号计数作为完整性测试不可靠（引号字符串里或转义内容里的花括号会造成假阳性），只应作为非正式的调试启发式。

### 乱序完成

```
call_A: fast API, returns first
call_B: slow API, returns second
call_C: median API, returns third
```

宿主的回复仍然必须引用这些 id：

```
[{role: "tool", tool_call_id: "call_A", content: ...},
 {role: "tool", tool_call_id: "call_B", content: ...},
 {role: "tool", tool_call_id: "call_C", content: ...}]
```

在 OpenAI 或 Anthropic 上，回复中的顺序不影响正确性。Gemini 接受任何顺序，只要 id 匹配。

### 基准：串行与并行

`code/main.py` 中的运行框架模拟三个执行器，延迟分别为 400、600 和 800 毫秒。串行总共跑 1800 毫秒。并行跑 max(400, 600, 800) = 800 毫秒。差值是常数，不是比例，因此工具越多，节省越大。

现实中的注意点：并行调用会压下游 API。对一个有速率限制的服务做 10 路扇出会失败。第 13 阶段 · 17 讲网关级背压；重试语义计划放在未来的阶段。

### 流式扇出的墙钟时间

如果模型本身在流式输出，你可以在某一次调用的参数一完整时就开始执行，而不必等所有调用都定稿。这是 OpenAI 记录了的优化，但并非所有 SDK 都暴露它。本课的运行框架会这样做：模拟流一旦产出一个完整的参数对象，宿主就启动那次调用。

```figure
tp-parallel-fanout
```

## 用起来

`code/main.py` 有两半。前一半用 `concurrent.futures.ThreadPoolExecutor` 串行和并行地运行三次模拟天气调用，并打印墙钟时间。后一半重放一份伪造的流式响应——三次并行调用的 `arguments` 分块交错在同一条流上——并用 `StreamAccumulator` 按 id 重组。没有 LLM，没有网络，只有重组逻辑。

要看的地方：

- 串行计时器打到 1.8 秒。同样的伪造延迟下，并行计时器打到 0.8 秒。
- 累加器按 id 缓冲来处理乱序到达的分块，并且只在每次调用的 JSON 完整时才解析。
- 执行器在某个 id 的参数一定稿就启动，而不是等所有流结束。

## 交付

本课产出 `outputs/skill-parallel-call-safety-check.md`。给定一份工具注册表，该技能审计哪些工具可以安全并行、哪些有顺序依赖、哪些会压垮下游速率限制——并返回一份修订后的注册表，带有逐工具的 `parallel_safe` 标志。

## 练习

1. 运行 `code/main.py` 并改变模拟延迟。确认并行与串行之比大约是 `max/sum`（真实运行会因线程调度、序列化和运行框架开销而略微偏离理想值）。在什么样的延迟分布下，并行不再重要？

2. 扩展累加器，处理「调用在流中途被取消」的情况：丢掉它的缓冲区，并发出一个 `cancelled` 事件。哪家提供商明确记录了这种情况？查看 Anthropic 的 `content_block_stop` 语义，以及 OpenAI 的 `finish_reason: "length"` 行为。

3. 用 `asyncio.gather` 替换线程池。两边都做基准。你应该会看到 async 上的小幅收益，因为上下文切换成本更低，但前提是执行器做的是真正的 I/O。

4. 选两个不应该并行的工具（例如先 `create_file` 再 `write_file`）。给注册表加一张 `ordering_dependency` 图，并据此给并行扇出设闸。这是依赖感知调度的最小机制，未来的智能体工程阶段会把它形式化。

5. 阅读 OpenAI 的并行函数调用一节，以及 Anthropic 的 `disable_parallel_tool_use` 文档。找出 Anthropic 建议关闭并行的那一种真实世界工具类型。（提示：对同一资源的有后果变更。）

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| 并行工具调用 | 「一轮里扇出」 | 模型在一条助手消息里发出多次工具调用 |
| `parallel_tool_calls` | 「OpenAI 的标志」 | 启用或禁用多次调用的发出 |
| `disable_parallel_tool_use` | 「Anthropic 的反义标志」 | 选择退出的标志；默认是启用并行 |
| 工具调用 id | 「关联句柄」 | 每次调用的标识符，结果消息必须原样回显 |
| 累加器 | 「流缓冲区」 | 按 id 的字符串缓冲区，用来装部分 `arguments` 分块 |
| 乱序完成 | 「最快的先到」 | 并行调用以不可预测的顺序完成；id 是胶水 |
| 依赖图 | 「顺序约束」 | 其输出要喂进其他工具输入的工具；不能并行 |
| 过早解析陷阱 | 「JSON.parse 炸了」 | 试图解析一条不完整的 `arguments` 字符串 |
| `streamFunctionCallArguments` | 「Gemini 3 的特性」 | 带每次调用唯一 id 的流式参数分块 |
| 按完成顺序回复 | 「不必等全部」 | 结果一到就回复，以 id 为键 |

## 延伸阅读

- [OpenAI — 并行函数调用](https://platform.openai.com/docs/guides/function-calling#parallel-function-calling) — 默认行为与选择退出标志
- [Anthropic — 并行工具使用](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use) — `disable_parallel_tool_use` 与结果批处理
- [Google — Gemini 函数调用的并行一节](https://ai.google.dev/gemini-api/docs/function-calling) — 自 Gemini 3 起以 id 关联的并行调用
- [OpenAI — 带工具的流式响应](https://platform.openai.com/docs/api-reference/responses-streaming) — OpenAI 流上分块参数的重组
- [Anthropic — 流式消息](https://docs.anthropic.com/en/api/messages-streaming) — 带 `input_json_delta` 的 `content_block_delta`
