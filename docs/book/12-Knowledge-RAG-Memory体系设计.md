# 第 12 章：Knowledge、RAG 与 Memory 体系设计

> 本章定位：把第 11 章的 Agent 应用分层继续向“知识层”和“记忆层”推进，说明如何让 Agent 同时利用企业知识库、检索增强生成（RAG）和跨轮次 Memory。

## 本章目标

学完本章后，你应能够：

- 区分 Knowledge、RAG、短期上下文、长期 Memory 和 Chat History 的职责边界。
- 在仓库中定位 RAG 与 Memory 相关 sample、源码类型和扩展点。
- 跑通或验证一个基于 `TextSearchProvider` 的最小 RAG 示例。
- 理解 `ChatHistoryMemoryProvider` 如何把历史消息写入向量存储并在后续调用中检索回来。
- 设计“企业内部知识与工单协同 Agent 系统”的知识检索和记忆增强增量。
- 明确哪些能力是仓库已经给出的，哪些是基于框架抽象可以继续扩展的企业级设计。

## 本章与前后章节的关系

- 前置知识：第 7 章已经讲过 Session、History、Context 与 Memory 的基础；第 10 章讨论过状态持久化；第 11 章建立了 Router、Planner、Specialist、Tool 层和状态层的应用架构视角。
- 本章解决：Agent 如何把“用户当前问题 + 企业知识 + 历史记忆”组合成可控上下文，而不是只依赖模型参数或一次性 Prompt。
- 后续衔接：第 13 章会继续讲 Tool、MCP 与外部系统集成；本章只把外部知识源视为检索数据源，不展开 API 权限、审批和副作用治理。

## 先建立整体认知

在 Agent 系统里，Knowledge、RAG 与 Memory 经常被混在一起，但它们不是同一个东西：

- **Knowledge**：相对稳定的业务知识，例如产品政策、操作手册、工单处理规范、FAQ、知识库文档。
- **RAG**：把用户问题转换为检索查询，从外部知识源取回相关片段，再把片段注入模型上下文，让模型基于可引用资料回答。
- **Memory**：围绕用户、会话、任务或 Agent 形成的历史事实与偏好，例如“这个用户偏好中文回答”“上一轮提到的工单编号”“用户喜欢海盗笑话”。
- **Chat History**：当前会话的消息记录，通常用于短期上下文；当历史太长时，需要裁剪、摘要或外溢到向量存储。

缺少这一层时，Agent 会出现三类问题：

1. 对企业私有知识不了解，只能给通用回答。
2. 多轮或跨会话时记不住用户偏好与历史事实。
3. 上下文无限膨胀，成本升高且回答容易被无关历史干扰。

本章的关键结论是：**RAG 解决“查企业知识”，Memory 解决“记用户和任务上下文”，二者都可以通过 `AIContextProvider` 类扩展点在模型调用前后参与上下文构造。**

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：
  - `dotnet/samples/02-agents/AgentWithRAG`
  - `dotnet/samples/02-agents/AgentWithMemory`
  - `dotnet/samples/01-get-started/04_memory`
  - `dotnet/src/Microsoft.Agents.AI/Memory`
  - `dotnet/src/Microsoft.Agents.AI/AIContextProviderDecorators`
  - `dotnet/src/Microsoft.Agents.AI.Foundry/Memory`
- 项目：
  - `AgentWithRAG_Step01_BasicTextRAG`
  - `AgentWithRAG_Step02_CustomVectorStoreRAG`
  - `AgentWithRAG_Step03_CustomRAGDataSource`
  - `AgentWithRAG_Step04_FoundryServiceRAG`
  - `AgentWithRAG_Step05_Neo4jGraphRAG`
  - `AgentWithMemory_Step01_ChatHistoryMemory`
  - `AgentWithMemory_Step02_MemoryUsingMem0`
  - `AgentWithMemory_Step04_MemoryUsingFoundry`
  - `AgentWithMemory_Step05_BoundedChatHistory`
  - `Agent_Step22_MemorySearch`
