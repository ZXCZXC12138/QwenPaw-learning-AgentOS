# QwenPaw 本地模型服务（LocalModels）深度解析

> **来源**：基于 QwenPaw 代码仓 `src/qwenpaw/local_models/`（6 文件，2819 行）全量源码分析：`llamacpp.py`（935 行）、`model_manager.py`（654 行）、`download_manager.py`（598 行）、`tag_parser.py`（373 行）、`manager.py`（243 行）
> **分析范围**：llama.cpp 运行时托管（下载 / 安装 / 启动 / 健康检查 / 关停）、GGUF 模型下载（双源 + 自动降级 + 独立进程）、推荐模型与内存预估、原始文本输出的标签解析（think / tool_call）、本地配置持久化
> **本文档包含六部分内容**：
> 1. 门面与四件套的职责切分
> 2. llama.cpp 运行时：从二进制下载到服务就绪
> 3. 模型下载：独立进程、双源降级、staging 转正
> 4. 推荐模型：按机器能力给建议
> 5. 标签解析：把原始文本翻译回结构化事件
> 6. 设计哲学总结

---

## 目录

### Part 1: 门面与四件套
- LocalModelManager：单一入口
- 组件间依赖注入

### Part 2: llama.cpp 运行时
- 二进制自托管：镜像下载与版本钉死
- 安装前置检查与系统门槛
- 端口协商、健康轮询、atexit 兜底

### Part 3: 模型下载
- 独立进程下载：不阻塞应用事件循环
- HuggingFace → ModelScope 的 AUTO 降级
- 下载状态七态机与可取消性

### Part 4: 推荐模型
- 推荐清单 × 本地已下载状态
- 内存探测与体积预估

### Part 5: 标签解析
- `<think>` / `<tool_call>` 双标签
- 流式场景下的不完整块处理

### Part 6: 设计哲学总结

---

# Part 1: 门面与四件套的职责切分

`local_models/` 的目标是让用户**不装任何东西**就能在本地跑模型：运行时（llama.cpp）和模型权重都由应用代下载、代管理、代启停。代码分四件套 + 一个门面：

```
manager.py               LocalModelManager（门面，单例）
  ├─ llamacpp.py         LlamaCppBackend：llama.cpp 二进制的下载/安装/服务生命周期
  ├─ model_manager.py    ModelManager：GGUF 模型权重下载与清单
  ├─ download_manager.py 共享的下载状态类型与进度追踪（两者复用）
  └─ tag_parser.py       原始文本输出的标签解析
```

门面的构造展示了依赖注入的纪律：

```python
def __init__(self, *, model_manager=None, llamacpp_backend=None):
    self._model_manager = model_manager or ModelManager()
    self._llamacpp_backend = llamacpp_backend or LlamaCppBackend()
    self._server_lifecycle_lock = asyncio.Lock()
```

两个组件可注入，测试里塞假实现即可；`_server_lifecycle_lock` 把服务启停串行化——**本地服务的启动/关停是状态迁移，并发迁移必须排队**。本地运行时配置持久化为 `LocalModelConfig`（`config.json`）：`max_context_length` 默认 65536（下限 32768）、`port` 为 `None` 表示自动选端口。

# Part 2: llama.cpp 运行时：从二进制下载到服务就绪

## 2.1 二进制自托管：版本钉死在代码里

```python
DEFAULT_LLAMA_CPP_BASE_URL = (
    # Mirror of "https://github.com/ggml-org/llama.cpp/releases/download"
    "https://download.qwenpaw.agentscope.io/files/models/llama_cpp"
)
DEFAULT_LLAMA_CPP_RELEASE_TAG = "b8744"
```

两个决策：① 从自建镜像下载而不是直连 GitHub——下载成功率是硬需求，注释标明镜像源；② 发行版号 `b8744` 硬编码——**运行时二进制是应用的一部分，版本必须随应用发版走**，`has_update(latest_version)` 提供升级检查。

## 2.2 安装门槛先检查

`check_llamacpp_installation`（装没装）与 `check_llamacpp_installability`（能不能装）是两个不同的问题——后者检查系统门槛，如 `_MIN_MACOS_VERSION = (13, 3)`。**不能装的机器要在下载前被告知，而不是装完跑不起来**。

## 2.3 服务生命周期

`setup_server` 是完整的启动链：解析模型文件（`_resolve_model_file`）→ 端口协商（`_resolve_server_port`：配置指定端口先验可用性 `_is_port_available`，否则 `_find_free_port` 绑定探测）→ `_create_server_process` 拉起进程 → `server_ready` 轮询健康端点 → 返回 `LlamaCppServerSetupResult(port, model_info)`。配套工程：

- **日志排空**：`_drain_server_logs` 后台任务持续消费子进程输出，避免管道缓冲写满导致子进程卡死；`_cancel_server_log_task` 负责收尾
- **退出兜底**：`atexit.register(self._shutdown_server_at_exit)`——应用退出时同步关停服务进程（快速路径 `kill_timeout=1.0`），正常关停给 3.0 秒优雅窗口。**孤儿 llama.cpp 服务占着显存是最差的用户体验**
- **状态可查**：`get_server_status` / `is_server_transitioning` 让前端知道服务处于稳定态还是迁移中

# Part 3: 模型下载：独立进程、双源降级、staging 转正

## 3.1 下载跑在独立进程

