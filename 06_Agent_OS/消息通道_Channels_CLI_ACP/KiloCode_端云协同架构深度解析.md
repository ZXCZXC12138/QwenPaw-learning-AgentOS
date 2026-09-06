# Kilo Code 端云协同架构深度解析

> 基于 kilocode-main 源码的逐文件分析，涵盖四层端云架构、六个场景故事、并发控制、IM 通道四个详细场景、KiloClaw 云端 Agent 五大设计原则完整代码示例、与 QwenPaw 深度对比。

---

## 目录

- [一、全景拓扑：端云协同的四层架构](#一端云协同的四层架构)
- [二、六个场景故事：从开机到多端协同](#二六个场景故事从开机到多端协同)
  - [场景 1：开机——唤醒本地引擎](#场景-1开机唤醒本地引擎)
  - [场景 2：登录——拿到云端通行证](#场景-2登录拿到云端通行证)
  - [场景 3：发指令——一次完整的 Agent 循环](#场景-3发指令一次完整的-agent-循环)
  - [场景 4：换电脑——Cloud Sessions 会话同步](#场景-4换电脑cloud-sessions-会话同步)
  - [场景 5：KiloClaw——云端替身](#场景-5kiloclaw云端替身)
  - [场景 6：代码补全——毫秒级云端快通道](#场景-6代码补全毫秒级云端快通道)
  - [总结：一张图看全六幕](#总结一张图看全六幕)
- [三、并发控制：端侧云侧同时发指令怎么办？](#三并发控制端侧云侧同时发指令怎么办)
  - [3.1 架构隔离：冲突在源头被消除](#31-架构隔离冲突在源头被消除)
  - [3.2 SessionRunCoordinator：同 Session 串行，跨 Session 并发](#32-sessionruncoordinator同-session-串行跨-session-并发)
  - [3.3 Location 检查：每个 Turn 前的位置校验](#33-location-检查每个-turn-前的位置校验)
  - [3.4 LifecycleConflict：SQLite 幂等写入](#34-lifecycleconflicsqlite-幂等写入)
- [四、同步失败的四种场景与恢复策略](#四同步失败的四种场景与恢复策略)
  - [4.1 Session 移动（Location Mismatch）](#41-session-移动location-mismatch)
  - [4.2 输入生命周期冲突（LifecycleConflict）](#42-输入生命周期冲突lifecycleconflict)
  - [4.3 网络断开导致的事件丢失](#43-网络断开导致的事件丢失)
  - [4.4 Cloud Sessions 导入的数据一致性](#44-cloud-sessions-导入的数据一致性)
- [五、IM 通道与五端协同](#五im-通道与五端协同)
  - [5.1 五端拓扑](#51-五端拓扑)
  - [5.2 KiloClaw 的 IM 通道](#52-kiloclaw-的-im-通道)
  - [5.3 Kilo for Slack（独立于 KiloClaw）](#53-kilo-for-slack独立于-kiloclaw)
  - [5.4 移动端 App](#54-移动端-app)
  - [5.5 场景故事：手机上给 KiloClaw 发 Telegram 消息](#55-场景故事手机上给-kiloclaw-发-telegram-消息)
  - [5.6 场景故事：手机 App 连接桌面端的 CLI](#56-场景故事手机-app-连接桌面端的-cli)
  - [5.7 场景故事：Slack 里 @Kilo 直接开 PR](#57-场景故事slack-里-kilo-直接开-pr)
  - [5.8 通道架构详解：配对机制与通道注册表](#58-通道架构详解配对机制与通道注册表)
  - [5.9 五端协同工作流：小明的一天](#59-五端协同工作流小明的一天)
- [六、KiloClaw 云端 Agent 架构](#六kiloclaw-云端-agent-架构)
  - [6.1 全景拓扑](#61-全景拓扑)
  - [6.2 五大设计原则](#62-五大设计原则)
  - [6.3 状态机](#63-状态机)
  - [6.4 事件驱动通信](#64-事件驱动通信)
- [七、设计哲学总结：与 QwenPaw 的对比](#七设计哲学总结与-qwenpaw-的对比)

---

## 一、端云协同的四层架构

Kilo Code 的端云协同不是"一个端 + 一个云"，而是**六个层次、三种粒度**的渐进式云端化。

```
┌─ 四层端云架构 ──────────────────────────────────────────────────┐
│                                                                   │
│  第 1 层：进程级（IDE ↔ CLI Backend）                              │
│  ┌──────────┐  fork 子进程  ┌──────────────────┐                  │
│  │ VS Code  │ ────────────→ │ CLI Backend       │                  │
│  │ (UI 壳)  │  SSE 长连接   │ node opencode     │                  │
│  │          │ ←──────────── │ serve --port 0    │                  │
│  └──────────┘               │ → localhost:54321  │                  │
│                             └──────────────────┘                  │
│                                                                   │
│  第 2 层：LLM 代理级（CLI Backend ↔ Kilo Gateway）                 │
│  ┌──────────────────┐  HTTPS POST  ┌──────────────────┐           │
│  │ CLI Backend      │ ────────────→ │ Kilo Gateway     │           │
│  │ (本地引擎)        │  Bearer JWT   │ api.kilo.ai      │           │
│  │                  │ ←──────────── │ → 路由到 Anthropic│           │
│  └──────────────────┘  SSE 流式     └──────────────────┘           │
│                                                                   │
│  第 3 层：数据级（本地 ↔ Cloud Sessions）                           │
│  ┌──────────────────┐  导出/导入   ┌──────────────────┐           │
│  │ 本地 SQLite      │ ───────────→ │ ingest.          │           │
│  │ ~/.kilocode/     │              │ kilosessions.ai  │           │
│  │ db.sqlite        │ ←─────────── │                  │           │
│  └──────────────────┘              └──────────────────┘           │
│                                                                   │
│  第 4 层：运行时级（本地 ↔ KiloClaw）                               │
│  ┌──────────────────┐  WebSocket  ┌──────────────────┐           │
│  │ VS Code / Mobile │ ──────────→ │ KiloClaw 沙箱    │           │
│  │                  │  REST API   │ Fly.io           │           │
│  │                  │ ←────────── │ 24/7 Agent       │           │
│  └──────────────────┘             └──────────────────┘           │
│                                                                   │
│  五个云端服务 URL：                                                │
│  • api.kilo.ai          — LLM 代理 + 认证 + 配额                  │
│  • ingest.kilosessions.ai — 会话数据上传/下载                      │
│  • chat.kiloapps.io     — KiloClaw REST API                      │
│  • events.kiloapps.io   — KiloClaw WebSocket 事件流               │
│  • api.kilo.ai/api/openrouter — OpenRouter 代理                   │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### 关键源文件索引

| 文件 | 职责 |
|------|------|
| `packages/kilo-gateway/src/server/routes.ts` (817 行) | Gateway 所有路由定义 |
| `packages/kilo-gateway/src/auth/device-auth.ts` (194 行) | 设备授权流程 |
| `packages/kilo-gateway/src/cloud-sessions.ts` (348 行) | 云端会话导入/导出 |
| `packages/kilo-gateway/src/api/constants.ts` (109 行) | API 端点常量 |
| `packages/kilo-vscode/src/kiloclaw/KiloClawProvider.ts` (1297 行) | KiloClaw VS Code 面板 |
| `packages/kilo-vscode/src/kiloclaw/event-service-client.ts` (385 行) | WebSocket 事件客户端 |
| `packages/kilo-vscode/src/kiloclaw/token-manager.ts` (108 行) | Token 缓存管理 |
| `packages/core/src/session/run-coordinator.ts` (105 行) | Session 并发协调器 |
| `packages/core/src/session/input.ts` (289 行) | Session 输入管理 |
| `packages/core/src/session/context-epoch.ts` (201 行) | 上下文纪元管理 |

---

## 二、用一个故事讲透：六个场景的完整流程

### 场景 1：开机——唤醒本地引擎

```
小明打开 VS Code，Kilo Code 扩展激活。此时发生的第一件事：

VS Code 进程（UI 线程）
    │
    │  "我需要 Agent 引擎"
    │
    ▼
ServerManager.startServer()
    │
    │  fork 一个子进程
    │  命令: node opencode serve --port 0
    │  （--port 0 = 让操作系统随机分配空闲端口）
    │
    ▼
CLI Backend 子进程启动
    │
    ├─ 加载 SQLite 数据库（~/.kilocode/db.sqlite）
    ├─ 初始化 Effect 服务图（Global Scope 单例）
    ├─ 启动 Hono HTTP Server
    │   → 监听在 http://localhost:54321
    │
    ▼
VS Code 收到端口号 54321
    │
    │  SDK Client 连接到 http://localhost:54321
    │  建立 SSE 事件流（长连接）
    │
    ▼
状态栏显示: ● Connected
```

**类比**：这就像打开 Chrome 浏览器。Chrome 窗口是"端"，但它背后启动了一个"渲染引擎进程"。你看到的 UI 和实际干活的引擎是两个独立进程，通过内部协议通信。

**关键点**：VS Code 扩展本身**不运行任何 Agent 代码**，它只是一个 UI 壳。真正的引擎是 spawn 出来的 CLI Backend 子进程。这就是"第一层端云"——**IDE 是端，CLI Backend 是云**。

### 场景 2：登录——拿到云端通行证

```
小明第一次使用，需要登录 Kilo 账号。他点击 "Sign in with Kilo"：

VS Code 点击 "Sign in"
    │
    │  POST http://localhost:54321/kilo/auth/login
    │  → CLI Backend 收到请求
    │
    ▼
CLI Backend（Device Auth Flow）
    │
    │  ① 向 Kilo Cloud 请求设备码
    │  POST https://api.kilo.ai/api/device-auth/codes
    │  ← { code: "ABCD-1234", url: "https://kilo.ai/activate", expiresIn: 900 }
    │
    │  ② 返回给 VS Code
    │  ← { verificationUrl, code }
    │
    ▼
VS Code 弹出通知
    │  "请在浏览器中打开 https://kilo.ai/activate"
    │  "输入验证码: ABCD-1234"
    │  自动打开浏览器 ──────────────────────→ 浏览器打开 kilo.ai/activate
    │                                        │
    │                                        │ 小明登录 GitHub/Google
    │                                        │ 输入验证码 ABCD-1234
    │                                        │ 点击 "Authorize"
    │                                        │
    │  ③ CLI Backend 开始轮询               │
    │  GET https://api.kilo.ai/api/device-auth/codes/ABCD-1234
    │  ← 202 Pending...                     │
    │  GET ...（3 秒后重试）                  │
    │  ← 202 Pending...                     │
    │  GET ...（3 秒后重试）                  │
    │  ← 200 { token: "eyJhbG...", userEmail: "ming@example.com" }
    │                                        │
    │  ④ 获取用户资料                        │
    │  GET https://api.kilo.ai/api/profile   │
    │  ← { email, name, organizations: [{id:"org_123", name:"Acme Corp"}] }
    │                                        │
    │  ⑤ 小明选择组织 "Acme Corp"            │
    │  POST http://localhost:54321/kilo/organization
    │  { organizationId: "org_123" }         │
    │                                        │
    │  ⑥ Token 存入本地 Auth Store（SQLite）  │
    │  { type:"oauth", access:"eyJ...", accountId:"org_123" }
    │
    ▼
状态栏显示: ● ming@example.com (Acme Corp)
```

**类比**：这就像酒店入住。你（VS Code）走到前台（CLI Backend），前台帮你打电话给总部（Kilo Cloud）确认身份。总部给了你一张房卡（JWT Token），前台把房卡存好，以后每次你需要服务时，前台出示房卡就行。

**关键点**：Token 存在**本地**，但验证在**云端**。CLI Backend 充当了"本地管家"的角色——它替你保管云端通行证。

### 场景 3：发指令——一次完整的 Agent 循环

```
小明在编辑器里选中一段代码，输入："帮我重构这个函数，加上错误处理"

小明输入指令
    │
    ▼
VS Code（端）
    │  POST http://localhost:54321/sessions/prompt
    │  { sessionID: "ses_abc123", text: "帮我重构..." }
    │
    ▼
CLI Backend（端）—— Session Runner 启动
    │
    │  ① 提升用户输入到 Session History
    │  SessionInput.promoteSteers(db, sessionID)
    │
    │  ② 加载系统上下文（Context Epoch）
    │  SystemContextRegistry.load()
    │  → 并行加载: AGENTS.md + 技能列表 + 日期 + ...
    │  → 组合为 SystemContext
    │  → 与上一代 Snapshot 比较 → 无变化 → 复用 Baseline
    │
    │  ③ 解析模型
    │  SessionRunnerModel.resolve(session)
    │  → session.model = { providerID: "kilo", modelID: "kilo-auto/free" }
    │  → baseURL = "https://api.kilo.ai/api/openrouter"
    │  → headers = { Authorization: "Bearer eyJ...",
    │               X-KILOCODE-ORGANIZATIONID: "org_123" }
    │
    │  ④ 构建 LLM 请求
    │  LLM.request({
    │    model: { id: "kilo-auto/free", provider: "kilo",
    │             baseURL: "https://api.kilo.ai/..." },
    │    system: [{ type: "system", text: "You are Kilo Code..." }],
    │    messages: [翻译后的历史消息...],
    │    tools: [file_read, file_write, shell, search, ...]
    │  })
    │
    │  ⑤ 发起流式请求 ──────────────────────────────────┐
    │  llm.stream(request)                               │
    │                                                    │
    ▼                                                    ▼
CLI Backend                                      Kilo Cloud（云）
    │                                                    │
    │  HTTPS POST https://api.kilo.ai/api/openrouter/v1/chat/completions
    │  Headers:                                          │
    │    Authorization: Bearer eyJ...                    │
    │    X-KILOCODE-ORGANIZATIONID: org_123              │
    │    X-KILOCODE-EDITORNAME: Visual Studio Code 1.114 │
    │  Body:                                             │
    │    { model: "anthropic/claude-sonnet-4", ... }     │
    │                                                    │
    │                              Kilo Gateway 收到请求  │
    │                              │                      │
    │                              ├─ 验证 JWT Token      │
    │                              ├─ 检查组织配额        │
    │                              ├─ 记录审计日志        │
    │                              ├─ 路由到 Anthropic    │
    │                              │                      │
    │                              ▼                      │
    │                         Anthropic API                │
    │                              │                      │
    │                              ▼                      │
    │                         流式响应开始                  │
    │                              │                      │
    │  ← SSE 事件流返回 ──────────────────────────────────┘
    │
    │  ⑥ 处理流式事件
    │  event: text-delta → 发布到 SSE → VS Code 实时显示
    │  event: tool-call → 进入工具结算
    │
    │  ⑦ 工具执行（在本地！）
    │  tool: file_read("/src/utils.ts")
    │  → 直接读取本地文件系统
    │  → 结果截断到 2000 字符
    │
    │  ⑧ 带着工具结果，发起下一轮 Provider Turn
    │  → 回到步骤 ④
    │  → 直到 LLM 不再调用工具 → 循环结束
    │
    ▼
VS Code 收到最终响应
    │  SSE event: session.completed
    │  状态栏: ✓ Done
```

**类比**：这就像你（VS Code）对一个远程助手（CLI Backend）说"帮我改代码"。助手先翻了翻你的文件柜（本地文件系统），然后打电话给总部的 AI（通过 Kilo Gateway 调用 Anthropic），AI 说"我需要看一下 utils.ts"，助手又从文件柜里翻出来给 AI 看，AI 最终给出修改方案。

**关键点**：
- **LLM 调用走云端**（通过 Gateway 代理）
- **工具执行走本地**（读文件、写文件、执行 Shell 都在本地）
- **Gateway 是中间人**——它不看到你的文件内容，只负责转发 LLM 请求和计费

### 场景 4：换电脑——Cloud Sessions 会话同步

```
小明在公司电脑写了一半，回家想在家里电脑的 VS Code 继续。

公司电脑（端 A）
    │
    │  小明的 Session 已经进行了 20 轮对话
    │  数据存在本地 SQLite: ~/.kilocode/db.sqlite
    │
    │  小明点击 "Share Session"
    │  POST http://localhost:54321/sessions/export
    │  { sessionID: "ses_abc123" }
    │
    │  CLI Backend 导出完整会话数据
    │  → 上传到 https://ingest.kilosessions.ai
    │  → 云端存储: { info, messages: [{info, parts}], files }
    │
    ▼
Kilo Cloud（云）
    │  存储了 Session 快照
    │  session_id: "ses_abc123"
    │  包含: 20 轮对话 + 所有工具调用结果 + 文件附件
    │
    ▼
家里电脑（端 B）
    │
    │  小明打开 VS Code，登录同一个 Kilo 账号
    │
    │  点击 "Import Session"
    │  GET http://localhost:54321/kilo/cloud-sessions
    │  → CLI Backend → Kilo Cloud
    │  ← { cliSessions: [{ session_id: "ses_abc123", title: "重构 utils..." }] }
    │
    │  VS Code 展示会话列表
    │  小明选择 "重构 utils..."
    │
    │  POST http://localhost:54321/kilo/cloud/session/import
    │  { sessionId: "ses_abc123" }
    │
    │  CLI Backend 执行导入:
    │  ① 从云端下载完整会话数据
    │  ② 验证数据完整性（消息树无环、ID 无重复）
    │  ③ 重新生成所有 ID（避免与本地冲突）
    │  ④ 写入本地 SQLite
    │     INSERT INTO sessions ...
    │     INSERT INTO messages ... （20 轮 × N 条消息）
    │     INSERT INTO parts ...    （每条消息的 N 个部分）
    │  ⑤ 发布 SessionCreated 事件
    │
    ▼
家里电脑的 VS Code 显示完整会话历史
    │  20 轮对话完整恢复
    │  小明可以继续工作
```

**类比**：这就像 Google Docs。你在公司电脑上编辑文档，文档保存在 Google 云端。回家打开另一台电脑，从 Google 云端下载最新版本继续编辑。

**关键点**：会话数据在**两端各存一份**（本地 SQLite + 云端），通过**导入**而非实时同步来保持一致性。

### 场景 5：KiloClaw——云端替身

```
小明要出差 3 天，但有一个复杂的重构任务需要持续执行。他不想让笔记本一直开着。

VS Code
    │
    │  小明点击 "Run in Cloud"
    │  GET http://localhost:54321/kilo/claw/status
    │
    ▼
CLI Backend → Kilo Cloud
    │
    │  KiloClaw Worker 收到请求
    │  ① 在 Fly.io 上启动一台沙箱虚拟机
    │     region: "iad" (华盛顿)
    │     size: { cpus: 4, memory_mb: 8192 }
    │
    │  ② 克隆小明的 Git 仓库到沙箱
    │
    │  ③ 在沙箱内启动一个完整的 Agent 引擎
    │     （和 CLI Backend 完全相同的 OpenCode Core）
    │
    │  ④ 返回状态
    │  ← { status: "running", sandboxId: "sb_xyz789" }
    │
    ▼
VS Code 获取直连凭证
    │  GET http://localhost:54321/kilo/claw/chat-credentials
    │  ← {
    │    token: "eyJ...",              // 小明的 JWT
    │    kiloChatUrl: "https://chat.kiloapps.io",
    │    eventServiceUrl: "wss://events.kiloapps.io"
    │  }
    │
    ▼
VS Code 直接连接到云端 Agent
    │
    │  REST API: https://chat.kiloapps.io
    │  → 发送指令: "重构 src/utils.ts，加上完整的错误处理"
    │
    │  WebSocket: wss://events.kiloapps.io
    │  → 实时接收 Agent 的执行事件
    │  → 文本增量、工具调用、进度更新...
    │
    ▼
KiloClaw 沙箱（Fly.io 云端）
    │
    │  Agent 引擎运行
    │  ├─ 读取文件（沙箱内的本地文件系统）
    │  ├─ 调用 LLM（通过内部 Gateway，不走公网）
    │  ├─ 执行工具（shell、file_write...）
    │  ├─ 循环 50 轮...
    │  │
    │  │  3 天后，重构完成
    │  │  Agent 状态: idle
    │  │
    │  ▼
    │  沙箱自动停止（节省资源）
    │  status: "stopped"
    │
    ▼
小明收到通知
    │  "重构任务已完成，点击查看结果"
    │
    │  小明 git pull 拉取云端 Agent 的修改
```

**类比**：这就像你雇了一个远程实习生（KiloClaw）。你把代码仓库的钥匙给他，他在远程服务器上干活。你可以通过微信（WebSocket）随时问他进度。他干完活你把成果拿回来。

**关键点**：KiloClaw 是**完全在云端运行的 Agent 实例**——它有自己的文件系统、自己的 Agent 引擎、自己的工具执行环境。它和 CLI Backend 用的是**同一套 OpenCode Core 代码**，只是运行位置不同。

### 场景 6：代码补全——毫秒级云端快通道

```
小明在编辑器里写代码，打了几行，停下来思考。Kilo Code 的自动补全触发了：

小明输入:
  function parseConfig(file: string) {
    const content = fs.readFileSync(file, 'utf-8')
    // ← 光标停在这里，等待补全

VS Code（端）
    │
    │  收集编辑器上下文:
    │  - 当前文件内容（光标前 + 光标后）
    │  - 最近浏览的 5 个代码片段
    │  - 编辑历史（diff）
    │  - 光标位置: line 3, col 4
    │
    │  POST http://localhost:54321/kilo/fim
    │  {
    │    prefix: "function parseConfig...",
    │    suffix: "  }\n}",
    │    provider: "kilo",
    │    model: "mistralai/codestral-2508"
    │  }
    │
    ▼
CLI Backend
    │
    │  POST https://api.kilo.ai/api/gateway/v1/fim
    │  Headers: { Authorization: Bearer eyJ...,
    │             X-KILOCODE-ORGANIZATIONID: org_123 }
    │  Body: { prefix, suffix, model: "mistralai/codestral-2508" }
    │
    ▼
Kilo Gateway（云）
    │
    │  路由到 Mistral Codestral API
    │  流式返回补全结果
    │
    ▼
VS Code 实时显示补全文本（灰色 ghost text）
    │  const config = JSON.parse(content)
    │  if (!config.version) {
    │    throw new Error("Missing version field")
    │  }
    │  return config
    │
    │  小明按 Tab 接受补全
```

**类比**：这就像输入法的联想词。你打了"你好"，输入法猜下一个词可能是"吗"。但 Kilo Code 的"联想"是云端的大模型在猜，而不是本地的小词典。

**关键点**：代码补全走的是**专用快速通道**（FIM endpoint），不是 Agent 对话通道。延迟要求是毫秒级的，所以 Gateway 会优先路由到速度最快的模型。

### 总结：一张图看全六幕

```
┌─ 小明的一天 ──────────────────────────────────────────────────┐
│                                                                │
│  第一幕: 开机                                                   │
│  VS Code ─spawn→ CLI Backend ─connect→ localhost:54321         │
│  [端 ↔ 端：进程级]                                             │
│                                                                │
│  第二幕: 登录                                                   │
│  CLI Backend ─Device Auth→ api.kilo.ai → JWT Token             │
│  [端 ↔ 云：认证级]                                             │
│                                                                │
│  第三幕: 写代码                                                 │
│  CLI Backend ─LLM请求→ Kilo Gateway ─代理→ Anthropic           │
│  工具执行 ← 本地文件系统                                        │
│  [端 ↔ 云：推理级]                                             │
│                                                                │
│  第四幕: 换电脑                                                 │
│  端A ─导出→ Cloud ─导入→ 端B                                   │
│  [端 ↔ 云：数据级]                                             │
│                                                                │
│  第五幕: 出差                                                   │
│  VS Code ─WebSocket→ KiloClaw (Fly.io 沙箱)                    │
│  [端 ↔ 云：运行时级]                                           │
│                                                                │
│  第六幕: 代码补全                                               │
│  VS Code ─FIM→ Kilo Gateway ─代理→ Mistral Codestral           │
│  [端 ↔ 云：毫秒级快通道]                                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**一句话总结**：Kilo Code 的端云协同不是"一个端 + 一个云"，而是**六个层次、三种粒度**的渐进式云端化——从进程级的 IDE↔Backend，到推理级的 LLM 代理，到数据级的会话同步，到运行时级的云端沙箱，每一层都可以独立启用或关闭。

---

## 三、并发控制：端侧云侧同时发指令怎么办？

### 3.1 架构隔离：冲突在源头被消除

KiloClaw（云侧）和 CLI Backend（端侧）是**两个完全独立的 Agent 引擎**：
- 不同的进程
- 不同的 SQLite 数据库
- 不同的 Session ID 空间
- 之间**没有共享状态**

它们之间唯一的联系是 JWT Token（用于 Kilo Gateway 计费），但 Token 不涉及执行状态。

**结论**：冲突在架构层面就被避免了。

### 3.2 SessionRunCoordinator：同 Session 串行，跨 Session 并发

```typescript
// run-coordinator.ts 核心接口
interface Coordinator<Key, E> {
  readonly run: (key: Key) => Effect.Effect<void, E>    // 启动或加入执行
  readonly wake: (key: Key) => Effect.Effect<void>      // 注册后续工作
  readonly interrupt: (key: Key) => Effect.Effect<void>  // 中断执行
}
```

**三条规则**：

| 规则 | 机制 | 效果 |
|------|------|------|
| 同一时刻只有一个人能拿笔 | `run()` 检查是否有 owner Fiber | 同 Session 串行执行 |
| 后来的人可以"预约" | `wake()` 设置 `pendingWake = true` | 当前 Turn 结束后自动继续 |
| 可以"抢笔" | `interrupt()` → `Fiber.interrupt(owner)` | 强制中断当前执行 |

### 3.3 Location 检查：每个 Turn 前的位置校验

```typescript
// runner/llm.ts — runTurnAttempt() 第一行
const session = yield* getSession(sessionID)
if (session.location.directory !== location.directory || 
    session.location.workspaceID !== location.workspaceID)
  return yield* Effect.interrupt  // ← 自我中断
```

**含义**：如果 Session 在 Provider Turn 执行期间被移动到其他目录，Runner 会在下一个 Turn 开始前检测到并自我中断。

### 3.4 LifecycleConflict：SQLite 幂等写入

```typescript
// input.ts — 利用 SQLite 唯一约束实现幂等
const stored = yield* db.insert(SessionInputTable)
  .values({...})
  .onConflictDoNothing()  // ← 唯一约束冲突时静默
  .returning({ id })
  .get()
if (!stored) return yield* Effect.die(new LifecycleConflict({ id: input.id }))
// → LifecycleConflict 不是真正的错误——它是"有人比我先到了"的信号
```

---

## 四、同步失败的四种场景与恢复策略

### 4.1 Session 移动（Location Mismatch）

| 项目 | 说明 |
|------|------|
| **触发条件** | Session 在一个 Turn 执行期间被移动到其他目录 |
| **检测机制** | 每个 Turn 前检查 `session.location` vs `location` |
| **恢复策略** | `Effect.interrupt` 自我中断 → 下次在新位置重新初始化 |
| **二次检查** | `context-epoch.ts` 的 `insert()` 在写入前再次验证 Session 位置 |

```typescript
// context-epoch.ts
class LocationMismatch extends Error {}
// insert() 里：
const placed = yield* db.select(...)
  .where(eq(SessionTable.directory, location.directory))
if (!placed) return yield* Effect.die(new LocationMismatch())
```

### 4.2 输入生命周期冲突（LifecycleConflict）

| 项目 | 说明 |
|------|------|
| **触发条件** | 两个客户端同时向同一 Session 发送相同 ID 的消息 |
| **检测机制** | SQLite `ON CONFLICT DO NOTHING` + 返回值检查 |
| **恢复策略** | 捕获 `LifecycleConflict` → 检查已有记录 → 内容一致则视为幂等成功 |
| **设计哲学** | 不是真正的错误，是"有人比我先到了"的信号 |

### 4.3 网络断开导致的事件丢失

| 项目 | 说明 |
|------|------|
| **触发条件** | WebSocket 连接断开（WiFi 切换、地铁等） |
| **检测机制** | WebSocket close 事件 + ping/pong 心跳 |
| **恢复策略** | 指数退避重连 → 重连后**刷新权威状态**（不重放丢失事件） |
| **设计哲学** | 实时流不可靠是事实，用刷新代替重放 |

```typescript
// event-service-client.ts — 重连逻辑
private scheduleReconnect(): void {
  const base = Math.min(30_000, 1000 * 2 ** this.reconnectAttempts)
  const delay = base * (0.5 + Math.random() * 0.5)  // 随机抖动防雪崩
  ...
}

// 重连后刷新权威状态
private async refreshOnReconnect(): Promise<void> {
  await this.refreshConversations()      // 重新拉取会话列表
  await this.refreshActiveMessages()     // 重新拉取当前会话消息
}
```

**CONTEXT.md 的设计规则**：
> "events.subscribe() does not automatically reconnect after transport loss.
>  The live-only stream fails with ClientError; consumers refresh authoritative 
>  state before explicitly opening a new subscription because events missed 
>  during disconnection cannot be replayed."

### 4.4 Cloud Sessions 导入的数据一致性

| 项目 | 说明 |
|------|------|
| **触发条件** | 导入的云端 Session 数据不完整或格式异常 |
| **检测机制** | Zod schema + superRefine 检查消息树完整性 |
| **恢复策略** | 验证失败则拒绝导入；事务保证原子性（全写入或全回滚） |

**验证清单**：
- ✗ 消息 ID 重复？→ "Duplicate message ID"
- ✗ parentID 指向不存在的消息？→ "Dangling message parent"
- ✗ parentID 形成环？→ "Circular message parent"
- ✗ 压缩尾巴指向不存在的消息？→ "Dangling compaction tail"
- ✗ 工具附件的 sessionID 不匹配？→ "Invalid tool attachment"

---

## 五、IM 通道与五端协同

### 5.1 五端拓扑

```
┌─ Kilo Code 的五端拓扑 ──────────────────────────────────────────────┐
│                                                                       │
│  ① 桌面端: VS Code / JetBrains / TUI → 本地 CLI Backend              │
│  ② 移动端: iOS App（审核中）/ Android App（已上架）                    │
│  ③ Web 端: app.kilo.ai（Kilo Chat + Dashboard）                      │
│  ④ IM 通道: Telegram / Discord / Slack（通过 KiloClaw Bot）           │
│  ⑤ 云端 Agent: KiloClaw（Fly.io 沙箱，24/7 运行）                     │
│                                                                       │
│  所有端共享: 同一 Kilo 账号 + 同一 Gateway 余额 + 同一 JWT Token       │
│  但各自独立: 各自的 Session 和 Agent 引擎                              │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### 5.2 KiloClaw 的 IM 通道

| 通道 | 配置方式 | 支持场景 |
|------|---------|---------|
| **Kilo Chat** | 零配置，默认启用 | Web + Mobile + VS Code |
| **Telegram** | BotFather Bot Token | DM + 群组（需配置 groupPolicy） |
| **Discord** | Discord Developer Portal Bot Token | DM + Server Channel |
| **Slack** | App-Level Token (xapp-) + Bot Token (xoxb-) | DM + Channel |

**配对机制（安全关键）**：
1. 新用户首次发消息 → 生成配对码
2. Dashboard 显示配对请求 → 用户点击 "Approve"
3. 连接建立 → 后续消息直接路由到 Agent

**OpenClaw 通道配置示例**：
```json
{
  "channels": {
    "telegram": {
      "groupPolicy": "allowlist",
      "groups": { "-1001234567890": { "requireMention": true } }
    },
    "discord": {
      "dmPolicy": "allowlist",
      "allowFrom": ["987654321098765432"],
      "groupPolicy": "disabled"
    },
    "slack": {
      "groupPolicy": "allowlist",
      "channels": { "C01234567": { "requireMention": true } }
    }
  }
}
```

### 5.3 Kilo for Slack（独立于 KiloClaw）

Kilo for Slack 是一个**独立的 Slack Bot**，不依赖 KiloClaw：
- 在 Slack 里 `@Kilo` 提问 → 读取仓库上下文 → 回答或开 PR
- 支持跨仓库操作、调试、代码审查
- 通过 app.kilo.ai 的 Integrations 页面配置

### 5.4 移动端 App

核心能力：
- 查看/管理 Session（包括远程 CLI 和扩展 Session）
- 创建 Cloud Agent Session
- 发送后续消息（排队处理）
- "Run on" 选择器：Cloud Agent（默认）/ 连接的 CLI 实例
- 文件双向传输（手机 ↔ CLI，最大 20 MiB / 4 MiB）
- GitHub PR 审查（diff、评论、合并）

### 5.5 场景故事：手机上给 KiloClaw 发 Telegram 消息

```
小明在地铁上，掏出手机打开 Telegram

  小明（手机 Telegram）
    │  "帮我看看今天的邮件有没有紧急的"
    │
    ▼
  Telegram Bot API（云端）
    │  转发消息给 KiloClaw 沙箱
    │  → 通过 Bot Token 认证
    │
    ▼
  KiloClaw 沙箱（Fly.io）
    │  OpenClaw Agent 引擎
    │  ├─ 读取 Google 邮箱（通过 Google OAuth）
    │  ├─ 筛选紧急邮件
    │  ├─ 生成摘要
    │  │
    │  ▼
    │  通过 Telegram Bot 回复小明：
    │  "有 3 封紧急邮件：
    │   1. AWS 账单异常告警
    │   2. 生产环境 500 错误
    │   3. 客户 A 的合同到期提醒"
    │
    ▼
  小明在手机上看到回复
    │  "帮我处理第 2 个，500 错误"
    │
    ▼
  KiloClaw 继续执行
    │  ├─ 访问监控系统
    │  ├─ 分析错误日志
    │  ├─ 定位 bug
    │  ├─ 创建修复分支
    │  ├─ 提交代码
    │  ├─ 开 PR
    │  │
    │  ▼
    │  "已创建 PR #347，修复了空指针异常，
    │   需要你 review 后合并"
```

### 5.6 场景故事：手机 App 连接桌面端的 CLI

```
小明在公司电脑上用 VS Code 写代码
  → kilo remote 启动
  → 本地 CLI Backend 暴露远程连接

小明下班后打开手机上的 Kilo Code App（Android）
  │
  │  首页 → "Run on" 选择器
  │  → 看到小明的 CLI 实例
  │  → 选择 "小明的 MacBook Pro"
  │
  │  新建 Session
  │  → 模式: Code
  │  → 模型: kilo-auto/free
  │  → 输入: "把刚才那个函数加上缓存"
  │
  ▼
  Kilo Cloud（中继）
    │  将消息转发到小明的 CLI Backend
    │  → SSE 事件流双向传输
    │
    ▼
  小明的 MacBook Pro（桌面端）
    │  CLI Backend 执行
    │  → 读取本地文件系统
    │  → 调用 LLM（通过 Kilo Gateway）
    │  → 修改代码
    │
    ▼
  手机 App 实时显示
    │  Agent 正在读取 src/cache.ts...
    │  Agent 正在修改 src/cache.ts...
    │  ✓ 完成
    │
    │  小明在手机上看到修改结果
    │  发送文件: cache.ts（4 MiB 以内）
    │  → send_file 工具 → 手机上打开分享面板
```

### 5.7 场景故事：Slack 里 @Kilo 直接开 PR

这是一个**独立于 KiloClaw** 的功能——**Kilo for Slack**：

```
团队在 Slack 频道讨论
  │
  │  同事 A: "payment 模块有个 bug，null pointer 异常"
  │  同事 B: "我看了日志，是 order 对象没初始化"
  │
  │  小明: "@Kilo 根据这个讨论，修复 order 模块的空指针异常"
  │
  ▼
  Kilo for Slack Bot（云端）
    │  ① 读取 Slack 线程上下文
    │  ② 访问连接的 GitHub 仓库
    │  ③ 定位 payment 模块
    │  ④ 分析 null pointer 原因
    │  ⑤ 创建修复分支
    │  ⑥ 提交代码
    │  ⑦ 开 PR
    │
    ▼
  Slack 回复:
    │  "已创建 PR #1234，修复了 order 模块的空指针异常。
    │   修改内容：在 OrderService.process() 中添加了 null 检查。
    │   链接: https://github.com/..."
    │
    │  团队在 Slack 里直接 review
```

### 5.8 通道架构详解：配对机制与通道注册表

**配对机制**是安全的关键：

```
Telegram 用户发消息给 Bot
  │
  ▼
KiloClaw 收到消息
  │  "这是第一次从这个 Telegram 用户收到消息"
  │  → 生成配对码: "Your pairing code is: 7842"
  │  → 回复给 Telegram 用户
  │
  ▼
Dashboard 显示配对请求
  │  "Telegram 用户 @xiaoming 请求连接，配对码: 7842"
  │  → 小明在 Dashboard 上点击 "Approve"
  │
  ▼
连接建立
  │  Telegram 用户 ↔ KiloClaw Agent
  │  后续消息直接路由到 Agent
```

**KiloClaw 通道注册表**：

```
┌─ KiloClaw 通道架构 ─────────────────────────────────────────────┐
│                                                                   │
│  KiloClaw 沙箱（Fly.io）                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  OpenClaw Agent 引擎                                  │        │
│  │                                                       │        │
│  │  Channel Registry（通道注册表）                        │        │
│  │  ├─ Kilo Chat（第一方，零配置）                        │        │
│  │  │   → chat.kiloapps.io（REST）                       │        │
│  │  │   → events.kiloapps.io（WebSocket）                │        │
│  │  │   → 不需要 Bot Token                               │        │
│  │  │                                                     │        │
│  │  ├─ Telegram（第三方）                                  │        │
│  │  │   → BotFather Bot Token                            │        │
│  │  │   → Telegram Bot API                               │        │
│  │  │   → 支持 DM + 群组                                 │        │
│  │  │                                                     │        │
│  │  ├─ Discord（第三方）                                   │        │
│  │  │   → Discord Developer Portal Bot Token             │        │
│  │  │   → Discord Gateway API                            │        │
│  │  │   → 支持 DM + Server Channel                       │        │
│  │  │                                                     │        │
│  │  └─ Slack（第三方）                                     │        │
│  │      → App-Level Token (xapp-) + Bot Token (xoxb-)    │        │
│  │      → Slack Socket Mode                              │        │
│  │      → 支持 DM + Channel                              │        │
│  │                                                       │        │
│  └──────────────────────────────────────────────────────┘        │
│                                                                   │
│  配对流程（Pairing）:                                              │
│  新设备/新用户首次连接 → 生成配对码 → Dashboard 审批                │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### 5.9 五端协同工作流：小明的一天

```
┌─ 小明的一天 ─────────────────────────────────────────────────────┐
│                                                                   │
│  09:00  桌面端（VS Code）                                         │
│         在公司电脑上开始写代码                                      │
│         Session: ses_desktop_001                                  │
│         → 本地 CLI Backend 执行                                   │
│                                                                   │
│  12:00  移动端（手机 App）                                         │
│         午休时打开手机                                             │
│         → "Run on: 小明的 MacBook Pro"                            │
│         → 继续上午的 Session                                      │
│         → 发送: "把那个函数加上错误处理"                            │
│         → 手机实时看到 Agent 执行过程                              │
│                                                                   │
│  14:00  Web 端（app.kilo.ai）                                     │
│         回到公司，打开浏览器                                        │
│         → Kilo Chat 查看 KiloClaw 的邮件摘要                      │
│         → Dashboard 查看 Cloud Agent 状态                         │
│                                                                   │
│  17:00  IM 通道（Telegram）                                        │
│         下班路上                                                   │
│         → Telegram 给 KiloClaw Bot 发消息                         │
│         → "帮我总结一下今天的代码变更"                              │
│         → KiloClaw 通过 Telegram 回复摘要                         │
│                                                                   │
│  20:00  云端 Agent（KiloClaw）                                     │
│         小明在看电视                                               │
│         → KiloClaw 在云端持续运行                                 │
│         → 自动执行定时任务: 每天 20:00 生成日报                    │
│         → 通过 Telegram 推送日报给小明                             │
│                                                                   │
│  所有端共享:                                                       │
│  - 同一个 Kilo 账号                                               │
│  - 同一个 Kilo Gateway 余额                                       │
│  - 同一个 JWT Token（认证）                                       │
│  - 但各自独立的 Session 和 Agent 引擎                              │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

**一句话总结**：Kilo Code 已经构建了一个**"桌面 + 移动 + Web + IM + 云端"五端协同**的完整体系。其中 **KiloClaw 是核心枢纽**——它是一个 24/7 运行的云端 Agent，通过 4 个 IM 通道接收指令，手机端 App 可以连接云端 Agent 或桌面端 CLI，Web 端提供 Dashboard 管理。这比 QwenPaw 的"全栈本地"模式多了一个完整的**云端常驻 Agent + 多 IM 通道**维度。

---

## 六、KiloClaw 云端 Agent 架构

### 6.1 全景拓扑

```
┌─ 用户侧（多端入口）───────────────────────────────────┐
│  VS Code Webview → KiloClawProvider                   │
│  Mobile App → Kilo Chat                               │
│  Web Browser → Kilo Chat                              │
│  Telegram/Discord/Slack → OpenClaw Channel            │
└────────────────────────┬──────────────────────────────┘
                         │
┌─ Kilo 云端（中继层）───┼──────────────────────────────┐
│  Kilo Gateway ── 认证 + 计费                          │
│  kilo-chat ──── 会话状态管理                           │
│  event-service ── WebSocket 事件分发                   │
│  kiloclaw ────── 沙箱生命周期管理                      │
│  notifications ── 移动推送                             │
└────────────────────────┬──────────────────────────────┘
                         │
┌─ Fly.io 沙箱（专属机器）─┼────────────────────────────┐
│  OpenClaw Agent 引擎    ▼                             │
│  ├─ Agent Core（循环 + 工具执行）                      │
│  ├─ Channel Registry（Telegram/Discord/Slack）        │
│  ├─ Tool Profiles（fs/shell/browser/...）              │
│  ├─ Skills / Cron Jobs / Memory / Sub-agents          │
│                                                       │
│  硬件: 2 vCPUs · 3 GB RAM · 10 GB persistent SSD     │
│  网络: 只绑定 loopback，外部流量经 Kilo 控制器代理      │
│  安全: 敏感数据静态加密                                │
└───────────────────────────────────────────────────────┘
```

### 6.2 五大设计原则

#### 原则 1：一人一机（Dedicated Machine）

每个用户独占一台虚拟机，不共享基础设施。

```
传统模式（多租户共享）:
  ┌─ 服务器 A ──────────────────────┐
  │  用户 X 的 Agent  │  用户 Y 的 Agent  │
  │  共享文件系统      │  共享文件系统      │
  │  共享进程空间      │  共享进程空间      │
  │  → 安全风险：一个 Agent 可以读另一个的数据  │
  └─────────────────────────────────┘

KiloClaw 模式（一人一机）:
  ┌─ 机器 X ──────────┐  ┌─ 机器 Y ──────────┐
  │  用户 X 的 Agent    │  │  用户 Y 的 Agent    │
  │  独立文件系统       │  │  独立文件系统       │
  │  独立进程空间       │  │  独立进程空间       │
  │  10 GB SSD 持久化   │  │  10 GB SSD 持久化   │
  │  → 物理级隔离       │  │  → 物理级隔离       │
  └───────────────────┘  └───────────────────┘
```

**哲学**：Agent 需要执行 Shell 命令、读写文件、操作浏览器——这些操作的破坏性远超传统 Web 应用。用"容器隔离"不够安全，用"虚拟机隔离"才能让 Agent 自由行动而不危及他人。

#### 原则 2：网络隔离 + 代理控制

```
沙箱内部:
  OpenClaw 绑定 127.0.0.1（loopback only）
  → 外部无法直接访问沙箱
  → 所有入站流量必须经过 Kilo 控制器代理

外部请求路径:
  用户 → Kilo Gateway → Kilo Controller → 沙箱 loopback
  Telegram Bot → OpenClaw Channel → 沙箱 loopback

出站流量:
  沙箱 → Kilo Gateway → LLM Provider（Anthropic/OpenAI/...）
  沙箱 → Google APIs（邮件/日历）
  沙箱 → GitHub API
```

**哲学**：沙箱是一个**受控的自主体**——它可以自由执行操作，但所有通信都经过代理层。代理层负责认证、计费、审计。这像一座有围墙的花园——花园里可以自由生长，但出入口有门卫。

#### 原则 3：乐观更新 + 权威刷新

这是 KiloClaw 客户端最精妙的设计模式。看 `KiloClawProvider.ts` 的消息发送逻辑：

```typescript
// 发送消息时的乐观更新
private async sendMessage(conversationId, content, inReplyToMessageId) {
  // ① 生成客户端 ID（ULID 格式）
  const clientId = ulid()
  const pendingId = `pending-${clientId}`
  
  // ② 立即在本地创建一条"待确认"消息
  const optimistic: Message = {
    id: pendingId,        // ← 临时 ID，带 pending- 前缀
    senderId: this.currentUserId,
    content,
    ...
  }
  this.messages = [...this.messages, optimistic]
  this.post({ type: "kiloclaw.messageOptimistic", ... })  // ← 立即显示
  
  // ③ 发送到服务器
  try {
    await this.chat.sendMessage({ conversationId, content, clientId, ... })
    // ④ 服务器会触发 message.created 事件
    //    → 事件中携带 clientId
    //    → 客户端用 clientId 匹配，将 pending 消息替换为服务器确认消息
  } catch (err) {
    // ⑤ 失败时移除乐观消息
    this.messages = this.messages.filter(m => m.id !== pendingId)
    this.post({ type: "kiloclaw.messageRemoved", ... })
  }
}

// 收到服务器事件时的调和
events.on("message.created", (ctx, e) => {
  // ⑥ 如果事件携带 clientId，找到对应的 pending 消息并替换
  if (e.clientId) {
    const pending = `pending-${e.clientId}`
    const idx = this.messages.findIndex(m => m.id === pending)
    if (idx !== -1) {
      this.messages = this.messages.map((m, i) => 
        i === idx ? serverMessage : m  // ← 用服务器完整消息替换
      )
      this.post({ type: "kiloclaw.messageReplaced", ... })
      return
    }
  }
  // ⑦ 如果没有匹配的 pending（可能是其他设备发的），直接追加
  this.messages = [...this.messages, serverMessage]
})
```

**哲学**：用户界面永远不应该等待网络。先发出去，再等确认。如果确认来了，无缝替换；如果失败了，悄悄回滚。这像微信——你发消息后立即看到，不用等服务器回复才显示。

#### 原则 4：断连自愈

`EventServiceClient` 的重连机制：

```typescript
// 指数退避重连（带随机抖动）
private scheduleReconnect(): void {
  const base = Math.min(30_000, 1000 * 2 ** this.reconnectAttempts)
  const delay = base * (0.5 + Math.random() * 0.5)  // ← 随机抖动防止雪崩
  this.reconnectAttempts++
  this.reconnectTimer = setTimeout(() => {
    this.connectOnce().catch((err) => {
      if (this.handleAuthFailure(err)) return  // ← 认证失败不重连
      if (!this.destroyed) this.scheduleReconnect()
    })
  }, delay)
}

// 重连后自动刷新权威状态
private async refreshOnReconnect(): Promise<void> {
  await this.refreshConversations()      // ← 重新拉取会话列表
  if (this.activeConversationId) {
    await this.refreshActiveMessages()   // ← 重新拉取当前会话消息
  }
}

// 重连后自动重新订阅上下文
ws.addEventListener("open", () => {
  const isReconnect = this.hasConnectedBefore
  this.connected = true
  this.reconnectAttempts = 0
  this.resubscribeContexts()   // ← 重新订阅之前订阅的所有 context
  if (isReconnect) {
    for (const h of this.reconnectHandlers) h()  // ← 触发刷新
  }
})
```

**关键设计**：重连后**不尝试重放丢失的事件**，而是直接刷新权威状态。因为：
1. 实时事件流（live stream）本质上不可靠——断了就断了
2. 权威状态（从 HTTP API 拉取）是可靠的——它永远是最新的
3. 重放事件需要服务端支持，增加了系统复杂度

**哲学**：接受不可靠，用刷新代替重放。这像 TCP 的"超时重传"——不追踪每个包，而是发现丢了就重新请求整个窗口。

#### 原则 5：分层认证 + 票据制

```
认证流程（两层）:

第一层：JWT Token（长期凭证）
  用户登录 → Device Auth Flow → JWT Token（有效期较长）
  TokenManager 缓存 Token，5 分钟新鲜度缓冲

第二层：一次性票据（短期凭证）
  WebSocket 连接时:
  ① POST /connect-ticket（带 JWT）→ 获取 30 秒有效的一次性票据
  ② WebSocket 升级（带票据）→ 服务端验证票据 → 建立连接
  
  为什么需要票据？
  → WebSocket 升级请求无法携带自定义 Header
  → 所以用 URL 参数 ?ticket=xxx 传递
  → 票据是一次性的，防止重放攻击
```

```typescript
// token-manager.ts — Token 缓存逻辑
async getOrFetch(): Promise<ChatToken> {
  // 缓存有效（距过期还有 5 分钟以上）→ 直接返回
  if (this.cached && Date.now() < this.expiresAtMs - FRESHNESS_BUFFER_MS) {
    return this.cached
  }
  // 最近失败过（5 秒内）→ 冷却，不重试
  if (this.lastFailedAt && Date.now() - this.lastFailedAt < RETRY_BACKOFF_MS) {
    throw new Error("on cooldown")
  }
  // 并发调用共享同一个 inflight Promise → 不会重复请求
  if (!this.inflight) {
    this.inflight = this.fetch().then(...)
  }
  return this.inflight
}
```

**哲学**：安全是分层的。长期凭证（JWT）用于身份认证，短期票据（Ticket）用于连接建立，两者职责分离。票据的 30 秒 TTL 和一次性使用，使得即使被截获也无法重放。

### 6.3 状态机

```
┌─ KiloClaw 实例生命周期 ──────────────────────────────────────┐
│                                                                │
│  provisioned ──start──→ starting ──ready──→ running            │
│       │                                       │                │
│       │                                  stop │  crash         │
│       │                                       ▼    ▼           │
│       │                                    stopped  ──recover──→ recovering
│       │                                       │                  │
│       │                                  start │                  │
│       │                                       ▼                  │
│       │                                  starting ←── restoring  │
│       │                                                          │
│       └──destroy──→ destroying（不可逆）                          │
│                                                                │
│  状态说明:                                                      │
│  provisioned  — 已创建，从未启动                                 │
│  starting     — 正在启动机器                                     │
│  running      — Agent 在线，可接收消息                            │
│  stopped      — 机器关闭，数据保留                               │
│  recovering   — 从异常停止中恢复                                 │
│  restoring    — 从快照恢复                                       │
│  destroying   — 永久删除（不可逆）                                │
│                                                                │
│  每个状态转换都会通过 WebSocket 推送给所有订阅的客户端            │
└────────────────────────────────────────────────────────────────┘
```

### 6.4 事件驱动通信

KiloClaw 的事件系统是一个**基于 Context 的发布-订阅模型**：

**Context 层级**：
- `/kiloclaw/{sandboxId}` — 沙箱级（会话列表变化）
- `/kiloclaw/{sandboxId}/{conversationId}` — 会话级（消息/输入状态）

**沙箱级事件**（订阅 `/kiloclaw/sb_123`）：

| 事件 | 说明 |
|------|------|
| conversation.created | 新会话创建 |
| conversation.renamed | 会话重命名 |
| conversation.left | 离开会话 |
| conversation.activity | 会话有新活动 |
| bot.status | Bot 上线/下线 |

**会话级事件**（订阅 `/kiloclaw/sb_123/conv_456`）：

| 事件 | 说明 |
|------|------|
| message.created | 新消息 |
| message.updated | 消息编辑 |
| message.deleted | 消息删除 |
| message.delivery_failed | 消息投递失败 |
| reaction.added / removed | 表情反应 |
| typing / typing.stop | 输入状态 |
| conversation.status | 会话状态变化（模型、Token 用量） |
| action.executed / delivery_failed | 执行审批 |

**设计哲学**：事件流是**分层的**——沙箱级事件变化慢（创建/重命名会话），会话级事件变化快（消息/输入状态）。客户端只订阅当前需要的 context，避免收到无关事件。切换会话时，取消旧 context 订阅，订阅新 context。

---

## 七、设计哲学总结：与 QwenPaw 的对比

| 维度 | Kilo Code（KiloClaw） | QwenPaw |
|------|----------------------|--------|
| **运行位置** | 云端 Fly.io 沙箱（24/7） | 本地进程（按需启动） |
| **隔离模型** | 一人一机（VM 级隔离） | 单进程多 Harness（进程级隔离） |
| **安全模型** | 网络隔离 + 代理控制 + 静态加密 | Hook 权限 + 沙箱（可选） |
| **持久化** | 10 GB SSD（区域固定） | 本地文件系统 |
| **生命周期** | 7 状态状态机（provisioned → running → stopped → ...） | 8 Phase 请求生命周期 |
| **通信模型** | WebSocket 事件流 + 乐观更新 | SSE 信封 + asyncio |
| **认证模型** | 两层（JWT + 一次性票据） | 单 JWT |
| **端云关系** | 云端是主体，端是入口 | 本地是主体，云是辅助 |
| **多端协同** | 多端共享同一个云端 Agent 状态 | 单端独占本地 Agent 状态 |
| **IM 通道** | 4 个（Kilo Chat + Telegram + Discord + Slack） | 0 |
| **移动端** | iOS/Android App | 无原生移动端 |
| **云端 Agent** | KiloClaw（Fly.io 沙箱，24/7 运行） | 无 |
| **会话同步** | Cloud Sessions（云端导入/导出） | 无（本地 SQLite 独占） |
| **代码补全** | FIM/Edit 专用端点（Codestral/Mercury） | 无独立补全服务 |
| **嵌入式** | Embedded OpenCode（同进程嵌入） | 无（必须独立进程） |

### 根本差异

- **KiloClaw 的哲学**：Agent 是一个**24/7 在线的数字助手**——它有自己的"家"（Fly.io 沙箱），有自己的"手机"（Telegram/Discord/Slack），有自己的"记忆"（持久化 SSD），有自己的"日程"（Cron Jobs）。用户通过各种入口（VS Code、手机 App、Web、IM）去找它说话。

- **QwenPaw 的哲学**：Agent 是一个**按需启动的计算引擎**——它活在用户的机器上，用户来了就启动，用户走了就停止。它通过 Hook/Mode/Plugin 获得能力，通过 Scroll 管理记忆，通过 StopGate 控制循环。

**一句话总结**：

KiloClaw 像雇了一个**全职远程助手**——他有自己的办公室（沙箱），有自己的电话（IM 通道），24/7 在线，你随时可以找他。QwenPaw 像请了一个**按需上门的顾问**——他来你的办公室（本地），用你的电脑（本地文件系统），干完活就走。两种设计没有优劣之分，只有场景不同——前者适合长期运行的个人助手，后者适合开发时的即时编码辅助。

### Kilo Code 并发控制四层防线

```
第 1 层：架构隔离（最根本）
  端侧和云侧是完全独立的引擎 → 冲突在架构层面被消除

第 2 层：Location 检查（每个 Turn 前）
  Runner 检查 Session 位置 → 不匹配则自我中断

第 3 层：SQLite 幂等（写入时）
  ON CONFLICT DO NOTHING + LifecycleConflict → 重复操作不会破坏数据

第 4 层：事件流分级（传输时）
  实时流（不可靠）vs 持久流（可重放）→ 断连后刷新权威状态
```

**一句话总结**：Kilo Code 不是通过"加锁"来解决并发冲突——它通过**架构隔离**让冲突不可能发生，通过**乐观检查**处理边界情况，通过**幂等写入**保证数据安全，通过**接受不可靠**简化网络处理。这是一种**"避免优于解决"**的设计哲学。
