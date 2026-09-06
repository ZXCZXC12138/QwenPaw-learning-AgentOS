# 策略产品案例：任务规划发散与 Subagent 协作失败问题的策略设计与迭代

## 案例背景

**场景**：用户使用 Mission Mode 让 Agent 完成一个复杂的多步骤任务——为一个 Web 项目添加用户认证功能。

**用户指令**：
> "帮我的项目加上用户认证。需要：1) 用户注册/登录 API；2) JWT token 生成和验证；3) 登录页面 UI；4) 路由守卫（未登录跳转登录页）；5) 写单元测试。项目用的是 FastAPI + React + PostgreSQL。"

这是一个典型的 5 步骤任务，涉及后端 API、前端 UI、数据库、测试多个领域。

---

### Badcase 1 — 规划发散（Doom Loop）

Agent 的 ReAct 循环行为：

```
Turn 1-3:  成功创建用户注册 API（backend/auth/register.py）✅
Turn 4-6:  成功创建用户登录 API（backend/auth/login.py）✅
Turn 7-8:  创建 JWT 工具（backend/auth/jwt.py）✅
Turn 9:    尝试创建登录页面 → 发现前端项目结构不熟悉
Turn 10:   调用 file_search 查找前端结构 → 发现是 React + Vite
Turn 11:   创建 login.tsx → 编译报错（缺少 react-router-dom 依赖）
Turn 12:   调用 run_shell "npm install react-router-dom" → 网络超时失败
Turn 13:   重试 "npm install react-router-dom" → 再次失败
Turn 14:   改用 "yarn add react-router-dom" → yarn 未安装
Turn 15:   改用 "pnpm add react-router-dom" → pnpm 未安装
Turn 16:   回到 "npm install react-router-dom" → 又失败（同 Turn 12）
Turn 17:   尝试手动下载 react-router-dom 源码 → 完全偏离任务
Turn 18:   尝试用 CDN 引入 → 与 Vite 项目不兼容
Turn 19:   再次尝试 npm install → 继续失败
Turn 20:   IterationGate 触发 TERMINATE（达到 40 次上限的一半）
```

**结果**：Agent 在"安装依赖"这一步卡了 11 轮，最终没有完成登录页面、路由守卫和单元测试。用户看到 Agent 反复尝试同一个命令，非常沮丧。

---

### Badcase 2 — Subagent 协作失败

用户使用 `delegate_external_agent` 工具将前端任务委托给子 Agent：

```
主 Agent: "我已经完成后端 API 了，现在需要创建登录页面。"
  → delegate_external_agent(task="创建 React 登录页面，使用 react-router-dom")
  
子 Agent 执行:
  Turn 1: 创建 login.tsx ✅
  Turn 2: 创建 auth_context.tsx ✅
  Turn 3: 修改 App.tsx 添加路由 ✅
  Turn 4: 运行测试 → 发现缺少依赖 → 尝试安装 → 失败
  Turn 5-8: 反复尝试安装依赖 → 全部失败
  子 Agent 返回: "登录页面代码已创建，但依赖安装失败，无法验证。"

主 Agent 收到子 Agent 结果:
  → 忽略"依赖安装失败"的警告
  → 继续创建路由守卫 → 路由守卫依赖 auth_context → auth_context 依赖未安装的包
  → 路由守卫代码也报错
  → 继续写单元测试 → 测试 import 失败
  → 全部失败，但 Agent 认为"任务已完成"
```

**结果**：子 Agent 的失败信号被主 Agent 忽略，后续步骤建立在失败的基础上，最终交付物全部不可用。

---

## 第一步：从 Badcase 归因

策略产品经理打开 Langfuse 追踪链路，分析两个 badcase：

### Badcase 1 归因

