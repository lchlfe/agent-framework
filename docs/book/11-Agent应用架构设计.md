# 第 11 章：Agent 应用架构设计

> 本章定位：把前面学到的单 Agent、Workflow、多 Agent、状态管理能力，组织成一个可扩展的应用级 Agent 架构。

## 本章目标

学完本章后，你应能够：

- 解释 Router、Planner、Specialist、Tool、状态层在 Agent 应用中的职责边界。
- 在本仓库中定位与应用架构相关的真实源码、sample 和关键类型。
- 基于已有 `dotnet/samples` 组合出“企业内部知识与工单协同 Agent 系统”的架构蓝图。
- 判断哪些能力是框架已经提供的，哪些需要在业务层补充实现。
- 设计一个可逐步演进的多 Agent + Workflow + Tool + 状态层调用链。

## 本章与前后章节的关系

- 前置知识：第 6 章的 `AIAgent` 抽象、第 8 章的 `WorkflowBuilder`、第 9 章的多 Agent 协作、第 10 章的 checkpoint 与状态恢复。
- 本章解决：如何把这些能力从“会用一个 sample”提升为“能设计一个完整应用架构”。
- 后续衔接：第 12 章会深入 Knowledge/RAG/Memory；第 13 章会深入 Tool、MCP 与外部系统；第 14 章会深入 Human-in-the-Loop；第 16 章以后会进入 Hosting、DurableTask 与生产化。

## 先建立整体认知

一个应用级 Agent 系统不应只有一个“什么都做”的 Agent。更推荐的分层是：

```text
用户入口
  ↓
Router / Triage：判断请求类型、风险级别、目标流程
  ↓
Planner：拆解任务、选择执行路径、决定是否进入 Workflow
  ↓
Specialist Agents / Executors：按领域完成检索、分析、生成、执行
  ↓
Tool 层：调用函数、MCP、外部 API、数据库、工单系统等
  ↓
状态层：保存会话状态、任务状态、共享状态、checkpoint、人工审批等待点
```

在 Agent Framework for .NET 里，这些分层不是一个固定的“应用模板项目”，而是由多个真实能力组合出来的：

- `AIAgent` 是所有 Agent 的基础抽象。
- `ChatClientAgent`、Foundry Agent、OpenAI Agent 等是不同后端实现。
- `WorkflowBuilder` / `AgentWorkflowBuilder` 提供编排能力。
- `GroupChatManager`、handoff builder、fan-out/fan-in edge 提供多 Agent 协作模式。
- `AITool`、`AIFunctionFactory.Create(...)`、`ApprovalRequiredAIFunction` 提供 Tool 调用与审批边界。
- `IWorkflowContext`、`StatefulExecutor<TState>`、`CheckpointManager`、`AgentSession` 提供状态能力。

如果缺少这一层设计，常见结果是：所有逻辑写进一个 Prompt；Tool 权限边界不清；任务无法恢复；多 Agent 互相抢活；上线后很难观测、调试和治理。

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：
  - `dotnet/src/Microsoft.Agents.AI.Abstractions`：Agent、Session、RunOptions、Response 等核心抽象。
  - `dotnet/src/Microsoft.Agents.AI`：`AIAgentBuilder`、`ChatClientAgent`、Logging/OpenTelemetry/FunctionInvocation 等装饰与扩展。
  - `dotnet/src/Microsoft.Agents.AI.Workflows`：Workflow 编排、Agent 编排、Router、Executor、状态与 checkpoint。
  - `dotnet/src/Microsoft.Agents.AI.DurableTask`：更偏生产长任务的 Durable Agent / Durable Workflow，后续第 16 章会深入。
  - `dotnet/src/Microsoft.Agents.AI.Hosting` 与 `dotnet/src/Microsoft.Agents.AI.Hosting.OpenAI`：服务化与 OpenAI 兼容接口，后续章节深入。
