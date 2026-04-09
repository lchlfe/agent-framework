# 第 5 章：Prompt、消息与最小交互模式

> 本章定位：把第 3 章的“能跑起来”和第 4 章的“会调用工具”收束成一个可解释、可改造、可排错的最小交互链路：Prompt 如何进入 Agent，用户输入如何变成消息，非流式与流式输出如何回到调用方。

## 本章目标

学完本章后，你应能够：

- 说清楚 `instructions`、`ChatMessage`、`AgentResponse`、`AgentResponseUpdate` 在一次 Agent 调用中的职责边界。
- 基于仓库真实 sample 跑通非流式与流式两种最小交互模式。
- 解释 `AIAgent.RunAsync(...)`、`AIAgent.RunStreamingAsync(...)` 到 `ChatClientAgent`、`IChatClient` 的执行链路。
- 知道何时只传字符串，何时显式构造 `ChatMessage`，何时创建 `AgentSession`。
- 在综合案例“企业内部知识与工单协同 Agent 系统”中补上系统 Prompt、消息输入、响应输出和流式输出的最小闭环。

## 本章与前后章节的关系

- 前置知识：第 3 章已经跑通最小 Agent；第 4 章已经知道 Tool / Function Calling 能让 Agent 执行动作。
- 本章解决：把“发一句话得到一句回答”的黑盒过程拆开，明确 Prompt、消息、响应、流式更新之间如何协作。
- 后续衔接：第 6 章会进一步展开 Agent 核心抽象；第 7 章会把本章的单轮/简单多轮消息升级为 Session、History、Context 与 Memory 体系。

## 先建立整体认知

在 Agent Framework 的最小交互中，可以先把链路理解为：

```text
开发者配置 instructions
        ↓
用户输入 string 或 ChatMessage
        ↓
AIAgent.RunAsync / RunStreamingAsync
        ↓
ChatClientAgent 准备 Session、ChatOptions、历史消息
        ↓
IChatClient.GetResponseAsync / GetStreamingResponseAsync
        ↓
AgentResponse 或 AgentResponseUpdate 返回给调用方
```

这里最容易混淆的是：

- **Prompt 不只是一段用户问题**：在 sample 中，`instructions: "You are good at telling jokes."` 是系统指令，决定 Agent 的角色和行为边界。
- **用户输入会被包装为消息**：`AIAgent.RunAsync(string message, ...)` 内部会创建 `new ChatMessage(ChatRole.User, message)`。
- **非流式响应是完整结果**：`AgentResponse` 通常包含一个或多个 `ChatMessage`，可用 `Text` 或 `ToString()` 取文本。
- **流式响应是增量更新**：`AgentResponseUpdate` 表示响应片段，适合边生成边显示。
- **Session 决定上下文是否延续**：不传 `AgentSession` 时是最小单次调用；传入同一个 session 时，后续调用可以保留对话上下文。

如果缺少这一层认知，后续做多轮会话、工具审批、RAG 或工作流时，很容易把系统指令、用户消息、工具消息、历史消息混在一起，导致 Prompt Injection、上下文丢失、响应截断或流式 UI 无法正确聚合。

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：
  - `dotnet/samples/01-get-started/01_hello_agent/`
  - `dotnet/samples/01-get-started/03_multi_turn/`
  - `dotnet/src/Microsoft.Agents.AI.Abstractions/`
  - `dotnet/src/Microsoft.Agents.AI/ChatClient/`
- 项目：
  - `dotnet/samples/01-get-started/01_hello_agent/01_hello_agent.csproj`
  - `dotnet/samples/01-get-started/03_multi_turn/03_multi_turn.csproj`
  - `dotnet/src/Microsoft.Agents.AI.Abstractions/Microsoft.Agents.AI.Abstractions.csproj`
  - `dotnet/src/Microsoft.Agents.AI/Microsoft.Agents.AI.csproj`
- Sample：
  - `dotnet/samples/01-get-started/01_hello_agent/Program.cs`：最小非流式与流式调用。
  - `dotnet/samples/01-get-started/03_multi_turn/Program.cs`：使用 `AgentSession` 保留多轮上下文。