| 可能原因 | 验证方式 | 结论 |
|---------|---------|------|
| DoomLoopGate 未触发 | 检查 DoomLoopGate 的配置和触发记录 | DoomLoopGate 的 `similarity_threshold=1.0`（完全相同才触发），Turn 12/16/19 的命令虽然都是 `npm install` 但参数略有不同（超时错误 vs 网络错误），相似度 0.85 < 1.0 → **根因 1** |
| 无失败收敛策略 | 检查连续失败后的行为 | Agent 连续失败 5 次后没有改变策略，仍在尝试同类方案 → **根因 2** |
| 无任务降级机制 | 检查任务规划是否有 fallback | 5 步任务是线性规划，没有"如果步骤 N 失败则跳过/降级"的机制 → **根因 3** |

### Badcase 2 归因

| 可能原因 | 验证方式 | 结论 |
|---------|---------|------|
| Subagent 结果未结构化 | 检查 delegate_external_agent 的返回格式 | 返回自由文本 "登录页面代码已创建，但依赖安装失败"——主 Agent 无法程序化判断成功/失败 → **根因 1** |
| 无依赖链检查 | 检查后续步骤是否验证前序步骤的输出 | 路由守卫直接使用 auth_context 而不验证其可用性 → **根因 2** |
| 无任务状态追踪 | 检查 Mission Mode 的 state 管理 | Mission state 只记录"步骤完成"，不记录"步骤质量"（部分完成/有警告/有错误） → **根因 3** |

**根因总结**：
1. **失败检测太迟钝**：DoomLoopGate 阈值过高，连续失败无收敛策略
2. **规划太刚性**：线性任务规划无 fallback、无降级、无跳过机制
3. **Subagent 协作太松散**：结果无结构化、依赖链无验证、状态无质量维度

---

## 第二步：提出假设

> **假设**：如果建立三层收敛策略——(1) 多级失败检测（从单次失败到模式失败到任务失败）；(2) 弹性任务规划（支持 fallback、降级、跳过、并行）；(3) 结构化 Subagent 协作（结果 schema + 依赖验证 + 质量状态）——则复杂任务的完成率和用户满意度会显著提升，同时平均轮次显著降低。

---

## 第三步：设计策略规则

设计的任务规划与 Subagent 协作策略规则（对应 QwenPaw 的 `loop/gates/` + `modes/mission/` + `agents/tools/agent_management.py` + `runtime/builder.py`）：

