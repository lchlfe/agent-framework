# 第 6 章：Agent 核心抽象

> 本章定位：从“会调用一个 Agent”升级到“看懂 Agent 抽象层、包装链路和扩展点”，为后续 Session、Workflow、多 Agent 协作打基础。

## 本章目标

学完本章后，你应能够：

- 说清 `AIAgent`、`AgentSession`、`AgentRunOptions`、`AgentResponse`、`AgentResponseUpdate` 的职责边界。
- 理解 `ChatClientAgent` 如何把 `IChatClient` 适配成 Agent。
- 理解 `AIAgentBuilder` 与 `DelegatingAIAgent` 如何组成 Agent 级中间件管道。
- 区分 Agent 层扩展、ChatClient 层扩展、Function invocation 层扩展、Context provider 扩展。
- 基于已有 sample 做一个最小的“企业内部知识与工单协同 Agent 系统”核心抽象增量。

## 本章与前后章节的关系

- 前置知识：第 3 章已经跑通最小 Agent；第 4 章知道 Tool / Function Calling；第 5 章理解 Prompt、`ChatMessage` 和最小交互链路。
- 本章解决：Agent、Abstractions、Builder、Extensions 到底是什么关系，以及一次 `RunAsync` 如何进入真实执行链路。
- 后续衔接：第 7 章会深入 Session、History、Context 与 Memory；第 8 章会把 Agent 放进 Workflow；第 9 章会讨论多个 Agent 如何协作。

## 先建立整体认知

在这个仓库里，Agent 不是一个“只能聊天的类”，而是一套围绕 `AIAgent` 组织的可组合抽象：

```text
调用方
  -> AIAgent.RunAsync / RunStreamingAsync
    -> 可选的 Agent 管道：AIAgentBuilder + DelegatingAIAgent
      -> 具体 Agent 实现：ChatClientAgent / FoundryAgent / 自定义 AIAgent / WorkflowHostAgent ...
        -> 底层模型、服务、工具、会话、上下文提供器
  <- AgentResponse / AgentResponseUpdate
```

本章要建立四个关键认知：

1. `AIAgent` 是统一入口，不等于某一个模型服务。
2. `Microsoft.Agents.AI.Abstractions` 定义了 Agent 能被调用、创建会话、序列化会话、返回响应的最小契约。
3. `AIAgentBuilder` 不是“创建 Prompt 的 Builder”，而是把一个 Agent 包装成多层管道的 Builder。
4. `Extensions` 的主要价值是把已有对象适配到 Agent 生态，例如 `IChatClient.AsAIAgent(...)`、`AIAgent.AsBuilder()`、`AIAgent.AsAIFunction()`。

如果缺少这一层抽象，应用代码会直接绑定某个模型 SDK 或某个服务 Agent，后续想加日志、审计、PII 过滤、审批、会话恢复、多 Agent 调用时都会变得困难。

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：
  - `dotnet/src/Microsoft.Agents.AI.Abstractions/`
  - `dotnet/src/Microsoft.Agents.AI/`
  - `dotnet/src/Microsoft.Agents.AI/ChatClient/`
  - `dotnet/samples/01-get-started/`
  - `dotnet/samples/02-agents/`
- 项目：
  - `Microsoft.Agents.AI.Abstractions`
  - `Microsoft.Agents.AI`
- Sample：
  - `dotnet/samples/01-get-started/01_hello_agent/Program.cs`
  - `dotnet/samples/02-agents/AgentProviders/Agent_With_CustomImplementation/Program.cs`
  - `dotnet/samples/02-agents/Agents/Agent_Step11_Middleware/Program.cs`
  - 可延伸参考：`dotnet/samples/02-agents/Agents/Agent_Step09_AsFunctionTool/Program.cs`
- 关键类型：
  - `AIAgent`
  - `DelegatingAIAgent`
  - `AIAgentBuilder`
  - `ChatClientAgent`
  - `ChatClientAgentOptions`
  - `AgentSession`
  - `AgentRunOptions`
  - `AgentResponse`
  - `AgentResponseUpdate`
  - `AIAgentMetadata`