- 关键类型：
  - `AIAgent`：Agent 调用入口，定义 `RunAsync`、`RunStreamingAsync`、`CreateSessionAsync` 等抽象。
  - `ChatClientAgent`：基于 `IChatClient` 的 Agent 实现。
  - `ChatMessage` / `ChatRole`：来自 `Microsoft.Extensions.AI`，表示带角色的消息。
  - `AgentResponse`：非流式调用返回的完整响应。
  - `AgentResponseUpdate`：流式调用返回的增量响应片段。
  - `AgentSession` / `ChatClientAgentSession`：会话上下文载体。
- 关键接口：
  - `IChatClient`：实际向底层模型服务发起 Chat 请求的接口。
- 关键扩展点：
  - `AsAIAgent(...)`：把 `ChatClient` 适配成 `AIAgent`。
  - `ChatClientAgentOptions.ChatOptions.Instructions`：保存系统指令。
  - `RunAsync(IEnumerable<ChatMessage> ...)`：显式传入消息集合。
  - `RunStreamingAsync(IEnumerable<ChatMessage> ...)`：显式传入消息集合并接收流式更新。

## 先跑通或先验证一个最小示例

### 示例目标

- 验证同一个 Agent 可以用非流式和流式两种方式回答。
- 观察 `instructions` 改变后输出风格如何变化。
- 验证 `AgentSession` 对多轮上下文的影响。

### 运行前提

- 已安装仓库要求的 .NET SDK。
- 已配置 Azure OpenAI 相关环境变量：
  - `AZURE_OPENAI_ENDPOINT`
  - `AZURE_OPENAI_DEPLOYMENT_NAME`，sample 中默认值为 `gpt-5.4-mini`。
- 本地身份可被 `DefaultAzureCredential` 使用。sample 注释也提醒：`DefaultAzureCredential` 方便开发，但生产环境应考虑使用更明确的凭据，例如 Managed Identity。

### 操作步骤

1. 进入最小 Agent sample：

   ```powershell
   cd D:\code\agent-framework\dotnet\samples\01-get-started\01_hello_agent
   dotnet run
   ```

2. 观察 `Program.cs` 中的核心配置：

   ```csharp
   AIAgent agent = new AzureOpenAIClient(
       new Uri(endpoint),
       new DefaultAzureCredential())
       .GetChatClient(deploymentName)
       .AsAIAgent(instructions: "You are good at telling jokes.", name: "Joker");
   ```

3. 观察非流式调用：

   ```csharp
   Console.WriteLine(await agent.RunAsync("Tell me a joke about a pirate."));
   ```

4. 观察流式调用：

   ```csharp
   await foreach (var update in agent.RunStreamingAsync("Tell me a joke about a pirate."))
   {
       Console.WriteLine(update);
   }
   ```

5. 再进入多轮对话 sample：

   ```powershell
   cd D:\code\agent-framework\dotnet\samples\01-get-started\03_multi_turn
   dotnet run
   ```

6. 关注 `AgentSession` 的用法：

   ```csharp
   AgentSession session = await agent.CreateSessionAsync();
   Console.WriteLine(await agent.RunAsync("Tell me a joke about a pirate.", session));
   Console.WriteLine(await agent.RunAsync("Now add some emojis to the joke and tell it in the voice of a pirate's parrot.", session));
   ```

### 预期现象

- `01_hello_agent` 会输出一次完整回答，然后再以流式更新形式输出一次回答。
- `03_multi_turn` 第二轮请求会引用第一轮的上下文，因为两次调用复用了同一个 `AgentSession`。
- 如果环境变量或身份未配置正确，程序会在创建客户端或调用模型时失败。

### 成功标志

- `RunAsync` 能得到完整文本。
- `RunStreamingAsync` 的 `await foreach` 能逐步打印 `AgentResponseUpdate` 的文本片段。
- 多轮 sample 中第二次请求能理解 “the joke” 指的是上一轮的海盗笑话。

