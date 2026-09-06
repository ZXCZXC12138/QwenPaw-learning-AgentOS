# QwenPaw 上下文管理与 Offload 机制 深度解析

> **来源**：基于 QwenPaw 代码仓 `src/qwenpaw/agents/context/`（scroll 引擎 + 视觉压缩）与 `agents/offloader.py` 全量源码分析
> **分析范围**：`agents/context/`（base / scroll / visual_compression 三个子包，约 9000 行）+ `agents/offloader.py`（176 行）
> **本文档包含五部分内容**：
> 1. 可插拔上下文管理架构：ContextManager 协议
> 2. Scroll 策略：持久化历史 + 驱逐索引 + 召回 REPL
> 3. Offloader：压缩产物落盘协议
> 4. 视觉压缩：把文本渲染成图片的激进路线
> 5. 安全门禁与优雅降级设计

---

## 目录

### Part 1: 可插拔架构
- ContextManager Protocol 的三个钩子
- "注入即替换，不注入即原生"的策略模式
- 为什么用注入而不是继承

### Part 2: Scroll 策略
- 设计核心：write-through + eviction-index
- history.db 持久化历史存储
- continuation summary：状态缓存而非替代品
- EvictionIndex：上下文内的导航索引
- 双召回通道：recall_history 正门 + recall_history_python REPL
- RecallLoopGuard 防召回死循环

### Part 3: Offloader 落盘协议
- 对话归档：按日期分组的 JSONL
- 工具结果归档：UUID 文本文件
- 过期清理与跨平台时间戳

### Part 4: 视觉压缩
- 核心思想：用图像 patch 换文本 token
- 三档 effort 预设
- 压缩管线九大组件

### Part 5: 安全与降级
- 沙箱召回的双层门禁（default-deny）
- 任何失败都降级到 native
- history.db 容量治理

---

# Part 1: 可插拔上下文管理架构

## 1.1 ContextManager Protocol

`agents/context/base.py` 定义了一个 `runtime_checkable` 的 Protocol，只有三个方法：

```python
@runtime_checkable
class ContextManager(Protocol):
    async def recover_from_context_overflow(self, agent) -> bool:
        """尝试缩短被模型提供商拒绝的输入。
        仅当恢复动作足以让重建模型输入并重试有意义时才返回 True。"""

    async def compress(self, agent, context_config=None,
                       instructions=None) -> None:
        """当上下文超过阈值时压缩。在 AgentScope 压缩中间件链之后调用。
        instructions 是一次性指引，不得持久化为活跃会话状态。"""

    def on_save(self, agent, blocks) -> None:
        """响应刚追加进 agent.state.context 的 blocks，
        把它们写穿（write-through）到持久存储。"""
```

三个钩子精确对应上下文生命周期的三个关键时刻：

| 钩子 | 触发时机 | 职责 |
|------|---------|------|
| `on_save` | 每轮消息入窗后 | 写穿持久化（不丢数据） |
| `compress` | token 超阈值 | 压缩窗口（控制成本） |
| `recover_from_context_overflow` | 提供商拒绝请求 | 溢出恢复（兜底自救） |

`QwenPawAgent`（react_agent）把自身的三个内部点委托给注入的 manager：

- `_save_to_context` → `on_save`（在基类 append 之后）
- `_compress_context_impl` → `compress`（替代原生压缩）
- 溢出重试 → `recover_from_context_overflow`

## 1.2 "注入即替换，不注入即原生"

`base.py` 的模块注释说得很直白：

> When no manager is injected, the agent keeps its native AgentScope behavior — so a strategy is purely additive and fully opt-in.

这是教科书式的策略模式应用：

- **默认路径**：AgentScope 原生压缩（什么都不装）
- **注入路径**：`LightContextConfig.strategy == "scroll"` 时，构建器调用唯一入口 `build_scroll_components()` 装配 scroll 策略
- **非 scroll 策略**：入口返回 `None`，agent 保持原生——功能完全 opt-in

## 1.3 为什么用注入而不是继承

`scroll/manager.py` 的模块注释点明了设计动机：

