# MCP 基础：无状态请求与 JSON-RPC

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 现代 MCP（模型上下文协议）没有握手，也没有协议会话。每个请求都必须自带足够的元数据，以便被独立理解、授权、路由和重试。

**Type:** 理解
**Languages:** Python
**Prerequisites:** 第 13 阶段，第 01 至 05 课
**Time:** ~55 分钟

## 学习目标

- 把 MCP 的服务端原语和它的客户端侧特性区分开。
- 为 MCP `2026-07-28` 构建合法的 JSON-RPC 2.0 请求与响应。
- 把协议版本、客户端能力和客户端身份附到每一个请求上。
- 使用 `server/discover`，并在没有握手的情况下处理 `UnsupportedProtocolVersionError`。
- 追踪一个独立请求，从校验一直到完整结果。

## 问题

一个 MCP 服务端可以在同一进程或同一个 HTTP worker 上，连续收到来自不同客户端、具备不同能力的两个请求。如果服务端记住上一个请求声明了什么，它就可能套用错误的权限，或返回错误的线上形态。

MCP `2026-07-28` 去掉了这种歧义。协议核心是无状态的。服务端必须根据当前请求来决定如何处理当前请求，而不是根据连接历史。

这改变了心智模型。旧顺序是先连接，再握手，然后才是操作。现代顺序更简单：

1. 客户端发送一个自描述请求。
2. 服务端校验该请求的版本和能力。
3. 服务端处理方法。
4. 服务端返回一个带类型的结果，或一个 JSON-RPC 错误。

下一个请求从头重复同一过程。

## 概念

### 服务端原语

MCP 服务端暴露三个主要原语：

1. **Tools** 是由模型控制的动作，用 `tools/list` 发现，用 `tools/call` 调用。
2. **Resources** 是以 URI 寻址的数据，用 `resources/list` 发现，用 `resources/read` 读取。
3. **Prompts** 是可复用模板，用 `prompts/list` 发现，用 `prompts/get` 渲染。

Roots、Sampling 和 Logging 仍留在 `2026-07-28` 的 schema 里以保持兼容，但它们已弃用。新实现应当用显式的工具或资源输入来表示 roots，用直接的模型提供商 API 来做 sampling，用 stderr 或 OpenTelemetry 来做日志。elicitation 仍然可通过多轮往返请求（Multi Round-Trip Requests）使用：服务端返回一个输入请求，客户端重试原来的操作。现代服务端从不发起一个独立的 JSON-RPC 请求。

### JSON-RPC 信封

MCP 使用 JSON-RPC 2.0：

- 请求：`{jsonrpc, id, method, params}`
- 响应：`{jsonrpc, id, result}` 或 `{jsonrpc, id, error}`
- 通知：`{jsonrpc, method, params}`，没有 `id`

请求的 `id` 关联一条响应。它并不创建协议会话。

### 必需的请求元数据

每个现代请求都在 `params` 里携带一个 `_meta` 对象：

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "tools/list",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "io.modelcontextprotocol/clientInfo": {
        "name": "course-client",
        "version": "1.0.0"
      }
    }
  }
}
```

协议版本和客户端能力是必需的。客户端身份是推荐的。它是自报的展示与调试数据，不是安全凭据。

服务端不得从更早的请求、一个 stdio 进程、一条 HTTP 连接，或单独一个传输头来推断这些值中的任何一个。

### 完整结果与服务端身份

每个成功的现代结果都包含 `resultType`。普通的最终结果使用 `"complete"`。服务端还应当在结果元数据里标识自己：

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "resultType": "complete",
    "tools": [],
    "ttlMs": 30000,
    "cacheScope": "public",
    "_meta": {
      "io.modelcontextprotocol/serverInfo": {
        "name": "notes-server",
        "version": "1.0.0"
      }
    }
  }
}
```

`tools/list`、`resources/list`、`prompts/list`、`resources/templates/list`、`resources/read` 和 `server/discover` 是可缓存结果。它们包含 `ttlMs` 和 `cacheScope`。一个安全默认是 `ttlMs: 0` 和 `cacheScope: "private"`。列表项应当有确定性顺序，使等价响应产生稳定的缓存键和稳定的模型上下文。

### 没有握手的发现

每个现代服务端都必须实现 `server/discover`。客户端可以在调用其他方法之前调用它，以取回：

- `supportedVersions`
- 服务端 `capabilities`
- 可选的用法 `instructions`
- 结果 `_meta` 中的服务端身份
- 缓存提示

发现有用，但它不是闸门。客户端可以先发 `tools/list`，因为那个请求已经携带了自己的协议版本和能力。

如果请求的版本不受支持，服务端返回 JSON-RPC 代码 `-32022`，并带上：

```json
{
  "requested": "2027-01-01",
  "supported": ["2026-07-28"]
}
```

客户端选择一个双方都支持的现代版本，并用一个新的 JSON-RPC 请求 id 重试。

### 一个请求的生命周期

按这个顺序追踪一个现代请求：

