# 第 14 章：Human-in-the-Loop 与审批机制

> 本章定位：把 Agent 从“可以自动调用工具和执行流程”，推进到“知道何时必须停下来等待人工确认”，建立高风险动作的审批、恢复与审计闭环。

## 本章目标

学完本章后，你应能够：

- 区分对话式人工确认、工具审批、工作流外部输入、Checkpoint 恢复这几类 Human-in-the-Loop 机制。
- 在仓库中定位 .NET 下与 `ApprovalRequiredAIFunction`、`ToolApprovalRequestContent`、`RequestInfoEvent`、`ExternalRequest`、`CheckpointManager` 相关的 sample 与源码入口。
- 亲手运行一个最小审批示例，并观察 Agent 因人工审批而中断、等待、恢复的执行过程。
- 明确哪些能力是仓库已有直接支持、哪些可以基于 sample 最小改造落地、哪些属于企业级理论延伸。
- 为“企业内部知识与工单协同 Agent 系统”补上高风险操作审批链路，例如：发起变更、批量修改工单、调用生产环境工具前必须人工确认。

## 本章与前后章节的关系

- 前置知识：第 8 章已经介绍 Workflow 与多步骤编排；第 10 章讨论了状态持久化与恢复；第 13 章介绍了 Tool、MCP 与外部系统集成。
- 本章解决：当 Agent 已经能查知识、调工具、跑流程之后，哪些步骤不能让模型直接决定，必须引入人类确认、审批或补充信息。
- 后续衔接：第 15 章会继续讨论业务场景建模与任务拆解；第 19 章会系统展开安全、权限、合规与治理。本章重点放在“人工介入如何进入执行链路”。

## 先建立整体认知

Human-in-the-Loop（HITL）不是简单地“弹个确认框”，而是在 Agent 执行链路中显式引入一个外部决策点，让系统在关键时刻暂停，把待执行信息交给人，再根据人工响应继续、拒绝、补充或改道。

在这个仓库里，HITL 主要有三种落点：

1. **Agent 工具审批**
   - 典型入口：`ApprovalRequiredAIFunction`。
   - 表现形式：模型想调用工具时，不是直接执行，而是先产出 `ToolApprovalRequestContent`。
   - 适合：删除、写入、外发通知、生产部署、跨系统写操作。

2. **Workflow 外部输入/人工响应**
   - 典型入口：`RequestPort`、`ExternalRequest`、`RequestInfoEvent`、`ExternalResponse`。
   - 表现形式：工作流执行到某一步时向外部世界发请求，等待人类或外部系统返回响应。
   - 适合：人工补充资料、客服确认、经理审批、人工兜底判断。

3. **Checkpoint + 人工介入后的恢复**
   - 典型入口：`CheckpointManager`、`RestoreCheckpointAsync(...)`。
   - 表现形式：工作流在等待人工时把状态保存下来，之后可以恢复继续执行。
   - 适合：长时审批、跨班次处理、服务重启后的续跑。

如果缺少这一层，系统会出现明显风险：

- 模型可能直接执行高风险工具调用。
- 流程等待人工时没有状态保存，服务重启后任务丢失。
- 无法记录“谁在什么时候批准了什么动作”。
- 工具调用与审批链路脱节，难以审计和追责。

本章的关键认知是：**审批不是 Prompt 语气问题，而是执行控制问题；Human-in-the-Loop 应该进入消息内容、工作流事件和状态恢复机制，而不是只停留在 UI 层。**

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：
  - `dotnet/samples/02-agents/Agents/Agent_Step01_UsingFunctionToolsWithApprovals`
  - `dotnet/samples/02-agents/Agents/Agent_Step11_Middleware`
  - `dotnet/samples/02-agents/Agents/Agent_Step19_InFunctionLoopCheckpointing`
  - `dotnet/samples/02-agents/AgentsWithFoundry/Agent_Step04_UsingFunctionToolsWithApprovals`
  - `dotnet/samples/02-agents/AGUI/Step04_HumanInLoop`
  - `dotnet/samples/03-workflows/HumanInTheLoop/HumanInTheLoopBasic`
  - `dotnet/samples/03-workflows/Checkpoint/CheckpointWithHumanInTheLoop`
  - `dotnet/samples/03-workflows/Agents/GroupChatToolApproval`
  - `dotnet/samples/03-workflows/Declarative/ToolApproval`
