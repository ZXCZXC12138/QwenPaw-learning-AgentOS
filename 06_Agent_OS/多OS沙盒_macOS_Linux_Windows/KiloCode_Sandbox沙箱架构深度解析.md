# Kilo Code Sandbox 沙箱架构深度解析

> **来源**：基于 kilocode-main 代码仓的全量源码分析（29 个文件，约 3500 行）
> **分析范围**：`@kilocode/sandbox`（引擎层）+ `opencode/kilocode/sandbox/`（策略管理层）
> **本文档包含四部分内容**：
> 1. Sandbox 架构深度解析（技术全景）
> 2. 用生活故事讲透 Sandbox 原理（通俗版）
> 3. Sandbox 模块外科手术式解耦分析（替换指南）
> 4. 引擎层三大核心接口的技术难点解析

---

## 目录

### Part 1: Sandbox 架构深度解析
- 双层架构全景图
- 引擎层 8 大核心组件（Profile / Context / Backend / Seatbelt / Bubblewrap / Network / Mutation / Filesystem）
- 策略管理层 7 大核心组件（Store / State / Preference / Config / Policy / Activation / 辅助模块）
- 完整执行路径示例
- 设计哲学总结

### Part 2: 用生活故事讲透 Sandbox 原理
- 保姆类比：为什么需要 Sandbox
- 工作手册类比：Profile 三维约束
- 物理锁类比：两大沙箱引擎
- 博物馆展柜类比：.git 三重保护
- 前台总机类比：网络代理
- 跑腿小哥类比：Mutation Worker
- 考勤表类比：状态管理
- 完整请求旅程

### Part 3: Sandbox 模块外科手术式解耦分析
- 18 个耦合点逐一扫描
- 25 个需替换 API 清单
- 4 阶段最小替换策略

### Part 4: 引擎层三大核心接口技术难点
- decorateFileSystem 难点
- prepareCommand 难点
- executeTool 难点

---


# Part 1: Sandbox 架构深度解析

> 基于 kilocode-main 代码仓全量源码分析（29 个文件，约 3500 行）

---

## Kilo Code Sandbox 架构全景

Kilo Code 的 sandbox 是一个**双层架构**的本地安全 confinement 系统，分为：

```
┌──────────────────────────────────────────────────────────────────────┐
│                    opencode/kilocode/sandbox/                        │
│                    ─── 策略管理层 (Policy Layer) ───                  │
│                                                                      │
│  config.ts    配置解析     store.ts     持久化存储                    │
│  state.ts     Session元数据 preference.ts 目录级偏好                  │
│  policy.ts    核心协调器   activation.ts 空闲检测+家族传播            │
│  network.ts   工具网络分类 inheritance.ts 授权令牌                    │
│  git.ts       Git命令分类  event.ts      状态变更事件                 │
├──────────────────────────────────────────────────────────────────────┤
│                    @kilocode/sandbox                                 │
│                    ─── 引擎层 (Engine Layer) ───                     │
│                                                                      │
│  profile.ts   约束画像     backend.ts    平台选择器                   │
│  seatbelt.ts  macOS后端    bubblewrap.ts Linux后端                    │
│  context.ts   运行时上下文  filesystem.ts 文件系统代理                 │
│  network.ts   网络拦截     proxy.ts      网络代理服务器               │
│  mutation.ts  写操作委托    path.ts       路径规范化                  │
│  destination.ts DNS解析    tls-client-hello.ts SNI检查               │
└──────────────────────────────────────────────────────────────────────┘
```

**核心设计思想**：不是用 Docker/VM 做重量级隔离，而是利用**操作系统原生沙箱**（macOS `sandbox-exec` / Linux `bwrap`）做**轻量级进程 confinement**——只限制 AI Agent 的工具执行（shell 命令、文件写入、网络请求），让用户的项目目录保持可读写，同时阻止 Agent 写到不该写的地方（如 `.git/`）或访问不该访问的网络。

---

## 第一层：引擎层 (`@kilocode/sandbox`)

### 1. Profile —— 约束画像

[profile.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/profile.ts) 定义了三维约束画像：

```typescript
interface Profile {
  filesystem: {
    allowWrite: PathRule[]    // 可写路径白名单（literal 或 subtree）
    denyWrite:  PathRule[]    // 显式拒绝路径（如沙箱策略目录本身）
    denyNames:  string[]      // 拒绝的文件/目录名（如 ".git"）
    temporaryDirectory?: string
  }
  network: {
    mode: "allow" | "deny" | "proxy"  // 三种网络模式
    allowedHosts: string[]             // proxy 模式下的白名单域名
  }
  environment: {
    deny: string[]                     // 剥离的环境变量（如 KILO_SERVER_PASSWORD）
    set:  Record<string, string>       // 强制设置的环境变量（如 TMPDIR）
  }
}
```

**关键设计**：`PathRule` 有两种粒度——`literal`（精确匹配一个文件）和 `subtree`（匹配整个目录树）。`denyNames` 是一个独特的设计：不管路径在哪，只要路径的任一部分叫 `.git`，就拒绝写入。这保护了 Git 历史不被 Agent 篡改。

### 2. Context —— 运行时上下文

[context.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/context.ts) 用 Effect 的 `Context.Reference` 实现了一个**纤维级（Fiber-level）** 的当前 Profile 注入：

```typescript
// 核心 API：
run(profile, effect)      // 在 profile 约束下执行 effect
unrestricted(effect)       // 无约束执行
enabled                    // 当前是否处于沙箱中
assertWrite(path)          // 检查 path 是否可写，不可写则 fail
```

**工作原理**：`run()` 将 Profile 注入 Effect 的 `CurrentProfile` 上下文，然后整个 effect 链条中所有文件系统操作、网络操作都会自动检查这个 Profile。这是典型的 Effect-TS 依赖注入模式——沙箱约束是**上下文传递**的，不是全局状态。

路径检查的核心逻辑（`assertTarget`）：
1. 将路径**规范化**（解析符号链接）
2. 检查是否命中 `denyWrite` 规则 → 拒绝
3. 检查路径的任一部分是否匹配 `denyNames`（如 `.git`）→ 拒绝
4. 检查是否命中 `allowWrite` 规则 → 不在白名单则拒绝

**Fail-closed 设计**：不在白名单 = 拒绝。不是"不在黑名单就放行"。

### 3. Backend —— 平台选择器

[backend.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/backend.ts) 根据 `process.platform` 选择后端：

```
darwin  → seatbelt  (macOS sandbox-exec)
linux   → bubblewrap (bwrap)
win32   → unavailable (不支持！)
```

`prepare()` 函数是核心入口：拿到一个 `Launch`（command + args + env），返回一个被沙箱包装过的 `Launch`。

### 4. Seatbelt —— macOS 后端

[seatbelt.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/seatbelt.ts) 调用 macOS 原生的 `/usr/bin/sandbox-exec`，生成 Apple Sandbox Profile Language (SBPL) 策略：

```
/usr/bin/sandbox-exec -p '<SBPL策略>' -DALLOW_WRITE_0=/project -DALLOW_WRITE_1=/data ... -- command args
```

生成的 SBPL 策略结构（[seatbelt-base.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/seatbelt-base.ts)）：

```scheme
(version 1)
(deny default)                    ; 默认拒绝一切
(allow process-exec)              ; 允许执行进程
(allow process-fork)              ; 允许 fork
(allow file-read*)                ; 读不限制！
(allow file-write*                ; 写操作：
  (require-all
    (require-any                  ; 必须在白名单路径内
      (literal (param "ALLOW_WRITE_0"))
      (subpath (param "ALLOW_WRITE_0"))
      ...
    )
    (require-not (literal ...))   ; 且不在黑名单路径
    (require-not (regex "..."))   ; 且不在 denyNames 路径
  )
)
```

**网络策略**（[seatbelt-network.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/seatbelt-network.ts)）：

| 模式 | SBPL 规则 |
|------|----------|
| `allow` | `(allow network-outbound)` + `(allow network-inbound)` |
| `deny` | `(deny network-outbound)` + `(allow network-inbound)` |
| `proxy` | deny 所有出站，但 allow 到 `localhost:{proxy.port}` 的出站 |

**关键设计决策**：
- **读不限制**：`(allow file-read*)` ——沙箱只约束写操作和网络，不约束读。这是因为 Agent 需要读项目内外的文件来理解代码，限制读会严重影响功能。
- **参数化路径**：用 `-D` 参数传入路径，避免路径中的特殊字符破坏 SBPL 语法。

### 5. Bubblewrap —— Linux 后端

[bubblewrap.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/bubblewrap.ts) 使用 Linux 的 `bwrap` 工具，通过 Linux 内核的 namespace 机制实现隔离：

