# QwenPaw Runtime 八阶段内核与 Loop 编译 深度解析

> **来源**：基于 QwenPaw 代码仓 `src/qwenpaw/runtime/`（25 个 Python 文件，约 7900 行）与 `src/qwenpaw/loop/`（19 文件，约 2300 行）全量源码分析
> **分析范围**：`Phase` 八阶段枚举、`Runtime.run()` 编排主循环、`HookRegistry` 拓扑排序与三种返回语义、`AgentBuilder` 每请求装配、`AgentExecutor` 心跳执行器、取消路径的状态保全、`loop/` 停止门目录（GateCatalog）与声明式循环模式编译
> **本文档包含五部分内容**：
> 1. 八阶段生命周期：固定骨架 + 可插拔钩子
> 2. 钩子系统：拓扑排序与三种返回语义
> 3. 每请求装配：AgentBuilder 与 AgentExecutor
> 4. 取消与错误路径：被中断的一轮不许丢
> 5. Loop 编译：把"什么时候停"变成声明式配置

---

## 目录

### Part 1: 八阶段生命周期
- `Phase` 枚举与阶段职责
- `Runtime.run()` 主循环结构
- 固定步骤与钩子阶段的边界

### Part 2: 钩子系统
- HookAction 三种返回语义
- HookContext：显式字段 + 两个逃生舱
- `_topo_sort`：before/after 约束与快速失败

### Part 3: 每请求装配
- AgentBuilder：工具、提示词、模型三路注入
- AgentExecutor：心跳包装与 finished_at 补章

### Part 4: 取消与错误路径
- asyncio 再取消问题的精确应对
- cancel-save：部分响应注入与悬空工具调用闭合

### Part 5: Loop 编译
- StopGate 三决策模型
- GateCatalog：七种内置门与严格参数校验
- `compile_loop_mode`：原子编译与互斥组
- 设计哲学总结

---

# Part 1: 八阶段生命周期：固定骨架 + 可插拔钩子

## 1.1 `Phase` 枚举

`runtime/phases.py`（41 行）是整个内核的骨架定义，八个阶段点覆盖一次请求的完整生命周期：

```
PRE_DISPATCH      请求归一化，发生在 slash 分发之前
POST_DISPATCH     slash 分发结束且未命中
PRE_AGENT_BUILD   session.load 等构建前准备
POST_AGENT_BUILD  agent 构建完成；注入模式上下文
PRE_EXECUTE       bootstrap / 提示词刷新 / 环境栈压入
POST_RESPONSE     session.save / cron 触发写回
ON_ERROR          异常归一化、取消信封
FINALLY           幂等清理（关闭 mcp、复位 ContextVar）
```

模块注释划了一条关键边界：

> Phase points are fixed; the slash-command registry and `AgentBuilder.build` sit between phases as **fixed steps and are not themselves hooks**. These hooks are the runtime-orchestration layer, distinct from the `agentscope.middleware` middlewares that wrap a single agent's reply loop. **The two are orthogonal.**

三层可插拔体系各有其位：**运行时钩子**（编排层，本文主角）≠ **AgentScope middleware**（单代理回复循环层）≠ **渠道命令路由**。阶段点固定不可扩展，能扩展的只有阶段上的钩子——**骨架封闭、插槽开放**。

## 1.2 `Runtime.run()` 主循环

`runtime.py`（544 行）的 `run()` 是唯一的编排入口，结构是"8 阶段 + 3 个固定步骤"的交替：

```
[phase 1] PRE_DISPATCH
[fixed 1] slash 命令分发（命中 → skip_agent）
[phase 2] POST_DISPATCH（仅未命中时）
[phase 3] PRE_AGENT_BUILD
[fixed 2] AgentBuilder.build + 模式 on_turn_start
[phase 4] POST_AGENT_BUILD
[phase 5] PRE_EXECUTE
          _apply_context_injections（动态上下文合并）
[fixed 3] AgentExecutor.run（SSE 流式执行）
[phase 6] POST_RESPONSE
          Envelope.finalize
── except ──  CancelledError / ConfigurationException / BaseException
── finally ─ agent.close() → [phase 8] FINALLY
```

每个前置阶段都支持三种转向（见 Part 2）：`SHORT_CIRCUIT` 直接吐信封结束请求，`SKIP_AGENT` 跳过两个固定代理步骤但继续走钩子。注释明确：**两个固定步骤是唯一接触 agent 的代码**——其余一切功能都长在 `LifecycleHook` / `AgentMode` 里，经由每工作区的 `HookRegistry` 注册。这就是洋葱模型的内核切面。

# Part 2: 钩子系统：拓扑排序与三种返回语义

## 2.1 HookAction：三种返回语义