- 项目：
  - `Microsoft.Agents.AI`
  - `Microsoft.Agents.AI.Workflows`
  - `Microsoft.Agents.AI.Hosting.OpenAI`
  - `Microsoft.Agents.AI.Declarative`
- Sample：
  - `Agent_Step01_UsingFunctionToolsWithApprovals`：最小函数工具审批。
  - `HumanInTheLoopBasic`：最小工作流人工输入。
  - `CheckpointWithHumanInTheLoop`：人工输入 + checkpoint + 恢复。
  - `GroupChatToolApproval`：多 Agent 协作中的工具审批。
  - `ToolApproval.yaml`：声明式 Workflow 中的 MCP Tool 审批循环。
  - `AGUI/Step04_HumanInLoop`：服务端/客户端之间转译审批请求。
- 关键类型：
  - `ApprovalRequiredAIFunction`
  - `ToolApprovalRequestContent`
  - `ToolApprovalResponseContent`
  - `FunctionCallContent`
  - `RequestPort`
  - `ExternalRequest`
  - `ExternalResponse`
  - `RequestInfoEvent`
  - `CheckpointManager`
  - `CheckpointInfo`
- 关键接口：
  - `ICheckpointManager`
  - `ICheckpointStore<TStoreObject>`
  - `IExternalRequestSink`
- 关键扩展点：
  - `approvalRequest.CreateResponse(bool approved)`：把人工结果返回给 Agent。
  - `request.CreateResponse(...)`：把外部输入返回给 Workflow。
  - `CheckpointManager.CreateJson(...)`：把 checkpoint 存入 JSON Store。
  - `RestoreCheckpointAsync(...)`：从已保存的节点恢复执行。

### 仓库已有支持 / 可基于 sample 改造 / 理论延伸

#### 1. 仓库已有直接支持

- `ApprovalRequiredAIFunction` 把某个工具包装为“调用前必须审批”。
- `ToolApprovalRequestContent` / `ToolApprovalResponseContent` 表示审批请求与审批响应消息。
- `RequestInfoEvent` + `ExternalRequest` + `ExternalResponse` 支持 Workflow 级人工输入。
- `CheckpointManager.Default` 支持内存 checkpoint；`CheckpointManager.CreateJson(...)` 可配合持久化存储扩展。
- OpenAI Hosting 层支持把审批请求/响应转换为流式事件，相关代码见 `dotnet/src/Microsoft.Agents.AI.Hosting.OpenAI/Responses/Streaming/FunctionApprovalRequestEventGenerator.cs` 与 `FunctionApprovalResponseEventGenerator.cs`。

#### 2. 可基于 sample 最小改造实现

- 把“天气查询”替换成“创建变更单”“批量更新工单”“发送邮件通知”等企业内工具，并保留审批环节。
- 把控制台审批替换成 Web 页面、IM 机器人、工单系统审批中心。
- 把 `CheckpointManager.Default` 改成 JSON 文件存储或 Cosmos 存储，实现跨进程恢复。
- 把 `HumanInTheLoopBasic` 的猜数字输入改成“让人工选择处理策略”“补充工单字段”。

#### 3. 理论延伸与企业级补充

- 多级审批：例如班组长审批后再到运维经理审批。
- 动态审批策略：按工具类型、目标系统、风险等级、金额、环境动态决定是否审批。
- 审批 SLA、超时自动撤回、代理审批、补审计原因等治理能力。
- 审批与 IAM、工单系统、审计日志、合规留痕一体化。

推荐阅读顺序：先看 `Agent_Step01_UsingFunctionToolsWithApprovals`，再看 `HumanInTheLoopBasic`，然后看 `CheckpointWithHumanInTheLoop`，最后看 `GroupChatToolApproval` 与 `AGUI/Step04_HumanInLoop`。

## 本章在综合案例中的增量

主案例统一为：**企业内部知识与工单协同 Agent 系统**。

在上一章结束时，这个系统已经具备：

- 能检索企业知识库回答常见问题。
- 能调用只读工具查询工单、知识条目、服务状态。
- 能调用部分写操作工具与外部系统集成。

但它仍然缺一个关键能力：**高风险动作不能直接执行，必须停下来等待人类确认。**

本章新增的能力是：

- 为写操作工具加审批门禁。
- 为复杂工作流加入“人工补充信息/人工放行”节点。
- 为等待审批的任务加入 checkpoint，支持稍后恢复。

