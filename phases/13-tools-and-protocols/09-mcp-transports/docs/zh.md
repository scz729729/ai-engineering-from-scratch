# MCP 传输：stdio 与无状态 Streamable HTTP

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 传输承载 MCP（模型上下文协议）消息。它不补上缺失的协议状态。在 `2026-07-28` 里，本地 stdio 和远程 Streamable HTTP 都承载自描述请求。

**Type:** 理解
**Languages:** Python
**Prerequisites:** 第 13 阶段，第 07 与 08 课
**Time:** ~65 分钟

## 学习目标

- 为本地子进程选择 stdio，为网络服务选择 Streamable HTTP。
- 实现现代的单端点、仅 POST 的 Streamable HTTP 契约。
- 把 MCP 版本、方法和名称头对照 JSON-RPC 消息体做镜像与校验。
- 正确投递请求作用域的 SSE，以及长寿命的 `subscriptions/listen` 流。
- 迁移基于会话的和遗留的 HTTP+SSE 部署，而不把遗留行为呈现为现代行为。

## 问题

更早的 Streamable HTTP 修订把协议协商和连接、会话行为绑在一起。服务端可以铸造 `Mcp-Session-Id`，暴露一条独立的 GET 流，接受 DELETE 来终止会话，并用 `Last-Event-ID` 恢复 SSE。

MCP `2026-07-28` 把这些机制从现代线路上移除。每个请求都可以落到任何一个健康的 worker 上，因为它的协议版本和客户端能力都随请求体传递。HTTP 头镜像选定的字段，供路由和策略使用，但服务端在执行之前把那些头对照消息体做校验。

结果是更容易扩展，也更容易推理。它也意味着，一个把 2025 年传输当成当前来教的服务端，教的是错误的失败模型和安全模型。

## 概念

### stdio

stdio 绑定用于客户端启动的子进程：

- 客户端向 stdin 每行写一条 UTF-8 JSON-RPC 消息。
- 服务端向 stdout 每行写一条 UTF-8 JSON-RPC 消息。
- 服务端把诊断写到 stderr。
- 服务端在 stdin EOF 时立即退出。
- 每个现代请求都在 `params._meta` 里携带版本和客户端能力。

进程可以活过许多次调用，但它不是一个现代协议会话。如果它意外退出，在途请求就丢了。重启进程，重新发现，重新列出，重新打开订阅，并用新的请求 id 重试安全操作。

### 2026-07-28 中的 Streamable HTTP

现代服务端暴露一个 MCP 端点，例如 `/mcp`，它接受 POST。

每一个 JSON-RPC 请求或通知都是一次新的 HTTP POST。消息体包含一条 JSON-RPC 消息。客户端不向服务端发送 JSON-RPC 响应。

对于一个请求，服务端返回二者之一：

- `Content-Type: application/json`，带一条 JSON-RPC 响应；或
- `Content-Type: text/event-stream`，带与该请求相关的通知，随后是最终的 JSON-RPC 响应。

对于一个被接受的通知，服务端返回 `202 Accepted`，没有消息体。

客户端同时通告两种响应类型：

```http
Accept: application/json, text/event-stream
```

### 仅 POST 就是仅 POST

现代 Streamable HTTP 没有独立的 GET 流，也没有 DELETE 会话端点。

- `GET /mcp` 返回 `405 Method Not Allowed`。
- `DELETE /mcp` 返回 `405 Method Not Allowed`。
- `Mcp-Session-Id` 被忽略，并且永不铸造或回显。
- `Last-Event-ID` 被忽略，因为现代流不可恢复。

如果一条请求作用域的流在其最终响应之前断开，客户端就丢失了那个在途请求。当重试安全时，它可以发一个带新 JSON-RPC id 的新请求。它不得尝试恢复流。

### Origin 校验

服务端校验传入连接上的 `Origin`，以防止 DNS 重绑定。如果该头存在且未被显式允许，返回 `403 Forbidden`。非浏览器客户端可以省略 `Origin`，官方传输规则允许这样做。

本地服务端应当绑定到 `127.0.0.1`，而不是每一个接口。网络服务仍然需要在每个请求上做身份认证和授权。Origin 校验不是身份认证。