## 核心概念拆解

### 1. Prompt / instructions 是什么

在 `01_hello_agent/Program.cs` 中，Prompt 的系统部分通过 `AsAIAgent(instructions: ..., name: ...)` 传入：

```csharp
.AsAIAgent(instructions: "You are good at telling jokes.", name: "Joker");
```

在 `ChatClientAgent` 构造函数中，这个参数被放进 `ChatClientAgentOptions.ChatOptions.Instructions`：

```csharp
ChatOptions = (tools is null && string.IsNullOrWhiteSpace(instructions)) ? null : new ChatOptions
{
    Tools = tools,
    Instructions = instructions
}
```

它的职责是告诉模型“你是谁、你应该如何回答、你有什么边界”。本章建议把它称为**系统 Prompt / instructions**，不要和最终用户输入混为一谈。

### 2. ChatMessage 是什么

`ChatMessage` 是带角色的消息对象，常见角色包括 `User`、`Assistant`、`System`、`Tool` 等。本章 sample 里直接传字符串：

```csharp
await agent.RunAsync("Tell me a joke about a pirate.")
```

但在 `AIAgent.RunAsync(string message, ...)` 源码中，它会被包装为：

```csharp
return this.RunAsync(new ChatMessage(ChatRole.User, message), session, options, cancellationToken);
```

因此，字符串输入不是直接扔给模型，而是变成了一个 `ChatRole.User` 的消息。后续如果你要显式控制角色、一次性传多条消息、混合上下文片段，就应使用 `RunAsync(IEnumerable<ChatMessage> ...)`。

### 3. AgentResponse 是什么

`AgentResponse` 是非流式调用的返回对象。源码位置：`dotnet/src/Microsoft.Agents.AI.Abstractions/AgentResponse.cs`。

关键点：

- `Messages`：保存响应消息集合。
- `Text`：把所有 `TextContent` 拼接成文本。
- `ToString()`：返回 `Text`，所以 sample 可以直接 `Console.WriteLine(await agent.RunAsync(...))`。
- `Usage`、`FinishReason`、`ResponseId`、`RawRepresentation`：用于诊断、成本、截断、底层 provider 调试。

源码中明确说明：典型响应可能只有一条消息，但复�� Agent 可能产生多条消息，例如工具调用、RAG 检索或中间步骤。

### 4. AgentResponseUpdate 与 Streaming 是什么

`AgentResponseUpdate` 是流式响应片段。源码位置：`dotnet/src/Microsoft.Agents.AI.Abstractions/AgentResponseUpdate.cs`。

它可以理解为“正在形成的 `AgentResponse` 的一块增量”。关键字段包括：

- `Role`：该片段所属角色。
- `Contents`：该片段的内容集合。
- `Text`：该片段中的文本。
- `MessageId`：用于把多个 update 归并到同一条消息。
- `ContinuationToken`：支持后台响应或断点续流的 provider 可使用。
- `FinishReason`：表示响应结束原因。

sample 中的：

```csharp
await foreach (var update in agent.RunStreamingAsync("Tell me a joke about a pirate."))
{
    Console.WriteLine(update);
}
```

之所以能打印文本，是因为 `AgentResponseUpdate.ToString()` 返回 `Text`。

### 5. Session 是什么

`03_multi_turn/Program.cs` 使用：

```csharp
AgentSession session = await agent.CreateSessionAsync();
```

同一个 `session` 被传入两次 `RunAsync` 或 `RunStreamingAsync`，使 Agent 可以保留上下文。`AIAgent` 源码注释说明：如果 `session` 为 `null`，调用时会创建新会话；如果提供 session，响应消息会更新到会话中。

本章只需要建立“Session 让上下文延续”的最小认知。更细的 History、Context、Memory 会在第 7 章展开。

### 6. 它们之间如何协作

一次非流式最小调用可以拆成：

