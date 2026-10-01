# MCP 工具契约与内容

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 只有发现、参数、结果、分页和传输元数据共同遵守同一份契约时，工具才适合被自动化。

**Type:** 动手做
**Languages:** Python
**Prerequisites:** 第 13 阶段，第 07、09、10 课
**Time:** ~120 分钟

## 学习目标

- 用 JSON Schema 2020-12 定义工具的输入和输出。
- 校验结构化结果，不要假定它们一定是 JSON 对象。
- 在 text、image、audio、资源链接和嵌入资源之间做选择。
- 在工具到达模型之前，拒绝不安全的 `x-mcp-header` 定义。
- 对参数头的值做编码，并核对请求头与正文完全一致。
- 遍历游标分页，但不解释游标的值。
- 为 `completion/complete` 的建议加上边界和授权。

## 问题

调用一个 Python 函数很容易。通过 AI 宿主调用一项远程能力，则是契约问题。

服务器发布一份描述符。客户端把这份描述符变成模型上下文和用户界面。模型生成参数。网关可能根据镜像出来的请求头路由这次请求。服务器执行工具。然后客户端决定结果是否足够安全、足够有效，可以交回给模型。

边界上只要有一处薄弱，整条链都会被带坏。

考虑五种失败：

- 描述符说结果是对象，服务器却返回了数组。
- `nextCursor` 是空字符串时，客户端停止了分页。
- 一个 token 参数被镜像进 HTTP 头，从而对中间人可见。
- 一个 Unicode 路由值被当作原始请求头发送，网关和源站随后解释出不同的字节。
- 补全端点向一个无权访问生产环境的调用方建议了生产环境。

这些失败都不是靠更好的提示词能修好的。它们需要明确的协议契约和应用契约。

## 契约流水线

把每次工具调用看成五道闸门：

1. **发现。** 读取一份确定的、分页的工具列表。
2. **准入。** 校验每个描述符，并施加本地安全策略。
3. **调用。** 校验参数，并构造传输元数据。
4. **执行。** 运行处理函数，并正确分类失败。
5. **消费。** 在交给模型使用之前，校验内容块和结构化输出。

```figure
mcp-contract-pipeline
```

宿主拥有准入闸门和消费闸门。服务器不能强迫客户端信任它的注解、schema 或输出。

## JSON Schema 是运行时边界

在 MCP（模型上下文协议）`2026-07-28` 中，`inputSchema` 和 `outputSchema` 使用 JSON Schema。当 `$schema` 缺省时，默认方言是 2020-12。

输入 schema 必须是一个 schema 对象。没有参数的工具仍应确切说明它接受什么：

```json
{
  "type": "object",
  "additionalProperties": false
}
```

这比 `{ "type": "object" }` 更严格，后者会接受任意属性。

输出 schema 是可选的。服务器一旦发布了一份，每一个完整的工具结果都承诺返回符合该 schema 的 `structuredContent`，包括 `isError: true` 的结果。错误标志分类的是执行结果；它并不豁免已发布的输出契约。客户端应当校验结果，而不是信任描述符。

### 结构化内容可以是任意 JSON 值

不要把 `structuredContent` 硬编码成字典。它可以是：

- 对象；
- 数组；
- 字符串；
- 数字；
- 布尔值；
- `null`。

这个工具返回一个数组：

```json
{
  "name": "tag_catalog",
  "inputSchema": {
    "type": "object",
    "additionalProperties": false
  },
  "outputSchema": {
    "type": "array",
    "items": {"type": "string"}
  }
}
```

它的成功结果是合法的：

```json
{
  "resultType": "complete",
  "content": [
    {
      "type": "text",
      "text": "[\"contracts\", \"mcp\", \"stateless\"]"
    }
  ],
  "structuredContent": ["contracts", "mcp", "stateless"],
  "isError": false
}
```

为了兼容，结构化结果还应在一个文本块里包含序列化后的 JSON。这段文本不是校验来源。`structuredContent` 才是。

### 一个小校验器仍然能讲清边界

