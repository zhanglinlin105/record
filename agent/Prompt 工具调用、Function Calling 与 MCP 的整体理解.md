# Prompt 工具调用、Function Calling 与 MCP 的整体理解

## 一、三者分别解决什么问题

三者都与大模型调用外部工具有关，但所处层级不同。

### 1. Prompt 中自定义工具调用格式

这是最原始的工具调用方式。

开发者在 Prompt 中告诉模型：

* 有哪些工具；
* 每个工具有什么作用；
* 调用时需要哪些参数；
* 模型应当按照什么 JSON 或文本格式输出；
* 工具执行结果应该采用什么格式返回。

例如：

```text
你可以调用天气查询工具。

调用格式：
{
  "tool": "get_weather",
  "arguments": {
    "city": "城市名称"
  }
}
```

模型可能输出：

```json
{
  "tool": "get_weather",
  "arguments": {
    "city": "杭州"
  }
}
```

但从模型角度看，这仍然只是普通文本生成。

应用程序需要自行完成：

```text
读取模型输出
    ↓
提取 JSON
    ↓
判断是否为工具调用
    ↓
校验字段和参数
    ↓
调用真实工具
    ↓
把结果重新发送给模型
```

这种方式的问题是输出不够稳定，模型可能：

* 在 JSON 前后添加解释；
* 输出 Markdown 代码块；
* 写错字段名称；
* 遗漏必要参数；
* 生成无法解析的 JSON；
* 混合自然语言和工具调用；
* 不遵守开发者约定的格式。

因此，Prompt 自定义工具调用本质上是：

> 开发者与模型之间通过自然语言临时约定一种工具调用格式，并由应用程序自行解析。

---

## 二、Function Calling 是什么

Function Calling 是模型服务商在 API 层面提供的原生工具调用机制。

开发者调用模型时，会把工具定义一并提供给模型。工具定义通常包含：

* 工具名称；
* 工具描述；
* 参数结构；
* 参数类型；
* 必填字段；
* 枚举、数组、对象等约束。

例如：

```json
{
  "name": "get_weather",
  "description": "查询指定城市的天气",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string"
      }
    },
    "required": ["city"]
  }
}
```

模型决定调用工具时，不再单纯输出一段普通文本，而是返回专门的工具调用对象，例如：

```json
{
  "name": "get_weather",
  "arguments": {
    "city": "杭州"
  }
}
```

应用程序收到工具调用以后：

```text
模型生成 Function Call
        ↓
应用程序读取工具名称和参数
        ↓
执行本地函数或远程服务
        ↓
把工具执行结果返回给模型
        ↓
模型生成最终回答
```

Function Calling 主要解决的是：

> 让模型能够通过模型厂商提供的标准接口，更稳定地表达“调用哪个工具、传入哪些参数”。

### Function Calling 的主要优势

相比在 Prompt 中手写格式，Function Calling 具有以下优势：

* 工具调用和普通文本回复在接口层面分离；
* 参数可以通过 JSON Schema 描述；
* 字段名称和参数类型更加稳定；
* 应用程序不需要从普通文本中猜测是否存在工具调用；
* 更容易进行参数校验、失败重试和多工具编排；
* 某些平台可以通过严格模式进一步提高 Schema 遵循率。

但 Function Calling 并不意味着模型会真正执行工具。

模型主要负责：

```text
理解用户意图
决定是否调用工具
选择工具
生成工具参数
```

应用程序仍然负责：

```text
执行函数
访问数据库
发送网络请求
处理权限
返回执行结果
```

---

## 三、Function Calling 定义的主要是工具输入

关于 Function Calling，需要注意一个容易混淆的地方。

开发者给模型提供的 Schema，通常重点定义的是：

```text
模型 → 工具
```

也就是工具的输入参数格式，例如：

```json
{
  "city": "杭州"
}
```

至于工具执行后返回什么，很多模型 API 并不会自动强制规定完整的结果结构。

工具可能返回：

```json
{
  "temperature": 31,
  "condition": "sunny"
}
```

也可能返回：

```text
杭州当前晴，气温 31℃
```

工具输出是否结构化，通常还需要开发者自行设计。

因此，更准确的说法是：

> Function Calling 主要标准化模型产生工具调用的方式，以及工具输入参数的结构；工具输出结构是否统一，还取决于模型平台和应用实现。

---

## 四、Function Calling 的局限

Function Calling 通常属于特定模型厂商的 API 能力。

不同模型平台可能使用不同格式，例如：

```text
OpenAI Function Calling
Claude Tool Use
Gemini Function Calling
其他模型的工具调用机制
```

它们在以下方面可能存在差异：