- Sample：
  - 最小自定义 Memory：`dotnet/samples/01-get-started/04_memory/Program.cs`
  - 基础 RAG：`dotnet/samples/02-agents/AgentWithRAG/AgentWithRAG_Step01_BasicTextRAG/Program.cs`
  - 自定义检索函数 RAG：`dotnet/samples/02-agents/AgentWithRAG/AgentWithRAG_Step03_CustomRAGDataSource/Program.cs`
  - 向量化历史记忆：`dotnet/samples/02-agents/AgentWithMemory/AgentWithMemory_Step01_ChatHistoryMemory/Program.cs`
  - 有界历史 + 溢出到向量库：`dotnet/samples/02-agents/AgentWithMemory/AgentWithMemory_Step05_BoundedChatHistory/Program.cs`
- 关键类型：
  - `TextSearchProvider`
  - `TextSearchProvider.TextSearchResult`
  - sample 内的 `TextSearchStore` 与 `TextSearchDocument`
  - `ChatHistoryMemoryProvider`
  - `ChatHistoryMemoryProviderOptions`
  - `ChatHistoryMemoryProviderScope`
  - `InMemoryChatHistoryProvider`
  - `BoundedChatHistoryProvider`（sample 内实现）
  - `ProviderSessionState<T>`
- 关键接口和抽象：
  - `AIContextProvider`
  - `MessageAIContextProvider`
  - `VectorStore`
  - `VectorStoreCollection<TKey,TRecord>`
  - `IKeywordHybridSearchable<TRecord>`
  - `IChatClient`
  - `IEmbeddingGenerator`
- 关键扩展点：
  - `ChatClientAgentOptions.AIContextProviders`
  - `ChatClientAgentOptions.ChatHistoryProvider`
  - `TextSearchProviderOptions.SearchTime`
  - `TextSearchProviderOptions.RecentMessageMemoryLimit`
  - `ChatHistoryMemoryProviderOptions.SearchTime`
  - `ChatHistoryMemoryProviderOptions.MaxResults`
  - `StorageInputRequestMessageFilter` / `StorageInputResponseMessageFilter`
  - `ChatHistoryMemoryProvider.State` 中的 `storageScope` 与 `searchScope`

### 仓库已有与理论延伸边界

- 仓库已有：基于 `TextSearchProvider` 的 RAG 上下文注入；基于 `VectorStore` 的文档存取示例；自定义检索函数示例；基于 `ChatHistoryMemoryProvider` 的历史消息向量记忆；Foundry Memory、Mem0、GraphRAG 等 sample 入口。
- sample 已展示但不一定生产可直接使用：`TextSearchStore` 在注释中明确是 sample store implementation，提供固定 schema，并说明未来可能成为一等 API；`InMemoryVectorStore` 适合演示，不适合作为长期持久化存储。
- 理论延伸：企业级文档清洗、分块策略、权限过滤、增量索引、重排序、答案引用校验、Prompt Injection 防护、多租户隔离和离线评测，本章会给设计建议，但仓库 sample 没有完整端到端实现。

推荐阅读顺序：先看 `AgentWithRAG_Step03_CustomRAGDataSource` 理解最小 RAG 协议，再看 `AgentWithRAG_Step01_BasicTextRAG` 理解向量存储路径；Memory 先看 `01-get-started/04_memory`，再看 `AgentWithMemory_Step01_ChatHistoryMemory` 和 `AgentWithMemory_Step05_BoundedChatHistory`。

## 先跑通或先验证一个最小示例

### 示例目标

验证 `TextSearchProvider` 如何在模型调用前执行检索，并把搜索结果作为上下文注入给 Agent。

### 运行前提

- 已进入仓库根目录 `D:\code\agent-framework`。
- 已具备 Azure OpenAI 访问能力。
- 设置环境变量：
  - `AZURE_OPENAI_ENDPOINT`
  - `AZURE_OPENAI_DEPLOYMENT_NAME`，未设置时 sample 默认使用 `gpt-5.4-mini`
  - `AZURE_OPENAI_EMBEDDING_DEPLOYMENT_NAME`，基础向量 RAG 示例默认使用 `text-embedding-3-large`
