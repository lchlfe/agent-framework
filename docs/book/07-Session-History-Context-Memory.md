# 第 7 章：Session、History、Context 与 Memory

> 本章定位：把前几章的单轮 Agent 调用升级为可恢复、可续聊、可注入上下文、可跨会话记忆的应用原型。

## 本章目标

学完本章后，你应能够：

- 区分 `AgentSession`、`ChatHistoryProvider`、`AIContextProvider`、`ChatHistoryMemoryProvider` 各自负责什么。
- 基于仓库 sample 跑通“序列化会话”和“跨会话记忆”两个最小场景。
- 解释一次多轮调用中，会话状态、历史消息、上下文注入、向量记忆如何参与 Agent 执行。
- 判断哪些能力是仓库已经提供的，哪些属于企业项目中需要继续扩展的长期记忆、权限与治理能力。

## 本章与前后章节的关系

- 前置知识：第 3-5 章中的最小 Agent、Prompt、消息输入输出；第 6 章中的 `AIAgent`、Builder、Options 等核心抽象。
- 本章解决：如何让 Agent 不再“每次调用都失忆”，而是能保存会话、读取历史、注入上下文、使用 Memory。
- 后续衔接：第 8 章会把有状态对话放入 Workflow；第 10 章会进一步展开状态持久化、恢复与长任务；第 12 章会深入 Knowledge、RAG 与长期 Memory 体系。

## 先建立整体认知

在 Agent 应用里，本章四个词经常被混用，但它们的边界不同：

- **Session**：一次用户与某个 Agent 的会话状态容器。仓库中的核心类型是 `AgentSession`，它可保存历史引用、Memory 引用和其他跨轮次状态，并可序列化/反序列化。
- **History**：按时间顺序保存的聊天消息。仓库中的核心抽象是 `ChatHistoryProvider`，它负责在调用前提供历史、调用后存储新消息。
- **Context**：本次模型调用实际看到的上下文，包括用户新输入、历史消息、系统指令、工具结果、Memory 检索结果等。仓库中 `AIContextProvider` 及其实现可在调用前注入额外上下文。
- **Memory**：比单个上下文窗口更长期的记忆。仓库中 `ChatHistoryMemoryProvider` 会把聊天消息写入 `VectorStore`，并在后续调用中按相似度检索相关历史。

如果缺少这一层，Agent 会出现典型问题：第二轮不知道第一轮说过什么；应用重启后无法续聊；用户偏好无法跨会话复用；上下文窗口无限增长导致成本和延迟上升；历史数据未经治理又可能带来隐私与间接 Prompt Injection 风险。

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：
  - `dotnet/src/Microsoft.Agents.AI.Abstractions/`
  - `dotnet/src/Microsoft.Agents.AI/Memory/`
  - `dotnet/src/Microsoft.Agents.AI/ChatClient/`
  - `dotnet/src/Microsoft.Agents.AI.Foundry/Memory/`
  - `dotnet/src/Microsoft.Agents.AI.CosmosNoSql/`
  - `dotnet/samples/02-agents/Agents/`
  - `dotnet/samples/02-agents/AgentWithMemory/`
- 项目：
  - `Microsoft.Agents.AI.Abstractions`
  - `Microsoft.Agents.AI`
  - `Microsoft.Agents.AI.Foundry`
  - `Microsoft.Agents.AI.CosmosNoSql`
- Sample：
  - `dotnet/samples/02-agents/Agents/Agent_Step03_PersistedConversations`
  - `dotnet/samples/02-agents/Agents/Agent_Step04_3rdPartyChatHistoryStorage`
  - `dotnet/samples/02-agents/Agents/Agent_Step17_AdditionalAIContext`
  - `dotnet/samples/02-agents/AgentWithMemory/AgentWithMemory_Step01_ChatHistoryMemory`
  - `dotnet/samples/02-agents/AgentWithMemory/AgentWithMemory_Step04_MemoryUsingFoundry`
  - `dotnet/samples/02-agents/AgentWithMemory/AgentWithMemory_Step05_BoundedChatHistory`
  - `dotnet/samples/02-agents/AgentsWithFoundry/Agent_Step02.1_MultiturnConversation`
  - `dotnet/samples/02-agents/AgentsWithFoundry/Agent_Step02.2_MultiturnWithServerConversations`
  - `dotnet/samples/02-agents/AgentsWithFoundry/Agent_Step06_PersistedConversations`
  - `dotnet/samples/02-agents/AgentsWithFoundry/Agent_Step22_MemorySearch`