```text
AsAIAgent(instructions, name)
  -> 创建 ChatClientAgent，保存 Instructions / Name
RunAsync("用户问题")
  -> AIAgent 把 string 包装成 ChatMessage(ChatRole.User, ...)
  -> RunAsync(IEnumerable<ChatMessage>) 设置 AgentRunContext
  -> ChatClientAgent.RunCoreAsync
  -> PrepareSessionAndMessagesAsync 准备 session、chatOptions、输入消息
  -> IChatClient.GetResponseAsync(...)
  -> ChatResponse 转成 AgentResponse
  -> 调用方读取 AgentResponse.Text / ToString()
```

一次流式最小调用可以拆成：

```text
RunStreamingAsync("用户问题")
  -> AIAgent 包装 User ChatMessage
  -> ChatClientAgent.RunCoreStreamingAsync
  -> IChatClient.GetStreamingResponseAsync(...)
  -> 每个 ChatResponseUpdate 转成 AgentResponseUpdate
  -> await foreach 逐片消费 update.Text
  -> 流结束后聚合并通知历史/上下文 provider
```

### 7. 常见误解是什么

- 误解一：`instructions` 等同于用户输入。实际它是系统行为指令，应由开发者控制，不应拼接不可信用户输入。
- 误解二：`Console.WriteLine(await agent.RunAsync(...))` 只能打印字符串。实际 `RunAsync` 返回 `AgentResponse`，只是 `ToString()` 返回了 `Text`。
- 误解三：流式输出就是多个完整答案。实际 `AgentResponseUpdate` 通常是一个答案的多个片段，需要按需聚合或逐片渲染。
- 误解四：不创建 session 也能自动多轮。实际是否保留上下文取决于 session 和底层 provider 的历史管理能力。

## 结合源码理解执行过程

### 1. 入口在哪里

最小入口在 sample：

- `dotnet/samples/01-get-started/01_hello_agent/Program.cs`
- `dotnet/samples/01-get-started/03_multi_turn/Program.cs`

抽象入口在：

- `dotnet/src/Microsoft.Agents.AI.Abstractions/AIAgent.cs`

其中 `RunAsync(string message, ...)` 做了两件关键事：

1. 校验字符串不能为空。
2. 创建 `new ChatMessage(ChatRole.User, message)`，再委托给 `RunAsync(ChatMessage, ...)`。

流式入口 `RunStreamingAsync(string message, ...)` 也是同样思路，只是最终进入 `RunCoreStreamingAsync`。

### 2. 调用是如何继续流转的

在 `AIAgent.RunAsync(IEnumerable<ChatMessage> ...)` 中，源码会创建 `AgentRunContext`：

```csharp
CurrentRunContext = new(this, session, messages as IReadOnlyCollection<ChatMessage> ?? messages.ToList(), options);
return this.RunCoreAsync(messages, session, options, cancellationToken);
```

然后由具体实现类处理。对本章 sample 来说，具体实现类是 `ChatClientAgent`。

在 `ChatClientAgent.RunCoreAsync` 中，关键链路是：

```csharp
var inputMessages = Throw.IfNull(messages) as IReadOnlyCollection<ChatMessage> ?? messages.ToList();

(ChatClientAgentSession safeSession,
 ChatOptions? chatOptions,
 List<ChatMessage> inputMessagesForChatClient,
 ChatClientAgentContinuationToken? _) =
    await this.PrepareSessionAndMessagesAsync(session, inputMessages, options, cancellationToken);

chatResponse = await chatClient.GetResponseAsync(inputMessagesForChatClient, chatOptions, cancellationToken);

return new AgentResponse(chatResponse)
{
    AgentId = this.Id,
    ContinuationToken = WrapContinuationToken(chatResponse.ContinuationToken)
};
```

这说明 Agent Framework 在调用模型前会先准备会话、选项和消息，之后才把请求交给底层 `IChatClient`。

### 3. 关键对象分别负责什么