- 能通过 `DefaultAzureCredential` 登录对应 Azure 资源。sample 注释也提醒：生产环境不应无脑使用默认凭据链，建议改为明确的托管身份或指定凭据。

### 操作步骤

1. 先运行不依赖真实向量库的自定义检索函数示例：

   ```powershell
   cd D:\code\agent-framework\dotnet\samples\02-agents\AgentWithRAG\AgentWithRAG_Step03_CustomRAGDataSource
   dotnet run
   ```

2. 再运行基于 `InMemoryVectorStore` 与 embedding 的基础 RAG 示例：

   ```powershell
   cd D:\code\agent-framework\dotnet\samples\02-agents\AgentWithRAG\AgentWithRAG_Step01_BasicTextRAG
   dotnet run
   ```

3. 如果要验证 Memory，可运行：

   ```powershell
   cd D:\code\agent-framework\dotnet\samples\02-agents\AgentWithMemory\AgentWithMemory_Step01_ChatHistoryMemory
   dotnet run
   ```

### 预期现象

- RAG 示例会依次询问退货、配送、帐篷保养等问题。
- Agent 应能依据 Contoso Outdoors 示例文档回答，并在可用时引用 source document。
- Memory 示例中，第二个新 session 会基于共享的搜索 scope 记住用户喜欢海盗笑话。

### 成功标志

- `AgentWithRAG_Step03_CustomRAGDataSource` 中，回答应围绕 `MockSearchAsync` 返回的退货、配送、帐篷保养片段展开。
- `AgentWithRAG_Step01_BasicTextRAG` 中，程序先 `UpsertDocumentsAsync(GetSampleDocuments())`，再对问题执行向量检索，回答内容与 `Contoso Outdoors Return Policy`、`Shipping Guide` 或 `TrailRunner Tent Care Instructions` 对应。
- `AgentWithMemory_Step01_ChatHistoryMemory` 中，第二个 session 的 “Tell me a joke that I might like.” 应体现对上一 session 偏好的召回。

## 核心概念拆解

### 1. Knowledge 是什么

Knowledge 是系统可检索、可治理、相对稳定的业务事实来源。它可以来自 Markdown、PDF、数据库、搜索服务、图数据库、工单系统或企业知识库。

在本仓库中，Knowledge 的最小形态体现在 `GetSampleDocuments()`：每个 `TextSearchDocument` 包含：

- `SourceId`
- `SourceName`
- `SourceLink`
- `Text`

这说明框架示例关注的不只是“把文本塞进 Prompt”，还保留了来源信息，方便回答时引用和排查。

### 2. RAG 是什么

RAG 的执行链路可以简化为：

```text
用户问题
  -> 检索查询
  -> 搜索知识源或向量库
  -> 返回 TopK 相关片段
  -> 注入模型上下文
  -> 模型基于上下文生成答案
```

在 `AgentWithRAG_Step01_BasicTextRAG` 中，关键代码是：

```csharp
AIContextProviders = [new TextSearchProvider(SearchAdapter, textSearchOptions)]
```

`SearchAdapter` 把 `TextSearchStore.SearchAsync(text, 1, ct)` 的结果转换成 `TextSearchProvider.TextSearchResult`。`TextSearchProviderOptions` 中的 `SearchTime = BeforeAIInvoke` 表示每次模型调用前先检索，再把结果注入上下文。

### 3. Memory 是什么

Memory 更关注“历史交互中沉淀出来、后续可能有用的信息”。仓库提供了两个层次的示例：

- `01-get-started/04_memory`：自定义 `UserInfoMemory : AIContextProvider`，从用户消息中抽取姓名和年龄，保存在 `ProviderSessionState<UserInfo>` 中，并在后续调用前提供 `Instructions`。
- `AgentWithMemory_Step01_ChatHistoryMemory`：使用 `ChatHistoryMemoryProvider` 把请求和响应消息写入向量存储，并在新 session 中按 `searchScope` 检索相关历史。

Memory 不等于把所有聊天记录永远塞进 Prompt。好的 Memory 需要筛选、分层、作用域隔离和过期策略。

### 4. RAG 与 Memory 如何协作