```bash
bwrap \
  --unshare-user --disable-userns \    # 隔离用户命名空间
  --unshare-pid \                       # 隔离 PID 命名空间
  --unshare-net \                       # 隔离网络命名空间(deny/proxy模式)
  --die-with-parent \                   # 父进程死则子进程死
  --new-session \                       # 新会话
  --ro-bind / / \                       # 根文件系统只读挂载
  --dev /dev \                          # 最小 /dev
  --bind /project /project \            # 项目目录可写挂载
  --bind /data /data \                  # 数据目录可写挂载
  --ro-bind /project/.git /project/.git \  # .git 只读重挂载
  --proc /proc \                        # 新 /proc
  --chdir /project \
  -- command args
```

**安全验证**（`validate()` 函数）：
1. 确保 bwrap 可执行文件本身**不在**可写路径内（否则 Agent 可以替换 bwrap）
2. 确保可写根目录内**不包含**嵌套挂载点（防止挂载点逃逸）
3. 对 network relay 和 seccomp 过滤器也做同样的验证

**`.git` 保护**（`protectedPaths()` 函数）：
- 对每个可写 subtree，递归扫描找到所有名为 `.git` 的目录
- 将它们用 `--ro-bind` 重新挂载为只读
- 如果一个目录无法枚举（EACCES），**fail closed**：把整个父目录重绑定为只读

**Bubblewrap 可执行文件选择**：
1. 优先用 `$KILO_BWRAP_PATH` 环境变量覆盖（测试用）
2. 否则先尝试 `/usr/bin/bwrap`
3. 再尝试与可执行文件同目录下的 `bwrap`（bundled），并验证 SHA256

### 6. Network —— 网络拦截与代理

网络沙箱有三种模式，实现方式截然不同：

#### `deny` 模式
最简单——在 SBPL 中 deny 所有出站网络，在 bwrap 中 `--unshare-net` 隔离网络命名空间。进程完全无法发网络包。

对于 Effect 层的 HTTP 请求（[network.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/network.ts) 的 `decorateHttpClient`），直接 fail。

#### `proxy` 模式（核心创新）

这是最精巧的设计。[proxy.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/proxy.ts) 实现了一个**完整的 HTTP/HTTPS 代理服务器**：

```
┌──────────────────────────────────────────────────────────────┐
│ 沙箱内进程                                                    │
│   ↓ HTTP_PROXY=http://kilo:TOKEN@127.0.0.1:PORT              │
│                                                              │
│  [macOS: seatbelt 只允许到 localhost:PORT 的网络出站]          │
│  [Linux: bwrap --unshare-net + network-relay 桥接]           │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│ 代理服务器 (proxy.ts)                                         │
│   1. Basic Auth 验证 Token                                    │
│   2. 解析目标 host:port                                       │
│   3. 检查是否在 allowedHosts 白名单内                          │
│   4. DNS 解析 → 验证是公网 IP（防 DNS rebinding）              │
│   5. HTTPS CONNECT: 解析 TLS ClientHello SNI 二次验证          │
│   6. 建立到目标的连接，双向 pipe                                │
└──────────────────────────────────────────────────────────────┘
```

**安全层次**：
1. **Token 认证**：每个代理会话生成 24 字节随机 Token，用 `timingSafeEqual` 比较
2. **白名单检查**：`allowedHosts` 精确匹配 `host:port`
3. **DNS 验证**：`resolveDestination()` 解析 DNS 后检查必须是**公网 IP**（`isPublicAddress`），阻止 DNS rebinding 攻击（DNS 返回 `127.0.0.1` 绕过白名单）
4. **SNI 验证**（[tls-client-hello.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/tls-client-hello.ts)）：对 HTTPS CONNECT 隧道，解析 TLS ClientHello 中的 SNI 扩展，确保实际 TLS 连接目标与白名单一致（防止 HTTP 代理请求里套着 HTTPS 连到别的域名）
5. **环境变量清洗**：清除所有 `HTTP_PROXY`/`HTTPS_PROXY` 等代理环境变量，然后注入代理自己的 URL

**Linux 的特殊处理**（[kilo-sandbox-network-relay.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/kilo-sandbox-network-relay.ts)）：

因为 bwrap 的 `--unshare-net` 让进程完全没有网络，但代理跑在 `127.0.0.1:PORT`。解决方案：
1. 代理服务器监听一个 Unix domain socket
2. network-relay 进程在 bwrap 内监听 `127.0.0.1:3128`
3. relay 把 TCP 连接桥接到 Unix socket
4. 同时用 seccomp 过滤器限制 relay 进程的系统调用

#### `allow` 模式
不限制网络。但注意：即使 `mode=allow`，如果配置了 `allowed_hosts`，也会升级到 `proxy` 模式。

### 7. Mutation —— 文件系统写操作委托

[mutation.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/mutation.ts) 解决了一个关键问题：**当 Effect 层的文件系统操作需要写入时，如何确保写入受沙箱约束？**

方案：spawn 一个独立的 worker 子进程（[kilo-sandbox-mutation-worker.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/kilo-sandbox-mutation-worker.ts)），这个子进程**本身也跑在沙箱内**（通过 `confine()` 包装），通过 stdin/stdout JSON 协议通信：

```
主进程                    Worker 子进程（也在沙箱内）
  │                            │
  │── JSON Request ──stdin──→  │
  │                            │── fs.writeFile(...)
  │←── JSON Response ─stdout── │
  │                            │ (exit)
```

`batchMutations()` 是一个优化：把多个写操作收集起来，一次性发给 worker，减少进程 spawn 开销。

### 8. Filesystem —— 透明文件系统代理

[filesystem.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/kilo-sandbox/src/filesystem.ts) 包装了 Effect 的 `FileSystem` 服务，让所有写操作（`writeFile`、`copy`、`rename`、`symlink`、`truncate` 等）自动经过沙箱检查：

- 无 Profile → 直接调用原生 fs
- 有 Profile → 先 `assertPath()` 检查路径，再通过 mutation runner 执行

读操作（`readFile`、`readLink` 等）**不包装**——沙箱不限制读。

---

## 第二层：策略管理层 (`opencode/kilocode/sandbox/`)

### 9. SandboxStore —— 持久化

[store.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/opencode/src/kilocode/sandbox/store.ts) 将每个 session 的沙箱状态持久化到磁盘：

```
{XDG_STATE_HOME}/kilo-sandbox-policy/
  {sha256(sessionID)}/
    {sha256(directory)}.json    ← 内容: { enabled, mode, allowedHosts, writablePaths, version }
```

写入用 **atomic write**（先写 `.tmp` 文件，再 `rename`），权限 `0o600`。读取时做严格的 schema 验证。

### 10. SandboxState —— Session 元数据

[state.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/opencode/src/kilocode/sandbox/state.ts) 将沙箱的 `enabled` 状态存在 Session 表的 `metadata` JSON 字段中（key: `kilocode.sandbox`）。这是**创建时**的选择——当用户通过 UI toggle 创建新 session 时，这个选择被记录在 session 元数据里。

### 11. SandboxPreference —— 目录级偏好

[preference.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/opencode/src/kilocode/sandbox/preference.ts) 记录每个项目目录的**最后一次 toggle 选择**：

```
{XDG_STATE_HOME}/kilo-sandbox-preference/
  {sha256(directory)}.json    ← 内容: true 或 false
```

这样新 session 会继承这个目录上次的选择。

### 12. SandboxConfig —— 配置解析

[config.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/opencode/src/kilocode/sandbox/config.ts) 从 `kilo.json` 配置中解析沙箱设置：

```json
{
  "sandbox": {
    "enabled": true,
    "network": "deny",           // "allow" | "deny"
    "writable_paths": ["~/extra"],
    "allowed_hosts": ["api.github.com:443"]
  }
}
```

解析逻辑（`resolve()`）：
- `network=allow` 且无 `allowed_hosts` → mode = `allow`
- `network=deny`（默认）且无 `allowed_hosts` → mode = `deny`
- 有 `allowed_hosts` → mode = `proxy`（即使 network 不是 deny）

**`scope()` 函数**的安全设计：本地（项目级）配置**不能放宽**全局配置。如果全局没开沙箱，本地开了也没用；本地不能取消全局的 network=deny。

### 13. SandboxPolicy —— 核心协调器

[policy.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/opencode/src/kilocode/sandbox/policy.ts) 是整个沙箱系统的**大脑**（633 行），协调所有组件。

#### 并发控制（三重信号量）

```
locks:     Semaphore(1) per session   ← 保护状态变更（toggle/refresh/inherit）
refreshes: Semaphore(1) per session   ← 保护 config reconcile 不重复
gates:     Semaphore(1M) per session  ← 工具执行门控（toggle 时阻塞新工具）
```

`gates` 的设计很精巧：用 100 万 permits 模拟一个"读锁"。`execute()` 取 1 permit，`toggle` 时取全部 100 万 permits——这样 toggle 会等所有正在执行的工具完成，同时阻止新工具开始执行。

#### Profile 构建（`profile()` 函数）

