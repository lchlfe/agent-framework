# 第 3 章：第一个 .NET Agent

> 本章定位：把仓库里的第一个 .NET Agent 真正跑起来，并借这个最小 sample 建立你对 `AIAgent`、provider 接入、同步调用与流式输出的第一层执行认知。

## 本章目标

学完本章后，你应能够：

- 知道仓库里第一个 .NET Agent 对应哪个 sample、哪个项目文件、依赖了哪些源码模块
- 看懂 `01_hello_agent` 的最小执行链路：读取配置 → 创建模型客户端 → 转成 `AIAgent` → 调用 `RunAsync` / `RunStreamingAsync`
- 独立完成一次最小运行、最小修改和最小验证，而不是只会复制官方示例
- 理解这个“最小 Agent”在后续 Tool、Session、Memory、Workflow 学习中的基线作用

## 本章与前后章节的关系

- 前置知识：第 1 章已经建立项目全景；第 2 章已经完成本地环境准备并看懂仓库主目录
- 本章解决：第一个 .NET Agent 到底怎么创建、怎么运行、怎么输出结果、最小链路里有哪些关键对象
- 后续衔接：第 4 章会在这个最小 Agent 基础上加入 Tool 与 Function Calling；第 5 章会继续展开 Prompt、消息和最小交互链路

## 先建立整体认知

### 这个主题是什么

“第一个 .NET Agent”并不是指一个复杂应用，而是指：

- 你先用最少代码创建一个 Agent
- 它能真正调用到底层模型服务
- 它能返回一次完整结果
- 它还能用流式方式逐步输出更新

仓库里这个入口就是：

- `dotnet/samples/01-get-started/01_hello_agent/Program.cs`

这不是一个花哨 demo，而是后面一切能力的出发点。

### 为什么它重要

后面所有内容几乎都建立在这个基线上：

- Tool：是在这个 Agent 上增加函数调用能力
- Multi-turn：是在这个 Agent 运行时加入会话状态
- Memory：是在这个 Agent 调用前后注入和提取上下文
- Workflow：是把多个执行节点和 Agent 串成流程
- Hosting：是把这个 Agent 从本地控制台提升为长期运行的服务

如果你现在没有把“最小 Agent 到底如何被创建和调用”看清，后面会一直停留在“知道 API 名字，但不理解执行链路”的状态。

### 在 Agent 系统里处于哪一层

这一章关注的是 **Agent 本体创建与最小调用层**。

你可以先把它放进这样一条链里：

1. provider 与认证配置
2. 底层模型客户端
3. `AIAgent` 抽象
4. 单次请求调用
5. 控制台输出结果

当前还没有进入：

- 工具调用
- 多轮状态
- 记忆注入
- 工作流编排
- 托管部署

所以这一章的边界很清楚：**只解决“最小 Agent 如何跑起来”**。

### 如果缺少这一层，会出现什么问题

如果你跳过这一章，后面最常见的问题有三个：

- 你知道 sample 能跑，但不知道真正的入口对象是谁
- 你会把 Tool、Session、Memory 当成“某些额外魔法”，而不是建立在 `AIAgent` 基线上的增量能力
- 你一旦遇到 provider、认证、环境变量问题，就不知道该从哪一层排查

## 本章在综合案例中的增量

本书统一采用的主线案例是：

- **企业内部知识与工单协同 Agent 系统**

### 本章之前，综合案例处于什么状态

- 只有业务目标，还没有任何可运行代码
- 还没有“一个最小可交互 Agent”的基座
- 用户请求、Prompt、模型调用、输出链路都还没有落地

### 本章新增了什么能力

本章为综合案例新增的是：

- 一个最小可运行的单 Agent 原型
- 一条从用户输入到模型输出的最短执行路径
- 一个后续继续叠加 Tool、Session、Memory 的基础承载点

### 这个能力属于哪一类

这一章新增能力属于：

- **仓库已有 sample 直接支持的部分**

因为 `01_hello_agent` 已经是仓库明确给出的第一步样例。

