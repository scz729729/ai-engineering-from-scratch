# MCP 安全：被投毒的元数据、路由，以及 MRTR 状态

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 无状态并不等于无需信任。它意味着每个请求都暴露出服务端和网关独立校验这次调用所需的证据。

**Type:** 理解
**Languages:** Python
**Prerequisites:** 第 13 阶段 · 07（MCP 服务端），第 13 阶段 · 08（MCP 客户端）
**Time:** ~60 分钟

## 学习目标

- 把工具描述、注解、客户端信息和服务器信息都当作不可信数据。
- 检测元数据投毒、描述符变更，以及跨服务端的名称碰撞。
- 校验 2026-07-28 的请求元数据和 Streamable HTTP 路由头。
- 保护 MRTR 的 `requestState` 不被篡改，并把确认绑定到确切参数。
- 把授权和速率限制应用到主体上，而不是一个已被移除的协议会话上。

## 问题

模型读工具描述来决定调用什么。路由器读工具名来决定把请求送到哪里。用户读标签来决定批准什么。一个恶意描述符可以同时瞄准这三者。

官方 MCP（模型上下文协议）安全指引很直接：除非描述和注解来自受信任的服务端，否则应把它们当作不可信。即便如此，部署信任也会变化。一次服务端更新、被攻陷的包、注册表错误，或网关合并，都可以改变模型看到的东西。

当前协议也改变了安全边界。在 2026-07-28 里没有核心握手，也没有传输会话。一个只按 `Mcp-Session-Id` 来键控批准、速率限制或审计历史的安全设计，不是当前的设计。

## 概念

### 值得检查的七个攻击面

用一份具体清单，而不是「小心一点」这种含糊指示。

1. **元数据投毒。** 描述里含有与所声明工具行为无关的指令。
2. **描述符抽地毯。** 一个此前已批准的名称、描述、schema 或注解发生了变化。
3. **跨服务端遮蔽。** 两个后端暴露同一个未限定的工具名，路由静默地选了其中一个。
4. **头与消息体混淆。** `Mcp-Method` 或 `Mcp-Name` 与 JSON-RPC 请求不一致。
5. **能力提升。** 对端声称某个扩展或客户端特性，服务端把那份声明误当成授权。
6. **MRTR 状态篡改。** 客户端改掉 `requestState`、回答了另一个问题，或把确认拿去配不同的参数复用。
7. **供应链身份混淆。** 一个眼熟的显示名被当成发布者或服务端身份的证明。

这些表面互相重叠。哈希钉住有助于发现描述符变更，但不能证明第一个描述符是安全的。静态扫描能抓住明显短语，但抓不住微妙指令。命名空间能防止一类碰撞，但不能防止一个恶意的、带命名空间的服务端。把这些控制叠起来。

### 当前请求信封是证据，不是身份

每个 2026-07-28 请求都包含：

```json
{
  "_meta": {
    "io.modelcontextprotocol/protocolVersion": "2026-07-28",
    "io.modelcontextprotocol/clientCapabilities": {
      "elicitation": {"form": {}}
    },
    "io.modelcontextprotocol/clientInfo": {
      "name": "security-lab",
      "version": "1.0.0"
    }
  }
}
```

在每个请求上校验版本和能力形态。用能力来选择兼容的响应形态。不要把 `clientInfo` 当作已认证的主体。它是自报的。

同样的警告适用于结果元数据里的 `io.modelcontextprotocol/serverInfo`。它对日志和调试有用。它不是证书、注册表证明，或授权决定。

### 在策略之前校验路由

对于 `tools/call`，Streamable HTTP 包含：

```text
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: notes.export
```

头里的方法必须等于消息体里的方法。头里的名称必须等于 `params.name`。在选择后端、应用 RBAC 或消耗速率限制令牌之前，用 `-32020` 拒绝不一致。

这个顺序关掉一种常见歧义：一个组件授权消息体，另一个组件按头来路由。