```typescript
writable = [
  ...project,           // 项目目录 + worktree
  Global.Path.data,     // ~/.local/share/kilo
  Global.Path.cache,    // ~/.cache/kilo
  Global.Path.config,   // ~/.config/kilo
  Global.Path.state,    // ~/.local/state/kilo
  Global.Path.tmp,      // 临时目录
  Global.Path.bin,      // 二进制目录
  Global.Path.log,      // 日志目录
  Global.Path.repos,    // 仓库目录
  ...extraWritable      // 用户配置的 writable_paths
]

denyWrite = [
  SandboxStore.root,       // 沙箱策略文件本身
  SandboxPreference.root,  // 偏好文件本身
  Global.Path.config       // 配置目录
]

denyNames = [".git"]       // 永远不写 .git

environment.deny = [
  "KILO_CONFIG",           // 防止 Agent 读配置
  "KILO_SERVER_PASSWORD",  // 防止 Agent 获取密码
  "KILO_SERVER_USERNAME",
  ...
]
```

#### 初始状态决策链

```
Session metadata (kilocode.sandbox.enabled)     ← 创建时 UI toggle
    ↓ undefined?
Per-directory preference (SandboxPreference)     ← 上次 toggle 的选择
    ↓ undefined?
Config default (kilo.json sandbox.enabled)       ← 默认 false
```

**关键**：一旦初始化，**配置变更不能改变已初始化的策略**。`reconcile()` 只更新 `mode`/`allowedHosts`/`writablePaths`，不改变 `enabled`。这防止了恶意项目配置关闭沙箱。

#### 工具执行流（`execute()` → `executeTool()`）

```
1. snapshot(sessionID)          ← 确保初始化
2. gated(sessionID, 1, ...)     ← 获取执行门控
3. current(sessionID, true)     ← 获取当前状态（可能触发 reconcile）
4. if (!enabled) → unrestricted ← 未启用则直接执行
5. backendSupport()             ← 检查后端是否可用
6. runSandbox(profile, effect)  ← 注入 Profile，执行 effect
   └─ context.run()
      ├─ normalize(profile)     ← 规范化所有路径（解析符号链接）
      ├─ withProxy()            ← 如果 proxy 模式，启动代理服务器
      └─ effect 执行中：
         ├─ filesystem 操作 → assertPath() 检查 → mutation worker 执行
         ├─ HTTP 请求 → decorateHttpClient() 拦截 → proxy 或 deny
         └─ shell 命令 → backend.prepare() 包装 → sandbox-exec/bwrap
```

#### 网络工具分类（[network.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/opencode/src/kilocode/sandbox/network.ts)）

[network-tools.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/opencode/src/kilocode/sandbox/network-tools.ts) 把工具分为三类：

| 分类 | 工具 | 网络策略 |
|------|------|---------|
| **opaque**（间接网络） | `semantic_search`, `lsp` | 需要 `assertNetwork` 检查 |
| **host**（宿主执行） | `interactive_terminal`, `notebook_execute`, `background_process` | 需要 `assertSandbox`（沙箱内禁止） |
| **builtin**（普通内建） | `bash`, `read`, `write` 等 | 走正常沙箱流程 |
| **MCP** | 所有 MCP 工具 | 一律需要 `assertNetwork` |

### 14. SandboxActivation —— 安全 Toggle

[activation.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/opencode/src/kilocode/sandbox/activation.ts) 确保开启沙箱时不会中断正在运行的工具：

- `family()`：递归收集 session 的所有子孙 session
- `idle()`：检查整个家族是否空闲——没有正在运行的：
  - Session 状态（busy）
  - Background Job
  - Background Process
  - Interactive Terminal
  - Notebook 请求

只有全部空闲才允许 toggle。

### 15. 辅助模块

- **[inheritance.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/opencode/src/kilocode/sandbox/inheritance.ts)**：一次性授权令牌（`si-UUID`），24 小时 TTL，用于跨目录的 session 继承
- **[git.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/opencode/src/kilocode/sandbox/git.ts)**：解析 shell 命令中的 git 子命令，区分只读（`status`/`log`/`diff`）和变更（`commit`/`push`/`merge`），用于权限判断

---

## 完整执行路径示例

**场景**：Agent 在 macOS 上执行 `echo hello > /project/output.txt`，沙箱已启用，network=deny

```
1. SandboxPolicy.executeTool(sessionID, bashTool, shell.execute("echo hello > ..."))
   │
2. snapshot(sessionID) → 读取/初始化 session 的 Snapshot
   │
3. gated(sessionID, 1) → 获取执行门控
   │
4. Network.tool(bashTool, effect) → bash 是 builtin，不加额外网络断言
   │
5. context.run(profile, effect)
   ├─ normalize(profile) → 规范化所有路径
   ├─ withProxy(profile, effect) → network=deny，不启动代理
   │
6. shell.execute("echo hello > /project/output.txt")
   ├─ backend.prepare(launch)
   │  ├─ environment() → 清洗环境变量，注入 TMPDIR
   │  ├─ seatbelt.generate(profile, launch)
   │  │  → /usr/bin/sandbox-exec -p '(version 1)(deny default)...' \
   │  │    -DALLOW_WRITE_0=/project -DALLOW_WRITE_1=/data ... \
   │  │    -- /bin/sh -c 'echo hello > /project/output.txt'
   │  └─ 返回包装后的 Launch
   │
   └─ spawn(/usr/bin/sandbox-exec, [...args])
      │
      └─ 内核执行 SBPL 策略：
         ├─ 文件读：允许
         ├─ 写 /project/output.txt：ALLOW_WRITE_0 匹配 → 允许
         ├─ 写 /project/.git/config：denyNames 匹配 ".git" → 拒绝
         ├─ 写 /etc/passwd：不在 allowWrite → 拒绝
         └─ 网络：deny network-outbound → 拒绝
```

---

## 设计哲学总结

| 原则 | 实现 |
|------|------|
| **OS 原生隔离** | macOS seatbelt / Linux bwrap，不用 Docker/VM |
| **Fail-closed** | 不在白名单 = 拒绝，不是"不在黑名单 = 放行" |
| **读不限，写限** | 读全放，写白名单。Agent 需要读代码理解上下文 |
| **.git 不可变** | denyNames 机制 + 递归扫描 + bwrap ro-bind |
| **网络三层** | allow（全放）/ deny（全断）/ proxy（白名单代理） |
| **DNS rebinding 防护** | 解析后验证公网 IP + TLS SNI 二次校验 |
| **配置不可放宽** | 本地配置不能削弱全局策略；已初始化后 enabled 不变 |
| **原子持久化** | tmp + rename，0o600 权限 |
| **并发安全** | 三重信号量（lock / refresh / gate） |
| **家族传播** | 父 session 开沙箱 → 所有子 session 继承，取交集 |
| **空闲才 toggle** | 开启沙箱前检查整个 session 家族无运行中工具 |

---
---

# Part 2: 用生活故事讲透 Sandbox 原理

---

## 一、为什么需要 Sandbox？

想象你请了一个**超级能干的保姆**（AI Agent）来家里干活。她什么都会——做饭、打扫、修电器、整理文件。但你心里有点慌：

> "她会不会打开我的保险箱？"
> "她会不会动我的日记本？"
> "她会不会偷偷打电话往外说家里的秘密？"

**Sandbox 就是给保姆划定的活动范围 + 行为规则。**

```
┌─────────────────────────────────────────────┐
│              你的家（电脑）                    │
│                                             │
│  ┌─────────────────────┐   ┌─────────────┐ │
│  │  保姆可以进的区域     │   │ 禁区        │ │
│  │  (allowWrite)       │   │ (.git/)     │ │
│  │                     │   │ 保险箱      │ │
│  │  ✅ 客厅（项目目录）  │   │ (denyWrite) │ │
│  │  ✅ 厨房（数据目录）  │   │             │ │
│  │  ✅ 书房（缓存目录）  │   │ 🔒 上锁了   │ │
│  │                     │   │             │ │
│  └─────────────────────┘   └─────────────┘ │
│                                             │
│  📞 电话（网络）：                           │
│  deny 模式 → 电话线被剪断                   │
│  proxy 模式 → 只能打给白名单上的人           │
│  allow 模式 → 随便打                        │
└─────────────────────────────────────────────┘
```

---

## 二、Profile —— "保姆的工作手册"

每次保姆上工前，你会给她一本手册，上面写着三条规则：

### 规则一：文件系统（哪些地方能写）

```
📋 工作手册 - 文件系统页

✅ 可以写的地方：
   - /Users/小明/my-project/     （项目目录）
   - /Users/小明/.kilo/data/     （Kilo 数据目录）
   - /Users/小明/.kilo/cache/    （Kilo 缓存目录）
   - /tmp/                       （临时目录）

❌ 绝对不能写的地方：
   - /Users/小明/my-project/.git/  （Git 历史！碰都不能碰！）
   - /Users/小明/.kilo/sandbox-policy/ （手册本身！不能改自己的规则！）

🚫 看到这些名字就绕道：
   - ".git"  （不管藏在哪层目录，看到就拒绝）
```

