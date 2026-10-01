# 构建 MCP 客户端：发现、路由，以及跨代回退

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 现代 MCP（模型上下文协议）客户端在每个请求上重复自己的契约。它最难的兼容决定，是知道一个旧服务端何时真的旧，以及一个现代服务端何时是在报告一个可纠正的错误。

**Type:** 动手做
**Languages:** Python
**Prerequisites:** 第 13 阶段，第 07 课
**Time:** ~85 分钟

## 学习目标

- 用当前元数据构建每一个 MCP `2026-07-28` 请求。
- 用 `server/discover` 探测 stdio 服务端，并选择一个双方都支持的版本。
- 只对显式列入允许列表的对端，授权一次有界的遗留探测。
- 只有在校验了一个受支持修订版的正向 `initialize` 结果之后，才接受遗留代际。
- 合并确定性的工具列表，而不静默覆盖碰撞。
- 把调用路由到拥有每个工具的对端，而不发明协议会话。

## 问题

一个智能体宿主通常与不止一个 MCP 服务端交谈。它必须发现每个服务端、合并工具目录、解决重复名称、路由调用，并从传输失败中恢复。

`2026-07-28` 修订让稳态更简单，因为每个请求都是自包含的。兼容性让启动更微妙。客户端可能遇到：

- 一个支持首选版本的现代服务端；
- 一个返回已识别的版本或头错误的现代服务端；
- 一个从未听说过 `server/discover` 的遗留服务端；
- 一个一直沉默、直到收到 `initialize` 的遗留服务端。

把每一次探测错误都当成遗留，是危险的。一个畸形的现代请求、一个过载的服务端、一个已死的进程，以及一个旧服务端，都可以产生同样的超时或连接关闭。那些信号是含糊的。客户端必须把显式的操作者意图和正向的协议证据结合起来，然后才选择遗留代际。

## 概念

### 一个对端，不是一个协议会话

为每个服务端进程或端点保留一条传输对端记录：

- 传输句柄或发送函数；
- 选定的协议代际和版本；
- 最近一次发现的服务端能力；
- 最近一次确定性的工具列表；
- 用于关联的待处理请求 id；
- 传输健康状况。

这是客户端的簿记。它不是协议会话状态。在现代 MCP 上，服务端仍然在每个请求上收到当前版本和能力。

### 从零构建每一个现代请求

```python
def modern_request(request_id, method, params, version, capabilities):
    return {
        "jsonrpc": "2.0",
        "id": request_id,
        "method": method,
        "params": {
            **params,
            "_meta": {
                "io.modelcontextprotocol/protocolVersion": version,
                "io.modelcontextprotocol/clientCapabilities": capabilities,
                "io.modelcontextprotocol/clientInfo": CLIENT_INFO,
            },
        },
    }
```

不要把元数据附到连接对象上一次，就假定它到达了线路。盖章并检查最终序列化的请求。

### 现代发现

`server/discover` 返回受支持的版本、服务端能力、说明、缓存提示，以及推荐的服务端身份。客户端选择双方都支持的最高现代版本。

对只做现代的客户端，发现是可选的，但在 stdio 上推荐这样做。有些遗留服务端会在初始化之前接受一次操作，因此先发 `tools/list` 可能产生一次含糊的成功。`server/discover` 制造一条干净的代际边界。

### stdio 兼容探测

一个跨代 stdio 客户端在任何其他请求之前，用它首选的现代元数据发送 `server/discover`。有三类结果：

1. **DiscoverResult。** 服务端是现代的。选择一个双方都支持的版本，并继续使用逐请求元数据。
2. **已识别的现代错误。** 服务端是现代的。对于 `-32022`，从 `data.supported` 里选择，并用一个新的请求 id 重试。对于头或能力错误，纠正请求。不要发送 `initialize`。
3. **含糊信号。** 一个未识别的 JSON-RPC 错误、超时、连接关闭或空响应并不能识别代际。除非那个确切的对端被配置为遗留兼容，否则失败关闭。