- 项目：
  - `Microsoft.Agents.AI.Abstractions`
  - `Microsoft.Agents.AI`
  - `Microsoft.Agents.AI.Workflows`
  - `Microsoft.Agents.AI.DurableTask`
- Sample：
  - `dotnet/samples/03-workflows/_StartHere/03_AgentWorkflowPatterns`：顺序、并发、handoff、group chat 四种 Agent 工作流模式。
  - `dotnet/samples/03-workflows/Agents/GroupChatToolApproval`：多 Specialist Agent + Tool + 人工审批。
  - `dotnet/samples/03-workflows/SharedStates`：Workflow 共享状态。
  - `dotnet/samples/03-workflows/Checkpoint/CheckpointAndResume`：checkpoint 与恢复。
  - `dotnet/samples/05-end-to-end/HostedAgents/AgentsInWorkflows`：把 Workflow 包装成一个 Agent。
- 关键类型：
  - `AIAgent`：核心 Agent 抽象。
  - `AgentSession`、`AgentRunOptions`、`AgentResponse`、`AgentResponseUpdate`：会话、运行参数和输出。
  - `AIAgentBuilder`：Agent 装饰管线，例如上下文、日志、OpenTelemetry 等。
  - `WorkflowBuilder`、`Workflow`、`Executor`、`StatefulExecutor<TState>`：Workflow 与执行节点。
  - `AgentWorkflowBuilder`：把多个 `AIAgent` 组合成 sequential、concurrent、handoff、group chat 模式。
  - `GroupChatManager`、`RoundRobinGroupChatManager`：多 Agent 轮转与发言管理。
  - `MessageRouter`：Workflow 内部按消息类型路由到 handler 的机制。
  - `IWorkflowContext`：状态读写、输出、外部请求等上下文入口。
  - `CheckpointManager`、`CheckpointInfo`、`StreamingRun`、`Run`：运行与恢复。
- 关键接口：
  - `IWorkflowContext`
  - `IWorkflowExecutionEnvironment`
  - `IExternalRequestContext`
  - `ICheckpointManager`、`ICheckpointStore`
- 关键扩展点：
  - 使用 `WorkflowBuilder.AddEdge`、`AddFanOutEdge`、`AddFanInBarrierEdge` 组织流程。
  - 使用 `AgentWorkflowBuilder.BuildSequential`、`BuildConcurrent`、`CreateHandoffBuilderWith`、`CreateGroupChatBuilderWith` 组织 Agent。
  - 使用 `AIFunctionFactory.Create(...)` 定义函数工具。
  - 使用 `ApprovalRequiredAIFunction` 给高风险工具增加审批。
  - 使用 `IWorkflowContext.QueueStateUpdateAsync`、`ReadStateAsync`、`StatefulExecutor<TState>` 管理状态。
  - 使用 `CheckpointManager.Default` 或自定义 checkpoint store 管理恢复点。

注意：仓库没有一个名为“Router/Planner/Specialist/Tool/State”的完整业务应用模板；这些是架构分层概念。本章会把它们映射到仓库已有的 Workflow、Agent、Tool、State 能力上。

## 先跑通或先验证一个最小示例

### 示例目标

验证“Router + Specialist”架构雏形：一个 triage agent 根据用户问题，把任务 handoff 给数学或历史专家。

### 运行前提

- 已安装 .NET SDK，能在 `dotnet` 目录下运行 sample。
- 已配置 Azure OpenAI：
  - `AZURE_OPENAI_ENDPOINT`
  - `AZURE_OPENAI_DEPLOYMENT_NAME`，sample 默认值中使用 `gpt-5.4-mini`，实际应替换为你环境中的 deployment。
- 已能通过 `AzureCliCredential` 或 sample 中使用的 credential 登录 Azure。

### 操作步骤

1. 进入 sample 目录：

   ```bash
   cd dotnet/samples/03-workflows/_StartHere/03_AgentWorkflowPatterns
   ```

