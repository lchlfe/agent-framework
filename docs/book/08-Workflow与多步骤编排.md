# 第 8 章：Workflow 与多步骤编排

> 本章定位：把前面“单个 Agent/Tool/Session”的能力，升级为由多个执行节点、路由边和共享状态组成的流程型 Agent 系统。

## 本章目标

学完本章后，你应能够：

- 解释 Workflow 要解决的问题：为什么不能只靠一个 Agent 一次性完成复杂任务。
- 在仓库中定位 Workflow 的 sample 与核心源码。
- 读懂 `WorkflowBuilder` 如何把 Executor 节点连接成有向图。
- 理解节点、边、输出、条件路由、fan-out/fan-in、状态和运行事件之间的关系。
- 跑通或验证一个最小 Workflow，并能做一个小改造观察执行链路变化。
- 知道企业级 Agent 流程编排中需要关注的稳定性、状态一致性、可观测性与恢复问题。

## 本章与前后章节的关系

- 前置知识：第 6 章的 Agent 核心抽象、第 7 章的 Session/History/Context/Memory。你需要知道 Agent 如何处理消息，以及上下文和状态为什么重要。
- 本章解决：如何把多个处理步骤组织成一个可执行流程，让复杂任务按“节点 -> 边 -> 状态 -> 输出”的方式流转。
- 后续衔接：第 9 章会进一步讨论多 Agent 协作设计；第 10 章会更深入讨论状态持久化、检查点和失败恢复。本章会提到这些能力，但不会展开到生产级部署细节。

## 先建立整体认知

Workflow 可以理解为“可执行的任务图”。单个 Agent 更像一个会思考和调用工具的处理单元，而 Workflow 关注的是：

- 先做哪一步；
- 哪一步的输出会传给下一步；
- 根据什么条件走不同分支；
- 哪些步骤可以并行；
- 并行结果如何汇总；
- 多个节点之间如何共享临时状态；
- 运行过程中如何流式观察事件与最终输出。

在 Agent 系统分层中，Workflow 位于“流程层/编排层”。如果缺少这一层，系统通常会出现几个问题：

- 复杂任务全部塞进一个 Prompt，逻辑难维护、难测试。
- 分支判断、重试、人工介入和并行处理很难显式表达。
- 每个步骤的输入输出边界不清晰，排错只能看最终答案。
- 多 Agent 协作没有稳定拓扑，容易变成不可控的自由对话。

本章要建立的核心心智模型是：**Workflow = Executor 节点 + Edge 路由 + IWorkflowContext 状态/输出能力 + 运行时事件流**。

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：
  - `dotnet/samples/03-workflows`
  - `dotnet/src/Microsoft.Agents.AI.Workflows`
  - `dotnet/src/Microsoft.Agents.AI.Workflows/Execution`
  - `dotnet/src/Shared/Workflows/Execution`
- 项目：
  - `dotnet/src/Microsoft.Agents.AI.Workflows/Microsoft.Agents.AI.Workflows.csproj`
  - `dotnet/src/Microsoft.Agents.AI.Workflows.Declarative/Microsoft.Agents.AI.Workflows.Declarative.csproj`
  - `dotnet/src/Microsoft.Agents.AI.DurableTask`：偏第 10、16 章的持久化和 Durable 运行能力，本章只作为延伸入口。
- Sample：
  - `dotnet/samples/03-workflows/_StartHere/01_Streaming/Program.cs`：最小顺序链路与流式事件。
  - `dotnet/samples/03-workflows/_StartHere/03_AgentWorkflowPatterns/Program.cs`：顺序、并发、handoff、group chat 等 Agent 工作流模式。
  - `dotnet/samples/03-workflows/ConditionalEdges/01_EdgeCondition/Program.cs`：条件边和分支路由。
  - `dotnet/samples/03-workflows/SharedStates/Program.cs`：共享状态、fan-out/fan-in 汇总。
  - `dotnet/samples/03-workflows/Concurrent`、`Loop`、`HumanInTheLoop`、`Checkpoint`、`Visualization`、`Observability`：分别对应并发、循环、人工介入、检查点、可视化和观测能力。