### 如果不加入这一章的能力，综合案例会卡在哪里

如果没有最小 Agent 基线，后面会直接卡住：

- 没法验证 Prompt 是否生效
- 没法验证 provider 与认证链路是否正确
- 没法验证输出格式、流式输出和最小交互闭环
- 后续 Tool 和 Workflow 会失去承载对象

## 源码与目录定位

本章重点对应以下仓库内容：

- 目录：`dotnet/samples/01-get-started/01_hello_agent/`
- 项目：`dotnet/samples/01-get-started/01_hello_agent/01_hello_agent.csproj`
- Sample：`dotnet/samples/01-get-started/01_hello_agent/Program.cs`
- 源码模块：`dotnet/src/Microsoft.Agents.AI/`
- 源码模块：`dotnet/src/Microsoft.Agents.AI.OpenAI/`
- 配置文件：`dotnet/global.json`
- 索引文档：`dotnet/samples/README.md`
- 样例设计说明：`dotnet/samples/AGENTS.md`
- 关键类型：`AIAgent`
- 关键扩展点：`AsAIAgent(...)`
- 底层客户端来源：`AzureOpenAIClient` 与 `GetChatClient(...)`

### 推荐先看的顺序

这一章建议按下面顺序阅读：

1. `dotnet/samples/README.md`
2. `dotnet/samples/AGENTS.md`
3. `dotnet/samples/01-get-started/01_hello_agent/01_hello_agent.csproj`
4. `dotnet/samples/01-get-started/01_hello_agent/Program.cs`

原因很简单：

- `README` 告诉你它在整套 samples 中的位置
- `AGENTS.md` 告诉你为什么这是第一个样例，以及默认 provider 选择是什么
- `.csproj` 告诉你它依赖了哪些包和源码项目
- `Program.cs` 才是最终执行入口

### 这个 sample 对应了哪些真实依赖

从 `01_hello_agent.csproj` 可以看到：

- 目标框架是 `net10.0`
- 依赖了 `Azure.AI.OpenAI`
- 依赖了 `Azure.Identity`
- 依赖了 `Microsoft.Extensions.AI.OpenAI`
- 通过 `ProjectReference` 引用了 `dotnet/src/Microsoft.Agents.AI.OpenAI/Microsoft.Agents.AI.OpenAI.csproj`

这说明它不是一个脱离仓库源码的孤立示例，而是直接吃本地框架源码。

## 先跑通或先验证一个最小示例

### 示例目标

- 编译并理解第一个 .NET Agent sample
- 确认环境变量、认证方式、provider 选择和执行入口都正确
- 能区分“一次完整调用”和“一次流式调用”两种运行方式

### 运行前提

- 机器上存在与 `dotnet/global.json` 匹配的 .NET SDK，文件中指定版本为 `10.0.200`
- 已安装 Azure CLI，并已执行过 `az login`（样例设计说明推荐这一开发认证路径）
- 已配置环境变量：
  - `AZURE_OPENAI_ENDPOINT`
  - `AZURE_OPENAI_DEPLOYMENT_NAME`
- 你的 Azure OpenAI 资源中存在可用 deployment

### 操作步骤

1. 先确认 sample 项目可以被编译：

```powershell
dotnet build .\dotnet\samples\01-get-started\01_hello_agent\01_hello_agent.csproj
```

2. 再实际运行 sample：

```powershell
dotnet run --project .\dotnet\samples\01-get-started\01_hello_agent\01_hello_agent.csproj
```

3. 打开 `Program.cs`，确认它做了以下 4 件事：
   - 读取环境变量
   - 创建 `AzureOpenAIClient`
   - 通过 `.GetChatClient(...).AsAIAgent(...)` 生成 `AIAgent`
   - 依次调用 `RunAsync(...)` 和 `RunStreamingAsync(...)`

### 预期现象

如果配置正确，程序会输出两段结果：

- 第一段是一次完整调用的结果
- 第二段是流式调用逐步输出的更新内容

在当前样例里，提示词是：