> The strategy form of the design: instead of subclassing the agent, it is injected into `QwenPawAgent` and drives the two delegated hooks.

继承的问题：上下文策略与 Agent 主体耦合，换策略要换类，多个 Agent 类型 × 多个策略 = 类爆炸。注入的好处：策略是 Agent 的一个可配置部件，随 `agent.json` 配置切换，且降级路径天然存在（注入失败 → 回到原生）。

## 1.4 ContextWindowUnfitError：压缩的硬边界

`types.py` 定义了一个专门的异常：

```python
class ContextWindowUnfitError(RuntimeError):
    """Raised when context compaction cannot fit the model input window."""
    def __init__(self, *, tokens: int, hard_limit: int):
        super().__init__(
            "CONTEXT_UNFIT: context compaction could not fit the active "
            f"request into the model input window ({tokens} > {hard_limit} "
            "tokens). Reduce the request/tool set, start a new turn, or "
            "switch to a model with a larger context window.",
        )
```

压缩不是万能的——当单次请求本身（含工具集）就超过模型窗口时，压缩无解。此时不静默截断、不硬塞，而是抛出一个**携带行动建议**的错误：减小请求/工具集、开新轮次、换大窗口模型。**错误信息即运维指南**。

---

# Part 2: Scroll 策略——本版本的默认上下文引擎

## 2.1 设计核心：write-through + eviction-index

`ScrollContextManager`（`scroll/manager.py`，2025 行，整个 scroll 引擎的中枢）的设计纲领：

> * `on_save` — every live turn is persisted to the durable `conversation_history` as it enters the window (write-through).
> * `compress` — past the token threshold, keep the recent tail (and the active turn), update a continuation summary, and fold the evicted middle into an in-context `EvictionIndex`. **The summary is a state cache, never a replacement for raw history.**

拆解成四个关键决策：

1. **写穿（write-through）**：每条消息进入活跃窗口的同时写入 `history.db`——压缩只影响窗口内可见性，永不丢数据。
2. **保尾驱逐**：压缩时保留最近的尾部 + 当前活跃轮次，驱逐中间段。
3. **延续摘要（continuation summary）**：被驱逐内容的状态缓存，用于维持对话连续性。
4. **驱逐索引（EvictionIndex）**：被驱逐内容在上下文中留下可导航的索引条目，模型需要时可按图索骥召回原文。

最关键的一句是加粗那句：**摘要是状态缓存，原始历史的替代品永远在磁盘上**。这与"摘要即丢失信息"的传统压缩路线划清了界限。

## 2.2 组件装配链（`build_scroll_components`）

`agents/context/__init__.py` 是唯一装配入口，返回三件套：

```python
@dataclass
class ScrollComponents:
    context_manager: Any  # ScrollContextManager（委托钩子）
    repl_tool: Any        # recall_history_python（沙箱化召回 REPL）
    recall_tool: Any      # recall_history（结构化召回正门）
```

装配流程：

```
读取 agent_config.running.light_context_config
    ↓ strategy == "scroll" 且 workspace 可用？
懒导入 scroll 子包（native 路径零成本）
    ↓
HistoryStore(workspace/history.db)          # 持久化历史
    + RecallLoopGuard()                      # 召回循环防护
    + ScrollContextManager(...)              # 上下文管理器
    + make_recall_history_python(...)        # 沙箱 REPL 工具
    + make_recall_history(...)               # 结构化召回工具
    ↓
返回 ScrollComponents → 构建器接线
```

注意 `HistoryStore` 之外还传入了一个可选的 `offloader`——仅当 `scroll_config.offload_dialog == True` 时才注入：**scroll 默认不向 `dialog/` 目录写任何东西**，旧式对话归档是显式 opt-in。

### 首次运行告知与容量告警

两个运维细节：

- **首次运行**：当工作区第一次出现 `history.db`，记一条 WARNING 说明"scroll 已是默认策略、这里创建了持久历史、如何回退到 native"——用"文件不存在"作为一次性信号，文件出现后自动静默。
- **容量告警**：`history.db`（含 WAL 副文件）超过 1 GiB 时每进程告警一次，提示调低 `history_retention_days`（默认 30 天自动清理）。