在规范化配置之后使用精确的 origin 匹配。像 `origin.startswith("https://trusted.example")` 这样的前缀检查是不安全的，因为它们可以接受攻击者控制的后缀。

### 必需的 HTTP 元数据头

每个现代 POST 请求都包含：

```http
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: notes_search
```

头规则：

- `MCP-Protocol-Version` 是必需的，并且必须等于 `params._meta.io.modelcontextprotocol/protocolVersion`。
- `Mcp-Method` 是必需的，并且必须等于 JSON-RPC 的 `method`。
- `Mcp-Name` 对 `tools/call`、`resources/read` 和 `prompts/get` 是必需的。
- `Mcp-Name` 等于 `params.name`，或对 `resources/read` 等于 `params.uri`。
- 头的值区分大小写，尽管头的名称不区分大小写。

不安全或非 ASCII 的 `Mcp-Name` 值使用确切的 UTF-8 Base64 哨兵：

```text
=?base64?{Base64EncodedValue}?=
```

服务端在把它与消息体比较之前解码该值。

缺失、畸形或不匹配的镜像头返回 HTTP `400` 和 JSON-RPC 代码 `-32020`。如果头和消息体一致地指向一个服务端不支持的版本，返回 HTTP `400` 和 `-32022`，并带上确切的错误数据，例如 `{"supported":["2026-07-28"],"requested":"2027-01-01"}`。

一个未知的现代方法返回 HTTP `404` 和 JSON-RPC `-32601`。JSON-RPC 消息体很重要，因为跨代客户端用它来区分现代错误和遗留端点未命中。

### 请求作用域的 SSE

服务端可以为一个长时间运行的请求选择 SSE：

```text
POST tools/call id=41
  <- notifications/progress related to id=41
  <- notifications/progress related to id=41
  <- JSON-RPC response id=41
stream closes
```

服务端不得在这条流上发送独立的 JSON-RPC 请求。Sampling、elicitation 和 roots 交互使用多轮往返请求（Multi Round-Trip Request）结果。关闭响应流会取消那个请求。

不要为了重放而添加 SSE 事件 id。`Last-Event-ID` 恢复不是现代修订的一部分。

### 长寿命变更使用 subscriptions/listen

变更通知使用客户端打开的请求，而不是独立的 GET：

