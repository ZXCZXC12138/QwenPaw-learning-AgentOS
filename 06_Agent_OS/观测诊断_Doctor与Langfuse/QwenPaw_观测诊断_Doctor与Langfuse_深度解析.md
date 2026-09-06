# QwenPaw 观测诊断（Doctor 与 Langfuse）深度解析

> **来源**：基于 QwenPaw 代码仓 `src/qwenpaw/observability/langfuse.py`（238 行）、`cli/doctor_*.py`（5 文件，约 3700 行）、`app/inbox_trace_store.py`（229 行）、`utils/logging.py`（341 行）、`hooks/observability/langfuse_hook.py`（91 行）全量源码分析
> **分析范围**：三套观测体系（Langfuse 分布式追踪 / 本地 inbox_traces / 应用日志）、Doctor 诊断命令（只读检查 × 保守修复分离）、修复分级（SAFE / READONLY / SYNC / MUTATING）、`--deep` 渠道连通性探测、doctor 扩展注册表
> **本文档包含四部分内容**：
> 1. 三套观测体系：各司其职的分工
> 2. Langfuse 集成：可选依赖与追踪作用域
> 3. Doctor：只读诊断与分级修复
> 4. 诊断的可扩展面

---

## 目录

### Part 1: 三套观测体系
- Langfuse：单次请求内部因果树
- inbox_traces：后台任务留档
- 应用日志：横切证据链

### Part 2: Langfuse 集成
- 可选依赖：无 langfuse 即全 no-op
- agent_trace_scope：根 span 与嵌套恢复
- tool_span 与 LLM generation kwargs

### Part 3: Doctor 命令
- 检查与修复的硬分离
- 修复四级分类与非交互白名单
- `--deep` 连通性：失败不是致命的

### Part 4: 扩展面与设计哲学

---

# Part 1: 三套观测体系：各司其职的分工

QwenPaw 的观测不是一个系统，而是三套各有领地的体系：

| 体系 | 载体 | 领地 | 生命周期 |
|------|------|------|----------|
| **Langfuse 追踪** | `observability/langfuse.py` + `hooks/observability/` | 单次请求内部的因果树：ReAct 循环 → 工具调用 → LLM generation | 可选外部服务，按请求开闭 |
| **本地 inbox_traces** | `app/inbox_trace_store.py` | 后台任务（Cron / 投递）的执行留档 | 按 `run_id` 落盘 JSON，工作区 `inbox_traces/` 目录 |
| **应用日志** | `utils/logging.py` | 横切一切模块的证据链 | 滚动文件 5 MiB × 3 备份 |

三者咬合而非重叠：Langfuse 回答"这一轮里模型为什么这样做"，inbox_traces 回答"昨晚那次后台作业到底跑了没、结果如何"，日志回答"系统在那个时刻整体处于什么状态"。

# Part 2: Langfuse 集成：可选依赖与追踪作用域

## 2.1 可选依赖：没有就全 no-op

模块文档第一句就是设计宣言：

> This module is intentionally optional: QwenPaw does not depend on langfuse at install time, so **every helper becomes a no-op when Langfuse is unavailable**.

启用判定 `is_langfuse_enabled()` 三重检查：`LANGFUSE_SECRET_KEY` 环境变量存在 + `importlib.util.find_spec("langfuse")` 找得到包 + `langfuse.openai` 子模块可导入。**观测是增益不是依赖**——没装的用户零负担，装了的用户自动获得追踪。

## 2.2 agent_trace_scope：根 span 与嵌套恢复

追踪上下文走 `ContextVar`（`_current_trace`），根 span 由 `agent_trace_scope` 异步上下文管理器创建——对应一次完整的 ReAct 循环。实现细节：

- **成功路径**：`root_span.update(output={"status": "success"})`
- **异常路径**：`level="ERROR"` + `status_message` + `{"status": "error"}`，然后 **re-raise**——观测不许吞异常
- **嵌套安全**：`finally` 中恢复**先前的**追踪上下文（有则还原、无则清空）——子代理嵌套场景下，内层作用域退出后外层上下文原样恢复
- **无客户端降级**：即使客户端初始化失败，也设置 ContextVar 让 `current_generation_kwargs` 保持自洽——**追踪结构完整，只是不外发**

`LangfuseTraceHook` 以 `PRE_EXECUTE` 钩子（priority 12）接入 Runtime 八阶段，注释写明位置依据："After ContextVarsSetupHook(10) so session/agent metadata is available, before BootstrapHook(20)"。trace 元数据携带完整身份链：`session_id` / `root_session_id` / `agent_id` / `root_agent_id` / `user_id` / `channel`——**子代理轮次也能追回根会话**。

## 2.3 工具与 LLM 两级观测

`tool_span` 在当前追踪下开 `as_type="tool"` 观测，命名 `tool.<name>`，异常时标错、`finally` 必 `end()`。LLM 侧由 `current_generation_kwargs` 生成 Langfuse 专属 kwargs（`trace_id` / `parent_observation_id` / `name="llm.<model>"`），由 OpenAI SDK 包装层透传——**追踪树：agent 根 span → 工具 span / LLM generation，因果链完整**。

## 2.4 本地轨迹与日志的配套工程

`inbox_trace_store.py`：每次后台运行一个 `run_id.json`，`_to_jsonable` 递归把 Pydantic 模型/任意对象压成 JSON（兜底 `{"repr": ...}`），原子写入 + 模块级异步锁。日志侧细节成色同样足：