- 关键类型：
  - `AgentSession`
  - `AgentSessionStateBag`
  - `ProviderSessionState<TState>`
  - `ChatClientAgentSession`
  - `ChatHistoryProvider`
  - `InMemoryChatHistoryProvider`
  - `ChatHistoryMemoryProvider`
  - `ChatHistoryMemoryProvider.State`
  - `ChatHistoryMemoryProviderScope`
  - `FoundryMemoryProvider`
  - `CosmosChatHistoryProvider`
- 关键接口/扩展点：
  - `AIAgent.CreateSessionAsync`
  - `AIAgent.SerializeSessionAsync`
  - `AIAgent.DeserializeSessionAsync`
  - `ChatHistoryProvider.ProvideChatHistoryAsync`
  - `ChatHistoryProvider.StoreChatHistoryAsync`
  - `AIContextProvider.InvokingAsync` / `InvokedAsync`
  - `ChatClientAgentOptions.AIContextProviders`
  - `VectorStore` 与 `VectorStoreCollection`

推荐先看 `Agent_Step03_PersistedConversations`，再看 `AgentWithMemory_Step01_ChatHistoryMemory`，最后看 `AgentWithMemory_Step05_BoundedChatHistory`。前者展示 Session 持久化，第二个展示跨会话 Memory，第三个展示窗口内 History 与向量 Memory 的组合策略。

## 先跑通或先验证一个最小示例

### 示例目标

- 验证 `AgentSession` 能被创建、序列化、反序列化，并保留上一轮对话上下文。
- 验证 `ChatHistoryMemoryProvider` 能把上一个 Session 中的用户偏好写入向量存储，并在新 Session 中检索使用。

### 运行前提

- 已配置 .NET SDK，并能在 `dotnet` 目录下运行 sample。
- 使用 Azure OpenAI 相关 sample 时，需要环境变量：
  - `AZURE_OPENAI_ENDPOINT`
  - `AZURE_OPENAI_DEPLOYMENT_NAME`
  - `AZURE_OPENAI_EMBEDDING_DEPLOYMENT_NAME`（Memory sample 需要）
- 本地开发可使用 `DefaultAzureCredential`，生产环境应改用更明确的凭据策略，例如 Managed Identity。

### 操作步骤

1. 进入仓库中的 .NET 目录：

   ```bash
   cd dotnet
   ```

2. 运行会话持久化 sample：

   ```bash
   dotnet run --project samples/02-agents/Agents/Agent_Step03_PersistedConversations
   ```

3. 观察输出中是否出现：

   ```text
   --- Serialized session ---
   ```

   以及后续让 Agent “用海盗口吻复述同一个笑话”的回答。

4. 运行跨会话 Memory sample：

   ```bash
   dotnet run --project samples/02-agents/AgentWithMemory/AgentWithMemory_Step01_ChatHistoryMemory
   ```

5. 观察第二个 Session 中，用户只说 `Tell me a joke that I might like.`，Agent 是否仍能利用第一轮里“用户喜欢 pirate jokes”的偏好。

### 预期现象

- `Agent_Step03_PersistedConversations` 会在第一次回答后打印序列化后的 Session JSON，然后用反序列化得到的 Session 继续对话。
- `AgentWithMemory_Step01_ChatHistoryMemory` 会先记录用户偏好，再在新 Session 中通过向量检索补充相关历史。

### 成功标志

- 你能看到序列化 Session JSON，并且反序列化后 Agent 仍知道上一轮笑话。
- 新 Session 中没有显式重复“我喜欢海盗笑话”，Agent 仍倾向于给出海盗相关笑话或承接该偏好。

## 核心概念拆解

### 1. Session 是什么