```json
{
  "jsonrpc": "2.0",
  "id": "listen-1",
  "method": "subscriptions/listen",
  "params": {
    "notifications": {
      "toolsListChanged": true,
      "resourceSubscriptions": ["notes://note-1"]
    },
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

POST 响应是一条长寿命的 SSE 流。它的第一条协议消息是 `notifications/subscriptions/acknowledged`。确认、每一条变更通知，以及最终结果，都在 `_meta` 里携带 `io.modelcontextprotocol/subscriptionId`，等于监听请求的 id。服务端可以发出 SSE 注释作为保活。当流断开时，客户端用一个新的请求 id 重新发出 `subscriptions/listen`，并重新获取受影响的数据。

`resources/subscribe` 和 `resources/unsubscribe` 属于遗留代际。不要在现代连接上使用它们。

### 显式的应用状态

移除协议会话并不禁止带状态的工作流。服务端可以铸造一个不透明的状态句柄，并把它作为普通工具结果返回。客户端在后来的调用上把那个句柄作为显式参数传入。

把句柄绑定到已认证主体，使它们不可猜测，让它们过期，并授权每一次使用。这让状态在应用层可见，而不是藏在传输亲和性里。

由隐藏的副本状态造成的失败是机械的：

1. 请求 A 到达副本 1，并在该进程的内存里创建一个草稿。
2. 响应不返回草稿句柄，因为实现假定连接能标识这份草稿。
3. 请求 B 是一次全新的 POST，并到达副本 2。
4. 副本 2 有合法的协议元数据，但没有办法命名或加载这份草稿，于是工作流失败，或读到错误的本地对象。
5. 粘性路由看起来修好了症状，直到一次重启、发布、重新调度或故障转移把下一个请求挪走。

正确的边界有两部分。协议上下文留在每个请求里。持久的应用状态活在共享存储里，位于一个由服务端铸造、返回给客户端的句柄之下。下一次调用提供那个句柄，任何一个副本都加载同一条记录，授权把记录绑定到已认证主体和租户。副本内存可以缓存一条记录，但它不能是正确性所要求的唯一副本。

按寿命选择状态机制。请求局部变量可以服务一次调用。一次短的 MRTR 延续可以使用受完整性保护的 `requestState`。一份草稿或持久任务需要一个显式句柄，加上共享持久化、过期、并发控制和幂等。这些对象都不是 MCP 协议会话。

### HTTP 跨代兼容

一个同时支持现代和遗留服务端的客户端先尝试一次现代 POST。如果它收到 HTTP `400`、`404` 或 `405`，它检查消息体：

- 一个已识别的现代 JSON-RPC 错误证明服务端是现代的。纠正请求，或重试一个被通告的版本。不要降级。
- 空消息体或未识别的响应可能表示一个遗留的 HTTP+SSE 服务端。只有那时才尝试旧的 GET 端点，并期待它的遗留 `endpoint` 事件。

服务端可以在迁移期间同时支持两代：把现代元数据路由到现代的仅 POST 实现，并为旧客户端保留分开的遗留端点。绝不要把遗留的 GET、DELETE、会话 id 或重放行为描述成 `2026-07-28` 的一部分。

```figure
tp-transport-handshake
```

## 用起来

`code/main.py` 用 Python 标准库实现一个有限的、现代的 Streamable HTTP 服务端。它校验 Origin 和镜像头，忽略已移除的会话头，为普通调用返回 JSON，并演示一条有限的 `subscriptions/listen` SSE 流。

```bash
cd code
python3 main.py --probe
python3 -m unittest discover tests -v
```

探测检查：

- 无效的 Origin 被拒绝；
- 发现在没有会话 id 的情况下成功；
- `Mcp-Session-Id` 和 `Last-Event-ID` 被忽略；
- 头不匹配返回 `-32020`；
- 不受支持的版本返回 `-32022`，并带上确切的 `supported` 和 `requested` 数据；
- 一个被接受的、无 id 的通知返回 HTTP `202`，没有消息体；
- GET 和 DELETE 返回 `405`；
- `subscriptions/listen` 是一条 POST 响应流，其确认、通知和最终结果都携带它的订阅 id。

## 交付

本课交付 `outputs/skill-mcp-transport-migrator.md`。它移除现代协议会话，增加头与消息体的校验，用 `subscriptions/listen` 替换独立的 GET，并让任何遗留桥接保持明显分开。

## 练习

1. 从一次 POST 里去掉 `Mcp-Method`。确认 HTTP `400` 和错误 `-32020`。
2. 发送彼此匹配的头和消息体版本 `2027-01-01`。确认 HTTP `400`、错误 `-32022`，以及确切数据 `{"supported":["2026-07-28"],"requested":"2027-01-01"}`。
3. 为一个非 ASCII 资源 URI 发送 Base64 哨兵 `Mcp-Name`。确认解码后的值与 `params.uri` 比较。
4. 在有限的监听流给出最终响应之前打断它。用一个新的 JSON-RPC id 重新发出它，并重新获取工具。
5. 给 ping 工具增加一个显式的工作流句柄。把它绑定到一个授权主体，而不使用连接亲和性。

## 关键术语

| 术语 | 含义 |
|------|---------|
| stdio | 在客户端启动的子进程上、以换行分隔的 JSON-RPC |
| Streamable HTTP | 单一端点，每条现代消息都是一次新的 POST |
| 请求作用域的 SSE | 包含相关通知和最终响应的 POST 响应流 |
| `subscriptions/listen` | 用于已选择加入的变更通知的长寿命 POST 请求 |
| 头不匹配 | 镜像头与消息体不一致时的 HTTP `400` 和 JSON-RPC `-32020` |
| Origin 校验 | 针对传入连接的 DNS 重绑定防御，不是身份认证 |
| 显式状态句柄 | 作为普通参数传递的应用令牌，而不是隐藏的会话状态 |
| 遗留桥接 | 仅为兼容而保留的、分开的更早代际行为 |

## 延伸阅读

- [MCP Transport Overview](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports)
- [MCP stdio Transport](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio)
- [MCP Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [MCP Subscriptions](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions)
- [MCP 2026-07-28 Changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
