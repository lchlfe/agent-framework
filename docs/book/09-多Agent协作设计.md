# 第 9 章：多 Agent 协作设计

> 本章定位：把第 8 章的 Workflow 编排能力，升级为多个具备不同职责的 Agent 共同完成复杂任务的协作系统。

## 本章目标

学完本章后，你应能够：

- 解释多 Agent 协作与单 Agent + 多 Tool、普通 Workflow 编排之间的区别。
- 在仓库中定位多 Agent、handoff、group chat、writer-critic、workflow as agent 等真实 sample。
- 使用 `AgentWorkflowBuilder`、`WorkflowBuilder`、`GroupChatManager` 等入口设计基础协作模式。
- 理解一次多 Agent 调用从用户输入到不同 Agent 响应、转交、汇总或审批的执行链路。
- 为综合案例“企业内部知识与工单协同 Agent 系统”增加分工协作能力。

## 本章与前后章节的关系

- 前置知识：第 6 章的 `AIAgent` / `ChatClientAgent` 核心抽象，第 7 章的会话与上下文，第 8 章的 Workflow 节点、边与运行方式。
- 本章解决：复杂任务不再由一个 Agent 一次性完成，而是拆给多个专家 Agent，通过顺序、并发、转交、群聊、评审循环等方式协同。
- 后续衔接：第 10 章会进一步处理多 Agent 协作中的任务状态、检查点、失败恢复和续跑问题；第 11 章会把这些协作模式提升为应用架构分层。

## 先建立整体认知

多 Agent 协作不是“多创建几个 Agent”这么简单，而是要回答四个问题：

1. **谁负责拆分任务**：是固定 Workflow，还是 triage/router Agent，还是自定义 manager。
2. **谁在什么时候发言或执行**：顺序执行、并发执行、条件分支、handoff 转交或 group chat 轮询。
3. **多个结果如何合并**：由下游 Agent 继续处理、由聚合 Executor 汇总，或由 critic/reviewer 进行质量门控。
4. **协作如何停止**：达到输出节点、审批通过、最大迭代次数、manager 判断完成，或发生错误后中断。

在 Agent 系统中，多 Agent 协作处在“任务编排层”和“智能决策层”之间：它既依赖 Workflow 的可控执行结构，也依赖每个 Agent 的 prompt、工具权限和模型能力。如果缺少这一层，复杂系统通常会出现两个问题：要么单 Agent prompt 过大、职责混乱；要么业务流程硬编码过多，难以根据语义灵活分派任务。

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：`dotnet/samples/03-workflows/`
- 项目：`dotnet/samples/03-workflows/_StartHere/*` 与 `dotnet/samples/03-workflows/Agents/*`
- Sample：
  - `dotnet/samples/03-workflows/_StartHere/02_AgentsInWorkflows/Program.cs`：把多个 `ChatClientAgent` 串成顺序翻译链。
  - `dotnet/samples/03-workflows/_StartHere/03_AgentWorkflowPatterns/Program.cs`：集中演示 `sequential`、`concurrent`、`handoffs`、`groupchat` 四类 Agent workflow pattern。
  - `dotnet/samples/03-workflows/_StartHere/07_WriterCriticWorkflow/Program.cs`：Writer / Critic / Summary 的评审迭代协作。
  - `dotnet/samples/03-workflows/Agents/GroupChatToolApproval/Program.cs`：QA 与 DevOps Agent 的 group chat 协作，并包含工具审批。
  - `dotnet/samples/03-workflows/Agents/GroupChatToolApproval/DeploymentGroupChatManager.cs`：自定义 `GroupChatManager` 选择下一位发言 Agent。
  - `dotnet/samples/03-workflows/Agents/WorkflowAsAnAgent/Program.cs` 与 `WorkflowFactory.cs`：把一个 workflow 包装成 `AIAgent`，用于更高层组合。
- 关键类型：`AIAgent`、`ChatClientAgent`、`AgentWorkflowBuilder`、`WorkflowBuilder`、`Workflow`、`StreamingRun`、`AgentResponseUpdateEvent`、`WorkflowOutputEvent`、`GroupChatManager`、`RoundRobinGroupChatManager`、`TurnToken`。
- 关键接口与上下文：`IWorkflowContext`、`IResettableExecutor`、`Executor<TIn,TOut>`、`MessageHandler`。
- 关键扩展点：`AgentWorkflowBuilder.BuildSequential`、`BuildConcurrent`、`CreateHandoffBuilderWith`、`CreateGroupChatBuilderWith`、`WithHandoffs`、`AddParticipants`、`GroupChatManager.SelectNextAgentAsync`、`workflow.AsAIAgent(...)`。