`hooks.py` 把钩子的返回语义收敛为三个枚举（模块 docstring 原文）：

| 语义 | 行为 |
|------|------|
| `CONTINUE` | 默认；进入下一个钩子/阶段 |
| `SHORT_CIRCUIT` | Runtime 发射 payload 并结束当前阶段；但 `ON_ERROR`（如有）与 `FINALLY` **仍然运行** |
| `SKIP_AGENT` | 只跳过两个固定步骤（build + execute）；所有钩子阶段照常按序执行 |

`HookResult.payload` 契约性地要求是 `Msg` 实例（短路时）——信封状态机用它发射完整 SSE 序列，**其他形状在信封层被拒绝**。`SKIP_AGENT` 是粘性的：阶段内后续钩子照跑，最终结果携带 `SKIP_AGENT` 交给 Runtime。注册表**不吞异常**——钩子抛错沿正常异常路径进入 `ON_ERROR` 链，注释强调"tests rely on this contract"。

## 2.2 HookContext：显式字段 + 两个逃生舱

`HookContext` 是贯穿八阶段的每请求上下文，设计上有明显的"反杂物袋"意图：

```python
# ── Identity ──          request / session_id / agent_id / root_*
# ── Containers ──        workspace / app_services（只读，钩子不得变更）
# ── Per-request state ── input_msgs / agent_config / session_state / agent / error
# ── Escape hatches ──
mode_state: dict[str, Any]   # 按模式名命名空间隔离（"coding"/"mission"…）
extras: dict[str, Any]       # 真正的临时字典，供钩子对传递（如 FINALLY 要 __exit__ 的句柄）
```

注释原文：稳定的长生命周期状态**以显式命名字段暴露，让 IDE 导航可用**；逃生舱只有两个且各有分工——`mode_state` 按模式名隔离私有状态，`extras` 只装短命的钩子对之间流量。还有一个主动注入面 `inject_context(content, priority, source)`：钩子收集动态上下文，运行时在 `PRE_EXECUTE` 后按优先级排序合并为一条系统提示消息插到输入头部——**注入有优先级、有来源标签、有统一合并点**。

## 2.3 `_topo_sort`：声明式排序约束 + 快速失败

钩子执行顺序不靠手工排号，而是三因素合成：

1. **`before` / `after` 元组**：钩子声明与具名钩子的相对位置（构建 DAG）
2. **`priority`**：数字越小越先（平局裁决）
3. **注册序**：最终平局裁决，保证跨运行确定性

实现细节一丝不苟：引用未注册名字的约束**只警告不失败**（no-op）；拓扑排序的就绪队列用二分插入按 `(priority, order_index)` 维持有序；**环检测抛 `HookCycleError` 在启动时炸响**——注释写明动机："misconfiguration fails fast at startup rather than silently misbehaving in production"。Cron 钩子必须早于 `SessionSaveHook`（priority 10/80 vs 90）这类跨插件的顺序约束，正是靠这套 `before/after` 机制表达的。

`HookRegistry` 每工作区一个（挂在 `Workspace.plugins.hook_registry`），拓扑序按阶段缓存、每次 `register` 失效；`merge()` 支持把工作区注册表与跨工作区注册表合并（左到右注册序）。

# Part 3: 每请求装配：AgentBuilder 与 AgentExecutor

## 3.1 AgentBuilder：代理不复用，每请求现装

`builder.py`（1290 行，runtime 最大文件）的模块定位：

> `AgentBuilder` fully constructs a `QwenPawAgent` for each request. It obtains tools from the per-workspace `QwenPawLocalWorkspace`, the system prompt from `PromptManager`, and the model from the factory, then injects all dependencies into the agent constructor.

**代理实例零缓存**：工具集、提示词、模型、依赖全部按本次请求的上下文（激活模式、生效技能、启用特性、子代理白名单）现场装配。`build_toolkit` 展示了装配流水线：

```
工作区工具（list_tools：agent_config × active_modes × active_skills × enabled_features）
  → extra_tools（经子代理过滤）
  → memory_tools（统一包上 PolicyGuardedTool 治理壳）
  → 最终一遍白名单过滤（工作区 + extras + memory 一网打尽）
  → 运行时技能加载（_bound_skill_loader_dirs 按语言变体解析 -en/-zh）
  → Toolkit
```

两个细节体现工程纪律：记忆工具**必须**经过 `PolicyGuardedTool` 包裹才能进工具箱（治理是强制的，不是可选的）；绑定技能目录解析时先找用户语言变体、落回 `-en`，缺 `SKILL.md` 的技能**只警告不注入**——残缺的技能不许静默生效。

## 3.2 AgentExecutor：心跳与时间戳补章

`executor.py`（116 行）驱动 `agent.reply_stream`，两个工程点：

