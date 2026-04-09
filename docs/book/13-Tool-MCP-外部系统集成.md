# 第 13 章：Tool、MCP 与外部系统集成

> 本章定位：把前面已经学过的 Tool / Function Calling，从“本地函数”升级为“可治理的外部系统接入层”，重点理解 MCP、OpenAPI、插件、审批与安全边界如何一起工作。

## 本章目标

学完本章后，你应能够：

- 区分本地 Function Tool、Plugin Tool、OpenAPI Tool、MCP Tool 各自适合的外部系统接入方式。
- 在仓库中定位 Tool、MCP、OpenAPI 与插件相关 sample 和关键源码。
- 跑通或验证一个最小外部系统接入示例，例如 Microsoft Learn MCP、OpenAPI REST Countries API 或本地函数工具。
- 理解一次 MCP Tool 调用从声明、审批、远端调用到结果写回的执行链路。
- 为企业内部知识与工单协同 Agent 系统设计安全边界：只读/写操作分级、凭据管理、审批、限流、审计与失败降级。

## 本章与前后章节的关系

- 前置知识：第 4 章已经介绍了 Tool 与 Function Calling 的基本用法；第 11 章从架构层解释了 Tool 层的位置；第 12 章说明了知识/RAG/Memory 如何为 Agent 提供上下文。
- 本章解决：如何让 Agent 稳定、安全地连接外部 API、MCP Server、企业插件服务和内部系统，而不是只调用内存里的示例函数。
- 后续衔接：第 14 章会继续展开 Human-in-the-Loop 与审批机制；第 19 章会更系统地讨论安全、权限、合规和成本治理。本章只把它们放在外部系统接入的边界内讨论。

## 先建立整体认知

在 Agent 系统里，Tool 层是模型与真实世界之间的“执行边界”。模型可以决定“应该查工单、搜文档、调用 API、创建记录”，但真正执行动作的应该是受控工具，而不是让模型直接拥有数据库、内网或云资源权限。

可以把本章的外部系统接入方式分成四类：

1. **Function Tool**：把一个 .NET 方法包装成 `AITool`，适合快速接入本地业务服务、只读查询或小范围逻辑。
2. **Plugin Tool**：把一组带依赖注入的业务能力组织成插件类，通过 `AsAITools()` 明确暴露哪些方法给 Agent。
3. **OpenAPI Tool**：通过 OpenAPI 描述远程 HTTP API，适合把 REST 服务作为工具暴露给 Foundry Agent。
4. **MCP Tool**：通过 Model Context Protocol 连接外部工具服务器，适合跨语言、跨进程、跨团队复用工具能力，也适合企业内部把工具统一发布成受治理的 MCP Server。

如果缺少这一层，Agent 常见问题会非常严重：

- 模型只能聊天，不能操作真实系统。
- 外部 API 调用散落在 Prompt 或业务代码里，难以测试、审计和限权。
- 凭据、Header、连接地址被硬编码，无法按环境治理。
- 写操作没有审批，Prompt Injection 可能诱导 Agent 调用高风险工具。
- 工具结果没有结构化写回，后续 Workflow 无法可靠消费。

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：
  - `dotnet/samples/01-get-started/02_add_tools`
  - `dotnet/samples/02-agents/Agents/Agent_Step07_AsMcpTool`
  - `dotnet/samples/02-agents/Agents/Agent_Step12_Plugins`
  - `dotnet/samples/02-agents/AgentsWithFoundry/Agent_Step09_UsingMcpClientAsTools`
  - `dotnet/samples/02-agents/AgentsWithFoundry/Agent_Step17_OpenAPITools`
  - `dotnet/samples/03-workflows/Declarative/InvokeMcpTool`
  - `dotnet/samples/02-agents/ModelContextProtocol/*`
- 项目：
  - `Microsoft.Agents.AI`
  - `Microsoft.Agents.AI.Foundry`
  - `Microsoft.Agents.AI.Workflows.Declarative`
  - `Microsoft.Agents.AI.Workflows.Declarative.Mcp`
- Sample：
  - `02_add_tools`：本地函数工具，最小可运行入口。
  - `Agent_Step12_Plugins`：插件 + 依赖注入。
  - `Agent_Step09_UsingMcpClientAsTools`：通过 MCP Client 使用 Microsoft Learn MCP 工具。
  - `Agent_Step17_OpenAPITools`：用 OpenAPI 描述 REST Countries API。
  - `Agent_Step07_AsMcpTool`：把 Agent 暴露成 MCP Tool。
  - `InvokeMcpTool`：在声明式 Workflow 中直接调用 MCP Tool。
