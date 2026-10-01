# 函数调用深入——OpenAI、Anthropic、Gemini

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 三家前沿提供商在 2024 年收敛到同一套工具调用循环，然后在其他一切事情上分道扬镳。OpenAI 使用 `tools` 和 `tool_calls`。Anthropic 使用 `tool_use` 和 `tool_result` 块。Gemini 使用 `functionDeclarations` 和唯一 id 关联。本课把三者并排对照，免得在一个提供商上能交付的代码，移植时直接坏掉。

**Type:** 动手做
**Languages:** Python（标准库，schema 转换器）
**Prerequisites:** 第 13 阶段 · 01（工具接口）
**Time:** ~75 分钟

## 学习目标

- 说出 OpenAI、Anthropic 与 Gemini 函数调用载荷在三个形态上的差异（声明、调用、结果）。
- 把一份工具声明翻译成全部三种提供商格式，并预测严格模式约束会在哪里不同。
- 在每个提供商上使用 `tool_choice`，以强制、禁止或自动选择工具调用。
- 知道各提供商的硬限制（工具数量、schema 深度、参数长度），以及超出限制时各自发出的错误特征。

## 问题

函数调用请求的形态因提供商而异。下面是 2026 年生产技术栈里的三个具体例子：

**OpenAI Chat Completions / Responses API。** 你传入 `tools: [{type: "function", function: {name, description, parameters, strict}}]`。模型的响应包含 `choices[0].message.tool_calls: [{id, type: "function", function: {name, arguments}}]`，其中 `arguments` 是一个你必须解析的 JSON 字符串。严格模式（`strict: true`）通过受约束解码强制 schema 合规。

**Anthropic Messages API。** 你传入 `tools: [{name, description, input_schema}]`。响应以 `content: [{type: "text"}, {type: "tool_use", id, name, input}]` 返回。`input` 已经解析好（是对象，不是字符串）。你用一条新的 `user` 消息回复，其中包含一个 `{type: "tool_result", tool_use_id, content}` 块。

**Google Gemini API。** 你传入 `tools: [{functionDeclarations: [{name, description, parameters}]}]`（嵌套在 `functionDeclarations` 之下）。响应以 `candidates[0].content.parts: [{functionCall: {name, args, id}}]` 到达，其中 `id` 在 Gemini 3 及更高版本中是唯一的，用于并行调用关联。你用 `{functionResponse: {name, id, response}}` 回复。

同一套循环。不同的字段名、不同的嵌套、不同的字符串与对象约定、不同的关联机制。一个在 OpenAI 上写天气智能体的团队，光是管道就要花两天移植到 Anthropic，再花一天移植到 Gemini。

本课构建一个转换器，把三种格式统一成一份规范工具声明，并在边缘路由。第 13 阶段 · 17 把同一模式推广成一个 LLM 网关。

## 概念

### 共同结构

每个提供商都需要五样东西：

1. **工具列表。** 每个工具的名称、描述和输入 schema。
2. **工具选择。** 强制某个具体工具、禁止工具，或让模型自己决定。
3. **调用发出。** 指名工具和参数的结构化输出。
4. **调用 id。** 把响应关联到正确的那次调用（对并行很重要）。
5. **结果注入。** 一条消息或一个块，把结果绑回那次调用。

### 逐字段的形态差异

| 方面 | OpenAI | Anthropic | Gemini |
|--------|--------|-----------|--------|
| 声明信封 | `{type: "function", function: {...}}` | `{name, description, input_schema}` | `{functionDeclarations: [{...}]}` |
| schema 字段 | `parameters` | `input_schema` | `parameters` |
| 响应容器 | 助手消息上的 `tool_calls[]` | 类型为 `tool_use` 的 `content[]` | 类型为 `functionCall` 的 `parts[]` |
| 参数类型 | 字符串化的 JSON | 已解析的对象 | 已解析的对象 |
| id 格式 | `call_...`（由 OpenAI 生成） | `toolu_...`（Anthropic） | UUID（Gemini 3+） |
| 结果块 | 角色 `tool`，`tool_call_id` | 带 `tool_result`、`tool_use_id` 的 `user` | 带匹配 `id` 的 `functionResponse` |
| 强制指定工具 | `tool_choice: {type: "function", function: {name}}` | `tool_choice: {type: "tool", name}` | `tool_config: {function_calling_config: {mode: "ANY"}}` |
| 禁止工具 | `tool_choice: "none"` | `tool_choice: {type: "none"}` | `mode: "NONE"` |
| 严格 schema | `strict: true` | schema 即 schema（始终强制） | 请求级的 `responseSchema` |