本课使用一个刻意缩小的 JSON Schema 子集，因为它可以留在 Python 标准库之内。它检查示例工具所用的机制：

- object、array、string、integer、number、boolean 和 null 类型；
- 必填属性；
- `additionalProperties: false`；
- 数组元素；
- enum 值；
- 字符串最小长度。

这不能替代完整的生产级校验器。可以复用的教训是校验发生的位置：描述符在发现之后，参数在执行之前，结构化结果在消费之前。

## 内容块承担不同的代价

`content` 数组可以组合多种内容类型。

| 类型 | 适用场景 | 主要边界 |
|------|------------|---------------|
| `text` | 人和模型都能读的摘要 | 把文本当作不可信输出 |
| `image` | 以 base64 编码的视觉证据 | 校验媒体类型和大小 |
| `audio` | 以 base64 编码的语音或录音输出 | 校验媒体类型和时长上限 |
| `resource_link` | 客户端以后可以去拉取的 URI | 对随后的资源读取重新授权 |
| `resource` | 直接嵌在结果里的数据 | 此刻就强制执行载荷和内容限制 |

资源链接并不能证明该资源出现在 `resources/list` 中。它是这次工具调用返回的引用。客户端在跟随该 URI 时，仍然要施加自己的资源策略。

嵌入资源省掉了另一次往返，但增大了当前响应的体积。大的、或会独立变化的产物用链接。必须与结果原子一起传递的小证据用嵌入资源。

本课的 `evidence_bundle` 结果包含全部五种类型。客户端在接受结果之前校验每一个块。

## `x-mcp-header` 是路由元数据

`inputSchema` 里的一个属性可以声明 `x-mcp-header`。在 Streamable HTTP 上，客户端把该参数镜像到 `Mcp-Param-{name}`。

```json
{
  "region": {
    "type": "string",
    "x-mcp-header": "Region"
  }
}
```

当 `region: "eu-west"` 时，传输可以发出：

```http
Mcp-Param-Region: eu-west
```

这条注解的存在，是为了让负载均衡器、网关或策略引擎不必解析 JSON 正文就能路由。它不是放凭证的地方。

协议对这条注解有约束：

- 头名称非空，并符合 HTTP field-name 的 token 语法；
- 头名称在忽略大小写的前提下唯一；
- 属性类型是 string、integer 或 boolean；
- 不允许 `number`；
- 注解只出现在 `inputSchema.properties` 的直接成员上；
- 整数值保持在 `-9007199254740991` 到 `9007199254740991` 之间。

位置规则是句法上的，并且失败即关闭。要遍历整棵 schema 树，而不只是你的校验器恰好能理解的那些属性。嵌套对象的 `properties` 下、`oneOf` 分支、`items`、经 `$ref` 到达的 definition、或任何输出 schema 里的注解都要拒绝。解析引用并不会把被引用节点变成顶层的直接属性。

本课增加一条部署策略：拒绝镜像 `password`、`secret`、`token`、`api_key` 或 `authorization` 这类名称的描述符。官方规范建议服务器作者不要镜像敏感参数。客户端可以把这条建议变成硬性的准入规则。

审计头名称，而不是它的值。示例代码记录 `Mcp-Param-Region`，同时把 `eu-west` 排除在审计事件之外。

### 构造 HTTP 头之前先编码值

参数值只有在同时满足以下条件时，才可以作为明文传输：它是非空字符串，由 `!` 到 `~` 的可见 ASCII 字符组成，并且不像编码哨兵。其他一切都使用下面这种精确形式：

```text
=?base64?{Base64UTF8}?=
```

`Base64UTF8` 是对精确 UTF-8 字节做的标准 base64。不要先裁剪、规范化或替换这个值。Unicode、空字符串、空格、制表符、控制字符、CR 或 LF、前导或尾随空白，以及任何以 `=?base64?` 开头的值，都要编码。把看起来像哨兵的值再编码一次，才能让接收方恢复字面上的原文，而不是把它当成传输语法去解码。

