# QwenPaw E2E 评测基建与 CI 验证 深度解析

> **来源**：基于 QwenPaw 代码仓 `e2e/` 工程与 `.github/workflows/` 全量源码分析
> **分析范围**：`e2e/`（Playwright + pytest 框架）、`.github/workflows/`（28 个 CI workflow）、`scripts/github/`（PR 质量门禁）
> **本文档包含五部分内容**：
> 1. E2E 框架总体架构
> 2. 数据隔离与安全护栏
> 3. Page Object 层与 Fixture 工厂
> 4. Mock 体系与双运行模式
> 5. CI 验证矩阵与 PR 质量门禁

---

## 目录

### Part 1: E2E 框架总体架构
- 技术栈与目录结构
- 配置系统（BrowserConfig / ServerConfig）
- 标记体系：优先级 × 模块 × 元数据三维标注

### Part 2: 数据隔离与安全护栏
- QWENPAW_WORKING_DIR 强制隔离
- 隔离测试服务器的启动脚本
- 为什么"拒绝启动"比"警告"更重要

### Part 3: Page Object 层与 Fixture 工厂
- 26 个页面对象的组织
- 懒加载 Fixture 工厂模式
- 用例台账：172 用例的模块分布

### Part 4: Mock 体系与双运行模式
- ui_smoke 模式：Playwright route 拦截
- integration 模式：真实后端
- requires_llm 标记与模型依赖

### Part 5: CI 验证矩阵与 PR 质量门禁
- tests.yml 五段式门禁
- E2E 专项流水线（smoke / integration / nightly 四级）
- real-behavior-proof：PR 真实行为证明
- 桌面端与发布链路验证

---

# Part 1: E2E 框架总体架构

## 1.1 技术栈与目录结构

QwenPaw 的前端 E2E 采用 **Playwright + pytest + Page Object 模式**，独立成 `e2e/` 工程：

```
e2e/
├── config/
│   └── settings.py            # 统一配置管理（环境变量覆盖）
├── pages/                     # Page Object 层（26 个页面对象）
│   ├── base_page.py           # 基础页面对象类
│   ├── chat_page.py  agents_page.py  channels_page.py ...
├── mocks/                     # API Mock（Playwright route 拦截）
│   ├── auth.py  agents.py  channels.py  sessions.py ...（12 个）
├── fixtures/                  # pytest fixtures
├── utils/
│   ├── helpers.py
│   └── report_generator.py    # Markdown 测试报告自动生成
├── tests/                     # 23 个测试文件 / 172 用例
│   └── conftest.py
├── scripts/
│   ├── start_test_server.sh   # 隔离测试服务器启动
│   └── stop_test_server.sh
├── conftest.py                # 全局 fixture 与页面对象注册
├── pytest.ini                 # 标记体系定义
├── E2E_COVERAGE_REPORT.md     # 用例覆盖台账
└── requirements.txt
```

## 1.2 配置系统

`e2e/config/settings.py` 用 dataclass 组织配置，支持环境变量覆盖：

**BrowserConfig**——浏览器行为配置：

```python
@dataclass
class BrowserConfig:
    browser_type: str = "chromium"
    headless: bool = True
    viewport_width: int = 1920
    viewport_height: int = 1080
    slow_mo: int = 0        # 慢放模式（毫秒），调试用
    timeout: int = 30000
    args: list = field(default_factory=lambda: [
        "--no-sandbox",
        "--disable-features=TranslateUI",   # 禁用翻译弹窗
        "--disable-notifications",
        "--no-first-run",
        ...
    ])
```

启动参数里有一处细节值得注意：`--disable-features=TranslateUI`。注释说明——被测系统为英文界面，Chrome 检测到语言不匹配会弹出"是否翻译此页"，遮挡元素、劫持焦点，导致测试假失败。**UI 自动化测试的稳定性有一半来自对这类环境噪声的逐一清除**。

**ServerConfig**——被测服务配置：

```python
@dataclass
class ServerConfig:
    base_url: str = "http://localhost:7077"
    api_base_url: str = ""     # 留空则使用 base_url + /api
    model_key: str = ""        # 模型连接测试用
```

7077 端口是 E2E 专用端口，与开发者日常使用的默认端口错开——又一个隔离细节。

## 1.3 标记体系：三维标注

`e2e/pytest.ini` 定义了三维标记系统：

```ini
# Marker design principles:
#   1) Priority tier: p0 / p1 / p2
#   2) Module tier: each test file gets a module marker
#   3) Metadata: use test_id for case IDs (e.g. @pytest.mark.test_id("P0-001"))
```

