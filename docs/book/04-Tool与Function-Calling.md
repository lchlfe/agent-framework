# 第 4 章：Tool 与 Function Calling

> 本章定位：把第 3 章的“会聊天的最小 Agent”推进到“能调用函数、查询系统、执行动作的 Agent”。

## 本章目标

学完本章后，你应能够：

- 解释 Tool、Function Calling、`AITool`、`AIFunction`、`AIFunctionFactory` 与 `FunctionInvokingChatClient` 的关系。
- 在 .NET Agent 中把一个 C# 方法暴露为可被模型调用的工具。
- 跑通仓库中已有的工具调用 sample，并理解一次工具调用从用户输入到工具结果再到最终回答的链路。
- 区分“仓库已有 sample 直接支持”“可基于 sample 改造”“本章理论延伸”的能力边界。
- 为综合案例“企业内部知识与工单协同 Agent 系统”增加第一个只读查询工具。

## 本章与前后章节的关系

- 前置知识：第 3 章已经跑通最小 `AIAgent`，理解 `RunAsync` / `RunStreamingAsync` 的基本调用方式。
- 本章解决：让 Agent 不只生成文本，而是能通过函数调用获得外部信息或触发业务动作。
- 后续衔接：第 5 章会继续讨论 Prompt、消息结构和最小交互模式；第 13 章会更深入展开 Tool、MCP、外部系统集成和企业级安全边界。

## 先建立整体认知

在 Agent 系统里，Tool 是 Agent 与外部世界交互的接口。模型本身不会真的访问天气系统、工单系统或数据库；它通常先根据用户问题决定“需要调用哪个工具、传哪些参数”，框架再执行本地函数或外部工具，最后把工具结果送回模型，让模型生成面向用户的回答。

本章先聚焦最小路径：把一个本地 C# 方法通过 `AIFunctionFactory.Create(...)` 包装成 `AIFunction`，再通过 `AsAIAgent(..., tools: [...])` 或 `ChatClientAgentOptions.ChatOptions.Tools` 提供给 Agent。更复杂的审批、MCP、OpenAPI、工作流工具、持久化与恢复，本章只点到为止，后续章节继续展开。

如果缺少 Tool 层，Agent 只能“根据已有上下文猜测”；加入 Tool 后，Agent 才能查询实时数据、调用业务服务、触发工单动作，并逐步演进为应用级系统。

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：
  - `dotnet/samples/01-get-started/02_add_tools/`
  - `dotnet/samples/02-agents/Agents/`
  - `dotnet/src/Microsoft.Agents.AI/`
- 项目 / Sample：
  - `dotnet/samples/01-get-started/02_add_tools/Program.cs`：最小函数工具示例，推荐首先阅读。
  - `dotnet/samples/02-agents/Agents/Agent_Step01_UsingFunctionToolsWithApprovals/Program.cs`：工具调用加人工审批。
  - `dotnet/samples/02-agents/Agents/Agent_Step09_AsFunctionTool/Program.cs`：把一个 Agent 暴露成 function tool。
  - `dotnet/samples/02-agents/Agents/Agent_Step11_Middleware/Program.cs`：函数调用中间件、结果覆盖、审批中间件。
  - `dotnet/samples/02-agents/Agents/Agent_Step19_InFunctionLoopCheckpointing/Program.cs`：工具调用循环中的逐服务调用持久化。
- 关键类型：
  - `AIAgent`
  - `AgentSession`
  - `ChatClientAgentRunOptions`
  - `AITool`
  - `AIFunction`
  - `AIFunctionFactory`
  - `ApprovalRequiredAIFunction`
  - `ToolApprovalRequestContent`
  - `FunctionCallContent`
  - `FunctionInvocationContext`
- 关键接口 / 外部依赖：
  - `IChatClient`
  - `FunctionInvokingChatClient`
  - `ChatMessage`
  - `AIContent`
- 关键扩展点：
  - `AsAIAgent(...)`
  - `BuildAIAgent(...)`
  - `AIAgent.AsBuilder().Use(...)` 中的 function invocation middleware
  - `AIAgent.AsAIFunction()`

说明：`AIFunction`、`AITool`、`FunctionInvokingChatClient` 等类型来自 `Microsoft.Extensions.AI` 相关包，仓库的 `dotnet/AGENTS.md` 对这些核心类型有简要说明；本仓库在 `dotnet/src/Microsoft.Agents.AI/FunctionInvocationDelegatingAgent*.cs` 中实现了 Agent 层函数调用中间件包装。