在一个企业支持 Agent 中，二者可以同时存在：

```text
当前用户问题
  + 当前会话最近消息（ChatHistoryProvider）
  + 企业知识检索结果（TextSearchProvider）
  + 用户长期偏好/历史事实（ChatHistoryMemoryProvider 或自定义 Memory）
  -> 模型生成答案
```

RAG 回答“企业规定是什么”，Memory 回答“这个用户/这个工单之前发生了什么”。如果用户问“我上次那个退款问题现在该怎么办”，RAG 需要找退款政策，Memory 需要找“上次那个问题”的上下文。

### 5. 常见误解是什么

- 误解一：向量库就是 Memory。实际上向量库只是存储与检索基础设施，Memory 还需要 scope、写入规则、召回策略和安全治理。
- 误解二：RAG 一定要有向量库。`AgentWithRAG_Step03_CustomRAGDataSource` 展示了可以用任意自定义搜索函数返回 `TextSearchResult`，背后可以是关键字搜索、数据库查询或企业搜索服务。
- 误解三：检索结果越多越好。结果过多会增加 token 成本，也会引入无关或冲突上下文。
- 误解四：Memory 应跨所有用户共享。生产环境必须按 user、tenant、application、agent、session 等维度隔离。

## 结合源码理解执行过程

### 1. RAG 入口在哪里

以 `AgentWithRAG_Step01_BasicTextRAG/Program.cs` 为例：

1. 创建 `AzureOpenAIClient`。
2. 创建 `InMemoryVectorStore`，并配置 `EmbeddingGenerator`。
3. 创建 `TextSearchStore(vectorStore, "product-and-policy-info", 3072)`。
4. 调用 `textSearchStore.UpsertDocumentsAsync(GetSampleDocuments())` 写入样本文档。
5. 创建 `SearchAdapter`，把 store 的结果转换为 `TextSearchProvider.TextSearchResult`。
6. 在 `ChatClientAgentOptions.AIContextProviders` 中注册 `new TextSearchProvider(SearchAdapter, textSearchOptions)`。
7. 调用 `agent.RunAsync(...)`。

### 2. RAG 调用是如何继续流转的

一次问题调用大致如下：

```text
agent.RunAsync("How long does standard shipping usually take?", session)
  -> ChatClientAgent 准备模型请求
  -> TextSearchProvider 在 BeforeAIInvoke 阶段调用 SearchAdapter
  -> SearchAdapter 调用 TextSearchStore.SearchAsync(query, top)
  -> TextSearchStore.SearchCoreAsync 使用 VectorStoreCollection.SearchAsync 或 HybridSearchAsync
  -> 返回 TextSearchResult
  -> TextSearchProvider 把结果作为上下文消息加入模型请求
  -> 模型基于上下文与 Instructions 回答
```

`TextSearchStore.SearchCoreAsync` 中有一个重要分支：如果底层 collection 支持 `IKeywordHybridSearchable<Dictionary<string, object?>>` 且未关闭 hybrid search，则使用 `HybridSearchAsync`；否则使用普通 `SearchAsync`。这说明 sample 已经展示了“向量检索 + 可选混合检索”的设计方向。

### 3. Memory 入口在哪里

以 `ChatHistoryMemoryProvider` 为例，核心入口在：

- `ProvideAIContextAsync`：模型调用前提供上下文。
- `ProvideMessagesAsync`：在 `BeforeAIInvoke` 模式下，根据当前请求文本检索相关历史，并返回一条包含 Memories 的 `ChatMessage`。
- `StoreAIContextAsync`：模型调用后，把本轮请求和响应消息写入向量库。
- `SearchChatHistoryAsync`：基于 `ApplicationId`、`AgentId`、`UserId`、`SessionId` 构造过滤条件，然后调用 collection 的 `SearchAsync`。

关键源码位置：`dotnet/src/Microsoft.Agents.AI/Memory/ChatHistoryMemoryProvider.cs`。

### 4. 关键对象分别负责什么