- **滚动策略**：5 MiB × 3 备份，环境变量可覆盖（`QWENPAW_LOG_MAX_SIZE` 支持 `KiB/MiB/GiB` 人类单位解析）
- **幂等装配**：重复初始化不产生重复 handler
- **`_SafeRotatingFileHandler`**：轮转失败时捕获错误并重开流——**日志系统自己坏了不许拖垮应用**
- **`sanitize_log_value`**：日志值脱敏——全仓所有模块的日志输出统一过这道门
- **命名空间过滤**：只放行 `qwenpaw.*` logger，三方库噪音不进文件

# Part 3: Doctor：只读诊断与分级修复

## 3.1 检查与修复的硬分离

`doctor_cmd.py` 模块注释一行定调：

> `qwenpaw doctor` — read-only checks. `qwenpaw doctor fix` — conservative repairs with backup.

`doctor_checks.py`（1311 行）文件头再强调一遍："Read-only diagnostics (no config or disk mutations)."——**诊断命令永远不会改变系统状态**，这让 `doctor` 可以放心进 CI、进巡检脚本。检查面覆盖极广：配置键未知扫描（`scan_unknown_config_keys`）、agent 档案与工作区一致性、模型连接（`check_enabled_agents_model_connections`）、Cron 作业文件、技能布局、MCP 客户端、记忆嵌入、浏览器就绪、安全基线（`security_baseline_notes`）、工作区卫生、磁盘空间、日志可写性……每条检查失败后附一行可操作提示（`_doctor_fix_hint`）。

## 3.2 修复四级分类

`doctor_fix_runner.py`（847 行）把修复动作分成四级，注释即政策：

```
SAFE_FIX_IDS       ensure-working-dir / ensure-workspace-dirs     随时可做
READONLY_FIX_IDS   validate-all-jobs-json    纯校验无写入，不需 --yes（供 CI 用非零退出码）
SYNC_FIX_IDS       reconcile-workspace-skills   复用应用同款调谐函数，不需 --yes
MUTATING           可能创建/改写 agent.json、jobs.json —— 必须 --yes
```

`--non-interactive` 模式只放行"安全 + 只读校验 + 技能调谐"三类，**危险动作即便带 `-y` 也拒绝**——非交互场景的保守是刻意的。所有变更修复先备份。`validate-all-jobs-json` 复用 `check_cron_jobs_files`——**CLI 校验与服务端校验是同一份代码**，不存在两套漂移的标准。`reconcile-workspace-skills` 同理复用应用的 `reconcile_workspace_manifest`。

## 3.3 `--deep` 连通性：失败不致命

`doctor_connectivity.py` 的定位注释：

> Failures are reported as **non-fatal notes** (firewalls and offline use are common).

深度探测只对显式启用的渠道做，内置探针（TCP 端口、HTTP 端点、回环地址绑定检查）与渠道自定义探针（`BaseChannel.doctor_connectivity_notes`，见渠道篇）共存。**连通性失败是提示不是诊断结论**——离线环境、企业防火墙是常态，误报比漏报更损害工具信誉。

# Part 4: 扩展面与设计哲学

## 4.1 doctor 注册表：插件参与诊断

`doctor_registry.py` 提供两条扩展路径：**setuptools entry point**（组名 `qwenpaw.doctor`，兼容遗留的 `copaw.doctor` 组）与导入时手工注册（`register_doctor_contribution`）。扩展函数签名统一：接受 `DoctorRunContext`（cfg / raw_cfg / cli_base_url / timeout / deep），返回信息行列表。第三方插件的诊断智慧可以汇入同一张体检单。

## 4.2 设计哲学总结

1. **观测三套体系各守一地**：Langfuse 管请求内因果树，inbox_traces 管后台任务留档，日志管横切证据链——咬合而不重叠。
2. **观测是增益不是依赖**：Langfuse 全链路 no-op 降级；追踪结构在本地保持完整，只是不外发。
3. **追踪不许吞异常**：错误标注后 re-raise；嵌套作用域退出必须还原外层上下文。
4. **诊断只读，修复分级**：`doctor` 永远不改系统，可进 CI；修复按风险四级分类，非交互模式白名单收紧，危险动作 `-y` 也拒。
5. **同一份校验代码两用**：CLI 的 jobs 校验与技能调谐直接复用服务端函数——标准不许漂移。
6. **日志自治**：滚动、幂等、脱敏、轮转自恢复——日志系统自身的故障不许传染给应用。
7. **连通性失败是提示不是判决**：离线与防火墙是常态，诊断工具的可信度建立在克制上。

---

## 附：关键文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `cli/doctor_checks.py` | 1311 | 只读检查集合 |
| `cli/doctor_cmd.py` | 1089 | doctor / doctor fix 命令编排 |
| `cli/doctor_fix_runner.py` | 847 | 分级修复执行器 |
| `cli/doctor_connectivity.py` | 361 | --deep 渠道连通性探测 |
| `cli/doctor_registry.py` | 125 | 扩展注册（entry point + 手工） |
| `observability/langfuse.py` | 238 | Langfuse 追踪作用域 |
| `hooks/observability/langfuse_hook.py` | 91 | PRE_EXECUTE 追踪钩子 |
| `app/inbox_trace_store.py` | 229 | 后台任务本地轨迹存储 |
| `utils/logging.py` | 341 | 日志装配、滚动、脱敏 |