- `Tell me a joke about a pirate.`

而 `instructions` 是：

- `You are good at telling jokes.`

所以预期输出会围绕“海盗笑话”展开。

### 成功标志

- 项目能够成功编译
- 程序可以创建 `AIAgent` 实例
- `RunAsync(...)` 能返回完整文本
- `RunStreamingAsync(...)` 能逐步输出更新
- 你能解释为什么是同一个 Agent 被调用了两次，而不是两套不同逻辑

### 我在仓库中的最小验证结果

我已经对这个 sample 做过一次最小验证：

```powershell
dotnet build .\dotnet\samples\01-get-started\01_hello_agent\01_hello_agent.csproj
```

观察结果：

- 构建成功
- 0 warning
- 0 error
- 本地成功编译到了 `bin\Debug\net10.0`

这说明：

- 项目引用链是通的
- 仓库源码与 sample 的工程关系是正常的
- 即使你暂时没有配置模型环境，也可以先完成“编译级验证”

## 核心概念拆解

### 1. `AIAgent` 是什么

`AIAgent` 是这一章最核心的对象。

在当前学习阶段，你可以先把它理解成：

- 一个统一的 Agent 抽象
- 它背后可以连接不同 provider 的底层客户端
- 它对外暴露统一的运行方式，例如 `RunAsync(...)` 和流式运行方式

它的价值不在于“帮你少写几行代码”，而在于：

- 把后续 Tool、Session、Memory、Workflow 的承载对象统一起来
- 让不同推理服务尽量使用一致的 Agent 入口

### 2. `AzureOpenAIClient` 在这里负责什么

`AzureOpenAIClient` 不是 Agent，本质上是 provider 侧客户端。

在当前 sample 里，它负责：

- 连接 Azure OpenAI endpoint
- 配合凭据完成认证
- 返回 chat client

也就是说：

- `AzureOpenAIClient` 解决“怎么连接模型服务”
- `AIAgent` 解决“怎么以 Agent 方式运行”

这两个层次不要混在一起。

### 3. `GetChatClient(...).AsAIAgent(...)` 为什么关键

这一串调用体现了本章最重要的抽象转换：

- 先拿到 chat client
- 再通过扩展方法把 chat client 包装成 `AIAgent`

这也是 `dotnet/samples/AGENTS.md` 明确建议优先理解的模式：

- `client.GetChatClient(deployment).AsAIAgent(...)`

这个模式的重要性在于：

- provider 接入层和 Agent 抽象层被明确分开
- 你以后换 provider、换配置、加 Tool，不必推翻整个调用方式

### 4. `RunAsync(...)` 与 `RunStreamingAsync(...)` 的边界

这两个方法都在“运行 Agent”，但适用场景不同：

#### `RunAsync(...)`
- 一次性拿到完整结果
- 适合命令行快速验证、简单问答、后处理逻辑
- 对新手最容易理解

#### `RunStreamingAsync(...)`
- 按更新流逐步返回结果
- 适合 UI、长输出、需要更快感知响应的场景
- 有助于你理解 Agent 输出不一定是“一次性字符串”

很多人第一次看 sample 时会误以为流式输出只是“另一个打印方法”。其实不是，它反映的是**执行结果的交付模式不同**。

### 5. `instructions` 与用户输入的关系

当前 sample 里有两个最小提示层：

- Agent 创建时的 `instructions`
- 调用时的用户请求字符串

这里先建立一个最小认知：

- `instructions` 更像这个 Agent 的长期行为设定
- 调用参数更像当前这一轮的具体任务

第 5 章会系统展开 Prompt 和消息结构，但在这一章你至少要先意识到：

- Agent 行为不是只由用户那一句话决定的
- `instructions` 改掉后，同一用户问题的输出风格可能明显变化

### 6. 常见误解是什么

#### 误解一：`AIAgent` 就是某个具体模型 SDK

不是。它是统一 Agent 入口，不等于具体 provider 的原始客户端。

#### 误解二：这个 sample 只是普通聊天 Demo

