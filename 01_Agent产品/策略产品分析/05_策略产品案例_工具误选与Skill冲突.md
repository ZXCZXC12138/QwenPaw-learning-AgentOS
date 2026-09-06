# 策略产品案例：工具误选与 Skill 冲突问题的策略设计与迭代

## 案例背景

**场景**：用户在一个编码工作区中使用 Agent 进行项目开发。工作区配置了以下工具和能力：

- **内置工具**：`read_file`、`write_file`、`edit_file`、`run_shell`、`file_search`（grep/glob）、`web_search`、`browser`
- **MCP 工具**：`github_create_pr`（创建 PR）、`jira_update_ticket`（更新 Jira）、`slack_send_message`（发 Slack）
- **Skill**：`code-review`（代码审查技能，包含自定义 prompt + 调用 `read_file` + `run_shell` 执行 lint）、`deploy`（部署技能，调用 `run_shell` + `github_create_pr`）

**Badcase 1 — 工具误选**：
> 用户："帮我把 `src/utils.py` 里的 `parse_date` 函数改一下，让它支持 ISO 8601 格式。"

Agent 没有使用 `edit_file` 做精准编辑，而是调用了 `run_shell` 执行 `cat src/utils.py` 读取全文，然后用 `run_shell` 执行 `sed -i` 做替换。结果 `sed` 命令写错了正则，把文件中所有 `date` 相关的变量都破坏了，导致 3 个其他函数报错。

用户反馈："你为什么不用 edit_file？sed 把整个文件搞坏了！"

**Badcase 2 — Skill 冲突**：
> 用户："跑一下代码审查，然后把结果发到 Slack 上。"

Agent 同时激活了 `code-review` Skill 和 `slack_send_message` MCP 工具。但 `code-review` Skill 内部也会调用 `run_shell` 执行 lint，而 lint 过程中产生了大量输出（2000+ tokens）。这些输出占满了上下文窗口，导致 Agent 在调用 `slack_send_message` 时"忘记"了审查结果的具体内容，最终发到 Slack 的是一条空消息："代码审查已完成。"

用户反馈："审查结果呢？你发了个寂寞？"

---

## 第一步：从 Badcase 归因

策略产品经理打开 Langfuse 追踪链路，分析两个 badcase 的事件流：

### Badcase 1 事件流

```
Turn 1:
  用户消息: "帮我把 src/utils.py 里的 parse_date 函数改一下..."
  LLM 推理 → TOOL_CALL(run_shell, "cat src/utils.py")     ←  应该用 read_file
  TOOL_OUTPUT(文件全文, 1500 tokens)
  LLM 推理 → TOOL_CALL(run_shell, "sed -i 's/old/new/' src/utils.py")  ← ❌ 应该用 edit_file
  TOOL_OUTPUT("sed: 1 replacement made")
  用户反馈: "sed 把文件搞坏了！"
```

**归因分析**：

| 可能原因 | 验证方式 | 结论 |
|---------|---------|------|
| 工具描述不清 | 检查 `run_shell` 和 `edit_file` 的 tool schema 描述 | `run_shell` 描述为"执行任意 shell 命令"——太宽泛，LLM 认为它"万能" → **根因 1** |
| 工具排序偏差 | 检查工具列表在 prompt 中的排列顺序 | `run_shell` 排在工具列表第 1 位（按字母序），`edit_file` 排第 8 位 → LLM 倾向于先看到先选 → **根因 2** |
| 缺少工具选择引导 | 检查 system prompt 中是否有工具选择策略 | 无工具选择策略提示 → **根因 3** |

### Badcase 2 事件流

```
Turn 1:
  用户消息: "跑一下代码审查，然后把结果发到 Slack 上。"
  Skill 激活: code-review
  TOOL_CALL(run_shell, "eslint src/ --format json")
  TOOL_OUTPUT(2000+ tokens lint 结果)  ← 占满上下文
  LLM 推理 → TOOL_CALL(slack_send_message, message="代码审查已完成。")  ←  丢失了审查结果
  TOOL_OUTPUT("Message sent to #general")
```