这会改变综合案例中的几个模块边界：

- `Tool Layer` 不再只有调用与返回，还要标记风险等级与审批要求。
- `Workflow Layer` 不再是一条纯自动流程，而是能暂停并等待外部响应。
- `State Layer` 不再只保存会话历史，还要保存审批中的任务状态与 checkpoint。
- `Governance Layer` 要记录审批人、审批时间、审批对象、审批结果。

如果不加入本章能力，综合案例会卡在这里：

- Agent 可以“建议”执行敏感操作，但不能安全地真正执行。
- 一旦进入人工确认，就只能靠临时对话维持状态，无法可靠恢复。
- 企业无法接受 Agent 在生产环境中直接调用高风险工具。

## 先跑通或先验证一个最小示例

### 示例目标

运行最小函数工具审批 sample，观察 Agent 在准备调用函数时先发出审批请求，待用户批准后再继续执行。

### 运行前提

- 已进入仓库根目录 `D:\code\agent-framework`。
- 已安装 .NET SDK，sample 项目目标框架见各自 `.csproj`。
- 已配置 Azure OpenAI 环境变量：
  - `AZURE_OPENAI_ENDPOINT`
  - `AZURE_OPENAI_DEPLOYMENT_NAME`，未配置时 sample 默认 `gpt-5.4-mini`
- 当前身份可通过 `DefaultAzureCredential` 访问对应 Azure OpenAI 资源。

### 操作步骤

1. 进入 `dotnet` 目录：

   ```bash
   cd D:\code\agent-framework\dotnet
   ```

2. 运行最小审批 sample：

   ```bash
   dotnet run --project samples/02-agents/Agents/Agent_Step01_UsingFunctionToolsWithApprovals/Agent_Step01_UsingFunctionToolsWithApprovals.csproj
   ```

3. 当控制台出现审批提示时，输入：

   ```text
   Y
   ```

4. 观察最终输出是否返回天气结果。

### 预期现象

- 第一次 `RunAsync(...)` 后，不会直接得到天气文本结果，而是会收集到 `ToolApprovalRequestContent`。
- 控制台会提示类似“agent would like to invoke the following function”。
- 输入 `Y` 后，sample 会把审批响应重新发回 Agent。
- Agent 才会继续执行工具并输出最终答案。

### 成功标志

- `approvalRequests.Count > 0`，说明审批请求已成功产生。
- 输入 `Y` 后流程继续而不是卡死。
- 最终控制台打印 `Agent: ... cloudy with a high of 15°C.` 一类结果。

### 进一步验证：工作流式人工输入

如果你想验证 Workflow 级 Human-in-the-Loop，而不依赖模型工具审批，可运行：

```bash
dotnet run --project samples/03-workflows/HumanInTheLoop/HumanInTheLoopBasic/HumanInTheLoopBasic.csproj
```

这个 sample 不依赖 LLM 工具审批，而是通过 `RequestInfoEvent` 把工作流请求抛给外部，再由控制台输入构造 `ExternalResponse` 回传。

## 核心概念拆解

### 1. `ApprovalRequiredAIFunction` 是什么

它是对普通 `AIFunction` 的一层包装，作用不是改变工具逻辑，而是改变执行策略：当模型要调用这个工具时，系统不会立即执行，而是先生成审批请求。

最小示例在 `dotnet/samples/02-agents/Agents/Agent_Step01_UsingFunctionToolsWithApprovals/Program.cs`：

```csharp
tools: [new ApprovalRequiredAIFunction(AIFunctionFactory.Create(GetWeather))]
```

这表示 `GetWeather` 虽然只是一个普通函数，但已经被声明为“需要人工批准后才能调用”。

### 2. `ToolApprovalRequestContent` 与 `ToolApprovalResponseContent` 是什么

这两个类型是审批机制的消息载体。

- `ToolApprovalRequestContent`：代表“系统请求人工审批某个工具调用”。
- `ToolApprovalResponseContent`：代表“人工已经批准或拒绝这个工具调用”。

在 sample 里，审批请求通过以下方式提取：

```csharp
List<ToolApprovalRequestContent> approvalRequests = response.Messages
    .SelectMany(m => m.Contents)
    .OfType<ToolApprovalRequestContent>()
    .ToList();
```

用户确认后，通过：

```csharp
functionApprovalRequest.CreateResponse(true)
```