推荐阅读顺序：先看 `_StartHere/02_AgentsInWorkflows` 理解“Agent 可作为 workflow executor”；再看 `_StartHere/03_AgentWorkflowPatterns` 对比四种协作模式；然后看 `GroupChatToolApproval` 和 `07_WriterCriticWorkflow` 理解更接近真实业务的协作控制。

## 先跑通或先验证一个最小示例

### 示例目标

验证多个 Agent 可以在一个 workflow 中按指定协作模式工作，并能从事件流中看到不同 executor / agent 的输出。

### 运行前提

- 已安装 .NET SDK，并能在仓库根目录执行 `dotnet run`。
- 已配置 Azure OpenAI：
  - `AZURE_OPENAI_ENDPOINT`
  - `AZURE_OPENAI_DEPLOYMENT_NAME`，如果未设置，sample 默认使用类似 `gpt-5.4-mini` 的部署名。
- 本章 sample 多数使用 `AzureCliCredential` 或 `DefaultAzureCredential`，本地需要先完成 `az login`，或按环境替换认证方式。

### 操作步骤

1. 进入仓库根目录 `D:\code\agent-framework`。
2. 运行 Agent workflow pattern 示例：

   ```bash
   cd dotnet/samples/03-workflows/_StartHere/03_AgentWorkflowPatterns
   dotnet run
   ```

3. 程序提示：

   ```text
   Choose workflow type ('sequential', 'concurrent', 'handoffs', 'groupchat'):
   ```

   可分别输入：

   - `sequential`：观察翻译 Agent 按顺序处理。
   - `concurrent`：观察多个翻译 Agent 并发处理同一输入。
   - `handoffs`：观察 `triage_agent` 在数学 tutor 与历史 tutor 之间转交。
   - `groupchat`：观察多个 Agent 在 group chat manager 控制下轮流发言。

4. 也可以运行带审批的群聊示例：

   ```bash
   cd dotnet/samples/03-workflows/Agents/GroupChatToolApproval
   dotnet run
   ```

### 预期现象

- `03_AgentWorkflowPatterns` 会输出不同 Agent / executor 的名称，例如 `history_tutor`、`math_tutor` 或翻译 Agent 的 executor id。
- `GroupChatToolApproval` 会输出：
  - `Starting group chat workflow for software deployment...`
  - `Agents: [QAEngineer, DevOpsEngineer]`
  - `[APPROVAL REQUIRED]`，表示 DevOps Agent 触发生产部署工具前需要审批。

### 成功标志

- 你能看到多个 Agent 不是一次性混在同一个响应里，而是由 workflow 事件流逐步输出。
- 你能区分当前发言或执行的是哪个 Agent。
- 在审批示例中，工具调用被 `ToolApprovalRequestContent` 中断，并通过 `SendResponseAsync` 恢复执行。

## 核心概念拆解

### 1. 顺序协作：Sequential

顺序协作适合“前一步产物就是后一步输入”的任务。例如 `_StartHere/02_AgentsInWorkflows/Program.cs` 中：

```csharp
var workflow = new WorkflowBuilder(frenchAgent)
    .AddEdge(frenchAgent, spanishAgent)
    .AddEdge(spanishAgent, englishAgent)
    .Build();
```

三个翻译 Agent 按固定顺序执行。优点是可控、易排错；缺点是延迟叠加，且上游错误会传递到下游。

### 2. 并发协作：Concurrent / Fan-out Fan-in

并发协作适合“多个专家独立处理同一输入，再汇总结果”的任务。`_StartHere/03_AgentWorkflowPatterns/Program.cs` 使用：

```csharp
AgentWorkflowBuilder.BuildConcurrent(...)
```

`Agents/WorkflowAsAnAgent/WorkflowFactory.cs` 则展示了更显式的 fan-out / fan-in：

```csharp
return new WorkflowBuilder(startExecutor)
    .AddFanOutEdge(startExecutor, [frenchAgent, englishAgent])
    .AddFanInBarrierEdge([frenchAgent, englishAgent], aggregationExecutor)
    .WithOutputFrom(aggregationExecutor)
    .Build();
```