```
任务规划与收敛策略 v1：

=== 失败检测与收敛策略（Failure Detection & Convergence）===

规则 F1：多级失败检测
  级别 1 — 单次失败（Single Failure）：
    - 工具调用返回错误 → 记录失败计数
    - 同一工具连续失败 2 次 → 触发级别 2

  级别 2 — 模式失败（Pattern Failure）：
    - 同一工具连续失败 3 次 → DoomLoopGate 触发 INTERRUPT_AND_CONTINUE
    - 注入收敛消息：
      "你已连续 3 次尝试 [npm install] 均失败。请：
       1. 分析失败原因（网络？权限？配置？）
       2. 尝试不同的解决方案（换源？离线安装？手动配置？）
       3. 如果仍无法解决，考虑跳过此步骤或告知用户"
    - 重置 DoomLoopGate 的相似度阈值：从 1.0 降至 0.7（更宽松的检测）

  级别 3 — 任务失败（Task Failure）：
    - 同一子任务连续失败 5 次 → 触发任务级收敛
    - StopGate 决策：INTERRUPT_AND_CONTINUE
    - 注入收敛消息：
      "子任务 [安装前端依赖] 已尝试 5 次均失败。建议：
       方案 A：跳过此步骤，继续后续任务（标注为'待手动处理'）
       方案 B：请求用户介入（告知具体错误信息）
       方案 C：尝试完全不同的方案（如使用 CDN 而非 npm）
       请选择一个方案继续。"

规则 F2：失败策略升级
  - 每次失败后，Agent 必须切换策略（不能重复同一方案）：
    · 第 1 次失败 → 重试（相同方案）
    · 第 2 次失败 → 变体（微调参数，如换源 --registry=https://...）
    · 第 3 次失败 → 替代方案（完全不同的方法，如 yarn/pnpm/手动）
    · 第 4 次失败 → 降级（降低目标，如跳过安装，手动创建依赖文件）
    · 第 5 次失败 → 上报（请求用户介入）
  - 策略升级由 `FailureConvergenceGate` 自动追踪和触发

规则 F3：反思检查点
  - 每完成一个子任务后，自动插入反思检查点：
    · 验证子任务输出是否可用（文件是否存在、代码是否可编译、测试是否通过）
    · 如果验证失败 → 不标记为"完成"，触发修复循环（最多 2 轮）
    · 2 轮修复仍失败 → 标记为"部分完成"，记录问题，继续后续任务

=== 弹性任务规划策略（Flexible Task Planning）===

规则 P1：任务依赖图（DAG）
  - 将线性任务列表转换为有向无环图（DAG）：
    ```
    用户认证任务 DAG：
    [注册API] ─┐
    [登录API] ─┼→ [JWT工具] ──┐
    [数据库模型]─┘              ├→ [登录页面] ──┐
                                │              ├→ [路由守卫] ──
                                └→ [单元测试] ──┘              └→ [集成测试]
    ```
  - 无依赖的步骤可并行执行（如注册 API 和数据库模型可同时进行）
  - 某步骤失败时，只阻塞其下游依赖步骤，不阻塞其他分支

规则 P2：任务降级与跳过
  - 每个步骤声明降级方案：
    · [登录页面] 失败 → 降级为"创建静态 HTML 登录页"（不依赖 npm 包）
    · [路由守卫] 失败 → 降级为"在 API 层添加认证中间件"（后端兜底）
    · [单元测试] 失败 → 降级为"创建测试骨架（todo 标记）"
  - 跳过条件：步骤失败且无降级方案 → 标记为"跳过"，记录原因

规则 P3：并行 Subagent 调度
  - 无依赖的步骤自动分配给 Subagent 并行执行：
    · 主 Agent 负责规划和协调
    · Subagent A：后端任务（注册 API + 登录 API + JWT）
    · Subagent B：前端任务（登录页面 + 路由守卫）
    · Subagent C：测试任务（单元测试）
  - 主 Agent 等待所有 Subagent 完成后汇总结果

=== 结构化 Subagent 协作策略（Structured Subagent Collaboration）===

规则 C1：Subagent 结果 Schema
  - delegate_external_agent 的返回必须是结构化 JSON：
    {
      "status": "success" | "partial" | "failed",
      "task": "创建 React 登录页面",
      "outputs": {
        "files_created": ["login.tsx", "auth_context.tsx"],
        "files_modified": ["App.tsx"],
        "tests_passed": 0,
        "tests_failed": 0,
        "tests_skipped": 3
      },
      "issues": [
        {
          "severity": "error" | "warning" | "info",
          "description": "依赖安装失败：npm install react-router-dom 超时",
          "impact": "login.tsx 无法编译验证",
          "suggestion": "尝试换源或手动创建依赖"
        }
      ],
      "dependencies_for_downstream": {
        "requires_packages": ["react-router-dom"],
        "requires_env_vars": [],
        "requires_files": ["auth_context.tsx"]
      }
    }

规则 C2：依赖链验证
  - 后续步骤开始前，自动验证前序步骤的 `dependencies_for_downstream`：
    · 如果前序步骤标记为 "partial" 或 "failed" → 警告主 Agent
    · 如果关键依赖缺失（如 required_packages 未安装）→ 阻止后续步骤
    · 主 Agent 必须先解决依赖问题才能继续

规则 C3：Subagent 失败传播
  - Subagent 返回 "failed" → 主 Agent 必须处理（不能忽略）：
    · 选项 A：重试 Subagent（最多 1 次）
    · 选项 B：主 Agent 亲自处理该子任务
    · 选项 C：跳过该子任务，标记原因
    · 选项 D：终止整个任务，告知用户
  - 强制选择：主 Agent 不能静默忽略 Subagent 的失败

规则 C4：Subagent 资源隔离
  - 每个 Subagent 有独立的 token 预算和迭代次数限制：
    · max_tokens_per_subagent: 50,000
    · max_iterations_per_subagent: 15
  - Subagent 超出预算 → 自动 TERMINATE，返回当前进度
  - 防止单个 Subagent 耗尽整个任务的资源

规则 C5：主 Agent 协调协议
  - 主 Agent 的协调职责：
    1. 任务分解 → 生成 DAG
    2. 依赖分析 → 确定并行/串行
    3. Subagent 调度 → 分配任务
    4. 结果汇总 → 合并所有 Subagent 输出
    5. 冲突解决 → 处理 Subagent 间的冲突（如文件冲突）
    6. 最终验证 → 确保整体任务完成
  - 主 Agent 不执行具体任务，只做协调（类似项目经理角色）
```