- 关键扩展点：
  - `AIAgent.RunCoreAsync(...)`
  - `AIAgent.RunCoreStreamingAsync(...)`
  - `AIAgent.CreateSessionCoreAsync(...)`
  - `AIAgent.SerializeSessionCoreAsync(...)`
  - `AIAgent.DeserializeSessionCoreAsync(...)`
  - `AIAgent.GetService(...)`
  - `AIAgentBuilder.Use(...)`
  - `AIAgentBuilder.UseAIContextProviders(...)`
  - `AIAgentExtensions.AsBuilder(...)`
  - `AIAgentExtensions.AsAIFunction(...)`

推荐阅读顺序：先看 `AIAgent.cs`，再看 `DelegatingAIAgent.cs`，然后看 `AIAgentBuilder.cs`，最后结合 `ChatClientAgent.cs` 和 middleware sample 理解真实调用链。

## 先跑通或先验证一个最小示例

### 示例目标

验证同一个 `AIAgent` 抽象可以同时支持：

- 真实模型后端的 Agent，例如 `01_hello_agent`。
- 不依赖 AI 服务的自定义 Agent，例如 `UpperCaseParrotAgent`。
- 通过 Builder 包装出来的中间件 Agent，例如 `Agent_Step11_Middleware`。

### 运行前提

- 已安装仓库要求的 .NET SDK。
- 如果运行 Azure OpenAI sample，需要配置：
  - `AZURE_OPENAI_ENDPOINT`
  - `AZURE_OPENAI_DEPLOYMENT_NAME`
  - 可用的 Azure 身份认证，例如开发环境中的 `DefaultAzureCredential`。
- 如果只想验证抽象层，可以优先运行自定义 Agent sample：`dotnet/samples/02-agents/AgentProviders/Agent_With_CustomImplementation/Program.cs`。它不调用真实 AI，而是把用户输入转成大写。

### 操作步骤

在仓库根目录执行，按你本机项目文件实际名称进入对应 sample 目录：

```bash
cd dotnet/samples/02-agents/AgentProviders/Agent_With_CustomImplementation
dotnet run
```

然后再阅读或运行中间件 sample：

```bash
cd dotnet/samples/02-agents/Agents/Agent_Step11_Middleware
dotnet run
```

如果 Azure OpenAI 环境已准备好，也可以回到入门 sample：

```bash
cd dotnet/samples/01-get-started/01_hello_agent
dotnet run
```

### 预期现象

- 自定义 Agent sample 中，`UpperCaseParrotAgent` 会把输入消息转成大写作为响应。
- middleware sample 中，会在控制台看到类似 `Pii Middleware - Filtered Messages Pre-Run`、`Guardrail Middleware - Filtered messages Pre-Run`、`Function Name: ... Pre-Invoke` 的日志。
- hello agent sample 中，`AIAgent agent = ...AsAIAgent(...)` 后可以直接 `agent.RunAsync(...)` 和 `agent.RunStreamingAsync(...)`。

### 成功标志

你能指出：

- 调用方只依赖 `AIAgent`。
- 具体行为来自不同实现：`ChatClientAgent` 或自定义 `UpperCaseParrotAgent`。
- 中间件通过 `originalAgent.AsBuilder().Use(...).Build()` 包装出新的 Agent，而不是改动原始 Agent 源码。

## 核心概念拆解

### 1. `AIAgent` 是什么

`AIAgent` 位于 `dotnet/src/Microsoft.Agents.AI.Abstractions/AIAgent.cs`，是所有 Agent 的统一基类。它负责定义 Agent 对外能做什么：

- 身份与描述：`Id`、`Name`、`Description`。
- 运行入口：`RunAsync(...)`、`RunStreamingAsync(...)`。
- 会话入口：`CreateSessionAsync(...)`。
- 会话持久化入口：`SerializeSessionAsync(...)`、`DeserializeSessionAsync(...)`。
- 服务发现入口：`GetService(...)`。
- 当前运行上下文：`AIAgent.CurrentRunContext`。

它的边界也很清晰：

- 它不强制要求底层一定是 OpenAI、Azure OpenAI、Foundry 或本地模型。
- 它不负责替开发者验证和清洗用户输入；源码注释明确提醒消息会原样流向模型、上下文提供器、历史存储和工具。
- 它不规定会话一定存在本地还是服务端；具体由实现类决定。

`AIAgent` 把多个便捷重载最终收敛到两个核心抽象方法：

```csharp
protected abstract Task<AgentResponse> RunCoreAsync(...);
protected abstract IAsyncEnumerable<AgentResponseUpdate> RunCoreStreamingAsync(...);
```