这里 `ConcurrentAggregationExecutor` 聚合两个 Agent 的结果，并实现 `IResettableExecutor`，避免同一个 workflow 多次运行时残留上一次的消息。

### 3. Handoff：由路由 Agent 转交给专家 Agent

Handoff 适合用户问题需要先判断领域，再交给对应专家。例如 `_StartHere/03_AgentWorkflowPatterns/Program.cs` 中：

```csharp
var workflow = AgentWorkflowBuilder.CreateHandoffBuilderWith(triageAgent)
    .WithHandoffs(triageAgent, [mathTutor, historyTutor])
    .WithHandoffs([mathTutor, historyTutor], triageAgent)
    .Build();
```

这里 `triage_agent` 的职责不是回答问题，而是判断该交给 `math_tutor` 还是 `history_tutor`。`.WithHandoffs([mathTutor, historyTutor], triageAgent)` 允许专家处理后再回到 triage，适合多轮对话或需要重新路由的场景。

### 4. Group Chat：由 Manager 控制多 Agent 发言

Group chat 适合“多个角色围绕同一个任务多轮协商”的场景。`03_AgentWorkflowPatterns` 使用 `RoundRobinGroupChatManager` 做轮询：

```csharp
AgentWorkflowBuilder.CreateGroupChatBuilderWith(
    agents => new RoundRobinGroupChatManager(agents) { MaximumIterationCount = 5 })
    .AddParticipants(...)
    .Build();
```

`GroupChatToolApproval` 则用自定义 `DeploymentGroupChatManager` 控制发言顺序：先 QA，再 DevOps。核心扩展点是：

```csharp
protected override ValueTask<AIAgent> SelectNextAgentAsync(
    IReadOnlyList<ChatMessage> history,
    CancellationToken cancellationToken = default)
```

这说明 group chat 的关键不只是“有哪些 Agent”，还包括“谁决定下一个 Agent”。

### 5. Writer-Critic：评审循环与质量门控

`_StartHere/07_WriterCriticWorkflow/Program.cs` 展示 Writer / Critic / Summary 协作：Writer 生成内容，Critic 用结构化输出判断是否通过；不通过则回到 Writer 修改，通过则进入 Summary 输出。

关键结构是：

```csharp
WorkflowBuilder workflowBuilder = new WorkflowBuilder(writer)
    .AddEdge(writer, critic)
    .AddSwitch(critic, sw => sw
        .AddCase<CriticDecision>(cd => cd?.Approved == true, summary)
        .AddCase<CriticDecision>(cd => cd?.Approved == false, writer))
    .WithOutputFrom(summary);
```

这个模式在企业系统里很常见：生成方案、审核风险、修改方案、最终确认。但必须设置最大迭代次数，例如 sample 中的 `MaxIterations = 3`，否则容易出现无限循环和成本失控。

### 6. Workflow as Agent：把协作系统封装成更大的 Agent

`Agents/WorkflowAsAnAgent/Program.cs` 中：

```csharp
var workflow = WorkflowFactory.BuildWorkflow(chatClient);
var agent = workflow.AsAIAgent("workflow-agent", "Workflow Agent");
var session = await agent.CreateSessionAsync();
```

这允许你把一个内部由多个 Agent 组成的 workflow，对外暴露成一个普通 `AIAgent`。这样上层系统不需要知道内部是并发、顺序还是聚合，只需要像调用一个 Agent 一样调用它。

## 结合源码理解执行过程

建议按“入口 -> 中间处理 -> 关键对象 -> 输出结果”的顺序阅读。

### 1. 入口在哪里

主要入口在 `Program.Main`：

- `_StartHere/03_AgentWorkflowPatterns/Program.cs`：根据控制台输入选择四种协作模式。
- `Agents/GroupChatToolApproval/Program.cs`：构造 QA 与 DevOps Agent，并运行群聊 workflow。
- `_StartHere/07_WriterCriticWorkflow/Program.cs`：构造 Writer、Critic、Summary 三个 executor，并建立条件循环。

这些 sample 都先创建 `IChatClient`，再用 `ChatClientAgent` 封装为具体角色。

### 2. 调用是如何继续流转的

以 `03_AgentWorkflowPatterns` 为例：