线上校验遵循一个确切序列。先校验 JSON-RPC 和元数据类型，再把头的值与消息体比较，然后检查匹配到的版本是否受支持。不匹配的头返回 HTTP 400 和 `-32020`。如果头和消息体一致地指向一个不受支持的版本，返回 HTTP 400 和 `-32022`，并且 `data` 恰好是 `{"supported":["2026-07-28"],"requested":"<actual>"}`。未知方法返回 HTTP 404 和 `-32601`。

当契约需要结构化的恢复信息时，每个错误对象都包含可选的 `data`。通知没有 `id`，因此它永远不会收到 JSON-RPC 成功或错误响应。一个被接受的 HTTP 通知返回 202 和空消息体。

### 钉住整个描述符

只哈希描述会漏掉 schema 和注解的变更。把用户批准过的描述符字段规范化并哈希：

```python
normalized = json.dumps(tool, sort_keys=True, separators=(",", ":"))
digest = hashlib.sha256(normalized.encode()).hexdigest()
```

把摘要存在一个限定键下，例如 `notes.export`，并连同发布者证据和批准时间一起放在这个玩具例子之外。

每次刷新时：

- 未知键：隔离，直到评审。
- 同一键、不同摘要：作为抽地毯隔离，直到重新批准。
- 重复的未限定名称：要求确定性的命名空间。
- 扫描器命中：阻断，并评审完整描述符。

哈希相等证明的是稳定，不是安全。一个被投毒的描述符，即便被完美钉住，仍然是被投毒的。

### 静态扫描是绊线

简单模式可以标记角色标签、指令覆盖、隐瞒、秘密访问，以及被遮掩的网络目的地。它们便宜到可以放在安装时和 CI 里。

它们不是语义证明。一条安全的描述可能在合法警告里包含被标记的短语。一条恶意描述可以避开每一个短语。把扫描器输出当作评审证据，而不是自动的清白分数。

### 合并之前先加命名空间

假设两个服务端都暴露 `search`。绝不要让发现顺序决定谁赢。

```text
notes.search
issues.search
```

限定名是公开的网关名。后端映射单独记录。稳定的名称让批准、审计、哈希钉和 `Mcp-Name` 路由都指向同一个对象。

### 能力是兼容性声明

逐请求的 `clientCapabilities` 告诉服务端，客户端能处理哪些协议特性。它并不授予客户端对工具、数据或动作的访问。

授权仍然来自已认证的主体和资源策略。顺序是：

1. 认证传输凭据。
2. 校验版本、头和请求形态。
3. 检查能力兼容性。
4. 授权主体、工具、资源和参数。
5. 执行，或请求用户输入。

### 保护无状态的 MRTR 确认

一个有后果的工具可能需要用户确认。当前 MCP 使用多轮往返请求（Multi Round-Trip Requests），而不是服务端到客户端的回调。

第一次响应：

```json
{
  "resultType": "input_required",
  "inputRequests": {
    "confirm": {
      "method": "elicitation/create",
      "params": {
        "mode": "form",
        "message": "Export notes to archive?",
        "requestedSchema": {
          "type": "object",
          "properties": {
            "confirm": {"type": "boolean"}
          },
          "required": ["confirm"]
        }
      }
    }
  },
  "requestState": "opaque-integrity-protected-value"
}
```