**维度一：运行模式**

| 标记 | 含义 |
|------|------|
| `ui_smoke` | UI 冒烟测试（Mock API，无需后端） |
| `integration` | 集成测试（需要运行中的后端） |
| `requires_llm` | 需要配置真实模型 Key（`QWENPAW_DASHSCOPE_API_KEY` 为空时自动跳过） |

**维度二：优先级**：`p0` / `p1` / `p2`（与后端集成测试同一套语义）

**维度三：模块**：23 个模块标记（`chat` / `agents` / `channels` / `skills` / `mcp` / `acp` / `backups` 等），与 `tests/test_*.py` 文件一一对应。

`pytest.ini` 里还保留了历史子标记（如 `sessions_core`、`skills_tag`）并明确标注 `(legacy)`，新用例统一用顶层模块标记——**标记体系的演化有文档化的迁移策略，避免标签膨胀**。

另外 `--strict-markers` 被强制开启：任何未注册的标记都会导致收集失败，杜绝拼写错误造成的静默漏测。

---

# Part 2: 数据隔离与安全护栏

## 2.1 强制隔离：拒绝启动而非警告

这是整个 E2E 基建中最硬核的设计。`e2e/README.md` 的警告：

> **WARNING — Test Isolation Required**
> E2E tests write seed data (inbox events, plan state, workspace files) into the backend's working directory. If that directory is your real `~/.qwenpaw` you will **corrupt your actual QwenPaw data**.
> The framework enforces this rule: if `QWENPAW_WORKING_DIR` is unset or points inside your home directory, pytest will **refuse to start** with a clear RuntimeError.

设计要点：

1. E2E 测试会向工作目录写入种子数据（inbox 事件、plan 状态、workspace 文件）——这是真实的副作用。
2. 如果工作目录是用户的真实 `~/.qwenpaw`，测试数据会**污染用户真实数据**。
3. 框架的选择不是打印警告，而是 **pytest 拒绝启动**（RuntimeError）。

**"拒绝启动"优于"警告"**：警告依赖人看到并遵守，在 CI 自动化场景下形同虚设；而拒绝启动把安全约束变成了物理约束。

## 2.2 隔离测试服务器启动脚本

`e2e/scripts/start_test_server.sh` 提供了一键隔离环境：

```bash
E2E_PORT="${QWENPAW_E2E_PORT:-7077}"
E2E_ROOT="/tmp/qwenpaw-e2e-test-work-dir"
E2E_WORKING_DIR="${E2E_ROOT}/working"
E2E_SECRET_DIR="${E2E_ROOT}/secret"
E2E_BACKUP_DIR="${E2E_ROOT}/backups"
PID_FILE="${E2E_ROOT}/qwenpaw-e2e.pid"

mkdir -p "$E2E_WORKING_DIR" "$E2E_SECRET_DIR" "$E2E_BACKUP_DIR"

export QWENPAW_WORKING_DIR="$E2E_WORKING_DIR"
export QWENPAW_SECRET_DIR="$E2E_SECRET_DIR"
export QWENPAW_BACKUP_DIR="$E2E_BACKUP_DIR"
export QWENPAW_AUTH_ENABLED=false
```

五个隔离维度：

| 维度 | 隔离手段 |
|------|---------|
| 工作目录 | `/tmp/qwenpaw-e2e-test-work-dir/working` |
| 密钥目录 | 独立 `secret/` 目录 |
| 备份目录 | 独立 `backups/` 目录 |
| 端口 | 7077（可覆盖） |
| 认证 | `QWENPAW_AUTH_ENABLED=false`（测试不走真实登录） |

脚本还做了防御性检查：启动前用 `lsof` 探测端口占用，若已有测试服务器在跑，提示先执行 `stop_test_server.sh` 而不是盲目杀进程。`--bg` 模式写 PID 文件，停止脚本凭 PID 精确回收。

---

# Part 3: Page Object 层与 Fixture 工厂

## 3.1 26 个页面对象

每个页面对象封装一个功能页面的定位器与操作方法，与产品的 11+ 个页面模块对齐：

