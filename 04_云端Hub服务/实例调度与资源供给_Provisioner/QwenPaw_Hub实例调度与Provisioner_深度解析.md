# QwenPaw Hub 实例调度与资源供给（Provisioner）深度解析

> **来源**：基于 QwenPaw 代码仓 `src/qwenpaw/hub/` 全量源码分析（26 个文件，约 6800 行）
> **分析范围**：`control_app.py`（控制面 API，1545 行）、`service.py`（RuntimeService）、`registry.py`（SQLite 注册表）、`provisioner.py` + `docker_provisioner.py` + `local_provisioner.py`（供给层）、`process_isolation.py`（OS 进程隔离）、`access_security.py` / `proxy_limits.py`（网络防护）
> **本文档包含五部分内容**：
> 1. Hub 控制面总体架构
> 2. RuntimeProvisioner 契约与双供给后端
> 3. 进程隔离层：三平台 Isolator
> 4. 注册表、租户与个人运行时
> 5. 网络安全防护：限流与代理约束

---

## 目录

### Part 1: 控制面总体架构
- Hub 是什么：多租户的 QwenPaw 实例管理器
- 组件清单与职责分层

### Part 2: RuntimeProvisioner 契约
- 六方法抽象契约
- preflight：不启动实例先探测安全边界
- Docker 供给器：镜像策略与容器生命周期
- Local 供给器：OS 沙箱进程树
- security_level 分级体系

### Part 3: 进程隔离层
- ProcessIsolator 抽象
- Linux Bubblewrap / macOS Seatbelt 实现
- UnsupportedIsolator：失败关闭而非开放

### Part 4: 注册表与个人运行时
- SQLite RuntimeRegistry：期望状态与观察状态
- RuntimeService：生命周期编排与并发锁
- 个人运行时（personal runtime）模型

### Part 5: 网络安全防护
- HubAccessSecurity：IP 级限流与黑名单
- proxy_limits：流式代理的尺寸与空闲约束
- 设计哲学总结

---

# Part 1: 控制面总体架构

## 1.1 Hub 是什么

`hub/` 是 QwenPaw 的**多租户控制面**：把"一个 QwenPaw 实例"抽象成可创建、启停、重建、审计的 **Runtime**，由 Hub 统一供给和监管。它是 QwenPaw 从"单机桌面应用"走向"多用户云服务"的关键模块。

```
┌─────────────────────────────────────────────────────────────┐
│                    control_app.py (FastAPI)                  │
│   认证 / 用户管理 / Runtime API / 审计 / 代理                 │
├─────────────────────────────────────────────────────────────┤
│                 service.py  RuntimeService                    │
│   create / start / stop / restart / rebuild / delete          │
│   per-runtime 锁 · preflight · provisioner 策略校验           │
├────────────────────────────┬────────────────────────────────┤
│  docker_provisioner.py     │  local_provisioner.py          │
│  容器供给（共享内核隔离）    │  本地进程树（OS 沙箱隔离）       │
├────────────────────────────┴────────────────────────────────┤
│       process_isolation.py  ProcessIsolator 抽象              │
│   LinuxBubblewrap / MacOSSeatbelt / Unsupported              │
├─────────────────────────────────────────────────────────────┤
│  registry.py (SQLite) · database.py · credentials.py         │
│  access_security.py · proxy_limits.py · websocket_proxy.py   │
└─────────────────────────────────────────────────────────────┘
```

## 1.2 组件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `control_app.py` | 1545 | FastAPI 控制面：认证、用户、Runtime API、审计 |
| `service.py` | 485 | RuntimeService 生命周期编排 |
| `docker_provisioner.py` | 552 | Docker 容器供给器 |
| `local_provisioner.py` | 477 | 本地进程供给器 |
| `process_isolation.py` | 452 | 三平台进程隔离器 |
| `database.py` / `registry.py` | 864 | SQLite 持久化 + Runtime 注册表 |
| `auth.py` / `credentials.py` | 841 | 用户认证 + 凭据管理 |
| `config.py` | 531 | Hub 配置模型 |
| `access_security.py` | 138 | IP 限流 / 黑名单 / 可信代理 |
| `proxy_limits.py` / `websocket_proxy.py` | 180 | 代理流量约束 |
| `windows_*.py`（3 个） | 639 | Windows 专属：进程隔离、反向隧道、运行时桥 |

