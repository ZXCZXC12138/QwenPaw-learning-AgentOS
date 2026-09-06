# QwenPaw 多 OS 沙盒（macOS / Linux / Windows）深度解析

> **来源**：基于 QwenPaw 代码仓 `src/qwenpaw/sandbox/`（9 个 Python 文件，约 9700 行）全量源码分析
> **分析范围**：五种隔离模式（Seatbelt / Bubblewrap / Landlock / Windows 三件套 / None）、`SandboxConfig` 约束词汇表、`report_unenforced_config` 约束透明机制、平台能力探测、macOS Seatbelt 配置文件编译、Linux Landlock 裸 syscall、Windows WRITE_RESTRICTED / AppContainer 原生隔离
> **本文档包含五部分内容**：
> 1. 统一抽象：一份配置，五种后端
> 2. 约束透明：未被执行的约束不许静默消失
> 3. macOS：Seatbelt 配置文件编译
> 4. Linux：Bubblewrap 优先、Landlock 裸 syscall 兜底
> 5. Windows：7200 行原生隔离工程

---

## 目录

### Part 1: 统一抽象
- `SandboxMode` 五模式与工厂分发
- 每工具调用一次的生命周期
- 白名单模型：未声明即拒绝

### Part 2: 约束透明
- `report_unenforced_config`：安全边界字段按 WARNING 报告
- 网络域名过滤的诚实降级（fail-OPEN 必须被看见）
- 挂载路径缺席的补漏报告

### Part 3: macOS Seatbelt
- deny-default 配置文件编译
- 路径注入防护
- 违规检测：拒绝宽泛子串匹配

### Part 4: Linux
- Bubblewrap 挂载命名空间（首选）
- Landlock：ctypes 裸 syscall 三步走
- 端口规则的 ABI v4 前提与如实报告

### Part 5: Windows
- 三后端分发：AppContainer / Elevated / Unelevated
- WRITE_RESTRICTED 令牌与伪造能力 SID
- 共享基建与退出清理
- 设计哲学总结

---

# Part 1: 统一抽象：一份配置，五种后端

## 1.1 五模式与工厂分发

`SandboxMode` 枚举五种隔离模式，`create_sandbox` 工厂按模式 + 平台条件分发：

```
SEATBELT     macOS sandbox-exec（Seatbelt 配置文件）
BUBBLEWRAP   Linux bubblewrap 挂载命名空间（首选）
LANDLOCK     Linux Landlock LSM（5.13+，兜底）
WINDOWS      Windows 10+ 原生，按两个条件二次分发：
               allow_read_all=False        → AppContainer
               allow_read_all=True + 管理员 → Elevated（专用账户）
               allow_read_all=True + 无管理员 → Unelevated
NONE         无隔离直接执行
```

`probe_sandbox_support()` 在启动时探测平台能力，返回 `SandboxCapability`（是否支持、检测到的模式、人读原因、Linux 的 Landlock ABI 版本）——**用哪种沙盒不是配置拍脑袋，是探测出来的事实**。

## 1.2 生命周期与约束词汇

模块文档一句话定了生命周期：**per-tool-call——每次工具调用创建并销毁一个沙盒**。隔离单元精确到单次命令执行，不存在跨调用共享的长寿沙盒。

约束词汇表由 `SandboxConfig` 定义，白名单模型（"unlisted paths and capabilities are denied"）：

| 约束 | 说明 |
|------|------|
| `mounts` | 路径权限声明（`MountSpec`：path + writable + executable） |
| `allow_read_all` | True = deny-list 模式；False = 仅 mounts 可读的 allow-list 模式 |
| `deny_paths` | 敏感路径显式拒绝，无视其他一切设置（如 `~/.ssh`） |
| `network_allow` | 域名白名单；`["*"]` 全开、`[]` 全关；域名级过滤是尽力而为 |
| `network_ports` | TCP 端口级控制，Linux Landlock ABI v4 原生支持 |
| `max_processes` / `max_memory_mb` | **任何后端今天都不执行**——值被接受并忽略，TODO 写明需 cgroups / Job objects |
| `env_mode` | `"inject"` 可用；`"allowlist"` **未实现**，所有后端实际行为都是 inject |

注意后三条：字段存在但做不到的事，docstring 直接写 "NOT ENFORCED" / "NOT IMPLEMENTED" 并给出技术原因——**配置面不假装自己更强**。

# Part 2: 约束透明：未被执行的约束不许静默消失

这是整个沙盒模块最闪光的机制。`report_unenforced_config` 的 docstring 原文：

> Silently dropping a constraint is worse than not offering it at all: the caller gets a weaker sandbox than it asked for with nothing in the log to say so. Each backend declares the fields it applies and this surfaces the remainder, so **adding a backend that forgets a field is loud rather than invisible**.