## 2.3 history.db：持久化历史存储

`scroll/history.py`（760 行）实现 SQLite 持久层。数据模型由 `types.py` 的 `LogEntry` 定义：

```python
@dataclass(frozen=True)
class LogEntry:
    kind: Literal["model_turn", "context_msg", "tool_result"]
    role: str | None
    name: str | None          # 工具名
    content: str | None       # 文本主体
    metadata: dict
    tool_call_id: str | None  # 关联 tool_call <-> tool_result
    tool_input: Any
    tool_state: str | None
    headline: str | None      # 该轮的 ⟦ … ⟧ 索引行
    blocks: list[dict] | None # 精确序列化的 block 字典
    created_at: str | None
```

设计要点：除了扁平化文本，每行还保存**结构化载荷**（blocks、tool_input、tool_call_id），使得行可以**精确重建原始线格式消息**——召回不是"回忆大意"，而是"恢复原文"。

保留策略：超过 `history_retention_days`（默认 30 天）的行在启动与关停时自动清除；设为 0 则永久保留（容量告警就是为此准备的护栏）。

## 2.4 continuation summary：带质量校验的摘要更新

摘要更新不是简单的"调一次模型"。从 `manager.py` 的方法清单能看到一套完整的工程化处理：

| 方法 | 职责 |
|------|------|
| `_update_continuation_summary` / `_inner` | 摘要更新主流程 |
| `_validated_summary_attempt` | 校验摘要质量（格式、长度、来源指针） |
| `_fit_summary_prompt` | **摘要提示词预算拟合**：提示词 + 输出预留必须塞进窗口 |
| `_raise_if_summary_interrupted` | 检测摘要生成被中断 |
| `_SummaryCandidateError` | 候选摘要未通过本地格式/质量检查 |
| `_SummaryInputBudgetError` | 固定的摘要提示词都塞不下 |

相关常量揭示了预算控制：

```python
_OUTPUT_RESERVE_RATIO = 0.05          # 输出预留比例
_MAX_OUTPUT_RESERVE_TOKENS = 4096     # 输出预留上限
_SUMMARY_UPDATE_TIMEOUT_SECONDS = 60  # 摘要更新超时
```

摘要还会记录**持久化来源区间**（`_evicted_span` 返回被驱逐消息的 seq 范围）——摘要自带溯源指针，指向它在 history.db 中覆盖的原文范围。

## 2.5 双召回通道

### 正门：`recall_history`（进程内结构化查询）

注释点明定位：

> Structured front door for the common recall ops (expand / search / recall_tool): in-process bound queries, no sandbox, no approval — so fold stubs and the eviction index stay readable even when the sandboxed REPL is unavailable.

进程内执行、无需沙箱、无需审批——保证即使沙箱不可用，驱逐索引也始终可读。

### 高级门：`recall_history_python`（沙箱 REPL）

`scroll/repl.py`（303 行）提供一个让模型**写 Python/SQL 查历史**的工具，设计细节：

- **无状态单元（stateless cells）**：每次调用起新进程，Python 变量不跨调用持久化；但 `ms` 的文件后台 scratch DB 保留派生表
- **`ms` 预定义**：cell 预导言构建 `ms` 对象（只读 ATTACH 的持久历史 + 文件后台 scratch DB，来自 `memoryspace.py`，2015 行）
- **输出约束**：大输出被截断且无游标，文档明确要求"用 `LIMIT ? OFFSET ?` 分页，永远不要打印宽结果集"
- **核心 API**：
  - `ms.expand(lo, hi)`——按 seq 区间取原始轮次
  - `ms.search(query, k, kind, all_agents, session_id, agent_id, created_on/from/to)`——FTS 关键词搜索，支持跨会话/跨 Agent、日期范围过滤，命中默认带回完整轮次与边界元数据
  - `ms.recall_tool(tool_call_id)`——按工具调用 ID 召回结果