---

## 第四步：设计实验验证

设计了 A/B 实验，使用 100 个复杂多步骤任务 badcase 回测（50 个规划发散 + 50 个 Subagent 协作失败）：

| 维度 | 对照组（当前策略） | 实验组 v1 |
|------|-----------------|----------|
| 失败检测 | DoomLoopGate threshold=1.0 | 多级失败检测（单次→模式→任务） |
| 失败收敛 | 无 | 失败策略升级（重试→变体→替代→降级→上报） |
| 任务规划 | 线性列表 | DAG 依赖图 + 并行调度 |
| 任务降级 | 无 | 每步骤声明降级方案 |
| Subagent 结果 | 自由文本 | 结构化 JSON schema |
| 依赖验证 | 无 | 前序依赖自动验证 |
| 失败传播 | 可忽略 | 强制处理（4 个选项必选其一） |
| 资源隔离 | 无 | Subagent 独立预算 |

**评测指标**：

```python
metrics = {
    # 核心指标
    "complex_task_completion_rate": 复杂任务完成率（所有步骤成功或部分成功）,
    "avg_turns_per_task": 平均每个任务的总轮次,
    "doom_loop_incidence_rate": 死循环发生率,
    "subagent_collaboration_success_rate": Subagent 协作成功率,

    # 过程指标
    "failure_convergence_rate": 失败收敛率（失败后成功切换策略的比例）,
    "task_degradation_rate": 任务降级率（使用降级方案完成的比例）,
    "parallel_execution_rate": 并行执行率（可并行步骤实际并行的比例）,
    "dependency_validation_accuracy": 依赖验证准确率,

    # 负面指标
    "silent_failure_rate": 静默失败率（Subagent 失败被忽略的比例）,
    "resource_exhaustion_rate": 资源耗尽率（单个 Subagent 耗尽总预算的比例）,
    "redundant_retry_rate": 冗余重试率（同一方案重复尝试超过 2 次的比例）,

    # 用户体验指标
    "user_intervention_rate": 用户介入率（需要用户手动处理的比例）,
    "user_satisfaction_score": 用户满意度,
}
```

---

## 第五步：实验结果与迭代

**v1 实验结果**（100 个复杂任务 badcase 回测）：

| 指标 | 对照组 | 实验组 v1 | 变化 |
|------|-------|----------|------|
| 复杂任务完成率 | 28% | 74% | +46pp ✅ |
| 平均轮次/任务 | 35.2 | 18.6 | -16.6 ✅ |
| 死循环发生率 | 42% | 8% | -34pp ✅ |
| Subagent 协作成功率 | 35% | 82% | +47pp ✅ |
| 失败收敛率 | 12% | 78% | +66pp ✅ |
| 冗余重试率 | 55% | 6% | -49pp ✅ |
| 静默失败率 | 38% | 2% | -36pp ✅ |
| **新问题：过度降级** | N/A | **出现** | **❌** |

**归因 v1 的新问题**：规则 P2（任务降级）过于激进——Agent 在遇到第一个困难时就选择降级，而不是充分尝试。例如，`npm install` 失败 1 次后就直接降级为"创建静态 HTML"，跳过了换源、检查网络等简单解决方案。用户反馈："为什么这么快就放弃了？多试几次不行吗？"

---