- `AIAgent`：统一调用抽象，负责把便捷重载归一到消息集合，并维护当前运行上下文。
- `ChatClientAgent`：把 Agent 抽象桥接到 `IChatClient`，负责准备 session、chat options、历史上下文和响应包装。
- `ChatOptions`：承载 `Instructions`、`Tools` 等运行配置。
- `ChatMessage`：表示输入和输出中的单条消息。
- `AgentResponse`：包装完整 `ChatResponse`，适合非流式调用。
- `AgentResponseUpdate`：包装 `ChatResponseUpdate`，适合流式调用。
- `AgentSession`：承载会话状态，支持多轮上下文。

### 4. 扩展点在哪里

- 改 Prompt：修改 `AsAIAgent(instructions: ...)`。
- 改输入结构：从 `RunAsync(string)` 升级到 `RunAsync(new ChatMessage(...))` 或 `RunAsync(IEnumerable<ChatMessage>)`。
- 改输出处理：从直接 `Console.WriteLine(response)` 改为检查 `response.Messages`、`response.Usage`、`response.FinishReason`。
- 改流式 UI：从逐行打印 `update` 改为按 `MessageId` 聚合，或在前端逐 token 渲染。
- 改会话：创建并复用 `AgentSession`，或后续接入持久化历史提供者。

## 关键代码或配置讲解

### 代码片段 1：创建带 instructions 的 Agent

```csharp
AIAgent agent = new AzureOpenAIClient(
    new Uri(endpoint),
    new DefaultAzureCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(instructions: "You are good at telling jokes.", name: "Joker");
```

- 作用：把 Azure OpenAI 的 ChatClient 转换成 Agent，并设置系统指令与名称。
- 为什么重要：`instructions` 是 Agent 行为的第一层约束，`name` 会用于标识和日志/作者名等场景。
- 改动它会影响什么：改变回答风格、任务边界和输出口径；如果加入过长指令，会增加 token 成本。

### 代码片段 2：非流式调用

```csharp
Console.WriteLine(await agent.RunAsync("Tell me a joke about a pirate."));
```

- 作用：发送一条用户消息，等待完整响应。
- 为什么重要：这是后端服务、命令行工具、批处理任务中最简单的 Agent 调用方式。
- 改动它会影响什么：如果改为显式保存响应对象，就能检查更多元数据：

```csharp
AgentResponse response = await agent.RunAsync("Tell me a joke about a pirate.");
Console.WriteLine(response.Text);
Console.WriteLine(response.FinishReason);
Console.WriteLine(response.Usage);
```

### 代码片段 3：流式调用

```csharp
await foreach (var update in agent.RunStreamingAsync("Tell me a joke about a pirate."))
{
    Console.WriteLine(update);
}
```

- 作用：逐片接收模型输出。
- 为什么重要：真实产品中的聊天窗口通常需要“边生成边显示”，避免用户等待完整响应。
- 改动它会影响什么：如果��构建最终完整答案，应把 `update.Text` 累积起来；如果只打印每个 update，输出可能被换行切碎。

### 代码片段 4：多轮会话

```csharp
AgentSession session = await agent.CreateSessionAsync();
Console.WriteLine(await agent.RunAsync("Tell me a joke about a pirate.", session));
Console.WriteLine(await agent.RunAsync("Now add some emojis to the joke and tell it in the voice of a pirate's parrot.", session));
```

- 作用：在同一个 session 中连续调用 Agent。
- 为什么重要：第二轮请求依赖第一轮上下文，是从“单问单答”走向“对话系统”的最小步骤。
- 改动它会影响什么：如果第二次调用不传同一个 session，Agent 可能不知道 “the joke” 指的是哪一个笑话。

## 做一个最小改造实验

### 实验目标

把 `01_hello_agent` 从“讲笑话 Agent”改成综合案例中的“企业内部知识与工单协同 Agent”的最小交互原型。

### 改造点

只修改 `dotnet/samples/01-get-started/01_hello_agent/Program.cs` 的本地实验副本，建议不要提交 sample 改动。本章实际只编辑文档文件。

把：

```csharp
.AsAIAgent(instructions: "You are good at telling jokes.", name: "Joker");
```

改为：