这实质上给了模型一个**对自己记忆库的编程访问能力**——召回从"固定查询"升级为"任意分析"。

### RecallLoopGuard

`recall_tool.py` 中的 `RecallLoopGuard` 防止模型陷入"召回 → 发现不够 → 再召回"的死循环。召回是昂贵的（每次都是工具调用轮次），必须有循环熔断。

## 2.6 压缩流程中的工具结果处理

`manager.py` 对工具结果有一整套专门处理（工具输出是上下文膨胀的最大来源）：

- `_tool_result_pointer_stub` / `_replace_tool_result_with_pointer`：把大工具结果替换为**指针存根**（内容已落盘，上下文只留引用）
- `_PROTECTED_RECENT_TOOL_RESULTS = 5`：最近 5 个工具结果受保护不折叠
- `_batch_fold_completed_tool_results`：批量折叠已完成工具调用的结果
- `acknowledge_model_input_tool_results`：跟踪哪些工具结果已经进了模型输入

配合 `offloader.py` 的 `offload_tool_result`（见 Part 3），形成"大结果落盘 → 上下文留指针 → 按需召回"的完整链路。

---

# Part 3: Offloader 落盘协议

## 3.1 实现 AgentScope Offloader 协议

`agents/offloader.py`（176 行）实现 AgentScope 的 `Offloader` 协议，让原生 `Agent.compress_context()` 自动把驱逐内容持久化：

```python
class QwenPawOffloader:
    """* offload_context   — 消息 → {dialog_path}/{YYYY-MM-DD}.jsonl（追加）
       * offload_tool_result — 工具输出 → {tool_results_dir}/{uuid}.txt"""
```

## 3.2 对话归档：按日期的跨会话时间线

```python
async def offload_context(self, session_id, msgs):
    # session_id 刻意不用！
    # 所有会话归档进共享的按日期分组文件
    messages_by_date: dict[str, list[Msg]] = {}
    for msg in msgs:
        date_str = msg.timestamp.split()[0] if msg.timestamp else today
        messages_by_date.setdefault(date_str, []).append(msg)
    for date_str, date_msgs in messages_by_date.items():
        filepath = f"{dialog_path}/{date_str}.jsonl"
        # 按时间戳排序后追加写入
```

关键设计决策：**`session_id` 被刻意忽略**——所有会话的消息汇入共享的按日期 JSONL 文件。注释解释了原因：这延续了原 `LightContextManager` 的**时间线设计**。跨会话的按日期时间线比按会话的孤立归档更适合"回忆某天发生了什么"的场景。

写入细节：同一日期的消息先按时间戳排序再追加，用 `aiofiles` 异步写避免阻塞事件循环，`ensure_ascii=False` 保留中文原文。

## 3.3 工具结果归档：UUID 文件

```python
async def offload_tool_result(self, session_id, tool_result):
    filepath = f"{tool_results_dir}/{uuid.uuid4().hex}.txt"
    output = tool_result.output
    # 兼容 list[dict]/dict 块结构，提取所有 text 块拼接
    await aiofiles.open(filepath, "w").write(content)
    return filepath  # 返回路径 → 上下文中的指针存根引用它
```

返回值是文件路径——调用方用它生成上下文内的指针存根，完成"内容落盘、指针留窗"的闭环。

## 3.4 过期清理的跨平台细节

```python
def cleanup_expired(self, retention_days: int = 5) -> int:
    for fp in tool_dir.glob("*.txt"):
        st = os.stat(fp)
        if sys.platform == "win32":
            ts = st.st_ctime          # Windows 用创建时间
        else:
            ts = getattr(st, "st_birthtime", st.st_mtime)  # macOS 有
            # birthtime，Linux 回退 mtime
```

三平台的文件创建时间获取各不相同（macOS `st_birthtime`、Linux 通常无、Windows `st_ctime`），这里逐一处理。工具结果文件默认只留 5 天——它们是一次性的大块头，保留价值远低于对话归档。

---

# Part 4: 视觉压缩——把文本渲染成图片

## 4.1 核心思想