```
pages/
├── base_page.py          # 基类：通用导航、等待、截图
├── chat_page.py          # 对话页
├── agents_page.py        # Agent 管理
├── channels_page.py      # 渠道配置
├── sessions_page.py      # 会话管理
├── skills_page.py / skill_pool_page.py   # 技能与技能池
├── mcp_page.py / tools_page.py           # MCP 与工具
├── memory_page.py        # 长期记忆
├── models_page.py        # 模型管理
├── cronjobs_page.py / heartbeat_page.py  # 定时任务与心跳
├── inbox_page.py / backups_page.py       # 收件箱与备份
├── coding_page.py / files_page.py        # Coding 模式与文件
├── acp_page.py / voice_page.py           # ACP 与语音
├── security_page.py / token_usage_page.py / agent_stats_page.py
├── environments_page.py / runtime_config_page.py
└── plugin_page.py
```

## 3.2 懒加载 Fixture 工厂

`e2e/conftest.py` 用一个工厂函数批量注册页面对象 fixture：

```python
def _make_page_fixture(import_path: str, class_name: str):
    """Factory: build a fixture from a Page class (lazy import)."""
    def _fixture(page):
        module = __import__(import_path, fromlist=[class_name])
        return getattr(module, class_name)(page)
    _fixture.__name__ = class_name
    return _fixture

channels_page = pytest.fixture(scope="function", name="channels_page")(
    _make_page_fixture("pages.channels_page", "ChannelsPage")
)
sessions_page = pytest.fixture(scope="function", name="sessions_page")(
    _make_page_fixture("pages.sessions_page", "SessionsPage")
)
# ... 其余 24 个同构注册
```

两个设计决策：

1. **懒导入**（`__import__`）：pytest 收集阶段不加载任何页面对象模块，26 个页面的导入成本只在真正使用时发生——收集速度不受页面数量影响。
2. **工厂消重复**：26 个 fixture 的定义从 26 段样板代码压缩为 26 行单行注册，新增页面只需加一行。

## 3.3 用例台账：172 用例的模块分布

`E2E_COVERAGE_REPORT.md` 是人工维护与自动生成结合的覆盖台账：

| 模块 | 测试文件 | P0 | P1 | P2 | 合计 |
|------|---------|----|----|----|----|
| Chat | test_chat.py | 4 | 4 | 3 | 11 |
| Agents | test_agents.py | 6 | 2 | 4 | 12 |
| Channels | test_channels.py | 3 | 5 | 2 | 10 |
| CronJobs | test_cronjobs.py | 2 | 2 | 4 | 8 |
| Environments | test_environments.py | 4 | 5 | 3 | 12 |
| Models | test_models.py | 4 | 4 | 2 | 10 |
| Runtime Config | test_runtime_config.py | 3 | 6 | 1 | 10 |
| Backups | test_backups.py | 4 | 4 | 2 | 10 |
| ACP | test_acp.py | 3 | 3 | 2 | 8 |
| ...（共 23 模块） | | | | | |
| **合计** | | **67** | **72** | **35** | **172** |

台账同时维护**每个用例的粒度信息**（测试类 → 测试方法 → 优先级 → 覆盖点描述），例如：

```
| TestMultiTurnConversation | test_multi_turn_context_awareness | P0 | 多轮对话+上下文记忆 |
| TestChatIMEInput | test_chat_ime_input | P2 | IME 组合事件处理 |
```

`utils/report_generator.py` 则负责从 pytest 运行结果自动生成 Markdown 报告（按模块映射展示名），形成"台账定义预期 → 运行产出实际 → 报告比对"的闭环。

值得注意的是用例设计覆盖了 **IME 输入法组合事件**这种本地化细节——对一个面向中文用户的桌面产品，输入法兼容是真实的崩溃源。

---

# Part 4: Mock 体系与双运行模式

## 4.1 ui_smoke 模式：Playwright route 拦截

`e2e/mocks/` 下 12 个模块（auth / agents / channels / sessions / backups / cron_jobs / environments / heartbeat / models / plugin_market / security / sidebar_sessions）各自提供 `register(page)` 函数，用 Playwright 的 route 机制拦截 API：

```python
def register(page: Page):
    """Register auth API route mocks."""

    def _handle_auth_status(route):
        route.fulfill(
            status=200,
            content_type="application/json",
            body=json.dumps({"enabled": True, "has_users": True}),
        )

    def _handle_auth_login(route):
        route.fulfill(
            status=200,
            content_type="application/json",
            body=json.dumps({
                "token": "mock-jwt-token-for-smoke-test",
                "username": "admin",
            }),
        )
```

**ui_smoke 模式不需要任何后端**：浏览器请求全部被前端层拦截并返回预设响应。它的定位是：

- 验证页面渲染、路由、表单交互、状态管理
- 可在纯前端 CI 环境运行（`npm run dev` 即可）
- 快速、零成本、零外部依赖