- 关键类型：
  - `Workflow`
  - `WorkflowBuilder`
  - `Executor<TInput, TOutput>` / `Executor<TInput>`
  - `IWorkflowContext`
  - `WorkflowEvent`、`WorkflowOutputEvent`、`WorkflowStartedEvent`、`WorkflowErrorEvent`、`WorkflowWarningEvent`
  - `StreamingRun`、`Run`、`InProcessExecution`
  - `AgentWorkflowBuilder`、`GroupChatWorkflowBuilder`、`HandoffWorkflowBuilder`
- 关键接口：
  - `IWorkflowContext`
  - `IWorkflowExecutionEnvironment`
- 关键扩展点：
  - `WorkflowBuilder.AddEdge`
  - `WorkflowBuilder.AddFanOutEdge`
  - `WorkflowBuilder.AddFanInBarrierEdge`
  - `WorkflowBuilder.WithOutputFrom`
  - `WorkflowBuilderExtensions.ForwardMessage`、`ForwardExcept`、`AddChain`
  - `IWorkflowContext.SendMessageAsync`
  - `IWorkflowContext.YieldOutputAsync`
  - `IWorkflowContext.ReadStateAsync`、`ReadOrInitStateAsync`、`QueueStateUpdateAsync`

推荐阅读顺序：先看 `dotnet/samples/03-workflows/README.md`，再看 `_StartHere/01_Streaming/Program.cs`，然后看 `WorkflowBuilder.cs`、`Workflow.cs`、`IWorkflowContext.cs`，最后再按需要看条件边、共享状态和 Agent 工作流模式 sample。

## 先跑通或先验证一个最小示例

### 示例目标

运行或阅读验证一个最小的顺序 Workflow：输入字符串先经过 `UppercaseExecutor` 转大写，再经过 `ReverseTextExecutor` 反转，运行时通过事件流观察每个 Executor 的完成结果。

### 运行前提

- 已安装仓库要求的 .NET SDK。
- 当前工作目录在仓库根目录：`D:\code\agent-framework`。
- 该最小 sample 不需要 Azure OpenAI 环境变量，因为它只使用本地 Executor。

### 操作步骤

1. 打开 sample：

   ```text
   dotnet/samples/03-workflows/_StartHere/01_Streaming/Program.cs
   ```

2. 观察核心代码：

   ```csharp
   UppercaseExecutor uppercase = new();
   ReverseTextExecutor reverse = new();

   WorkflowBuilder builder = new(uppercase);
   builder.AddEdge(uppercase, reverse).WithOutputFrom(reverse);
   var workflow = builder.Build();

   await using StreamingRun run = await InProcessExecution.RunStreamingAsync(workflow, input: "Hello, World!");
   await foreach (WorkflowEvent evt in run.WatchStreamAsync())
   {
       if (evt is ExecutorCompletedEvent executorCompleted)
       {
           Console.WriteLine($"{executorCompleted.ExecutorId}: {executorCompleted.Data}");
       }
   }
   ```

3. 在对应 sample 项目目录执行 `dotnet run`。如果你的仓库 sample 是由解决方案统一管理，也可以在 IDE 中直接把该 sample 作为启动项目运行。

### 预期现象

你会看到 Workflow 在执行过程中输出类似以下信息：

```text
UppercaseExecutor: HELLO, WORLD!
ReverseTextExecutor: !DLROW ,OLLEH
```

实际格式可能随事件类型和运行方式略有不同，但关键是能看到两个 Executor 按边的顺序先后完成。

### 成功标志

- `UppercaseExecutor` 收到原始输入并返回大写文本。
- `ReverseTextExecutor` 收到上一步的输出并返回反转文本。
- `WatchStreamAsync()` 能观察到运行过程中的 `WorkflowEvent`。
- Workflow 没有报 unreachable executor、unbound executor 或类型不匹配相关错误。

## 核心概念拆解

### 1. Workflow 是什么

`Workflow` 是可执行的流程定义。源码 `dotnet/src/Microsoft.Agents.AI.Workflows/Workflow.cs` 中可以看到它保存了几类关键信息：

- `StartExecutorId`：起始节点 ID。
- `ExecutorBindings`：Executor 绑定表，按 executor ID 索引。
- `Edges`：边集合，按源节点 ID 分组。
- `OutputExecutors`：哪些节点的输出会冒泡为 Workflow 输出。
- `Ports`：外部请求端口，用于人工介入或外部交互等场景。