- `AIContextProvider`：在 Agent 调用前后注入、保存或转换上下文。
- `TextSearchProvider`：负责把“搜索函数”适配成 AI 上下文来源。
- `TextSearchStore`：sample 中的文档存储与检索封装，定义固定 schema，包括 `Text`、`SourceName`、`SourceLink`、`TextEmbedding` 等字段。
- `VectorStore`：向量存储抽象，可替换为内存、Qdrant、Azure AI Search 或其他实现。
- `ChatHistoryMemoryProvider`：把历史消息作为长期记忆写入向量库，并在后续根据语义相似度召回。
- `ProviderSessionState<T>`：把 provider 的状态与 `AgentSession` 绑定，例如 Memory 的 storage/search scope。
- `InMemoryChatHistoryProvider`：管理本地聊天历史；在 RAG 示例中还通过 message filter 避免把 AIContextProvider 生成的检索消息再次写进聊天历史，防止历史膨胀。

### 5. 扩展点在哪里

- 换知识源：替换 `SearchAdapter` 或 `MockSearchAsync`，保持返回 `IEnumerable<TextSearchProvider.TextSearchResult>` 即可。
- 换向量库：替换 `InMemoryVectorStore` 为其他 `Microsoft.Extensions.VectorData` 实现。
- 改召回数量：调整 `textSearchStore.SearchAsync(text, 1, ct)` 中的 `top`，或调整 Memory 的 `MaxResults`。
- 改检索时机：调整 `TextSearchProviderOptions.SearchTime` 或 `ChatHistoryMemoryProviderOptions.SearchTime`。
- 改作用域：调整 `ChatHistoryMemoryProvider.State(storageScope, searchScope)`，例如按 `UserId` 跨 session 检索，或按 `SessionId` 只在当前会话检索。
- 改持久化策略：把 `InMemoryVectorStore` 换成持久化向量库，并给 collection 设计 schema、索引和权限过滤。

## 关键代码或配置讲解

### 代码片段 1：注册 RAG 上下文 Provider

```csharp
TextSearchProviderOptions textSearchOptions = new()
{
    SearchTime = TextSearchProviderOptions.TextSearchBehavior.BeforeAIInvoke,
};

AIAgent agent = azureOpenAIClient
    .GetChatClient(deploymentName)
    .AsAIAgent(new ChatClientAgentOptions
    {
        ChatOptions = new() { Instructions = "You are a helpful support specialist for Contoso Outdoors. Answer questions using the provided context and cite the source document when available." },
        AIContextProviders = [new TextSearchProvider(SearchAdapter, textSearchOptions)],
    });
```

- 作用：让 Agent 每次调用模型前先执行检索，把知识片段注入上下文。
- 为什么重要：这是 RAG 与 Agent 主调用链的连接点。
- 改动它会影响什么：如果移除 `TextSearchProvider`，模型将无法看到外部知识片段；如果修改 `SearchTime`，会改变检索触发时机。

### 代码片段 2：避免 RAG 检索消息污染聊天历史

`AgentWithRAG_Step01_BasicTextRAG` 中配置了：

```csharp
ChatHistoryProvider = new InMemoryChatHistoryProvider(new InMemoryChatHistoryProviderOptions
{
    StorageInputRequestMessageFilter = messages => messages.Where(m =>
        m.GetAgentRequestMessageSourceType() != AgentRequestMessageSourceType.AIContextProvider &&
        m.GetAgentRequestMessageSourceType() != AgentRequestMessageSourceType.ChatHistory)
}),
```

- 作用：检索结果用于本轮推理，但不重复写入聊天历史。
- 为什么重要：如果把每次 RAG 结果都写进历史，后续轮次会不断叠加旧检索片段，导致 token 膨胀和上下文污染。
- 改动它会影响什么：过滤过严可能导致必要历史缺失；过滤过松可能导致重复上下文和成本上升。

### 代码片段 3：Memory 的存储与搜索 scope

`AgentWithMemory_Step01_ChatHistoryMemory` 中：

```csharp
AIContextProviders = [new ChatHistoryMemoryProvider(
    vectorStore,
    collectionName: "chathistory",
    vectorDimensions: 3072,
    session => new ChatHistoryMemoryProvider.State(
        storageScope: new() { UserId = "UID1", SessionId = Guid.NewGuid().ToString() },
        searchScope: new() { UserId = "UID1" }))]
```