## 4.2 integration 模式：真实后端

`integration` 标记的用例走真实后端：启动隔离测试服务器（7077 端口、/tmp 工作目录），浏览器发真实请求。验证的是**前后端协议的真实对接**——API 字段名、错误码、流式响应、SSE 行为。

两种模式的分工：

| 维度 | ui_smoke | integration |
|------|----------|-------------|
| 后端 | 无（route 拦截） | 真实子进程 |
| 验证目标 | 前端渲染与交互 | 前后端协议对接 |
| 运行环境 | 任何有 Node 的机器 | 需 Python 环境 + 隔离目录 |
| 失败含义 | 前端代码回归 | 接口契约或后端逻辑回归 |

## 4.3 requires_llm：模型依赖的显式声明

涉及真实模型调用的用例打 `requires_llm` 标记——当 `QWENPAW_DASHSCOPE_API_KEY` 为空时自动跳过。这保证：

- 开源贡献者的本地环境（无 API Key）能跑全量冒烟
- 有 Key 的 CI/维护者环境能验证真实模型链路
- **测试成本可控**：真实模型调用只发生在声明了的用例上

---

# Part 5: CI 验证矩阵与 PR 质量门禁

## 5.1 tests.yml 五段式门禁

后端主流水线的门禁链（`.github/workflows/tests.yml`）：

```
spam-gate          PR 垃圾过滤（外部贡献者频率限制）
    ↓
approval-gate      maintainer 批准（environment: maintainer-approved）
    ↓
unit-tests         矩阵：py3.11/3.13 × ubuntu/macos/windows
contract-tests     契约测试
integrated-tests   集成测试（PR 只跑 p0，push 跑 p0+p1）
coverage-report    合并覆盖率数据（不重跑测试）
    ↓
test-summary       汇总判定：任一失败即整体失败
```

关键设计：

1. **门禁前置**：外部贡献者的测试消耗先被 `spam-gate` + `approval-gate` 拦截，维护者批准后才动用 CI 资源——开源项目防算力滥用的标准姿势。
2. **跨平台单测矩阵**：macOS / Linux / Windows 全覆盖（沙箱在三平台实现不同），Linux 额外安装 `bubblewrap` 并调整 AppArmor 配置。
3. **覆盖率零增量**：覆盖率数据只在单一矩阵组合（ubuntu + py3.13，tracer 开销最低）采集，`.coverage.*` 文件作为短生命周期 artifact 上传，`coverage-report` job 只合并渲染。
4. **集成层按事件分级**：PR 跑 `integration and p0`（约 2 分钟），push 到主干跑 `p0 or p1`，手动触发可用 `integration_marker` 参数任意覆盖。

## 5.2 E2E 专项流水线：四级体系