`dotnet/src/Microsoft.Agents.AI.Abstractions/AgentSession.cs` 中的注释明确说明：`AgentSession` 是所有 agent thread 的基础抽象，包含特定对话的状态，可能包括 conversation history、外部 history 引用、memories、外部 memory 引用，以及 Agent 需要跨运行持久化的其他状态。

关键点：

- `AgentSession` 总是由 `AIAgent` 创建，典型入口是 `CreateSessionAsync`。
- Session 不一定能跨不同 Agent 复用，因为不同 Agent 可能给 Session 附加不同的行为。
- Session 可以通过 `SerializeSessionAsync` 转成 `JsonElement`，再通过 `DeserializeSessionAsync` 恢复。
- `StateBag` 会随 Session 序列化，可用于保存 provider 的状态，例如 Memory 的 scope 或 History provider 的内部状态。

### 2. History 是什么

`ChatHistoryProvider` 是管理聊天历史的抽象。它的职责不是“回答问题”，而是：

- 调用前：取出按时间顺序排列的历史消息，合并进本次请求上下文。
- 调用后：把本次用户输入和模型输出写回历史。
- 必要时：做截断、摘要、归档等优化。

源码中特别提醒：`ChatHistoryProvider` 会通过 `InvokingContext` 和 `InvokedContext` 拿到 `AgentSession`，因此 provider 不应把 session-specific 信息放在自身字段里，而应放在 `AgentSession.StateBag` 中。否则多个 Session 共用一个 provider 实例时容易串数据。

### 3. Context 是什么

Context 是一次模型调用真正看到的输入集合。它不等于完整历史，也不等于 Memory 全量内容。典型 Context 由以下部分组成：

- 系统指令或 Agent instructions。
- 用户本轮输入。
- `ChatHistoryProvider` 提供的近期历史。
- `AIContextProvider` 注入的补充材料。
- Tool 调用及函数结果。
- Memory 检索结果。

仓库中 `Agent_Step17_AdditionalAIContext` 是理解“额外上下文注入”的参考入口；`ChatHistoryMemoryProvider` 继承自 `MessageAIContextProvider`，也是一种把检索结果注入上下文的实现。

### 4. Memory 是什么

`ChatHistoryMemoryProvider` 位于 `dotnet/src/Microsoft.Agents.AI/Memory/ChatHistoryMemoryProvider.cs`。它的注释说明：该 provider 会把聊天历史写入 `VectorStore`，之后用语义相似度检索相关历史，以增强当前对话。

它支持两类典型使用方式：

- 自动检索并注入上下文：调用前根据当前消息搜索相关记忆，并把结果加入上下文。
- On-demand function calling：把搜索能力暴露为函数工具，由模型决定何时搜索。

仓库已支持基于 `VectorStore` 的 Memory provider，也有 Foundry Memory 相关实现和 sample。企业项目中常见的“用户画像长期记忆”“组织级知识偏好”“审计可删除记忆”等，则通常需要在这些抽象上继续设计数据模型、权限和生命周期。

### 5. 它们之间如何协作

一次典型多轮调用可理解为：

```text
用户输入
  -> AIAgent.RunAsync(input, session)
  -> ChatHistoryProvider 在调用前加载历史
  -> AIContextProvider / ChatHistoryMemoryProvider 注入额外上下文或检索记忆
  -> ChatClient 调用模型
  -> Agent 得到响应
  -> ChatHistoryProvider 写回本轮消息
  -> ChatHistoryMemoryProvider 写入向量存储
  -> session.StateBag 记录 provider 状态
  -> 必要时 SerializeSessionAsync 持久化 session
```

### 6. 常见误解是什么

- 误解 1：Session 就等于完整聊天记录。实际上 Session 可以保存完整记录，也可以只保存外部存储引用、scope 或 provider 状态。
- 误解 2：Memory 就是把所有历史塞进 prompt。真正的 Memory 通常需要检索、裁剪、摘要、权限过滤和遗忘策略。
- 误解 3：向量库里的历史一定可信。源码注释明确警告：检索出的内容会作为上下文进入 LLM，存储被污染时会形成间接 Prompt Injection。
- 误解 4：Provider 可以把状态存在自身字段。仓库设计要求 session-specific 状态应放入 `AgentSession.StateBag`。

## 结合源码理解执行过程