2. 运行项目：

   ```bash
   dotnet run
   ```

3. 程序提示：

   ```text
   Choose workflow type ('sequential', 'concurrent', 'handoffs', 'groupchat'):
   ```

   输入：

   ```text
   handoffs
   ```

4. 在交互中分别输入数学问题和历史问题，例如：

   ```text
   Q: What is 12 * 18?
   Q: Why was the Treaty of Versailles important?
   ```

### 预期现象

- `triage_agent` 不直接回答，而是把任务转交给合适的 Specialist。
- 数学问题由 `math_tutor` 回答，历史问题由 `history_tutor` 回答。
- sample 中 `RunWorkflowAsync` 会打印执行事件里的 `ExecutorId`，你能看到不同 Agent 的输出切换。

### 成功标志

你能说明下面这段 sample 的架构意义：

```csharp
ChatClientAgent triageAgent = new(client,
    "You determine which agent to use based on the user's homework question. ALWAYS handoff to another agent.",
    "triage_agent",
    "Routes messages to the appropriate specialist agent");

var workflow = AgentWorkflowBuilder.CreateHandoffBuilderWith(triageAgent)
    .WithHandoffs(triageAgent, [mathTutor, historyTutor])
    .WithHandoffs([mathTutor, historyTutor], triageAgent)
    .Build();
```

这里的 `triage_agent` 就是 Router 的一种实现方式；`math_tutor` 和 `history_tutor` 是 Specialist；`AgentWorkflowBuilder` 把它们组合成可运行的 Workflow。

## 核心概念拆解

### 1. Router 是什么

Router 负责决定“请求该去哪里”，不负责完成所有业务。它可以有三种实现方式：

- Prompt 驱动的 Router：像 `03_AgentWorkflowPatterns` 里的 `triage_agent`，通过模型判断 handoff 目标。
- 规则驱动的 Router：用普通 C# `Executor` 或业务代码根据字段、意图、权限、风险级别路由。
- Workflow 内部消息 Router：源码中的 `MessageRouter` 根据消息类型查找 handler，这是框架内部机制，不等同于业务 Router，但帮助你理解 Workflow 如何把消息投递给不同节点。

源码入口：`dotnet/src/Microsoft.Agents.AI.Workflows/Execution/MessageRouter.cs`。

`MessageRouter.RouteMessageAsync(...)` 的核心行为是：

- 根据消息运行时类型查找 handler。
- 支持基类和接口 handler。
- 找不到精确类型时可以走 catch-all。
- handler 抛异常时会包装成 `CallResult.RaisedException(...)`。

业务 Router 可以借鉴这个思想：不要把所有请求塞给一个 Agent，而要先做类型、意图、权限、风险和目标流程的识别。

### 2. Planner 是什么

Planner 负责“如何完成任务”。在本仓库中，Planner 没有被固定成一个名为 `PlannerAgent` 的类型，但可以通过以下方式实现：

- 一个有明确 instructions 的 `ChatClientAgent`，输出计划或下一步动作。
- 一个 Workflow 中的 `Executor`，把输入拆成多个步骤。
- `AgentWorkflowBuilder.BuildSequential(...)` 表示固定计划。
- `BuildConcurrent(...)` 表示并行计划。
- handoff / group chat 表示动态协作计划。

例如 `AgentWorkflowBuilder.BuildSequential(...)` 在源码中会把多个 `AIAgent` 绑定成 executor，再通过 `WorkflowBuilder.AddEdge(previous, next)` 串起来。这就是一种“静态 Planner”：计划在构建 Workflow 时已经确定。

### 3. Specialist 是什么

Specialist 是面向某一领域或某一动作的专门 Agent/Executor。它应该具备清晰的边界：

- QA Specialist：只负责测试验证。
- DevOps Specialist：只负责部署准备与部署动作。
- Knowledge Specialist：只负责知识检索和基于证据回答。
- Ticket Specialist：只负责工单查询、创建、更新。