**① 心跳包装。** 事件流套上 `_iter_with_heartbeat`：长静默期（典型如工具审批等待）发射保活信封而不是让连接断掉。这与 Cron 模块的 `CRON_KEEPALIVE_INTERVAL_SECONDS=60` 是同一类问题的两处解答——**凡是可能被上游判死的长连接，都用心跳续命**。

**② `finished_at` 补章**（`_maybe_stamp_finished_at`）。docstring 记录了 issue #6826：AgentScope 只在 `REPLY_END` 事件的 app-service 路径写 `finished_at`，而 QwenPaw 走自己的 `_save_to_context` 持久化——结果会话快照没有真实完成时间，长工具调用的轮次被严重低报。解法：执行器在 `REPLY_END` 事件时回填最后一条 assistant 消息的 `finished_at`。**Best-effort by design：失败只记日志，绝不影响 SSE 流**。

# Part 4: 取消与错误路径：被中断的一轮不许丢

`run()` 的异常分支是全文最见功力的部分。捕获 `CancelledError` / `KeyboardInterrupt` 后的注释直击 asyncio 的语义深坑：

> The Task's `_must_cancel` flag may still be True after catching CancelledError, causing the next await to raise CancelledError again.

取消后**下一个 `await` 会再次抛取消**——所以 `ON_ERROR` 钩子要包在二次 `try` 里（被跳过也能继续），而取消信封必须无条件发射：前端 SDK 需要 `{object:response, status:completed}` 事件才能退出加载态。

## 4.1 cancel-save 为什么是硬编码而非钩子

`_try_save_on_cancel` 的 docstring 自问自答了这个设计决策：

> **Why hardcoded instead of a hook?** This runs *outside* the try/except that wraps `hooks.run(Phase.ON_ERROR)`, so it executes even when re-cancellation skips all ON_ERROR hooks. The synchronous parts (inject + state_dict) complete before any `await`, and `asyncio.shield` protects the I/O — guarantees that a generic hook framework cannot provide.

三步保全：

1. **部分响应注入**：`Envelope.collect_partial_blocks` 收集被中断迭代的流式文本/思考块，去重后写入代理上下文——中断前吐出的半句话不丢
2. **悬空工具调用闭合**：AgentScope 的 `_close_unfinished_tool_calls` 在生成器 `finally` 里打补丁，但其 `yield` 在生成器被销毁时触发 `RuntimeError`——QwenPaw 复制了纯变更逻辑（无 yield），给每个没有结果的 `ToolCallBlock` 补上 `ToolResultBlock(state=INTERRUPTED)`，输出附带 `<system-reminder>The tool call has been interrupted by the user.</system-reminder>`
3. **shield 落盘**：`state_dict()` 同步快照在任何 `await` 前完成，I/O 包在 `asyncio.shield` 里——外层再取消，内层保存也在后台跑完；`StateProxy` 持有数据独立副本，`finally` 里的 `agent.close()` 污染不到它

更难得的是 TODO 注释把局限写得明明白白：目前取消路径只有 `SessionSaveHook` 有等价物，其他 `POST_RESPONSE` 钩子（如 `CronMemoryRestoreHook`）在 /stop 时被跳过；未来应统一为专门的 `ON_CANCEL` 阶段。**当前能做到哪、欠了什么、打算怎么还，全在注释里**。

# Part 5: Loop 编译：把"什么时候停"变成声明式配置

## 5.1 StopGate 三决策模型

`loop/gates/base.py` 定义了代理循环的停止裁决抽象。每次评估，门返回三种决策之一：

```
BYPASS                  门没有意见，跳过
INTERRUPT_AND_CONTINUE  打断当前模式，注入一段提示，循环继续
TERMINATE               立即结束代理循环
```

`INTERRUPT_AND_CONTINUE` 是精髓——停止条件触发不一定是死刑：门可以调用自己的 `build_continuation()` 生成一段"换个策略继续"的用户轮消息注入循环（如 doom-loop 门检测到重复后要求改道）。有状态门实现 `reset_turn()` / `reset_session()` 两级重置；`StopHandler` 遍历时做故障隔离（单个门 reset 抛错只警告，不拖垮整批）。

`runner.py` 的 `_filter_by_scope` 解决多模式共存：非 `default` 作用域的门一旦激活，**该作用域独占**，default 门让位；无作用域（`scope=""`）的门永远运行——模式特化的停止纪律优先于全局默认。

## 5.2 GateCatalog：七种内置门 + 严格参数模型

`catalog.py`（293 行）是"用户可编辑循环模式"的门目录，每种门一个目录条目：

