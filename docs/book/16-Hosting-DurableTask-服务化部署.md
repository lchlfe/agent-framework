# 第 16 章：Hosting、DurableTask 与服务化部署

> 本章定位：把前面完成的 Agent 原型放进可托管、可恢复、可部署的运行环境中，理解从本地 demo 到生产服务之间缺的那一层。

## 本章目标

学完本章后，你应能够：

- 说明 Agent Framework 中 Hosting、Azure Functions Hosting、DurableTask 三者分别解决什么问题。
- 在仓库中定位服务化部署相关源码与 sample，尤其是 `dotnet/samples/04-hosting` 与 `dotnet/src/Microsoft.Agents.AI.*Hosting*`。
- 跑通或验证一个 Azure Functions Durable Agent 示例，并理解 HTTP 请求如何进入 durable agent session。
- 判断哪些能力是仓库已提供的，哪些是生产部署时需要业务侧补齐的边界。
- 为“企业内部知识与工单协同 Agent 系统”补上服务入口、长任务持久化与生产运行边界。

## 本章与前后章节的关系

- 前置知识：第 8-10 章已经讲过 Workflow、状态持久化与恢复；第 11-15 章已经把 Agent 放到业务架构、工具、审批和任务拆解中。
- 本章解决：如何把这些能力挂到真实 host 中，让它不只是控制台 sample，而是能通过 HTTP、Azure Functions、Durable Task 被外部调用、恢复和运维。
- 后续衔接：第 17 章会进一步展开日志、OpenTelemetry、调试和故障排查；第 18-19 章会补评测、安全、权限、合规与成本治理。本章只建立部署和运行边界，不把观测与治理细节讲满。

## 先建立整体认知

服务化部署不是“把 `dotnet run` 换成云上运行”这么简单。对 Agent 系统来说，生产运行至少要回答四个问题：

- 入口在哪里：用户、系统或其他 Agent 通过 HTTP、A2A、MCP 或后台任务调用 Agent。
- 生命周期由谁管理：模型客户端、工具、Agent 实例、会话状态是否由依赖注入和 host 统一管理。
- 长任务如何恢复：模型调用、工具调用、人工审批、工作流节点可能跨请求、跨进程、跨重启继续执行。
- 失败边界在哪里：哪些异常可以重试，哪些必须返回给调用方，哪些需要人工介入。

在本仓库中，与本章最相关的是三条线：

1. **通用 Hosting**：`Microsoft.Agents.AI.Hosting` 提供把 Agent 注册进 `IServiceCollection` 的能力。
2. **Azure Functions Hosting**：`Microsoft.Agents.AI.Hosting.AzureFunctions` 提供把 Agent 暴露为 Functions 触发器与 Durable Agent 的能力。
3. **DurableTask 支撑**：`Microsoft.Agents.AI.DurableTask` 提供 Durable Agent、Durable Workflow、session 与状态序列化等能力。

如果缺少这一层，前面章节做出的系统会停留在“单进程内可运行”的状态：服务重启后会话丢失、长任务无法恢复、外部系统无法通过稳定 API 调用、生产扩缩容和排错也没有清晰边界。

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：
  - `dotnet/samples/04-hosting`
  - `dotnet/samples/04-hosting/DurableAgents`
  - `dotnet/samples/04-hosting/DurableWorkflows`
  - `dotnet/samples/04-hosting/A2A`
  - `dotnet/samples/01-get-started/06_host_your_agent`
  - `dotnet/samples/03-workflows/Observability/AspireDashboard`
  - `dotnet/src/Microsoft.Agents.AI.Hosting`
  - `dotnet/src/Microsoft.Agents.AI.Hosting.AzureFunctions`
  - `dotnet/src/Microsoft.Agents.AI.DurableTask`
  - `dotnet/src/Microsoft.Agents.AI.Workflows`
- 项目：
  - `Microsoft.Agents.AI.Hosting`
  - `Microsoft.Agents.AI.Hosting.AzureFunctions`
  - `Microsoft.Agents.AI.Hosting.A2A`
  - `Microsoft.Agents.AI.Hosting.A2A.AspNetCore`
  - `Microsoft.Agents.AI.Hosting.AGUI.AspNetCore`
  - `Microsoft.Agents.AI.DurableTask`