也不是。它的价值在于展示“如何把底层 chat 能力包装进 Agent 抽象”。后面所有能力都会加在这个抽象周围。

#### 误解三：只要能 `RunAsync` 就学完了

这只是基线。你现在学到的是“一个 Agent 可以被创建并运行”，而不是“一个完整 Agent 系统已经建立完成”。

## 结合源码理解执行过程

建议按“入口 -> 中间处理 -> 关键对象 -> 输出结果”的顺序理解。

### 1. 入口在哪里

入口文件就是：

- `dotnet/samples/01-get-started/01_hello_agent/Program.cs`

关键代码结构非常短，核心只有三段：

1. 读配置
2. 建 Agent
3. 跑调用

### 2. 调用是如何继续流转的

按 `Program.cs` 的顺序看，一次最小调用链大致如下：

1. `Environment.GetEnvironmentVariable(...)` 读取 `AZURE_OPENAI_ENDPOINT`
2. `Environment.GetEnvironmentVariable(...)` 读取 `AZURE_OPENAI_DEPLOYMENT_NAME`
3. `new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())` 创建 provider 客户端
4. `.GetChatClient(deploymentName)` 获取 chat client
5. `.AsAIAgent(instructions: ..., name: ...)` 转成 Agent 抽象
6. `agent.RunAsync("Tell me a joke about a pirate.")` 触发一次完整调用
7. `agent.RunStreamingAsync("Tell me a joke about a pirate.")` 再触发一次流式调用
8. 控制台打印返回结果或更新流

这条链要点很清楚：

- 环境变量负责配置
- provider client 负责接模型
- `AsAIAgent` 负责把底层客户端变成 Agent
- `RunAsync` / `RunStreamingAsync` 负责真正执行

### 3. 关键对象分别负责什么

#### `endpoint`
- 作用：定位 Azure OpenAI 服务地址
- 出问题会怎样：连不到服务，Agent 根本无法建立有效调用链

#### `deploymentName`
- 作用：指定要使用的模型 deployment
- 出问题会怎样：即使 endpoint 正确，也可能调用到不存在或错误的模型部署

#### `DefaultAzureCredential`
- 作用：为开发环境提供默认认证链
- 出问题会怎样：认证失败，或者认证顺序导致意外凭据被尝试

#### `AzureOpenAIClient`
- 作用：Azure OpenAI provider 客户端
- 出问题会怎样：无法获取 chat client，自然后续无法生成 Agent

#### `AIAgent agent`
- 作用：统一承载本轮 Agent 行为与后续能力扩展
- 出问题会怎样：后续 Tool、Session、Memory 都失去挂载点

### 4. 扩展点在哪里

这个最小 sample 的扩展点其实已经很明显：

- **改 `instructions`**：改变 Agent 行为设定
- **改请求内容**：改变当前任务
- **改 provider 或 credential**：改变底层推理服务接入方式
- **改调用方式**：完整输出 or 流式输出

后面几章会依次把更多扩展点接上来：

- 第 4 章：加 Tool
- 第 5 章：细化 Prompt、消息与交互结构
- 第 7 章：加 Session
- 第 8-10 章：走向 Workflow 与状态管理

## 关键代码或配置讲解

### 代码片段 1：环境变量读取
- 位置：`Program.cs` 第 10-11 行附近
- 作用：读取 endpoint 和 deployment 名称
- 为什么重要：这是整个 Agent 调用链最上游的配置入口
- 改动它会影响什么：服务地址错了、deployment 错了，后面全错

### 代码片段 2：`DefaultAzureCredential`
- 位置：`Program.cs` 第 16-18 行附近
- 作用：为 Azure OpenAI 调用提供默认认证链
- 为什么重要：没有认证，provider 客户端无法正常工作
- 改动它会影响什么：不同认证策略会影响本地开发体验、部署方式和生产安全边界