`Workflow` 本身不是“智能体”，而是一个图结构和运行协议描述。真正处理消息的是图中的 Executor 或 Agent Executor。

### 2. Executor 节点是什么

Executor 是 Workflow 图中的处理节点。比如 `_StartHere/01_Streaming/Program.cs` 中：

- `UppercaseExecutor : Executor<string, string>`：接收 `string`，返回 `string`。
- `ReverseTextExecutor : Executor<string, string>`：接收 `string`，返回 `string`。

在条件边 sample 中，节点可以包装 Agent：

- `SpamDetectionExecutor : Executor<ChatMessage, DetectionResult>` 调用 spam detection agent。
- `EmailAssistantExecutor : Executor<DetectionResult, EmailResponse>` 调用 email assistant agent。
- `SendEmailExecutor : Executor<EmailResponse>` 通过 `context.YieldOutputAsync` 产出最终文本。

因此，Executor 的职责是把“一个步骤”封装成明确的输入输出边界；它可以是纯代码、Agent 调用、工具调用、外部系统调用，也可以是桥接多个消息协议的适配器。

### 3. Edge 边是什么

Edge 是节点之间的消息路由规则。`WorkflowBuilder.cs` 中 `AddEdge` 的注释说明它会添加一条从 source executor 到 target executor 的有向边，并可附带 condition。

常见边类型包括：

- 直接边：`AddEdge(source, target)`，source 的输出会传给 target。
- 条件边：`AddEdge(source, target, condition: ...)`，只有满足条件时才路由。
- Fan-out 边：`AddFanOutEdge(source, [target1, target2])`，一个输出分发到多个目标。
- Fan-in barrier 边：`AddFanInBarrierEdge([source1, source2], target)`，等待多个上游都产生消息后再进入汇总节点。

边不是简单的函数调用。它表达的是运行时消息在图中的下一跳。`WorkflowBuilder.Build()` 会校验是否存在未绑定节点、不可达节点等结构性问题。

### 4. Output 是什么

Workflow 中一个 Executor 返回值不一定就是整个 Workflow 的最终输出。需要通过：

```csharp
.WithOutputFrom(reverse)
```

或在 Executor 上使用 `context.YieldOutputAsync(...)`，让某些节点的结果冒泡为 `WorkflowOutputEvent`。

这点很重要：中间节点的返回值主要用于图内流转；最终输出是你明确标记出来、给调用方消费的结果。

### 5. IWorkflowContext 是什么

`IWorkflowContext` 是 Executor 在运行时访问 Workflow 能力的入口。源码中它提供了几类能力：

- `AddEventAsync`：向 Workflow 输出事件队列添加事件。
- `SendMessageAsync`：把消息发送给连接的下游 Executor。
- `YieldOutputAsync`：产出 Workflow 输出。
- `RequestHaltAsync`：请求在当前 SuperStep 结束后停止。
- `ReadStateAsync`、`ReadOrInitStateAsync`：读取状态。
- `QueueStateUpdateAsync`、`QueueClearScopeAsync`：排队更新或清理状态。
- `TraceContext`：获取当前消息的 trace 上下文。
- `ConcurrentRunsEnabled`：判断当前执行环境是否支持同一个 Workflow 实例的并发运行。

注意 `QueueStateUpdateAsync` 的语义：状态更新是排队的，其他 Executor 通常要到下一个 SuperStep 才能看到。这和直接修改全局变量不同。

### 6. 状态是什么

状态是多个 Executor 之间共享数据和协调的机制。`SharedStates/Program.cs` 展示了典型做法：

1. `FileReadExecutor` 读取文件内容，生成 `fileID`，把文件内容写入共享状态 scope。
2. `WordCountingExecutor` 和 `ParagraphCountingExecutor` 都拿到 `fileID`，分别从共享状态读取文件内容。
3. 两个并行分支分别统计字数和段落数。
4. `AggregationExecutor` 汇总结果并 `YieldOutputAsync` 输出。

相关源码在 `Execution/StateManager.cs`。其中 `_queuedUpdates` 说明状态更新会先进入队列，`PublishUpdatesAsync` 再发布到实际状态 scope。这种设计能避免同一个 SuperStep 内状态读写顺序不稳定。

### 7. 常见误解是什么