## 先跑通或先验证一个最小示例

### 示例目标

- 运行 `02_add_tools`，观察模型如何通过 `GetWeather` 工具回答“Amsterdam 天气如何”。
- 同时验证非流式 `RunAsync` 和流式 `RunStreamingAsync` 都可以使用工具。

### 运行前提

- 安装 .NET 10 SDK 或仓库要求的 SDK 版本。
- 已登录 Azure CLI：`az login`。
- 有可用的 Azure OpenAI endpoint 与 deployment。
- 设置环境变量：

```powershell
$env:AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT_NAME="gpt-5.4-mini"
```

注意：仓库 README 中提示，如果模型不支持 function calling，工具调用会不可用。例如 `dotnet/samples/02-agents/AgentProviders/Agent_With_ONNX/README.md` 明确说明 ONNX 不支持 function calling，传入的 function tools 会被忽略。

### 操作步骤

```powershell
cd D:\code\agent-framework\dotnet\samples\01-get-started\02_add_tools
dotnet build
dotnet run --no-build
```

也可以直接：

```powershell
dotnet run
```

### 预期现象

`Program.cs` 中的核心代码是：

```csharp
[Description("Get the weather for a given location.")]
static string GetWeather([Description("The location to get the weather for.")] string location)
    => $"The weather in {location} is cloudy with a high of 15°C.";

AIAgent agent = new AzureOpenAIClient(
    new Uri(endpoint),
    new DefaultAzureCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(instructions: "You are a helpful assistant", tools: [AIFunctionFactory.Create(GetWeather)]);
```

运行后，Agent 应该回答 Amsterdam 的天气，并且内容会引用工具返回的固定结果，例如 cloudy、15°C。

### 成功标志

- 程序能正常启动并输出 Agent 回答。
- 回答中包含或明显基于 `GetWeather` 的返回值：`cloudy with a high of 15°C`。
- 流式调用部分也能持续输出结果，而不是完全忽略工具。

## 核心概念拆解

### 1. Tool 是什么

Tool 是 Agent 可调用能力的抽象。它可以是本地函数、远程 API、另一个 Agent、MCP 工具、OpenAPI 工具，或者更复杂的业务动作。本章的最小 Tool 是 C# 静态方法 `GetWeather`。

### 2. Function Calling 是什么

Function Calling 是模型与工具之间的协作协议：模型先产生“我要调用某个函数及参数”的结构化意图，框架执行该函数，再把工具结果加入消息上下文，模型据此继续生成自然语言回答。它不是“模型直接执行 C# 代码”，真正执行的是框架侧的函数调用逻辑。

### 3. `AIFunction` 与 `AIFunctionFactory` 是什么

`AIFunction` 是一种特殊的 `AITool`，用于表示可调用的本地函数。`AIFunctionFactory.Create(GetWeather)` 会根据方法名、参数、返回值和 `[Description]` 元数据创建工具描述，让模型知道工具能做什么、参数是什么。

`[Description]` 很重要：它会影响模型是否能正确选择工具和填充参数。描述模糊时，模型可能不用工具、参数填错，或者选择错误工具。

### 4. `FunctionInvokingChatClient` 负责什么

`dotnet/AGENTS.md` 将 `FunctionInvokingChatClient` 描述为给 `IChatClient` 增加函数调用能力的 decorator。直观理解，它负责工具调用循环：

```text
用户消息
  -> Agent 调用 chat client
  -> 模型返回 function call
  -> FunctionInvokingChatClient 执行 AIFunction
  -> 工具结果写回消息
  -> 再次调用模型
  -> 得到最终自然语言回答
```

### 5. 审批工具与普通工具有什么不同

`Agent_Step01_UsingFunctionToolsWithApprovals` 使用：

```csharp
new ApprovalRequiredAIFunction(AIFunctionFactory.Create(GetWeather))
```

这表示工具调用前会产生 `ToolApprovalRequestContent`，程序需要让用户确认，再通过 `functionApprovalRequest.CreateResponse(...)` 把审批结果传回 Agent。这个 sample 是仓库已有的 Human-in-the-Loop 最小入口，但完整审批机制会在第 14 章深入展开。

### 6. 常见误解是什么