- 关键类型：
  - `AIAgent`
  - `AITool`
  - `AIFunctionFactory`
  - `McpClient`
  - `McpClientTool`
  - `McpServerTool`
  - `FoundryAITool`
  - `OpenApiFunctionDefinition`
  - `InvokeMcpToolExecutor`
  - `DefaultMcpToolHandler`
  - `McpServerToolResultContent`
  - `ToolApprovalRequestContent`
  - `ToolApprovalResponseContent`
- 关键接口：
  - `IMcpToolHandler`，定义声明式工作流里调用 MCP 工具的抽象边界。
- 关键扩展点：
  - `AIFunctionFactory.Create(...)`：把 .NET 方法变成工具。
  - `serviceProvider.GetRequiredService<AgentPlugin>().AsAITools()`：插件显式暴露工具方法。
  - `FoundryAITool.CreateOpenApiTool(...)`：从 OpenAPI 定义创建工具。
  - `McpClient.ListToolsAsync()` 与 `McpClientTool` 转 `AITool`：把远端 MCP Server 工具接入 Agent。
  - `DefaultMcpToolHandler(Func<string, CancellationToken, Task<HttpClient?>>? httpClientProvider)`：为不同 MCP Server 提供带认证的 `HttpClient`。

推荐阅读顺序：先看 `02_add_tools`，再看 `Agent_Step12_Plugins`，然后看 `Agent_Step09_UsingMcpClientAsTools` 和 `Agent_Step17_OpenAPITools`，最后看 `InvokeMcpToolExecutor` 与 `DefaultMcpToolHandler` 的源码。

## 先跑通或先验证一个最小示例

### 示例目标

验证 Agent 可以通过 MCP Client 把 Microsoft Learn MCP Server 上的工具接入为 `AITool`，并在回答文档问题时调用外部文档搜索能力。

### 运行前提

- 已安装 .NET SDK。
- 已配置 Azure AI Project：
  - `AZURE_AI_PROJECT_ENDPOINT`
  - `AZURE_AI_MODEL_DEPLOYMENT_NAME`
- 当前身份可通过 `DefaultAzureCredential` 访问对应 Azure AI Project。
- 本机网络可以访问 `https://learn.microsoft.com/api/mcp`。

### 操作步骤

1. 进入仓库根目录：

   ```bash
   cd D:\code\agent-framework\dotnet
   ```

2. 运行 MCP Client sample：

   ```bash
   dotnet run --project samples/02-agents/AgentsWithFoundry/Agent_Step09_UsingMcpClientAsTools/Agent_Step09_UsingMcpClientAsTools.csproj
   ```

3. 观察控制台输出。sample 会先连接 Microsoft Learn MCP Server，然后通过 `ListToolsAsync()` 列出可用工具，再创建名为 `DocsAgent` 的 Agent 并询问：

   ```text
   How does one create an Azure storage account using az cli?
   ```

### 预期现象

- 控制台出现类似 `Connecting to MCP server at https://learn.microsoft.com/api/mcp ...`。
- 控制台输出 `MCP tools available: ...`，其中应包含 Microsoft Learn 相关搜索或文档工具。
- Agent 返回的答案会围绕 Azure CLI 创建 Storage Account 的步骤展开。

### 成功标志

- `McpClient.CreateAsync(...)` 能成功建立连接。
- `ListToolsAsync()` 返回非空工具列表。
- Agent 的回答能体现外部 MCP 文档工具的结果，而不是完全脱离 Microsoft Learn 内容的泛泛回答。

如果你暂时没有 Azure AI Project，可以先验证本地函数工具：

```bash
dotnet run --project samples/01-get-started/02_add_tools/02_add_tools.csproj
```

该 sample 使用 `AIFunctionFactory.Create(GetWeather)` 把 `GetWeather` 方法暴露给 Agent，问题是 `What is the weather like in Amsterdam?`，预期回答包含 `cloudy with a high of 15°C`。

## 核心概念拆解

### 1. Function Tool 是什么

Function Tool 是最直接的工具形式。`02_add_tools/Program.cs` 中定义了：