### 1. 入口在哪里

最小入口在 `Agent_Step03_PersistedConversations/Program.cs`：

```csharp
AgentSession session = await agent.CreateSessionAsync();
Console.WriteLine(await agent.RunAsync("Tell me a joke about a pirate.", session));
JsonElement serializedSession = await agent.SerializeSessionAsync(session);
AgentSession resumedSession = await agent.DeserializeSessionAsync(serializedSession);
Console.WriteLine(await agent.RunAsync("Now tell the same joke in the voice of a pirate...", resumedSession));
```

这段代码展示了 Session 生命周期的主线：创建 -> 使用 -> 序列化 -> 恢复 -> 继续使用。

### 2. 调用是如何继续流转的

在 ChatClient Agent 路径中，Agent 调用会围绕 `AgentSession` 组织状态。若配置了 History 或 Context provider，框架会在模型调用前后触发 provider：

- 调用前：把 session 中已有的历史和 provider 注入的上下文合并到请求消息中。
- 模型调用：ChatClient 根据最终消息集合生成响应。
- 调用后：provider 收到 request/response，把需要保留的内容写回 session 或外部存储。

### 3. 关键对象分别负责什么

- `AgentSession`：保存会话状态和 provider 状态，是续聊与恢复的核心容器。
- `AgentSessionStateBag`：随 Session 序列化的任意状态字典。
- `ChatHistoryProvider`：加载和保存按顺序排列的聊天消息。
- `InMemoryChatHistoryProvider`：仓库提供的内存历史实现，适合 demo 或短期状态。
- `ChatHistoryMemoryProvider`：把聊天消息写入向量存储，并在后续调用中检索相关记忆。
- `VectorStore`：Memory 的底层存储抽象，可替换为不同向量库实现。
- `FoundryMemoryProvider`：面向 Foundry Memory 的实现，适合进一步了解云端 Memory 服务。

### 4. 扩展点在哪里

- 想换历史存储：实现或配置 `ChatHistoryProvider`，参考 `CosmosChatHistoryProvider` 和 `Agent_Step04_3rdPartyChatHistoryStorage`。
- 想换长期记忆：使用 `ChatHistoryMemoryProvider` 的 `VectorStore` 抽象，或参考 `FoundryMemoryProvider`。
- 想控制记忆范围：调整 `ChatHistoryMemoryProvider.State` 中的 `storageScope` 与 `searchScope`。
- 想控制上下文窗口：参考 `AgentWithMemory_Step05_BoundedChatHistory` 中的 `BoundedChatHistoryProvider` 与 `TruncatingChatReducer`。

## 关键代码或配置讲解

### 代码片段 1：Session 序列化与恢复

位置：`dotnet/samples/02-agents/Agents/Agent_Step03_PersistedConversations/Program.cs`

- 作用：展示 `CreateSessionAsync`、`SerializeSessionAsync`、`DeserializeSessionAsync` 的最小闭环。
- 为什么重要：这是 Web/API 场景中“每个请求恢复会话、处理本轮输入、再保存会话”的基础。
- 改动它会影响什么：如果你不传入恢复后的 `AgentSession`，第二轮 Agent 就无法基于上一轮状态继续回答。

### 代码片段 2：ChatHistoryMemoryProvider 的配置

位置：`dotnet/samples/02-agents/AgentWithMemory/AgentWithMemory_Step01_ChatHistoryMemory/Program.cs`

```csharp
AIContextProviders = [new ChatHistoryMemoryProvider(
    vectorStore,
    collectionName: "chathistory",
    vectorDimensions: 3072,
    session => new ChatHistoryMemoryProvider.State(
        storageScope: new() { UserId = "UID1", SessionId = Guid.NewGuid().ToString() },
        searchScope: new() { UserId = "UID1" }))]
```

- 作用：把聊天历史写入向量库，并允许同一用户跨 Session 搜索旧消息。
- 为什么重要：`storageScope` 控制“写到哪里”，`searchScope` 控制“从哪里搜”。sample 使用相同 `UserId`、不同 `SessionId` 来展示跨会话用户偏好。
- 改动它会影响什么：如果把 `searchScope` 也限制到当前 `SessionId`，第二个 Session 就无法找回第一个 Session 的偏好。