`agents/context/visual_compression/` 实现了一条激进路线：**把长文本渲染成图片，用视觉模型的图像 token 替代文本 token**。

配置常量（`config.py`）揭示了机制：

```python
CANVAS_WIDTH = 1568              # 画布宽度
CANVAS_MAX_HEIGHT = 728          # 画布最大高度
IMAGE_PATCH_SIZE = 28            # 图像 patch 尺寸
IMAGE_COST_SAFETY_MARGIN = 1.10  # 成本安全边际
MAX_VISUAL_COST_RATIO = 0.90     # 视觉成本占比上限
CHARS_PER_TEXT_TOKEN_FALLBACK = 4.0  # 文本 token 估算基准
```

逻辑：视觉模型按 patch 计图像 token（28×28 像素一个 patch），一张 1568×728 的图约 `(1568/28) × (728/28) ≈ 1456` 个 token，而同等面积可渲染数万字符的等宽文本——当文本量超过临界点，**图片比文本便宜**。`IMAGE_COST_SAFETY_MARGIN` 与 `MAX_VISUAL_COST_RATIO` 保证只有真正划算时才切换。

## 4.2 三档 effort 预设

```python
EFFORT_PRESETS = {
    "low":    EffortPreset(cell_width=5, line_height=8,
                           readable_chars_per_image=28_080,
                           static_min_chars=2_000,
                           tool_result_min_chars=6_000,
                           history_keep_recent_messages=6),
    "medium": EffortPreset(cell_width=4, ...,
                           readable_chars_per_image=35_100, ...),
    "high":   EffortPreset(cell_width=3, line_height=7,
                           readable_chars_per_image=53_040, ...),
}
```

压缩强度越高，字符单元越小（字体更密）、单张图能容纳的字符越多（28080 → 53040）、触发压缩的门槛越低（6000 → 4000 字符）、保留的近期消息越少（6 → 4 条）。**策略参数全部代码持有（code-owned），不暴露给模型或用户配置篡改**。

## 4.3 压缩管线九大组件

```
visual_compression/
├── config.py              # 策略与预设（本文已析）
├── pipeline/
│   ├── budget.py          # token 预算分配
│   ├── messages.py        # 消息压缩
│   ├── tool_results.py    # 工具结果压缩
│   ├── tool_schemas.py    # 工具定义压缩
│   ├── static_context.py  # 静态上下文（系统提示等）压缩
│   ├── history.py         # 历史折叠（≥10 条起折，50 条网格化，10 条冻结网格）
│   ├── precision.py       # 数值/精度保持
│   ├── request.py         # 请求整体编排
│   └── receipt.py         # 压缩回执（记录压了什么、省了多少）
├── rendering/
│   └── renderer.py        # 文本 → 图像渲染
└── runtime/
    ├── middleware.py      # 运行时中间件接入点
    └── recovery.py        # 失败恢复（渲染失败回退文本）
```

几个值得注意的常量：

```python
FACTSHEET_MAX_ENTRIES = 96        # 事实清单最大条目
FACTSHEET_MAX_SCAN_CHARS = 262_144
MAX_IMAGES_PER_REQUEST = 64       # 单请求最多 64 张图
MAX_IMAGES_PER_TOOL_RESULT = 10   # 单工具结果最多 10 张
HISTORY_MIN_COLLAPSE_MESSAGES = 10  # 历史至少 10 条才折叠
```

`receipt.py`（压缩回执）是审计性设计：每次压缩产出记录——压了哪些内容、原成本、压缩后成本。压缩不能是黑箱。

---

# Part 5: 安全门禁与优雅降级

## 5.1 沙箱召回的双层门禁（default-deny）

`recall_history_python` 会执行**模型生成的 Python 代码**——这是高危操作。`scroll_unsandboxed_allowed()` 实现了双层门禁：