- Sample：
  - `dotnet/samples/04-hosting/DurableAgents/AzureFunctions/01_SingleAgent`
  - `dotnet/samples/04-hosting/DurableAgents/AzureFunctions/02_AgentOrchestration_Chaining`
  - `dotnet/samples/04-hosting/DurableAgents/AzureFunctions/03_AgentOrchestration_Concurrency`
  - `dotnet/samples/01-get-started/06_host_your_agent`
  - `dotnet/samples/04-hosting/A2A/*`
  - `dotnet/samples/03-workflows/Observability/AspireDashboard`
- 关键类型：
  - `AgentHostingServiceCollectionExtensions`
  - `DurableAgentsOptionsExtensions`
  - `DurableTaskClientExtensions`
  - `DurableAIAgent`
  - `DurableAIAgentProxy`
  - `DurableAgentSession`
  - `DurableAgentRunOptions`
  - `DefaultDurableAgentClient`
  - `DurableWorkflowClient`
  - `DurableWorkflowRunner`
- 关键接口：
  - `IDurableAgentClient`（内部接口，用于 durable agent 调用抽象）
  - `IHostedAgentBuilder`
  - `IChatClient`
- 关键扩展点：
  - `services.AddAIAgent(...)`
  - `ConfigureDurableAgents(options => options.AddAIAgent(...))`
  - `ConfigureDurableAgents(options => options.AddAIAgentFactory(...))`
  - `DurableTaskClient.AsDurableAgentProxy(...)`

需要明确的是：仓库已经提供 Azure Functions + DurableTask 的示例和框架扩展；生产级部署清单、云资源编排、权限模型、CI/CD、灾备策略则主要需要业务项目自行补齐。本章会基于已有 sample 推导这些生产边界，而不是假装仓库已经提供完整平台化部署方案。

## 先跑通或先验证一个最小示例

### 示例目标

用 `dotnet/samples/04-hosting/DurableAgents/AzureFunctions/01_SingleAgent` 验证一个 Agent 被 Azure Functions 托管后，可以通过 HTTP 端点调用，并由 Durable Agent 机制管理会话。

### 运行前提

- 已安装 .NET SDK，版本以仓库要求为准。
- 已安装 Azure Functions Core Tools，用于本地运行 Functions 应用。
- 准备 Azure OpenAI 环境变量：
  - `AZURE_OPENAI_ENDPOINT`
  - `AZURE_OPENAI_DEPLOYMENT_NAME`
  - 可选：`AZURE_OPENAI_KEY`。未设置 key 时，sample 使用 `DefaultAzureCredential`，本地通常需要 `az login`。
- Durable Task 后端依赖按 sample 父级 README 配置。sample 引用了 `Microsoft.Azure.Functions.Worker.Extensions.DurableTask.AzureManaged`，运行环境需要对应 Durable Task provider 可用。

### 操作步骤

进入 sample 目录：

```powershell
cd dotnet/samples/04-hosting/DurableAgents/AzureFunctions/01_SingleAgent
```

设置环境变量后启动 Functions：

```powershell
$env:AZURE_OPENAI_ENDPOINT="https://<your-resource>.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT_NAME="<your-deployment>"
# 如果使用 key：
$env:AZURE_OPENAI_KEY="<your-key>"

func start
```

在另一个终端调用 HTTP 端点：

```powershell
Invoke-RestMethod -Method Post `
  -Uri http://localhost:7071/api/agents/Joker/run `
  -ContentType text/plain `
  -Body "Tell me a joke about a pirate."
```

也可以使用 sample 自带的 `demo.http`：

```http
POST http://localhost:7071/api/agents/Joker/run
Content-Type: text/plain

Tell me a joke about a pirate.
```

### 预期现象

- Functions 控制台能看到应用启动、HTTP 请求进入以及 Durable Task 相关执行日志。
- HTTP 响应返回由名为 `Joker` 的 Agent 生成的文本。
- 如果使用 JSON 请求并设置 `Accept: application/json`，响应中会包含类似 `thread_id`、`response.Messages`、`Usage` 的结构。