因此，当你要实现一个全新 Agent 时，核心任务不是“继承一个聊天类”，而是实现同步响应、流式响应、会话创建与会话序列化的契约。

### 2. `AgentSession` 是什么

`AgentSession` 是 Agent 会话状态的承载对象。第 7 章会深入讲它，这里只需要掌握边界：

- `AIAgent` 的运行方法允许传入 `AgentSession? session`。
- 如果传入 `null`，具�� Agent 可以创建默认 session。
- 不同 Agent 实现通常要求自己的 session 类型，例如 `ChatClientAgent` 期望 `ChatClientAgentSession`，自定义 sample 中的 `UpperCaseParrotAgent` 期望 `CustomAgentSession`。
- 传错 session 类型会抛出“session type is not compatible”的错误。

在 `Agent_With_CustomImplementation` sample 中，`UpperCaseParrotAgent` 内部定义了：

```csharp
internal sealed class CustomAgentSession : AgentSession
```

并在 `RunCoreAsync` / `RunCoreStreamingAsync` 中检查传入 session 类型。这说明 session 是 Agent 的状态边界，不应在多个不兼容 Agent 之间随意复用。

### 3. `AgentRunOptions` 是什么

`AgentRunOptions` 是单次运行时的附加配置。它和 Agent 构造时配置不同：

- 构造时配置：定义 Agent 的默认行为，例如 `instructions`、默认 tools、name、description。
- 运行时配置：定义这一次调用要临时覆盖或追加什么，例如 `ChatClientAgentRunOptions` 中的 `ChatOptions`、临时 tools、`ChatClientFactory`。

在 middleware sample 中：

```csharp
var options = new ChatClientAgentRunOptions(new()
{
    Tools = [AIFunctionFactory.Create(GetWeather, name: nameof(GetWeather))]
});
```

这说明同一个 Agent 可以在某次调用中临时获得一个工具，而不必把工具永久加入 Agent 默认配置。

### 4. `AgentResponse` 与 `AgentResponseUpdate` 是什么

- `AgentResponse`：非流式调用的完整结果，通常由 `RunAsync` 返回。
- `AgentResponseUpdate`：流式调用中的增量结果，通常由 `RunStreamingAsync` 返回。

`ChatClientAgent.RunCoreAsync` 会把底层 `ChatResponse` 包装成 `AgentResponse`，并设置 `AgentId`、`ContinuationToken` 等信息。流式路径则把底层 `ChatResponseUpdate` 逐步转换为 `AgentResponseUpdate`。

### 5. `ChatClientAgent` 是什么

`ChatClientAgent` 位于 `dotnet/src/Microsoft.Agents.AI/ChatClient/ChatClientAgent.cs`，它的职责是把 `Microsoft.Extensions.AI.IChatClient` 适配为 `AIAgent`。

它主要处理：

- 默认 instructions、tools、name、description。
- `ChatOptions` 与运行时 options 的合并。
- session 创建和类型校验。
- 聊天历史与上下文提供器的加载。
- 调用底层 `IChatClient.GetResponseAsync(...)` 或 `GetStreamingResponseAsync(...)`。
- 将底层响应转换为 `AgentResponse` / `AgentResponseUpdate`。
- 通过 `GetService(...)` 暴露 `AIAgentMetadata`、`IChatClient`、`ChatOptions`、`ChatClientAgentOptions` 等服务。

所以当你看到 sample 中：

```csharp
AIAgent agent = new AzureOpenAIClient(...)
    .GetChatClient(deploymentName)
    .AsIChatClient()
    .AsAIAgent(instructions: "You are good at telling jokes.", name: "Joker");
```

可以理解为：Azure OpenAI SDK 创建了 chat client，`AsIChatClient()` 把它接入 `Microsoft.Extensions.AI`，`AsAIAgent(...)` 再把它包装为 `ChatClientAgent` 这一类 Agent。

### 6. `DelegatingAIAgent` 是什么

`DelegatingAIAgent` 位于 `dotnet/src/Microsoft.Agents.AI.Abstractions/DelegatingAIAgent.cs`，是 Agent 级装饰器基类。

它默认把所有操作透明转发给内部 Agent：

- `CreateSessionCoreAsync` 转发到 `InnerAgent.CreateSessionAsync`。
- `SerializeSessionCoreAsync` / `DeserializeSessionCoreAsync` 转发给内部 Agent。
- `RunCoreAsync` 转发给 `InnerAgent.RunAsync`。
- `RunCoreStreamingAsync` 转发给 `InnerAgent.RunStreamingAsync`。
- `GetService` 先看自己，再向内部 Agent 传递。