1. 创建 `ChatClientAgent`，为每个 Agent 配置 `instructions`、`name` 和可选描述。
2. 使用 `AgentWorkflowBuilder` 构建 `Workflow`。
3. 调用：

   ```csharp
   await using StreamingRun run = await InProcessExecution.RunStreamingAsync(workflow, messages);
   await run.TrySendMessageAsync(new TurnToken(emitEvents: true));
   ```

4. 通过 `run.WatchStreamAsync()` 监听事件。
5. 遇到 `AgentResponseUpdateEvent` 时，根据 `ExecutorId` 输出当前 Agent 的流式响应。
6. 遇到 `WorkflowOutputEvent` 时，将输出转换为 `List<ChatMessage>` 并返回。

注意 `TurnToken`：sample 注释说明 Agent 被包装为 executor 后，会缓存消息，收到 turn token 后才开始处理。这是理解 Agent-in-workflow 执行时机的关键。

### 3. 关键对象分别负责什么

- `ChatClientAgent`：具体专家角色，封装模型调用、instructions 和工具列表。
- `AgentWorkflowBuilder`：快速构建常见多 Agent 模式，例如顺序、并发、handoff、group chat。
- `WorkflowBuilder`：更底层、更通用的 workflow 图构建器，可组合 Agent 与自定义 Executor。
- `GroupChatManager`：决定 group chat 中下一位发言 Agent，并控制迭代次数等策略。
- `StreamingRun`：一次 workflow 运行实例，用于发送 token、响应审批请求、监听事件流。
- `AgentResponseUpdateEvent`：Agent 流式输出事件，可用于 UI 展示、日志和调试。
- `WorkflowOutputEvent`：workflow 产出最终结果的事件。
- `IWorkflowContext`：executor 读写状态、输出结果、参与 workflow 上下文交互的入口。

### 4. 扩展点在哪里

- 想新增专家角色：增加新的 `ChatClientAgent`，配置不同 prompt、名称、工具权限。
- 想改变固定顺序：修改 `BuildSequential` 的 Agent 列表或 `WorkflowBuilder.AddEdge`。
- 想改变路由规则：修改 `triageAgent` 的 instructions 或 `WithHandoffs` 拓扑。
- 想改变群聊发言策略：继承 `GroupChatManager` 并重写 `SelectNextAgentAsync`。
- 想增加质量门控：参考 Writer-Critic，用 `AddSwitch` 根据结构化结果路由。
- 想把复杂协作隐藏起来：参考 `workflow.AsAIAgent(...)`，对外暴露为单个 Agent。

## 关键代码或配置讲解

### 代码片段 1：多模式 AgentWorkflowBuilder

位置：`dotnet/samples/03-workflows/_StartHere/03_AgentWorkflowPatterns/Program.cs`

- 作用：集中演示顺序、并发、handoff、group chat 四种多 Agent 模式。
- 为什么重要：这是本章最推荐先看的入口，因为它把“协作拓扑”压缩在一个文件里，便于横向对比。
- 改动它会影响什么：调整 Agent 列表会改变参与者；调整 `WithHandoffs` 会改变可转交路径；调整 `MaximumIterationCount` 会改变 group chat 最多轮次。

### 代码片段 2：DeploymentGroupChatManager

位置：`dotnet/samples/03-workflows/Agents/GroupChatToolApproval/DeploymentGroupChatManager.cs`

- 作用：根据当前 group chat 的迭代次数选择 QA 或 DevOps Agent。
- 为什么重要：它展示了群聊不是完全失控的自然语言讨论，而是可以把发言权控制在代码策略里。
- 改动它会影响什么：如果把第一轮改为 DevOps，可能跳过测试先执行部署准备；如果提高或降低 `MaximumIterationCount`，可能影响是否有足够轮次完成测试、回滚计划和部署。

### 代码片段 3：Writer-Critic 的 AddSwitch

位置：`dotnet/samples/03-workflows/_StartHere/07_WriterCriticWorkflow/Program.cs`

- 作用：根据 `CriticDecision.Approved` 决定进入 Summary 还是回到 Writer。
- 为什么重要：它展示了多 Agent 协作中“结果不是只靠文本判断”，可以通过结构化输出驱动流程分支。
- 改动它会影响什么：如果取消最大迭代保护，可能导致循环无法收敛；如果 Critic 的 JSON schema 或 prompt 不稳定，流程分支可能失败。

