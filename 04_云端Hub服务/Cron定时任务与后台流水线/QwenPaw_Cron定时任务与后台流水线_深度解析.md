# QwenPaw Cron 定时任务与后台流水线 深度解析

> **来源**：基于 QwenPaw 代码仓 `src/qwenpaw/app/crons/` 与 `src/qwenpaw/hooks/cron/` 全量源码分析（约 2600 行）
> **分析范围**：`models.py`（任务规格模型，279 行）、`manager.py`（CronManager 调度核心，875 行）、`executor.py`（CronExecutor 执行器，309 行）、`heartbeat.py`（心跳任务，404 行）、`contracts.py`（服务作业契约）、`repo/`（仓储层）、`hooks/cron/cron_hook.py`（三个生命周期钩子）
> **本文档包含五部分内容**：
> 1. 定时任务体系总览：三类作业共用一个调度器
> 2. 任务规格模型：声明式 Spec 与时区陷阱治理
> 3. CronManager：调度核心的工程细节
> 4. CronExecutor：执行、投递与可观测性
> 5. Cron 生命周期钩子：让定时运行"无上下文而留痕"

---

## 目录

### Part 1: 体系总览
- 三类定时作业：用户 Cron / Heartbeat / 服务作业
- 组件结构与数据流

### Part 2: 任务规格模型
- ScheduleSpec：cron 与 once 双模调度
- 星期几数字的标准化：一个跨调度器的经典坑
- DispatchSpec 与 JobRuntimeSpec：投递与运行时约束

### Part 3: CronManager 调度核心
- 启动流程：加载、清洗、注册、自愈
- keepalive：为不唤醒事件循环的平台兜底
- 并发信号量与错过/超限事件处理
- 仓储层与执行历史

### Part 4: CronExecutor 执行流水线
- text 与 agent 两种任务类型
- stream / final / silent 三种投递模式
- trace 三段式：创建、增量追加、终结
- 投递失败的降级路径

### Part 5: 生命周期钩子与服务作业契约
- CronContextHook：给请求打"定时来源"标签
- 记忆隔离/恢复：每次运行无上下文，持久会话全留痕
- ServiceCronJob：后端中立的作业声明
- Heartbeat：HEARTBEAT.md 驱动的自主心跳
- 设计哲学总结

---

# Part 1: 定时任务体系总览

## 1.1 三类定时作业

QwenPaw 的 `app/crons/` 不是简单的"定时发消息"，而是一个**统一调度平面**，承载三类语义完全不同的作业：

| 作业类别 | 作业 ID 特征 | 声明方 | 语义 |
|---------|-------------|--------|------|
| 用户 Cron 任务 | 服务端生成的 UUID | 用户（API/CLI/Console） | 定时让 Agent 干活并把结果投递到渠道 |
| Heartbeat 心跳 | `_heartbeat`（内部） | 配置（`get_heartbeat_config`） | 按间隔用 `HEARTBEAT.md` 作为查询主动运行 Agent |
| 服务作业 | `_service:<source>:<key>` | 工作区服务（如 memory_manager） | 后台周期任务（如记忆的 auto_memory / auto_dream） |

三类作业共用同一个 `AsyncIOScheduler` 实例、同一套错过（misfire）策略与事件监听，但**内部作业（心跳与服务作业）不进入用户的执行历史与跳过记录**——`_is_internal_job()` 按 ID 前缀过滤。这是"统一基建、隔离语义"的典型分层。

## 1.2 组件结构与数据流