- 误解一：Workflow 就是多个函数顺序调用。实际上它是图结构，支持条件、并发、汇聚、循环、外部请求和事件流。
- 误解二：Executor 返回值就是最终响应。实际上必须通过 `WithOutputFrom` 或 `YieldOutputAsync` 指定输出。
- 误解三：状态更新立刻对所有节点可见。实际上状态更新有 SuperStep 边界，其他 Executor 往往下一步才看到。
- 误解四：只要编译通过，Workflow 图一定正确。`Build()` 会做结构校验，但类型兼容和业务语义仍需要运行时验证与测试。
- 误解五：并行节点可以随意共享可变字段。sample 中 `AggregationExecutor` 有 `_messages`，教学上直观，但企业落地时要注意并发、复用和重入风险。

## 结合源码理解执行过程

建议用 `_StartHere/01_Streaming/Program.cs` 建立最小调用链：

```text
Main
  -> new UppercaseExecutor / new ReverseTextExecutor
  -> new WorkflowBuilder(uppercase)
  -> AddEdge(uppercase, reverse)
  -> WithOutputFrom(reverse)
  -> Build()
  -> InProcessExecution.RunStreamingAsync(workflow, "Hello, World!")
  -> run.WatchStreamAsync()
  -> ExecutorCompletedEvent / WorkflowOutputEvent
```

### 1. 入口在哪里

最直接的入口在 sample 的 `Main()`：

```csharp
WorkflowBuilder builder = new(uppercase);
builder.AddEdge(uppercase, reverse).WithOutputFrom(reverse);
var workflow = builder.Build();
await using StreamingRun run = await InProcessExecution.RunStreamingAsync(workflow, input: "Hello, World!");
```

对于框架源码，构建入口在 `WorkflowBuilder`：

- 构造函数 `WorkflowBuilder(ExecutorBinding start)` 记录起始 Executor。
- `AddEdge` / `AddFanOutEdge` / `AddFanInBarrierEdge` 逐步记录图结构。
- `WithOutputFrom` 记录哪些 Executor 是输出源。
- `Build()` 校验并创建 `Workflow`。

### 2. 调用是如何继续流转的

以顺序链路为例：

1. `RunStreamingAsync` 启动 Workflow，并把初始输入投递给 `StartExecutorId` 对应的 Executor。
2. `UppercaseExecutor.HandleAsync` 返回大写文本。
3. 运行时根据 `Edges` 找到从 `UppercaseExecutor` 出发的边，把大写文本路由到 `ReverseTextExecutor`。
4. `ReverseTextExecutor.HandleAsync` 返回反转文本。
5. 因为 `reverse` 被 `WithOutputFrom(reverse)` 标记为输出源，运行时将结果作为输出事件冒泡。
6. 调用方通过 `WatchStreamAsync()` 观察 `WorkflowEvent`。

源码中 `MessageRouter.RouteMessageAsync` 负责把某个消息按类型交给 Executor 注册的 handler；如果找不到合适 handler 且没有 catch-all，消息就不会被该 Executor 处理。

### 3. 关键对象分别负责什么

- `WorkflowBuilder`：构建阶段对象，负责收集节点、边、输出源并做图校验。
- `Workflow`：构建完成后的流程定义，保存起点、Executor 绑定、边、端口、输出节点等信息。
- `Executor`：运行阶段的业务处理单元。
- `IWorkflowContext`：Executor 与运行时交互的入口，负责消息、输出、状态、事件等。
- `MessageRouter`：根据消息运行时类型找到可处理该消息的 handler。
- `StateManager`：管理状态 scope、排队更新与发布。
- `StreamingRun` / `Run`：运行结果句柄，分别用于流式观察和一次性查看运行事件。

### 4. 扩展点在哪里

- 想增加一个步骤：新增一个 `Executor`，再用 `AddEdge` 接入图。
- 想按条件分支：给 `AddEdge` 传入 condition，参考 `ConditionalEdges/01_EdgeCondition/Program.cs`。
- 想并行处理：使用 `AddFanOutEdge`，参考 `SharedStates/Program.cs` 或 `Concurrent` sample。
- 想汇总并行结果：使用 `AddFanInBarrierEdge`。
- 想输出中间进度：在 Executor 中调用 `context.AddEventAsync` 或使用流式运行观察事件。
- 想在多个节点共享数据：使用 `ReadStateAsync` / `QueueStateUpdateAsync`，并明确 scope 名。
- 想把多个 Agent 组合起来：参考 `AgentWorkflowBuilder.BuildSequential`、`BuildConcurrent`、`CreateHandoffBuilderWith`、`CreateGroupChatBuilderWith`。