# Part 2: RuntimeProvisioner 契约

## 2.1 六方法抽象契约

`provisioner.py` 定义供给器接口：

```python
class RuntimeProvisioner(ABC):
    """Manage runtime lifecycle without exposing deployment internals."""
    name: str
    security_level: str

    def configure(self, config): ...        # 热更新后端配置（不重启 Hub）
    def validate_config(self, value): ...   # 归一化单运行时的后端配置

    @abstractmethod
    def preflight(self, root_dir) -> RuntimeProvisionerAvailability: ...
    @abstractmethod
    def start(self, record, credentials) -> RuntimeRecord: ...
    @abstractmethod
    def stop(self, record) -> RuntimeRecord: ...
    @abstractmethod
    def status(self, record) -> RuntimeRecord: ...
    @abstractmethod
    def close(self) -> None: ...
```

两个设计点：

1. **"不暴露部署细节"**（docstring 原话）：上层服务只谈 Runtime 生命周期，不知道下面是容器还是进程——供给技术可替换。
2. **`preflight` 独立于 `start`**：

> Probe the **real runtime boundary** without launching QwenPaw.

预检探测的是**真实的隔离边界**（沙箱可执行文件是否存在、Docker daemon 是否可用），而不是启动实例后再发现不行。返回值 `RuntimeProvisionerAvailability(available, reason)` 带原因——失败可解释。

## 2.2 Docker 供给器

```python
class DockerRuntimeProvisioner(RuntimeProvisioner):
    name = "docker"
    security_level = "isolated-container-shared-kernel"
```

镜像策略：

```python
DOCKER_HUB_IMAGE = "docker.io/agentscope/qwenpaw"
ALIYUN_ACR_IMAGE = ("agentscope-registry.ap-southeast-1."
                    "cr.aliyuncs.com/agentscope/qwenpaw")
OFFICIAL_DOCKER_IMAGES = {"docker_hub": ..., "aliyun_acr": ...}
OFFICIAL_DOCKER_TAGS = ("latest", "pre")
PULL_POLICIES = frozenset({"always", "if_not_present", "never"})
_IMAGE_PATTERN = re.compile(r"^[A-Za-z0-9][A-Za-z0-9._:/@-]{0,511}$")
```

细节解读：

- **双官方镜像源**：Docker Hub + 阿里云 ACR（新加坡区）——为不同网络环境的用户提供可达的镜像源
- **`pre` 标签**：预发布通道，`latest` / `pre` 双轨
- **三种拉取策略**：`always`（总拉最新）/ `if_not_present`（本地优先）/ `never`（仅本地）——离线部署可用 `never`
- **镜像名正则校验**：511 字符上限的严格字符集——防止镜像名注入
- **超时配置**：启动 90 秒（含镜像拉取）、停止 20 秒

## 2.3 Local 供给器

```python
class LocalProcessRuntimeProvisioner(RuntimeProvisioner):
    name = "local"
    security_level = "isolated-local-required"

    def __init__(self, *, isolator=None, ...):
        self._isolator = isolator or platform_process_isolator()
        if self._isolator.name == "macos-seatbelt":
            self.security_level = "isolated-local"
        elif self._isolator.name == "linux-bubblewrap":
            ...
```

Local 供给器在宿主机上起**完整的 QwenPaw 进程树**，但必须包在 OS 沙箱里。注意 `security_level` 是**动态的**：初始为 `isolated-local-required`（要求隔离），探测到具体沙箱后升级为对应的确定级别——**安全等级由实际探测结果决定，不由配置宣称**。