### 代码片段 3：有界历史与溢出 Memory

位置：`dotnet/samples/02-agents/AgentWithMemory/AgentWithMemory_Step05_BoundedChatHistory/BoundedChatHistoryProvider.cs`

- 作用：用 `InMemoryChatHistoryProvider` 保留近期窗口，用 `ChatHistoryMemoryProvider` 把被截断的旧消息归档到向量库。
- 为什么重要：真实应用不能无限增长 prompt；需要把“近期上下文”和“可检索长期记忆”分层处理。
- 改动它会影响什么：调小 `maxSessionMessages` 会更频繁触发截断和归档；调大则上下文更完整但成本和延迟更高。

## 做一个最小改造实验

### 实验目标

让主案例“企业内部知识与工单协同 Agent 系统”从单轮问答升级为“同一员工跨会话保留偏好”的原型。

### 改造点

基于 `AgentWithMemory_Step01_ChatHistoryMemory` 做最小改造：

1. 把 Agent instructions 从讲笑话改成工单助手：

   ```csharp
   ChatOptions = new() { Instructions = "You are an internal IT support agent. Remember the user's support preferences when appropriate." }
   ```

2. 把第一轮输入改为：

   ```text
   我是财务部员工，之后给我排查电脑问题时，请优先给 Windows 方案，并用中文回答。
   ```

3. 把第二个 Session 的输入改为：

   ```text
   我的电脑连不上公司 VPN，应该怎么排查？
   ```

4. 保持 `storageScope.UserId = "UID1"`，并让 `searchScope` 仍只按 `UserId` 搜索。

### 你应该观察什么

- 第二个 Session 是否能倾向于中文回答。
- 第二个 Session 是否优先给出 Windows 相关 VPN 排查步骤。
- 如果把 `searchScope` 改成新的随机 `SessionId`，这种偏好是否消失。

### 可能出现的问题

- 向量检索未命中：第一轮偏好文本太短或第二轮问题相关性不足，可把偏好写得更明确。
- 记忆过度影响回答：如果旧偏好不再适用，需要设计用户可清除、可覆盖的记忆机制。
- 本地 `InMemoryVectorStore` 重启即丢失：sample 是演示用，生产环境应使用可持久化向量存储。

## 本章在综合案例中的增量

主案例统一为：**企业内部知识与工单协同 Agent 系统**。

- 本章之前：系统已经可以用最小 Agent 接收员工问题、基于 Prompt 回答，并可调用简单 Tool，但每次交互基本是短记忆或无记忆。
- 本章加入：会话状态、聊天历史、上下文注入和跨会话 Memory。
- 修改的模块：
  - 用户入口需要携带 `userId`、`sessionId`。
  - Agent 层需要创建或恢复 `AgentSession`。
  - 状态层需要保存序列化 Session，或保存外部 history/memory 的引用。
  - Memory 层需要按员工、部门、会话等 scope 写入和检索历史。
- 加入后能力变化：员工可以连续追问同一工单；应用重启后可恢复会话；系统能记住员工偏好，例如语言、系统环境、常用部门。
- 如果不加入本章能力：工单助手只能处理一次性问题，无法承接上下文，也无法逐步演进到 Workflow、状态恢复和 RAG。
- 与仓库已有内容关系：
  - 仓库已有 sample 直接支持：Session 序列化、ChatHistoryMemoryProvider、Foundry Memory、有界 History。
  - 可基于已有 sample 最小改造：按 `userId/sessionId/departmentId` 设置 `storageScope` 与 `searchScope`。
  - 仓库未直接给出现成样例：完整企业工单系统的权限模型、数据保留策略、用户可管理长期记忆界面。

## 常见问题与排错

### 问题 1：第二轮对话没有记住第一轮

- 现象：Agent 像第一次见到用户一样回答。
- 原因：没有复用同一个 `AgentSession`，或反序列化后没有把 `resumedSession` 传入 `RunAsync`。
- 排查方法：检查调用是否类似 `agent.RunAsync(input, session)`，而不是每轮都新建 session。
- 修复方式：在请求结束时保存 `SerializeSessionAsync` 的结果；下一次请求开始时用 `DeserializeSessionAsync` 恢复。