## 做一个最小改造实验

### 实验目标

把 handoff 示例从“数学 / 历史家教”改造成综合案例中的“企业知识 / 工单处理”路由。

### 改造点

在 `dotnet/samples/03-workflows/_StartHere/03_AgentWorkflowPatterns/Program.cs` 的 `handoffs` 分支中，参考以下思路改 prompt 与 Agent 名称：

- `knowledgeAgent`：只回答公司制度、FAQ、内部知识相关问题。
- `ticketAgent`：只处理工单创建、状态查询、升级建议相关问题。
- `triageAgent`：判断用户请求是知识问答还是工单处理，并始终 handoff 给对应专家。

伪代码结构保持不变：

```csharp
var workflow = AgentWorkflowBuilder.CreateHandoffBuilderWith(triageAgent)
    .WithHandoffs(triageAgent, [knowledgeAgent, ticketAgent])
    .WithHandoffs([knowledgeAgent, ticketAgent], triageAgent)
    .Build();
```

### 你应该观察什么

- 输入“请解释 VPN 申请流程”，应倾向交给知识 Agent。
- 输入“帮我创建一个无法登录邮箱的工单”，应倾向交给工单 Agent。
- 多轮输入时，专家处理后是否能回到 triage 重新判断下一轮问题。

### 可能出现的问题

- 如果 triage prompt 没有明确“必须 handoff”，模型可能自己回答，导致专家 Agent 没有被调用。
- 如果两个专家职责描述重叠，handoff 可能不稳定。
- 如果缺少 `TurnToken`，Agent 作为 workflow executor 时可能不会按预期开始执行。

## 本章在综合案例中的增量

主案例仍然是：**企业内部知识与工单协同 Agent 系统**。

在前面章节中，它已经具备最小 Agent、Prompt、Tool、会话上下文和基础 Workflow。本章新增的是“多角色分工协作层”：

- 新增 `TriageAgent`：判断用户请求属于知识问答、工单处理、运维支持还是需要人工审批。
- 新增 `KnowledgeAgent`：负责解释政策、流程、FAQ，可在后续章节接入 RAG。
- 新增 `TicketAgent`：负责生成工单草稿、查询工单状态、建议优先级。
- 新增 `OpsAgent`：负责运维诊断、环境状态检查、回滚建议。
- 新增 `ReviewAgent` 或 `CriticAgent`：对高风险回复、变更建议、生产操作建议做质量门控。

推荐从仓库 sample 映射如下：

| 综合案例能力 | 仓库参考 | 属于哪类支持 |
| --- | --- | --- |
| 请求分流到知识或工单专家 | `_StartHere/03_AgentWorkflowPatterns` 的 handoff | 基于已有 sample 最小改造 |
| 多专家并发给出诊断建议 | `BuildConcurrent` 或 `WorkflowAsAnAgent` 的 fan-out/fan-in | 仓库已有 sample 直接支持 |
| 工单回复先生成再评审 | `_StartHere/07_WriterCriticWorkflow` | 仓库已有 sample 直接支持 |
| QA / DevOps 式运维协作与审批 | `Agents/GroupChatToolApproval` | 仓库已有 sample 直接支持 |
| 企业知识检索、真实工单 API | 后续 RAG / Tool / MCP 章节展开 | 框架可扩展，当前本章不深入 |

加入本章能力后，系统从“一个 Agent 处理所有请求”变为“入口 Agent 负责路由，专家 Agent 负责专业处理，评审 Agent 负责质量门控”。如果不加入这一层，综合案例会在职责边界、prompt 复杂度、工具权限隔离和高风险任务审核上很快失控。

## 常见问题与排错

### 问题 1：运行 sample 后没有 Agent 输出

- 现象：程序启动后没有看到预期的 Agent 响应。
- 原因：未发送 `TurnToken`，或 Azure OpenAI 环境变量 / 认证未配置成功。
- 排查方法：检查 sample 中是否执行了 `await run.TrySendMessageAsync(new TurnToken(emitEvents: true));`；检查 `AZURE_OPENAI_ENDPOINT`、`AZURE_OPENAI_DEPLOYMENT_NAME` 与 `az login`。
- 修复方式：补齐环境变量；完成 Azure 登录；不要删除 `TurnToken` 发送逻辑。

### 问题 2：handoff 没有转交给专家 Agent