1. 解析一个 JSON-RPC 信封。
2. 确认 `jsonrpc` 是 `"2.0"`，存在 `id`，`method` 是字符串，`params` 是对象。
3. 要求 `params._meta` 里有版本字符串和能力对象；畸形或缺失的元数据是 `-32602`。
4. 在 HTTP 边界，把版本、方法和适用的名称头与消息体比较。不匹配就是 `-32020`，即使两个版本值中有一个不受支持也一样。
5. 在相等已经确立之后，用 `-32022` 拒绝一个匹配但不受支持的版本。
6. 检查必需的能力，然后按 `method` 路由，并校验该方法专有的参数。
7. 在其处理程序运行之前，对这个具体操作做身份认证和授权。
8. 返回一个带服务端身份的完整结果。
9. 忘掉请求作用域内的协议元数据。

这个顺序防止两个组件去解释不同的调用。网关不得在源端执行 `params.name: notes.delete` 的同时，去授权 `Mcp-Name: notes.read`。它也让畸形输入、头混淆、版本协商、能力失败、授权失败和处理程序失败成为彼此不同的证据。

关闭 stdin 或一条 HTTP 响应会结束传输活动。它并不终止协议会话，因为现代 MCP 没有协议会话。

### 显式的遗留兼容

直到 `2025-11-25` 的版本使用 `initialize`、`notifications/initialized`、连接作用域的能力，以及在更早的 Streamable HTTP 上可选的协议会话。当一个跨代客户端与旧服务端交谈时，那种行为仍然相关。

把各代分开。现代请求由必需的逐请求元数据来识别。遗留连接只通过文档化的回退路径来选择。不要把 `initialize` 当作 `2026-07-28` 服务端的默认。

因此「无状态」有随代而异的含义。在 `2026-07-28` 里，它是一条协议不变量：每个普通请求都可以独立解释，并且不存在 MCP 会话。在直到 `2025-11-25` 的版本里，初始化和协商出的能力属于一条连接，因此兼容适配器可以保留那份遗留连接状态。一个跨代实现不是一台过于宽松的状态机。它是一个无状态的现代核心，旁边放一个隔离的遗留适配器，并且在任一解析器运行之前有一个显式的选择决定。

两种含义都不禁止持久的应用状态。工作流、任务或草稿可以活在共享存储里的一个不透明句柄后面。客户端把那个句柄作为普通输入发送，每个副本都对其使用做身份认证和授权。协议上下文不得泄漏进那个存储，去代替被移除的会话。

```figure
mcp-tool-call
```

## 用起来

`code/main.py` 在没有框架的情况下构建、校验、追踪并分派现代 MCP 消息。运行：

```bash
python3 code/main.py
python3 -m unittest discover code/tests -v
```

在输出里留意三条不变量：

- 每个请求都重复自己的 `_meta` 字段。
- 每个成功结果都是 `resultType: "complete"`，并包含服务端身份。
- 列表结果按确定性顺序排列，并带有显式的缓存提示。

## 交付

本课交付 `outputs/skill-mcp-handshake-tracer.md`。历史文件名保持稳定，但该产物现在是一个无状态请求追踪器。它独立审计每条消息，并且只在遗留握手流量真正出现时才给它贴上标签。

## 练习

1. 把一个请求的协议版本改成 `2027-01-01`。确认错误代码是 `-32022`，并且数据通告了受支持的版本。
2. 从第二个请求里去掉 `io.modelcontextprotocol/clientCapabilities`。确认服务端不会复用第一个请求的能力。
3. 把内存中的工具注册表反转。确认 `tools/list` 仍然返回相同的确定性顺序。
4. 把 `cacheScope` 从 `public` 改成 `private`。解释在每种情况下，哪些授权上下文可以复用该响应。
5. 增加一个可选的省略 `clientInfo` 的测试。请求应当仍然合法，因为客户端身份是推荐的，不是必需的。

## 关键术语

| 术语 | 含义 |
|------|---------|
| 无状态协议 | 每个请求都提供解释自身所需的元数据 |
| 请求元数据 | `params._meta` 中的版本、客户端能力，以及推荐的客户端身份 |
| `server/discover` | 用于版本、能力、说明和身份的强制服务端方法 |
| `resultType` | 每个成功的现代结果上的判别字段 |
| 可缓存结果 | 包含必需的 `ttlMs` 和 `cacheScope` 提示的结果 |
| 协议代际 | 现代的逐请求元数据，或遗留的连接作用域初始化 |
| 传输寿命 | 进程、连接或响应流的寿命，不是协议会话状态 |
| `-32022` | 不受支持的协议版本错误，带有请求的版本和受支持的版本 |

## 延伸阅读

- [MCP Architecture](https://modelcontextprotocol.io/specification/2026-07-28/architecture)
- [MCP Base Protocol](https://modelcontextprotocol.io/specification/2026-07-28/basic)
- [MCP Server Discovery](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
- [MCP 2026-07-28 Changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