### 问题 2：跨 Session Memory 没生效

- 现象：第二个 Session 无法记住用户偏好。
- 原因：`searchScope` 与第一轮写入的 `storageScope` 不重叠；向量库没有持久化；embedding 配置错误。
- 排查方法：检查 `UserId`、`SessionId`、collectionName、vectorDimensions、embedding deployment 是否一致。
- 修复方式：让同一用户的 `searchScope` 至少包含相同 `UserId`；生产环境替换可持久化 VectorStore。

### 问题 3：上下文越来越长，成本和延迟上升

- 现象：多轮后响应变慢��token 使用增加，甚至超过模型上下文限制。
- 原因：把所有 History 都直接塞进上下文，没有截断、摘要或归档策略。
- 排查方法：打印每轮消息数量，参考 verify-samples 中对 `Chat history has N messages.` 的检查方式。
- 修复方式：使用 `InMemoryChatHistoryProviderOptions.ChatReducer`、`TruncatingChatReducer`，或参考 `BoundedChatHistoryProvider` 做窗口 + 向量归档。

### 问题 4：恢复 Session 后出现安全或隐私风险

- 现象：Agent 被旧消息“带偏”，或历史中含有敏感信息。
- 原因：Session JSON、History、Memory 都可能包含 PII 和用户输入；仓库注释明确说明恢复不可信 Session 等同于接受不可信输入。
- 排查方法：审计 Session 存储来源、访问控制、加密策略、消息角色是否被篡改。
- 修复方式：对存储做访问控制和加密；对可疑历史做校验、过滤或隔离；不要从不可信来源恢复会话。

## 企业级落地提醒

- 稳定性：不要依赖进程内内存保存关键会话。Web/API 场景应把序列化 Session 或外部引用保存到数据库、缓存或专用状态存储。
- 可维护性：明确区分近期 History、检索型 Memory、业务状态。不要把所有状态都塞进 Prompt，也不要把所有业务字段都塞进聊天历史。
- 权限边界：Memory 的 `searchScope` 必须按用户、租户、部门、数据等级隔离。跨用户检索很容易造成数据泄露。
- 安全风险：向量库、历史库和 Session 存储一旦被污染，检索结果会影响模型行为，必须防范间接 Prompt Injection。
- 成本控制：对历史消息做窗口化、摘要或向量检索；限制每次 Memory 检索数量，例如 `ChatHistoryMemoryProvider` 默认最大结果数是小集合而非全量注入。
- 可观测性：记录 sessionId、userId、history message count、memory hit count、检索耗时，但避免在 Trace 日志中泄露完整 PII。
- 可测试性：为多轮对话写回归测试，至少覆盖“同 Session 续聊”“跨 Session 用户偏好”“清除记忆后不再命中”三类行为。
- 持久化与恢复：本章只介绍会话与记忆的基础，长任务 checkpoint、失败恢复和工作流状态会在第 10 章进一步展开。

## 本章小结

- 本章最重要的结论 1：`AgentSession` 是会话状态容器，不只是聊天记录；它能保存 provider 状态并支持序列化恢复。
- 本章最重要的结论 2：`ChatHistoryProvider` 管近期历史，`AIContextProvider` 管上下文注入，`ChatHistoryMemoryProvider` 管可检索的向量化记忆。
- 本章最重要的结论 3：Memory 能提升连续性，但必须配套 scope、权限、遗忘、审计和成本控制，否则会带来泄露与注入风险。

下一章将进入 Workflow 与多步骤编排：当会话和上下文能够稳定管理后，我们就可以把“员工问题分类 -> 检索知识 -> 调用工单工具 -> 返回处理建议”组织成可维护的流程。

## 本章建议产出物

- 一次 `Agent_Step03_PersistedConversations` 运行记录。
- 一次 `AgentWithMemory_Step01_ChatHistoryMemory` 运行记录。
- 一张 Session/History/Context/Memory 调用链图。
- 一个把笑话 sample 改造成“企业内部工单偏好记忆”的实验记录。
- 一份 Memory scope 与权限边界问题清单。

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