### 成功标志

- `POST /api/agents/Joker/run` 返回 200。
- 返回内容符合 `JokerInstructions = "You are good at telling jokes."` 的行为预期。
- 带上 `thread_id` 继续请求时，同一会话可以延续上下文。

## 核心概念拆解

### 1. Hosting 是什么

Hosting 是把 Agent 从“一个对象”变成“一个可被 host 管理的服务”的过程。它负责：

- 注册 Agent、模型客户端、工具与依赖服务。
- 管理生命周期，例如 Singleton、Scoped、Transient。
- 统一暴露调用入口，例如 HTTP、A2A、AG-UI、Azure Functions trigger。
- 给观测、配置、鉴权、部署、扩缩容留下接入点。

源码入口是 `dotnet/src/Microsoft.Agents.AI.Hosting/AgentHostingServiceCollectionExtensions.cs`。其中 `AddAIAgent` 会把 Agent 按 name 注册为 keyed service，并返回 `IHostedAgentBuilder`，供后续挂载工具等扩展。

### 2. Azure Functions Hosting 是什么

Azure Functions Hosting 是面向 serverless/function app 的托管方式。它让 Agent 以 Functions 应用的形式运行，并自动生成可调用端点或触发器。

在 `01_SingleAgent/Program.cs` 中，核心代码是：

```csharp
using IHost app = FunctionsApplication
    .CreateBuilder(args)
    .ConfigureFunctionsWebApplication()
    .ConfigureDurableAgents(options => options.AddAIAgent(agent, timeToLive: TimeSpan.FromHours(1)))
    .Build();
app.Run();
```

这段代码表达了三个动作：

- 创建 Functions host。
- 启用 Functions Web Application。
- 把 `Joker` 这个 `AIAgent` 注册为 Durable Agent，并设置会话或状态的生存时间边界。

`DurableAgentsOptionsExtensions` 中的 `AddAIAgent` 默认启用 HTTP trigger；也可以通过重载控制 HTTP trigger 与 MCP tool trigger 是否启用。

### 3. DurableTask 是什么

DurableTask 提供可持久化、可恢复的编排能力。对 Agent 系统而言，它主要解决：

- 长任务不应依赖单个 HTTP 请求生命周期。
- 多步骤工作流需要在失败、重启、等待人工输入后恢复。
- 会话状态需要被序列化并跨调用保存。
- Agent 调用需要可被编排器、实体或 client 代理协调。

`DurableAIAgent` 是关键类型之一。它在 `RunCoreAsync` 中检查 session 是否为 `DurableAgentSession`，然后通过 Durable Task entity 调用 `AgentEntity.Run`。这说明 durable agent 的真实执行不是简单地在当前 HTTP 请求线程里直接跑完，而是交给 Durable Task 的实体/编排机制协调。

### 4. 它们之间如何协作

一次最小调用可以理解为：

```text
HTTP Client
  -> Azure Functions HTTP endpoint
  -> Durable Agent trigger/metadata
  -> DurableTaskClient / DurableAIAgentProxy
  -> Durable Task entity or orchestration
  -> registered AIAgent
  -> IChatClient / tools
  -> AgentResponse
  -> HTTP response
```

其中：

- Hosting 负责“注册和暴露”。
- Azure Functions 负责“接收请求和运行函数应用”。
- DurableTask 负责“持久化执行、session、恢复和编排”。
- Agent Framework 负责“模型调用、工具调用、消息与响应抽象”。

### 5. 常见误解是什么

- 误解一：只要用了 DurableTask 就天然生产可用。实际还需要配置后端存储、权限、监控、限流、重试、超时、密钥管理和成本保护。
- 误解二：Durable Agent 等于流式响应。源码中 `DurableAIAgent.RunCoreStreamingAsync` 明确说明 streaming 不被 durable agents 原生支持，而是把完整响应转换成单个 update。
- 误解三：`DefaultAzureCredential` 可以直接用于所有生产环境。sample 注释已经提醒，生产中应优先考虑明确的凭据策略，例如 Managed Identity，避免 fallback probing 带来的延迟和安全风险。
- 误解四：HTTP endpoint 就是完整 API 产品。生产 API 还需要鉴权、配额、幂等、schema、版本管理和审计。