* 工具定义字段；
* 参数 Schema 支持范围；
* 工具调用返回结构；
* 调用 ID 表达方式；
* 并行工具调用方式；
* 工具执行结果回传格式；
* 严格模式支持；
* 多轮工具调用机制。

如果工具直接嵌入项目并绑定某个模型平台，更换模型时可能需要重新适配。

例如：

```text
项目 A 的天气工具 → OpenAI 格式
项目 B 的天气工具 → Claude 格式
项目 C 的天气工具 → Gemini 格式
```

工具数量和模型数量增加后，适配工作会越来越多。

---

# 五、MCP 是什么

MCP 全称为 Model Context Protocol，即模型上下文协议。

MCP 不是某一家模型的 Function Calling 格式，而是一种让 AI 应用与外部工具、数据源和服务进行标准化通信的协议。

典型结构为：

```text
大模型
  ↓
AI 应用 / MCP Host
  ↓
MCP Client
  ↓ MCP 协议
MCP Server
  ↓
数据库、文件、GitHub、浏览器、业务系统
```

其中：

### MCP Host

用户实际使用的 AI 应用或 Agent 宿主，例如：

* 桌面 AI 应用；
* IDE；
* 智能助手；
* Agent 平台；
* 企业内部 AI 系统。

Host 负责：

* 调用模型；
* 管理对话；
* 展示结果；
* 管理 MCP Client；
* 处理用户权限和确认。

### MCP Client

MCP Client 位于 Host 内部，负责与 MCP Server 建立连接和进行协议通信。

它负责：

* 初始化连接；
* 能力协商；
* 查询工具列表；
* 发起工具调用；
* 接收工具结果；
* 处理错误、通知和取消。

### MCP Server

MCP Server 将外部能力按照 MCP 规范暴露出来，例如：

* 文件读写；
* 数据库查询；
* GitHub 操作；
* 搜索引擎；
* 浏览器控制；
* 企业订单系统；
* 内部知识库。

MCP Server 通常不需要知道前端使用的是哪一种模型，只需要按照 MCP 协议提供服务。

---

## 六、MCP 不只是工具调用格式

MCP 的范围比普通 Function Calling 更大。

Function Calling 主要关注：

```text
工具定义
工具调用
工具结果
```

MCP 还可以涉及：

* Tools；
* Resources；
* Prompts；
* 连接生命周期；
* 初始化和能力协商；
* 请求和响应；
* 错误处理；
* 通知；
* 进度信息；
* 取消机制；
* 多种内容类型；
* 本地和远程传输；
* 用户授权和权限控制。

因此 MCP 并不只是规定模型应该输出哪种 JSON，而是规定：

> AI 应用与外部工具服务器之间如何发现能力、建立连接、发送请求和返回结果。

---

# 七、Function Calling 和 MCP 的关系

Function Calling 与 MCP 并不是竞争或替代关系，它们通常处于调用链的不同位置。

```text
用户
  ↓
大模型
  ↓ Function Calling
AI 应用 / Agent / MCP Host
  ↓ MCP
MCP Server
  ↓
实际工具和业务系统
```

可以这样理解：

> Function Calling 是模型侧的工具调用接口。

> MCP 是应用侧连接外部工具服务的通用协议。

Function Calling 解决：

```text
模型如何表达：
“我要调用 get_weather，并传入 city=杭州”
```

MCP 解决：

```text
应用如何找到 get_weather 工具，
怎样把请求发送到工具服务器，
以及怎样接收执行结果。
```

二者之间需要 AI 应用或适配层进行转换。

---

# 八、Function Calling 与 MCP 的完整转换过程

一个完整的转换通常包含三个阶段。

## 1. MCP 工具定义转换成 Function Calling 定义

MCP Server 对外暴露工具：

```json
{
  "name": "get_weather",
  "description": "查询指定城市的天气",
  "inputSchema": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string"
      }
    },
    "required": ["city"]
  }
}
```

MCP Host 获取工具列表后，把它转换为当前模型支持的工具定义格式。

例如转换为某模型的 Function Calling 格式：

```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "查询指定城市的天气",
    "parameters": {
      "type": "object",
      "properties": {
        "city": {
          "type": "string"
        }
      },
      "required": ["city"]
    }
  }
}
```

这一步的目的，是让模型知道：

* 有哪些 MCP 工具；
* 工具分别做什么；
* 每个工具需要什么参数。

---

## 2. Function Calling 转换成 MCP 工具请求

模型决定调用工具后，生成：

```json
{
  "name": "get_weather",
  "arguments": {
    "city": "杭州"
  }
}
```

AI 应用将其转换为 MCP 请求：

```json
{
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "杭州"
    }
  }
}
```

然后通过 MCP Client 发送给 MCP Server。

这一步的目的，是把：

```text
模型的工具调用意图
```