```
┌──────────────────────────────────────────────────────────────┐
│  api.py  FastAPI /cron/*  （CRUD · pause/resume · run · history）│
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│  manager.py  CronManager                                       │
│   AsyncIOScheduler · per-job Semaphore · 事件监听              │
│   触发器构建（Cron/Date/Interval）· 状态与历史簿记             │
├──────────────┬──────────────────────┬────────────────────────┤
│ executor.py  │ heartbeat.py         │ contracts.py           │
│ CronExecutor │ run_heartbeat_once   │ ServiceCronJob         │
│ （用户作业） │ （心跳作业）          │ （服务作业声明）        │
├──────────────┴──────────────────────┴────────────────────────┤
│  workspace.stream_query(req)  →  Runtime 8-Phase 管线          │
│  hooks/cron/cron_hook.py：CronContext / MemoryIsolate / Restore │
├───────────────────────────────────────────────────────────────┤
│  channel_manager.send_event / send_text → 渠道投递             │
│  inbox_trace_store：trace 三段式留痕 · inbox_store：结果收件箱 │
└───────────────────────────────────────────────────────────────┘
```

数据流的关键点：**定时任务不是一条旁路**——agent 类型的作业最终走的是与用户消息完全相同的 `workspace.stream_query()` 入口，只是请求上多打了 `source: cron` 的标记，由钩子在管线内调整行为。

# Part 2: 任务规格模型

## 2.1 ScheduleSpec：双模调度

```python
class ScheduleSpec(BaseModel):
    type: Literal["cron", "once"] = "cron"
    cron: Optional[str] = None
    run_at: Optional[datetime] = None
    timezone: str = "UTC"
    repeat_every_days: Optional[int] = Field(default=None, ge=1)
    repeat_end_type: Optional[Literal["never", "until", "count"]] = None
```

`once` 模式不是"只跑一次"的同义词——它支持 `repeat_every_days` 的日级重复，且重复终止条件三选一（永不 / 直到某日 / 计次）。`_validate_schedule_type` 校验器在两种模式间**互斥清空无关字段**（cron 模式清掉 run_at 与 repeat 系列，once 模式清掉 cron）——规格对象永远处于自洽状态，不存在"既是 cron 又带 run_at"的幽灵组合。

`normalize_cron_5_fields` 还做了宽容归一：4 段输入视为"时 日 月 周"（自动补 0 分），3 段视为"日 月 周"——但 6 段（带秒）直接拒绝。**支持简化输入，拒绝超纲能力**（秒级调度被显式排除）。

## 2.2 星期几数字的标准化：一个跨调度器的经典坑

`models.py` 开头有一段极具信息量的注释：

```python
# APScheduler v3 uses ISO 8601 weekday numbering (0=Mon … 6=Sun) for
# CronTrigger(day_of_week=...), while standard crontab uses 0=Sun … 6=Sat.
# from_crontab() does NOT convert either.
```

**同一个数字 `0`，在 POSIX crontab 里是周日，在 APScheduler 里是周一**——用户写 `0 9 * * 0`（期望周日上午九点）会被 APScheduler 解释成周一。且 APScheduler 自己的 `from_crontab()` 也不做转换。

QwenPaw 的解法是**归一到无歧义的英文缩写**：

```python
_CRONTAB_NUM_TO_NAME = {"0": "sun", "1": "mon", ..., "7": "sun"}

def _crontab_dow_to_name(field: str) -> str:
    # 处理 *、单值、逗号列表、范围（1-5 → mon-fri）、步长（*/2）
```

`mon`–`sun` 缩写在两个体系里含义一致，于是在**校验时**（`normalize_cron_5_fields`）就把第五段从数字转成缩写，下游所有触发器构建都不再有歧义。注意 `"7": "sun"`——crontab 习惯里 0 和 7 都表示周日，这里一并收敛。这是**把兼容性问题在边界处一次性消化**的范例：模型层负责翻译，调度层只见干净数据。

## 2.3 DispatchSpec 与 JobRuntimeSpec

```python
class DispatchSpec(BaseModel):
    channel: str = Field(default=DEFAULT_CHANNEL)
    target: DispatchTarget            # user_id + session_id
    mode: Literal["stream", "final"] = "stream"
    silent: bool = False              # 只跑不投递

class JobRuntimeSpec(BaseModel):
    max_concurrency: int = 1
    timeout_seconds: int = 120
    misfire_grace_seconds: int = 600
    share_session: bool = True
    tool_safety: bool = False
```