布尔值渲染为小写的 `true` 或 `false`。整数以十进制渲染，并且必须留在 JavaScript 安全整数范围内。超出该范围的值应被拒绝，而不是由中间人取整。

### 服务器核对镜像副本

生成请求头只是客户端这一半。在 Streamable HTTP 边界上，服务器必须：

1. 在忽略头名称大小写的前提下，找出已识别的 `Mcp-Param-*` 名称；
2. 若存在精确的 base64 哨兵形式，则解码它；
3. 把解码后的文本与对应的 JSON 正文参数做精确比较；
4. 在分发之前，拒绝缺失、重复、意外、格式错误或不匹配的已识别请求头。

拒绝时使用 HTTP `400`，JSON-RPC 错误码为 `-32020`。正文的值及其编码后的头形式都不属于审计记录。只记录已识别的头名称和拒绝类别。

`code/main.py` 直接建模了这条边界。[第 09 课](../../09-mcp-transports/)涵盖更宽的 Streamable HTTP 校验顺序，包括方法与协议版本的一致性。

## 分页游标是不透明的

MCP 的列表操作使用游标分页。服务器选择页大小和游标格式。客户端只做一个判断：

```python
if result.get("nextCursor") is None:
    break
cursor = result["nextCursor"]
```

不要写成这样：

```python
if not result.get("nextCursor"):
    break
```

空字符串是合法游标。真值判断会停得太早。

客户端不得解码游标、递增它、把它和先前的游标比较以推断顺序，或推断页码。服务器可以对游标签名、把它绑定到目录版本，或把它映射到私有状态。那是服务器的实现细节。

示例服务器故意在第一页之后返回 `""`。客户端必须在第二次请求中发送这个精确的值。它的轨迹是：

```text
<first request with no cursor>
<second request with cursor "">
```

无效游标产生 JSON-RPC invalid params，错误码 `-32602`。

## 补全是一个授权面

`completion/complete` 为提示词参数和资源模板参数提供建议。它对交互式表单有用，但可能泄露普通列表方法所保护的名称。

一次补全请求指名一个引用，以及正在补全的参数：

```json
{
  "method": "completion/complete",
  "params": {
    "ref": {
      "type": "ref/prompt",
      "name": "deployment_review"
    },
    "argument": {
      "name": "environment",
      "value": "st"
    }
  }
}
```

结果最多返回 100 个值，并可能报告 `total` 和 `hasMore`。

施加与被引用的提示词或资源相同的授权边界。示例中的分析师会收到 `development` 和 `staging`。只有操作员能收到 `production`。

生产环境的补全还需要：

- 输入校验；
- 感知调用方的过滤；
- 客户端的请求去抖；
- 服务器的速率限制；
- 有界的结果数量；
- 不暴露敏感建议值的日志。

补全是辅助，不是绕过发现。

## 两层错误

把协议错误和工具执行错误分开。

当 MCP 请求无法被正确分发时，使用 JSON-RPC 错误：

- 未知的工具名；
- 畸形的请求形状；
- 缺少请求元数据；
- 无效游标。

当调用已经到达工具、并且工具报告了一个可处理的失败时，使用带 `isError: true` 的完整工具结果：

- 报告源不可用；
- 日期超出支持范围；
- 业务规则拒绝所请求的操作。

模型通常能修复工具执行错误。它们修不好一台违反了自己输出 schema 的服务器。

如果工具声明了输出 schema，就把可处理的失败建模在该 schema 之内。示例中 `route_report` 的失败会返回所请求的 region，并带上 `accepted: false`，同时附有人类可读的错误文本和 `isError: true`。

## 动手做

`code/main.py` 用 Python 标准库构建边界的两侧。

服务器实现：

- 按请求校验 MCP 元数据；
- 带 tools 和 completions 能力的 `server/discover`；
- 确定的 `tools/list` 分页；
- 四个工具描述符，其中一个必须被拒绝；
- 数组形式的结构化输出；
- 当前每一种工具内容块类型；
- 一道 Streamable HTTP 一致性闸门，它解码已识别的参数头，并在不匹配时返回 HTTP `400` 加上 JSON-RPC `-32020`；
- 经过授权并限流的补全。