**归因分析**：

| 可能原因 | 验证方式 | 结论 |
|---------|---------|------|
| Skill 输出未压缩 | 检查 code-review Skill 的输出处理 | Skill 输出的 2000 tokens lint 结果未经压缩直接进入上下文 → **根因 1** |
| Skill 与 MCP 工具无编排 | 检查 Skill 和 MCP 工具的协作机制 | Skill 和 MCP 工具独立执行，无结果传递机制 → **根因 2** |
| 工具白名单未限制 | 检查 governance 策略 | `run_shell` 在 Skill 内外都可用，无 Skill 级工具白名单 → **根因 3** |

---

## 第二步：提出假设

> **假设**：如果建立三层工具编排策略——(1) 工具描述优化 + 选择引导，降低误选率；(2) Skill 级工具白名单 + 输出压缩，防止 Skill 执行污染上下文；(3) 工具链编排（Skill → MCP 工具的结果传递），确保多步骤任务的信息不丢失——则工具选择准确率和 Skill 执行成功率会显著提升。

---

## 第三步：设计策略规则

设计的工具与 Skill 编排策略规则（对应 QwenPaw 的 `agents/tools/` + `agents/skill_system/` + `governance/` + `drivers/` + `modes/`）：

```
工具与 Skill 编排策略 v1：

=== 工具选择策略（Tool Selection）===

规则 S1：工具描述优化
  - 每个工具的 schema 描述必须包含：
    · 适用场景（"当你需要精准修改文件的某几行时使用"）
    · 不适用场景（"不要用于读取整个文件，请用 read_file"）
    · 与其他工具的区分（"与 run_shell 的区别：edit_file 更安全，支持 diff 预览"）
  - 工具描述总 token 数控制在 200 tokens 以内（避免描述本身占满窗口）

规则 S2：工具选择引导（写入 system prompt）
  - 在 system prompt 中注入工具选择策略段落：
    """
    工具选择原则：
    1. 文件读取 → 优先 read_file，不要用 run_shell cat
    2. 文件编辑 → 优先 edit_file（精准行编辑），不要用 run_shell sed
    3. 文件写入 → 优先 write_file，不要用 run_shell echo > 
    4. 文件搜索 → 优先 file_search（grep/glob），不要用 run_shell find
    5. run_shell 仅用于上述工具无法覆盖的场景（编译、安装、git 操作等）
    """

规则 S3：工具排序优化
  - 工具列表不按字母序排列，按"安全优先级"排列：
    · 安全工具（read_file, edit_file, write_file）→ 排在前面
    · 中等风险工具（file_search, web_search）→ 排在中间
    · 高风险工具（run_shell, browser）→ 排在后面
  - LLM 倾向于选择列表中靠前的工具（position bias）

=== Skill 编排策略（Skill Orchestration）===

规则 O1：Skill 级工具白名单
  - 每个 Skill 声明其允许使用的工具子集：
    code-review Skill:
      allowed_tools: [read_file, run_shell]  # 只能读文件和执行 lint
      blocked_tools: [write_file, edit_file]  # 不允许修改文件（只审查）
    deploy Skill:
      allowed_tools: [run_shell, github_create_pr]
      blocked_tools: [edit_file, write_file]  # 部署不应修改代码
  - Skill 激活时，governance 层自动应用白名单
  - Skill 内尝试调用白名单外的工具 → DENY + 提示"此工具不在 Skill 允许范围内"

规则 O2：Skill 输出压缩
  - Skill 执行产生的工具输出自动压缩：
    · 超过 500 tokens 的工具输出 → 自动摘要（保留关键信息：错误数、警告数、通过/失败状态）
    · 结构化数据（JSON lint 结果）→ 提取统计摘要（"12 errors, 3 warnings, 0 fatal"）
    · 压缩后的摘要进入上下文，原始输出存入 Skill 执行缓存
  - 用户或 Agent 可通过 recall 工具查看完整输出

规则 O3：Skill 结果结构化输出
  - 每个 Skill 必须声明输出 schema：
    code-review Skill output:
      {
        "status": "pass" | "fail" | "warning",
        "summary": "一句话摘要",
        "details": "关键发现（压缩后）",
        "raw_output_ref": "指向完整输出的引用 ID"
      }
  - Skill 执行完成后，结构化输出进入上下文（而非原始工具输出）

=== 工具链编排策略（Tool Chain Orchestration）===

规则 C1：Skill → 工具的结果传递
  - 当用户指令包含多步骤（"先做 A，然后做 B"）时：
    · 步骤 A 的结构化输出自动作为步骤 B 的输入上下文
    · 例：code-review 的 summary → 自动注入 slack_send_message 的 message 参数
  - 实现方式：Mission Mode 的 state 管理（modes/mission/state.py）
    · 每个步骤的输出写入 mission state
    · 下一步骤的 prompt 自动包含前序步骤的 state

规则 C2：工具调用依赖图
  - 当检测到工具间存在数据依赖时，自动编排执行顺序：
    · read_file → edit_file（先读后改）
    · code-review → slack_send_message（先审查后发送）
  - 依赖图由 LLM 推理 + 规则引擎共同决定：
    · 规则引擎：已知依赖（file_read → file_edit）
    · LLM 推理：新发现的依赖（Skill A 的输出是 Skill B 的输入）

规则 C3：工具调用预算
  - 每个 turn 的工具调用次数上限：
    · 安全工具：不限
    · 中等风险工具：最多 5 次/turn
    · 高风险工具（run_shell）：最多 3 次/turn
  - 超过预算 → StopGate 触发 INTERRUPT_AND_CONTINUE
  - 注入提醒："本 turn 已调用 run_shell 3 次，请确认是否继续"
```