客户端取得输入，并用一个新的 JSON-RPC id 重试原来的方法：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "notes.export",
    "arguments": {"query": "private", "destination": "archive"},
    "requestState": "opaque-integrity-protected-value",
    "inputResponses": {
      "confirm": {
        "action": "accept",
        "content": {"confirm": true}
      }
    },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {
        "elicitation": {"form": {}}
      }
    }
  }
}
```

每个 `inputRequests` 的值都是一个完整的内嵌请求，带 `method` 和 `params`。它的键必须匹配 `inputResponses` 里的对应条目。表单 elicitation 使用对象根的 `requestedSchema`，并且在服务端请求它之前，客户端必须已经声明了表单 elicitation 能力。

当前能力有两种合法的表单声明。`{"elicitation":{}}` 隐式支持表单 elicitation，而 `{"elicitation":{"form":{}}}` 则显式声明它。仅 URL 的声明，例如 `{"elicitation":{"url":{}}}`，不支持表单请求。服务端返回 HTTP 400 和 `-32021`，并且 `data.requiredCapabilities` 等于 `{"elicitation":{"form":{}}}`。

把 `requestState` 当作敌意输入。对它签名或加密，校验它，并在重放要紧时把它绑定到方法、工具、确切参数、目的、过期时间、主体，以及一次性 nonce。本课代码用 HMAC 和确切参数匹配，让这条边界看得见。

nonce 账本不得活在单个网关对象里面。可运行的模型注入一个有界、按 TTL 修剪的重放存储，可以由多个网关实例共享。它的原子认领就是执行边界：只有经过校验的接受，或显式的终态拒绝，才会消耗状态。畸形响应或 `cancel` 什么都不执行，并且在过期前仍可重试。生产集群需要在共享的持久存储里做同样的条件认领。

不要把隐藏的确认上下文存在协议会话里。任何一个服务端实例都应当能校验这次重试。

### 高风险调用的二选规则

沿三条轴给一次调用分类：

- 它消费不可信输入。
- 它可以访问敏感数据。
- 它造成一个有后果的外部动作。

单个自动步骤不应当把三者全部组合起来。拆开它、降低特权，或通过 MRTR 请求显式的用户输入。这是一条设计启发式，不是协议能力。

### 在执行之前削减权限

单靠无状态并不是安全。它去掉了隐藏的协议历史，但一个自包含的请求仍然可以要求一个权限过大的处理程序泄漏数据，或做出不可逆的变更。安全来自在每个边界削减权限：

1. **带类型的动词。** 暴露一个有界操作，例如 `archive_note`，而不是一个能表达无关权力的泛型 `run` 或 `request` 工具。
2. **经过校验的参数。** 在可行处使用封闭 schema，拒绝未知字段，把标识符规范化一次，限制大小，并在策略评估之前校验目的地、租户和资源所有权。
3. **当前授权。** 把已认证主体绑定到确切的动词、资源、环境和规范化后的参数。工具注解和客户端能力并不授予这份权限。
4. **绑定到动作的批准。** 对于有后果的调用，把批准绑定到带类型动词和规范化参数的摘要，再加上主体、过期时间和一次性策略。任何字段变化都需要一次新的决定。
5. **一等拒绝。** 把模型拒绝、过期批准、用户拒绝和不安全目的地建模为普通结果，它们不执行任何副作用。不要把拒绝翻译成一个更弱的回退工具。
6. **经脱敏的审计证据。** 记录谁提出请求、用了哪个被接纳的描述符和策略版本、授权了哪个规范化目标、决定为何允许或拒绝，以及执行是否已经开始。存摘要或脱敏值，而不是秘密。

每一步都收窄下一个组件可以做什么。最终的处理程序应当收到一条已经校验过的领域命令，而不是原始模型文本加上宽泛凭据。在 MRTR 重试、任务更新或网关转发的调用上，把整条链再走一遍。更早的批准不会把后来的请求变成受信任的会话流量。

### 当前与遗留的交互路径

Roots、Sampling 和 Logging 对新的 2026-07-28 实现已弃用。网关可以只把更旧的请求通道代码保留为一条按版本设闸的兼容路径。

不要围绕按会话的 sampling 限制器去构建新的防御。把配额应用到已认证主体、签发者、资源、工具和时间窗口上。对于当前的交互式工作，检查 MRTR 的输入请求和响应。

### 无状态传输检查

- 在单一 POST 端点接受现代 MCP 消息。
- 对现代的 GET 和 DELETE 返回 405。
- 不要铸造 `Mcp-Session-Id`，也不要依赖它。
- 把遗留的会话头和重放头当作权限输入时忽略它们。
- 为那次 POST 返回 JSON，或请求作用域的 SSE。
- 只把 `subscriptions/listen` 用于已选择加入的长寿命变更通知。

```figure
tp-tool-poisoning
```

## 动手构建

`code/main.py` 实现一个小型的进程内安全网关模型。它规范化并钉住完整的工具描述符，报告元数据投毒和遮蔽，校验现代请求信封和路由值，并用带签名的 `requestState` 以及注入的共享重放存储，完成一轮两回合的确认导出。

模型从 HTTP 适配器已经解析了 JSON 消息体和路由头之后开始。它不校验 `Content-Type` 或 `Accept`。把同一个分派器接到第 09 课完整的 Streamable HTTP 适配器上，后者要求 `Content-Type: application/json`，以及一个同时包含 `application/json` 和 `text/event-stream` 的 `Accept` 值。

运行它：

```bash
cd phases/13-tools-and-protocols/15-mcp-security-tool-poisoning
python3 code/main.py
python3 -m unittest discover code/tests -v
```

这个样例有意改变一个描述符。扫描器和摘要比较产生彼此独立的发现。随后导出演示 `input_required` 响应和无状态重试。

## 用起来

用你自己已批准服务端的一份规范化快照替换 `SAFE_TOOLS`。把凭据和秘密排除在快照之外。在更新其摘要之前，评审每一个新的或已变更的描述符。

在网关上，于发现期间跑同样的检查，并在分派之前再跑一次。缓存可以减少发现工作，但缓存的批准必须过期，或在描述符变化时失效。

## 交付

本课交付 `outputs/skill-mcp-threat-model.md`。它产出一份当前协议的威胁模型，覆盖元数据、路由、能力、授权、MRTR、缓存、注册表和兼容边界。

## 练习

1. 把已认证主体和当前授权决定绑定到封存的 MRTR 状态上，然后拒绝在不同主体下的重试。
2. 用持久的条件插入替换内存中的重放存储，并证明两个进程不能都认领同一个 nonce。
3. 在重放认领之后、模拟导出之前注入一次失败。定义并测试使恢复安全的事务或幂等规则。
4. 改变一个工具的 `inputSchema`，但不改它的描述。确认整份描述符钉住能抓住它。
5. 增加一条策略：当 `tools/list` 因主体而不同时，拒绝公共缓存。
6. 建模网关后面的一个更旧服务端。把所有握手和会话行为都放在一条显式的 `2025-11-25` 兼容分支后面。

## 关键术语

| 术语 | 含义 |
|------|---------|
| 元数据投毒 | 嵌在工具描述符里的指令或欺骗性声称 |
| 抽地毯 | 对一个此前已批准描述符的变更 |
| 工具遮蔽 | 由重复的未限定名称造成的含糊路由 |
| 头不匹配 | 路由头与 JSON-RPC 消息体不一致，错误 `-32020` |
| 哈希钉 | 完整的已批准描述符的摘要 |
| MRTR | 服务端请求输入时的无状态响应与重试模式 |
| `requestState` | 不透明的往返值，必须当作不可信输入 |
| 能力声明 | 关于协议兼容性的陈述，不是授权 |
| 隐式表单支持 | 一个空的 `elicitation` 能力对象，等价于表单支持 |
| 限定工具名 | 稳定的网关名，例如 `notes.search` |

## 延伸阅读

- [MCP 安全与信任指引](https://modelcontextprotocol.io/specification/2026-07-28#security-and-trust--safety)
- [多轮往返请求](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr)
- [Streamable HTTP 传输](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [已弃用特性](https://modelcontextprotocol.io/specification/2026-07-28/deprecated)