```csharp
[Description("Get the weather for a given location.")]
static string GetWeather([Description("The location to get the weather for.")] string location)
    => $"The weather in {location} is cloudy with a high of 15°C.";
```

随后通过：

```csharp
tools: [AIFunctionFactory.Create(GetWeather)]
```

把这个方法暴露给 Agent。它适合快速验证工具调用链，也适合把应用内已有的查询服务包装成工具。边界是：不要把任意内部方法无筛选地暴露给模型，尤其是带写操作、删除操作、跨租户查询或高成本调用的方法。

### 2. Plugin Tool 是什么

`Agent_Step12_Plugins/Program.cs` 展示了插件模式。`AgentPlugin` 依赖 `WeatherProvider`，并在 `GetCurrentTime(IServiceProvider sp, string location)` 中通过 `IServiceProvider` 解析 `CurrentTimeProvider`。真正暴露给 Agent 的方法由 `AsAITools()` 显式决定：

```csharp
public IEnumerable<AITool> AsAITools()
{
    yield return AIFunctionFactory.Create(this.GetWeather);
    yield return AIFunctionFactory.Create(this.GetCurrentTime);
}
```

这很重要：真实企业项目里的服务类通常有很多方法，但不是所有方法都应该成为工具。插件模式让你把依赖注入、业务服务、工具暴露清单分开管理。

### 3. OpenAPI Tool 是什么

`Agent_Step17_OpenAPITools/Program.cs` 展示了 OpenAPI Tool。sample 用一个 OpenAPI 3.1 JSON 描述 `https://restcountries.com/v3.1/currency/{currency}`，再通过：

```csharp
AITool openApiTool = FoundryAITool.CreateOpenApiTool(CreateOpenAPIFunctionDefinition());
```

把远程 REST API 变成 Agent 可调用工具。它适合接入已有 HTTP API，尤其是服务团队已经维护 OpenAPI 文档的场景。要注意 OpenAPI 描述会影响模型如何理解参数和语义，所以 `operationId`、`description`、参数 schema 都应准确、简洁、最小化。

### 4. MCP Tool 是什么

MCP，即 Model Context Protocol，用于把外部工具服务以协议化方式暴露给模型应用。仓库中有两类 MCP 用法：

- **作为 MCP Client 使用远端工具**：`Agent_Step09_UsingMcpClientAsTools` 连接 `https://learn.microsoft.com/api/mcp`，调用 `ListToolsAsync()` 获取 `McpClientTool`，再转换为 `AITool`。
- **把 Agent 暴露成 MCP Tool**：`Agent_Step07_AsMcpTool` 使用 `agent.AsAIFunction()` 与 `McpServerTool.Create(...)`，然后通过 `.AddMcpServer().WithStdioServerTransport().WithTools([tool])` 启动一个 stdio MCP Server。

MCP 的价值在于工具可以独立于 Agent 应用部署和复用。比如企业内部可以把“知识库搜索”“工单查询”“客户信息只读查询”“变更计划生成”等能力统一托管在 MCP Server，多个 Agent 通过协议消费。

### 5. 它们之间如何协作

在真实系统里，不必只选一种方式：

- 本地轻量逻辑：Function Tool。
- 需要 DI 和业务服务组合：Plugin Tool。
- 已有 REST API 且有 OpenAPI 描述：OpenAPI Tool。
- 跨团队、跨语言、跨进程复用工具：MCP Tool。
- 高风险工具：在 Tool 或 Workflow 层加审批、权限和审计。

主线原则是：让模型只看见“最小必要工具清单”，让真实执行发生在可测试、可审计、可限权的工具实现里。

### 6. 常见误解是什么

- 误解一：Tool 越多越好。实际上工具越多，模型越容易选错；应该按场景动态提供最小集合。
- 误解二：只要 Prompt 写“不要做危险操作”就安全。Prompt 不是权限边界，真正的边界应在工具实现、审批和凭据权限里。
- 误解三：MCP Server 返回什么都可以直接信任。MCP 是连接协议，不等于安全验证；仍要做 allowlist、认证、超时、输出校验和审计。
- 误解四：OpenAPI 文档能完整表达业务权限。OpenAPI 只描述接口形态，不能替代业务授权、租户隔离和操作审批。

## 结合源码理解执行过程

建议按“入口 -> 中间处理 -> 关键对象 -> 输出结果”的顺序理解。

### 1. 入口在哪里