机制分四层：

**① 每后端自报执法面。** 每个后端声明 `_ENFORCED_FIELDS`（本配置下真正应用的字段集合），工厂比对调用方实际请求的约束（`_requested_constraints` 只统计非默认值的字段），差集逐项报告。

**② 安全边界字段按严重级分层。** `_SECURITY_BOUNDARY_FIELDS`（mounts / deny_paths / network_allow / network_ports / env_mode / platform_hints ...）被丢弃时按 **WARNING** 报告，其余按 DEBUG。注释给出理由：操作员配了 `deny_paths` 看到干净日志会合理推断路径已受保护——静默丢弃制造的是**虚假的安全感**；`env_mode=allowlist` 虽未实现，其意图正是把 API key、云凭据挡在沙盒外，丢弃它等于凭据泄漏边界失守。

**③ 网络降级的方向必须被看见。** 所有后端都无法按域名过滤网络，降级方向是 **fail-OPEN**（全开）而非 fail-closed。`NETWORK_DOMAIN_HINT` 原文："Domain-level filtering is unavailable on every backend; all network access is ALLOWED."——这句话会出现在每一次相关报告里，操作员不可能错过。

**④ 字段外的实践层补漏。** `_report_missing_mount_paths`：一个字段原则上被执法，不代表具体路径真的被绑定——挂载路径启动时不存在就会被跳过，这条单独报告（"路径只有在沙盒启动时存在才会被绑定"）。注释还解释了为什么不是错误：`~/.cache/uv` 这类工具缓存在工具首跑前合法地不存在。

# Part 3: macOS：Seatbelt 配置文件编译

## 3.1 deny-default 配置编译

`MacOSSandbox`（350 行）把 `SandboxConfig` 编译成 Seatbelt `.sb` 配置文件，经 `sandbox-exec -p '<profile>'` 在内核中强制执行：

```
基础系统路径只读（/System、/usr/lib、/usr/share、/Library、/dev）
workspace_dir 读写
mounts 声明的路径按 writable 定读写
~/.ssh 等敏感路径显式 deny
网络由 network_allow 的绝对形态（全开/全关）控制
```

注释点明 Seatbelt 的能力边界：网络只能全开或全关、无法按域名过滤，所以域名白名单**报告为被忽略（网络最终全开），而不是静默降级**——与 Part 2 的透明机制一脉相承。

## 3.2 路径注入防护

配置文件是把路径字符串嵌进 S 表达式，`_sanitize_seatbelt_path` 专门防 Seatbelt 规则注入：转义双引号、反斜杠、括号；含换行符的路径直接 `ValueError`（"no valid filesystem path contains them"）。

## 3.3 违规检测的精确性

检测沙盒违规的正则值得细读，注释记录了为什么不能用宽泛匹配：

> Substring matching against generic words like `"deny"` or `"sandbox"` is far too lossy — application logs routinely contain those tokens and would be mis-flagged as sandbox violations.

只匹配 Seatbelt 内核真正输出的诊断形态：`deny(1) file-read-data ...`、`Sandbox: <bin>(<pid>) deny(1) ...`、`sandbox-exec:` 前缀、完整的 `Operation not permitted` 短语。并且 TODO 诚实标注：这仍是启发式，健壮方案应读 Endpoint Security 的结构化事件流而非刮 stderr。

另外 `_SUPPORTED_HINT_KEYS = frozenset({"seatbelt_extra_rules"})`：platform_hints 只有这一个键会被编译进配置文件，其他键（包括拼错的）**一律丢弃且不宣称 platform_hints 已被执法**——"err loud"：报告列出全部键名，让拼写错误可被发现。

# Part 4: Linux：Bubblewrap 优先、Landlock 裸 syscall 兜底

## 4.1 Bubblewrap（首选）

`BubblewrapSandbox`（281 行）用挂载命名空间隔离——文件系统约束在挂载层完成，语义清晰且无需内核特性探测，故为首选。

## 4.2 Landlock：ctypes 直达内核

`linux_sandbox.py`（821 行）在没有 bubblewrap 的环境下走 Landlock LSM，子进程执行序列：

```
prctl(PR_SET_NO_NEW_PRIVS) → landlock_create_ruleset → landlock_add_rule
→ landlock_restrict_self → exec
```

全部通过 **ctypes 裸 syscall** 完成（`SYS_LANDLOCK_CREATE_RULESET = 444` 等），不依赖第三方绑定。代码里完整铺开了 ABI v1–v4 的访问权限位掩码（`LANDLOCK_ACCESS_FS_*`、v2 加 `REFER`、v3 加 `TRUNCATE`、v4 加 `NET_BIND_TCP/NET_CONNECT_TCP`），按探测到的 ABI 版本组合掩码。