- 作用：每个 session 存储时使用不同 `SessionId`，但搜索时只按 `UserId`，因此新 session 可以检索同一用户旧 session 的记忆。
- 为什么重要：这是长期 Memory 与短期 Session 的边界设计。
- 改动它会影响什么：如果 `searchScope` 也包含当前 `SessionId`，跨 session 记忆召回会消失；如果 scope 太宽，可能发生跨用户或跨租户数据泄漏。

### 代码片段 4：有界历史与向量溢出

`AgentWithMemory_Step05_BoundedChatHistory` 的设计是：

```csharp
var boundedProvider = new BoundedChatHistoryProvider(
    maxSessionMessages: 4,
    vectorStore,
    collectionName: "chathistory-overflow",
    vectorDimensions: 3072,
    session => new ChatHistoryMemoryProvider.State(
        storageScope: new() { UserId = "UID1", SessionId = sessionId },
        searchScope: new() { UserId = "UID1" }));
```

- 作用：最近 4 条非 system 消息留在 session state，更旧消息溢出到向量库，后续按语义召回。
- 为什么重要：这是控制上下文窗口与长期记忆成本的典型方案。
- 改动它会影响什么：`maxSessionMessages` 越小，越依赖向量召回；越大，短期上下文更完整但 token 成本更高。

## 做一个最小改造实验

### 实验目标

把基础 RAG 示例从“只取 Top1 搜索结果”改为“取 Top2 搜索结果”，观察回答是否引用更多上下文，同时理解召回数量与上下文质量的关系。

### 改造点

在 `dotnet/samples/02-agents/AgentWithRAG/AgentWithRAG_Step01_BasicTextRAG/Program.cs` 中，将：

```csharp
var searchResults = await textSearchStore.SearchAsync(text, 1, ct);
```

改为：

```csharp
var searchResults = await textSearchStore.SearchAsync(text, 2, ct);
```

然后运行：

```powershell
cd D:\code\agent-framework\dotnet\samples\02-agents\AgentWithRAG\AgentWithRAG_Step01_BasicTextRAG
dotnet run
```

### 你应该观察什么

- 回答是否包含更多来源或更多细节。
- 对明确问题，例如 shipping，是否仍然集中在 Shipping Guide，而不是被 return policy 或 tent care 干扰。
- 输出 token 是否明显增加。

### 可能出现的问题

- TopK 增大后，模型可能引用不相关片段。
- 如果 embedding deployment 与 `vectorDimensions: 3072` 不匹配，可能出现向量维度相关错误。
- 如果环境变量或 Azure 登录不可用，程序会在创建客户端或调用模型时失败。

## 本章在综合案例中的增量

主案例仍然是 **企业内部知识与工单协同 Agent 系统**。

### 本章之前

到第 11 章为止，综合案例已经有了应用分层认知：用户入口、Agent 决策层、工具层、状态层和流程层。但它仍主要依赖 Prompt、当前会话和工具调用，缺少两个关键能力：

- 对企业知识库的可靠查询能力。
- 对用户偏好、历史工单上下文和跨 session 信息的长期记忆能力。

### 本章加入的新能力

本章新增 **知识检索层 + 记忆层**：

```text
用户问题
  -> Agent 决策层
  -> Knowledge Retriever：检索政策、FAQ、SOP、产品文档
  -> Memory Provider：召回用户偏好、历史工单、当前任务事实
  -> 模型生成带来源的回答或处理建议
```

### 修改了哪些模块、流程或边界

- 用户入口：仍然接收自然语言问题，但需要带上 `UserId`、`TenantId`、`SessionId` 等上下文，以便检索与记忆隔离。
- Agent 决策层：判断问题是否需要知识检索或记忆召回。
- 知识层：参考 `TextSearchProvider`，把企业知识库封装成 `SearchAdapter`。
- 状态层：参考 `ChatHistoryMemoryProvider.State`，明确 `storageScope` 与 `searchScope`。
- 回答层：要求回答中尽量引用 `SourceName` / `SourceLink`，避免无来源的断言。