## 关键代码或配置讲解

### 代码片段 1：最小顺序编排

```csharp
WorkflowBuilder builder = new(uppercase);
builder.AddEdge(uppercase, reverse).WithOutputFrom(reverse);
var workflow = builder.Build();
```

- 作用：定义从 `uppercase` 到 `reverse` 的顺序链路，并指定 `reverse` 是 Workflow 输出源。
- 为什么重要：这是最小 Workflow 的骨架，说明编排不是写在 Prompt 里，而是写成可检查的图结构。
- 改动它会影响什么：如果去掉 `AddEdge`，下游节点不会收到消息；如果去掉 `WithOutputFrom(reverse)`，调用方可能看不到最终结果。

### 代码片段 2：条件路由

来自 `ConditionalEdges/01_EdgeCondition/Program.cs`：

```csharp
var workflow = new WorkflowBuilder(spamDetectionExecutor)
    .AddEdge(spamDetectionExecutor, emailAssistantExecutor, condition: GetCondition(expectedResult: false))
    .AddEdge(emailAssistantExecutor, sendEmailExecutor)
    .AddEdge(spamDetectionExecutor, handleSpamExecutor, condition: GetCondition(expectedResult: true))
    .WithOutputFrom(handleSpamExecutor, sendEmailExecutor)
    .Build();
```

- 作用：先做垃圾邮件检测，再根据 `DetectionResult.IsSpam` 走不同分支。
- 为什么重要：真实业务流程往往不是固定顺序，而是依赖分类、风险等级、审批结果等条件动态路由。
- 改动它会影响什么：如果两个条件都可能为 true，会出现多分支同时执行；如果两个条件都为 false，下游可能没有节点继续处理。

### 代码片段 3：共享状态与并行汇总

来自 `SharedStates/Program.cs`：

```csharp
var workflow = new WorkflowBuilder(fileRead)
    .AddFanOutEdge(fileRead, [wordCount, paragraphCount])
    .AddFanInBarrierEdge([wordCount, paragraphCount], aggregate)
    .WithOutputFrom(aggregate)
    .Build();
```

以及状态写入：

```csharp
await context.QueueStateUpdateAsync(fileID, fileContent, scopeName: FileContentStateConstants.FileContentStateScope, cancellationToken);
```

- 作用：文件读取后并行做字数统计和段落统计，再汇总输出。
- 为什么重要：把大任务拆成并行子任务可以提升吞吐，也能让每个节点职责更清晰。
- 改动它会影响什么：如果去掉 fan-in barrier，汇总节点可能无法判断何时所有分支完成；如果 scope/key 不一致，读状态会失败。

## 做一个最小改造实验

### 实验目标

在 `_StartHere/01_Streaming/Program.cs` 中新增一个 `AppendSuffixExecutor`，让流程从“两步”变为“三步”：

```text
Hello, World!
  -> UppercaseExecutor
  -> ReverseTextExecutor
  -> AppendSuffixExecutor
```

最终输出类似：

```text
!DLROW ,OLLEH [workflow done]
```

### 改造点

1. 新增 Executor：

```csharp
internal sealed class AppendSuffixExecutor() : Executor<string, string>("AppendSuffixExecutor")
{
    public override ValueTask<string> HandleAsync(string message, IWorkflowContext context, CancellationToken cancellationToken = default) =>
        ValueTask.FromResult($"{message} [workflow done]");
}
```

2. 修改构建逻辑：

```csharp
AppendSuffixExecutor append = new();

WorkflowBuilder builder = new(uppercase);
builder
    .AddEdge(uppercase, reverse)
    .AddEdge(reverse, append)
    .WithOutputFrom(append);
var workflow = builder.Build();
```

### 你应该观察什么

- 事件流中应该多出 `AppendSuffixExecutor` 的完成事件。
- 最终输出源从 `reverse` 变成 `append` 后，Workflow 输出应该包含后缀。
- 如果你忘记把 `WithOutputFrom(reverse)` 改成 `WithOutputFrom(append)`，你可能仍然只能看到反转结果，而看不到新增节点的最终结果。