- 误解 1：只要写了函数，Agent 一定会调用。实际是否调用取决于模型、工具描述、Prompt、用户问题和 provider 能力。
- 误解 2：Tool 是安全的。实际上工具可能读写企业系统，必须做权限、参数校验、审计和审批。
- 误解 3：Function Calling 等于外部系统集成全部完成。本章只覆盖本地函数工具的最小路径，真实企业 API、MCP、OpenAPI 会在后续章节扩展。

## 结合源码理解执行过程

### 1. 入口在哪里

最小入口是 `dotnet/samples/01-get-started/02_add_tools/Program.cs`：

1. 定义 `GetWeather(string location)`。
2. 用 `AIFunctionFactory.Create(GetWeather)` 包装成工具。
3. 用 `AsAIAgent(..., tools: [...])` 创建带工具的 Agent。
4. 调用 `agent.RunAsync("What is the weather like in Amsterdam?")`。

### 2. 调用是如何继续流转的

简化链路如下：

```text
RunAsync("What is the weather like in Amsterdam?")
  -> ChatClientAgent 接收用户消息
  -> IChatClient 携带 instructions 和 tools 调用模型
  -> 模型决定调用 GetWeather(location: "Amsterdam")
  -> FunctionInvokingChatClient 执行 AIFunction
  -> GetWeather 返回固定天气文本
  -> 工具结果作为消息内容返回给模型
  -> 模型生成最终回答
  -> AgentResponse 输出到 Console
```

`Agent_Step19_InFunctionLoopCheckpointing/Program.cs` 的注释明确说明，当 Agent 使用工具时，`FunctionInvokingChatClient` 可能执行多轮循环：`service call -> tool execution -> service call`。该 sample 还展示了多个城市、多个工具调用时如何在每次 service call 后持久化聊天历史。

### 3. 关键对象分别负责什么

- `AIAgent`：Agent 抽象，暴露 `RunAsync` / `RunStreamingAsync`。
- `IChatClient`：底层聊天模型客户端抽象。
- `AITool`：工具的通用抽象。
- `AIFunction`：本地函数工具。
- `AIFunctionFactory`：从 C# 方法生成 `AIFunction`。
- `ApprovalRequiredAIFunction`：把普通函数工具包装为需要审批的工具。
- `ToolApprovalRequestContent`：模型想调用工具但需要用户批准时返回的内容。
- `FunctionInvocationContext`：函数调用中间件可以读取的上下文，包含函数、参数和调用内容。

### 4. 扩展点在哪里

仓库中的 `dotnet/src/Microsoft.Agents.AI/FunctionInvocationDelegatingAgentBuilderExtensions.cs` 提供了 function invocation middleware 扩展：

```csharp
public static AIAgentBuilder Use(
    this AIAgentBuilder builder,
    Func<AIAgent, FunctionInvocationContext,
    Func<FunctionInvocationContext, CancellationToken, ValueTask<object?>>,
    CancellationToken, ValueTask<object?>> callback)
```

它会检查 inner agent 是否支持 `FunctionInvokingChatClient`，否则抛出异常。对应实现 `FunctionInvocationDelegatingAgent.cs` 会把 `ChatClientAgentRunOptions` 中的 `Tools` 包装成 `MiddlewareEnabledFunction`，从而在真正执行 `AIFunction` 前后插入逻辑。

`Agent_Step11_Middleware/Program.cs` 展示了两个典型用途：

- 调用前后打印函数名。
- 对 `GetWeather` 的返回结果进行覆盖，把天气改成 sunny、25°C。

## 关键代码或配置讲解

### 代码片段 1：用 `[Description]` 描述函数和参数

- 位置：`dotnet/samples/01-get-started/02_add_tools/Program.cs`
- 作用：告诉模型这个函数的语义和参数含义。
- 为什么重要：模型选择工具时依赖这些元数据；描述越清楚，误调用概率越低。
- 改动影响：如果把描述改得过于笼统，例如 “Do something”，模型可能不知道何时调用。

### 代码片段 2：`AIFunctionFactory.Create(GetWeather)`

- 位置：`dotnet/samples/01-get-started/02_add_tools/Program.cs`
- 作用：把 C# 方法注册为 Agent 可见工具。
- 为什么重要：这是从“普通方法”到“模型可调用工具”的关键转换。
- 改动影响：如果不传入 `tools`，Agent 就无法通过 function calling 获取天气结果。