| 流水线 | 触发 | 内容 |
|--------|------|------|
| `e2e-smoke.yml` | console/** 或 e2e/** 变更的 PR/push | Playwright UI 冒烟（15 分钟超时） |
| `frontend-tests.yml` | console/** 变更 | Vitest 前端单测 |
| `e2e-integration.yml` | **仅手动触发**（可自定义 marker，默认 p0） | 复用 `_e2e-job.yml` 可复用 workflow |
| `full-tests-nightly.yml` | 定时 | 四级覆盖全量（含 E2E） |

`e2e-smoke.yml` 的执行链：

```
setup-node 20 + npm ci（console/）
    ↓
npm run dev & + wait-on http://localhost:5173（30s 超时）
    ↓
setup-python 3.12 + pip cache + playwright install chromium --with-deps
    ↓
运行 Playwright UI 冒烟
```

**可复用 workflow**（`_e2e-job.yml`）把 E2E 执行体抽成一个共享 job，`e2e-integration.yml` 与 nightly 流水线都以 `uses: ./.github/workflows/_e2e-job.yml` 复用，只传不同的 `test_marker`——E2E 环境搭建（前端构建 + 后端启动 + Playwright 安装）的成本只维护一份。

## 5.3 real-behavior-proof：PR 真实行为证明

这是一道独特的**贡献质量门禁**，从 openclaw 项目移植：

> Check that PRs from external contributors include a real problem description and validation evidence in the PR body. Maintainer and bot PRs are auto-skipped. Complements pr-spam-gate (which filters by volume) with a **quality filter**.

分工明确：`pr-spam-gate` 管**数量**（频率限制），`real-behavior-proof` 管**质量**（证据要求）。

策略实现（`scripts/github/real_behavior_proof_policy.py`）解析 PR body 的 Markdown 结构：

```python
class ProofStatus(str, Enum):
    PASSED = "passed"     # 有真实问题描述 + 验证证据
    MISSING = "missing"   # 缺少必要章节
    SKIPPED = "skipped"   # 维护者/机器人/豁免标签

def _extract_sections(body: str) -> dict[str, str]:
    """Split a PR body into a {heading: body} mapping.
    Headings inside fenced code blocks are NOT treated as section
    boundaries. ..."""
```

几个对抗性细节：

1. **HTML 注释遮蔽**：模板里注释掉的占位文字不算"作者撰写的内容"。
2. **代码围栏状态机**：```` ``` ```` 内的标题不被当作章节分隔——如果证据部分就是一段围栏里的终端输出，它依然算有效证据。（注释还记录了移植时踩过的坑：最初的正则把整个围栏块删掉了，见 #6626。）
3. **安全执行**：workflow 用 `ref: ${{ github.workflow_sha }}` 检出**工作流所在版本**而非 PR head，永不执行不受信任的 PR 代码；`persist-credentials: false` 收紧权限。
4. **降级处理**：fork PR 无权打标签，失败时仅报告；同仓 PR 才会被加 `needs-context` 标签。

这套机制把"改了什么"必须配"证据证明确实修好了"变成了硬约束——对 Agent 这种**行为难以目视验证**的产品尤其重要。

## 5.4 桌面端与发布链路验证

除测试流水线外，验证矩阵还覆盖交付链路：

- `fork-verify-desktop.yml`：fork 提交验证桌面构建可行性（不接触密钥）
- `desktop-build.yml` → `desktop-promote.yml` → `desktop-publish.yml`：Tauri 桌面端的构建、晋级、发布三段式
- `scripts/verify/desktop_verify.py` + `launch_tauri_macos.sh` / `launch_tauri_windows.ps1`：桌面端本地验证工具
- `release-verify.yml` + `release-duty.yml`：发布验证与发布值班轮转
- `pr-ai-review.yml`：AI 辅助代码评审
- `codeql.yml`：静态安全扫描

## 5.5 设计哲学总结

1. **安全约束物理化**：会污染用户数据的操作不是警告而是拒绝启动——护栏必须是物理的，不是劝说的。
2. **隔离的多维性**：端口、工作目录、密钥目录、备份目录、认证开关，五个维度同时隔离才算隔离。
3. **测试分级运行**：同一套用例按标记组合出冒烟/集成/夜间/手动四种运行形态，基建一份、策略多份。
4. **质量门禁双层化**：数量门禁（防刷）与质量门禁（证据要求）分离，对开源协作场景缺一不可。
5. **对抗性思维贯穿**：从 Chrome 翻译弹窗到 HTML 注释伪装，测试基建假设一切都会出错，逐个封堵。
6. **台账即契约**：`E2E_COVERAGE_REPORT.md` 把 172 个用例的优先级与覆盖点写成可审计的台账——覆盖率不只是数字，是"测了什么"的明细。

---

## 附：CI Workflow 全清单速查

| Workflow | 职责 |
|----------|------|
| `tests.yml` | 后端主流水线（单测/契约/集成/覆盖率五段门禁） |
| `frontend-tests.yml` | Vitest 前端单测 |
| `e2e-smoke.yml` | Playwright UI 冒烟 |
| `e2e-integration.yml` | E2E 集成（手动触发） |
| `_e2e-job.yml` | E2E 可复用执行体 |
| `full-tests-nightly.yml` | 夜间全量四级覆盖 |
| `creator-app-tests.yml` | Creator 应用测试 |
| `pr-ai-review.yml` / `pr-spam-gate.yml` / `pr-under-review.yml` / `pr-welcome.yml` | PR 门禁与协作流 |
| `real-behavior-proof.yml` | PR 真实行为证据检查 |
| `fork-verify.yml` / `fork-verify-desktop.yml` | Fork 构建验证 |
| `desktop-*.yml`（5 个） | 桌面端构建/晋级/发布/Release |
| `docker-release.yml` / `publish-pypi.yml` | 容器与 PyPI 发布 |
| `release-verify.yml` / `release-duty.yml` | 发布验证与值班 |
| `codeql.yml` / `pre-commit.yml` / `npm-format.yml` | 安全扫描与格式检查 |
| `issue-welcome.yml` / `first-time-contributor-welcome.yml` | 社区协作流 |
| `deploy-website.yml` | 官网部署 |