### 你真正会撞上的限制

- **OpenAI。** 每个请求 128 个工具。schema 深度为 5。参数字符串 <= 8192 字节。严格模式要求没有 `$ref`，没有带重叠的 `oneOf`/`anyOf`/`allOf`，每个属性都列在 `required` 中。
- **Anthropic。** 每个请求 64 个工具。schema 深度实际上无界，但实用上限是 10。没有严格模式标志；schema 是一份契约，模型倾向于遵守。
- **Gemini。** 每个请求 64 个函数。schema 类型是 OpenAPI 3.0 子集（与 JSON Schema 2020-12 略有分歧）。自 Gemini 3 起，并行调用使用唯一 id。

### `tool_choice` 行为

三种人人都支持的模式，名字各不相同。

- **Auto。** 模型选择工具或文本。默认。
- **Required / Any。** 模型必须至少调用一个工具。
- **None。** 模型不得调用工具。

再加上每个提供商独有的一种模式：

- **OpenAI。** 按名称强制某个具体工具。
- **Anthropic。** 按名称强制某个具体工具；`disable_parallel_tool_use` 标志把单次与多次分开。
- **Gemini。** `mode: "VALIDATED"` 让每个响应都经过 schema 校验器，不论模型意图如何。

### 并行调用

OpenAI 的 `parallel_tool_calls: true`（默认）在一条助手消息里发出多次调用。你把它们全部跑完，并用一条成批的 tool 角色消息回复，其中每个 `tool_call_id` 对应一条。Anthropic 历史上是单次调用；`disable_parallel_tool_use: false`（自 Claude 3.5 起为默认）启用多次。Gemini 2 允许并行调用，但不给出稳定 id；Gemini 3 加上 UUID，使乱序响应能干净地关联。

### 流式

三者都支持流式工具调用。线上格式不同：

- **OpenAI。** `tool_calls[i].function.arguments` 的增量分块陆续到达。你一直累加，直到 `finish_reason: "tool_calls"`。
- **Anthropic。** 块开始 / 块增量 / 块停止事件。`input_json_delta` 分块携带部分参数。
- **Gemini。** `streamFunctionCallArguments`（Gemini 3 新增）发出带 `functionCallId` 的分块，使多个并行调用可以交错。

第 13 阶段 · 03 深入并行与流式重组。本课聚焦声明形态和单次调用形态。

### 错误与修复

无效参数错误看起来也不一样。

- **OpenAI（非严格）。** 模型返回 `arguments: "{bad json}"`，你的 JSON 解析失败，你注入一条错误消息并重新调用。
- **OpenAI（严格）。** 校验发生在解码期间；无效 JSON 不可能出现，但可能出现 `refusal`。
- **Anthropic。** `input` 可能包含意料之外的字段；schema 是建议性的。在服务端校验。
- **Gemini。** OpenAPI 3.0 的怪癖：对象字段上的 `enum` 会被静默忽略；你得自己校验。

### 转换器模式

你代码里的一份规范工具声明看起来像这样（形态由你选定）：

```python
Tool(
    name="get_weather",
    description="Use when ...",
    input_schema={"type": "object", "properties": {...}, "required": [...]},
    strict=True,
)
```

三个很小的函数把它翻译成三种提供商形态。`code/main.py` 里的运行框架正是这样做的，然后把一次伪造的工具调用分别走一遍每个提供商的响应形态。不需要网络——本课教的是形态，不是 HTTP。

生产团队把这个转换器包进 `AbstractToolset`（Pydantic AI）、`UniversalToolNode`（LangGraph）或 `BaseTool`（LlamaIndex）。第 13 阶段 · 17 交付一个网关，在这三者中的任意一个前面暴露 OpenAI 形态的 API。