能力探测（`_probe_linux_landlock`）三步：内核版本 ≥ 5.13 → `/sys/kernel/security/lsm` 含 "landlock" → `landlock_create_ruleset(NULL, 0, VERSION)` 拿 ABI 版本。每步失败都有独立的人读原因。

端口规则有一个精细的如实报告（`_LANDLOCK_PORT_HINT`）：

> Landlock port rules require ABI v4+ AND network_allow=[] (the wholesale block); with the network open no port rule is installed.

端口规则挂在和整体网络封锁相同的访问掩码上——网络开着的时候端口规则无处可挂，所以**不安装并明确告知原因**，而不是装作已生效。

# Part 5: Windows：7200 行原生隔离工程

Windows 一家占了全模块 74% 的代码量（三个文件共约 7160 行），因为 Windows 没有现成的轻量沙盒命令，全部要自己用 Win32 API 搭。

## 5.1 三后端分发

| 后端 | 触发条件 | 机制 |
|------|----------|------|
| `WindowsAppContainerSandbox` | `allow_read_all=False` | AppContainer SID（`S-1-15-2-*`）：**只有带容器 SID 显式 ACE 的路径可访问**，其余全拒 |
| `WindowsElevatedSandbox` | `allow_read_all=True` + 管理员 | 专用本地账户 + WRITE_RESTRICTED 令牌 + WFP 防火墙规则 |
| `WindowsUnelevatedSandbox` | `allow_read_all=True` + 无管理员 | 从当前进程令牌派生 WRITE_RESTRICTED，无需管理员 |

## 5.2 WRITE_RESTRICTED 的核心技巧

Unelevated 后端的模块注释点破机制：

> Write access is gated by a **fabricated capability SID**; read/execute access is unrestricted.

WRITE_RESTRICTED 令牌的语义是"写操作要过 restricting SID 列表检查，读执行走正常 DACL"。QwenPaw **伪造一个随机能力 SID**，只把它授给允许写入的路径的 ACE——写面变成一个精确的白名单，读面保持全量。Elevated 后端更进一步：专用账户把爆炸半径从当前用户缩小到一个临时身份。

## 5.3 共享基建与退出清理

`windows_unelevated_sandbox.py` 同时是三个后端的共享地基：`WindowsSandboxBase` 基类、ctypes 结构体（`_PROCESS_INFORMATION` / `_STARTUPINFOW` / `_SID_AND_ATTRIBUTES`）、SID/令牌/ACL 助手、stdio 管道、Job object、管道输出解码。AppContainer 与 Elevated 后端大量复用这些私有函数——**一处基建，三处复用**。

细节清单：违规检测正则包含**中文区域设置的 "Access is denied" 等价文案**（`_WC.VIOLATION_RE`）；孤儿元数据巡检（`_iter_orphaned_metadata`）；每个后端自带 `shutdown_cleanup`，`shutdown_all_sandboxes()` 在应用退出时依次清理三种后端的全部工件（非 Windows 平台 no-op、可重复调用）。

## 5.4 设计哲学总结

1. **隔离单元精确到单次工具调用**：per-tool-call 生命周期，没有共享长寿沙盒。
2. **约束不许静默消失**：每后端自报执法面，差集按字段严重级报告；安全边界字段丢失按 WARNING 炸出来——"Adding a backend that forgets a field is loud rather than invisible"。
3. **降级方向必须被看见**：网络域名过滤做不到就明说全开（fail-OPEN），端口规则挂不上就明说不装——假保护比无保护更危险。
4. **能做到的与做不到的都写进 docstring**：`max_processes` / `max_memory_mb` / `env_mode=allowlist` 标注 NOT ENFORCED 并附技术原因——配置面不贩卖虚假安全感。
5. **平台原生机制用到底**：Seatbelt 配置文件编译 + 注入防护、Landlock ctypes 裸 syscall 按 ABI 版本组掩码、Windows 伪造能力 SID 把写面变成白名单。
6. **检测要精确**：违规识别只匹配内核/系统的真实诊断形态，宽泛关键词会造成误报，注释写明启发式的边界与理想方案。

---

## 附：关键文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `windows_unelevated_sandbox.py` | 2986 | Windows 共享基建 + 非提权沙盒 |
| `windows_elevated_sandbox.py` | 2928 | 提权沙盒（专用账户 + WFP） |
| `windows_appcontainer_sandbox.py` | 1247 | AppContainer 沙盒 |
| `linux_sandbox.py` | 821 | Landlock LSM 沙盒 |
| `config.py` | 763 | 配置、能力探测、工厂、约束透明 |
| `macos_sandbox.py` | 350 | Seatbelt 沙盒 |
| `bubblewrap_sandbox.py` | 281 | Bubblewrap 沙盒（Linux 首选） |
| `local_sandbox.py` | 255 | LocalSandbox 基类与 NoneSandbox |