---

## 第四步：设计实验验证

设计了 A/B 实验，使用 300 个工具相关 badcase 回测（150 个工具误选 + 150 个 Skill 冲突）：

| 维度 | 对照组（当前策略） | 实验组 v1 |
|------|-----------------|----------|
| 工具描述 | 简短功能描述 | 场景化描述（适用/不适用/区分） |
| 工具排序 | 字母序 | 安全优先级排序 |
| 选择引导 | 无 | system prompt 注入工具选择原则 |
| Skill 工具白名单 | 无（Skill 可使用所有工具） | Skill 级白名单 |
| Skill 输出压缩 | 无（原始输出进入上下文） | 自动摘要 + 结构化输出 |
| 工具链编排 | 无（步骤间无结果传递） | Mission state 传递 + 依赖图 |

**评测指标**：

```python
metrics = {
    # 核心指标
    "tool_selection_accuracy": 工具选择准确率（是否选了最合适的工具）,
    "skill_execution_success_rate": Skill 执行成功率,
    "multi_step_task_completion_rate": 多步骤任务完成率,

    # 过程指标
    "run_shell_overuse_rate": run_shell 过度使用率（应该用专用工具却用了 run_shell 的比例）,
    "skill_output_context_overhead": Skill 输出占上下文的 token 比例,
    "tool_call_budget_violation_rate": 工具调用预算违规率,

    # 安全指标
    "file_corruption_rate": 文件损坏率（因工具误选导致文件被破坏的比例）,
    "unauthorized_tool_call_rate": 未授权工具调用率（Skill 调用白名单外工具的比例）,

    # 用户体验指标
    "avg_turns_to_complete": 平均完成轮次,
    "user_satisfaction_score": 用户满意度,
}
```

---

## 第五步：实验结果与迭代

**v1 实验结果**（300 个 badcase 回测）：