翻译成：

```text
MCP Server 可以执行的标准请求
```

---

## 3. MCP 返回结果转换成模型工具结果

MCP Server 执行工具后，可能返回：

```json
{
  "content": [
    {
      "type": "text",
      "text": "杭州当前晴，气温 31℃"
    }
  ],
  "isError": false
}
```

AI 应用再将这个结果包装为当前模型 API 要求的工具结果格式，并传回模型。

模型根据结果生成最终回答，例如：

```text
杭州当前天气晴，气温约为 31℃。
```

完整流程为：

```text
MCP tools/list
      ↓
MCP 工具定义
      ↓ 转换
模型 Function Calling 工具定义
      ↓
模型选择工具并生成参数
      ↓ 转换
MCP tools/call
      ↓
MCP Server 执行工具
      ↓
MCP Result
      ↓ 转换
模型 Tool Result
      ↓
模型生成最终回答
```

---

# 九、相互转换的实际意义

## 1. 实现模型与工具解耦

没有 MCP 时，每种工具可能都要适配不同模型：

```text
天气工具 × OpenAI
天气工具 × Claude
天气工具 × Gemini

数据库工具 × OpenAI
数据库工具 × Claude
数据库工具 × Gemini
```

随着模型和工具数量增加，适配组合会迅速增长。

引入 MCP 后，可以变成：

```text
OpenAI 适配器 ─┐
Claude 适配器  ├→ MCP → 多个 MCP Server
Gemini 适配器 ─┘
```

每个模型只需要实现模型工具调用格式与 MCP 之间的适配。

每个工具只需要实现一次 MCP Server。

因此，原来的多对多关系可以转变成围绕 MCP 的统一连接方式。

---

## 2. MCP 工具可以跨模型复用

一个文件系统 MCP Server 可以同时被不同模型宿主调用：

```text
OpenAI 模型
Claude 模型
Gemini 模型
本地模型
      ↓
同一个文件系统 MCP Server
```

MCP Server 不需要知道调用它的是哪一种模型。

只要宿主能够将模型的工具调用转换为 MCP 请求，就可以复用同一套工具。

---

## 3. 更换模型时不需要重写所有工具

例如原来的系统使用：

```text
OpenAI Function Calling ↔ MCP
```

后来更换为：

```text
Claude Tool Use ↔ MCP
```

只需要修改模型侧适配器，后面的 MCP Server、数据库服务和业务接口不需要全部重写。

所以：

> 模型可以替换，工具服务保持稳定。

---

## 4. 现有 Function Calling 工具可以包装成 MCP Server

如果一个项目里已经存在大量本地函数：

```python
search_order(order_id)
query_inventory(product_id)
generate_report(report_type)
```

可以在这些函数外面增加 MCP Server 封装：

```text
已有业务函数
      ↓
MCP Server 包装
      ↓
成为标准 MCP 工具
      ↓
供多个 AI 项目调用
```

这可以把原来只能在单一项目中使用的函数，转变为可跨项目复用的工具服务。

---

## 5. 统一鉴权、权限、日志和审计

工具调用经常涉及：

* 登录认证；
* 用户身份；
* 数据访问权限；
* API 密钥；
* 调用日志；
* 审计记录；
* 限流；
* 超时；
* 重试；
* 错误处理；
* 危险操作确认。

这些逻辑可以集中在 MCP Host、MCP Client 或 MCP Server 中，而不必由每个模型和项目分别实现。

例如模型产生：

```json
{
  "name": "delete_file",
  "arguments": {
    "path": "/data/test.txt"
  }
}
```

但应用程序可以在真正调用 MCP Server 之前检查：

* 当前用户有没有删除权限；
* 文件路径是否在允许范围；
* 是否需要用户确认；
* 是否需要记录审计日志；
* 是否允许该模型执行高风险操作。

模型只负责表达调用意图，程序负责保证执行安全。

---

# 十、为什么不让模型直接发送完整 MCP 请求

理论上可以提示模型生成：

```json
{
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "杭州"
    }
  }
}
```

但通常不应该让模型直接管理完整的 MCP 通信。

因为 MCP 还涉及：

* 连接建立；
* 初始化；
* 能力协商；
* 会话管理；
* 请求 ID；
* 网络传输；
* 鉴权；
* 超时；
* 重试；
* 取消；
* 通知；
* 错误恢复。

这些工作要求确定性和可靠性，更适合由程序处理，而不是交给概率生成的大模型。

合理的职责划分是：

```text
模型负责：
理解意图
选择工具
生成业务参数
理解工具结果
组织最终回答

程序负责：
协议转换
网络通信
参数校验
权限判断
工具执行
超时重试
错误处理
结果包装
```

---