## 结合源码理解执行过程

### 1. 入口在哪里

最小 Azure Functions 入口在：

- `dotnet/samples/04-hosting/DurableAgents/AzureFunctions/01_SingleAgent/Program.cs`

它创建 Azure OpenAI client：

```csharp
AzureOpenAIClient client = !string.IsNullOrEmpty(azureOpenAiKey)
    ? new AzureOpenAIClient(new Uri(endpoint), new AzureKeyCredential(azureOpenAiKey))
    : new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential());
```

然后构造 Agent：

```csharp
AIAgent agent = client.GetChatClient(deploymentName).AsAIAgent(JokerInstructions, JokerName);
```

最后通过 `ConfigureDurableAgents` 注册到 Functions host。

### 2. 调用是如何继续流转的

HTTP 请求命中 `/api/agents/Joker/run` 后，Azure Functions Hosting 根据注册的 agent name 找到对应 Durable Agent 配置。`DurableTaskClientExtensions.AsDurableAgentProxy` 展示了代理创建过程：

- 校验 `durableClient`、`FunctionContext`、`agentName`。
- 调用 `ValidateAgentIsRegistered` 确认 agent 已注册。
- 创建 `DefaultDurableAgentClient`。
- 返回 `DurableAIAgentProxy(agentName, agentClient)`。

这说明在 Azure Functions 场景里，外部请求面对的是代理对象；代理对象再把请求投递给 Durable Task 管理的 agent 执行单元。

### 3. 关键对象分别负责什么

- `AgentHostingServiceCollectionExtensions`：把 Agent 作为 keyed service 注册进 DI，保证 name 与 agent.Name 一致。
- `DurableAgentsOptionsExtensions`：在 Azure Functions durable agents 配置阶段添加 agent 或 agent factory，并记录 HTTP/MCP trigger 选项。
- `DurableTaskClientExtensions`：把 `DurableTaskClient` 转换成 durable agent proxy。
- `DurableAIAgent`：在 Durable Task orchestration context 中创建 `DurableAgentSession`，并通过 entity 调用运行 agent。
- `DurableAgentSession`：表达 durable agent 的会话标识和可序列化 session。
- `DurableAgentRunOptions`：控制 durable agent 运行选项，例如工具调用开关、允许的工具名等。
- `DurableWorkflowClient` / `DurableWorkflowRunner`：面向 durable workflow 的运行与状态管理入口，适合多节点工作流。

### 4. 扩展点在哪里

- 如果只是托管单个 Agent：优先从 `ConfigureDurableAgents(options => options.AddAIAgent(...))` 开始。
- 如果 Agent 依赖 DI 中的服务：使用 `AddAIAgentFactory`，把数据库、工具、配置、权限服务从 `IServiceProvider` 取出。
- 如果需要多 Agent 或编排：查看 `02_AgentOrchestration_Chaining`、`03_AgentOrchestration_Concurrency` 和 `DurableWorkflows` sample。
- 如果需要跨服务互调：查看 `dotnet/samples/04-hosting/A2A` 及 `Microsoft.Agents.AI.Hosting.A2A`。
- 如果需要本地观测验证：查看 `dotnet/samples/03-workflows/Observability/AspireDashboard`，它与下一章的可观测性主题衔接更紧。

## 关键代码或配置讲解

### 代码片段 1：Agent 注册到 Functions host

```csharp
.ConfigureDurableAgents(options => options.AddAIAgent(agent, timeToLive: TimeSpan.FromHours(1)))
```

- 作用：把一个已构造好的 `AIAgent` 注册成 Azure Functions 中可被调用的 Durable Agent。
- 为什么重要：这是 demo 对象变成服务端点的关键边界。
- 改动它会影响什么：改变 agent name、TTL、trigger 配置会影响 endpoint 是否可用、会话保留时间和调用方式。

### 代码片段 2：默认启用 HTTP trigger