| 指标 | 对照组 | 实验组 v1 | 变化 |
|------|-------|----------|------|
| 工具选择准确率 | 58% | 84% | +26pp ✅ |
| Skill 执行成功率 | 45% | 79% | +34pp ✅ |
| 多步骤任务完成率 | 52% | 81% | +29pp ✅ |
| run_shell 过度使用率 | 38% | 12% | -26pp ✅ |
| 文件损坏率 | 15% | 3% | -12pp ✅ |
| Skill 输出上下文占比 | 35% | 8% | -27pp ✅ |
| **新问题：工具选择僵化** | N/A | **出现** | **❌** |

**归因 v1 的新问题**：规则 S3（安全优先级排序）+ S2（选择引导）导致 LLM **过度规避** `run_shell`。在一个需要执行 `npm install` 的场景中，LLM 尝试用 `write_file` 手动创建 `package-lock.json`（因为 `run_shell` 排在最后），结果完全错误。工具选择引导变成了"工具选择限制"。

---

## 第六步：策略迭代 v2

调整规则：

```
v2 改进：

工具选择策略增强：
- S2 增加"例外声明"机制：
  · 工具选择引导中明确列出 run_shell 的正当使用场景：
    "run_shell 适用于：编译、安装依赖、git 操作、进程管理、批量文件操作"
  · 当 LLM 选择 run_shell 时，如果场景匹配正当使用场景 → 不扣分
  · 引入"工具选择置信度"评分：
    · 选择专用工具且场景匹配 → confidence = 1.0
    · 选择 run_shell 且场景匹配正当用途 → confidence = 0.8
    · 选择 run_shell 但场景有专用工具 → confidence = 0.3

Skill 编排策略增强：
- O1 增加"工具升级"机制：
  · Skill 白名单内的工具如果执行失败（如 read_file 文件不存在），
    允许 Skill 请求"工具升级"到 run_shell（需用户确认）
  · 避免 Skill 因工具限制而卡死

- O3 增加"结果摘要质量"评估：
  · 摘要必须包含：状态（pass/fail）、关键数字（错误数）、行动建议
  · 如果摘要质量不达标（缺少关键数字）→ 重新生成摘要

工具链编排策略增强：
- C1 增加"结果传递衰减"：
  · 步骤 A 的结果传递到步骤 B 时，只传递结构化摘要（不传递原始输出）
  · 如果步骤 B 需要更多细节 → 通过 recall 工具主动获取
  · 避免前序步骤的大量输出污染后续步骤的上下文
```

**v2 实验结果**：

| 指标 | 对照组 | v1 | v2 |
|------|-------|-----|-----|
| 工具选择准确率 | 58% | 84% | **91%** |
| Skill 执行成功率 | 45% | 79% | **88%** |
| 多步骤任务完成率 | 52% | 81% | **92%** |
| run_shell 过度使用率 | 38% | 12% | **8%** |
| run_shell 正当使用成功率 | N/A | 62% | **94%** |
| 文件损坏率 | 15% | 3% | **1%** |
| Skill 输出上下文占比 | 35% | 8% | **5%** |
| 未授权工具调用率 | 22% | 2% | **1%** |
| 平均完成轮次 | 9.5 | 7.2 | **5.8** |

---

## 第七步：策略产品化

验证有效后，将策略规则产品化：

1. **写入 Agent Core (tools/)**：
   - 修改所有内置工具的 schema 描述（场景化描述 + 适用/不适用 + 区分）
   - 修改 `tool_registry.py`：工具按安全优先级排序（而非字母序）
   - 新增 `tool_selection_guide.py`：工具选择引导注入 system prompt

2. **写入 Skill System (skill_system/)**：
   - 新增 `skill_manifest.py`：Skill 声明 `allowed_tools` / `blocked_tools` / `output_schema`
   - 修改 `runtime_cache.py`：Skill 输出自动压缩 + 结构化输出
   - 新增 `skill_output_compressor.py`：Skill 输出摘要生成器