这使你可以只重写你关心的一小段行为，例如运行前过滤消息、运行后记录日志、拦截工具审批，而不需要重新实现完整 Agent。

### 7. `AIAgentBuilder` 是什么

`AIAgentBuilder` 位于 `dotnet/src/Microsoft.Agents.AI/AIAgentBuilder.cs`，它提供 Agent 管道构建能力。

关键点：

- 构造函数接收一个 inner agent 或 inner agent factory。
- `Use(...)` 添加中间层。
- `Build(...)` 从内向外包装 Agent。
- 源码中特意说明：为了符合直觉，先添加的 factory 会成为最外层。

例如：

```csharp
var middlewareEnabledAgent = originalAgent
    .AsBuilder()
    .Use(FunctionCallMiddleware)
    .Use(FunctionCallOverrideWeather)
    .Use(PIIMiddleware, null)
    .Use(GuardrailMiddleware, null)
    .Build();
```

这不是创建一个全新模型，而是在 `originalAgent` 外面叠加多层 Agent 能力。

### 8. `Extensions` 是什么

本章关注两类 Extensions：

- `AIAgentExtensions`：位于 `dotnet/src/Microsoft.Agents.AI/AgentExtensions.cs`。
  - `AsBuilder(this AIAgent innerAgent)`：把已有 Agent 转成 Builder 起点。
  - `AsAIFunction(this AIAgent agent, ...)`：把 Agent 暴露为一个 `AIFunction`，让其他 Agent 通过工具调用它。
- ChatClient 相关扩展：在 `Microsoft.Agents.AI` / `Microsoft.Agents.AI.OpenAI` 等目录中，用于把 `IChatClient`、OpenAI client、Azure OpenAI client 等转换为 Agent。

`AsAIFunction` 是理解多 Agent 协作的关键之一：它把一个 Agent 包装成标准 function，使另一个 Agent 可以把它当 Tool 调用。但源码注释也提醒，如果传入固定 session，这个 function 是有状态的，不应在多个并发会话中随意复用。

## 结合源码理解执行过程

### 1. 入口在哪里

最小入口通常出现在 sample：

```csharp
Console.WriteLine(await agent.RunAsync("Tell me a joke about a pirate."));
```

这会进入 `AIAgent.RunAsync(string message, ...)`：

1. 校验字符串非空。
2. 把字符串包装成 `new ChatMessage(ChatRole.User, message)`。
3. 调用 `RunAsync(ChatMessage, ...)`。
4. 再收敛到 `RunAsync(IEnumerable<ChatMessage>, ...)`。
5. 设置 `AIAgent.CurrentRunContext`。
6. 调用具体实现的 `RunCoreAsync(...)`。

### 2. 调用是如何继续流转的

如果具体实现是 `ChatClientAgent`，核心链路是：

```text
AIAgent.RunAsync(string)
  -> AIAgent.RunAsync(IEnumerable<ChatMessage>)
    -> ChatClientAgent.RunCoreAsync(...)
      -> PrepareSessionAndMessagesAsync(...)
        -> 合并 ChatOptions / AgentRunOptions
        -> 创建或校验 ChatClientAgentSession
        -> 加载 ChatHistoryProvider
        -> 调用 AIContextProvider
      -> ApplyRunOptionsTransformations(...)
      -> IChatClient.GetResponseAsync(...)
      -> NotifyProvidersOfNewMessagesAtEndOfRunAsync(...)
      -> new AgentResponse(chatResponse)
```

如果是流式调用，则是：

```text
AIAgent.RunStreamingAsync(string)
  -> AIAgent.RunStreamingAsync(IEnumerable<ChatMessage>)
    -> ChatClientAgent.RunCoreStreamingAsync(...)
      -> PrepareSessionAndMessagesAsync(...)
      -> IChatClient.GetStreamingResponseAsync(...)
      -> yield return AgentResponseUpdate
      -> 流结束后通知历史与上下文提供器
```

### 3. 关键对象分别负责什么

- `ChatMessage`：输入和输出消息的通用结构。
- `AgentSession`：一次会话的状态边界。
- `AgentRunOptions`：单次运行的临时配置。
- `ChatOptions`：传给底层 `IChatClient` 的模型调用配置。
- `ChatHistoryProvider`：负责加载和保存会话历史。
- `AIContextProvider` / `MessageAIContextProvider`：负责在运行前后补充上下文或处理结果。
- `AIAgent.CurrentRunContext`：让下游装饰器或 ChatClient 管道知道当前 Agent、session、请求消息和运行选项。