### 可能出现的问题

- 忘记添加 `AddEdge(reverse, append)`：`append` 会在 `Build()` 校验时被判定为不可达，或根本没有被纳入图。
- `AppendSuffixExecutor` 的输入类型写错：运行时消息路由找不到匹配 handler。
- 没有指定输出源：流程执行了，但调用方没有拿到你想要的最终输出。

## 本章在综合案例中的增量

主案例仍然是：**企业内部知识与工单协同 Agent 系统**。

在前面章节中，它已经具备了最小 Agent、Prompt、Tool 调用和基本上下文能力。到本章之前，它更像一个“会回答、会调用工具的单点助手”。本章给它增加的是“流程层”：把工单处理拆成可观察、可测试、可维护的多步骤 Workflow。

建议本章新增的流程如下：

```text
用户提交问题/工单
  -> IntakeExecutor：规范化输入，生成 ticketId
  -> ClassifyExecutor：判断是知识问答、普通工单还是高风险操作
  -> 条件边：
       知识问答 -> RetrieveKnowledgeExecutor -> DraftAnswerExecutor -> ReturnAnswerExecutor
       普通工单 -> CreateTicketExecutor -> ReturnTicketStatusExecutor
       高风险操作 -> RiskReviewExecutor -> 等待人工确认（第 14 章深入）
```

本章中属于仓库已有 sample 直接支持的部分：

- 顺序链路：对应 `_StartHere/01_Streaming`。
- 条件分支：对应 `ConditionalEdges/01_EdgeCondition`。
- 并行与汇总：对应 `SharedStates` 和 `Concurrent`。
- Agent 放入 Workflow：对应 `_StartHere/03_AgentWorkflowPatterns`。

可以基于已有 sample 做最小改造实现的部分：

- 把邮件分类 sample 改造成工单分类：`SpamDetectionExecutor` 可类比为 `ClassifyTicketExecutor`。
- 把 `DetectionResult.IsSpam` 改为 `TicketRoute` 或 `RiskLevel`。
- 把 `EmailStateScope` 改为 `TicketStateScope`，存储 `ticketId`、原始问题、分类结果等。

仓库未在本章完整展开但框架能力支持的部分：

- 真正接入企业工单系统。
- 高风险操作审批的完整人工介入体验。
- 工作流检查点、跨进程恢复和服务化部署。这些会在第 10、14、16 章继续展开。

如果不加入本章的 Workflow 能力，主案例会卡在“每次请求都让单 Agent 自己决定所有步骤”的阶段，难以做到可审计、可测试、可恢复和可分工。

## 常见问题与排错

### 问题 1：Workflow cannot be built because there are unbound executors

- 现象：调用 `Build()` 时提示存在 unbound executors。
- 原因：图中出现了占位的 Executor 绑定，但没有用实际 Executor 绑定替换；或者使用了需要绑定的声明式/模板式构建方式但遗漏 `BindExecutor`。
- 排查方法：检查 `WorkflowBuilder.cs` 中 `Validate()` 的逻辑，它会检查 `_unboundExecutors`。回到你的构建代码，确认每个节点都有具体实例或注册。
- 修复方式：为每个占位节点调用 `BindExecutor`，或直接在构建图时传入具体 Executor 实例。

### 问题 2：Workflow cannot be built because there are unreachable executors

- 现象：`Build()` 报某些 executor unreachable。
- 原因：某个 Executor 被加入或绑定了，但从 `StartExecutorId` 没有边能到达它。
- 排查方法：画出从起点开始的图，检查是否漏写 `AddEdge`、`AddFanOutEdge` 或 fan-in 的 source/target。
- 修复方式：补上从起点可达的边；如果是故意暂时不用的节点，不要把它加入当前 Workflow。

### 问题 3：下游节点没有执行

- 现象：上游节点执行了，但下游节点没有日志、没有事件、没有输出。
- 原因：可能是边条件为 false、消息类型不匹配、下游 handler 无法处理该消息，或没有调用 `SendMessageAsync` / 没有返回值。
- 排查方法：
  - 检查 `AddEdge` 的 condition。
  - 检查上游返回值类型与下游 `Executor<TInput>` 输入类型。
  - 用流式运行查看 `ExecutorCompletedEvent` 和 `WorkflowErrorEvent`。