### 代码片段 3：审批包装 `ApprovalRequiredAIFunction`

- 位置：`dotnet/samples/02-agents/Agents/Agent_Step01_UsingFunctionToolsWithApprovals/Program.cs`
- 作用：让工具调用先返回审批请求，而不是立即执行。
- 为什么重要：企业场景中，写工单、重启服务、修改权限等动作不能让模型直接执行。
- 改动影响：加入审批后，调用链会多出 `ToolApprovalRequestContent` 与用户审批消息。

### 代码片段 4：函数调用中间件

- 位置：`dotnet/samples/02-agents/Agents/Agent_Step11_Middleware/Program.cs`
- 作用：拦截工具调用，记录日志、修改参数、覆盖结果、接入审批或风控。
- 为什么重要：企业级 Tool 层需要审计和治理，中间件是可维护的插入点。
- 改动影响：中间件中不调用 `next(...)` 就会替换原函数行为；调用顺序也会影响结果。

## 做一个最小改造实验

### 实验目标

把天气工具改造成“企业内部知识与工单协同 Agent 系统”的第一个只读工单查询工具。

### 改造点

以 `dotnet/samples/01-get-started/02_add_tools/Program.cs` 为基础，把 `GetWeather` 改成：

```csharp
[Description("Get the status of an internal support ticket by ticket id.")]
static string GetTicketStatus([Description("The internal ticket id, for example INC-1001.")] string ticketId)
    => ticketId.ToUpperInvariant() switch
    {
        "INC-1001" => "INC-1001 is assigned to IT Support. Current status: Waiting for user confirmation.",
        "INC-1002" => "INC-1002 is assigned to Network Team. Current status: Investigating VPN connectivity issue.",
        _ => $"Ticket {ticketId} was not found in the demo ticket system."
    };
```

然后将工具注册改为：

```csharp
tools: [AIFunctionFactory.Create(GetTicketStatus)]
```

并把测试问题改为：

```csharp
Console.WriteLine(await agent.RunAsync("Please check the status of ticket INC-1002."));
```

### 你应该观察什么

- Agent 是否主动调用 `GetTicketStatus`。
- 回答是否包含 `Network Team`、`Investigating VPN connectivity issue` 等工具返回信息。
- 如果把 ticket id 写得不清楚，例如“帮我看一下那个 VPN 工单”，模型可能无法正确传参，这说明真实系统需要上下文、澄清问题或检索能力。

### 可能出现的问题

- 模型没有调用工具：检查函数和参数的 `[Description]`，以及模型是否支持 function calling。
- 参数错误：把参数描述写得更具体，或在 Prompt 中要求“如果缺少 ticket id，先向用户追问”。
- 工具返回过长：真实系统中应返回结构化摘要，避免把大段日志直接塞回模型上下文。

## 本章在综合案例中的增量

综合案例名称保持为：**企业内部知识与工单协同 Agent 系统**。

- 本章之前：系统只有第 3 章级别的最小 Agent，只能回答用户自然语言问题，不能访问企业内部数据。
- 本章加入：一个只读工具 `GetTicketStatus(ticketId)`，用于查询演示工单状态。
- 修改模块：
  - Agent 创建时新增 `tools: [AIFunctionFactory.Create(GetTicketStatus)]`。
  - Prompt / instructions 可补充“当用户询问具体工单编号时，优先调用工单状态工具”。
- 能力变化：用户可以问“INC-1002 现在处理到哪了”，Agent 能基于工具结果回答，而不是凭空猜测。
- 如果不加入本章能力：综合案例无法接入工单系统，仍然停留在聊天机器人层面，后续的工单流转、审批和状态管理都缺少基础。

按照 README 的分类，本章综合案例内容边界如下：

1. 仓库已有 sample 直接支持：`02_add_tools` 已支持把本地 C# 函数注册为工具；`Agent_Step01_UsingFunctionToolsWithApprovals` 已支持审批型函数工具。
2. 可基于已有 sample 改造：把 `GetWeather` 改成 `GetTicketStatus`，用内存中的 `switch` 模拟工单系统。
3. 理论延伸：真实连接 ServiceNow、Jira、Azure DevOps、企业 CMDB 或数据库属于外部系统集成，会在第 13 章展开；写操作审批属于第 14 章重点。

## 常见问题与排错

### 问题 1：Agent 没有调用工具