**代码里的真实对应**（[profile.ts](file:///Users/zyz/Documents/QoderCN/2026-08-23/chat-2/kilocode-main/packages/opencode/src/kilocode/sandbox/policy.ts#L228-L271)）：

```typescript
// 实际构建 Profile 的代码
const writable = [
  ...project,              // 项目目录
  Global.Path.data,        // 数据
  Global.Path.cache,       // 缓存
  Global.Path.tmp,         // 临时
  // ...
]

const denyWrite = [SandboxStore.root]    // 不能改自己的规则
const denyNames = [".git"]               // 看到 .git 就拒绝
```

### 规则二：网络（能不能打电话）

```
📋 工作手册 - 网络页

三种模式（老板选一个）：

🔴 deny 模式（最严格）：
   "电话线我剪了，你干活期间不能上网。"
   → Agent 无法访问任何网站

🟡 proxy 模式（精准控制）：
   "电话可以打，但只能打给这几个号码："
   - api.github.com:443  （拉代码用）
   - registry.npmjs.org:443  （装包用）
   "打别的号码？挂断。"

🟢 allow 模式（最宽松）：
   "电话随便打。"
```

### 规则三：环境变量（不能知道某些秘密）

```
📋 工作手册 - 信息隔离页

以下信息你不能知道（从你的记忆里抹掉）：
   ❌ KILO_SERVER_PASSWORD    （服务器密码）
   ❌ KILO_SERVER_USERNAME    （服务器用户名）
   ❌ KILO_CONFIG             （配置文件在哪）

以下信息告诉你：
   ✅ TMPDIR = /tmp/kilo-xxx  （你的临时工作区在这）
```

---

## 三、两大沙箱引擎 —— "两种物理锁"

手册写好了，但怎么**强制执行**呢？你不能光靠保姆自觉。你需要**物理锁**。

Kilo Code 根据操作系统选择不同的"锁"：

### macOS：`sandbox-exec`（苹果自带的安保系统）

这是 macOS 内置的内核级沙箱。就像小区物业的安保系统——你改不了，它是系统的一部分。

**举个具体例子**：

Agent 想执行 `npm install`，Kilo Code 实际执行的是：

```bash
/usr/bin/sandbox-exec \
  -p '(version 1)
       (deny default)           ← 默认禁止一切
       (allow process-exec)     ← 允许运行程序
       (allow file-read*)       ← 读文件随便
       (allow file-write*       ← 写文件有条件：
         (require-all
           (require-any          ← 必须在这些目录里：
             (subpath "/Users/小明/my-project")
             (subpath "/tmp/kilo-xxx")
           )
           (require-not           ← 但不能是这些：
             (regex ".*/\.git(/|$)")
           )
         )
       )
       (deny network-outbound)  ← 不能上网
      ' \
  -- npm install
```

**类比**：就像给保姆戴上了一个**智能手环**，手环里写满了规则。保姆的每个动作都会经过手环检查——手伸向保险箱？手环报警，动作被拦截。

### Linux：`bwrap`（Bubblewrap，用 Linux 内核的 namespace）

Linux 没有 `sandbox-exec`，但有更强大的 **namespace** 机制。这就是 Docker 用的同一套技术。

**类比**：`bwrap` 就像把保姆**关进一个透明的房间**。

```
┌─── bwrap 创建的"透明房间" ──────────────────────┐
│                                                  │
│  整个文件系统像一张大地图，铺在地板上              │
│                                                  │
│  📁 整个根目录 /  → 用透明玻璃盖住（只读）         │
│     保姆能看到所有文件，但不能改                    │
│                                                  │
│  📁 /Users/小明/my-project/ → 掀开玻璃（可写）    │
│     只有这块地方保姆可以动                          │
│                                                  │
│  📁 /Users/小明/my-project/.git/ → 玻璃盖住（只读）│
│     项目目录里，但 .git 单独盖住！                  │
│                                                  │
│  📞 电话线 → --unshare-net（剪断）                │
│     这个房间完全没有电话                            │
│                                                  │
│  🚪 门 → --die-with-parent                       │
│     你关机（父进程退出），房间自动拆除               │
└──────────────────────────────────────────────────┘
```

**实际命令长这样**：

```bash
bwrap \
  --unshare-user \       # 隔离用户空间
  --unshare-pid \        # 隔离进程空间（看不到外面的进程）
  --unshare-net \        # 隔离网络（完全断网）
  --die-with-parent \    # 父死子死
  --ro-bind / / \        # 整个根目录只读
  --bind /project /project \   # 项目目录可写
  --ro-bind /project/.git /project/.git \  # .git 单独只读
  -- npm install
```

---

## 四、.git 保护 —— "博物馆展柜"

为什么 `.git` 这么特殊？因为 `.git` 里存着整个项目的**完整历史**。如果 Agent 不小心改了 `.git`，就像有人偷偷改了历史书的某一页——所有 commit 记录都可能乱掉。

Kilo Code 用了**三重保护**：

```
第一重：denyNames = [".git"]
   → 路径里任何一部分叫 ".git" 就拒绝
   → 不管 /project/.git 还是 /project/src/.git 都拦

第二重（Linux）：--ro-bind /project/.git /project/.git
   → 即使 /project 可写，.git 被单独"玻璃罩"盖住
   → 就像博物馆里的展柜，你能看但不能摸

第三重（Linux 扫描）：递归扫描可写目录，找到所有 .git
   → 如果项目里有子模块也有 .git，全部保护
   → 扫描不到的目录？fail closed → 整个区域变只读
```

**真实场景**：

```
Agent 想执行：git commit -m "fix bug"

步骤 1: git commit 需要写 .git/objects/... 
步骤 2: 沙箱检查 → 路径包含 ".git" → denyNames 命中！
步骤 3: 拒绝！💥

等等...那 git commit 怎么用？
→ 答案是：git 命令走的是 shell 工具
→ shell 工具在沙箱里执行
→ git 本身也需要写 .git
→ 所以 Kilo Code 的 git.ts 会分析命令
→ 只读的 git 命令（log, status, diff）→ 放行
→ 变更的 git 命令（commit, push）→ 需要特殊处理
```

---

## 五、网络代理 —— "前台总机"

`proxy` 模式是最精巧的设计。想象公司有个**前台总机**：

```
┌─── 公司（沙箱）──────────────────────────────────┐
│                                                   │
│  保姆（Agent）想打电话                             │
│  ↓                                                │
│  拿起电话 → 只能打到前台总机 127.0.0.1:3128        │
│  （其他号码全被运营商拦截了）                       │
│                                                   │
├───────────────────────────────────────────────────┤
│  前台总机（Proxy Server）                          │
│                                                   │
│  保姆："请帮我接 api.github.com"                   │
│  总机：                                            │
│    1. "请问您的分机号？" → Token 认证              │
│    2. 查白名单 → "api.github.com" ✅ 在名单上      │
│    3. 查电话号码 → DNS 解析 → 140.82.121.6        │
│    4. 检查是不是外线 → 公网 IP ✅ 不是内网地址      │
│    5. "好的，帮您接通" → 建立连接                  │
│                                                   │
│  保姆："请帮我接 evil-hacker.com"                  │
│  总机：查白名单 → ❌ 不在名单 → "对不起，无法接通"  │
│                                                   │
│  保姆："请帮我接 127.0.0.1:8080"（想攻击本地服务）  │
│  总机：DNS 解析 → 127.0.0.1 → 非公网 IP → ❌ 拒绝 │
└───────────────────────────────────────────────────┘
```

**为什么要检查公网 IP？** 防止 **DNS rebinding 攻击**：

```
攻击场景：
  白名单里有 "my-dns.com"
  黑客让 my-dns.com 解析到 127.0.0.1
  如果不检查 IP → 代理帮 Agent 连到了本机 → 可以攻击本地服务
  
Kilo Code 的防护：
  resolveDestination() → DNS 解析 → 检查 isPublicAddress()
  127.0.0.1 → range() !== "unicast" → 不是公网地址 → 拒绝！
```

**HTTPS 还多一层 SNI 检查**：

```
HTTPS 场景（更隐蔽的攻击）：
  Agent 通过 HTTP 代理请求 CONNECT api.github.com:443
  代理同意后建立隧道
  但 Agent 在 TLS 握手时 SNI 写的是 evil.com
  
Kilo Code 的防护（tls-client-hello.ts）：
  代理会"偷看" TLS ClientHello 包里的 SNI 字段
  发现 SNI ≠ api.github.com → 断开连接！
```

---

## 六、Mutation Worker —— "跑腿小哥"

当 Agent 要写文件时，不是直接写，而是叫一个**跑腿小哥**：

```
Agent："帮我写一个文件 /project/output.txt，内容是 hello"
  │
  ↓ 写一张纸条（JSON Request）
  │ { op: "writeFile", path: "/project/output.txt", data: "aGVsbG8=" }
  │
  ↓ 纸条通过 stdin 递给跑腿小哥
  │
跑腿小哥（mutation-worker）：
  │ 1. 他自己也戴着沙箱手环（confine）
  │ 2. 看看纸条：要写 /project/output.txt
  │ 3. /project 在白名单里 → 可以写
  │ 4. 执行 fs.writeFile(...)
  │ 5. 写好了，通过 stdout 回复：{ ok: true }
  │
  ↓ 回复
  │
Agent："收到，写好了"
```

**为什么要多此一举？** 因为跑腿小哥**也在沙箱里**，受同样的规则约束。即使 Agent 试图绕过检查直接写文件，操作系统级的沙箱也会拦截。

**批量模式**（`batchMutations`）：

```
Agent 要写 10 个文件：
  普通模式：叫 10 次跑腿小哥 → 慢
  批量模式：写一张大纸条，10 个操作一起给 → 快
```

---

## 七、状态管理 —— "保姆的考勤表"

### 三层存储

```
┌─── 第一层：Session 元数据（数据库里）─────────────┐
│  "这个 Session 创建时，用户选了开/关沙箱"          │
│  → 最高优先级，创建时的选择不可被后续配置覆盖       │
└──────────────────────────────────────────────────┘
         ↓ 没有？
┌─── 第二层：目录偏好（文件）──────────────────────┐
│  "上次在这个项目里，用户选了开/关"                │
│  → 新 Session 继承上次的选择                     │
└──────────────────────────────────────────────────┘
         ↓ 没有？
┌─── 第三层：全局配置（kilo.json）─────────────────┐
│  "老板的默认规定"                                │
│  → 最低优先级，默认是 false（不开）              │
└──────────────────────────────────────────────────┘
```

### 配置不能"自己给自己解锁"

```
恶意项目配置 kilo.json：
{
  "sandbox": { "enabled": false }   ← 想关掉沙箱！
}

Kilo Code 的处理：
  1. 用户在全局设置里开了沙箱
  2. Session 初始化时 enabled = true
  3. 项目配置说 enabled = false
  4. reconcile() 只更新 mode/allowedHosts/writablePaths
  5. enabled 不变！→ 沙箱继续开着 ✅
  
  就像保姆手册上写的：
  "本手册一经生效，雇主不能通过贴便条的方式修改规则"
```

---

## 八、完整的一天 —— 一个请求的旅程

```
小明在 VS Code 里对 Kilo Code 说：
"帮我在 src/utils.ts 里加一个 formatDate 函数"

0. 用户发了指令
   → SandboxPolicy.executeTool(sessionID, writeTool, write("src/utils.ts", ...))

1. 📋 查手册
   → snapshot(sessionID) → 读取这个 Session 的沙箱状态
   → enabled: true, mode: deny, writablePaths: []

2. 🔒 拿门禁卡
   → gated(sessionID, 1) → 确认没有人在 toggle 沙箱

3. 🏗️ 构建约束画像
   → profile() 生成：
     allowWrite: [/project, /tmp/kilo-xxx, ...]
     denyWrite:  [/project/.kilo/sandbox-policy]
     denyNames:  [".git"]
     network:    { mode: "deny" }

4. 🍎 调用 macOS 沙箱
   → seatbelt.generate() 生成 SBPL 策略
   → 包装命令：/usr/bin/sandbox-exec -p '...' -- node -e 'writeFile(...)'

5. ✅ 执行
   → 写 /project/src/utils.ts
   → 路径匹配 allowWrite[0]（/project 的 subtree）
   → 不包含 ".git"
   → 写入成功！

6. 📝 返回结果
   → "已在 src/utils.ts 中添加 formatDate 函数"
```

---

## 九、一张图总结

```
┌─────────────────── Kilo Code Sandbox ───────────────────┐
│                                                          │
│  用户/配置                                                │
│    ↓                                                     │
│  ┌─────────────────────────────────────────────┐        │
│  │         策略管理层（大脑）                     │        │
│  │  决定：开不开？怎么限？限什么？                │        │
│  │  config → state → preference → snapshot       │        │
│  └─────────────────┬───────────────────────────┘        │
│                    ↓                                     │
│  ┌─────────────────────────────────────────────┐        │
│  │         引擎层（手脚）                        │        │
│  │                                              │        │
│  │  macOS: sandbox-exec (SBPL)                 │        │
│  │  Linux: bwrap (namespace)                   │        │
│  │                                              │        │
│  │  管三件事：                                  │        │
│  │  📁 文件写入 → 白名单检查                    │        │
│  │  📞 网络请求 → deny/白名单代理               │        │
│  │  🔑 环境变量 → 敏感信息剥离                  │        │
│  └─────────────────────────────────────────────┘        │
│                    ↓                                     │
│  ┌─────────────────────────────────────────────┐        │
│  │         操作系统内核（最终裁判）               │        │
│  │  不管 Agent 多聪明，内核说不行就是不行         │        │
│  └─────────────────────────────────────────────┘        │
│                                                          │
│  核心哲学：                                               │
│  "不是信任 Agent 然后监控它，                              │
│   而是不信任 Agent 然后用物理锁限制它"                    │
└──────────────────────────────────────────────────────────┘
```

**一句话总结**：Kilo Code 的 Sandbox 就像给 AI 保姆戴上了**操作系统级别的手环**——手册（Profile）写着哪些地方能去、哪些电话能打、哪些秘密不能知道；物理锁（seatbelt/bwrap）在操作系统内核层面强制执行；前台总机（proxy）精确控制网络白名单；跑腿小哥（mutation worker）确保文件写入也受约束。四层防护，让 AI 只能在你划定的圈子里干活。

---
---

# Part 3: Sandbox 模块外科手术式解耦分析

### 一、Sandbox 模块的分层结构

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 3: VSCode 前端层 (kilo-vscode)                               │
│  sandbox-bootstrap.ts, sandbox-session.ts, KiloProvider.ts          │
│  职责: UI toggle、状态展示、session 创建时注入 metadata              │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 2: 策略管理层 (opencode/kilocode/sandbox/)                    │
│  policy.ts, store.ts, state.ts, config.ts, activation.ts,           │
│  preference.ts, inheritance.ts, network.ts, network-tools.ts,       │
│  git.ts, event.ts                                                   │
│  职责: 状态管理、生命周期、工具执行路由、网络分类                      │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 1: 引擎层 (@kilocode/sandbox)                                │
│  profile.ts, backend.ts, seatbelt.ts, bubblewrap.ts, context.ts,    │
│  filesystem.ts, network.ts, proxy.ts, mutation.ts, path.ts,         │
│  destination.ts, tls-client-hello.ts, mutation-protocol.ts          │
│  职责: OS 级进程 confinement、文件系统代理、网络代理                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 二、全部耦合点清单（按外部模块逐一列出）

#### 🔴 耦合点 1：`packages/core/src/fs-util.ts` — 全局文件系统代理

**耦合方式**：import `@kilocode/sandbox`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L2 | `import { decorateFileSystem, ensureDirectory }` | 引入沙箱装饰器 |
| L57 | `decorateFileSystem(yield* FileSystem.FileSystem)` | **把 Effect 原生 FileSystem 包装成沙箱感知版** |
| L118 | `ensureDirectory(fs, path)` | `ensureDir` 操作走沙箱约束的 mkdir |
| L133 | `ensureDirectory(fs, dirname(path))` | `writeWithDirs` 操作走沙箱约束的 mkdir |

**解耦难度**：⭐⭐⭐（核心基础设施，所有文件写入都经过这里）

**替换方案**：将 `decorateFileSystem()` 替换为你自己的 sandbox 装饰器。你的 sandbox 需要提供一个等价的 `decorateFileSystem(fs)` 函数，接受 Effect 的 `FileSystem` 接口，返回一个写操作受约束的新 `FileSystem`。

---

#### 🔴 耦合点 2：`packages/core/src/cross-spawn-spawner.ts` — 进程 spawn 拦截

**耦合方式**：import `@kilocode/sandbox`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L4 | `import { prepareCommand as prepareSandbox }` | 引入沙箱命令包装 |
| L394 | `prepareSandbox(command, dir, env)` | **每个 shell 命令 spawn 前，用沙箱包装命令** |

**解耦难度**：⭐⭐⭐（所有 shell/bash 工具的执行都经过这里）

**替换方案**：你的 sandbox 需要提供 `prepareCommand(command, cwd, env)` → 返回被沙箱包装后的 `ChildProcess.StandardCommand`。在 macOS 上你的实现可以调用自己的 seatbelt/bwrap，或者用 Docker/nsjail 等替代方案。

---

#### 🔴 耦合点 3：`packages/opencode/src/session/tools.ts` — 工具执行入口

**耦合方式**：import `@/kilocode/sandbox/policy`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L24 | `import * as SandboxPolicy` | 引入策略管理 |
| L175 | `SandboxPolicy.executeTool(ctx.sessionID, item, effect)` | **所有内建工具的执行都经过此函数** |
| L482 | `SandboxPolicy.executeMcp(ctx.sessionID, entry, effect)` | **所有 MCP 工具的执行都经过此函数** |

**解耦难度**：⭐⭐⭐⭐（这是工具执行的总入口）

**替换方案**：`executeTool()` 和 `executeMcp()` 是核心 API。你的策略层需要提供等价函数：
- `executeTool(sessionID, tool, effect)` → 在沙箱约束下执行工具 effect
- `executeMcp(sessionID, tool, effect)` → 在沙箱约束下执行 MCP 工具 effect

---

#### 🟠 耦合点 4：`packages/opencode/src/tool/shell.ts` — Shell 工具的提权执行

**耦合方式**：import `@/kilocode/sandbox/policy` + `@/kilocode/sandbox/git`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L29 | `import { mutates as mutatesGit } from "@/kilocode/sandbox/git"` | 判断 git 命令是否变更型 |
| L30 | `import * as SandboxPolicy` | 引入策略 |
| L758 | `SandboxPolicy.executeEscalated(approved, effect)` | **用户批准提权后，绕过沙箱执行** |

**解耦难度**：⭐⭐

**替换方案**：`executeEscalated(approved, effect)` 很简单——`approved ? unrestricted(effect) : effect`。你的实现只需提供同样的提权旁路逻辑。

---

#### 🟠 耦合点 5：`packages/opencode/src/tool/code-mode.ts` — Code Mode 工具

**耦合方式**：import `@/kilocode/sandbox/policy`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L24 | `import * as SandboxPolicy` | 引入策略 |
| L151 | `SandboxPolicy.executeMcp(sessionID, tool, effect)` | MCP 工具在 code-mode 中的沙箱执行 |
| L222 | `SandboxPolicy.networkRestricted(sessionID)` | **查询 session 是否网络受限**，决定是否隐藏 MCP 工具目录 |

**解耦难度**：⭐⭐

**替换方案**：`networkRestricted()` 返回 boolean，你的策略层需要提供等价查询。

---

#### 🟠 耦合点 6：`packages/opencode/src/tool/task.ts` — Task（子 Agent）工具

**耦合方式**：import `@/kilocode/sandbox/policy`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L24 | `import * as SandboxPolicy` | 引入策略 |
| L184 | `SandboxPolicy.fallback(cfg)` | **从全局配置提取沙箱 fallback 快照** |
| L186 | `SandboxPolicy.inherit(parentID, childID, fallback)` | **子 session 继承父 session 的沙箱策略** |
| L206 | `SandboxPolicy.inherit(parentID, childID, fallback)` | 创建子 session 后再次同步继承 |

**解耦难度**：⭐⭐⭐（涉及 session 父子继承链）

**替换方案**：你需要提供：
- `fallback(config)` → 从配置对象提取沙箱快照
- `inherit(parentID, childID, fallback?)` → 子 session 继承父的沙箱约束

---

#### 🟠 耦合点 7：`packages/opencode/src/session/session.ts` — Session 生命周期

**耦合方式**：import `@/kilocode/sandbox/policy` + `@/kilocode/sandbox/inheritance`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L38 | `import * as SandboxInheritance` | 引入授权令牌 |
| L44 | `import * as SandboxPolicy` | 引入策略 |
| L614 | `sandboxFallback?: SandboxPolicy.Snapshot` | **Session.create 的参数类型依赖** |
| L660 | `SandboxPolicy.inherit(source, result.id, fallback, sourceDir)` | **创建 session 时继承沙箱策略** |
| L726 | `SandboxPolicy.dispose(sessionID, effect)` | **删除 session 时清理沙箱状态** |

**解耦难度**：⭐⭐⭐

**替换方案**：
- `Snapshot` 类型需要导出（`{ enabled, mode, allowedHosts, writablePaths, version }`）
- `inherit()` 和 `dispose()` 需要在 session 生命周期中调用

---

#### 🟠 耦合点 8：`packages/opencode/src/session/prompt.ts` — Prompt 处理

**耦合方式**：import `@/kilocode/sandbox/policy`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L21 | `import * as SandboxPolicy` | 引入策略 |
| L908 | `SandboxPolicy.networkRestricted(sessionID)` | **判断网络是否受限，决定是否跳过某些资源解析** |

**解耦难度**：⭐

**替换方案**：只需提供 `networkRestricted(sessionID)` 查询。

---

#### 🟡 耦合点 9：`packages/opencode/src/tool/registry.ts` — 工具注册表

**耦合方式**：import `@/kilocode/sandbox/network`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L69 | `import * as ToolNetwork` | 引入网络分类 |
| L324 | `s.builtin.map(ToolNetwork.builtin)` | **给内建工具打 builtin 标记** |
| L409 | `ToolNetwork.isBuiltin(tool) ? ToolNetwork.builtin(result) : result` | 工具包装时保留 builtin 标记 |
| L504 | `ToolNetwork.httpLayer` | **注入沙箱感知的 HTTP Client Layer** |

**解耦难度**：⭐⭐⭐

**替换方案**：
- `builtin(tool)` / `isBuiltin(tool)` → 工具标记系统（Symbol 属性）
- `httpLayer` → 一个 Effect Layer，提供受沙箱约束的 HTTP Client

---

#### 🟡 耦合点 10：`packages/opencode/src/mcp/index.ts` — MCP 服务

**耦合方式**：import `@/kilocode/sandbox/network`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L44 | `import * as SandboxNetwork` | 引入网络分类 |
| L725 | `SandboxNetwork.remote(tool)` | **给远程 MCP 工具打 remote 标记** |

**解耦难度**：⭐

**替换方案**：`remote(tool)` 只是给对象加一个 Symbol 标记。

---

#### 🟡 耦合点 11：`packages/opencode/src/config/config.ts` — 配置合并

**耦合方式**：import `@/kilocode/sandbox/config`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L50 | `import { SandboxConfig }` | 引入配置解析 |
| L568 | `SandboxConfig.scope(next, scope)` | **配置合并时，对 sandbox 字段做 scope 限制**（本地不能放宽全局） |

**解耦难度**：⭐

**替换方案**：提供 `scope(config, source)` 函数，限制本地配置不能放宽全局沙箱设置。

---

#### 🟡 耦合点 12：`packages/opencode/src/kilocode/tool/encoded-io.ts` — 文件写入工具

**耦合方式**：import `@kilocode/sandbox`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L3 | `import { batchMutations, enabled, ensureDirectory }` | 引入沙箱 API |
| L26 | `if (!(yield* enabled)) return yield* fs.writeWithDirs(path, data)` | **检查是否在沙箱中** |
| L27-32 | `batchMutations(...)` | **批量写操作走沙箱约束** |

**解耦难度**：⭐⭐

**替换方案**：提供 `enabled`（boolean 查询）、`batchMutations(effect)`（批量写优化）、`ensureDirectory(fs, path)`。

---

#### 🟡 耦合点 13：`packages/opencode/src/kilocode/tool/background-process.ts` — 后台进程

**耦合方式**：import `@kilocode/sandbox`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L9 | `import { enabled as sandboxed }` | 引入沙箱状态查询 |
| L128-129 | `if (action === "start" && (yield* sandboxed))` → 拒绝 | **沙箱开启时禁用后台进程** |

**解耦难度**：⭐

**替换方案**：提供 `enabled` boolean 查询即可。

---

#### 🟡 耦合点 14：`packages/opencode/src/kilocode/tool/agent-manager.ts` — Agent 管理器

**耦合方式**：import `@/kilocode/sandbox/inheritance`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L7 | `import * as SandboxInheritance` | 引入授权令牌 |
| L533 | `SandboxInheritance.issue({ sessionID, directory, count })` | **为跨目录子 Agent 发放一次性继承令牌** |

**解耦难度**：⭐

**替换方案**：提供 `issue({ sessionID, directory, count })` → 返回 token 字符串。

---

#### 🟡 耦合点 15：`packages/opencode/src/kilocode/server/httpapi/handlers/sandbox.ts` — HTTP API

**耦合方式**：import `@/kilocode/sandbox/activation` + `@/kilocode/sandbox/policy`

| 调用位置 | 调用内容 | 作用 |
|---------|---------|------|
| L3-4 | `import SandboxActivation, SandboxPolicy` | 引入激活和策略 |
| L21 | `SandboxActivation.idle(sessionID, family)` | **检查 session 家族是否空闲** |
| L28 | `SandboxPolicy.configuredSupport()` | 查询后端支持状态 |
| L30 | `SandboxPolicy.status(sessionID)` | 查询沙箱状态 |
| L33 | `SandboxPolicy.toggleGuarded(...)` | **带保护的 toggle（含空闲检查+家族传播）** |
| L53 | `SandboxActivation.family(sessionID)` | 获取 session 家族列表 |

**解耦难度**：⭐⭐⭐

**替换方案**：HTTP API 层需要完整的状态查询 + toggle 接口。

---

#### 🟢 耦合点 16：`packages/core/src/v1/config/config.ts` — 旧版配置 Schema

| 调用位置 | 内容 |
|---------|------|
| L143-163 | `sandbox` 配置 Schema 定义（enabled, network, writable_paths, allowed_hosts） |
| L321-331 | 旧版 `sandbox` boolean + `sandbox_restrict_network` + `sandbox_writable_paths` |

**解耦难度**：⭐（只是 Schema 定义，直接替换即可）

---

#### 🟢 耦合点 17：`packages/core/src/project/sql.ts` — 数据库 Schema

| 调用位置 | 内容 |
|---------|------|
| L16 | `sandboxes: DatabasePath.absoluteArrayColumn().notNull()` |

**解耦难度**：⭐（这是 project 表的 `sandboxes` 字段，存的是沙箱目录路径列表，与沙箱引擎无关）

---

#### 🟢 耦合点 18：VSCode 前端层 (`kilo-vscode`)

| 文件 | 耦合内容 | 解耦难度 |
|------|---------|---------|
| `sandbox-bootstrap.ts` | `client.sandbox.status()` / `client.sandbox.toggle()` | ⭐⭐ |
| `sandbox-session.ts` | `SANDBOX_METADATA_KEY = "kilocode.sandbox"`，session metadata 注入 | ⭐ |
| `KiloProvider.ts` | `sandbox.status` 事件监听，`handleSetSandboxDefault()` | ⭐⭐ |
| `provider-multi-version.ts` | `ensureSandbox()` 在 session 创建时同步沙箱状态 | ⭐⭐ |
| `agent-manager/tool-start.ts` | `sandboxInheritanceToken` 跨进程传递 | ⭐ |

---

### 三、解耦接口汇总（你需要实现的 API 清单）

```
┌─────────────────────────────────────────────────────────────────────┐
│              你需要替换/实现的 Sandbox API 接口                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  【引擎层 API】（替换 @kilocode/sandbox）                            │
│                                                                     │
│  1. decorateFileSystem(fs) → FileSystem         ← fs-util.ts        │
│     包装 Effect FileSystem，写操作受沙箱约束                          │
│                                                                     │
│  2. prepareCommand(cmd, cwd, env) → Command     ← cross-spawn.ts    │
│     spawn 前包装命令（加 sandbox-exec / bwrap / docker 前缀）         │
│                                                                     │
│  3. enabled → Effect<boolean>                   ← encoded-io.ts     │
│     当前是否在沙箱中                              ← background.ts    │
│                                                                     │
│  4. batchMutations(effect) → effect             ← encoded-io.ts     │
│     批量写操作优化（可选，不做就逐个执行）                              │
│                                                                     │
│  5. ensureDirectory(fs, path)                   ← encoded-io.ts     │
│     沙箱约束的 mkdir                                                │
│                                                                     │
│  6. assertNetwork(url) / assertSandbox(url)     ← network.ts       │
│     网络访问断言（工具分类用）                                        │
│                                                                     │
│  7. builtin(tool) / isBuiltin(tool)             ← registry.ts      │
│     工具标记系统                                                     │
│                                                                     │
│  8. remote(tool)                                ← mcp/index.ts     │
│     MCP 远程工具标记                                                 │
│                                                                     │
│  9. httpLayer → Layer                           ← registry.ts      │
│     沙箱感知的 HTTP Client Effect Layer                              │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  【策略层 API】（替换 opencode/kilocode/sandbox/）                   │
│                                                                     │
│  10. executeTool(sessionID, tool, effect) → effect  ← tools.ts     │
│      在沙箱约束下执行内建工具                                        │
│                                                                     │
│  11. executeMcp(sessionID, tool, effect) → effect   ← tools.ts     │
│      在沙箱约束下执行 MCP 工具                                       │
│                                                                     │
│  12. executeEscalated(approved, effect) → effect   ← shell.ts      │
│      提权旁路                                                       │
│                                                                     │
│  13. networkRestricted(sessionID) → boolean         ← code-mode.ts │
│      查询 session 是否网络受限                        ← prompt.ts   │
│                                                                     │
│  14. fallback(config) → Snapshot                    ← task.ts      │
│      从配置提取沙箱快照                                               │
│                                                                     │
│  15. inherit(parentID, childID, fallback?, dir?)    ← session.ts   │
│      子 session 继承父的沙箱策略                      ← task.ts     │
│                                                                     │
│  16. dispose(sessionID, effect) → effect            ← session.ts   │
│      删除 session 时清理沙箱状态                                      │
│                                                                     │
│  17. status(sessionID) → Status                     ← httpapi      │
│      查询沙箱状态（enabled, available, version）                      │
│                                                                     │
│  18. configuredSupport() → Support                  ← httpapi      │
│      查询后端支持状态                                                 │
│                                                                     │
│  19. toggle(sessionID) → Status                     ← httpapi      │
│      开关沙箱                                                       │
│                                                                     │
│  20. toggleGuarded(sessionID, guard, family, preflight)             │
│      带保护的 toggle（含空闲检查）                    ← httpapi      │
│                                                                     │
│  21. family(sessionID) → Target[]                   ← activation   │
│      获取 session 家族列表                                            │
│                                                                     │
│  22. idle(sessionID, family) → boolean              ← activation   │
│      检查 session 家族是否空闲                                        │
│                                                                     │
│  23. issue({ sessionID, directory, count }) → token ← agent-mgr    │
│      发放一次性继承令牌                                               │
│                                                                     │
│  24. scope(config, source) → config                 ← config.ts    │
│      配置 scope 限制（本地不能放宽全局）                                │
│                                                                     │
│  25. Snapshot 类型导出                              ← session.ts   │
│      { enabled, mode, allowedHosts, writablePaths, version }        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 四、最小替换策略（推荐顺序）

```
Phase 1: 引擎层替换（最核心，3 个文件）
  ├─ fs-util.ts        → 替换 decorateFileSystem()
  ├─ cross-spawn-spawner.ts → 替换 prepareCommand()
  └─ encoded-io.ts     → 替换 enabled / batchMutations / ensureDirectory

Phase 2: 策略层替换（1 个核心文件 + 辅助文件）
  ├─ session/tools.ts  → 替换 executeTool() / executeMcp()
  ├─ session/session.ts → 替换 inherit() / dispose()
  ├─ tool/task.ts      → 替换 fallback() / inherit()
  └─ tool/shell.ts     → 替换 executeEscalated()

Phase 3: 查询类替换（简单 boolean 查询）
  ├─ code-mode.ts      → 替换 networkRestricted()
  ├─ prompt.ts         → 替换 networkRestricted()
  ├─ background-process.ts → 替换 enabled
  └─ tool/registry.ts  → 替换 builtin() / isBuiltin() / httpLayer

Phase 4: HTTP API + VSCode 前端
  ├─ httpapi/handlers/sandbox.ts → 替换 status/toggle/family/idle
  └─ kilo-vscode/*    → 替换 sandbox SDK 调用
```

**核心结论**：真正难替换的只有 **3 个引擎层接口**（`decorateFileSystem`、`prepareCommand`、`executeTool`），其余 22 个接口要么是简单的 boolean 查询，要么是生命周期钩子，实现起来都很直接。

---
---

# Part 4: 引擎层三大核心接口技术难点

---

## 把 Sandbox 想象成一家"快递公司"

```
┌──────────────────────────────────────────────────────────────┐
│  快递公司（Kilo Code Sandbox）                                │
│                                                              │
│  前台（策略层）              仓库+车队（引擎层）               │
│  ├─ 接单登记（executeTool）   ├─ 分拣系统（decorateFileSystem）│
│  ├─ 查件（status）           ├─ 运输车（prepareCommand）      │
│  ├─ 开关门店（toggle）       └─ 安检门（网络代理/文件检查）    │
│  ├─ 继承关系（inherit）                                       │
│  └─ 空闲检查（idle）                                         │
└──────────────────────────────────────────────────────────────┘
```

**策略层的 22 个接口**，都是"前台"工作——登记、查询、开关门、查空闲。这些是**纯逻辑**，跟操作系统无关，用任何语言都能写。

**引擎层的 3 个接口**，是"仓库+车队"——它们直接跟**操作系统内核**打交道。这才是真正的技术壁垒。

---

## 逐个拆解：为什么这 3 个难

### 1. `decorateFileSystem(fs)` — 为什么难？

**它做的事**：把 Effect 框架的 `FileSystem` 服务"劫持"掉。

```
正常流程：
  Agent 写文件 → fs.writeFile("/project/output.txt", data) → 直接写磁盘

沙箱流程：
  Agent 写文件 → 被装饰过的 fs.writeFile(...)
    → 检查当前 Effect 上下文里有没有 Profile
    → 有 → 路径是否在 allowWrite 里？是否在 denyNames 里？
    → 通过 → 走 mutation worker（spawn 子进程，子进程也在沙箱里）
    → 不通过 → 抛 PermissionDenied 错误
```

**难在哪**：

| 难点 | 具体挑战 |
|------|---------|
| **Effect-TS 深度集成** | 不是简单包装一个 `fs.write()`。Effect 的 `FileSystem` 是一个 Service，用 `Context` 注入，用 `Layer` 组合。你的装饰器必须理解 Effect 的 `FileSystem`、`Sink`、`PlatformError` 等类型系统 |
| **透明性要求** | 上层代码（`encoded-io.ts`、`fs-util.ts`）调用 `fs.writeFile()` 时**完全不知道**自己被沙箱拦截了。你的装饰器必须做到接口 100% 兼容——读操作直通，写操作拦截，错误类型完全一致 |
| **Mutation Worker 协议** | 写操作不是直接执行，而是 spawn 一个子进程通过 stdin/stdout JSON 协议通信。你要实现整套 Request/Response 协议、batch 优化、错误映射 |
| **路径规范化** | 符号链接解析、`.git` 名称扫描、canonicalize——这些要在 Effect 的 `Effect<string, PlatformError>` 类型里完成 |

**对比**：策略层的 `networkRestricted(sessionID)` 只是从 Map 里读一个 boolean，一行代码。

---

### 2. `prepareCommand(cmd, cwd, env)` — 为什么难？

**它做的事**：在每个 shell 命令 **spawn 之前**，把命令"改装"成沙箱版本。

```
原始命令：
  /bin/sh -c "npm install"

macOS 沙箱改装后：
  /usr/bin/sandbox-exec \
    -p '(version 1)(deny default)(allow process-exec)...' \
    -DALLOW_WRITE_0=/project \
    -DALLOW_WRITE_1=/tmp/kilo-xxx \
    -- /bin/sh -c "npm install"

Linux 沙箱改装后：
  /usr/bin/bwrap \
    --unshare-user --unshare-pid --unshare-net \
    --ro-bind / / \
    --bind /project /project \
    --ro-bind /project/.git /project/.git \
    -- /bin/sh -c "npm install"
```

**难在哪**：

| 难点 | 具体挑战 |
|------|---------|
| **OS 内核知识** | 你必须理解 macOS `sandbox-exec` 的 SBPL 语法（一种极其冷门的 Apple 专用策略语言），或者 Linux `bwrap` 的 namespace 语义（user/pid/net namespace 各自隔离什么） |
| **安全正确性** | 不是"能跑就行"。如果 SBPL 策略写错一个括号，Agent 就能逃出沙箱。bwrap 的 `validate()` 函数要检查：可执行文件本身不在可写路径内、可写根目录不含嵌套挂载点、`.git` 被递归扫描并保护——任何一个检查遗漏都是安全漏洞 |
| **Shell 语义处理** | `launch.shell` 可能是 `true`、`string`、或 `undefined`，每种情况的命令拼接方式不同。引号转义、参数拼接必须精确 |
| **网络代理集成** | proxy 模式下，还要在命令前插入 network-relay 进程、注入 `HTTP_PROXY` 环境变量、挂载 Unix socket——这些都在 `prepareCommand` 里完成 |
| **跨平台分支** | `process.platform === "darwin"` 走 seatbelt，`"linux"` 走 bubblewrap，`"win32"` 直接不可用。你的替代方案也需要处理跨平台 |

**对比**：策略层的 `inherit(parentID, childID)` 只是从父 session 读一个 JSON，写到子 session——纯数据操作。

---

### 3. `executeTool(sessionID, tool, effect)` — 为什么难？

**它做的事**：这是**所有工具执行的总入口**，它把策略管理和引擎层**缝合**在一起。

```typescript
// 简化后的核心逻辑（policy.ts 的 execute 函数）：
function execute(sessionID, effect) {
  // 1. 初始化 snapshot（首次调用时从 3 个数据源决策初始状态）
  yield* snapshot(sessionID)
  
  // 2. 拿执行门控（100万 permits 的信号量，确保 toggle 时能阻塞）
  return yield* gated(sessionID, 1, Effect.gen(function* () {
    // 3. 获取当前状态（可能触发 config reconcile）
    const active = yield* current(sessionID, true)
    
    // 4. 未启用 → 直通
    if (!active.state.enabled) return yield* unrestricted(effect)
    
    // 5. 检查后端是否可用
    const support = backendSupport(...)
    if (!support.available) throw new Error(...)
    
    // 6. 构建 Profile，注入 Effect 上下文，执行
    return yield* runSandbox(
      profile(instanceCtx, mode, writablePaths, allowedHosts),
      effect  // ← 这个 effect 内部会调用 decorateFileSystem 和 prepareCommand
    )
  }))
}
```

**难在哪**：

| 难点 | 具体挑战 |
|------|---------|
| **三重并发控制** | `locks`（状态变更互斥）、`refreshes`（config reconcile 互斥）、`gates`（执行门控）——三个信号量协同工作，任何一个死锁都会卡住整个系统 |
| **Profile 构建** | `profile()` 函数要收集项目目录、worktree、8 个 Global.Path、用户自定义路径，构建 denyWrite/denyNames，剥离敏感环境变量——这些都要跟你的 sandbox 方案匹配 |
| **Effect 上下文注入** | `runSandbox(profile, effect)` 把 Profile 注入 Effect 的 `CurrentProfile` 上下文，然后 effect 内部所有的 `fs.writeFile`、`http.get`、`spawn` 都从这个上下文读取约束。这是 Effect-TS 的 Fiber 级依赖注入，不是全局变量 |
| **网络代理生命周期** | proxy 模式下，`withProxy()` 要在 effect 执行前启动代理服务器（`acquire`），执行后关闭（`release`）——用 Effect 的 `acquireUseRelease` 保证异常安全 |
| **Config 热更新** | `current(sessionID, true)` 在每次工具执行时检查 config 版本（`revision`），如果配置变了就自动 reconcile——但只更新 mode/hosts/paths，不改 enabled |

**对比**：策略层的 `status(sessionID)` 只是从 Map 读一个对象返回——5 行代码。

---

## 一句话总结

```
策略层接口 = 数据管理（CRUD + 查询）
  → 任何会写 TypeScript 的人都能实现
  
引擎层接口 = 操作系统级安全隔离 + Effect-TS 深度集成
  → 需要同时懂：
     ① OS 沙箱机制（seatbelt / bwrap / 你自己的方案）
     ② Effect-TS 的 Context/Layer/FileSystem/PlatformError 类型系统
     ③ 并发控制（信号量、门控、acquireUseRelease）
     ④ 安全工程（fail-closed、路径规范化、DNS rebinding 防护）
```

**所以**：如果你要用自己的 sandbox 方案（比如 Docker、nsjail、Firejail、或者自研方案），你需要重点攻克的是：

1. **怎么拦截 Effect 的 FileSystem 写操作** → 你的 `decorateFileSystem`
2. **怎么在 spawn 前包装命令** → 你的 `prepareCommand`
3. **怎么把这两者缝合进 Effect 的上下文注入模型** → 你的 `executeTool`

其余 22 个接口都是"照着搬"的活。

---

## 附录：涉及的核心源文件清单

### 引擎层 (`packages/kilo-sandbox/src/`)

| 文件 | 行数 | 职责 |
|------|------|------|
| `profile.ts` | 30 | 三维约束画像类型定义 |
| `context.ts` | 77 | Effect Fiber 级 Profile 注入 |
| `backend.ts` | 125 | 平台选择器（darwin/linux/win32） |
| `seatbelt.ts` | 83 | macOS sandbox-exec 后端 |
| `seatbelt-base.ts` | 113 | SBPL 基础策略模板 |
| `seatbelt-network.ts` | 22 | SBPL 网络策略规则 |
| `bubblewrap.ts` | 365 | Linux bwrap 后端 |
| `network.ts` | 152 | 网络拦截层 |
| `proxy.ts` | 302 | HTTP/HTTPS 代理服务器 |
| `mutation.ts` | 234 | 文件写操作委托 |
| `mutation-protocol.ts` | 125 | Worker 通信协议 |
| `kilo-sandbox-mutation-worker.ts` | 141 | Worker 进程入口 |
| `destination.ts` | 73 | 网络目标解析和验证 |
| `path.ts` | 93 | 路径规范化和匹配 |
| `kilo-sandbox-network-relay.ts` | 55 | Linux 网络中继进程 |

### 策略管理层 (`packages/opencode/src/kilocode/sandbox/`)

| 文件 | 行数 | 职责 |
|------|------|------|
| `policy.ts` | 633 | 核心协调器（三重信号量 + Profile 构建 + 执行流） |
| `store.ts` | 94 | 持久化存储（atomic write + schema 验证） |
| `state.ts` | 92 | Session 元数据（数据库事务） |
| `activation.ts` | 100 | 空闲检测 + 家族传播 |
| `config.ts` | 62 | 配置解析（scope 限制） |
| `preference.ts` | 36 | 目录级偏好 |
| `network.ts` | 41 | 工具网络分类 |
| `network-tools.ts` | 11 | opaque/host 工具列表 |
| `inheritance.ts` | 41 | 一次性授权令牌 |
| `git.ts` | 114 | Git 命令分类（只读/变更） |
| `event.ts` | 16 | 状态变更事件定义 |