`allocate_loopback_port()` 用"绑定 0 端口再读回"的经典手法向操作系统要空闲端口，避免端口竞争。

## 2.4 security_level 分级体系

| 级别 | 供给器 | 含义 |
|------|--------|------|
| `isolated-container-shared-kernel` | docker | 容器隔离，与宿主共享内核 |
| `isolated-local` | local（Seatbelt/Bubblewrap 可用） | 本地进程 + OS 沙箱 |
| `isolated-local-required` | local（沙箱未探测到） | 要求隔离但未确认 |

这个分级直接服务控制面的策略判断（`RuntimeService.security_level(provisioner_name)` / `require_provisioner_available()`）——**能不能提供服务，取决于能不能提供承诺的安全边界**。

# Part 3: 进程隔离层

## 3.1 ProcessIsolator 抽象

```python
class ProcessIsolator(ABC):
    @abstractmethod
    def prepare(self, ...): ...   # 准备隔离环境
    @abstractmethod
    def launch(self, ...) -> IsolatedLaunch: ...  # 生成隔离启动
    def release(self, runtime_id): ...  # 释放
```

配套 `ManagedProcess` Protocol（poll / terminate / kill / wait）——隔离进程的生命周期操作被抽象成统一接口。

## 3.2 三平台实现

**LinuxBubblewrapIsolator**：包装 `bwrap`，`prepare()` 构建挂载/命名空间参数，`_probe()` 探测 bwrap 可用性（对应 CI 里安装 `bubblewrap` + 关 AppArmor userns 限制的操作）。

**MacOSSeatbeltIsolator**：包装 `/usr/bin/sandbox-exec`，核心是 `_profile()` 动态生成 SBPL 沙箱档案。特别的是它有 **`_probe_loopback_denied()`**——探测沙箱是否正确拒绝了环回网络越权，即**用攻击视角验证防御生效**，而不只是"档案加载成功"。

**UnsupportedProcessIsolator**：平台无可用沙箱时的兜底实现——但它的 `prepare()` 会**拒绝执行**（携带原因）。

```python
class UnsupportedProcessIsolator(ProcessIsolator):
    def __init__(self, reason: str): ...
    def prepare(self, ...):  # → 抛错，不提供裸奔的进程
```

**失败关闭（fail-closed）**：没有沙箱就没有供给，绝不降级为"裸进程"。`platform_process_isolator()` 工厂按平台选择实现。

# Part 4: 注册表与个人运行时

## 4.1 SQLite RuntimeRegistry

`registry.py` 用 SQLite 持久化 Runtime 元数据，表结构字段揭示了数据模型：

```
runtimes(runtime_id, tenant_id, owner_user_id, runtime_type,
         provisioner, desired_state, observed_state,
         endpoint_json, storage_json, config_json, status_json,
         metadata_json, revision, created_at, updated_at, observed_at)
```

关键设计：

- **期望状态（desired_state）与观察状态（observed_state）分离**：声明式管理——用户表达期望，控制面持续协调观察值向期望收敛（K8s 式调谐循环的思想）
- **`revision` 乐观锁字段**：并发更新冲突检测
- **`ensure_tenant()`**：每次写入前确保租户记录存在——多租户是一等公民
- **JSON 列存复杂结构**（endpoint/storage/config/status/metadata）：关系模型管身份与状态，文档模型管配置细节

## 4.2 RuntimeService 生命周期编排

`service.py` 的方法集：

```
create / list / list_page / get / start / stop / restart /
rebuild / status / delete / close
```

并发与策略控制：

- **`_runtime_lock(runtime_id)`**：每个 Runtime 一把 RLock，生命周期操作互斥（`_start_locked` / `_create_locked`）
- **`_preflight_provisioners()`**：启动前预检所有供给器
- **`_validate_provisioner_policy()`**：策略校验（哪些供给器被允许）
- **`require_provisioner_available()`**：供给器不可用时直接拒绝创建——不给用户"创建了但起不来"的运行时