### 4. 扩展点在哪里

常见扩展位置如下：

| 扩展位置 | 适合做什么 | 典型源码 / sample |
| --- | --- | --- |
| 继承 `AIAgent` | 实现完全自定义 Agent 后端 | `Agent_With_CustomImplementation/Program.cs` |
| 继承 / 使用 `DelegatingAIAgent` | Agent 层装饰、审计、过滤、降级 | `DelegatingAIAgent.cs`、`AIAgentBuilder.Use(...)` |
| `AIAgentBuilder.Use(...)` | ���速插入中间件 | `Agent_Step11_Middleware/Program.cs` |
| `ChatClientAgentOptions` | 配置 instructions、tools、history、context providers | `ChatClientAgent.cs` |
| `ChatClientAgentRunOptions` | 单次调用临时工具、ChatOptions、ChatClientFactory | `Agent_Step11_Middleware/Program.cs` |
| `AsAIFunction(...)` | 把一个 Agent 暴露给另一个 Agent 调用 | `AgentExtensions.cs` |
| `GetService(...)` | 从 Agent 管道中取内部服务或元数据 | `AIAgent.cs`、`ChatClientAgent.cs` |

## 关键代码或配置讲解

### 代码片段 1：最小 Agent 创建

来自 `dotnet/samples/01-get-started/01_hello_agent/Program.cs`：

```csharp
AIAgent agent = new AzureOpenAIClient(
        new Uri(endpoint),
        new DefaultAzureCredential())
    .GetChatClient(deploymentName)
    .AsIChatClient()
    .AsAIAgent(instructions: "You are good at telling jokes.", name: "Joker");
```

- 作用：把 Azure OpenAI chat client 适配成 `AIAgent`。
- 为什么重要：应用后续只依赖 `AIAgent`，而不是直接绑定 Azure OpenAI SDK。
- 改动它会影响什么：改 `instructions` 会影响系统行为；改 `name` 会影响响应消息的 author、日志和多 Agent 可识别性。

### 代码片段 2：自定义 Agent 实现

来自 `dotnet/samples/02-agents/AgentProviders/Agent_With_CustomImplementation/Program.cs`：

```csharp
internal sealed class UpperCaseParrotAgent : AIAgent
{
    public override string? Name => "UpperCaseParrotAgent";

    protected override ValueTask<AgentSession> CreateSessionCoreAsync(...)
        => new(new CustomAgentSession());

    protected override async Task<AgentResponse> RunCoreAsync(...)
    {
        session ??= await this.CreateSessionAsync(cancellationToken);
        ...
        return new AgentResponse
        {
            AgentId = this.Id,
            ResponseId = Guid.NewGuid().ToString("N"),
            Messages = responseMessages
        };
    }
}
```

- 作用：说明 `AIAgent` 并不要求一定调用 LLM；只要满足契约，就可以是任意执行逻辑。
- 为什么重要：企业里可以把内部规则引擎、检索服务、审批系统包装成 Agent。
- 改动它会影响什么：如果不正确处理 session 或 response messages，后续历史、流式输出、多 Agent 组合都会出现异常。

### 代码片段 3：Agent 管道中间件

来自 `dotnet/samples/02-agents/Agents/Agent_Step11_Middleware/Program.cs`：

```csharp
var middlewareEnabledAgent = originalAgent
    .AsBuilder()
    .Use(FunctionCallMiddleware)
    .Use(FunctionCallOverrideWeather)
    .Use(PIIMiddleware, null)
    .Use(GuardrailMiddleware, null)
    .Build();
```

- 作用：在不修改 `originalAgent` 的情况下叠加函数调用日志、工具结果覆盖、PII 过滤、护栏等能力。
- 为什么重要：企业级 Agent 需要把治理能力做成可复用管道，而不是散落在业务代码里。
- 改动它会影响什么：中间件顺序会影响输入输出处理顺序；某个中间件如果不调用 `innerAgent.RunAsync(...)`，链路会被截断。

## 做一个最小改造实验

### 实验目标

把主案例“企业内部知识与工单协同 Agent 系统”从前几章的最小聊天 / Tool 原型，升级为具备核心抽象边界的 Agent：