`dotnet/samples/03-workflows/Agents/GroupChatToolApproval/Program.cs` 中：

- `qaEngineer` 是 QA Specialist，拥有 `RunTests` tool。
- `devopsEngineer` 是 DevOps Specialist，拥有 `CheckStagingStatus`、`CreateRollbackPlan`、`DeployToProduction` tool。
- `DeploymentGroupChatManager` 是协作管理者，控制谁在什么时候发言。

这比“一个 Agent 拿所有工具”更容易控制权限、成本和风险。

### 4. Tool 层是什么

Tool 层是 Agent 与外部世界交互的边界。仓库里最直接的入口是：

```csharp
AIFunctionFactory.Create(RunTests)
AIFunctionFactory.Create(CheckStagingStatus)
new ApprovalRequiredAIFunction(AIFunctionFactory.Create(DeployToProduction))
```

Tool 应按副作用分级：

- 只读工具：查知识库、查工单、查 staging 状态。
- 低风险写工具：添加备注、创建草稿、生成报告。
- 高风险写工具：生产部署、关闭工单、退款、删除数据、发送外部通知。

对于高风险工具，`GroupChatToolApproval` 展示了 `ToolApprovalRequestContent` 的处理方式：当 `DeployToProduction` 被调用时，workflow 先产生审批请求，sample 再通过 `run.SendResponseAsync(...)` 模拟批准。

### 5. 状态层是什么

状态层至少包括四类：

- 会话状态：`AgentSession`，用于同一个 Agent 的多轮会话。
- Workflow 共享状态：`IWorkflowContext.QueueStateUpdateAsync`、`ReadStateAsync`。
- Executor 私有状态：`StatefulExecutor<TState>`。
- 运行恢复状态：`CheckpointManager`、`CheckpointInfo`。

`dotnet/samples/03-workflows/SharedStates/Program.cs` 展示了共享状态：

```csharp
await context.QueueStateUpdateAsync(fileID, fileContent,
    scopeName: FileContentStateConstants.FileContentStateScope,
    cancellationToken);

var fileContent = await context.ReadStateAsync<string>(message,
    scopeName: FileContentStateConstants.FileContentStateScope,
    cancellationToken);
```

`dotnet/samples/03-workflows/Checkpoint/CheckpointAndResume/Program.cs` 展示了 checkpoint：

```csharp
var checkpointManager = CheckpointManager.Default;
await using StreamingRun checkpointedRun =
    await InProcessExecution.RunStreamingAsync(workflow, NumberSignal.Init, checkpointManager);
```

在企业架构里，状态层不只是“存历史消息”，还要支持任务进度、审批等待、工具调用结果、幂等键、恢复点、审计记录。

## 结合源码理解执行过程

建议按“入口 -> 中间处理 -> 关键对象 -> 输出结果”的顺序理解。

### 1. 入口在哪里

以 `03_AgentWorkflowPatterns` 的 handoff 为例：

```csharp
await using StreamingRun run = await InProcessExecution.RunStreamingAsync(workflow, messages);
await run.TrySendMessageAsync(new TurnToken(emitEvents: true));
await foreach (WorkflowEvent evt in run.WatchStreamAsync())
{
    ...
}
```

入口对象是 `workflow`，运行方式是 `InProcessExecution.RunStreamingAsync(...)`，输出通过 `WorkflowEvent` 流观察。

### 2. 调用是如何继续流转的

一次请求的大致流转是：

```text
用户 ChatMessage
  → InProcessExecution.RunStreamingAsync
  → Workflow / Executor 图
  → AgentWorkflowBuilder 生成的 Agent executor
  → Router / triage agent 决定 handoff
  → Specialist agent 执行
  → 如需工具，则触发 AITool / AIFunction
  → 如需审批，则产生 RequestInfoEvent / ToolApprovalRequestContent
  → WorkflowOutputEvent 输出最终结果
```