`DurableAgentsOptionsExtensions.AddAIAgent` 中有如下逻辑：

```csharp
FunctionsAgentOptions agentOptions = new() { HttpTrigger = { IsEnabled = true } };
configure?.Invoke(agentOptions);
options.AddAIAgent(agent);
s_agentOptions[agent.Name] = agentOptions;
```

- 作用：为注册的 Agent 创建默认 Functions 触发选项。
- 为什么重要：解释了为什么 sample 中注册 Agent 后可以直接通过 HTTP endpoint 调用。
- 改动它会影响什么：如果禁用 HTTP trigger，外部 HTTP 请求将无法直接访问该 Agent；如果启用 MCP tool trigger，则 Agent 可能作为 MCP tool 暴露。

### 代码片段 3：Durable Agent 对 streaming 的边界

`DurableAIAgent.RunCoreStreamingAsync` 中写明：

```csharp
// Streaming is not supported for durable agents, so we just return the full response
// as a single update.
```

- 作用：定义 durable agent 的 streaming 行为边界。
- 为什么重要：生产 API 如果承诺 token-by-token streaming，就不能直接把 durable agent 的这个方法误解为真实流式模型输出。
- 改动它会影响什么：需要设计另一套流式通道、事件推送或前端轮询策略，而不是只改调用方。

### 代码片段 4：生产凭据提醒

`01_SingleAgent/Program.cs` 中 sample 注释提醒：

```csharp
// WARNING: DefaultAzureCredential is convenient for development but requires careful consideration in production.
```

- 作用：提醒开发环境与生产环境的凭据策略不同。
- 为什么重要：服务化部署时，凭据解析链、权限范围和冷启动延迟都会影响稳定性和安全性。
- 改动它会影响什么：生产建议改为明确的 Managed Identity 或受控密钥注入方式，并限制模型资源访问权限。

## 做一个最小改造实验

### 实验目标

把 `01_SingleAgent` 从“讲笑话 Agent”改造成综合案例中的“企业知识与工单协同 Agent 系统”的服务化入口雏形。

### 改造点

仅在本地实验中修改 sample，不要求提交到本手册仓库：

1. 将 agent name 从 `Joker` 改为 `SupportTriageAgent`。
2. 将 instructions 改为：

```text
You are an internal support triage agent. Classify the employee request, ask for missing information, and suggest whether a ticket should be created. Do not execute high-risk actions without approval.
```

3. 将请求改为：

```powershell
Invoke-RestMethod -Method Post `
  -Uri http://localhost:7071/api/agents/SupportTriageAgent/run `
  -ContentType text/plain `
  -Body "My VPN stopped working after password reset. Should I open an urgent ticket?"
```

4. 可选：把 `timeToLive` 从 1 小时改短，例如 10 分钟，观察会话保留策略对继续对话的影响。

### 你应该观察什么

- HTTP endpoint 会随 agent name 改变。
- Agent 行为会从讲笑话变成支持分诊。
- 如果继续传入同一个 `thread_id`，应该围绕同一问题延续上下文。
- TTL 过短时，长时间后继续对话可能无法按预期恢复上下文。

### 可能出现的问题

- 改了 name 但请求仍然打 `/api/agents/Joker/run`，会出现找不到 agent 或路由不可用。
- 只改 prompt 不改 endpoint，调用的仍是旧 agent name。
- 使用 `DefaultAzureCredential` 但本地没有登录 Azure，模型客户端初始化或请求会失败。
- Durable backend 没有准备好，Functions 启动后 Durable Agent 调用可能失败。

## 本章在综合案例中的增量

本书主案例是“企业内部知识与工单协同 Agent 系统”。到第 15 章为止，它已经具备业务拆解、知识检索、工具接入、审批节点和任务分工的设计。本章新增的是“生产级服务入口与长任务运行边界”。

新增后，主案例模块可以调整为：

```text
员工入口 / ITSM 系统 / Teams Bot
  -> Agent API Gateway 或 Azure Functions HTTP endpoint
  -> SupportTriageAgent durable endpoint
  -> Durable workflow：分类 -> 检索 -> 建议 -> 审批 -> 工单动作
  -> 工单系统 / 知识库 / 审批系统
  -> 状态存储、日志、审计、评测与治理
```