客户端实现：

- 描述符准入；
- 全树的 `x-mcp-header` 位置校验和敏感字段策略；
- 精确的可见 ASCII 明文或 base64 UTF-8 值编码；
- 会跟随空字符串的不透明游标循环；
- 参数和结果校验；
- 内容块校验；
- 只含名称、不含值的头审计事件。

那个故意不安全的描述符是教学数据。它证明：拒绝一个工具并不会阻止合法工具加载。

## 使用

在仓库根目录：

```bash
cd phases/13-tools-and-protocols/28-mcp-tool-contracts-and-content/code
python3 main.py
python3 -m unittest discover tests -v
```

演示会打印已准入的工具、被拒绝的描述符、两次分页请求、结构化数组内容、内容块类型、镜像的头名称、该值是否需要编码、HTTP 一致性状态，以及按调用方过滤后的补全值。

## 交互实验

打开 `code/main.py`，找到 `TOOLS`。

1. 把 `tag_catalog.outputSchema.type` 从 `array` 改成 `object`。
2. 运行演示。客户端应当拒绝返回的数组。
3. 恢复 schema。
4. 保持第一页的 `nextCursor` 为 `""`，然后让最后一页返回 `nextCursor: None`，而不是省略该字段。
5. 运行测试，并比较游标轨迹。
6. 给一个字符串属性加上 `x-mcp-header: "Authorization"`。
7. 确认描述符准入在调用之前就拒绝了它。
8. 试一下包含 Unicode、换行、两侧空格，以及字面文本 `=?base64?SGVsbG8=?=` 的 `region` 值。解码每一个发出的请求头，证明原始值被精确保留。
9. 把注解移到 `oneOf`、`items` 或 `$ref` definition 之下。确认每个描述符都被拒绝，即使演示从未用到那条分支。
10. 去掉已识别的请求头，或改变它解码后的值。确认 HTTP 边界返回状态 `400` 和 JSON-RPC 码 `-32020`。

要点不是记住一种 JSON 形状。而是看着每一道闸门在它负责的边界上失败。

## 练习实验

用一个 `search_evidence` 工具扩展契约实验。

要求：

1. 它的输入 schema 接受 `query`、`limit`，以及一个安全的 `region` 路由字段。
2. 它的输出 schema 是对象数组，对象含 `uri`、`title` 和 `score`。
3. 结果包含兼容文本，以及每项一个资源链接。
4. 参数拒绝未知属性。
5. `limit` 由应用层校验限制上界。
6. 无权访问某个 URI 的调用方，永远不会通过补全或工具输出看到该 URI。
7. 测试要包括一个不合规的 score、一条无效的头注解，以及一份两页的列表。
8. 头值测试覆盖可见 ASCII、Unicode、控制字符、空白、看起来像哨兵的文本，以及 JavaScript 安全整数的两端边界。
9. HTTP 夹具接受大小写不敏感的头名称，但用状态 `400` 和错误码 `-32020` 拒绝缺失或不匹配的已识别值。

## 交付产物

`outputs/skill-mcp-contract-reviewer.md` 是一份扁平、可复用的审查技能。把工具描述符、示例结果、分页行为和补全策略交给它。它返回准入决定、结果校验计划、头策略，以及具体的失败测试。

## 验证

当下列陈述为真时，本课才算完成：