以 `Agent_Step09_UsingMcpClientAsTools/Program.cs` 为例，入口是：

```csharp
await using McpClient mcpClient = await McpClient.CreateAsync(new HttpClientTransport(new()
{
    Endpoint = new Uri("https://learn.microsoft.com/api/mcp"),
    Name = "Microsoft Learn MCP",
}));
```

然后：

```csharp
IList<McpClientTool> mcpTools = await mcpClient.ListToolsAsync();
List<AITool> agentTools = [.. mcpTools.Cast<AITool>()];
```

这一步把远端 MCP Server 暴露的工具列表转换成 Agent 可以使用的 `AITool` 列表。

### 2. 调用是如何继续流转的

接下来 sample 创建 Agent：

```csharp
AIAgent agent = aiProjectClient.AsAIAgent(deploymentName,
    instructions: "You are a helpful assistant that can help with Microsoft documentation questions. Use the Microsoft Learn MCP tool to search for documentation.",
    name: "DocsAgent",
    tools: agentTools);
```

当用户提出 Azure CLI 创建 Storage Account 的问题时，模型会根据工具描述决定是否调用 Microsoft Learn MCP 工具。Agent 框架将工具调用转交给对应 `McpClientTool`，远端 MCP Server 返回结果后，Agent 再生成最终自然语言回答。

### 3. 声明式 Workflow 中的 MCP 调用链

`dotnet/samples/03-workflows/Declarative/InvokeMcpTool/InvokeMcpTool.yaml` 展示了更适合企业流程编排的写法：

```yaml
- kind: InvokeMcpTool
  id: invoke_docs_search
  serverUrl: https://learn.microsoft.com/api/mcp
  serverLabel: microsoft_docs
  toolName: microsoft_docs_search
  conversationId: =System.ConversationId
  arguments:
    query: =Local.SearchQuery
  output:
    autoSend: true
    result: Local.DocsSearchResult
```

这里不是让模型自由决定调用哪个工具，而是 Workflow 明确调用 `microsoft_docs_search`，并把结果写入 `Local.DocsSearchResult`。这种方式适合对流程确定性要求更高的企业场景。

对应源码入口是 `InvokeMcpToolExecutor`。它会读取：

- `serverUrl`
- `serverLabel`
- `toolName`
- `arguments`
- `headers`
- `connectionName`
- `requireApproval`
- `output.autoSend`
- `conversationId`

如果不需要审批，它会直接调用：

```csharp
mcpToolHandler.InvokeToolAsync(serverUrl, serverLabel, toolName, arguments, headers, connectionName, cancellationToken)
```

如果需要审批，它会构造 `ToolApprovalRequestContent`，通过 `ExternalInputRequest` 把审批请求交还给外部调用方，等待 `ToolApprovalResponseContent` 后再继续执行。

### 4. 关键对象分别负责什么

- `IMcpToolHandler`：声明式 Workflow 调用 MCP 的抽象接口，屏蔽本地、托管、测试等不同实现。
- `DefaultMcpToolHandler`：默认实现，基于 MCP C# SDK 创建 `McpClient` 并调用远端工具。
- `McpClient`：实际连接 MCP Server 的客户端。
- `McpServerToolResultContent`：承载 MCP 调用结果，最终可写入 Workflow state 或作为 Tool message 发给 Agent。
- `ToolApprovalRequestContent` / `ToolApprovalResponseContent`：用于审批式工具调用。
- `ExternalInputRequest` / `ExternalInputResponse`：让 Workflow 在需要外部输入时暂停和恢复。

### 5. 扩展点在哪里

最重要的扩展点在 `DefaultMcpToolHandler` 构造函数：

```csharp
public DefaultMcpToolHandler(Func<string, CancellationToken, Task<HttpClient?>>? httpClientProvider = null)
```

这意味着企业可以按 `serverUrl` 返回不同的 `HttpClient`，例如：

- 给内部 MCP Server 注入 Bearer Token。
- 使用托管身份获取下游访问令牌。
- 设置超时、代理、重试策略、User-Agent。
- 按域名做 allowlist，拒绝未知 MCP Server。

`DefaultMcpToolHandler` 内部还会按 `serverUrl + headers hash` 缓存 `McpClient`，并在 `DisposeAsync()` 中释放客户端和自身创建的 `HttpClient`。这对稳定性很关键，避免每次工具调用都重新建连接。