三个产品化细节：

- **`silent` 模式**：`"Run an agent task without delivering its events to the channel."`——Agent 照跑、会话与 trace 照存，只是不打扰用户。适合"夜间维护型"任务。且校验器规定 `silent` 仅支持 agent 任务（纯文本任务不投递就没有存在意义）。
- **`tool_safety` 双档**：`False`（默认）= 工具执行 OFF 审批，全放行，适合可信的自动化任务；`True` = AUTO 模式，危险工具需审批——注释明确警告这可能**阻塞无人值守执行**。无人值守与工具安全的张力被显式交给用户选择，而不是偷偷选边。
- **`save_result_to_inbox` 产品规则**：未显式设置时按组合推导——`text + cron（周期）` 默认关（周期发固定文本入收件箱会很吵），其余默认开。规则写在代码注释里（"Product rule"），可追溯。

# Part 3: CronManager 调度核心

## 3.1 启动流程：加载、清洗、注册、自愈

`start()` 的编排顺序：

```
1. repo.load()  →  加载 JobsFile（version=2）
2. prune_orphan_history(valid_job_ids)   # 清掉已不存在作业的历史
3. 注册调度器事件监听（MISSED / MAX_INSTANCES）
4. scheduler.start()
5. 逐个 _register_or_update(job)
6. 注册心跳（如配置启用）
7. _register_memory_jobs()   # 记忆服务的周期作业
8. 启动 keepalive 任务
```

第 5 步的自愈逻辑值得注意：

```python
except Exception as e:
    logger.warning("Skipping invalid cron job during startup: ...")
    if job.enabled:
        disabled_job = job.model_copy(update={"enabled": False})
        await self._repo.upsert_job(disabled_job)
        logger.warning("Auto-disabled invalid cron job: ...")
```

**启动不因坏作业失败，但坏作业会被自动禁用并落盘**——不是跳过一次（下次启动还会再报错），而是把 `enabled` 置为 False 持久化。系统启动的健壮性与问题的可见性同时保住。

## 3.2 keepalive：为不唤醒事件循环的平台兜底

```python
# APScheduler's AsyncIOScheduler processes due jobs via loop call_later
# wakeups; on some platforms (e.g. WSL2) a long-delay call_later does
# not reliably wake an otherwise-idle loop, so cron jobs misfire until
# the next HTTP request arrives (see issue #6471).
CRON_KEEPALIVE_INTERVAL_SECONDS = 60

async def _keepalive_loop(self) -> None:
    while self._started:
        await asyncio.sleep(CRON_KEEPALIVE_INTERVAL_SECONDS)
```

一个真实的平台缺陷及其工程对策：WSL2 上 asyncio 的长延迟 `call_later` 不能可靠唤醒空闲事件循环，导致定时任务**直到下一次 HTTP 请求到来才触发**。对策不是换调度器，而是一个每 60 秒空转一次的常驻任务——保持事件循环持续"扫表"，代价极小。**注释里标注了 issue 编号**，让这段看似无意义的代码可溯源。

## 3.3 并发信号量与事件处理

每个作业注册时创建独立的并发运行时：

```python
@dataclass
class _Runtime:
    sem: asyncio.Semaphore

# _register_or_update:
self._rt[spec.id] = _Runtime(
    sem=asyncio.Semaphore(spec.runtime.max_concurrency))
```

`_execute_once` 全程持有该信号量——`max_concurrency=1`（默认）意味着同一作业的上一次没跑完，新触发就排队。调度器层面则通过事件监听补记原因：