```figure
function-call-args
```

## 用起来

`code/main.py` 定义一个规范的 `Tool` 数据类，以及三个转换器，分别发出 OpenAI、Anthropic 和 Gemini 的声明 JSON。然后它把每种形态下手工构造的提供商响应解析成同一个规范调用对象，表明表层之下语义是相同的。运行它，并排对照三份声明。

要看的地方：

- 三份声明块的差别只在信封和字段名。
- 三份响应块的差别在于调用住在哪里（顶层 `tool_calls`、`content[]` 块、`parts[]` 条目）。
- 一个 `canonical_call()` 函数从全部三种响应形态中抽出 `{id, name, args}`。

## 交付

本课产出 `outputs/skill-provider-portability-audit.md`。给定针对一个提供商的函数调用集成，该技能产出一份可移植性审计：它依赖哪些提供商限制，哪些字段需要改名，以及移植到另外两个提供商时什么会坏。

## 练习

1. 运行 `code/main.py`，验证三份提供商声明 JSON 都序列化了同一个底层 `Tool` 对象。修改规范工具，增加一个 enum 参数，并确认只有 Gemini 转换器需要处理 OpenAPI 怪癖。

2. 为每个提供商增加一个 `ListToolsResponse` 解析器，提取模型在 `list_tools` 或发现调用之后返回的工具列表。OpenAI 原生没有这个；记下这种不对称。

3. 实现 `tool_choice` 转换：把规范的 `ToolChoice(mode="force", tool_name="x")` 映射成全部三种提供商形态。然后再映射 `mode="any"` 和 `mode="none"`。对照本课的差异表。

4. 选三家提供商中的一家，从头到尾阅读它的函数调用指南。找出其 schema 规范里另外两家不支持的一个字段。候选：OpenAI 的 `strict`、Anthropic 的 `disable_parallel_tool_use`、Gemini 的 `function_calling_config.allowed_function_names`。

5. 写一个测试向量：一次参数违反已声明 schema 的工具调用。把它分别跑过每个提供商的校验器（第 01 课里的标准库校验器可以当代理），并记录哪些错误会触发。写下在生产环境里你会用哪家提供商来追求严格性。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| 函数调用 | 「工具使用」 | 提供商层面的 API，用于发出结构化工具调用 |
| 工具声明 | 「工具规格」 | 名称 + 描述 + JSON Schema 输入载荷 |
| `tool_choice` | 「强制 / 禁止」 | auto / required / none / 指定名称这几种模式 |
| 严格模式 | 「schema 强制」 | OpenAI 的标志，把解码约束到与 schema 匹配 |
| `tool_use` 块 | 「Anthropic 的调用形态」 | 内联内容块，含 id、name、input |
| `functionCall` 部分 | 「Gemini 的调用形态」 | 一条 `parts[]` 条目，含 name、args 和 id |
| 参数即字符串 | 「字符串化的 JSON」 | OpenAI 把参数作为 JSON 字符串返回，而不是对象 |
| 并行工具调用 | 「一轮里扇出」 | 一条助手消息里的多次工具调用 |
| 拒绝 | 「模型拒绝」 | 仅严格模式才有的拒绝块，用来代替一次调用 |
| OpenAPI 3.0 子集 | 「Gemini 的 schema 怪癖」 | Gemini 使用一种类似 JSON Schema 的方言，有细小差异 |

## 延伸阅读

- [OpenAI — 函数调用指南](https://platform.openai.com/docs/guides/function-calling) — 权威参考，包括严格模式和并行调用
- [Anthropic — 工具使用概述](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview) — `tool_use` 与 `tool_result` 块的语义
- [Google — Gemini 函数调用](https://ai.google.dev/gemini-api/docs/function-calling) — 并行调用、唯一 id，以及 OpenAPI 子集
- [Vertex AI — 函数调用参考](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/multimodal/function-calling) — Gemini 的企业表面
- [OpenAI — 结构化输出](https://platform.openai.com/docs/guides/structured-outputs) — 严格模式 schema 强制的细节