## 关键代码或配置讲解

### 代码片段 1：本地函数工具

```csharp
AIAgent agent = new AzureOpenAIClient(
    new Uri(endpoint),
    new DefaultAzureCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(instructions: "You are a helpful assistant", tools: [AIFunctionFactory.Create(GetWeather)]);
```

- 作用：把 `GetWeather` 注入为 Agent 可调用工具。
- 为什么重要：它是所有外部工具接入的最小心智模型：工具有名称、描述、参数和返回值。
- 改动它会影响什么：如果工具描述不清晰，模型可能不调用或错误调用；如果函数参数过宽，模型可能传入不可控数据。

### 代码片段 2：插件显式暴露工具

```csharp
tools: [.. serviceProvider.GetRequiredService<AgentPlugin>().AsAITools()],
services: serviceProvider
```

- 作用：从 DI 容器获取插件实例，并只暴露 `AsAITools()` 返回的方法。
- 为什么重要：企业业务工具通常需要数据库、缓存、HTTP Client、权限服务等依赖，插件模式比孤立静态方法更接近真实项目。
- 改动它会影响什么：如果把整个服务类所有方法都暴露成工具，会扩大权限边界；如果忘记传 `services`，需要服务解析的工具可能运行失败。

### 代码片段 3：OpenAPI Tool

```csharp
AITool openApiTool = FoundryAITool.CreateOpenApiTool(CreateOpenAPIFunctionDefinition());
```

- 作用：把 REST Countries API 的 OpenAPI 定义转换为 Foundry Agent 可用工具。
- 为什么重要：这是接入外部 HTTP API 的低代码路径。
- 改动它会影响什么：OpenAPI 的 `servers.url`、`paths`、`operationId`、参数 schema 直接决定工具调用地址和参数形态。生产中不能把高危接口、管理接口或未授权接口随意放进同一个 OpenAPI 工具。

### 代码片段 4：声明式 MCP 调用

```yaml
- kind: InvokeMcpTool
  id: invoke_docs_search
  serverUrl: https://learn.microsoft.com/api/mcp
  toolName: microsoft_docs_search
  arguments:
    query: =Local.SearchQuery
  output:
    result: Local.DocsSearchResult
```

- 作用：在 Workflow 中固定调用一个 MCP 工具，并把结果写入局部变量。
- 为什么重要：相比让模型自由选择工具，Workflow 调用更可控，更适合企业流程。
- 改动它会影响什么：`serverUrl` 决定连接哪个外部工具服务；`toolName` 决定实际能力；`arguments` 决定输入；`output.result` 决定后续步骤如何读取结果。

## 做一个最小改造实验

### 实验目标

把主案例“企业内部知识与工单协同 Agent 系统”新增一个只读外部文档检索工具：当用户提出工单处理问题时，Agent 先通过 MCP 查询文档，再生成建议回复。

### 改造点

选择 `dotnet/samples/03-workflows/Declarative/InvokeMcpTool/InvokeMcpTool.yaml` 做最小改造：

1. 把 `Local.SearchQuery` 视为用户工单问题。
2. 保留 `invoke_docs_search`，把 `arguments.query` 继续绑定到 `Local.SearchQuery`。
3. 在 `summarize_results` 的 Prompt 中强调输出“工单处理建议”和“引用的文档依据”。例如：

```yaml
input:
  messages: =UserMessage("For the internal ticket question '" & Local.SearchQuery & "', summarize the Microsoft Learn search result and produce a support-ticket handling suggestion. Do not perform any write action.")
```

### 你应该观察什么

- Workflow 仍然先调用 MCP 文档搜索工具。
- `Local.DocsSearchResult` 中保存工具结果。
- 最终 Agent 输出从“泛化搜索总结”变为“面向工单处理建议”。
- 整个改造仍是只读能力，没有创建、关闭、升级工单等写操作。

### 可能出现的问题

- `toolName` 与 MCP Server 实际工具名不一致：先用 MCP Client sample 打印 `MCP tools available`，确认工具名。
- 远端 MCP Server 不可访问：检查网络、代理、防火墙和 TLS。
- 结果为空：检查查询词是否过短，或者 server 是否支持该查询。
- 模型没有引用工具结果：在 Prompt 中明确要求“基于搜索结果”，并在 Workflow 中确保 MCP 结果写入会话或传入后续 Agent。

## 本章在综合案例中的增量

