---
layout: post
title: "DeepSeek Harness 与 AgentScope Java 深度对比：两条不同的 Agent 工程化路线"
date: 2026-08-21 00:10:00 +0800
comments: true
categories: ai
---

最近深度读了两个很有代表性的 Agent 项目：

- [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)
- [AgentScope Java](https://github.com/agentscope-ai/agentscope-java)

它们都在解决同一个问题：**怎样把“大模型 + 工具调用”变成一个能够长期运行、可以恢复、可扩展、可治理的 Agent 系统。**

但读完代码后，我发现它们其实不在同一条赛道上。

DeepSeek Harness 更像一套为 Coding Agent 产品准备的**可组合运行时内核**；AgentScope Java 更像一套面向 Java 企业应用的**Agent 框架、Harness 与服务平台**。

如果只看 README，容易得出“两个项目功能差不多，只是一个用 TypeScript、一个用 Java”的结论。真正进入源码后，会看到两套完全不同的架构取向：

- DeepSeek Harness 把重点放在运行时语义：插件生命周期、事件溯源、精确回放、工具执行流水线、能力替换
- AgentScope Java 把重点放在生产运行：多租户状态、分布式恢复、长期记忆、沙箱、渠道接入和统一控制面

先说我的结论：

> **如果你想造一个 Coding Agent 产品，或者深度定制 Agent Runtime，DeepSeek Harness 更值得研究；如果你想把 Agent 纳入现有 Java 企业系统，并进一步做成可运营的服务，AgentScope Java 路线更直接。**

<!-- more -->

## 一、先别急着比功能：它们不是同一层级的产品

在讨论这两个项目之前，先要回答一个基础问题：什么是 Harness？

一个最简单的 Agent 循环通常只有几步：

1. 把历史消息和工具描述发给模型
2. 模型决定回复，或者调用工具
3. 执行工具，把结果放回上下文
4. 继续循环，直到输出最终答案

这只是 ReAct Loop。真正进入生产后，系统还要处理很多模型并不关心、但工程上无法回避的问题：

- 任务跑到一半进程崩了，怎么恢复？
- 用户中途追加指令，应该插入哪一步？
- 多个工具能否并行，写操作如何避免竞争？
- 哪些命令可以直接运行，哪些必须审批？
- 上下文太长时，如何压缩但不破坏工具调用配对？
- 子 Agent 的状态、权限和工作区如何隔离？
- 同一个 Agent 如何同时服务多个用户和会话？
- 如何观察、审计和干预一个正在运行的任务？

Harness 就是包围在模型循环之外的这一整层工程系统。

从这个定义看，两边虽然都叫 Harness，但边界不同。

DeepSeek Harness 的产品边界从底层事件、工具和文件系统能力，一直延伸到 Web UI、CLI、SDK 和插件开发体系。运行中的 `dsh` 本身就是一棵插件树。

AgentScope Java 则更明确地分成三层：

- `agentscope-core` 提供模型、消息、工具、事件和 `ReActAgent`
- `agentscope-harness` 在 Core 之上叠加工作区、记忆、压缩、技能、子 Agent、沙箱等能力
- `agentscope-service` 再向上提供 Agent 注册、Session 管理、团队编排、Dashboard 和分布式控制面

所以，更准确的对比不是“框架 A 对框架 B”，而是：

> **一套高度可组合的 Agent 产品运行时，对比一套 Java Agent 框架及其生产服务平台。**

## 二、DeepSeek Harness：把整个 Agent Runtime 做成插件树

DeepSeek Harness 最鲜明的设计不是工具多，也不是 UI 完整，而是它真的贯彻了“一切皆插件”。

底层的 [Cordis](https://github.com/cordiverse/cordis) 提供共享 Context、服务注册、类型化事件和可撤销副作用。模型适配器、工具注册表、会话日志、Agent Loop，甚至持久化与沙箱策略，都是挂载在同一棵树上的插件。

这意味着系统里没有一个必须不断打补丁的“神圣内核”。一个插件注册服务、工具或监听器时，这些注册都会绑定到插件生命周期；插件卸载时，副作用也会被撤销。依赖服务消失，消费它的插件会一起卸载；服务恢复后，再重新加载。

这套设计带来三个很强的能力。

### 1）运行时能力可以真正替换

传统框架常见的“可插拔”，其实只是允许你传一个接口实现。DeepSeek Harness 的粒度更细：

- 模型是 provider
- 文件系统是 provider
- Shell、PTY、LSP 是 provider
- Session Persistence 是 provider
- Subagent 也是 provider
- 工具、提示词片段、审批和遥测都通过事件与服务组合

例如把本地文件系统和进程执行替换成 E2B provider，Shell、文件工具等消费者可以继续复用同一套上层语义。它强调的是“能力接缝”（capability seam），而不是在每个工具内部判断当前跑在本机、容器还是远端。

这很适合构建 Coding Agent：底层执行世界会变化，但上层 Agent Loop 不应该跟着分叉。

### 2）插件卸载和 HMR 是架构能力，不是开发便利

因为注册本身就是可撤销副作用，DeepSeek Harness 可以在运行时卸载、替换、重新挂载插件。配置文件变化时，Loader 会比较配置项，只重载发生变化的部分；无效的新配置不会摧毁当前可用的插件树。

这不只是“保存文件后少重启一次”。它说明这个系统从一开始就在处理一个困难问题：**动态变化的能力集合，如何不留下失效引用和脏状态。**

对于需要安装第三方插件、切换 Agent Profile、运行不同产品形态的系统，这个底座很有价值。

### 3）扩展行为主要通过事件进入，而不是修改主循环

DeepSeek Harness 把事件分成三个域：

- Session Event：需要持久化、重启后仍然存在的事实
- `agent/*` Event：正在运行的 Agent 状态与控制
- Capability Event：文件系统、工具、遥测等能力自己的策略扩展点

例如工具执行不是简单调用一个函数，而是经过：

`pre-execute → 单调守卫 → execute → post-execute → finalize → result`

审批、沙箱、超时、重试、指标、结果改写都可以挂在流水线上，不必让具体工具直接依赖这些策略。

这类设计的好处是扩展面非常清楚；代价则是概念多、学习曲线陡。你必须理解 Context、Service、Effect、Scope、Waterfall Event、Profile、Bundle 和 Patch，才能真正驾驭它。

## 三、DeepSeek Harness 最值得看的设计：事件日志才是真相

我认为 DeepSeek Harness 最有含金量的部分，不是插件，而是它对 Session 的定义。

它把会话建模为一份**只追加的事件日志**。`turn/start`、`step/start`、用户消息、模型流式 chunk、完整 assistant message、工具调用、工具结果和结束边界都会进入日志。

模型看到的上下文不是另一份独立状态，而是通过 `deriveMessages()` 从日志投影出来。

它甚至把原则写得非常直接：

> 模型可见即已记录。

这句话很重要。很多 Agent 系统同时维护三份相似但不完全一致的状态：

- 发给模型的上下文
- 展示给用户的聊天记录
- 用于恢复任务的持久化快照

一旦三者漂移，就会出现最难查的 Agent Bug：界面上看到了某条消息，但模型没看到；模型根据某段隐藏上下文做了决定，但重启恢复后这段上下文丢了。

DeepSeek Harness 试图从结构上消灭这种分叉。原始流式 chunk 保证 UI 可以精确回放；完整消息用于派生模型历史；压缩通过 surface replacement 表达；fork、resume、transcript 和 telemetry 都从同一事件流产生。

Crash Recovery 也遵循这个思路：如果载入时发现一个 `turn/start` 没有对应的 `turn/end`，系统不会粗暴截断已经持久化的长任务，而是补上一个 `interrupted` 结束事件，保留中断前已经发生的事实。

这种做法更像事件溯源系统，而不是传统聊天记录。

它的收益是：

- 可重放
- 可审计
- 可恢复
- 可 fork
- UI 与模型上下文更容易保持一致

但它也把兼容性压力集中到了事件协议上。当前 Session 格式版本仍是 `0`，官方明确表示没有兼容承诺；一旦事件含义变化，迁移就会成为非常严肃的问题。

## 四、工具执行：DeepSeek Harness 更像调度器，AgentScope 更像企业框架

工具调用是比较两个项目时最能体现设计差异的地方。

### DeepSeek Harness：每次调用动态分类

DeepSeek Harness 会为每一个待执行调用查询 `executionMode`：

- `parallel` 可以与相邻安全调用重叠
- `exclusive` 必须独占执行，并形成顺序屏障

Agent Loop 使用滚动并发池执行安全调用，同时保证结果仍按模型调用顺序写回。更关键的是，分类发生在调用级，而不只是工具级。理论上，同一个工具可以根据本次参数判断它是只读操作，还是需要独占的写操作。

它的工具流水线还把权限、单调守卫、沙箱、超时、重试、结果规范化和持久化观察分开。任何一层失败，都要被转成唯一、可记录的最终结果。

### AgentScope Java：默认并发，加安全标记

AgentScope Java 2.0 的 Toolkit 默认开启并行执行。工具可以通过 `concurrencySafe` 标记自己是否能安全并发；连续的安全工具会并发运行，不安全工具形成串行槽位，输出顺序仍然保留。

这已经比“整个批次要么全并行、要么全串行”细很多，也很符合 Java 工程团队的使用习惯。但它当前主要还是工具定义级标记，表达力不如 DeepSeek Harness 的调用级动态分类。

因此，如果你在做文件编辑、终端、数据库变更这类副作用复杂的 Coding Agent，DeepSeek Harness 的执行语义更精细；如果大多数工具本身就是业务 API，AgentScope 的模型更直接，也更容易让普通团队理解。

## 五、Code Mode：不是让 Agent 写业务代码，而是让它编排工具

DeepSeek Harness 还有一个很有辨识度的能力：Code Mode。

普通 Function Calling 会把每个工具的完整 JSON Schema 暴露给模型，模型每调用一次工具，结果又会进入对话上下文。工具一多，Schema 和中间结果会持续占用 Token。

Code Mode 则只向模型暴露一个 `run_code` 入口和一份自动生成的 TypeScript 或 Python SDK。模型在临时程序中调用：

```
const files = await tools.glob({ pattern: "**/*.java" })
const results = await Promise.all(
  files.slice(0, 10).map(file => tools.read({ path: file }))
)
return results.map(item => extractWhatMatters(item))
```

中间工具结果留在本次执行局部，只有 `return` 或 `console.log` 的内容回到模型上下文。

这带来的价值不是“模型会写代码”——Coding Agent 本来就会——而是：

- 用程序表达循环、分支和聚合
- 独立只读调用可以显式并发
- 大量中间结果不必全部进入对话
- 工具返回值保留结构化类型，而不是来回解析文本

它本质上是把一部分 Agent 编排从自然语言推理迁移到一次性的确定性程序中。

不过 Code Mode 也不是免费午餐。当前每次运行都是全新状态，不是持久 REPL；中间值不能从会话日志重建，而且执行局部没有统一字节上限。如果程序拿回超大对象，仍然可能造成 worker 内存压力。

## 六、AgentScope Java：Core、Harness、Service 三层一起解决生产问题

如果说 DeepSeek Harness 的关键词是“可组合”，AgentScope Java 的关键词就是“可运营”。

它的双层 Agent 结构很清楚。

`ReActAgent` 负责核心推理循环，把模型、Toolkit、权限、Middleware、事件和 `AgentState` 组合起来；`HarnessAgent` 是外层包装，通过固定顺序的 Middleware 和工具，增加工作区、长期记忆、上下文压缩、技能、子 Agent、沙箱和 Plan Mode。

这种设计没有追求“一切皆插件”的纯粹性，而是刻意保留一个容易使用的 Builder：

```
HarnessAgent agent = HarnessAgent.builder()
        .name("assistant")
        .model("dashscope:qwen-plus")
        .workspace(Paths.get(".agentscope/workspace"))
        .build();
```

对于 Java 团队，这种体验很熟悉：核心抽象稳定，基础设施通过 Builder 和 Spring Boot Starter 接入，扩展行为走 Middleware。

更重要的是，AgentScope Java 没有停在 SDK 层。仓库中的 `agentscope-service` 已经把 Agent 运行进一步拆成 Gateway、Control Plane、Dataplane 和 Scheduler，并提供 Dashboard、Managed Agents 与 Agent Teams。

因此它真正想解决的不是“如何写出一个聪明的 Agent”，而是：

> **一家公司里有很多 Agent、很多用户和很多运行副本时，怎样统一注册、恢复、观察、调度和治理。**

## 七、状态模型：一个强调重放，一个强调恢复与隔离

两边都重视持久化，但设计中心不同。

### DeepSeek Harness：从事件重建会话

DeepSeek Harness 以 append-only Session Event Log 为 Source of Truth。状态恢复、模型历史和 UI 展示尽量从事件投影得到。

这是偏运行时语义的选择：重点是“过去到底发生了什么”。

### AgentScope Java：从 RuntimeContext 找到 AgentState

AgentScope Java 把状态分成几层：

- `RuntimeContext`：当前调用的 `userId`、`sessionId` 和临时属性，不持久化
- `AgentState`：对话上下文、权限、Plan Mode 和工具状态，通过 `AgentStateStore` 跨调用恢复
- Session JSONL：长期保留的对话流水账
- Workspace Memory：每日只追加记忆和聚合后的 `MEMORY.md`
- Sandbox Snapshot：容器文件、安装依赖和运行环境的快照

同一个 Agent 实例可以服务多个用户和会话。系统通过 `(userId, sessionId)` 定位状态；同一 Session 的调用串行化，不同 Session 可以并行。再配合 Redis、MySQL、PostgreSQL、OSS 或 COS 等后端，实例本身可以保持无状态并水平扩容。

这是偏服务架构的选择：重点是“下一次请求由任何副本接手时，怎样继续运行”。

两种模型没有绝对高下。

- 要做可回放、可 fork、过程保真的 Agent 产品，事件溯源更有吸引力
- 要融入现有用户、租户、数据库和服务治理体系，状态存储模型更容易落地

## 八、沙箱：同样是隔离，关注点完全不同

DeepSeek Harness 的本地 Sandbox 更贴近开发者桌面和 Coding Agent。Linux 可使用 bubblewrap / Landlock，macOS 使用 Seatbelt，Windows 有 ACL restricted-token runner。它强调同一宿主执行世界里的文件写入边界，并且在无法落实策略时 fail closed，而不是悄悄降级为无限制执行。

远端执行则不被塞进同一个进程沙箱接口，而是通过替换文件系统与进程 provider，把整个执行世界一起移动。目前仓库也已经提供 E2B 的文件系统和进程实现。

AgentScope Java 的沙箱更偏服务部署。它把 Docker、Kubernetes、Daytona、E2B 和 AgentRun 视为同类后端，并进一步考虑：

- 按 User、Session、Agent 或 Global 隔离
- 容器消失后从快照恢复
- 快照放在本机、Redis、JDBC 或对象存储
- 多副本同时操作同一沙箱时使用分布式锁
- 对话状态和可执行环境一起跨请求续跑

所以，DeepSeek Harness 更关心“这条命令在当前执行世界里是否越界”；AgentScope Java 更关心“这个租户的执行环境能否隔离、持久并被另一台机器接手”。

## 九、多 Agent：一个先抽象 provider，一个先建设运营体系

DeepSeek Harness 把 Subagent 做成可替换 capability seam。不同 provider 可以在同一 Context 共存：进程内 spawn、基于历史 fork，以及对接 Codex、Claude Code、ACP 或 DSH SDK 的外部执行器。

它还区分一次性子 Agent 和可继续对话的子 Agent。后者拥有持久 Session、可冷恢复的 Activation、唯一 inbox 和明确的父子授权关系。实验性的 Agent Teams 则在其上增加 roster、task board 和 mailbox。

AgentScope Java 的多 Agent 更贴近业务使用：

- `agent_spawn` 创建同步或后台子任务
- `agent_send` 向已创建的子 Agent 继续发消息
- 超时的同步任务可以自动转为后台任务
- 后台结果通过消息总线反向通知父 Agent
- Agent Teams 提供成员、任务认领、任务板和邮箱
- Service 控制面负责跨实例注册、路由与观察

简单说，DeepSeek Harness 优先解决的是“如何把不同 Subagent 实现放到同一个抽象后面”；AgentScope Java 优先解决的是“如何让一组 Agent 在企业平台里持续协作”。

## 十、一张表看懂核心差异

| 维度 | DeepSeek Harness | AgentScope Java |
| --- | --- | --- |
| 核心定位 | 可组合 Agent Runtime 与产品底座 | Java Agent 框架、Harness 与服务平台 |
| 技术栈 | TypeScript / Node.js，含 Web、Native 与 Python SDK | Java 17 / Reactor / Maven / Spring Boot，Service 含 Go 与前端 |
| 核心抽象 | Cordis Context、Plugin、Service、Event、Effect、Scope | Agent、ReActAgent、HarnessAgent、Middleware、RuntimeContext、AgentState |
| 执行循环 | Turn / Step 边界明确，行为大量通过事件拦截 | ReAct 核心循环稳定，Harness 通过 Middleware 叠加能力 |
| 状态真相 | Append-only Session Event Log | AgentStateStore + Session Log + Workspace / Memory |
| 回放能力 | 强，原始 chunk、调用与结果都进入事件日志 | 更偏事件流展示和状态恢复，不以统一事件溯源为核心 |
| 工具并发 | 调用级 `parallel` / `exclusive` 分类与屏障 | Toolkit 批量并发，工具级 `concurrencySafe` 标记 |
| 工具扩展 | 多阶段 waterfall、守卫、审批、沙箱、结果规范化 | Toolkit、权限系统、Middleware、执行配置 |
| Code Mode | 原生支持 TypeScript / Python 工具编排 | 不是当前核心设计 |
| 沙箱 | 本机强制隔离 + E2B provider，可替换整个执行世界 | Docker / K8s / Daytona / E2B / AgentRun，快照与分布式锁完整 |
| 多 Agent | 多 provider、一次性与可继续子 Agent、实验性 Teams | 同步/后台子 Agent、持久任务、Teams 与控制面 |
| 企业集成 | 需要自行组合和建设 | Redis、MySQL、PostgreSQL、OSS、COS、Nacos、Spring Boot、Channel |
| 产品形态 | CLI、Web、Headless、SDK、插件 Profile | Core、Harness、Extensions、Service、Dashboard |
| 当前成熟度 | `0.1.0-rc.8`，Developer Preview | 2.0 已 GA，主干继续快速迭代 |

## 十一、两边各自最需要警惕什么

### DeepSeek Harness：架构漂亮，不等于现在就适合所有生产系统

它的风险主要有三个。

第一，项目仍处于 Developer Preview，当前版本是 `0.1.0-rc.8`，官方明确提示后续会有破坏性变化。Session 格式也是预发布的 v0，没有内置迁移链。

第二，Cordis 的抽象能力很强，但认知成本也高。插件树、Service 注入、Scope 隔离、Effect 回收、Waterfall Event、Bundle 和 Patch 共同工作时，调试难度不低。

第三，它更像一个完整产品底座，而不是一个轻量 SDK。如果团队只是想在已有业务里增加几个 Agent 能力，引入整套运行时未必划算。

### AgentScope Java：生产面完整，但核心类正在承受复杂度

AgentScope Java 的风险也很明显。

首先，`ReActAgent` 和 `HarnessAgent` 都已经是体量很大的中心类。大量能力虽然通过 Middleware、Toolkit 和 Builder 暴露，但编排与兼容逻辑仍集中在核心实现里。未来功能继续增加时，维护成本值得关注。

其次，它的“可扩展”更接近企业框架的扩展方式，而不是 DeepSeek Harness 那种运行时级可替换。你可以加 Middleware、工具和 Store，但很难像 Cordis 一样把整个 Agent Loop 当成普通插件换掉。

最后，AgentScope Service 已经覆盖控制面、托管运行时和 Teams，能力很诱人，但这也意味着更重的部署和运维成本。只有当组织真的存在多 Agent、多租户、多副本和统一治理需求时，这一层平台才有足够回报。

## 十二、到底怎么选

如果你的目标是下面这些，优先研究 DeepSeek Harness：

- 做自己的 Coding Agent、IDE Agent 或桌面 Agent
- 希望替换模型、文件系统、Shell、Subagent 和持久化实现
- 强调会话回放、fork、恢复和过程保真
- 需要插件市场、动态装卸或多种产品 Profile
- 想研究 Code Mode 和更精细的工具执行语义

如果你的目标是下面这些，优先考虑 AgentScope Java：

- 企业核心系统本来就在 Java / Spring Boot 生态
- 一个 Agent 需要服务大量用户和 Session
- 需要 Redis、数据库、对象存储与跨副本恢复
- 需要长期记忆、技能治理、Channel 和多租户隔离
- 希望进一步建设 Agent Dashboard、Managed Agents 或统一控制面

还有一种常见情况：团队只是验证一个小型 Agent 功能。

这种时候，两套方案都可能太重。直接使用简单的模型 SDK、少量工具和一个清楚的状态边界，往往更合适。**不要因为 Harness 很先进，就默认所有 Agent 都需要 Harness。**

## 十三、最后的判断：它们代表了 Agent 工程化的两种未来

DeepSeek Harness 展示的是一种 Runtime-first 路线：

> 把 Agent 的每一项能力都变成可组合、可替换、可回收的运行时部件，再用统一事件日志保证执行语义。

AgentScope Java 展示的是一种 Production-first 路线：

> 把 Agent 纳入成熟的应用服务体系，解决租户、状态、沙箱、存储、渠道、调度和控制面，再让企业真正运营它。

前者更像在设计 Agent 时代的“运行时内核”，后者更像在建设 Agent 时代的“应用服务器与控制平台”。

两边最终很可能会互相靠近：DeepSeek Harness 会继续补生产基础设施，AgentScope Java 也会继续细化运行时语义。但至少在当前阶段，选型边界非常清楚：

> **造 Agent Runtime 和产品，选 DeepSeek Harness；在企业系统里运行和治理 Agent，选 AgentScope Java。**

这也说明 Agent 开发已经进入了一个新阶段。竞争不再只是“谁的模型更聪明”或“谁的 Demo 工具更多”，而是：

**谁能把不确定的模型行为，装进一套可恢复、可解释、可扩展、可治理的软件系统。**

---

本文阅读基线为 2026 年 8 月 21 日，主要参考两个仓库当时的主干源码与文档：DeepSeek Harness `141eb6f`（`0.1.0-rc.8`）和 AgentScope Java `643905e`。关键资料包括 [DeepSeek Harness 架构](https://github.com/deepseek-ai/deepseek-harness/blob/141eb6fef83422698aef7a981029e843e8161534/docs/architecture.zh.md)、[Agent 生命周期](https://github.com/deepseek-ai/deepseek-harness/blob/141eb6fef83422698aef7a981029e843e8161534/docs/agent-lifecycle.zh.md)、[工具执行流水线](https://github.com/deepseek-ai/deepseek-harness/blob/141eb6fef83422698aef7a981029e843e8161534/docs/tool-execution-pipeline.zh.md)、[AgentScope Harness 架构](https://github.com/agentscope-ai/agentscope-java/blob/643905e6bfdaabae50264048f69d58a43628c64c/docs/v2/zh/docs/harness/architecture.md)、[AgentScope Agent](https://github.com/agentscope-ai/agentscope-java/blob/643905e6bfdaabae50264048f69d58a43628c64c/docs/v2/zh/docs/building-blocks/agent.md) 与 [AgentScope Service](https://github.com/agentscope-ai/agentscope-java/blob/643905e6bfdaabae50264048f69d58a43628c64c/agentscope-service/README_zh.md)。