本章新增能力属于三类中的第 1 和第 2 类：

- 仓库已有 sample 直接支持：Azure Functions 托管单 Agent、Durable Agent HTTP 调用、session/thread_id 交互。
- 可以基于已有 sample 最小改造实现：把 `Joker` 改成 `SupportTriageAgent`，把 endpoint 作为企业支持入口雏形。
- 仓库未直接给出现成完整实现：生产 API 网关、企业鉴权、ITSM 真实工单系统、审批系统、云资源 IaC、蓝绿发布和灾备策略。

如果不加入本章能力，主案例会卡在“架构和本地原型成立，但不能作为稳定服务被企业系统调用”的阶段。

## 服务化部署与生产运行边界

### 1. 部署形态边界

仓库展示了多种 hosting 方向：

- Azure Functions + DurableTask：适合事件驱动、长任务、serverless 和可恢复编排。
- A2A hosting：适合 Agent 与 Agent 之间作为服务互调。
- AG-UI/AspNetCore hosting：适合前后端交互式 Agent UI 场景。
- Aspire Dashboard sample：更偏本地开发与观测验证，不等同于完整生产部署平台。

生产选型时不要只看“能不能跑”，还要看：请求延迟、并发模型、状态后端、扩缩容、冷启动、网络隔离、私有化要求、成本和组织运维能力。

### 2. 长任务与恢复边界

DurableTask 适合把多步骤任务拆成可恢复执行单元，但 Agent 任务仍要注意：

- 模型输出不是天然幂等：重试可能得到不同结果。
- Tool 调用有副作用：创建工单、发邮件、改权限必须设计幂等键和审批状态。
- 人工审批可能跨很长时间：TTL、状态保留、过期策略要和业务 SLA 对齐。
- 编排代码要遵守 Durable Task 的确定性要求；不要在 orchestrator 中直接做不可重放的外部调用。

### 3. 配置与凭据边界

开发环境可以用本地环境变量和 `DefaultAzureCredential`，生产环境应至少考虑：

- 用 Managed Identity 或专用凭据，不要把模型 key 写进代码或镜像。
- 每个环境独立配置 deployment name、endpoint、限流策略和日志级别。
- 对高成本模型调用设置配额、超时和降级策略。
- 对工具侧权限做最小授权，避免 Agent 通过工具获得过大系统权限。

### 4. API 边界

sample 直接暴露 `/api/agents/{agentName}/run`，生产系统通常还需要：

- API Gateway 或反向代理统一入口。
- 用户身份认证与租户隔离。
- 请求 schema 校验和响应版本控制。
- 幂等请求 ID、trace ID、业务 correlation ID。
- 对外部调用方明确同步/异步语义：立即返回结果、返回任务 ID，还是轮询状态。

### 5. 扩缩容与并发边界

Agent 服务扩缩容时要关注：

- Durable backend 是否成为瓶颈。
- 模型服务的 TPM/RPM 限额是否低于 Functions 并发能力。
- Tool 后端系统是否能承受并发。
- 会话是否与实例绑定；应尽量避免依赖进程内状态。
- 对同一 `thread_id` 或同一工单的并发请求是否需要串行化或乐观锁。

## 常见问题与排错

### 问题 1：Functions 启动时报 `AZURE_OPENAI_ENDPOINT is not set`

- 现象：应用启动失败，抛出 `InvalidOperationException`。
- 原因：`Program.cs` 在启动时强制读取 `AZURE_OPENAI_ENDPOINT`。
- 排查方法：检查当前终端环境变量是否设置，注意 PowerShell、CMD、IDE 启动配置之间不共享环境变量。
- 修复方式：设置 `AZURE_OPENAI_ENDPOINT` 和 `AZURE_OPENAI_DEPLOYMENT_NAME`，或使用 Functions 本地配置文件/云端应用配置注入。

### 问题 2：使用 `DefaultAzureCredential` 认证失败

