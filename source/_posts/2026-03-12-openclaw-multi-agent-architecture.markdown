---
layout: post
title: "OpenClaw多Agent方案"
date: 2026-03-12 01:50
comments: true
categories: ai
---

很多团队把 Agent 做到第二阶段都会遇到同一个问题：

- 单个 Agent 已经不够用，要拆“角色分工”。

- 一台机器放不下所有任务，要跨多台主机部署。

- 代理之间需要通信协作，但又不能互相污染上下文。

这篇文章基于 OpenClaw 官方文档，系统讲清四件事：**单机多 Agent 怎么搭**、**单主机多 OpenClaw 实例何时可用**、**多 OpenClaw 主机怎么协作**、**代理间如何通信才稳定可控**。

<!-- more -->

## 1. 先统一心智模型：OpenClaw 的多 Agent 不是“多人格提示词”

在 OpenClaw 里，一个 agent 本质是一个“完整隔离单元”，它拥有自己的：

- workspace（包括 AGENTS.md/SOUL.md/USER.md 与本地文件上下文）

- agentDir（认证与 agent 级状态）

- sessions（会话与路由状态）

所以“多 Agent”是工程级隔离，而不是同一上下文里改几段提示词。

---

## 2. 单机多 Agent：推荐的标准架构

先给结论：**一个 Gateway + 多个 agent + 显式 bindings 路由** 是官方推荐范式。

### 2.1 架构骨架

- 一台主机只跑一个 Gateway（官方架构约束）。

- Gateway 持有所有渠道连接（Telegram/WhatsApp/Slack/...）。

- 通过 `bindings` 把不同 channel/account/peer 的入站消息，确定性路由到不同 agent。

- 每个 agent 使用独立 workspace 和 agentDir，避免认证与会话冲突。

### 2.2 最小配置示例（单机多 Agent）

```
{
  "agents": {
    "list": [
      {
        "id": "assistant",
        "default": true,
        "workspace": "~/.openclaw/workspace-assistant",
        "agentDir": "~/.openclaw/agents/assistant/agent"
      },
      {
        "id": "research",
        "workspace": "~/.openclaw/workspace-research",
        "agentDir": "~/.openclaw/agents/research/agent"
      }
    ]
  },
  "bindings": [
    {
      "agentId": "research",
      "match": {
        "channel": "telegram",
        "peer": { "kind": "direct", "id": "tg:123456789" }
      }
    },
    {
      "agentId": "assistant",
      "match": { "channel": "telegram" }
    }
  ]
}
```

### 2.3 单机模式下的通信方式

同一 Gateway 内，agent 间协作优先走 session 工具：

- `sessions_send`：向另一个 session 发消息并等待结果（可超时控制）。

- `sessions_spawn`：派生隔离子代理执行长任务，完成后回传摘要。

这两类通信在同一 Gateway 内是“内生通信”，上下文边界更清晰、可追踪性更好。

### 2.4 单主机多 OpenClaw 实例：可做，但要明确边界

有些团队会在同一台机器上启动多个 OpenClaw 进程（而不是一个 Gateway 托管多个 agent）。这个方案不是主路径，但在“强隔离”诉求下可用。

适用场景：

- 你需要把不同业务线彻底隔离到不同进程生命周期。

- 你希望为不同实例配置不同端口、不同状态目录、不同系统服务策略。

必须满足的工程约束：

- 每个实例使用独立 `OPENCLAW_STATE_DIR`。

- 每个实例使用独立 Gateway 端口（避免 WS/HTTP 端口冲突）。

- 每个实例的 channel 账号登录状态独立管理，避免同账号并发占用。

- 每个实例独立 workspace/agentDir，避免会话与认证污染。

实践建议：如果目标只是“多人格/多角色协作”，优先选择**单 Gateway + 多 agent + bindings**。只有当你明确需要“进程级隔离”时，再采用单主机多实例。

---

## 3. 多 OpenClaw 主机：现实可用的 3 种协作拓扑

这里最容易误解：**OpenClaw 官方当前核心是“单 Gateway 控制面”**。跨主机协作不是默认内建成“跨 Gateway 透明 RPC”。

所以工程上要用“桥接层”把多主机连起来。

### 3.1 拓扑 A：渠道桥接（推荐起步）

- Host-A 的 agent 通过消息渠道（如 Telegram Bot/群）发任务。

- Host-B 作为另一个 OpenClaw 实例在同渠道接收并处理。

- 处理结果再通过渠道回发到 Host-A 或人工会话。

优点：部署快，稳定性高，几乎不用改 OpenClaw 内核。

### 3.2 拓扑 B：Webhook / Hook 事件桥接

- Host-A 将任务打包成结构化事件（JSON）投递给 Host-B 的接收端。