```python
_UNSANDBOXED_ENV = "QWENPAW_ALLOW_UNSANDBOXED_RECALL"

def scroll_unsandboxed_allowed(scroll_config) -> bool:
    # 第一层：部署层环境变量（只有能设置进程环境变量的运维能开）
    if os.environ.get(_UNSANDBOXED_ENV, "").lower() not in _TRUTHY:
        return False
    # 第二层：per-agent 配置
    return bool(getattr(scroll_config, "allow_unsandboxed", False))
```

注释把威胁模型讲得很透：

> running recall unsandboxed executes model-authored Python as the agent user with zero isolation. In a multi-tenant deployment an untrusted `agent.json` / API payload must never be able to turn the sandbox off on its own — that would be a privilege-escalation path.

**关键原则**：`agent.json`（可被租户控制的配置）里的 `allow_unsandboxed` 标志**只有**在部署层环境变量也放行时才生效。攻击者控制了租户配置也升不了权。两个开关缺一不可，默认拒绝。

## 5.2 任何失败都降级到 native

`build_scroll_components` 的整个装配体被一个大 try 包裹：

```python
try:
    # 懒导入 + 装配全部组件
    ...
    return ScrollComponents(...)
except Exception:  # noqa: BLE001 - any scroll failure degrades to native
    if history is not None:
        history.close()   # 清理半成品
    logger.warning("scroll: failed to wire components — "
                   "falling back to native context management")
    return None
```

降级安全的论证写在注释里：**native 模式把完整历史留在上下文中，所以降级永远是安全的**——最多是多花点 token，绝不会丢数据或崩构建。懒导入也是同一哲学：缺依赖包不是构建失败，而是静默回退。

## 5.3 容量与保留治理

| 机制 | 参数 | 默认 |
|------|------|------|
| history.db 自动清理 | `history_retention_days` | 30 天（0 = 永久） |
| 容量告警 | `_DB_SIZE_WARN_BYTES` | 1 GiB（含 WAL） |
| 工具结果清理 | `retention_days` | 5 天 |
| 摘要超时 | `_SUMMARY_UPDATE_TIMEOUT_SECONDS` | 60 秒 |

## 5.4 设计哲学总结

1. **摘要 ≠ 丢失**：continuation summary 是缓存，history.db 是权威。模型随时可以召回原文——这是对"上下文压缩必然损信息"的根本性反驳。
2. **写穿保证零丢失**：消息入窗即落盘，压缩只是窗口视图的变化。
3. **召回能力分级**：正门（结构化、无沙箱、总可用）+ 高级门（编程化、沙箱、按门禁开放）——可用性与能力分层供给。
4. **降级永远安全**：任何组件失败都回到 native，最坏结果只是成本上升。
5. **安全边界在部署层**：租户配置永远不能单独打开危险开关。
6. **成本意识贯穿**：从视觉压缩的"图片比文本便宜"计算，到输出预留比例，到工具结果指针化——每一个设计都在问"这值多少 token"。

---

## 附：上下文管理组件文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `context/base.py` | 51 | ContextManager Protocol |
| `context/types.py` | 53 | LogEntry / ExecutionResult / ContextWindowUnfitError |
| `context/__init__.py` | 240 | 装配入口 + 沙箱门禁 + 运维告警 |
| `context/scroll/manager.py` | 2025 | ScrollContextManager 中枢 |
| `context/scroll/history.py` | 760 | history.db 持久层 |
| `context/scroll/eviction_index.py` | 421 | 驱逐索引 |
| `context/scroll/memoryspace.py` | 2015 | ms 记忆空间（SQL 访问层） |
| `context/scroll/recall_tool.py` | 1060 | recall_history 正门 + RecallLoopGuard |
| `context/scroll/repl.py` | 303 | recall_history_python 沙箱 REPL |
| `context/scroll/continuation_summary.py` | 755 | 摘要生成与校验 |
| `context/scroll/sync.py` | 907 | 窗口 ↔ 持久层同步 |
| `context/scroll/serialize.py` | 347 | 消息序列化 |
| `context/scroll/prompt.py` | 203 | 召回提示词 |
| `context/visual_compression/` | ~1000 | 视觉压缩管线 |
| `agents/offloader.py` | 175 | Offloader 协议实现 |