### 代码片段 3：`.AsAIAgent(instructions: ..., name: ...)`
- 位置：`Program.cs` 第 19-20 行附近
- 作用：把 chat client 包装成 `AIAgent`
- 为什么重要：它是“模型客户端 → Agent 抽象”的关键转换点
- 改动它会影响什么：Agent 的名字、行为说明和后续调用效果都会受到影响

### 代码片段 4：`RunAsync(...)`
- 位置：`Program.cs` 第 22-23 行附近
- 作用：执行一次完整响应调用
- 为什么重要：它是最容易验证 Agent 是否真正可用的入口
- 改动它会影响什么：输入内容变化会直接改变模型输出；后续第 4 章开始会逐步受 Tool 和上下文影响

### 代码片段 5：`RunStreamingAsync(...)`
- 位置：`Program.cs` 第 25-29 行附近
- 作用：逐步接收流式更新
- 为什么重要：它让你提前建立“结果可以按流交付”的认知
- 改动它会影响什么：输出体验、UI 集成方式和调用观察方式都会不同

## 做一个最小改造实验

### 实验目标

通过只改很少几行代码，观察 Agent 行为变化，确认你不是“照抄 sample”，而是真的理解了最小执行链路。

### 改造点

推荐做两个最低成本实验。

#### 实验 A：改 Agent 角色设定

把：

- `You are good at telling jokes.`

改成例如：

- `You are a concise IT support assistant who answers in Chinese.`

然后把用户输入改成：

- `My VPN is disconnected. Give me three troubleshooting steps.`

#### 实验 B：只保留一种输出模式

先注释掉 `RunStreamingAsync(...)`，只保留 `RunAsync(...)`。

然后再反过来：

- 注释掉 `RunAsync(...)`
- 只保留 `RunStreamingAsync(...)`

### 你应该观察什么

#### 实验 A 观察点
- 同一个最小 Agent，只改 `instructions` 和用户请求，输出风格会明显变化
- 这说明 Agent 行为至少同时受“长期设定”和“当前输入”两层影响
- 这也是后面 Prompt 设计的基础

#### 实验 B 观察点
- 完整输出和流式输出的交付体验不同
- 业务上并不是总要两者同时使用
- UI 类应用往往更关心流式输出，而后台处理更可能直接要完整结果

### 可能出现的问题

- 你改了 Prompt，但输出变化不明显：可能是提示改动太弱，或者模型回复本来就较稳定
- 你只保留流式输出后觉得“看起来更乱”：这是因为控制台逐条打印 update，不等于最终渲染逻辑必须这样
- 你把实验做复杂了：这一章只需要最小改动，不要提前引入 Tool、Session 或 Workflow

## 常见问题与排错

### 问题 1：`AZURE_OPENAI_ENDPOINT is not set.`
- 现象：程序启动即抛出 `InvalidOperationException`
- 原因：`Program.cs` 直接要求必须存在 `AZURE_OPENAI_ENDPOINT`
- 排查方法：检查当前终端会话和系统环境变量里是否已设置该值
- 修复方式：先正确设置环境变量，再重新运行

### 问题 2：deployment 名称错了或模型不可用
- 现象：程序能启动，但调用时报 provider 侧错误
- 原因：`AZURE_OPENAI_DEPLOYMENT_NAME` 配置了错误 deployment，或资源中没有该模型部署
- 排查方法：去 Azure 资源里核对 deployment 真实名称；不要只写模型家族名，要写实际 deployment 名称
- 修复方式：更新环境变量为真实可用 deployment 名称

### 问题 3：认证失败
- 现象：endpoint 和 deployment 看起来都对，但请求仍失败
- 原因：`DefaultAzureCredential` 没有找到有效凭据，或本地登录状态不可用
- 排查方法：先确认是否执行过 `az login`；再看是否使用了与你预期不同的凭据链来源
- 修复方式：在开发环境先用 Azure CLI 登录；到生产环境再换成更明确的凭据类型，例如托管身份