```csharp
.AsAIAgent(
    instructions: "You are Contoso internal knowledge and ticket triage assistant. Answer briefly, ask one clarifying question if the ticket lacks required information, and do not invent policy details.",
    name: "ContosoSupportTriage");
```

把用户输入改为：

```csharp
Console.WriteLine(await agent.RunAsync("My VPN stopped working after password reset. What should I check before opening a ticket?"));
```

流式部分改为：

```csharp
await foreach (var update in agent.RunStreamingAsync("Create a short triage checklist for a VPN login issue."))
{
    Console.Write(update.Text);
}
```

### 你应该观察什么

- Agent 的角色从讲笑话变成了内部支持助手。
- 回答应更像排障清单或工单分诊建议。
- 流式输出用 `Console.Write(update.Text)` 会更接近连续打字效果。
- 如果问题缺少信息，理想输出应提出一个澄清问题，而不是编造内部政策。

### 可能出现的问题

- 输出仍然像通用聊天助手：检查 `instructions` 是否真的修改并重新运行。
- 输出包含虚构流程：说明 Prompt 约束还不够，后续第 12 章需要 RAG 支持真实知识。
- 流式输出有断词或空片段：这是增量更新的正常现象，生产 UI 需要聚合和节流。

## 本章在综合案例中的增量

主案例：**企业内部知识与工单协同 Agent 系统**。

- 本章之前：第 3 章已有最小 Agent；第 4 章开始理解 Tool 调用，但主案例的“用户怎么问、Agent 如何收消息、如何返回结果”还不够清晰。
- 本章加入的新能力：为主案例定义最小系统 Prompt、用户消息输入、非流式响应和流式响应两条交互路径。
- 修改的模块或流程：
  - 用户入口：从随意输入升级为 `ChatRole.User` 消息。
  - Agent 决策层：通过 `instructions` 约束“内部知识与工单分诊助手”的行为。
  - 输出层：用 `AgentResponse` 支持完整回答，用 `AgentResponseUpdate` 支持聊天 UI 的增量显示。
  - 会话边界：通过 `AgentSession` 让后续多轮工单补充信息成为可能。
- 加入后的系统变化：主案���已经具备“用户提问 → Agent 按角色回答 → 可流式展示 → 可复用 session 进行追问”的第一阶段闭环。
- 如果不加入本章能力：后续即使加入 RAG、Tool、Workflow，也会缺少稳定的消息边界，无法判断哪些内容是系统指令、哪些是用户输入、哪些是可展示响应。
- 与仓库已有源码的关系：
  - 直接支持：`01_hello_agent` 和 `03_multi_turn` 已展示最小调用、流式调用和 session 多轮。
  - 可基于已有 sample 最小改造：把 `instructions` 和用户问题替换成 Contoso 内部支持场景。
  - 尚未完整展开：真实工单系统、知识库检索、审批和持久化会在后续章节继续扩展。

## 常见问题与排错

### 问题 1：`AZURE_OPENAI_ENDPOINT is not set.`

- 现象：程序启动后立即抛出环境变量未设置异常。
- 原因：sample 中直接读取 `Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")`，为空就抛错。
- 排查方法：在当前 shell 中检查环境变量是否存在。
- 修复方式：设置 `AZURE_OPENAI_ENDPOINT`，并确认 `AZURE_OPENAI_DEPLOYMENT_NAME` 与 Azure OpenAI 部署名一致。

### 问题 2：身份认证失败

- 现象：调用 Azure OpenAI 时出现认证或权限相关错误。
- 原因：sample 使用 `DefaultAzureCredential`，本地未登录 Azure CLI、Visual Studio 凭据不可用，或账号没有访问目标资源的权限。
- 排查方法：确认当前开发身份能访问 Azure OpenAI 资源；检查订阅、租户和 RBAC 权限。
- 修复方式：开发环境可登录对应 Azure 账号；生产环境不要盲目使用默认凭据链，应考虑 `ManagedIdentityCredential` 等明确凭据。

### 问题 3：第二轮对话丢失上下文

