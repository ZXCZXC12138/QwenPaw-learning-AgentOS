# Harness 产品思想与架构思想分析

## 一、岗位描述的核心命题

岗位描述揭示了一个关键判断：**当前阶段的核心约束不在模型能力，而在智能体的运行策略**。具体拆解为 5 个策略课题：

| 策略课题 | 岗位描述原文 | 对应 QwenPaw 架构层 |
|---------|------------|-------------------|
| 上下文工程 | 有限窗口内信息分层、筛选与压缩 | Agent Core (context/scroll) + Runtime (builder 中间件) |
| 记忆策略 | 长任务中记忆的写入、召回与失效 | Agent Core (memory/) + Loop (StopGate) |
| 工具编排 | 模型准确选择和使用的工具体系 | Harness (capabilities/resolver) + Modes (行为束) |
| 规划与反思 | 任务失败时收敛而非发散 | Loop (DoomLoopGate, RubricGate) |
| 评测驱动 | 评测和数据链路推动同步迭代 | 基础设施层 (observability/) + Governance |

---

## 二、Harness 层的产品思想：从"适配器"到"策略产品"

QwenPaw 的 harness 层表面上是一个**适配器模式**的工程实现，但从策略产品经理视角，它实际上承载了 3 个深层产品思想：

### 1. 能力投射（Capability Projection）——"让异构 Agent 共享同一套能力边界"

HarnessCapabilityResolver 不只是解析 MCP 和 Skills，它的产品本质是：

> **定义"一个 Agent 能做什么"的标准化描述语言**

- `HarnessCapabilities` 模型有 20+ 布尔字段（authentication、model_selection、reasoning_effort、tool_stream、commands、approval_presets...）
- 这实际上是一个**能力矩阵**——它回答了"这个 Provider 在哪些维度上可用"
- 产品意义：用户不需要理解 Codex 和 Qoder 的底层差异，只需要看到统一的能力描述

**策略产品启示**：能力矩阵是 Agent 产品化的核心抽象。它决定了：
- 用户能看到什么功能（前端渲染）
- 系统能调度什么资源（运行时决策）
- 评测能度量什么维度（质量评估）

### 2. 事件标准化（Event Normalization）——"所有 Agent 行为都可观测、可比较"

HarnessEvent 定义了 7 种标准事件类型（TEXT_DELTA、REASONING_DELTA、TOOL_STARTED/PROGRESS/COMPLETED、COMPLETED、ERROR、CANCELLED）。

> **产品本质：将不可比的异构 Agent 行为，转化为可比较的标准化行为流**

这意味着：
- 不同 Provider 的同一类行为（如工具调用）可以用同一套指标度量
- 评测系统可以跨 Provider 比较"工具选择准确率"、"任务完成率"
- 用户界面可以统一展示不同 Agent 的执行过程

**策略产品启示**：事件标准化是评测体系的前提。没有标准化事件模型，就无法建立跨 Agent 的评测基准。

### 3. 会话桥接（Session Bridging）——"第三方 Agent 的对话也是产品资产"

HarnessSessionBridge 将第三方 Agent 的对话"物化"到 QwenPaw 的 session 格式。

> **产品本质：对话数据是核心资产，不因 Agent 来源不同而割裂**

这意味着：
- 用户在 Codex 中的对话历史可以在 QwenPaw 中恢复
- 所有对话数据进入统一的存储和检索体系
- 记忆系统可以跨 Agent 来源工作

---

## 三、Harness 层的架构思想：从工程模式到策略框架

### 1. 适配器模式 → 策略隔离

```
HarnessAdapter ABC（统一契约）
    ├── CodexAdapter  → 适配 Codex CLI 协议
    ├── QoderAdapter  → 适配 Qoder 协议
    └── 未来: Claude Code Adapter → 适配 Claude 协议
```

**架构思想**：适配器模式在这里不只是工程解耦，更是**策略隔离**。每个 Adapter 封装了：
- 该 Provider 的认证策略（start_login / logout）
- 该 Provider 的模型选择策略（models）
- 该 Provider 的对话恢复策略（history）
- 该 Provider 的能力发现策略（discover_mcp / discover_skills）

**策略产品启示**：当你在设计"工具与 Skill 编排"策略时，应该先定义统一的策略接口（类似 HarnessAdapter ABC），再让每个具体场景实现。这样策略可以独立迭代，不互相污染。