如果是普通 Workflow executor，则消息会进入 `Executor` 的 handler。内部的 `MessageRouter` 会根据消息类型把消息路由到可处理的 handler。它不是业务 Router，但说明了 Workflow 执行层如何分发消息。

### 3. 关键对象分别负责什么

- `AIAgent`：对外暴露 `RunAsync`、`RunStreamingAsync`、`CreateSessionAsync` 等能力，是所有 Agent 的共同抽象。
- `AIAgentBuilder`：给 Agent 加中间层，例如上下文增强、日志、OpenTelemetry、函数调用行为。
- `WorkflowBuilder`：构建 executor 图。
- `AgentWorkflowBuilder`：把 `AIAgent` 快速组合成常见多 Agent 工作流模式。
- `GroupChatManager`：决定 group chat 中下一轮由谁发言、何时停止。
- `IWorkflowContext`：executor 读写状态、产出结果、请求外部输入的上下文。
- `CheckpointManager`：在 super step 结束时创建恢复点。

### 4. 扩展点在哪里

- 想换 Router：替换 triage agent 的 instructions，或写一个规则型 executor。
- 想换 Planner：把固定 `BuildSequential` 改为动态 handoff 或 group chat。
- 想加 Specialist：新增 `ChatClientAgent`，并通过 `WithHandoffs`、`AddParticipants` 或 `AddEdge` 接入。
- 想加 Tool：用 `AIFunctionFactory.Create(...)` 创建函数工具，并只挂给需要它的 Specialist。
- 想加状态：在 executor 中使用 `IWorkflowContext`，或继承 `StatefulExecutor<TState>`。
- 想加恢复：运行 workflow 时提供 `CheckpointManager`，生产环境再替换为持久化 checkpoint store。

## 关键代码或配置讲解

### 代码片段 1：handoff Router 与 Specialist

位置：`dotnet/samples/03-workflows/_StartHere/03_AgentWorkflowPatterns/Program.cs`

```csharp
ChatClientAgent triageAgent = new(client,
    "You determine which agent to use based on the user's homework question. ALWAYS handoff to another agent.",
    "triage_agent",
    "Routes messages to the appropriate specialist agent");

var workflow = AgentWorkflowBuilder.CreateHandoffBuilderWith(triageAgent)
    .WithHandoffs(triageAgent, [mathTutor, historyTutor])
    .WithHandoffs([mathTutor, historyTutor], triageAgent)
    .Build();
```

- 作用：构建一个由 Router 分发到 Specialist 的多 Agent workflow。
- 为什么重要：它展示了本章最核心的架构思想——路由与专家职责分离。
- 改动它会影响什么：改 `triageAgent` 的 instructions 会影响路由策略；增删 `WithHandoffs` 会影响可达的协作路径。

### 代码片段 2：GroupChat + Tool + 审批

位置：`dotnet/samples/03-workflows/Agents/GroupChatToolApproval/Program.cs`

```csharp
ChatClientAgent devopsEngineer = new(
    client,
    "You are a DevOps engineer responsible for deployments...",
    "DevOpsEngineer",
    "DevOps engineer who handles deployments",
    [
        AIFunctionFactory.Create(CheckStagingStatus),
        AIFunctionFactory.Create(CreateRollbackPlan),
        new ApprovalRequiredAIFunction(AIFunctionFactory.Create(DeployToProduction))
    ]);
```

- 作用：给 DevOps Specialist 只配置它需要的工具，并将生产部署标记为需要审批。
- 为什么重要：Tool 权限应绑定到角色，而不是绑定到全局 Agent。
- 改动它会影响什么：如果去掉 `ApprovalRequiredAIFunction`，生产部署会少一道人工确认边界；如果把部署工具给所有 Agent，会扩大权限风险。

### 代码片段 3：共享状态