已识别的现代协议错误包括：

- `-32020` HeaderMismatch
- `-32021` MissingRequiredClientCapability
- `-32022` UnsupportedProtocolVersion

已识别的现代错误即使对端在遗留允许列表上，也仍然是现代的。一旦服务端证明它理解现代错误词汇，发送 `initialize` 就是降级。

不要把 `-32601` 当作正向的遗留证据。它只是让一个显式列入允许列表的对端有资格做一次遗留探测。同样的规则适用于超时、连接关闭或空响应。

### 允许列表是操作者意图，不是证据

遗留兼容必须是一份被钉住的对端配置的显式属性：

```python
client.add_server("archive", archive_transport, allow_legacy=True)
```

把这个选择绑定到已配置的命令或端点。不要用通配符让任意服务端自己选择进入更弱的语义。没有 `allow_legacy=True` 的对端，在含糊的发现结果之后失败，并且永远不会收到 `initialize`。

允许列表授予的是探测许可。它并不选择代际。客户端在传输强制的截止时间下发送一次 `initialize`，然后要求以下全部成立：

- 一条 JSON-RPC `2.0` 响应，带匹配的请求 id；
- 恰好一个 `result`，并且没有 `error`；
- `protocolVersion` 位于客户端已配置的遗留修订集合中；
- 一个对象值的 `capabilities` 字段；
- 一个 `serverInfo` 对象，其 `name` 和 `version` 字段是非空字符串。

超时、连接关闭、错误响应、畸形结果、不匹配的 id，或不受支持的修订，都失败关闭。只有结构上合法的正向结果才选择遗留代际。代码把 `legacy_probe_timeout_ms` 传给传输适配器；真正的 stdio 或 HTTP 适配器必须强制那个截止时间，而不是仅仅记录它。

为这个传输对端缓存选定的代际。不要在每次调用之前都再探测一次。

### 遗留是一条兼容分支

一旦有界探测返回合法的正向遗留证据，客户端就严格按照该修订所定义的方式使用选定的遗留版本：

1. 校验响应信封和关联 id。
2. 校验协商出的修订位于已配置的遗留集合中。
3. 记录经过校验的能力和服务端身份。
4. 只在所有检查通过之后发送 `notifications/initialized`。
5. 在该传输寿命内使用遗留请求形态。

这条分支是为了与已知对端互操作而存在的。它不是新服务端或新请求的默认设计。如果传输重启或其端点改变，丢掉对端代际缓存并重新协商。

### 发现并缓存工具

对每个活跃对端调用 `tools/list`。现代结果包含 `resultType`、`ttlMs` 和 `cacheScope`。在正确的授权上下文里遵守新鲜度提示。过期之后，或在订阅的列表变更事件之后，重新获取。

客户端必须把遗留服务端缺失的 `resultType` 当作 `"complete"`。不要在更早协商代际的响应上要求现代缓存字段。

服务端应当返回确定性顺序。客户端在合并之前也应当排序，这样本地注册表顺序就不依赖进程启动时序。

### 对碰撞安全的命名空间合并

两个服务端可能都暴露 `search`。选择一条已声明的策略：

1. **碰撞时加前缀。** 保留第一个规范名，并把后来的碰撞暴露为 `<server>/<tool>`。
2. **碰撞时拒绝。** 不加载重复项，并呈现一个清晰的配置错误。
3. **静默覆盖。** 绝不要用这个。它隐藏了模型选定的动作会送到哪个服务端。

同时存储规范名和本地名。模型看到规范名。发出去的 `tools/call` 使用拥有该工具的服务端所声明的本地名。

### 路由一次调用

路由是一次纯粹的查找：

```text
canonical tool name
  -> peer name + local tool name
  -> new JSON-RPC request id
  -> modern request metadata or explicit legacy shape
  -> matching response id
```