# 十一、MCP 并不能保证完全匹配和完全正确

MCP 能统一协议，但不能保证所有工具调用百分之百正确。

即使结构符合 Schema，也可能存在语义错误。

例如：

```json
{
  "city": "浙江"
}
```

从参数类型看，`city` 是字符串，所以结构合法。

但“浙江”是省级行政区，不一定符合工具所期望的具体城市名称。

因此，Schema 可以保证：

```text
参数结构大致正确
```

但不能保证：

```text
参数业务含义一定正确
```

其他可能出现的问题还包括：

* 模型选错工具；
* 工具调用时机错误；
* 模型遗漏调用；
* 参数值不合理；
* 用户权限不足；
* MCP Server 执行失败；
* 外部 API 超时；
* 数据源返回错误；
* 不同 MCP Client 支持的能力不同；
* 不同模型支持的 JSON Schema 范围不同。

所以更准确的说法是：

> MCP 可以大幅减少模型、项目和工具之间的重复适配，但不能保证模型理解、业务语义和工具执行全部正确。

---

# 十二、Function Calling 与 MCP 的转换不一定完全无损

MCP 和 Function Calling 的能力范围并不完全一致。

普通 Function Calling 通常只关注：

* 工具定义；
* 工具调用；
* 工具结果。

MCP 可能还包含：

* Tools；
* Resources；
* Prompts；
* 图片、音频和资源内容；
* 结构化结果；
* 进度信息；
* 通知；
* 取消机制；
* 能力协商；
* 连接状态。

因此，把 MCP 能力转换成普通 Function Calling 时，部分 MCP 能力可能无法直接表达。

实际转换层可能需要处理：

```text
工具名称映射
JSON Schema 兼容
参数类型转换
调用 ID 映射
错误结构转换
文本和图片结果转换
权限和确认
进度信息
取消机制
模型能力差异
```

所以二者之间的转换，并不只是修改几个 JSON 字段名，而是完整的协议适配。

---

# 十三、最适合的工程架构

实际项目中，可以建立一套与具体模型厂商无关的内部工具表示：

```python
ToolDefinition(
    name="get_weather",
    description="查询指定城市的天气",
    input_schema={...}
)
```

然后分别进行转换：

```text
统一 ToolDefinition
      ├→ OpenAI Function Calling
      ├→ Claude Tool Use
      ├→ Gemini Function Calling
      └→ MCP Tool
```

实际调用也先统一成内部对象：

```text
OpenAI Function Call ─┐
Claude Tool Use       ├→ 统一 ToolCall → MCP tools/call
Gemini Function Call ─┘
```

这样核心业务逻辑就不会直接依赖某一家模型的格式。

模型接口变化时，只需要调整对应适配器。

---

# 十四、三种方式的整体对比

| 方式               | 主要解决的问题              | 格式由谁定义          | 稳定性   | 复用范围         |
| ---------------- | -------------------- | --------------- | ----- | ------------ |
| Prompt 自定义工具调用   | 让模型按约定文本表达工具调用       | 项目开发者           | 较低    | 通常局限于单个项目    |
| Function Calling | 让模型通过原生接口稳定地产生工具调用   | 模型厂商和开发者 Schema | 较高    | 通常围绕某个模型 API |
| MCP              | 让 AI 应用统一发现和调用外部工具服务 | MCP 协议          | 协议层较高 | 可跨项目、跨宿主、跨模型 |

---

# 十五、最终结论

可以把三者理解为三个层次：

```text
Prompt 自定义工具调用
解决：如何让模型按照开发者约定的文本格式表达工具调用

Function Calling
解决：模型如何通过模型厂商的原生接口稳定地产生工具调用

MCP
解决：AI 应用如何通过统一协议连接、发现和调用外部工具服务
```

Function Calling 和 MCP 的关系是：

```text
Function Calling 是模型侧的工具调用接口
MCP 是应用与工具服务之间的通用通信协议
中间适配层负责二者之间的翻译和桥接
```

Function Calling 与 MCP 互相转换的核心价值是：

* 让不同模型调用同一套 MCP 工具；
* 让同一个 MCP 工具跨项目复用；
* 更换模型时减少工具重写；
* 把项目内部函数封装成通用工具服务；
* 集中管理权限、鉴权、日志和错误处理；
* 让模型负责理解和决策，让程序负责可靠执行。

最简洁的概括是：

> Prompt 自定义工具调用是“自己约定模型输出格式”；Function Calling 是“模型厂商原生支持工具调用”；MCP 是“AI 应用与外部工具之间的统一连接协议”。

> Function Calling 负责让模型表达“我要调用什么工具”，MCP 负责让应用真正找到并调用这个工具，转换层负责把两边的格式和能力连接起来。