- 对外只暴露 `AIAgent`。
- 内部通过 Builder 增加基础治理中间件。
- 为后续第 7 章的会话、记忆与上下文管理预留边界。

### 改造点

基于 `Agent_Step11_Middleware` 的思路，在你的实验分支中构造一个支持工单场景的 Agent 名称与 Prompt：

```csharp
var supportAgent = azureOpenAIClient.AsIChatClient()
    .AsAIAgent(
        instructions: "你是企业内部知识与工单协同 Agent。优先澄清问题，必要时调用工具查询工单或知识。不要编造内部政策。",
        name: "EnterpriseSupportAgent",
        description: "Assists employees with internal knowledge and support tickets.");

var governedSupportAgent = supportAgent
    .AsBuilder()
    .Use(GuardrailMiddleware, null)
    .Use(PIIMiddleware, null)
    .Build();
```

如果你暂时没有 Azure OpenAI 环境，可以把 `supportAgent` 换成 `UpperCaseParrotAgent`，仍然可以验证 `AIAgentBuilder` 的包装链路。

### 你应该观察什么

- 业务调用方变量类型仍然是 `AIAgent`。
- 中间件日志能证明调用先进入外层 Agent，再进入内层 Agent。
- 当输入包含敏感信息或禁止词时，中间件会改变传给内层 Agent 的消息。
- 如果改动 `Use(...)` 顺序，日志顺序与过滤效果可能发生变化。

### 可能出现的问题

- 如果中间件没有调用 `innerAgent.RunAsync(...)`，底层模型不会被调用。
- 如果中间件创建新的 `ChatMessage` 时丢失 `Role`、`Contents` 或 `AuthorName`，模型输入和响应会异常。
- 如果把某个 Agent 的 session 传给不兼容 Agent，会触发 session 类型错误。

## 本章在综合案例中的增量

主案例统一为：**企业内部知识与工单协同 Agent 系统**。

在第 3-5 章结束时，它大致处于“最小可运行原型”状态：有一个 Agent、基本 Prompt、可能有一个查询类 Tool，但还没有清晰的抽象边界。

本章新增的是“Agent 核心抽象层”：

- 用户入口不再直接依赖某个具体模型 SDK，而是依赖 `AIAgent`。
- Agent 决策层先用 `name`、`description`、`instructions` 明确身份与能力边界。
- 治理层通过 `AIAgentBuilder` 插入 PII 过滤、护栏、函数调用日志等中间件。
- 工具层可以继续使用前面章节的 Tool，但要明确哪些工具是 Agent 默认 tools，哪些是单次 `AgentRunOptions` 注入。
- 后续知识层、状态层、流程层都围绕 `AgentSession`、`AIContextProvider`、Workflow 等扩展，而不是推翻现有调用入口。

加入这一章能力后，系统从“能跑的 demo”变成“可继续演进的应用骨架”。如果缺少这一层，后续加 Memory、Workflow、多 Agent、审批和部署时，会不断改调用方和业务代码，难以维护。

当前新增能力与仓库源码关系：

1. 仓库已有 sample 直接支持：`01_hello_agent` 展示 Agent 创建；`Agent_Step11_Middleware` 展示 Builder 和中间件；`Agent_With_CustomImplementation` 展示自定义 Agent。
2. 可以基于已有 sample 做最小改造：把 sample 的 `Joker`、weather 场景替换为 `EnterpriseSupportAgent` 与工单 / 知识查询工具。
3. 仓库未直接给出现成完整主案例：企业内部知识与工单系统需要在后续章节继续补齐会话、RAG、Workflow、审批、部署、评测和安全治理。

## 常见问题与排错

### 问题 1：`RunAsync` 看起来没有进入我的中间件

- 现象：控制台没有打印中间件日志，或过滤逻辑没生效。
- 原因：调用的可能还是 `originalAgent`，而不是 `.AsBuilder().Use(...).Build()` 返回的新 Agent。
- 排查方法：检查变量名和调用点，确认执行的是 `middlewareEnabledAgent.RunAsync(...)`。
- 修复方式：把包装后的 Agent 作为业务层唯一依赖传递，避免同时保留多个容易混淆的 Agent 变量。

### 问题 2：session 类型不兼容