创建审批响应，并作为新的用户消息再次送回 Agent。

### 3. Workflow 里的 `RequestPort` / `ExternalRequest` 是什么

如果说工具审批是“模型想执行工具时先问人”，那么 Workflow 的 `RequestPort` 是“流程设计者主动定义一个外部输入节点”。

在 `HumanInTheLoopBasic` 中，Workflow 运行过程中会发出 `RequestInfoEvent`：

```csharp
case RequestInfoEvent requestInputEvt:
    ExternalResponse response = HandleExternalRequest(requestInputEvt.Request);
    await handle.SendResponseAsync(response);
    break;
```

这里的含义是：

- 工作流先生成一个 `ExternalRequest`。
- Runner 把它包装成 `RequestInfoEvent` 暴露给外部世界。
- 外部代码读取请求内容，调用 `request.CreateResponse(...)` 生成 `ExternalResponse`。
- 再通过 `SendResponseAsync(...)` 把外部结果送回工作流。

这是一种比工具审批更泛化的 HITL 机制，因为它不仅可以用于“批准/拒绝”，也可以用于“请输入缺失字段”“请选择下一步处理路径”。

### 4. Checkpoint 在人工审批里为什么重要

审批很少总是秒回。真实企业里常见情况是：

- 需要值班经理半小时后批准。
- 审批发生在另一个系统里。
- 服务在等待过程中可能重启或扩容。

因此，仅有“暂停等待响应”还不够，还必须能保存当前运行状态。`CheckpointWithHumanInTheLoop` sample 展示了这一点：

```csharp
var checkpointManager = CheckpointManager.Default;
await using StreamingRun checkpointedRun = await InProcessExecution
    .RunStreamingAsync(workflow, new SignalWithNumber(NumberSignal.Init), checkpointManager);
```

当每个 super step 结束时，系统会自动生成 checkpoint；稍后可通过：

```csharp
await checkpointedRun.RestoreCheckpointAsync(savedCheckpoint, CancellationToken.None);
```

从某个节点恢复执行。

### 5. 它们之间如何协作

可以把整条 HITL 链路理解为三层：

1. **决策层**：哪些工具/步骤需要人参与。
2. **承载层**：审批请求通过什么内容类型或事件类型表达。
3. **恢复层**：等待期间的状态如何保存，返回后如何续跑。

对应到仓库：

- Agent 工具审批：`ApprovalRequiredAIFunction` + `ToolApprovalRequestContent`。
- Workflow 外部输入：`RequestInfoEvent` + `ExternalRequest` + `ExternalResponse`。
- 长时等待恢复：`CheckpointManager` + `CheckpointInfo` + `RestoreCheckpointAsync(...)`。

### 6. 常见误解是什么

- 误解一：审批只是 UI 逻辑。实际上仓库把审批表达为消息内容和事件，它属于执行链路的一部分。
- 误解二：只要 Prompt 说“高风险操作前先确认”就够了。Prompt 不是强制机制，`ApprovalRequiredAIFunction` 才是执行门禁。
- 误解三：有人工审批就不需要持久化。只要审批不是立即完成，就必须考虑 checkpoint 或会话持久化。
- 误解四：批准/拒绝已经覆盖所有 HITL 场景。很多真实场景需要的是“补充信息”“改参数”“转人工处理”。

## 结合源码理解执行过程

建议按“入口 -> 中间处理 -> 关键对象 -> 输出结果”的顺序理解。

### 1. 入口在哪里

#### Agent 工具审批入口

`dotnet/samples/02-agents/Agents/Agent_Step01_UsingFunctionToolsWithApprovals/Program.cs`：

```csharp
AIAgent agent = new AzureOpenAIClient(...)
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a helpful assistant",
        tools: [new ApprovalRequiredAIFunction(AIFunctionFactory.Create(GetWeather))]);
```

这里把一个普通函数放进 Agent，但同时声明它需要审批。

#### Workflow 人工输入入口

`dotnet/samples/03-workflows/HumanInTheLoop/HumanInTheLoopBasic/Program.cs`：

```csharp
await using StreamingRun handle = await InProcessExecution.RunStreamingAsync(workflow, NumberSignal.Init);
```

这会启动工作流流式执行，外部代码随后通过监听事件流来处理人工输入请求。

### 2. 调用是如何继续流转的

#### 工具审批链路