| 事件 | 处理 | 记录 |
|------|------|------|
| `EVENT_JOB_MISSED`（错过宽限期） | `_handle_job_missed` | `skipped` + "late by Xs, grace=Ys" |
| `EVENT_JOB_MAX_INSTANCES`（并发满） | `_handle_job_max_instances` | `skipped` + 被跳过的时间点 |

两类事件都走 `_record_skipped()`：更新状态、追加历史（`CRON_HISTORY_LIMIT = 50` 环形上限），且**都先检查 `_is_internal_job` 跳过内部作业**——内部作业的调度噪音不污染用户的作业历史。

## 3.4 触发器构建与仓储层

`_build_trigger` 按规格分发到三种 APScheduler 触发器：

- `once` 无重复 → `DateTrigger`
- `once` + `repeat_every_days` → `IntervalTrigger`（count 终止条件换算成 end_date）
- `cron` → `CronTrigger`（强制 5 段，否则抛 `ConfigurationException` 快速失败——注释明确"先验证再动调度器状态"）

仓储层是干净的端口/适配器结构：

```
repo/base.py      BaseJobRepository（ABC，11 个方法）
repo/json_repo.py JSON 文件实现（324 行）
```

`JobsFile(version=2)` 带版本号——规格模型演进时有迁移抓手。`prune_orphan_history` 在启动时清理孤儿历史，历史与作业的一致性由加载流程保证而非依赖运行期自律。

# Part 4: CronExecutor 执行流水线

## 4.1 两种任务类型的执行路径

`CronExecutor.execute()` 是唯一的执行入口：

**text 任务**：直接 `channel_manager.send_text()`，投递失败不影响作业本身"成功"——返回 `delivery_status` 单独表达。

**agent 任务**：构造请求注入三个关键上下文：

```python
request_context["source"] = "cron"
request_context["cron_job_id"] = job.id or ""
request_context["approval_level"] = (
    ToolExecutionLevel.AUTO.value if job.runtime.tool_safety
    else ToolExecutionLevel.OFF.value)
```

`source: cron` 是后续钩子识别定时请求的凭据；`approval_level` 把 Part 2 的 `tool_safety` 开关翻译成运行时工具执行级别。

会话隔离策略由 `share_session` 决定：

```python
if share_session:
    req["session_id"] = target_session_id or f"cron:{job.id}"
else:
    # Use job.id (not run_id) so all runs of this job accumulate in the
    # same dedicated session, giving users a complete history.
    req["session_id"] = f"{target_session_id}:cron:{job.id}"
    req["session_source"] = "cron"
```

隔离模式用 `job.id` 而非每次运行的 `run_id` 作会话后缀——**隔离的是"与用户会话的混合"，不是"运行之间的连续性"**，用户仍能看到该作业的完整历史。同时 `get_or_create_chat()` 注册 ChatSpec，让这个定时会话出现在前端会话列表里。

## 4.2 stream / final / silent 三种投递模式

事件流的消费循环是投递策略的核心：

```python
async for event in self._workspace.stream_query(req):
    if job.dispatch.silent:
        continue                    # 只消费，不投递
    if job.dispatch.mode == "final":
        if (event.object == "message"
                and event.status == RunStatus.Completed):
            final_event = event     # 只留最后一条完成消息
        continue
    await _deliver(event)           # stream：逐事件实时转发
if final_event is not None:
    await _deliver(final_event)
```

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `stream` | 每个事件实时转发到渠道 | 交互式体验，像真人对话 |
| `final` | 消费全流，只投递最后的完成消息 | 只要结果不要过程 |
| `silent` | 全流消费但不投递 | 夜间维护、静默巡检 |

`final` 模式还处理了"流里没有完成消息"的退化情况——置 `final_no_content` 并告警，投递状态记为 `no_content` 而非假装成功。

## 4.3 trace 三段式与超时治理

执行前记录基线，执行中创建 trace，结束后按会话增量追加：