位置：`dotnet/samples/03-workflows/SharedStates/Program.cs`

```csharp
await context.QueueStateUpdateAsync(fileID, fileContent,
    scopeName: FileContentStateConstants.FileContentStateScope,
    cancellationToken);
```

- 作用：把上游 executor 读到的文件内容存入共享状态，供下游 executor 读取。
- 为什么重要：复杂任务不能只靠消息文本传递所有中间结果，状态层可以承载结构化上下文。
- 改动它会影响什么：修改 `scopeName` 或 key 会影响后续 `ReadStateAsync` 是否能读到数据。

### 代码片段 4：checkpoint 与恢复

位置：`dotnet/samples/03-workflows/Checkpoint/CheckpointAndResume/Program.cs`

```csharp
CheckpointInfo savedCheckpoint = checkpoints[CheckpointIndex];
await checkpointedRun.RestoreCheckpointAsync(savedCheckpoint, CancellationToken.None);
```

- 作用：将 workflow 恢复到某个 checkpoint 后继续执行。
- 为什么重要：应用级 Agent 通常包含长任务和人工等待，必须能失败恢复。
- 改动它会影响什么：恢复点选择不同，后续执行路径和重复执行风险也不同。

## Router / Planner / Specialist / Tool / 状态层分工建议

| 层级 | 主要职责 | 仓库映射 | 不建议做什么 |
|---|---|---|---|
| Router | 分类、路由、风险判断、选择目标流程 | triage `ChatClientAgent`、规则型 `Executor`、handoff workflow | 不负责完成全部业务，不持有所有高风险工具 |
| Planner | 拆任务、决定顺序/并发/审批/恢复点 | `WorkflowBuilder`、`AgentWorkflowBuilder.BuildSequential/BuildConcurrent`、自定义 Planner Agent | 不直接执行外部副作用动作 |
| Specialist | 领域处理、生成、检索、分析、执行准备 | `ChatClientAgent`、`AIAgent`、自定义 `Executor` | 不跨越过多领域，不拿无关工具 |
| Tool | 外部系统调用、函数调用、MCP、数据库/API | `AIFunctionFactory.Create`、`AITool`、`ApprovalRequiredAIFunction` | 不隐藏高风险副作用，不跳过审批/审计 |
| 状态层 | 会话、任务进度、共享状态、checkpoint、审批等待 | `AgentSession`、`IWorkflowContext`、`StatefulExecutor<TState>`、`CheckpointManager` | 不只存自然语言历史，不忽略幂等与恢复 |

## 本章在综合案例中的增量

主案例仍然是：**企业内部知识与工单协同 Agent 系统**。

在前面章节中，它已经具备：

- 一个可对话的最小 Agent。
- 一些基础 Tool。
- 多轮会话与上下文管理。
- 简单 Workflow 和状态恢复概念。

本章加入的新能力是：**应用级分层架构**。

建议把主案例拆成以下模块：

```text
SupportRouterAgent
  - 判断请求是：知识问答 / 工单查询 / 工单创建 / 故障升级 / 高风险变更

SupportPlannerAgent 或 Workflow Planner Executor
  - 决定走单步回答、RAG 检索、工单流程、审批流程还是多 Agent 协作

KnowledgeSpecialistAgent
  - 负责知识检索与基于证据回答，后续第 12 章深入 RAG

TicketSpecialistAgent
  - 负责工单查询、创建草稿、追加备注，后续第 13 章深入外部系统集成

OpsSpecialistAgent
  - 负责运维诊断、运行只读检查工具，高风险修复需要审批

Tool 层
  - SearchKnowledgeBase：只读
  - GetTicketStatus：只读
  - CreateTicketDraft：低风险写
  - EscalateTicket / ExecuteRemediation：高风险写，需要审批

状态层
  - SupportConversationSession：会话状态
  - SupportTaskState：任务进度、分类结果、当前工单 ID、审批状态
  - Workflow checkpoint：长任务恢复点
```