1. 用户发出问题，例如“Amsterdam 的天气如何”。
2. 模型决定要调用 `GetWeather`。
3. 因为工具被包装成 `ApprovalRequiredAIFunction`，系统不直接调用函数，而是返回 `ToolApprovalRequestContent`。
4. 调用方扫描 `response.Messages`，提取待审批内容。
5. 人类输入 `Y/N`。
6. 通过 `approvalRequest.CreateResponse(...)` 生成审批响应。
7. 再次调用 `agent.RunAsync(userInputResponses, session)`。
8. 若已批准，系统真正执行函数，再由模型生成最终回答。

这条链路也被单元测试验证。`dotnet/tests/Microsoft.Agents.AI.UnitTests/ChatClient/ChatClientAgent_ApprovalsTests.cs` 明确写出了两轮流程：

- Turn 1：用户请求 -> 模型返回 `FunctionCallContent` -> 转成 `ToolApprovalRequestContent`。
- Turn 2：用户回传 `ToolApprovalResponseContent` -> 系统执行函数 -> 继续模型调用 -> 产出最终回答。

#### Workflow 人工输入链路

1. 工作流运行到 `RequestPort`。
2. 运行时创建 `ExternalRequest`。
3. 对外发出 `RequestInfoEvent`。
4. 外部系统或人工处理该请求。
5. 用 `request.CreateResponse(...)` 生成 `ExternalResponse`。
6. `SendResponseAsync(...)` 把结果送回工作流。
7. Workflow 继续后续 executor 或输出最终结果。

#### Checkpoint 恢复链路

1. 工作流带 `checkpointManager` 执行。
2. 每个 super step 完成后产生 `CheckpointInfo`。
3. 调用方保存这些 checkpoint 元信息。
4. 需要恢复时，调用 `RestoreCheckpointAsync(savedCheckpoint, ...)`。
5. Runner 从该 checkpoint 继续产生事件流。

### 3. 关键对象分别负责什么

- `ApprovalRequiredAIFunction`：声明某个工具需要审批。
- `ToolApprovalRequestContent`：携带待审批工具调用的信息，包括底层 `FunctionCallContent`。
- `ToolApprovalResponseContent`：携带审批结果。
- `FunctionCallContent`：实际的工具调用名与参数。
- `RequestInfoEvent`：Workflow 向外部发起请求时暴露的事件。
- `ExternalRequest`：外部请求实体。
- `ExternalResponse`：外部返回实体。
- `CheckpointManager`：提交、检索 checkpoint 的统一入口。
- `CheckpointInfo`：checkpoint 元数据句柄。

### 4. 扩展点在哪里

#### 扩展点一：流式审批事件

`dotnet/src/Microsoft.Agents.AI.Hosting.OpenAI/Responses/Streaming/FunctionApprovalRequestEventGenerator.cs` 中：

```csharp
public override bool IsSupported(AIContent content) => content is ToolApprovalRequestContent;
```

当输出内容中包含 `ToolApprovalRequestContent` 时，会被转成 `response.function_approval.requested` 流式事件。这说明审批并不只是 SDK 内部对象，也可以进入前端事件流。

对应测试见：

- `dotnet/tests/Microsoft.Agents.AI.Hosting.OpenAI.UnitTests/FunctionApprovalTests.cs`

测试验证了：

- 事件类型为 `response.function_approval.requested`
- 会带 `request_id`
- 会带函数名与参数
- 事件序列在 `response.in_progress` 和 `response.completed` 之间

#### 扩展点二：服务端/客户端转译审批协议

`dotnet/samples/02-agents/AGUI/Step04_HumanInLoop/Server/ServerFunctionApprovalServerAgent.cs` 与 `Client/ServerFunctionApprovalClientAgent.cs` 展示了如何在前后端之间把审批请求转成 `request_approval` 工具调用模式。

这很重要，因为真实系统里审批不一定发生在同一个进程内，可能要跨 HTTP、WebSocket 或前端 SDK。

#### 扩展点三：声明式 MCP Tool 审批

`dotnet/samples/03-workflows/Declarative/ToolApproval/ToolApproval.yaml` 展示了声明式 Workflow 中的审批循环入口。虽然 YAML 本身更偏配置视角，但它说明审批可以进入声明式工作流，而不仅是手写 C#。同时，`dotnet/src/Microsoft.Agents.AI.Declarative/Extensions/McpServerToolApprovalModeExtensions.cs` 说明了 MCP Tool 审批模式可映射为：