### 2. 能力解析器 → 动态策略组合

HarnessCapabilityResolver 的工作流程：
```
workspace_dir/skills/ → 扫描技能目录 → 读取 SKILL.md
workspace MCP 配置 → 解析凭证引用 → 应用策略（工具白名单）
→ 构建 HarnessCapabilities 实例
```

**架构思想**：能力不是静态声明的，而是**从运行时数据动态解析**的。这意味着：
- 用户安装新 Skill → 能力矩阵自动更新
- 凭证变更 → 能力矩阵自动失效重建（通过 `_credential_revision()` 版本追踪）
- 策略变更（工具白名单）→ 能力矩阵自动反映

**策略产品启示**：记忆机制、工具编排、上下文策略都应该是"动态解析"而非"静态配置"。策略应该从运行时状态中涌现，而不是预先写死。

### 3. 事件翻译层 → 策略可观测性

TextStream + ToolStream 将 HarnessEvent 翻译为 QwenPaw 的 Message/TextContent/DataContent。

**架构思想**：翻译层是**策略可观测性的基础设施**。它确保：
- 每个文本增量可追踪（TEXT_DELTA → 有序 segment）
- 每个工具调用可追踪（TOOL_STARTED → TOOL_PROGRESS → TOOL_COMPLETED）
- 推理过程可追踪（REASONING_DELTA → 独立 reasoning segment）

**策略产品启示**：岗位描述中"从批量 badcase 出发完成归因"——这要求每个 Agent 行为都有完整的可观测链路。Harness 的事件模型就是这条链路的起点。

---

## 四、用 QwenPaw 架构映射岗位 5 大策略课题

| 岗位策略课题 | QwenPaw 对应机制 | 策略产品经理应关注的核心问题 |
|------------|-----------------|--------------------------|
| **上下文分层与预算分配** | Scroll 系统（context/）+ builder 中间件链（工具结果裁剪、视觉压缩） | 上下文窗口是有限资源，如何分层（system/user/tool/memory）、如何压缩（摘要/裁剪/视觉压缩）、如何分配预算（各层 token 占比） |
| **记忆机制设计** | memory/（embedding + proactive trigger）+ HarnessSessionBridge（跨 Agent 会话恢复） | 记忆的写入时机（何时存）、召回策略（何时取）、失效机制（何时忘）、冲突消解（矛盾记忆如何处理） |
| **工具与 Skill 编排** | HarnessCapabilityResolver + Modes（行为束）+ Governance（工具调用策略检查） | 工具发现（Agent 如何知道有哪些工具）、工具选择（Agent 如何选对工具）、工具白名单（安全边界）、Skill 与 MCP 的编排边界 |
| **任务规划与 Subagent 协作** | Loop Engineering（StopGate 规则引擎）+ Mission Mode（多步任务跟踪） | 规划范式选择（ReAct vs Plan-and-Execute）、失败时的收敛策略（DoomLoopGate 注入警告）、Subagent 协作边界 |
| **评测体系与后训练数据** | HarnessEvent 标准化 + observability/（Langfuse）+ Governance（ALLOW/DENY/ASK 决策记录） | 评测指标设计（基于标准化事件）、badcase 归因链路（从事件流追溯到策略决策）、数据标准（什么数据进入后训练） |

---

## 五、核心结论

岗位描述中的 WorkBuddy 和 QwenPaw 的 harness 层共享同一个底层认知：

> **Agent 产品的核心竞争力不在模型，而在"运行策略"——即如何在有限资源（上下文窗口、token 预算、工具集）约束下，通过策略设计让模型发挥最大效能。**

Harness 层的架构思想可以提炼为三个原则：

1. **标准化优先**：先定义统一的事件模型和能力描述语言，再实现具体适配。这为评测、可观测、跨 Agent 比较奠定基础。
2. **动态解析优于静态配置**：能力、策略、记忆都从运行时状态动态涌现，而非预先写死。
3. **策略隔离与可组合**：通过适配器模式隔离异构差异，通过能力矩阵描述统一接口，使策略可以独立迭代和组合。

作为策略产品经理，核心工作就是**设计这些策略的"规则"和"边界"**——什么情况下写入记忆、什么情况下召回、什么工具对什么任务可用、上下文如何分层压缩——然后用评测数据验证策略效果，驱动迭代。