### 加入后系统能力发生了什么变化

- 员工问“退款政策是什么”时，Agent 可以检索最新政策文档，而不是凭模型记忆回答。
- 员工问“我上次那个 VIP 客户的退货问题怎么办”时，Agent 可以同时召回历史工单上下文和当前退货政策。
- 当会话变长时，可以采用 `BoundedChatHistoryProvider` 类似思路，把近期消息留在窗口，旧消息进入向量库。

### 如果不加入这一章的能力会卡在哪里

系统只能做通用问答和简单工具调用，无法可靠回答企业私有知识，也无法跨 session 延续个性化服务；一旦对话变长，还会被上下文窗口和 token 成本限制。

### 与仓库已有源码的关系

- 仓库已有 sample 直接支持：`AgentWithRAG_Step01_BasicTextRAG`、`AgentWithRAG_Step03_CustomRAGDataSource`、`AgentWithMemory_Step01_ChatHistoryMemory`。
- 可以基于已有 sample 最小改造：把 `MockSearchAsync` 替换为企业知识库查询；把固定 `UID1` 改为真实用户和租户 scope；把 `InMemoryVectorStore` 替换为持久化向量库。
- 仓库未直接给出现成样例、但可基于框架能力扩展：完整的文档增量索引流水线、权限感知检索、答案引用校验、检索质量评测、跨知识源重排序和企业审计闭环。

## 常见问题与排错

### 问题 1：运行 sample 提示 `AZURE_OPENAI_ENDPOINT is not set`

- 现象：程序启动即抛出 `InvalidOperationException`。
- 原因：sample 通过 `Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")` 读取 Azure OpenAI endpoint。
- 排查方法：在当前 shell 中检查环境变量是否存在。
- 修复方式：设置 `AZURE_OPENAI_ENDPOINT`，并确认账号可访问该资源。

### 问题 2：embedding 维度不匹配

- 现象：向量写入或搜索时报维度不一致。
- 原因：sample 使用 `vectorDimensions: 3072`，默认 embedding deployment 是 `text-embedding-3-large`；如果换成其他 embedding 模型，维度可能不同。
- 排查方法：确认 `AZURE_OPENAI_EMBEDDING_DEPLOYMENT_NAME` 对应模型及其向量维度。
- 修复方式：同步修改 `TextSearchStore(..., 3072)` 或 `ChatHistoryMemoryProvider(..., vectorDimensions: 3072)` 的维度参数。

### 问题 3：RAG 回答没有引用知识片段

- 现象：回答像通用模型知识，没有提到 source document。
- 原因：搜索函数没有返回结果、TopK 太小、query 与文档不匹配，或 Instructions 没要求引用来源。
- 排查方法：在 `SearchAdapter` 或 `MockSearchAsync` 中打印 query 和返回结果数量。
- 修复方式：调整检索 query、TopK、文档分块、source 字段和 system instructions。

### 问题 4：Memory 在新 session 中没有生效

- 现象：第二个 session 记不住第一个 session 的偏好。
- 原因：`searchScope` 太窄，例如按新 `SessionId` 检索；或旧消息没有成功写入向量库。
- 排查方法：检查 `ChatHistoryMemoryProvider.State` 的 `storageScope` 与 `searchScope`；打开日志观察是否调用 `UpsertAsync` 和 `SearchAsync`。
- 修复方式：对同一用户跨 session 记忆，搜索 scope 至少要共享 `UserId`；生产中还应加入 `TenantId` 或 `ApplicationId` 等隔离字段。

### 问题 5：上下文越来越长、成本越来越高

- 现象：多轮对话后延迟增加、费用上升，甚至超出上下文窗口。
- 原因：把检索结果和历史消息重复写入聊天历史，或没有限制历史窗口。
- 排查方法：检查 `ChatHistoryProvider` 的 message filter；观察每轮模型请求中的 messages 数量。
- 修复方式：参考 `AgentWithRAG_Step01_BasicTextRAG` 的 `StorageInputRequestMessageFilter`，并参考 `AgentWithMemory_Step05_BoundedChatHistory` 使用有界历史与向量溢出。