- 现象：抛出类似“provided session type is not compatible with this agent”的异常。
- 原因：不同 Agent 实现有自己的 session 类型，例如 `ChatClientAgentSession` 与自定义 `CustomAgentSession` 不能随意混用。
- 排查方法：查看 session 是由哪个 Agent 的 `CreateSessionAsync` 创建的，再看当前调用的是哪个 Agent。
- 修复方式：由即将运行的 Agent 创建 session；如果需要跨 Agent 协作，不要直接共享不兼容 session，应设计上层会话映射。

### 问题 3：`AsAIFunction` 后并发调用行为异常

- 现象：多个会话使用同一个 agent function 后上下文串扰或结果不可预测。
- 原因：`AsAIFunction` 如果传入固定 `AgentSession`，生成的 function 是有状态的；源码注释提醒不要在并发会话或并行 function calls 中复用同一个 session。
- 排查方法：检查 `agent.AsAIFunction(..., session: someSession)` 是否被注册为全局共享工具。
- 修复方式：无状态场景不传 session；有状态场景为每个用户 / 会话创建独立 function 或在外层做并发隔离。

### 问题 4：中间件顺序导致结果不符合预期

- 现象：PII 已被过滤，后续日志看不到原始输入；或 guardrail 先把内容替换，PII 过滤不再匹配。
- 原因：`AIAgentBuilder.Build` 会把先添加的 factory 放在更外层，顺序会影响前处理和后处理。
- 排查方法：在每个 `Use(...)` 的 middleware 中打印 Pre / Post 日志，观察实际顺序。
- 修复方式：把安全拦截、审计、业务改写等能力按明确策略排序，并写单元测试覆盖。

### 问题 5：以为 `AIAgent` 会自动做安全校验

- 现象：用户输入、工具返回或 LLM 输出包含危险内容，但仍被继续传递或渲染。
- 原因：`AIAgent` 源码注释明确说明框架不会自动验证或清洗消息内容。
- 排查方法：检查输入入口、工具调用参数、HTML 渲染、SQL / Shell 执行等边界是否有显式校验。
- 修复方式：在 Agent 管道、Tool 层、输出渲染层增加防御；高风险工具必须引入审批或权限检查。

## 企业级落地提醒

- 稳定性：不要把所有逻辑写在一个巨大 Agent 中。底层模型适配、会话、工具、治理中间件应分层，方便替换和回滚。
- 可维护性：业务代码尽量依赖 `AIAgent`，把具体 provider 选择放在组合根或工厂里。
- 权限边界：`AsAIFunction` 能让 Agent 调 Agent，但也会扩大能力边界；必须限制哪些 Agent 能调用哪些工具或子 Agent。
- 安全风险：系统消息必须由开发者控制；用户消息、工具结果和 LLM 输出都应视为不可信。
- 成本控制：Builder 中间件适合加入日志、token 统计、模型路由、降级策略，但不要在每层重复调用模型。
- 可观测性：优先使用中间件记录 AgentId、AgentName、session id、tool name、异常与耗时，为第 17 章可观测性打基础。
- 可测试性：自定义 `UpperCaseParrotAgent` 这种无模型实现很适合做单元测试替身，验证调用链和 session 逻辑。
- 持久化与恢复：本章只讲抽象入口；真实持久化、恢复、长期记忆要等第 7 章和第 10 章展开。

## 本章小结

- 本章最重要的结论 1：`AIAgent` 是统一调用契约，负责运行、流式运行、会话创建、会话序列化和服务发现，但不绑定具体模型服务。
- 本章最重要的结论 2：`ChatClientAgent` 把 `IChatClient` 适配成 Agent，并处理 options、history、context providers、底层响应转换等执行链路。
- 本章最重要的结论 3：`AIAgentBuilder` 与 `DelegatingAIAgent` 是实现 Agent 管道和企业级治理扩展的关键，不应只把它理解成语法糖。

下一章将深入 `Session`、`History`、`Context` 与 `Memory`。理解本章后，你会更容易看懂为什么 session 是 Agent 的状态边界，以及为什么上下文注入应该走明确的 provider / middleware，而不是随意拼接字符串。

## 本章建议产出物

- 一次 `Agent_With_CustomImplementation` sample 运行记录。
- 一张 `AIAgent.RunAsync -> ChatClientAgent.RunCoreAsync -> IChatClient.GetResponseAsync` 调用链图。
- 一个把 `EnterpriseSupportAgent` 包装为治理管道的最小改造实验记录。
- 一份 session 类型兼容性与中间件顺序问题清单。

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