| 门类型 | 类别 | 成本 | 说明 |
|--------|------|------|------|
| `iteration` | limits | none | 固定迭代数上限（1–500） |
| `doom_loop` | safety | none | 重复工具调用检测，窗口 2–20 + 相似度阈值 |
| `token_budget` | limits | none | total/prompt/completion 三项至少配一项 |
| `timeout` | limits | none | 仅在循环边界检查的墙钟上限 |
| `tool_call_budget` | limits | none | 全局 + 按工具的调用预算 |
| `qualitative_rubric` | quality | none | 自然语言质量评估（互斥组 `completion_rubric`） |
| `completion_rubric` | quality | **model_call** | 完成信号检测（同互斥组） |

三个设计点：

- **参数模型全部 `extra="forbid"`**：未知参数直接拒绝；`model_validator` 做跨字段校验（如 `token_budget` 至少一个限额、`completion_rubric` 信号不得含换行）。用户配置在编译前就过完严格校验。
- **互斥组**：`qualitative_rubric` 与 `completion_rubric` 同属 `completion_rubric` 组——两种完成判定不能同时启用，`validate_exclusive_groups` 在编译期拒绝。
- **成本标注**：目录条目带 `cost: none | model_call`，`describe()` 把 JSON Schema 与成本一起暴露给前端——**要花模型钱的门，配置界面上说得清清楚楚**。

## 5.3 `compile_loop_mode`：原子编译

`compiler.py`（45 行）把声明式的 `CustomLoopModeConfig` 编译成可执行的 `StopHandler`：

```python
def compile_loop_mode(config, catalog=None) -> StopHandler:
    # 1. 先校验全部门参数（一个坏，整体不编译）
    for gate in config.gates:
        gate_catalog.validate_params(gate.type, gate.params)
    # 2. 仅对启用的门校验互斥组
    gate_catalog.validate_exclusive_groups([g.type for g in enabled])
    # 3. 实例化 + order = index * 10（预留插入空隙）
    # 4. StopHandler().replace(configured)  ← 原子替换
```

编译是原子的：全部参数校验通过才实例化任何一个门，`replace()` 一次性换掉整组门——**不存在"编译到一半"的半成品模式**。`order = index * 10` 的间距设计为后续插入门留空隙，是配置友好性的小心思。门目录本身是进程级不可变单例（`_CATALOG = GateCatalog(_entries())`）——可扩展性的入口在配置层（声明哪些门、什么参数），而不是运行时改目录。

## 5.4 设计哲学总结

1. **骨架封闭、插槽开放**：八个阶段点与三个固定步骤写死在内核里；一切可变行为长在钩子、模式、门里。扩展永远不触碰主循环。
2. **顺序是声明出来的**：钩子用 `before/after` 表达相对顺序，拓扑排序 + 环检测在启动时快速失败——跨插件的顺序契约（如 Cron 钩子早于会话保存）可表达、可校验。
3. **可插拔不意味可失控**：钩子异常不被注册表吞掉、短路不豁免 `FINALLY`、记忆工具强制过治理壳——每个插槽旁边都有不可拆除的护栏。
4. **每请求现装，零缓存代理**：工具集/提示词/模型按请求上下文现场合成，配置变更无需失效缓存。
5. **被中断的轮次不许丢**：部分响应注入、悬空工具调用闭合、`asyncio.shield` 落盘，对 asyncio 再取消语义逐字应对；做不到钩子化的保全逻辑硬编码进内核，并把欠债写进 TODO。
6. **停止条件是可编译的配置**：循环门走"目录 + 严格参数模型 + 原子编译"，互斥组与成本标注在编译期生效——"什么时候停"从硬编码纪律变成用户可编辑、系统可校验的声明。

---

## 附：关键文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `runtime/builder.py` | 1290 | AgentBuilder：每请求代理装配 |
| `runtime/envelope.py` | 937 | SSE 信封状态机 |
| `runtime/runtime.py` | 544 | Runtime：八阶段编排主循环 |
| `runtime/tool_guard.py` | 427 | 运行时工具守卫 |
| `runtime/prompt_contributors.py` | 400 | 提示词贡献者 |
| `runtime/hooks.py` | 337 | 钩子抽象、拓扑排序、注册表 |
| `runtime/tool_registry.py` | 322 | 工具注册表 |
| `runtime/executor.py` | 116 | AgentExecutor：心跳执行器 |
| `runtime/phases.py` | 41 | Phase 八阶段枚举 |
| `loop/catalog.py` | 293 | 门目录与严格参数模型 |
| `loop/gates/doom_loop.py` | 287 | 重复保护门 |
| `loop/gates/runner.py` | 215 | 停止门执行与作用域过滤 |
| `loop/gates/handler.py` | 180 | StopHandler 组合器 |
| `loop/compiler.py` | 45 | 声明式模式 → StopHandler 原子编译 |