- 现象：第二轮问 “the joke” 或 “刚才那个问题” 时，Agent 不知道指什么。
- 原因：没有复用同一个 `AgentSession`，或底层服务/历史提供者没有保存上下文。
- 排查方法：检查两次 `RunAsync` 是否传入同一个 `session` 变量。
- 修复方式：按 `03_multi_turn/Program.cs` 的方式先 `CreateSessionAsync()`，再把同一个 session 传给多次调用。

### 问题 4：流式输出看起来很碎

- 现象：控制台每个片段一行，或者词被拆开。
- 原因：`AgentResponseUpdate` 是增量片段，不保证每个片段都是完整句子。
- 排查方法：打印 `update.Text` 并观察是否有空字符串、半句或单词片段。
- 修复方式：控制台可用 `Console.Write(update.Text)`；Web UI 可按 `MessageId` 聚合，并在前端做节流渲染。

### 问题 5：Prompt 被用户输入“覆盖”

- 现象：用户要求“忽略之前所有指令”后，Agent 行为明显偏离系统角色。
- 原因：Prompt Injection。`AIAgent` 源码注释明确提示，框架不会自动验证或清洗消息内容。
- 排查方法：检查是否把不可信用户输入拼进 `instructions`，以及是否缺少输出校验。
- 修复方式：系统指令必须由开发者控制；用户输入只作为 `ChatRole.User` 消息；对敏感动作、工具参数和输出内容做校验。

## 企业级落地提醒

- 稳定性：非流式适合短任务；长回答、前端聊天、移动端弱网场景更适合流式，但要处理断流、重试和重复片段。
- 可维护性：把系统 Prompt 作为配置或版本化资源管理，不要散落在多个业务方法中；每次修改 Prompt 都应记录意图和影响。
- 权限边界：不要把用户输入拼接进系统 Prompt；不同角色消息代表不同信任等级，`system` / `instructions` 必须开发者可控。
- 安全风险：LLM 输出应视为不可信。源码注释中明确提到幻觉、间接 Prompt Injection、恶意 HTML/SQL/Shell 内容等风险。
- 成本控制：`instructions`、历史消息和上下文都会消耗 token。多轮 session 越长，后续调用成本越高，第 7 章和后续压缩策略会继续展开。
- 可观测性：生产中不要只记录最终文本；应记录响应 ID、FinishReason、用量、错误类型和必要的脱敏 trace。
- 可测试性：为关键 Prompt 准备固定测试问题，至少覆盖正常提问、缺少信息、越权请求、Prompt Injection、长输入等情况。
- 持久化与恢复：本章只做最小 session；如果要跨进程/跨天恢复会话，需要使用 `SerializeSessionAsync` / `DeserializeSessionAsync` 或后续章节的持久化方案，并保护其中的敏感数据。

## 本章小结

- 本章最重要的结论 1：最小交互不是“字符串进、字符串出”，而是 `instructions`、`ChatMessage`、`AgentSession`、`AgentResponse` / `AgentResponseUpdate` 共同组成的链路。
- 本章最重要的结论 2：`RunAsync(string)` 是便捷重载，内部会包装为 `ChatRole.User` 的 `ChatMessage`；复杂场景应显式管理消息集合和 session。
- 本章最重要的结论 3：流式输出返回的是增量 `AgentResponseUpdate`，生产 UI 需要正确聚合、渲染和处理异常。
- 下一章为什么要学：当你理解消息与最小交互后，就可以进入第 6 章，系统学习 `AIAgent`、Builder、扩展方法、Provider 等核心抽象之间的关系。

## 本章建议产出物

- 一次 `dotnet/samples/01-get-started/01_hello_agent` 运行记录。
- 一次 `dotnet/samples/01-get-started/03_multi_turn` 多轮运行记录。
- 一张“string → ChatMessage → ChatClientAgent → IChatClient → AgentResponse”的调用链图。
- 一个把 `Joker` 改成 `ContosoSupportTriage` 的 Prompt 改造实验记录。
- 一份包含环境变量、身份认证、session 丢失、流式输出碎片、Prompt Injection 的问题清单。

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