- 现象：`triage_agent` 自己回答，或一直转给错误专家。
- 原因：triage prompt 不够强；专家描述不清晰；`WithHandoffs` 没有配置需要的转交边。
- 排查方法：打印 `AgentResponseUpdateEvent.ExecutorId`，确认实际执行的是哪个 Agent；检查 `ChatClientAgent` 的 `name` 和 description。
- 修复方式：在 triage instructions 中明确“ALWAYS handoff to another agent”；让专家职责互斥；补齐双向或必要转交路径。

### 问题 3：group chat 停不下来或过早结束

- 现象：Agent 轮流发言太多，成本上升；或任务还没完成就结束。
- 原因：`MaximumIterationCount` 设置不合理，或自定义 `GroupChatManager.SelectNextAgentAsync` 的策略过于简单。
- 排查方法：查看 manager 的迭代计数与 history；观察每轮 `ExecutorId`。
- 修复方式：设置明确最大轮次；在 manager 中加入完成条件；复杂场景不要只依赖 round-robin。

### 问题 4：Writer-Critic 循环无法收敛

- 现象：Critic 一直要求修改，Writer 反复生成，成本持续增加。
- 原因：Critic 标准过严；Writer 没有吸收反馈；缺少最大迭代保护。
- 排查方法：查看 `CriticDecision.Feedback` 与 `FlowState.Iteration`。
- 修复方式：保留 `MaxIterations`；让 Critic 的 approval 标准更明确；必要时在达到上限后进入人工审核而不是自动通过。

### 问题 5：并发聚合结果混入上一次运行

- 现象：同一个 workflow 多次运行时，聚合结果包含历史消息。
- 原因：共享的有状态 executor 没有实现或正确执行 `IResettableExecutor.ResetAsync`。
- 排查方法：参考 `Agents/WorkflowAsAnAgent/WorkflowFactory.cs` 的 `ConcurrentAggregationExecutor`。
- 修复方式：对共享有状态 executor 实现 `IResettableExecutor`，在 `ResetAsync` 中清理列表、缓存、计数器等状态。

## 企业级落地提醒

- 稳定性：多 Agent 会放大模型不确定性。关键流程不要只靠自然语言“商量”，要用 Workflow 边、handoff 拓扑、manager 策略和结构化输出约束。
- 可维护性：每个 Agent 的职责、工具权限、输入输出 schema 应单独管理。不要把所有业务规则塞进一个超长 system prompt。
- 权限边界：不同 Agent 应拥有不同工具集。参考 `GroupChatToolApproval`，高风险工具如生产部署、删除数据、发送通知，应使用审批包装或人工确认。
- 成本控制：并发和群聊会显著增加模型调用次数。必须设置最大轮次、最大迭代次数和必要的短路条件。
- 可观测性：记录 `ExecutorId`、handoff 路径、group chat 发言顺序、工具调用与审批结果，否则生产问题很难复盘。
- 可测试性：为 triage 路由准备固定测试集，例如“知识问题必须给 KnowledgeAgent”“工单问题必须给 TicketAgent”，避免 prompt 修改后路由退化。
- 持久化与恢复：本章只讨论协作模式本身；长流程、多轮审批、失败续跑需要结合第 10 章的状态管理、checkpoint 和 rehydrate 能力。

## 本章小结

- 本章最重要的结论 1：多 Agent 协作的核心不是 Agent 数量，而是职责拆分、转交流程、结果合并和停止条件。
- 本章最重要的结论 2：仓库已通过 `03-workflows` 下的多个 sample 展示顺序、并发、handoff、group chat、writer-critic、workflow as agent 等主要模式。
- 本章最重要的结论 3：企业级多 Agent 系统必须显式控制权限、轮次、状态和可观测性，不能依赖模型自由发挥。

下一章将继续处理：当多 Agent 协作变成长任务、跨轮次任务或审批中断任务时，状态如何保存、恢复与续跑。

## 本章建议产出物

- 一次 `03_AgentWorkflowPatterns` 的四种模式运行记录。
- 一张“用户 -> TriageAgent -> KnowledgeAgent / TicketAgent / OpsAgent -> ReviewAgent -> 输出”的调用链图。
- 一个将 handoff 示例改造成企业知识 / 工单路由的最小实验记录。
- 一份多 Agent 协作问题清单：路由错误、循环不收敛、工具权限过大、审批缺失、成本过高。

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