- 从不要求审批
- 总是要求审批
- 对指定工具要求审批

## 关键代码或配置讲解

### 代码片段 1：最小函数审批包装

```csharp
AIAgent agent = new AzureOpenAIClient(
    new Uri(endpoint),
    new DefaultAzureCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a helpful assistant",
        tools: [new ApprovalRequiredAIFunction(AIFunctionFactory.Create(GetWeather))]);
```

- 作用：把普通函数工具升级为审批前置工具。
- 为什么重要：这是仓库中最直接、最容易复用的人工审批入口。
- 改动它会影响什么：如果去掉 `ApprovalRequiredAIFunction`，工具将直接执行，不再触发审批请求。

### 代码片段 2：处理审批请求并续跑

```csharp
while (approvalRequests.Count > 0)
{
    List<ChatMessage> userInputResponses = approvalRequests
        .ConvertAll(functionApprovalRequest =>
        {
            Console.WriteLine($"The agent would like to invoke the following function, please reply Y to approve: Name {((FunctionCallContent)functionApprovalRequest.ToolCall).Name}");
            return new ChatMessage(ChatRole.User, [functionApprovalRequest.CreateResponse(Console.ReadLine()?.Equals("Y", StringComparison.OrdinalIgnoreCase) ?? false)]);
        });

    response = await agent.RunAsync(userInputResponses, session);
    approvalRequests = response.Messages.SelectMany(m => m.Contents).OfType<ToolApprovalRequestContent>().ToList();
}
```

- 作用：实现“收到审批请求 -> 人工确认 -> 把确认结果送回 Agent -> 若仍有审批继续循环”的闭环。
- 为什么重要：它清楚展示了审批不是单次中断，而是可能多次往返。
- 改动它会影响什么：如果不把 `CreateResponse(...)` 产生的内容送回同一个 `session`，审批链路会断开。

### 代码片段 3：Workflow 人工输入处理

```csharp
case RequestInfoEvent requestInputEvt:
    ExternalResponse response = HandleExternalRequest(requestInputEvt.Request);
    await handle.SendResponseAsync(response);
    break;
```

- 作用：在 Workflow 层消费外部请求并回送外部响应。
- 为什么重要：它说明 HITL 不只用于工具审批，还可以是流程中的任意人工节点。
- 改动它会影响什么：如果不调用 `SendResponseAsync(...)`，工作流会一直停在等待状态。

### 代码片段 4：Checkpoint 恢复

```csharp
CheckpointInfo savedCheckpoint = checkpoints[CheckpointIndex];
await checkpointedRun.RestoreCheckpointAsync(savedCheckpoint, CancellationToken.None);
```

- 作用：把运行状态恢复到某个审批相关的历史节点。
- 为什么重要：审批通常跨时间，不可假设总在同一次进程生命周期内完成。
- 改动它会影响什么：如果 checkpoint 没有持久化到可靠存储，进程退出后将无法恢复。

## 做一个最小改造实验

### 实验目标

把最小函数审批示例从“天气查询审批”改造成“工单批量关闭审批”，观察审批文案、工具参数和拒绝分支的变化。

### 改造点

基于 `samples/02-agents/Agents/Agent_Step01_UsingFunctionToolsWithApprovals/Program.cs` 做最小改造：

1. 把工具方法改成：

   ```csharp
   [Description("Bulk close support tickets. Requires approval.")]
   static string CloseTickets(
       [Description("Comma-separated ticket ids")] string ticketIds,
       [Description("Closure reason")] string reason)
       => $"Tickets closed: {ticketIds}; reason: {reason}";
   ```

2. 保留：

   ```csharp
   new ApprovalRequiredAIFunction(AIFunctionFactory.Create(CloseTickets))
   ```

3. 把用户问题改成：

   ```text
   Please close tickets INC-1021, INC-1022 and INC-1023 as duplicate incidents.
   ```

4. 分别测试输入 `Y` 和 `N`。

### 你应该观察什么

- 审批请求里的函数名从 `GetWeather` 变成 `CloseTickets`。
- 若批准，Agent 会继续执行并给出关闭结果。
- 若拒绝，Agent 应转向解释未执行，或给出下一步建议。
- 你会直观看到：审批机制与具体业务工具无关，可以被快速迁移到企业场景。

### 可能出现的问题