### 问题 4：编译通过，但运行前后出现 SDK 相关 stderr
- 现象：`dotnet build` 最终显示成功，但 stderr 中可能仍出现 SDK/nuget 解析类提示
- 原因：本地多 SDK/工具链环境下，某些子过程或探测过程可能输出额外诊断信息
- 排查方法：不要只盯 stderr 一行，要一起看：
  - 退出码
  - 最终是否“已成功生成”
  - warning/error 统计
- 修复方式：优先以最终构建结果为准；如果后续运行异常，再回头核对 `dotnet/global.json` 与本机 SDK 安装情况

### 问题 5：误把 sample 运行成功当成“系统已经设计好了”
- 现象：能输出一段文本后，就觉得框架已经掌握完了
- 原因：把最小运行验证误解成完整系统能力
- 排查方法：问自己是否已经理解 Tool、Session、Memory、Workflow 分别将加在什么位置
- 修复方式：把这一章定位为“Agent 基线建立”，不是“全栈能力完结”

## 企业级落地提醒

### 1. 稳定性：不要在生产里直接依赖 `DefaultAzureCredential` 的开发体验默认值

`dotnet/samples/AGENTS.md` 已明确提醒：

- `DefaultAzureCredential` 适合开发阶段
- 生产中需要更谨慎地选择明确凭据

原因是：

- 凭据探测链可能带来额外延迟
- 可能尝试到并非你预期的身份来源
- 权限边界不清时容易埋下安全问题

### 2. 可维护性：最小 sample 应尽快从 `Program.cs` 拆到可管理结构

教学 sample 把所有内容放在一个文件里没有问题，但真实项目里至少会拆出：

- 配置层
- provider 创建层
- Agent 构建层
- 业务请求处理层
- 日志与异常处理层

否则一旦往里加 Tool、Memory 和 Workflow，控制台代码会迅速膨胀。

### 3. 安全性：环境变量只是配置入口，不等于完整安全方案

即使这章只是最小 Agent，也要尽早建立边界意识：

- endpoint 和 deployment 不属于最敏感机密，但仍应通过规范配置管理
- 真正的凭据管理不能依赖随意散落的本地登录状态
- 后续加 Tool 时，风险会从“只读模型调用”扩大为“可能执行动作”

### 4. 成本控制：从第一天就要关注调用模式与输出方式

虽然这一章很简单，但已经能看到两个成本相关信号：

- 同一请求如果既跑 `RunAsync(...)` 又跑 `RunStreamingAsync(...)`，本质上是两次调用
- Prompt、上下文和输出模式都会影响 token 消耗与响应延迟

学习阶段可以这样做，但真实系统里要明确：

- 哪些场景只需要完整输出
- 哪些场景必须流式返回
- 哪些调用值得缓存、复用或降级

## 本章小结

- 本章最重要的结论 1：第一个 .NET Agent 的核心不是“打印一段文本”，而是建立 `provider client -> AIAgent -> Run` 这条最小执行链
- 本章最重要的结论 2：`01_hello_agent` 是后续 Tool、Prompt、Session、Memory、Workflow 的统一基线，不是孤立 demo
- 本章最重要的结论 3：最小 Agent 学习的重点是分清配置、认证、provider 接入、Agent 抽象和输出模式这几个层次

下一章之所以要学 Tool 与 Function Calling，是因为当最小 Agent 已经能稳定接收请求并输出结果后，下一步自然就是让它不只会“回答”，还会“调用函数去做事”。

## 本章建议产出物

- 一次 `01_hello_agent` 的编译或运行记录
- 一张“环境变量 → 客户端 → AIAgent → RunAsync / RunStreamingAsync”的最小调用链图
- 一次改 `instructions` 的实验记录
- 一份本章排错清单：endpoint、deployment、认证、SDK 版本

## 自查清单

- [x] 是否明确说明了本章解决什么问题
- [x] 是否明确标出了相关源码、目录或 sample
- [x] 是否至少解释了一个真实执行链路
- [x] 是否给了可执行或可验证步骤
- [x] 是否给了一个最小改造实验
- [x] 是否写了常见问题与排错
- [x] 是否补了企业级落地提醒
- [x] 是否避免脱离源码空讲概念