- 现象：启动成功但模型调用失败，日志中出现 Azure credential 或权限错误。
- 原因：本地未 `az login`、登录账号无 Azure OpenAI 权限，或生产环境没有配置 Managed Identity。
- 排查方法：确认是否设置了 `AZURE_OPENAI_KEY`；若未设置，检查 Azure CLI 登录状态和资源权限。
- 修复方式：本地可设置 key 或登录 Azure；生产建议使用 Managed Identity 并授予最小必要权限。

### 问题 3：请求 `/api/agents/Joker/run` 返回 404 或找不到 Agent

- 现象：HTTP endpoint 不存在，或 Durable Agent 报未注册。
- 原因：agent name 改了但 URL 未同步；或者 `ConfigureDurableAgents` 没有注册对应 agent。
- 排查方法：检查 `const string JokerName` 或新 agent name；检查 Functions 启动日志中生成的函数端点。
- 修复方式：确保 URL 中的 `{agentName}` 与 `agent.Name` 完全一致。

### 问题 4：期望真实 streaming，但只收到一次完整响应

- 现象：调用 streaming API 时没有 token-by-token 更新。
- 原因：`DurableAIAgent.RunCoreStreamingAsync` 当前是把完整响应转换成单个 update。
- 排查方法：查看 `DurableAIAgent.cs` 中关于 streaming 的注释。
- 修复方式：生产中若需要真实流式体验，应单独设计非 durable 的 streaming 通道、事件推送或前端轮询任务状态。

### 问题 5：长任务重试后重复创建工单

- 现象：同一用户请求在 ITSM 中生成多个工单。
- 原因：DurableTask 可以帮助恢复和重试，但副作用工具没有幂等控制。
- 排查方法：检查工具调用是否传入业务幂等键，例如 `requestId`、`ticketDraftId`、`approvalId`。
- 修复方式：所有写操作工具都要设计幂等键和“已执行”状态；高风险动作必须经过审批状态机。

## 企业级落地提醒

- 稳定性：不要依赖进程内静态状态保存会话；长任务状态应进入 DurableTask 后端或业务数据库。
- 可维护性：把 Agent 构造、工具注册、模型客户端、配置读取拆开，避免全部堆在 `Program.cs`。
- 权限边界：Agent name 不是权限边界。即使 endpoint 中包含 agent name，也必须有用户身份、角色与资源权限校验。
- 安全风险：Prompt 不能替代权限控制；所有工具调用都要在服务端重新校验权限和参数。
- 成本控制：Functions 扩容、Durable 重试和模型调用叠加后可能放大成本，必须设置限流、超时、最大轮次和模型降级策略。
- 可观测性：本章只定位 hosting；上线前必须结合第 17 章补日志、trace、metrics、correlation ID 和失败重放策略。
- 可测试性：为 host 层补集成测试，至少覆盖 endpoint 可用、agent name 注册、认证失败、模型失败、工具失败和 durable 恢复。
- 持久化与恢复：TTL 不是随便填的参数，应结合业务 SLA、审计保留、隐私删除和成本做决策。

## 本章小结

- 本章最重要的结论 1：Hosting 解决的是 Agent 的服务化入口和生命周期管理；DurableTask 解决的是长任务、状态和恢复边界。
- 本章最重要的结论 2：仓库已经提供 `04-hosting`、`Microsoft.Agents.AI.Hosting.AzureFunctions`、`Microsoft.Agents.AI.DurableTask` 等真实 sample 与源码入口，但完整生产部署仍需要业务侧补齐鉴权、配置、限流、CI/CD 和治理。
- 本章最重要的结论 3：综合案例在本章从“可设计的复杂业务系统”升级为“可作为服务入口运行的 Agent 系统雏形”。下一章需要继续补上可观测性、调试与故障排查，否则服务虽然能运行，但出了问题难以定位。

## 本章建议产出物

- 一次 `01_SingleAgent` 的本地 Functions 运行记录。
- 一张从 HTTP endpoint 到 Durable Agent 再到模型调用的调用链图。
- 一个把 `Joker` 改为 `SupportTriageAgent` 的最小改造实验记录。
- 一份生产部署边界清单：鉴权、配置、状态、限流、重试、成本、观测、灾备。

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