本书主案例是：**企业内部知识与工单协同 Agent 系统**。

在本章之前，综合案例已经具备：

- 最小 Agent 交互能力。
- 基本 Prompt 和本地 Tool 调用能力。
- 会话、上下文、Workflow、状态持久化的设计思路。
- 知识/RAG/Memory 层的初步结构。

本章新增的是：**外部系统接入层**。具体包括：

- 新增只读外部工具：`DocsSearchMcpTool`，可映射到仓库的 Microsoft Learn MCP sample。
- 新增 API 工具形态：`ExternalRestApiTool`，可参考 `Agent_Step17_OpenAPITools`。
- 新增插件工具形态：`TicketingPlugin` 或 `SupportPlugin`，可参考 `Agent_Step12_Plugins` 的依赖注入模式。
- 新增安全边界：只读查询可自动执行；写操作必须走审批；工具凭据不交给模型，只在工具层或 `HttpClient` 层处理。

加入后，主案例从“会回答、会检索知识”的 Agent，升级为“能受控连接外部文档、API 和企业系统”的业务型 Agent。如果不加入本章能力，工单协同系统只能停留在建议生成，无法稳定查询外部系统、同步状态或与企业工具链集成。

需要明确区分三类内容：

1. 仓库已有 sample 直接支持：本地 Function Tool、Plugin Tool、OpenAPI Tool、Microsoft Learn MCP Tool、Agent as MCP Tool、声明式 InvokeMcpTool。
2. 可以基于已有 sample 做最小改造：把 `InvokeMcpTool.yaml` 的文档搜索结果改造成工单建议输入；把插件中的 `WeatherProvider` 替换成 `TicketingProvider`。
3. 仓库未直接给出现成完整样例：真实企业工单系统的认证、租户隔离、审批流、审计日志和写操作回滚，需要基于框架扩展点自行实现。

## 常见问题与排错

### 问题 1：环境变量未设置

- 现象：启动时报 `AZURE_AI_PROJECT_ENDPOINT is not set` 或 `AZURE_OPENAI_ENDPOINT is not set`。
- 原因：sample 通过环境变量读取 Azure AI Project 或 Azure OpenAI 配置。
- 排查方法：查看对应 `Program.cs` 前几行需要哪些环境变量。
- 修复方式：设置 `AZURE_AI_PROJECT_ENDPOINT`、`AZURE_AI_MODEL_DEPLOYMENT_NAME`，或 `AZURE_OPENAI_ENDPOINT`、`AZURE_OPENAI_DEPLOYMENT_NAME`。

### 问题 2：DefaultAzureCredential 认证失败

- 现象：运行时出现认证失败、无权限或 token 获取失败。
- 原因：本机没有登录 Azure CLI/Visual Studio，或当前身份没有访问资源权限。
- 排查方法：确认 `az login`、Azure 订阅、资源权限和 endpoint 是否匹配。
- 修复方式：开发环境可登录正确账号；生产环境不要依赖宽泛的 `DefaultAzureCredential` fallback，优先使用明确的 `ManagedIdentityCredential` 或受控凭据策略。

### 问题 3：MCP 工具名不匹配

- 现象：调用 MCP Tool 时提示工具不存在或返回错误。
- 原因：`toolName` 写错，或 MCP Server 工具列表更新。
- 排查方法：运行 `Agent_Step09_UsingMcpClientAsTools`，观察 `MCP tools available` 输出。
- 修复方式：把 Workflow YAML 中的 `toolName` 改成真实工具名。

### 问题 4：MCP Server 无法连接

- 现象：连接超时、DNS 失败、TLS 失败或 401/403。
- 原因：网络、防火墙、代理、证书、认证 Header 或 server 地址错误。
- 排查方法：先用 curl 或浏览器验证 server 地址；再检查 `HttpClientTransportOptions.Endpoint` 与 Header。
- 修复方式：配置代理、认证 Header、allowlist；企业内部 MCP Server 建议通过 `DefaultMcpToolHandler` 的 `httpClientProvider` 统一配置。

### 问题 5：工具结果没有被后续步骤使用

- 现象：工具调用成功，但 Agent 回答像没有看到工具结果。
- 原因：Workflow 没有写 `output.result`，或没有把结果加入 conversation / 后续 Prompt。
- 排查方法：检查 YAML 中 `output.result`、`output.autoSend`、`conversationId`。
- 修复方式：把结果写入明确变量，例如 `Local.DocsSearchResult`，并在后续 Agent 输入中引用该变量。