几个 GB 的模型下载放在应用事件循环里是自杀行为。方案是 `multiprocessing`：`ProcessDownloadTask` + `ProcessDownloadTaskSpec` 定义任务，`_download_worker` 是子进程入口，进度通过 `mp.Queue` 回流——`DownloadProgressTracker` 在主进程侧消费，`DownloadTaskMessageType` 分 `PROGRESS` / `RESULT` 两类消息。`_drain_queue_message` 处理 `Empty` 异常——**队列协议对进程崩溃也要有说法**（worker 内 `traceback` 全文回传给主进程）。

## 3.2 下载源：AUTO 降级

```python
class DownloadSource(str, Enum):
    HUGGINGFACE = "huggingface"
    MODELSCOPE = "modelscope"
    # First try Hugging Face, then fall back to ModelScope if unreachable
    AUTO = "auto"
```

`AUTO` 模式先探测 HuggingFace 可达性（`_probe_huggingface`），不通自动切 ModelScope——**网络环境是用户属性，不是错误**。每个源各有配套的体积预估（`_estimate_huggingface_size` / `_estimate_modelscope_size`）与 GGUF 存在性预检（`_check_*_gguf_exists`）——下载开始前就知道"有没有这个文件、大概多大"。

## 3.3 状态机与可取消性

`DownloadTaskStatus` 七态：`idle → pending → downloading → (canceling) → completed / failed / cancelled`。`CANCELING` 是独立中间态——**取消是异步过程，不是布尔翻转**。`cancel_download` 通知子进程，`_finalize_download_result` 统一处理终态。落盘走 staging 转正：`_promote_staging_directory` 先下到临时目录、完整后才转正——**半个文件不许以"已下载"的身份出现在清单里**（`_check_gguf_exists` 与 `is_downloaded` 只认完整品）。另有 `_detect_available_memory_gb` 探测可用内存——为推荐决策提供硬件事实。

# Part 4: 推荐模型：按机器能力给建议

`ModelManager.get_recommended_models` 返回 `list[LocalModelInfo]`，并逐个标注本地下载状态（"check local download status for each recommended model"）。`LocalModelInfo` 继承自 `providers.ModelInfo`——**本地模型条目与云端模型条目是同一个类型家族**，只扩展 `size_bytes` 等本地专属字段。这个类型复用意义不小：本地模型在模型清单、Provider 面、UI 展示里天然同权，不需要一套平行的"本地模型"概念。

`download_model` 与 `start_download` 分离（不同调用面的入口共用同一套底层），`remove_downloaded_model` 删除时连同 `_cleanup_path` 清理残余——**下载能取消，模型能删除，磁盘卫生有人管**。

# Part 5: 标签解析：把原始文本翻译回结构化事件

本地模型（如 Qwen3-Instruct）没有云端那样的原生结构化输出——推理和工具调用都以文本标签形式嵌在原始输出里。`tag_parser.py` 模块文档：

> Handles `<think>...</think>` (reasoning) and `<tool_call>...function calling) tags that local models like Qwen3-Instruct embed in their raw text output.

实现要点：

- **双标签正则**：`_THINK_RE` / `_TOOL_CALL_RE` 均为非贪婪 + `re.DOTALL`——标签块可以跨行，且多个块各自独立匹配，不许吞掉中间正文。
- **工具调用体是 JSON**：`<tool_call>` 内容按 JSON 解析，配 `uuid` 生成调用 ID——翻译产物与云端原生 tool_call 事件同构，**上层 ReAct 循环不需要知道模型是本地的还是云端的**。
- **流式友好**：解析器面向增量文本设计——流式输出中标签可能只到了一半（`<tool_call>` 开了还没闭合），未闭合的块必须留在缓冲里等下一个 chunk，不许当成正文吐给用户。
- **协议对齐的价值**：Qwen3 系列模型的标签约定与云端工具调用协议对齐，本地与云端走同一条工具执行链路——**本地模型不是功能降级的二等公民**。

# Part 6: 设计哲学总结

1. **零安装承诺靠全托管兑现**：运行时二进制与模型权重都由应用代下载、代安装、代启停——用户侧没有前置步骤。
2. **运行时版本是应用的一部分**：llama.cpp 发行版号钉在代码里，下载走自建镜像——升级随应用发版，不做运行时的"最新即最好"。
3. **装不上要先说**：`installability` 与 `installation` 是两个问题，系统门槛检查在下载之前——别让用户下了两个 GB 才收到坏消息。
4. **重活进独立进程**：下载跑在 `multiprocessing` 子进程，进度走队列协议——应用事件循环永远保持响应；队列协议对子进程崩溃也有明确说法。
5. **网络环境是用户属性**：HuggingFace 不通自动降级 ModelScope，体积与存在性预检先行——下载前就有完整预期。
6. **完整性不许打折**：staging 转正、七态状态机、取消是异步过程——半途的文件、半途的状态都不许冒充完成态。
7. **退出路径负责到底**：`atexit` 兜底关停 + 优雅/快速两级超时——孤儿推理服务占显存是必须消灭的事故形态。
8. **本地模型与云端同构**：`LocalModelInfo` 继承统一模型类型，标签解析把原始文本翻译回标准事件——上层链路不感知模型在哪里跑。

---

## 附：关键文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `local_models/llamacpp.py` | 935 | llama.cpp 二进制管理与服务生命周期 |
| `local_models/model_manager.py` | 654 | GGUF 模型下载、推荐清单、内存探测 |
| `local_models/download_manager.py` | 598 | 下载状态类型与进度追踪（复用件） |
| `local_models/tag_parser.py` | 373 | think / tool_call 标签解析 |
| `local_models/manager.py` | 243 | LocalModelManager 门面与配置持久化 |
| `utils/command_runner.py` | — | 托管子进程：启动/关停/超时 |