- 模型不一定每次都选择调用工具，提示词可适度增强，例如要求“如需执行工单关闭，必须使用工具”。
- 如果拒绝后模型输出不稳定，可在 instructions 中补充“审批拒绝时明确告知未执行任何写操作”。
- 如果想验证长时等待，则需再进一步接入 Workflow Checkpoint，而不是只用单轮控制台审批。

## 可执行 / 可验证步骤补充

### 验证一：多 Agent 场景中的审批

运行：

```bash
dotnet run --project samples/03-workflows/Agents/GroupChatToolApproval/GroupChatToolApproval.csproj
```

观察点：

- QA 与 DevOps 两个 Agent 会协作推进部署任务。
- 当 DevOps Agent 尝试调用 `DeployToProduction` 时，会触发审批请求。
- sample 中自动批准后，流程继续完成。

这说明审批不仅适用于单 Agent，也适用于多 Agent Workflow。

### 验证二：人工输入 + checkpoint 恢复

运行：

```bash
dotnet run --project samples/03-workflows/Checkpoint/CheckpointWithHumanInTheLoop/CheckpointWithHumanInTheLoop.csproj
```

观察点：

- 每次 super step 完成后会输出 checkpoint 创建信息。
- sample 会在第一次完整跑完后，从第 2 个 checkpoint 恢复继续执行。
- 你可以验证人工输入与恢复是同一条链路中的两个环节。

## 关键执行链路

下面用“企业内部知识与工单协同 Agent 系统”中的“批量关闭工单”举例，串起一条完整链路：

1. 用户请求：请关闭一批重复工单。
2. Router Agent 判断该请求属于写操作，不可直接执行。
3. Specialist Agent 组织参数，准备调用 `CloseTickets` 工具。
4. 因该工具被包装为 `ApprovalRequiredAIFunction`，系统产生 `ToolApprovalRequestContent`。
5. 前端或审批中心收到审批请求，展示工单列表、原因、影响范围。
6. 审批人点击批准或拒绝。
7. 系统把结果包装为 `ToolApprovalResponseContent` 回传同一会话。
8. 若批准，工具执行；若拒绝，Agent 生成未执行说明与建议。
9. 若审批等待时间较长，Workflow 在等待态生成 checkpoint。
10. 后续恢复时，从 checkpoint 继续，无需重新规划整个任务。
11. 审计系统记录：请求人、审批人、工具名、参数摘要、审批结果、执行时间。

## 常见问题与排错

### 问题 1：运行 sample 后没有出现审批提示，而是直接得到最终答案
- 现象：控制台直接打印天气或其他结果，没有 `ToolApprovalRequestContent`。
- 原因：
  - 工具没有使用 `ApprovalRequiredAIFunction` 包装。
  - 模型没有选择调用工具，而是直接文本回答。
- 排查方法：
  - 检查 tools 列表是否为 `new ApprovalRequiredAIFunction(...)`。
  - 检查问题是否足够引导模型使用工具。
  - 查看 `response.Messages.SelectMany(m => m.Contents)` 是否真的存在审批内容。
- 修复方式：
  - 确保高风险工具统一用审批包装。
  - 调整 instructions，让模型在相关任务中优先使用工具。

### 问题 2：审批后再次调用 `RunAsync(...)` 没有继续执行
- 现象：输入 `Y` 后没有最终结果，或者响应与前一轮脱节。
- 原因：
  - 没有把审批响应发回同一个 `session`。
  - 构造返回消息时没有使用 `approvalRequest.CreateResponse(...)`。
- 排查方法：
  - 检查第二次 `RunAsync(...)` 是否仍使用原来的 `AgentSession`。
  - 检查传入消息内容是否为 `ToolApprovalResponseContent`。
- 修复方式：
  - 始终复用原会话。
  - 不要自己手写审批响应结构，优先使用 `CreateResponse(...)`。

### 问题 3：Workflow 一直卡在等待状态
- 现象：`WatchStreamAsync()` 只看到请求事件，后续没有输出。
- 原因：未调用 `SendResponseAsync(...)`，或者构造的 `ExternalResponse` 与请求不匹配。
- 排查方法：
  - 确认 `RequestInfoEvent` 分支中是否真正执行了 `handle.SendResponseAsync(response)`。
  - 检查 `request.CreateResponse(...)` 是否来自同一个 `request` 对象。