3. **写入 Governance 层 (governance/)**：
   - 新增 `skill_tool_policy.py`：Skill 级工具白名单策略引擎
   - 修改 `policy.py`：新增 `SKILL_SCOPE` 决策维度（Skill 范围内的工具调用检查）

4. **写入 Loop Engineering 层**：
   - 新增 `ToolBudgetGate`（StopGate 的一种）：每 turn 工具调用次数检查
   - 新增 `ToolSelectionGate`：检查工具选择是否合理（基于置信度评分）

5. **写入 Modes 层 (modes/mission/)**：
   - 修改 `state.py`：支持步骤间结构化结果传递
   - 新增 `tool_chain_orchestrator.py`：工具调用依赖图自动编排

6. **写入评测体系**：
   - 将"工具选择准确率"和"Skill 执行成功率"纳入常规回归评测集
   - 建立工具使用 dashboard：各工具调用频次、误选率、预算违规率

7. **写入用户产品**：
   - 前端新增"工具面板"：用户可查看当前可用工具及其适用场景
   - 新增 `/tools` 命令：查看和管理工具配置
   - Skill 执行时显示"工具白名单"提示（透明度）

---

## 案例总结：工具与 Skill 编排的核心洞察

| 洞察 | 说明 |
|------|------|
| **工具描述是策略，不是文档** | 工具描述直接影响 LLM 的选择行为，场景化描述比功能描述有效 3 倍 |
| **工具排序是隐性策略** | position bias 真实存在，安全工具排前面能显著降低风险工具误选 |
| **引导 ≠ 限制** | v1 的教训：过度引导导致 LLM 僵化，必须同时声明"例外场景" |
| **Skill 需要"沙箱化"** | Skill 级工具白名单 + 输出压缩 = Skill 执行的隔离边界 |
| **多步骤任务需要"信息管道"** | 步骤间的结果传递不能依赖上下文窗口，需要结构化的 state 机制 |
| **工具调用需要预算** | 无限制的工具调用是死循环和上下文耗尽的主要来源 |
| **透明度建立信任** | 用户需要看到 Skill 的工具白名单和工具选择理由 |

---

## 与 QwenPaw 架构的映射

| 策略规则 | QwenPaw 对应层 | 对应文件/机制 |
|---------|--------------|-------------|
| 工具描述优化 | Agent Core (tools/) | 各工具文件的 schema 描述 |
| 工具选择引导 | Runtime (builder) | system prompt 注入（prompt_manager） |
| 工具安全优先级排序 | Agent Core (tools/) | tool_registry.py 排序逻辑 |
| Skill 工具白名单 | Governance + Skill System | 新增 skill_tool_policy.py |
| Skill 输出压缩 | Skill System | 新增 skill_output_compressor.py |
| Skill 结构化输出 | Skill System | skill_manifest.py output_schema |
| 工具链结果传递 | Modes (mission) | state.py + tool_chain_orchestrator |
| 工具调用依赖图 | Modes (mission) + Governance | 依赖图推理 + 规则引擎 |
| ToolBudgetGate | Loop Engineering | StopGate 规则引擎（gates/）新增 |
| ToolSelectionGate | Loop Engineering | StopGate 规则引擎（gates/）新增 |
| 工具面板 | 前端 Console | 新增 ToolPanel 组件 |
| /tools 命令 | Runtime (commands/) | 新增 tools_handler |

---

## 工具与 Skill 编排的策略框架总结

```
工具与 Skill 编排 = 选择策略 + 隔离策略 + 编排策略 + 预算策略

选择策略回答：LLM 如何选对工具？
  → 工具描述优化、选择引导、排序策略、置信度评分

隔离策略回答：Skill 执行的边界在哪里？
  → 工具白名单、输出压缩、结构化输出、沙箱化

编排策略回答：多个工具/Skill 如何协作？
  → 结果传递、依赖图、Mission state、信息管道

预算策略回答：工具调用的资源上限是什么？
  → 次数限制、token 预算、StopGate 检查、异常收敛
```