## 第六步：策略迭代 v2

调整规则：

```
v2 改进：

失败收敛策略增强：
- F2 增加"充分尝试"要求：
  · 级别 1（重试）：必须尝试至少 2 次（不同参数/配置）才能升级到级别 2
  · 级别 2（变体）：必须尝试至少 2 种变体才能升级到级别 3
  · 级别 3（替代）：必须尝试至少 1 种替代方案才能升级到级别 4
  · 级别 4（降级）：必须获得用户确认才能降级（不能自动降级）
  · 级别 5（上报）：自动触发

- F1 增加"失败原因分类"：
  · 网络错误 → 建议换源/检查网络/离线方案
  · 权限错误 → 建议 sudo/修改权限/换目录
  · 配置错误 → 建议检查配置文件/版本兼容性
  · 依赖冲突 → 建议版本锁定/清除缓存/重建
  · 不同错误类型触发不同的收敛建议

弹性任务规划增强：
- P1 增加"关键路径分析"：
  · 识别 DAG 中的关键路径（最长依赖链）
  · 关键路径上的步骤优先分配资源
  · 非关键路径的步骤可以延迟或降级

- P2 增加"降级审批"：
  · 降级必须经过主 Agent 的"降级审批"流程
  · 审批条件：已尝试 ≥ 3 种方案 + 每种方案失败原因不同
  · 降级后在最终报告中标注"此步骤使用了降级方案"

Subagent 协作增强：
- C3 增加"失败恢复协议"：
  · Subagent 返回 "failed" → 主 Agent 先分析失败原因
  · 如果失败原因是"可修复的"（如依赖缺失）→ 主 Agent 修复后重试
  · 如果失败原因是"不可修复的"（如需求不明确）→ 上报用户
  · 重试最多 1 次，避免无限重试

- C5 增加"协调开销控制"：
  · 主 Agent 的协调轮次上限：总轮次的 20%
  · 如果协调开销过高 → 减少 Subagent 数量，改为串行执行
  · 小任务（≤ 3 步骤）不使用 Subagent，主 Agent 直接执行
```

**v2 实验结果**：

| 指标 | 对照组 | v1 | v2 |
|------|-------|-----|-----|
| 复杂任务完成率 | 28% | 74% | **86%** |
| 平均轮次/任务 | 35.2 | 18.6 | **14.2** |
| 死循环发生率 | 42% | 8% | **4%** |
| Subagent 协作成功率 | 35% | 82% | **91%** |
| 失败收敛率 | 12% | 78% | **89%** |
| 冗余重试率 | 55% | 6% | **3%** |
| 静默失败率 | 38% | 2% | **1%** |
| 过度降级率 | N/A | 25% | **5%** |
| 用户介入率 | 60% | 35% | **18%** |
| 并行执行率 | 0% | 45% | **62%** |
| 协调开销占比 | N/A | 28% | **12%** |

---

## 第七步：策略产品化

验证有效后，将策略规则产品化：

1. **写入 Loop Engineering 层 (loop/gates/)**：
   - 新增 `FailureConvergenceGate`：多级失败检测 + 策略升级追踪
   - 修改 `doom_loop.py`：降低默认相似度阈值（1.0 → 0.7），增加失败原因分类
   - 新增 `TaskDegradationGate`：降级审批流程（检查是否满足降级条件）

2. **写入 Modes 层 (modes/mission/)**：
   - 修改 `state.py`：任务状态增加质量维度（success/partial/failed/degraded/skipped）
   - 新增 `task_dag.py`：任务依赖图构建和关键路径分析
   - 新增 `degradation_registry.py`：每步骤的降级方案注册表
   - 修改 `handler.py`：支持 DAG 调度和并行 Subagent 分配

3. **写入 Agent Core (agents/tools/agent_management.py)**：
   - 修改 `delegate_external_agent`：返回结构化 JSON schema
   - 新增 `subagent_result_validator.py`：Subagent 结果验证器
   - 新增 `dependency_chain_checker.py`：依赖链自动验证