- 修复方式：
  - 对每个外部请求都显式返回响应。
  - 保持 request/response 一一对应。

### 问题 4：恢复 checkpoint 后状态不正确或无法恢复
- 现象：恢复失败、恢复后流程异常、找不到 checkpoint。
- 原因：
  - 使用内存 checkpoint，但进程已结束。
  - 自定义类型没有正确序列化配置。
- 排查方法：
  - 检查是否使用了 `CheckpointManager.Default`，它仅是内存实现。
  - 若状态包含自定义类型，检查 `CheckpointManager.CreateJson(...)` 的 `JsonSerializerOptions` 是否完整。
- 修复方式：
  - 生产场景不要依赖纯内存 checkpoint。
  - 为自定义消息和状态类型配置 JSON 序列化选项，必要时使用文件或数据库存储。

### 问题 5：前后端审批协议不一致
- 现象：前端收到了 `request_approval`，但客户端 Agent 无法识别，或服务端收不到正确审批结果。
- 原因：服务端/客户端对审批消息的转译协议不一致。
- 排查方法：
  - 参考 `AGUI/Step04_HumanInLoop` 中 `ServerFunctionApprovalServerAgent` 与 `ServerFunctionApprovalClientAgent` 的双向转换逻辑。
  - 检查 `approval_id`、`function_name`、`function_arguments` 字段是否一致。
- 修复方式：
  - 统一审批消息 schema。
  - 优先复用 sample 的转译模式，不要从零定义另一套协议。

## 企业级落地提醒

### 1. 稳定性

审批天然会拉长任务时长，所以必须考虑：

- 审批等待超时后如何处理。
- 服务重启后如何恢复等待中的任务。
- 审批消息是否幂等，避免重复批准导致重复执行。

### 2. 权限边界

不要把“审批通过”误解为“工具拥有无限权限”。正确做法是：

- 工具本身仍只拥有最小必要权限。
- 审批只是放行某次动作，不应扩大工具运行身份权限。
- 生产环境、测试环境、预发环境应使用不同身份与审批策略。

### 3. 安全风险

高风险写操作建议至少记录：

- 原始用户请求
- 模型生成的工具名与参数
- 审批人身份
- 审批意见与时间
- 最终执行结果

这样在 Prompt Injection、误操作、参数漂移、越权执行发生时，才能回溯责任链。

### 4. 可维护性

建议把审批策略从 Prompt 中抽离，单独建模，例如：

- 哪些工具必须审批
- 哪些参数组合必须升级审批级别
- 哪些用户角色可以直接通过
- 哪些情况必须转人工而不是继续自动执行

这样策略变更不需要频繁改 Prompt 或改每个 Agent 逻辑。

### 5. 可观测性

审批链路至少应暴露以下指标：

- 审批请求数
- 批准率 / 拒绝率
- 平均审批等待时长
- 审批后执行成功率
- 因超时取消的任务数

否则你无法判断系统到底是“自动化提效”，还是“把人工流程藏到了系统内部”。

### 6. 本章对综合案例的企业落地建议

在“企业内部知识与工单协同 Agent 系统”中，建议按风险分层：

- **无需审批**：知识检索、只读查询、FAQ 回答。
- **单人审批**：关闭工单、发送通知、更新标签、创建低风险变更。
- **双人或升级审批**：生产环境变更、批量操作、跨系统同步、影响客户的数据写入。
- **禁止自动化**：删除审计记录、导出敏感数据、越权权限变更。

## 本章小结

- 本章最重要的结论 1：Human-in-the-Loop 是执行控制机制，不是单纯的交互体验设计。
- 本章最重要的结论 2：仓库已经直接提供了工具审批、Workflow 外部输入、Checkpoint 恢复三类关键能力。
- 本章最重要的结论 3：企业要真正落地审批机制，必须把审批、状态持久化、审计和权限边界一起设计。

下一章之所以重要，是因为当你知道“哪些步骤需要人工介入”之后，还要进一步回答“整个业务任务到底该如何拆分、分配和建模”，这正是第 15 章要解决的问题。

## 本章建议产出物

- 一次 `Agent_Step01_UsingFunctionToolsWithApprovals` 的运行记录。
- 一张“审批请求 -> 人工响应 -> 工具执行 -> 最终回答”的调用链图。
- 一次“批量关闭工单审批”的最小改造实验记录。
- 一份高风险工具审批分级清单。

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