```
read_session_messages()  → baseline_count（执行前消息数）
create_trace(run_id, meta={job_id, silent, ...})
asyncio.wait_for(_run(), timeout=runtime.timeout_seconds)
append_trace_from_session_delta(..., baseline_count)
finalize_trace(run_id, status=success/timeout/cancelled/error)
```

**trace 的增量来自"执行前后会话消息差"**（session delta）——不需要侵入 Runtime 内部打点，靠会话快照差分就能还原一次运行产生了什么。超时（`asyncio.wait_for`）、取消、异常三条路径**都**执行同样的"追加增量 + 终结 trace"——可观测性不因失败而丢失。

## 4.4 投递失败的降级路径

`_execute_once` 区分两种失败：**执行失败**（异常）与**投递失败**（`delivery_status == "failed"`）。后者有专门的降级：

```python
if delivery_failed:
    await append_inbox_event(
        event_type="cron_delivery_failed_fallback",
        title=f"Cron result not delivered: {job.name}",
        body="Task executed successfully, but channel delivery failed.")
```

**任务成功但渠道投递失败 → 结果不丢，落到 Inbox 收件箱并显式告知**。加上正常路径的 `cron_result` 事件（受 `save_result_to_inbox` 控制），用户作业的结果至少有两条可达路径：渠道直投 + 收件箱兜底。后台任务的异常还会经 `_task_done_cb` 推送到 console push store（`❌ Cron job [...] failed`）——失败对前端可见，而不是静默吞掉。

# Part 5: 生命周期钩子与服务作业契约

## 5.1 CronContextHook：来源标记

```python
class CronContextHook(LifecycleHook):
    phase = Phase.PRE_DISPATCH
    priority = 5

    async def run(self, ctx: HookContext) -> HookResult:
        source = getattr(ctx.request, "session_source", None)
        if source == "cron":
            ctx.extras[IS_CRON_KEY] = True
```

在管线最早期（`PRE_DISPATCH`，priority=5）打标。模块注释点明用途：下游钩子据此调整行为，例如 `BootstrapHook` 会跳过引导语注入——**定时任务不需要"你好，我能帮你做什么"这类面向人类的开场白**。

## 5.2 记忆隔离/恢复：每次运行无上下文，持久会话全留痕

这是整套体系里最精巧的一对钩子，模块 docstring 直接给出设计意图：

> snapshot & clear agent memory before execution, then restore the full history plus new messages afterward so **each cron run is context-free while the persisted session still accumulates all runs' conversations**.

```
PRE_EXECUTE（agent 构建 + 会话加载之后，priority=10）
  CronMemoryIsolateHook:
    snapshot = list(state.context)      # 快照全部历史
    snapshot_summary = state.summary
    state.context.clear()               # 清空 → 本次运行从零开始
    state.summary = ""

POST_RESPONSE（priority=80，必须早于 SessionSaveHook 的 90）
  CronMemoryRestoreHook:
    new_messages = list(state.context)  # 本次运行产生的新消息
    state.context = snapshot + new_messages   # 历史 + 新增
    state.summary = old_summary
```

解决的是一个真实的矛盾：**持久会话**要求每次运行的记录都累积下来（用户随时可查完整历史），但**运行时上下文**若带着历次运行记录，定时任务会越跑越贵、且被陈旧上下文带偏。快照-清空-恢复的三明治结构让两个诉求各自成立。钩子顺序用 priority 显式编排：恢复（80）必须赶在会话保存（90）之前，否则落盘的就是空历史——**钩子的执行顺序本身是设计约束，写在注释里**。

## 5.3 ServiceCronJob：后端中立的作业声明

```python
@dataclass(frozen=True)
class ServiceCronJob:
    """A cron job declared by a workspace service.

    The declaration intentionally contains no APScheduler types. Services
    own job semantics and configuration; CronManager owns scheduling.
    """
    key: str
    cron: str
    callback: Callable[[], Awaitable[None]]
    misfire_grace_seconds: int = 600
    jitter_seconds: int = 0
```