4. **写入 Runtime 层 (runtime/builder.py)**：
   - 新增 Subagent 资源隔离：每个 Subagent 独立的 token/iteration 预算
   - 修改中间件链：Subagent 输出自动压缩（防止污染主 Agent 上下文）

5. **写入评测体系**：
   - 将"复杂任务完成率"和"死循环发生率"纳入常规回归评测集
   - 建立任务执行 dashboard：DAG 可视化、失败收敛路径、Subagent 协作状态

6. **写入用户产品**：
   - 前端新增"任务进度面板"：DAG 可视化 + 每步骤状态（成功/部分/失败/降级/跳过）
   - 新增 `/task` 命令：查看和管理当前任务（暂停/跳过/降级/重试）
   - 降级时弹出用户确认对话框（透明度 + 可控感）

---

## 案例总结：任务规划与收敛的核心洞察

| 洞察 | 说明 |
|------|------|
| **失败检测需要多级** | 单次失败可能是偶发，模式失败才是真正问题，任务失败需要升级处理 |
| **收敛 ≠ 放弃** | 收敛是策略升级（重试→变体→替代→降级→上报），不是直接放弃 |
| **降级需要审批** | 自动降级导致过度降级，必须设置"充分尝试"门槛 + 用户确认 |
| **DAG 优于线性** | 线性规划一步卡住全盘阻塞，DAG 允许并行 + 局部失败不影响其他分支 |
| **Subagent 结果必须结构化** | 自由文本返回让主 Agent 无法程序化判断，结构化 schema 是协作基础 |
| **失败不可静默** | Subagent 失败必须强制处理，静默忽略是协作失败的最大来源 |
| **协调本身有成本** | Subagent 不是越多越好，小任务串行更高效，协调开销需要控制 |
| **反思检查点是质量保障** | 每步骤完成后的自动验证防止"假完成"（代码写了但不能用） |

---

## 与 QwenPaw 架构的映射

| 策略规则 | QwenPaw 对应层 | 对应文件/机制 |
|---------|--------------|-------------|
| 多级失败检测 | Loop Engineering | 新增 FailureConvergenceGate（gates/） |
| DoomLoopGate 阈值优化 | Loop Engineering | 修改 doom_loop.py（similarity_threshold） |
| 失败策略升级 | Loop Engineering | FailureConvergenceGate 内部状态机 |
| 反思检查点 | Loop Engineering + Modes | 新增 VerificationGate + mission/state.py |
| 任务 DAG | Modes (mission) | 新增 task_dag.py |
| 任务降级与跳过 | Modes (mission) | 新增 degradation_registry.py |
| 并行 Subagent 调度 | Modes (mission) + Agent Core | 修改 handler.py + agent_management.py |
| Subagent 结果 Schema | Agent Core (tools) | 修改 delegate_external_agent 返回格式 |
| 依赖链验证 | Agent Core (tools) | 新增 dependency_chain_checker.py |
| Subagent 失败传播 | Modes (mission) | 修改 handler.py 强制处理逻辑 |
| Subagent 资源隔离 | Runtime (builder) | Subagent 独立 token/iteration 预算 |
| 协调开销控制 | Modes (mission) | 主 Agent 协调轮次上限 |
| 任务进度面板 | 前端 Console | 新增 TaskProgressPanel（DAG 可视化） |
| /task 命令 | Runtime (commands/) | 新增 task_handler |

---

## 任务规划与收敛的策略框架总结

```
任务规划与收敛 = 失败检测 + 弹性规划 + 结构化协作 + 反思验证

失败检测回答：什么时候该收敛？
  → 多级检测（单次→模式→任务）、策略升级、失败原因分类

弹性规划回答：失败后怎么办？
  → DAG 依赖图、降级方案、跳过机制、并行调度

结构化协作回答：Subagent 如何可靠协作？
  → 结果 Schema、依赖验证、失败传播、资源隔离

反思验证回答：如何确保"完成"是真的完成？
  → 检查点验证、质量状态、假完成检测、最终汇总
```