- Host-B 的 Hook/入口会话消费事件并触发 agent 执行。

- 回传可走 webhook 回调或消息渠道。

优点：便于与现有后端系统集成，适合平台化。

### 3.3 拓扑 C：外部任务总线桥接（企业场景）

- 两台 OpenClaw 通过外部队列/任务系统（如 Redis Streams、Kafka、SQS）解耦。

- OpenClaw 只做“智能执行器”，任务编排交给外部工作流层。

优点：吞吐与治理能力最好，但实施复杂度最高。

---

## 4. 代理间通信与协作：建议采用“分层协议”

### 4.1 协作协议（建议）

- 控制消息：任务编号、优先级、超时、重试策略。

- 业务消息：输入参数、上下文摘要、期望输出格式。

- 回执消息：状态（accepted/running/done/failed）、结果摘要、错误分类。

### 4.2 角色分工（建议）

- Orchestrator Agent：接任务、拆解、派发、汇总。

- Worker Agent：按契约执行单一子任务。

- Reviewer Agent：做一致性检查与质量门禁。

### 4.3 三条硬规则

- **幂等键**：每个任务都要有业务 id，重复投递可安全去重。

- **超时与降级**：`sessions_send`/外部桥接必须有 timeout 与 fallback。

- **上下文最小化**：跨代理只传“必要上下文 + 结构化结果”，不要整段历史硬转发。

### 4.4 协议补充：A2A（Google Agent2Agent）与 ANP（Agent Network Protocol）

在跨系统、跨组织的多代理协作里，可以把 OpenClaw 作为运行时，把 A2A/ANP 当作“外部互联协议层”。

- **A2A（Agent2Agent）**：Google 发起的开放互操作协议，强调任务生命周期、能力发现（Agent Card）、长任务状态同步，以及基于 HTTP / SSE / JSON-RPC 的标准化通信。

- **ANP（Agent Network Protocol）**：国内社区推动的开源协议，重点放在开放网络中的智能体身份与安全通信（例如 DID 身份、认证与加密通道）。

一个实用落地方式：

- OpenClaw 内部（同 Gateway 或同实例）优先用 `sessions_send` / `sessions_spawn`。

- OpenClaw 与外部 Agent 平台协作时，可通过桥接层接入 A2A。

- 在跨组织、需要身份自治与安全互信的场景，可评估引入 ANP 作为身份与通信补充层。

注意：协议选型要看你的治理边界——企业内统一平台优先 A2A 生态兼容，开放网络互联优先关注 ANP 的身份与安全模型成熟度。

---

## 5. 实操中的常见坑

- 把多个 agent 复用同一个 `agentDir`，导致认证/会话冲突。

- bindings 规则不够具体，出现“路由命中漂移”。

- 把跨主机协作当成同机 `sessions_send`，结果通信链路设计不完整。

- 没有幂等，重试时重复执行副作用（重复发消息/重复写库）。

- 没有统一状态模型，排障时看不到“任务卡在哪一跳”。

---

## 6. 给团队的落地建议（从 0 到 1）

- 第一步：先在单机把“多 Agent + bindings + sessions_spawn”跑顺。

- 第二步：引入一种跨主机桥接（先渠道桥接，再升级 webhook/队列）。

- 第三步：补齐治理（幂等、重试、审计、告警、权限边界）。

这样做的好处是：每一步都可验证，不会把复杂度一次性打满。

---

## 7. 总结

OpenClaw 的多 Agent 架构要点可以压缩成一句话：

> **同机协作用 Gateway 内生会话工具，跨机协作用外部桥接层；隔离、路由、幂等是稳定性的三根柱子。**

如果你正在做多代理系统，不要先追求“最智能”，先把“能稳定协作”做出来。

---

## 参考资料（权威来源）

- OpenClaw 官方文档：Gateway Architecture  
  https://docs.openclaw.ai/concepts/architecture

- OpenClaw 官方文档：Multi-Agent Routing  
  https://docs.openclaw.ai/concepts/multi-agent

- OpenClaw 官方文档：Session Tools（sessions_send / sessions_spawn）  
  https://docs.openclaw.ai/concepts/session-tool

- OpenClaw 官方文档：Sub-Agents  
  https://docs.openclaw.ai/tools/subagents

- OpenClaw 官方文档：Configuration Reference  
  https://docs.openclaw.ai/gateway/configuration-reference

- Google Developers Blog：Announcing the Agent2Agent Protocol (A2A)  
  https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/

- A2A 协议官方文档（Linux Foundation 托管）  
  https://a2a-protocol.org/latest/

- ANP 官方文档（中文）  
  https://www.agent-network-protocol.com/zh/guide/

- ANP 身份与加密通信层  
  https://agentnetworkprotocol.com/docs/concepts/identity/