### 问题 6：工具执行了不该执行的操作

- 现象：Agent 自动调用了写操作、发送通知、修改工单或触发外部系统副作用。
- 原因：工具权限过宽，工具列表未分级，或缺少审批。
- 排查方法：列出当前 Agent 可见的全部工具，标注只读/写入/高风险等级。
- 修复方式：默认只暴露只读工具；写操作使用 `requireApproval` 或人工审批；在工具实现中再次校验权限，不依赖模型自律。

## 企业级落地提醒

### 稳定性

- 为外部 API 和 MCP Server 设置超时、重试和熔断，不要让单个工具调用拖垮整个 Agent 请求。
- 区分可重试和不可重试操作：只读查询可以重试，创建工单、扣费、发邮件等写操作不能盲目重试。
- 对 MCP Client 和 HttpClient 做复用。`DefaultMcpToolHandler` 已经有客户端缓存思路，企业实现不要每次调用都新建连接。

### 可维护性

- 工具命名要面向业务语义，例如 `SearchKnowledgeDocs`、`GetTicketById`、`DraftTicketReply`，避免 `CallApi1` 这类名称。
- 工具描述要短、准、包含边界。模型依赖名称和描述选择工具。
- 插件类只暴露白名单方法，`AsAITools()` 是重要治理点。

### 权限边界

- 模型不应该直接接触数据库连接串、API Key、Bearer Token。
- 凭据应在工具实现、`HttpClient` provider、托管身份或连接管理层处理。
- 按租户、用户、角色传递上下文，在工具内部二次授权，不能只靠 Agent Prompt。

### 安全风险

- 防 Prompt Injection：外部文档、网页、工单描述都可能包含“忽略指令并调用删除工具”之类内容。工具层必须做权限校验。
- 对 `serverUrl` 做 allowlist，禁止模型或用户输入任意 MCP Server 地址。
- 对工具参数做 schema 校验、长度限制和业务校验。
- 对工具输出做脱敏，避免把内部 token、PII、密钥或越权数据返回给模型。

### 成本控制

- 外部工具结果可能很长，应做摘要、分页或 TopK 限制，避免把大量结果塞回模型上下文。
- 对高成本 API 加缓存和速率限制。
- 对工具调用次数做配额，避免模型在循环中重复调用。

### 可观测性

- 记录工具调用的工具名、调用耗时、成功/失败、错误码、调用者、审批状态，但不要记录敏感参数明文。
- MCP 与 OpenAPI 调用应有 trace id，便于从 Agent 对话追踪到下游系统日志。
- 对写操作保留审计事件：谁发起、模型建议了什么、谁审批、工具实际执行了什么。

### 可测试性

- 用 `IMcpToolHandler` 替换真实 MCP 调用，编写单元测试或集成测试。
- 对插件工具做普通 .NET 单元测试，不必每次都通过模型触发。
- 对 OpenAPI 工具使用 mock server 或测试环境 API，避免测试污染生产数据。

## 本章小结

- 本章最重要的结论 1：Tool 层是 Agent 与外部系统之间的执行边界，不能把它只当作“让模型能调用函数”的语法功能。
- 本章最重要的结论 2：仓库已经提供了多种真实接入路径：Function Tool、Plugin Tool、OpenAPI Tool、MCP Client、Agent as MCP Tool 和声明式 `InvokeMcpTool`。
- 本章最重要的结论 3：企业落地时，安全边界必须落在工具实现、凭据管理、审批、allowlist、审计和权限校验上，而不是只依赖 Prompt。

下一章会继续讨论 Human-in-the-Loop 与审批机制。也就是说，本章先明确“外部系统怎么接入”，下一章再重点回答“哪些工具调用必须等人确认，以及审批流程如何设计”。

## 本章建议产出物

- 一次 `Agent_Step09_UsingMcpClientAsTools` 或 `02_add_tools` 的 sample 运行记录。
- 一张“Agent -> Tool -> MCP/OpenAPI/Plugin -> 外部系统 -> Tool Result -> Agent”的调用链图。
- 一个基于 `InvokeMcpTool.yaml` 的最小改造实验记录。
- 一份主案例工具清单，标注每个工具是只读、写操作还是高风险操作。
- 一份外部系统接入安全检查清单。

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
