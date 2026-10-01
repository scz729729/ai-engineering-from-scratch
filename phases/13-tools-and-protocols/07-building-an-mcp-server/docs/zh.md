# 构建 MCP 服务端：无状态的 Python 与 TypeScript

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 现代 MCP（模型上下文协议）服务端不记住一次握手。它校验每一个请求上的元数据，运行一个处理程序，并返回一个带类型的结果。

**Type:** 动手做
**Languages:** Python、TypeScript
**Prerequisites:** 第 13 阶段，第 06 课
**Time:** ~85 分钟

## 学习目标

- 为 MCP `2026-07-28` 实现强制的 `server/discover`。
- 在每一个请求上校验协议版本和客户端能力。
- 以确定性的列表顺序暴露工具、资源和提示模板。
- 在正确的结果上返回 `resultType`、服务端身份和缓存提示。
- 在 Python 和 TypeScript 中，通过换行分隔的 stdio 提供同一份无状态契约。

## 问题

一个在第一条消息之后就存储客户端能力的服务端，容易构建，难以运营。同一进程可能服务顺序到来的多个客户端。一个远程请求可能落到另一个 worker 上。一份过期的能力声明会让行为越过授权边界泄漏。

MCP `2026-07-28` 通过让每个请求自描述，解决了这个问题的协议部分。你的应用仍然可以保留持久的笔记、任务，或显式的状态句柄。它不能保留的，是会改变后来请求如何被解码的隐藏协议状态。

本课把一个笔记服务端构建两遍。Python 和 TypeScript 版本对协议核心都只用各自的标准库。两者暴露相同的方法，并强制同一份线上契约。

## 概念

### 现代分派循环

```text
read one JSON-RPC line
parse the envelope
if it is a notification, do not respond
validate params._meta for this request
route by method
wrap success with resultType and serverInfo
write one JSON-RPC response line
forget request-scoped metadata
```

三条 stdio 规则仍然重要：

- 只把 JSON-RPC 消息写到 stdout。诊断送到 stderr。
- 用换行分隔消息，并刷新每一条响应。
- 当 stdin 到达 EOF 时立即退出。

进程寿命是传输寿命。它不是一个现代 MCP 会话。

### 请求校验

每个请求都必须有：

```json
{
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "io.modelcontextprotocol/clientInfo": {
        "name": "notes-client",
        "version": "1.0.0"
      }
    }
  }
}
```

前两个字段是必需的。`clientInfo` 是推荐的。校验一个出现了的身份形态，但不要把它当作身份认证。

如果版本不受支持，返回代码 `-32022`，并带上 `requested` 和 `supported`。缺失的请求元数据是无效参数，代码 `-32602`。绝不要用上一次调用去填补缺失字段。

### 强制发现

现代服务端必须实现 `server/discover`。一个完整的发现结果包括受支持的现代版本、能力、可选说明、缓存提示，以及结果 `_meta` 中的服务端身份：

```json
{
  "resultType": "complete",
  "supportedVersions": ["2026-07-28"],
  "capabilities": {
    "tools": {"listChanged": false},
    "resources": {"listChanged": false, "subscribe": false},
    "prompts": {"listChanged": false}
  },
  "ttlMs": 3600000,
  "cacheScope": "public",
  "_meta": {
    "io.modelcontextprotocol/serverInfo": {
      "name": "notes-server",
      "version": "2.0.0"
    }
  }
}
```

发现并不解锁服务端。客户端可以不调用发现就调用 `tools/list`，因为 `tools/list` 已经携带同样的请求元数据。

### Tools

`tools/list` 返回一份确定性的工具描述符列表。稳定顺序改善响应缓存，并保持模型上下文稳定。结果还要求 `ttlMs` 和 `cacheScope`。

`tools/call` 返回内容块和 `isError`。当协议信封或方法参数无效时，使用 JSON-RPC 错误。当一次合法的工具调用跑起来了、但工具本身失败时，使用 `isError: true`。

工具注解仍然是提示，不是强制：

- `readOnlyHint`
- `destructiveHint`
- `idempotentHint`
- `openWorldHint`

宿主应当用它们来做确认和呈现。服务端仍然必须强制真正的授权。

### Resources

`resources/list` 返回稳定的 URI 描述符。`resources/read` 返回带类型的内容。在 `2026-07-28` 里两者都可缓存，因此两者都包含 `ttlMs` 和 `cacheScope`。