### 问题 6：检索结果带来 Prompt Injection 风险

- 现象：文档中包含“忽略之前指令”“泄露系统提示”等恶意内容，模型可能跟随执行。
- 原因：`ChatHistoryMemoryProvider` 源码注释明确提醒：从向量库检索并注入上下文的内容如果被污染，会影响模型行为；检索结果默认按原样进入上下文。
- 排查方法：对外部文档、用户生成内容和历史消息进行来源审计；检查是否把不可信内容当作系统指令注入。
- 修复方式：把检索内容明确标记为 untrusted context；加入内容过滤、引用边界、权限校验和回答约束。

## 企业级落地提醒

### 稳定性

- 不要把 `InMemoryVectorStore` 当作生产长期存储；sample 主要用于演示。
- 检索链路应有超时、重试、降级策略。检索失败时，Agent 应说明“无法访问知识库”，而不是编造答案。
- 文档索引与查询路径要版本化，避免模型回答混用新旧政策。

### 可维护性

- 文档 schema 要清晰，包括 `SourceId`、`SourceName`、`SourceLink`、`LastUpdatedAt`、`TenantId`、`Acl` 等字段。
- RAG adapter 应作为独立模块，而不是散落在业务 Prompt 中。
- Memory 的 scope 规则要写成架构决策记录，尤其是“什么可以跨 session 共享，什么必须只属于当前任务”。

### 权限边界

- 检索时必须做权限过滤，不能只依赖模型“不要泄露”。
- 多租户系统中，向量库 collection、namespace 或 filter 必须包含租户隔离字段。
- Memory 搜索 scope 不能只用自然语言相似度，还要先按用户、租户、应用、Agent 等维度过滤。

### 安全风险

- 检索内容和历史 Memory 都可能包含 Prompt Injection、敏感信息或过期规则。
- 日志中不要记录完整 query、检索结果和用户隐私。`ChatHistoryMemoryProvider` 源码也提醒 Trace 日志可能包含 PII。
- 对用户消息、模型响应和检索片段都要设计脱敏与审计策略。

### 成本控制

- 控制 TopK、chunk 大小和上下文注入格式。
- 对高频问题可缓存检索结果或最终答案，但必须考虑权限和版本。
- 对 Memory 进行摘要、过期和压缩，避免把所有历史都长期向量化。

### 可观测性与可测试性

- 记录每次 RAG 的 query、命中文档 ID、score、TopK、是否使用 hybrid search，但注意脱敏。
- 建立 RAG 回归集：问题、期望命中文档、期望答案要点、禁止回答内容。
- 对 Memory 建立测试：同用户跨 session 可召回，不同用户和不同租户不能召回。

## 本章小结

- 本章最重要的结论 1：RAG 与 Memory 都是上下文工程的一部分，但 RAG 面向企业知识，Memory 面向用户、会话和任务历史。
- 本章最重要的结论 2：仓库已经提供了 `TextSearchProvider`、`ChatHistoryMemoryProvider`、`AIContextProvider`、`VectorStore` 等关键入口，可以支撑最小 RAG 和记忆增强实践。
- 本章最重要的结论 3：生产级 Knowledge/RAG/Memory 不只是“接一个向量库”，还必须处理权限、scope、成本、注入风险、持久化、评测和可观测性。

下一章会进入 Tool、MCP 与外部系统集成。理解本章后，你已经知道 Agent 如何“读知识”和“记上下文”；下一步要解决的是 Agent 如何安全、稳定地“调用系统做事”。

## 本章建议产出物

- 一次 `AgentWithRAG_Step03_CustomRAGDataSource` 或 `AgentWithRAG_Step01_BasicTextRAG` 的运行记录。
- 一张“用户问题 -> RAG 检索 -> Memory 召回 -> 模型回答”的调用链图。
- 一个 TopK 从 1 改为 2 的最小改造实验记录。
- 一份企业知识库字段设计草案。
- 一份 Memory scope 规则清单：哪些信息按 session 保存，哪些按 user 保存，哪些绝不跨租户共享。

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