当拥有它的传输不可用时，不要发送调用。重新连接或重启传输，然后重新跑发现和 `tools/list`。在损坏的传输上丢失的现代在途请求，可以在操作的安全策略允许时，用一个新的 JSON-RPC id 重试。

### 通知与订阅

现代的列表和资源变更只到达客户端打开的 `subscriptions/listen` 流上。客户端发送通知过滤器，等待 `notifications/subscriptions/acknowledged`，并用通知元数据里的监听请求 id 关联事件。

断开时，打开一个新的监听请求，并重新获取相关的列表或资源。现代流不用 `Last-Event-ID` 恢复。

### 没有服务端发起的请求

现代服务端不会用独立的 JSON-RPC 请求去调用客户端以做 sampling、elicitation 或 roots。它们返回 `input_required`，客户端在完成内嵌的输入请求之后重试原来的请求。

在完成输入期间，不要阻塞对端的响应读取器。保留关联，并为重试创建一个新的 JSON-RPC id。

```figure
tp-client-merge
```

## 用起来

`code/main.py` 使用进程内的对端函数，以便协议决定保持可见。它连接到两个现代对端和一个有意列入允许列表的遗留对端，然后合并并路由它们的工具。传输可调用对象收到一个超时预算，因此兼容分支不能把一次无界探测藏起来。

```bash
cd code
python3 main.py
python3 -m unittest discover tests -v
```

测试证明普通演示会漏掉的边界：

- 现代请求重复元数据；
- `-32022` 重试现代发现，而不做初始化；
- 已识别的现代错误永不降级，即使对端在允许列表上；
- 超时、连接关闭、空响应和未识别错误，在没有允许列表时不触发 `initialize`；
- 列入允许列表的对端只有在一个合法、受支持的 `initialize` 结果之后才变成遗留；
- 畸形和不受支持的遗留结果让对端不可用；
- 成功选定的代际在传输寿命内被缓存。

## 交付

本课交付 `outputs/skill-mcp-client-harness.md`。它搭建现代请求盖章、stdio 代际协商、确定性命名空间合并、路由，以及一条失败关闭的遗留兼容分支。

## 练习

1. 让一个假服务端返回 `-32022`，且没有双方都支持的版本。确认客户端失败，而不是发送 `initialize`。
2. 把一个假遗留服务端列入允许列表，让它有界的 `initialize` 探测超时，并证明对端保持 `unknown` 且不可用。
3. 为两个授权上下文增加 `cacheScope: "private"` 的工具列表。确认客户端从不把一个上下文的缓存结果分享给另一个。
4. 把碰撞策略改成拒绝，并让启动失败，错误里同时出现两个对端名。
5. 增加一个有限的 `subscriptions/listen` 模拟器。在流丢失时，用一个新的请求 id 重新监听，并重新获取工具。

## 关键术语

| 术语 | 含义 |
|------|---------|
| 对端 | 客户端侧的记录，对应一个服务端传输及其已发现的数据 |
| 协议代际 | 现代的逐请求元数据，或遗留的初始化语义 |
| 发现探测 | 用来识别 stdio 代际的初始 `server/discover` |
| 已识别的现代错误 | 证明现代行为并禁止遗留回退的错误 |
| 遗留允许列表 | 操作者配置，允许对一个被钉住的对端做一次有界兼容探测 |
| 正向遗留证据 | 针对一个显式受支持的遗留修订、合法且已关联的 `initialize` 结果 |
| 合并后的命名空间 | 所有活跃对端上的规范工具名 |
| 碰撞策略 | 针对重复工具名的加前缀或拒绝规则 |
| 代际缓存 | 为一个传输对端存储的、选定的现代或遗留行为 |
| 传输恢复 | 重启或重连，重新发现，重新列出，并用新 id 安全重试 |

## 延伸阅读

- [MCP 规范 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/)
- [MCP Server Discovery](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
- [MCP stdio Transport](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio)
- [MCP Versioning](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning)
- [MCP Tools](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)