对用户专属的笔记数据使用 `cacheScope: "private"`。共享缓存不得跨授权上下文复用一份私有响应。

现代的变更投递不使用 `resources/subscribe`。客户端打开 `subscriptions/listen`，并请求 `resourceSubscriptions` 或列表变更类别。第 10 课构建那个流程。

### Prompts

`prompts/list` 可缓存且确定。`prompts/get` 用参数渲染一个具名提示模板。渲染出的提示结果是完整的，但它不属于那些要求缓存提示的可缓存列表或读取结果。

### 每个成功结果都带类型

这些例子对每一次成功都用一个包装器：

```python
def complete(payload):
    return {
        "resultType": "complete",
        **payload,
        "_meta": {SERVER_INFO_KEY: SERVER_INFO},
    }
```

列表、读取和发现处理程序再加上 `ttlMs` 和 `cacheScope`。把这个包装器集中起来，防止某个处理程序静默漏掉现代结果字段。

### 没有服务端发起的请求

现代服务端可以发送与某个客户端请求相关的通知，或在客户端打开的 `subscriptions/listen` 流上发送通知。它不得发送自己的 JSON-RPC 请求。

当处理程序需要 sampling、elicitation 或 roots 输入时，它返回一个 `input_required` 结果。客户端完成内嵌的输入请求，并用一个新的请求 id 重试原来的方法。第 11 课讲那种多轮往返请求（Multi Round-Trip Request）模式。

### 显式的遗留兼容

一个跨代服务端也可以在一条清晰分开的遗留分支上实现 `2025-11-25` 握手。当必需的现代 `_meta` 字段存在时，它选择现代行为；当它收到 `initialize` 时，选择遗留行为。

不要把一个 `2026-07-28` 请求送进遗留握手路径。不要把现代的 `resultType` 字段盖到遗留初始化结果上。本课的代码有意只做现代，以便它的不变量保持可见。

```figure
t3-dispatch-loop
```

## 用起来

运行 Python 服务端的有限演示和测试：

```bash
cd code
python3 main.py --demo
python3 -m unittest discover tests -v
```

用 TypeScript 运行器运行 TypeScript 移植：

```bash
npx tsx main.ts --demo
```

演示发送 `server/discover`，列出每个原语，调用工具，并展示一个不受支持版本的错误。每个现代请求都重复元数据。每次成功都包含服务端身份。

## 交付

本课交付 `outputs/skill-mcp-server-scaffolder.md`。它产出一份现代服务端计划，带有发现契约、逐请求校验、确定性的可缓存列表，以及一个可选的隔离遗留适配器。

## 练习

1. 从一个请求里去掉能力，并证明服务端不会复用上一个请求的声明。
2. 反转 `TOOLS`、`PROMPTS` 和笔记的插入顺序。确认所有列表结果仍然稳定。
3. 增加一个有破坏性的 `notes_delete` 工具，并在执行器内部要求一次授权检查。把 `destructiveHint` 只当作 UX 提示。
4. 增加带 `ttlMs`、`cacheScope` 和确定性顺序的 `resources/templates/list`。
5. 为 `2025-11-25` 构建一个分开的遗留适配器。增加测试，证明现代请求永远不会进入它。

## 关键术语

| 术语 | 含义 |
|------|---------|
| 无状态服务端 | 根据每个请求自己的元数据处理它，没有协议会话记忆 |
| `server/discover` | 通告版本和能力的强制现代方法 |
| 完整结果 | 带 `resultType: "complete"` 的成功现代结果 |
| 可缓存结果 | 带 `ttlMs` 和 `cacheScope` 的发现、列表或资源读取结果 |
| 确定性列表 | 同一份逻辑注册表产生相同的条目顺序 |
| 服务端身份 | 结果 `_meta` 中推荐的 `io.modelcontextprotocol/serverInfo` |
| 工具错误 | 一次合法工具调用返回带 `isError: true` 的内容 |
| 协议错误 | 通过 `error` 返回的无效 JSON-RPC 或 MCP 请求 |

## 延伸阅读

- [MCP 规范 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/)
- [MCP Server Discovery](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
- [MCP Tools](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)
- [MCP Resources](https://modelcontextprotocol.io/specification/2026-07-28/server/resources)
- [MCP Prompts](https://modelcontextprotocol.io/specification/2026-07-28/server/prompts)
- [MCP stdio Transport](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio)