这个增量修改了系统边界：从“一个 Agent 尝试回答所有问题”，升级为“Router 决定入口，Planner 决定流程，Specialist 执行领域任务，Tool 只按权限暴露，状态层负责连续性与恢复”。

它与仓库已有源码的关系：

- 仓库已有 sample 直接支持：handoff、group chat、function tool、approval、shared state、checkpoint。
- 可基于已有 sample 最小改造实现：把 `math_tutor/history_tutor` 改成 `KnowledgeSpecialistAgent/TicketSpecialistAgent/OpsSpecialistAgent`。
- 仓库未直接给出现成完整样例：完整企业工单系统 API、真实权限模型、真实 RAG 索引、审批后台；这些需要在第 12-14 章继续扩展。

如果不加入本章的架构分层，综合案例会卡在“能跑 demo，但不能治理复杂业务”的阶段：工具权限混乱、状态散落、任务无法清晰恢复、多 Agent 责任不清。

## 做一个最小改造实验

### 实验目标

把 `03_AgentWorkflowPatterns` 的 homework handoff 改造成主案例的“支持请求路由”雏形。

### 改造点

在本地实验分支中修改 `dotnet/samples/03-workflows/_StartHere/03_AgentWorkflowPatterns/Program.cs` 的 handoff 分支：

- 把 `historyTutor` 改名为 `knowledgeAgent`，instructions 改为“只回答企业知识库和内部政策类问题；不知道时说明需要检索”。
- 把 `mathTutor` 改名为 `ticketAgent`，instructions 改为“只处理工单查询、创建和状态解释类问题”。
- 把 `triageAgent` instructions 改为“判断用户请求应该交给知识专家还是工单专家；必须 handoff”。
- 把用户输入改成：
  - “How do I request VPN access?”
  - “Please check ticket INC-12345.”

### 你应该观察什么

- Router 是否能稳定把 VPN/政策类问题交给 Knowledge Specialist。
- Router 是否能把 INC 工单类问题交给 Ticket Specialist。
- Specialist 是否会越权回答自己领域外的问题。
- 如果 Router 选错，是否是 instructions 不够清晰，还是 Specialist 的 description 不够明确。

### 可能出现的问题

- 模型没有 handoff：检查 handoff builder 是否正确配置，Agent 是否支持工具调用，因为源码注释说明 handoff 依赖当前 Agent 调用 tool 的能力。
- Specialist 互相越权：收紧 instructions，并明确“只处理某类请求”。
- 多轮上下文混乱：检查 `messages` 是否持续追加，以及 Specialist 输出是否被加入对话。

## 常见问题与排错

### 问题 1：把 Router 写成了万能 Agent

- 现象：Router 既分类，又检索，又调用工单工具，还直接执行高风险动作。
- 原因：没有区分路由、规划、执行和工具层边界。
- 排查方法：列出每个 Agent 持有的 tools，看是否所有工具都挂在 Router 上。
- 修复方式：Router 只保留分类和 handoff 能力；把业务 tools 移到 Specialist；高风险 tools 包装为 `ApprovalRequiredAIFunction`。

### 问题 2：handoff 不发生

- 现象：`triage_agent` 自己回答，不交给 Specialist。
- 原因：instructions 不够强；provider 不支持或未正确传递 tools；handoff 路径没配置。
- 排查方法：查看 `AgentWorkflowBuilder.CreateHandoffBuilderWith(...)` 的 `WithHandoffs(...)` 是否包含目标 Agent；观察 `AgentResponseUpdateEvent` 中的 `ExecutorId` 是否切换。
- 修复方式：明确写“ALWAYS handoff”；检查 Agent 后端是否支持 function/tool calling；简化 Specialist 数量先验证两节点 handoff。

### 问题 3：状态读不到