- 修复方式：调整 condition、统一消息 DTO 类型，必要时增加适配 Executor 做类型转换。

### 问题 4：流程执行了但没有最终输出

- 现象：Executor 事件都正常，但调用方没有看到 `WorkflowOutputEvent`。
- 原因：没有调用 `WithOutputFrom`，或者最终节点没有 `YieldOutputAsync`，或者输出节点配置错了。
- 排查方法：检查 Workflow 构建代码中的 `WithOutputFrom(...)`，确认指定的是最终输出节点而不是中间节点。
- 修复方式：将最终节点加入 `WithOutputFrom`；对于 `Executor<TInput>` 无返回值节点，使用 `context.YieldOutputAsync`。

### 问题 5：共享状态读不到

- 现象：`ReadStateAsync` 返回 null，或抛出 “state not found”。
- 原因：key 不一致、scope 名不一致、读取发生在状态发布前，或写入节点没有实际执行。
- 排查方法：
  - 检查 `QueueStateUpdateAsync` 和 `ReadStateAsync` 的 key/scope 是否完全一致。
  - 检查是否存在条件边导致写入节点没有执行。
  - 注意 SuperStep 边界：其他 Executor 通常下一步才看到更新。
- 修复方式：统一状态常量，例如 sample 中的 `EmailStateConstants.EmailStateScope`、`FileContentStateConstants.FileContentStateScope`；必要时调整边结构，确保先写后读。

## 企业级落地提醒

- 稳定性：不要把所有业务逻辑塞进一个大 Executor。优先拆成输入输出明确的小节点，让失败点更容易定位。
- 可维护性：为每个 Executor 使用稳定、可读的 ID。边条件不要写成难以测试的匿名复杂表达式，可以抽成命名函数。
- 权限边界：Workflow 中不同节点可能访问不同外部系统。不要因为它们在同一个流程里就共享同一组高权限凭据。
- 安全风险：条件路由不能只依赖模型自由文本。高风险分支应使用结构化输出、显式枚举和人工确认。
- 成本控制：并行 fan-out 会放大模型调用次数。对高成本 Agent 节点要加预算、超时和最大轮次限制。
- 可观测性：优先使用 streaming、事件和后续 Observability sample，而不是只看最终输出。生产系统需要记录每个节点的输入摘要、输出摘要、耗时和错误。
- 可测试性：每个 Executor 应能单独测试；边条件应能用固定 DTO 做单元测试；完整 Workflow 应有端到端 golden case。
- 持久化与恢复：长流程、人工介入和外部系统写操作需要检查点与幂等设计。本章只建立 Workflow 认知，第 10 章会继续展开状态管理、持久化和恢复。
- 并发与复用：如果 Executor 内部持有可变字段，例如 sample 的 `_messages`，要确认 Workflow 实例是否会并发运行、是否会被复用，以及是否需要实现重置或改为状态存储。

## 本章小结

- 本章最重要的结论 1：Workflow 是多步骤 Agent 系统的流程层，本质是 Executor 节点、Edge 路由、状态和事件输出组成的可执行图。
- 本章最重要的结论 2：`WorkflowBuilder` 负责构建和校验图；`Workflow` 保存构建后的流程定义；`IWorkflowContext` 是运行时节点访问消息、输出、状态和事件的入口。
- 本章最重要的结论 3：真实业务流程要显式处理条件分支、并行汇总、共享状态和输出边界，否则系统很难测试、排错和演进。

下一章会进入多 Agent 协作设计。理解本章后，你已经知道“流程图怎么跑”；下一章要进一步回答“多个 Agent 在这个流程图中如何分工、协商和交接”。

## 本章建议产出物

- 一次 `_StartHere/01_Streaming` sample 运行记录。
- 一张你自己的 Workflow 调用链图：`StartExecutor -> 中间节点 -> 输出节点`。
- 一个最小改造实验记录：新增 `AppendSuffixExecutor` 或新增一个条件分支。
- 一份问题清单：记录 unreachable executor、无输出、状态读不到等问题的复现和修复方式。
- 一个综合案例增量草图：企业内部知识与工单协同 Agent 的工单分类 Workflow。

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