21 行代码，两条设计原则：

1. **后端中立**：声明里故意不含任何 APScheduler 类型——记忆服务声明"我有个周期作业"，不知道也不关心底层是什么调度器。调度引擎可替换，服务代码零改动。
2. **职责分界**：服务拥有作业语义（干什么），CronManager 拥有调度（怎么排）。`jitter_seconds` 支持错峰——多个服务作业不必挤在同一秒触发。

CronManager 侧的 `_register_service_jobs` 用 `_service:<source>:<key>` 命名空间隔离作业 ID，拒绝重复 key、拒绝含 `:` 的非法 key——服务作业的注册是严格校验的，坏声明只记日志不拖垮启动。

## 5.4 Heartbeat：HEARTBEAT.md 驱动的自主心跳

`heartbeat.py` 实现 QwenPaw 的"主动心跳"：按配置间隔把 `HEARTBEAT.md` 文件内容作为查询运行 Agent，让 Agent 自主决定要不要做点什么、要不要联系用户。

调度表达式双轨：

```python
def is_cron_expression(every: str) -> bool:
    # 5 段 cron（第五段支持 mon-fri 命名范围）
def parse_heartbeat_every(every: str) -> int:
    # "30m" / "1h" / "2h30m" / "90s" → 秒
```

先判 `is_cron_expression` 再走间隔解析——两种语法严格区分，非法值回退默认 30 分钟（记警告而非抛错）。

`_in_active_hours()` 体现产品克制：心跳只在用户配置的活跃时段内运行（按用户时区换算，支持跨午夜区间如 22:00–06:00），时区非法时降级 UTC 并告警。**主动式 Agent 不打扰用户**，这是心跳机制能长期开着的前提。

投递侧 `target=last`（发给最近一次对话的渠道）与 `target=inbox`（只进收件箱）双目标，消息预览提取覆盖 text / thinking / tool_result 三种内容块——心跳的输出以可读摘要形式触达用户。

## 5.5 设计哲学总结

1. **统一调度平面，隔离作业语义**：用户作业、心跳、服务作业共用调度器，但内部作业不污染用户可见的状态与历史。
2. **边界处消化兼容性**：crontab 与 APScheduler 的星期编号冲突在校验层归一为英文缩写，下游永不见歧义。
3. **失败可见、可溯、不扩散**：坏作业启动时自动禁用落盘；执行/投递/超时/取消四条路径都终结 trace；投递失败有收件箱兜底。
4. **平台缺陷用最小代价兜底**：60 秒 keepalive 对抗 WSL2 事件循环不唤醒问题，注释标注 issue 编号可溯源。
5. **快照-清空-恢复三明治**：定时运行无上下文（便宜且不被带偏），持久会话全留痕（可审计）——两个矛盾诉求用钩子顺序精确编排。
6. **后端中立的作业契约**：`ServiceCronJob` 不含调度器类型，服务拥有语义、管理器拥有调度。
7. **主动式 Agent 的克制**：心跳受活跃时段约束，无人值守场景的工具审批策略显式交给用户选择。

---

## 附：Cron 模块文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `manager.py` | 875 | CronManager：调度、并发、事件、状态簿记 |
| `heartbeat.py` | 404 | 心跳：表达式解析、活跃时段、运行与投递 |
| `executor.py` | 309 | CronExecutor：执行与三模式投递 |
| `models.py` | 279 | 规格模型：Schedule/Dispatch/Runtime/Spec |
| `api.py` | 200 | FastAPI `/cron/*` 路由 |
| `repo/json_repo.py` | 324 | JSON 文件仓储实现 |
| `repo/base.py` | 79 | BaseJobRepository 契约 |
| `contracts.py` | 21 | ServiceCronJob 服务作业声明 |
| `hooks/cron/cron_hook.py` | 134 | 三个生命周期钩子 |