`rebuild` 与 `restart` 的区分体现了供给器的抽象价值：重建是销毁后按规格再造（存储可能保留），重启是进程级操作——两者对上层是同一个 API 面。

## 4.3 个人运行时（personal runtime）模型

`control_app.py` 里的产品化设计：

```python
def personal_tenant_id(user: HubUser) -> str: ...
async def personal_runtime(user: HubUser) -> RuntimeRecord: ...
async def ensure_personal_runtime(user: HubUser) -> RuntimeRecord: ...
```

**每个用户有一个专属的个人运行时**——`ensure_personal_runtime` 按需创建。租户模型上个人用户自成租户，与组织租户共用同一套基础设施。配套的权限链：

```
require_user → require_admin           # 用户/管理员分级
require_runtime_access                 # Runtime 级访问控制
validate_credential_scope              # 凭据作用域校验
record_audit                           # 全操作审计
require_loopback_runtime               # 只允许回环地址的运行时
```

# Part 5: 网络安全防护

## 5.1 HubAccessSecurity：IP 级限流

```python
class HubAccessSecurity:
    """Resolve client addresses and enforce configurable request limits."""

    def __init__(self, config, *, clock=time.monotonic):
        self._attempts: dict[tuple[str, str], deque[float]] = {}
        self._blocked_until: dict[tuple[str, str], float] = {}
```

机制：

- **滑动窗口限流**：每（IP, 动作）维护尝试时间戳队列（`deque`），超阈值即封禁到 `blocked_until`
- **IP 黑名单**：支持 IPv4/IPv6 网段（`ipaddress.IPv4Network | IPv6Network`）
- **可信代理白名单**：`client_ip()` 只信任来自显式可信代理的转发头——**防止 X-Forwarded-For 伪造绕过限流**
- **配置热更新**：`configure()` 时清空旧尝试记录与封禁——新配置不携带陈旧状态
- **可注入时钟**：`clock` 参数让限流逻辑可测试

## 5.2 proxy_limits：流式代理约束

Hub 把请求代理到个人运行时，代理层必须防御资源滥用：

```python
async def limited_request_stream(stream, *, max_bytes,
                                 idle_timeout_seconds, completion_event):
    """Yield a request body while bounding its size and idle duration."""
    while True:
        async with asyncio.timeout(idle_timeout_seconds):
            chunk = await anext(iterator)
        total += len(chunk)
        if total > max_bytes:
            raise ProxyRequestTooLargeError(...)
        yield chunk
```

双重约束：**总字节上限** + **空闲超时**（慢速攻击防护）。`send_with_response_header_timeout()` 更精细——等待响应头有超时，但**不计入请求上传时间**（大文件上传不应被响应头超时误杀），用 `asyncio.wait` 双任务竞速实现。

## 5.3 设计哲学总结

1. **安全等级由探测决定，不由宣称决定**：`security_level` 随沙箱探测结果变化，preflight 先于 start。
2. **失败关闭**：无沙箱 → 无供给；供给器不可用 → 拒绝创建。
3. **声明式生命周期**：期望状态与观察状态分离，控制面负责调谐。
4. **多租户一等公民**：租户贯穿注册表、权限、审计。
5. **个人运行时即产品**：`ensure_personal_runtime` 让"每人一个隔离实例"成为开箱体验。
6. **代理层防滥用**：尺寸 + 空闲双约束，转发头只信可信代理。
7. **攻击视角验证**：Seatbelt 探测包含"环回网络是否被正确拒绝"的负向测试。

---

## 附：与云端 Agent 平台的定位对照

QwenPaw Hub 可以看作**单人/小团队版的云端 Agent 托管平台**：它提供了商业云平台的控制面要素（多租户、供给器抽象、声明式生命周期、审计、限流），但整体设计面向自托管场景——SQLite 而非分布式存储、单机进程/容器而非集群编排。这是"把云的能力装进个人服务器"的典型路径。