- 现象：下游 executor 调用 `ReadStateAsync` 返回 null。
- 原因：key 或 `scopeName` 不一致；上游状态更新还未进入可见点；并发下状态隔离理解错误。
- 排查方法：对照 `SharedStates` sample，确认 `QueueStateUpdateAsync` 和 `ReadStateAsync` 使用相同 key/scope。
- 修复方式：统一状态 key 常量；把状态对象设计成结构化类型；必要时使用 `StatefulExecutor<TState>` 封装读写。

### 问题 4：恢复后重复执行外部动作

- 现象：从 checkpoint 恢复后，某个外部 API 或写操作被再次调用。
- 原因：checkpoint 只恢复 workflow 状态，不自动保证外部系统幂等。
- 排查方法：查看恢复点位于工具调用前还是调用后；检查工具是否传入幂等键。
- 修复方式：高风险 Tool 使用幂等 ID；把外部动作结果写入状态；恢复时先检查状态再决定是否重试。

### 问题 5：GroupChat 无限讨论或成本过高

- 现象：多个 Specialist 反复发言，迟迟没有最终输出。
- 原因：`GroupChatManager` 停止条件过宽；`MaximumIterationCount` 太高；任务没有明确完成判据。
- 排查方法：参考 `GroupChatToolApproval` 中 `MaximumIterationCount = 4`；观察每轮发言 agent 与内容。
- 修复方式：给 manager 增加明确停止规则；限制最大轮数；对于确定性流程优先使用 sequential workflow。

## 企业级落地提醒

- 稳定性：Router 和 Planner 的输出会影响后续路径，关键场景要增加规则兜底和失败回退流程。
- 可维护性：每个 Specialist 应有单独 instructions、tool 列表、测试用例和 owner，不要把所有业务逻辑写在一个大 Prompt 里。
- 权限边界：Tool 按 Agent 最小授权；高风险工具使用审批包装；外部系统仍要做服务端权限校验，不能只依赖 Prompt。
- 安全风险：`AIAgent` 源码注释明确提醒，用户消息、assistant 输出、tool 消息都应视为不可信；渲染、执行、入库前必须校验和清洗。
- 成本控制：Router 可用轻量模型或规则先过滤；只有复杂请求进入 Planner 或多 Agent group chat。
- 可观测性：为 Router 决策、Planner 计划、Tool 调用、审批结果、checkpoint id 打日志；后续第 17 章会系统展开。
- 可测试性：为每类请求准备路由评测集，验证“知识问答/工单/运维/审批”是否进入正确路径。
- 持久化与恢复：开发阶段可用 `CheckpointManager.Default`；生产应考虑 DurableTask、Cosmos checkpoint store 或其他持久化实现，并设计幂等。

## 本章小结

- 本章最重要的结论 1：应用级 Agent 架构应分层，Router、Planner、Specialist、Tool、状态层各有职责，不能让一个 Agent 承担所有事情。
- 本章最重要的结论 2：仓库没有固定业务模板，但已经提供了组合这些分层的真实能力：`AIAgent`、`WorkflowBuilder`、`AgentWorkflowBuilder`、`GroupChatManager`、`AITool`、`IWorkflowContext`、`CheckpointManager`。
- 本章最重要的结论 3：主案例“企业内部知识与工单协同 Agent 系统”在本章从应用原型升级为分层架构蓝图，后续将继续补齐 RAG、外部系统、审批与生产化。

下一章将进入 Knowledge、RAG 与 Memory 体系设计，重点解决 Knowledge Specialist 如何基于企业知识库可靠回答，而不是只靠模型记忆。

## 本章建议产出物

- 一次 `03_AgentWorkflowPatterns` 的 handoff 运行记录。
- 一张主案例分层架构图。
- 一个把 homework handoff 改成支持请求路由的最小改造实验记录。
- 一份 Router/Planner/Specialist/Tool/状态层职责清单。
- 一份高风险 Tool 审批与幂等问题清单。

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