- `tools/list` 在重复调用时返回相同的逻辑顺序。
- 当 `nextCursor` 为 `""` 时，客户端会发出第二次请求。
- 不安全的敏感头描述符被排除，同时其他工具仍然可用。
- 数组能通过它的数组输出 schema。
- 对象会在同一份数组 schema 上失败。
- 错误结果不能省略或违反已发布的输出 schema。
- text、image、audio、资源链接和嵌入资源块都能通过校验。
- 头审计事件包含名称，不包含值。
- 可见 ASCII 明文保持明文；Unicode、控制字符、带填充、空值，以及看起来像哨兵的值，都能经精确的 base64 UTF-8 编码往返还原。
- 超出 JavaScript 安全范围的镜像整数会被拒绝。
- `oneOf`、`items`、嵌套对象、`$ref` definition 或输出 schema 之下的注解，会在准入阶段被拒绝。
- 大小写不敏感的已识别头名称，只有在解码值与正文精确匹配时才通过；缺失或不匹配的副本产生 HTTP `400` 和 JSON-RPC `-32020`。
- 分析师的补全永远不返回 `production`。
- 工具失败使用 `isError: true`；畸形的协议调用使用 JSON-RPC `error`。

## 生产失败模式

| 失败 | 学习者看到什么 | 正确应对 |
|---------|-----------------------|------------------|
| 客户端假定输出是对象 | 合法数组失败，或被悄悄包了一层 | 按已发布的 schema 校验，不要只用对象类型 |
| 空游标被当成假值 | 最后几页消失 | 只要 `nextCursor` 存在且不是 null，就继续 |
| 敏感值被镜像 | 秘密出现在代理、WAF 或追踪数据里 | 拒绝该描述符，把秘密留在受保护的请求数据中 |
| 原始 Unicode 或空白被镜像 | 网关和源站不一致，或值被规范化 | 使用精确的 base64 UTF-8 哨兵编码，解码后再比较 |
| 注解藏在 schema 分支里 | 客户端在准入时漏掉路由元数据 | 遍历整棵 schema 树，只允许顶层的直接属性 |
| 大整数被镜像 | JavaScript 中间人把路由值取整 | 拒绝 JavaScript 安全整数范围之外的值 |
| 请求头和正文不一致 | 网关路由到一个目标，源站却执行另一个 | 在分发之前用 HTTP `400` 和 JSON-RPC `-32020` 拒绝 |
| 输出 schema 被忽略 | 下游代码消费了损坏的结构 | 在模型或应用使用之前校验 |
| 资源链接被自动信任 | 调用方跟随了一个未授权的 URI | 每一次资源读取都重新授权 |
| 补全共享全局建议 | 隐藏的租户名泄露 | 按调用方、引用和授权过滤 |
| 工具注解被当成策略 | 破坏性操作绕过确认 | 在注解之外强制执行授权和审批 |
| 一个畸形工具弄坏发现 | 整台服务器变得不可用 | 拒绝坏描述符，并独立准入合法工具 |

## 与毕业项目的连接

第 13 阶段的毕业项目需要一个能合并多台服务器工具的网关。本课提供它的准入核心。

用这份产物给四块毕业项目证据打分：

- 确定且完整的分页发现；
- 在暴露给模型之前校验描述符；
- 经过校验的结构化输出，加上有界的内容块；
- 保持授权边界的补全和路由元数据。

不要仅凭一次成功的 `tools/call` 就声称网关兼容。要留下描述符、分页轨迹、已准入工具集、被拒绝工具集，以及一份通过校验的结果。

## 关键术语

| 术语 | 含义 |
|------|---------|
| `inputSchema` | 定义所接受工具参数的 JSON Schema 对象 |
| `outputSchema` | 定义 `structuredContent` 的可选 JSON Schema |
| `structuredContent` | 工具结果产生的任意 JSON 值 |
| 内容块 | 带类型的文本、图像、音频、资源链接或嵌入资源 |
| `x-mcp-header` | 把一个原始类型参数镜像进 Streamable HTTP 元数据的 schema 注解 |
| 不透明游标 | 服务器签发的分页令牌，客户端不解释它的值 |
| 补全引用 | 其参数正在被补全的提示词名称或资源 URI/模板 |
| 准入 | 客户端决定暴露或拒绝一份已发现描述符 |

## 延伸阅读

- [MCP Tools](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)
- [MCP Completion](https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/completion)
- [MCP Pagination](https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/pagination)
- [MCP Streamable HTTP Parameter Headers](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http#custom-headers-from-tool-parameters)