- 现象：回答像普通聊天，没有引用工具返回值。
- 原因：模型不支持 function calling、工具没有注册、描述不清晰，或用户问题没有触发工具需求。
- 排查方法：确认 `tools: [AIFunctionFactory.Create(...)]` 是否传入；检查 sample README 中的 provider 限制；把用户问题改成明确需要工具的句子。
- 修复方式：换用支持 function calling 的模型；补充 `[Description]`；在 instructions 中说明何时必须使用工具。

### 问题 2：运行时报环境变量或认证错误

- 现象：`AZURE_OPENAI_ENDPOINT is not set`，或 Azure 认证失败。
- 原因：环境变量未设置，未执行 `az login`，或账号缺少 Azure OpenAI 权限。
- 排查方法：在 PowerShell 中执行 `echo $env:AZURE_OPENAI_ENDPOINT`；执行 `az account show`；检查角色是否包含 `Cognitive Services OpenAI Contributor`。
- 修复方式：设置 endpoint 和 deployment；重新登录 Azure CLI；在生产环境优先使用 Managed Identity 等明确凭据。

### 问题 3：审批型工具卡住

- 现象：`Agent_Step01_UsingFunctionToolsWithApprovals` 输出“please reply Y to approve”后不继续。
- 原因：sample 等待控制台输入；如果在自动化环境中运行，需要提供输入。
- 排查方法：观察是否出现 `ToolApprovalRequestContent`；确认控制台是否等待用户输入。
- 修复方式：手动输入 `Y`；自动化验证时通过标准输入传入 `Y`；服务化场景要结合持久化会话保存等待审批状态。

### 问题 4：函数调用中间件不生效或抛异常

- 现象：使用 `AsBuilder().Use(FunctionCallMiddleware)` 后报不支持。
- 原因：`FunctionInvocationDelegatingAgentBuilderExtensions.cs` 要求 inner agent 或包装链路支持 `FunctionInvokingChatClient`。
- 排查方法：查看异常信息是否包含 `FunctionInvokingChatClient`；确认使用的是 `ChatClientAgent` 相关路径。
- 修复方式：基于支持函数调用的 chat client 构建 Agent；不要把 function invocation middleware 套到不支持该能力的 Agent 上。

## 企业级落地提醒

- 稳定性：工具函数要设置超时、重试、熔断和降级策略。模型可能连续调用多个工具，长时间阻塞会放大端到端延迟。
- 可维护性：工具命名、参数、返回结构要稳定，避免频繁改动导致 Prompt 和评测集失效。
- 权限边界：不要因为模型“想调用”就允许执行。只读查询、低风险写操作、高风险动作应有不同授权和审批策略。
- 安全风险：对工具参数做白名单、长度、格式和权限校验，避免 Prompt Injection 诱导 Agent 调用越权工具。
- 成本控制：工具结果不要无节制塞入上下文；对大结果做摘要、分页或结构化裁剪。
- 可观测性：记录工具名、参数摘要、耗时、结果状态和调用链 ID，但避免记录敏感明文。
- 可测试性：为每个工具准备固定输入输出的单元测试和 Agent 级回归用例，验证模型是否按预期选择工具。

## 本章小结

- 本章最重要的结论 1：Tool 是 Agent 从“聊天”走向“做事”的关键接口；本地函数可通过 `AIFunctionFactory.Create(...)` 暴露给 Agent。
- 本章最重要的结论 2：Function Calling 的本质是模型提出结构化调用意图，框架执行工具并把结果回填给模型，不是模型直接执行代码。
- 本章最重要的结论 3：仓库已提供从最小工具、审批工具、Agent-as-function-tool、中间件到工具循环持久化的多个 sample；本章先掌握最小路径，后续再扩展到企业级外部系统和审批治理。

下一章会进入 Prompt、消息与最小交互模式。理解消息结构后，你会更清楚工具调用请求、工具结果、审批请求这些内容如何在对话上下文中流转。

## 本章建议产出物

- 一次 `dotnet/samples/01-get-started/02_add_tools` 的运行记录。
- 一张工具调用链图：用户消息 -> 模型 function call -> 工具执行 -> 工具结果 -> 最终回答。
- 一个最小改造实验记录：`GetWeather` 改为 `GetTicketStatus`。
- 一份问题清单：模型不调用工具、参数错误、审批卡住、认证失败、provider 不支持 function calling。

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
