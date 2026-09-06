# QwenPaw 测试金字塔与评测基准设计 深度解析

> **来源**：基于 QwenPaw 代码仓全量测试体系源码分析
> **分析范围**：`tests/`（unit / contract / integration / e2e 四层）+ `e2e/`（前端 E2E）+ `Makefile` + `.github/workflows/`
> **本文档包含五部分内容**：
> 1. 测试体系全景与规模盘点
> 2. 四层测试金字塔逐层拆解
> 3. 优先级基准（p0/p1/p2）设计哲学
> 4. 契约测试框架：防止"修一个子类，崩一片兄弟"
> 5. 评测基准的运行编排与质量门禁

---

## 目录

### Part 1: 测试体系全景
- 规模盘点：四层测试的量化数据
- 目录结构与职责边界
- 为什么 Agent 产品需要比传统软件更厚的测试层

### Part 2: 四层金字塔逐层拆解
- Layer 1: Unit 单元测试（32 个模块镜像目录）
- Layer 2: Contract 契约测试（接口合规性）
- Layer 3: Integration 集成测试（真实子进程 + 10 个 IM 协议 Mock）
- Layer 4: E2E 端到端测试（Playwright UI 自动化）

### Part 3: 优先级基准设计
- p0 / p1 / p2 的判定标准：用户影响面而非技术复杂度
- 三级基准的运行成本与触发时机

### Part 4: 契约测试框架
- BaseContractTest 抽象契约机制
- Channel 契约的四类验证
- 契约覆盖率检查脚本

### Part 5: 运行编排与设计哲学
- Makefile 命令矩阵
- pytest 全局 Fixture 的隔离设计
- 六条设计哲学总结

---

# Part 1: 测试体系全景

## 1.1 规模盘点

QwenPaw 的测试体系是一个四层金字塔结构，各层规模如下：

| 层级 | 目录 | 规模 | 职责 |
|------|------|------|------|
| **Unit** | `tests/unit/` | 32 个模块子目录（与源码模块一一对应） | 函数/类级别逻辑验证 |
| **Contract** | `tests/contract/` | channels / providers / security / browser 四类契约 + 通用 runner | 基类接口合规性，防子类回归 |
| **Integration** | `tests/integration/` | 122 个测试文件 | 真实子进程启动 FastAPI 应用，HTTP 冒烟 |
| **E2E（后端）** | `tests/e2e/` | 3 个文件 | 端到端场景 |
| **E2E（前端）** | `e2e/` | 23 个测试文件 / 26 个页面对象 / 172 个用例 | Playwright UI 自动化 |

几个关键数字：

- **单元测试目录是源码目录的镜像**：`tests/unit/` 下的 `agents/`、`runtime/`、`sandbox/`、`hub/`、`governance/`、`tool_calls/` 等 32 个子目录，与 `src/qwenpaw/` 的模块划分完全对齐。这意味着"看到源码模块就能找到它的测试"。
- **集成测试覆盖了 10 种 IM 协议的 Mock**：dingtalk、feishu、wecom、qq、telegram、wechat_ilink、matrix、mattermost、xiaoyi、yuanbao——每一个都有独立的 `mock_*.py` 模拟服务端。
- **E2E 用例总数 172 个**，其中 P0（核心）67 个、P1（重要）72 个、P2（边缘）35 个，覆盖 23 个功能模块。

## 1.2 目录结构

```
QwenPaw/
├── tests/
│   ├── conftest.py              # 全局 Fixture（隔离、Mock、自动标记）
│   ├── unit/                    # Layer 1：单元测试（32 个模块镜像目录）
│   │   ├── agents/  runtime/  sandbox/  hub/  governance/  ...
│   ├── contract/                # Layer 2：契约测试
│   │   ├── __init__.py          #   BaseContractTest 框架核心
│   │   ├── channels/            #   渠道契约（DingTalk/Console/...）
│   │   ├── providers/           #   模型提供商契约
│   │   ├── security/            #   安全组件契约
│   │   ├── browser/             #   浏览器组件契约
│   │   └── runner/              #   契约运行器（控制命令契约等）
│   ├── integration/             # Layer 3：集成测试（122 文件）
│   │   ├── mock_dingtalk_im.py  #   10 个 IM 协议 Mock 服务
│   │   ├── mock_feishu_im.py
│   │   ├── mock_wecom_gateway.py
│   │   ├── mock_telegram_api.py
│   │   ├── ... （共 10 个）
│   │   └── test_*.py            #   按 p0/p1/p2 标记的 HTTP 冒烟
│   └── e2e/                     # Layer 4a：后端端到端
├── e2e/                         # Layer 4b：前端 UI 自动化
│   ├── config/  pages/  mocks/  fixtures/  utils/  tests/
│   ├── E2E_COVERAGE_REPORT.md   # 172 用例覆盖台账
│   └── scripts/start_test_server.sh
└── Makefile                     # 测试命令编排入口
```

## 1.3 为什么 Agent 产品需要更厚的测试层

传统 Web 应用的测试重点是"输入 → 输出"的确定性逻辑。但 Agent 产品有三类特殊风险，迫使 QwenPaw 加厚测试金字塔的中间层：

1. **多态实现风险**：一个 `BaseChannel` 基类派生出钉钉、飞书、企微、QQ、Telegram 等十几个子类。修复基类的一个 bug 可能让某个从未被单独测试的子类在生产环境崩溃。→ **契约测试层**就是为此而生。
2. **外部依赖不可控风险**：LLM API、IM 平台 API 都有配额、限流、网络不确定性。单测里必须完全 Mock，集成测试里用本地模拟服务端替代。→ 10 个 `mock_*.py` 协议模拟器就是为此而生。
3. **真实行为回归风险**：Agent 的价值在于"真的能发消息、真的能跑起来"。纯 Mock 测试通过不代表产品可用。→ 集成测试用**真实子进程**启动完整应用、**真实 HTTP 请求**验证主路径，E2E 层再验证真实浏览器交互。

---

# Part 2: 四层金字塔逐层拆解

## 2.1 Layer 1: Unit 单元测试

### 目录镜像原则

`tests/unit/` 的 32 个子目录与 `src/qwenpaw/` 完全镜像：

```
tests/unit/                     src/qwenpaw/
├── agents/          ←→         ├── agents/
├── runtime/         ←→         ├── runtime/
├── loop/            ←→         ├── loop/
├── sandbox/         ←→         ├── sandbox/
├── security/        ←→         ├── security/
├── governance/      ←→         ├── governance/
├── hub/             ←→         ├── hub/
├── harnesses/       ←→         ├── harnesses/
├── drivers/         ←→         ├── drivers/
├── providers/       ←→         ├── providers/
├── tool_calls/      ←→         ├── tool_calls/
├── checkpoints/     ←→         ├── checkpoints/
├── token_usage/     ←→         ├── token_usage/
├── observability/   ←→         ├── observability/
├── local_models/    ←→         ├── local_models/
├── channels/        ←→         ├── (app/channels)
├── plugins/  pawapp/  backup/  tunnel/  cli/  config/  services/ ...
```

这个镜像结构的价值：**测试不是附加物，而是模块的第二公民**。新增一个源码模块，评审时立刻能发现缺少对应测试目录。

### 全局 Fixture 的隔离设计（`tests/conftest.py`，467 行）

单元测试的可信度取决于隔离程度。QwenPaw 的全局 conftest 提供了一套系统化的隔离设施：

**（1）第三方库缺失兜底**

```python
_MISSING_MODULES = {
    "aibot",     # WeCom AI Bot SDK
    "lark_oapi", # Feishu Lark SDK
}
for _module in _MISSING_MODULES:
    if _module not in sys.modules:
        sys.modules[_module] = MagicMock()
```

测试环境不装商业 SDK，用 MagicMock 顶替——保证单测零外部依赖。

**（2）环境隔离三件套**

| Fixture | 作用 |
|---------|------|
| `temp_workspace` | 临时工作目录，测完即删 |
| `temp_copaw_home` | 伪造的 HOME 环境（含 `.copaw/skills`、`.copaw/logs`），并**主动清除 9 个敏感环境变量**（OPENAI_API_KEY、DASHSCOPE_API_KEY、DINGTALK_APP_SECRET 等），防止测试意外打到真实 API |
| `clean_env` | 清除 CoPaw 专属环境变量 |

清除敏感 token 是一个容易被忽视但极其重要的设计：一旦测试带着真实 API Key 运行，不仅产生费用，还会让测试结果依赖网络状态。

**（3）领域 Mock 预配置**

`mock_llm_provider` 预配置了 `chat()` / `chat_stream()` / `embed()` / `complete()` 四个方法的返回值；`mock_channel` 预配置了 `send_message()` / `send_file()` 等。测试作者拿到的不是空 Mock，而是"开箱即用的假 LLM / 假渠道"，大幅降低写测试的门槛，同时保证 Mock 行为与真实接口一致。

**（4）Provider 单例隔离（autouse）**

```python
@pytest.fixture(autouse=True)
def isolated_secret_dir(monkeypatch, tmp_path):
    """Isolate all tests from real disk provider data."""
    secret_dir = tmp_path / ".qwenpaw.secret"
    monkeypatch.setattr(_provider_manager_module, "SECRET_DIR", secret_dir)
    monkeypatch.setattr(_provider_manager_module.ProviderManager,
                        "_instance", None)
```

`ProviderManager` 是全局单例，会从磁盘读持久化配置并修改自身状态（如 `base_url`）。这个 autouse fixture 对**每一个测试**强制重置单例并把密钥目录指向临时目录，防止测试之间通过全局状态互相污染。

**（5）自动标记机制**

```python
def pytest_collection_modifyitems(config, items):
    for item in items:
        path_str = item.path.as_posix()
        if "/unit/" in path_str:
            item.add_marker(pytest.mark.unit)
        elif "/integration/" in path_str:
            item.add_marker(pytest.mark.integration)
        elif "/e2e/" in path_str:
            item.add_marker(pytest.mark.e2e)
```

按目录自动打 `unit` / `integration` / `e2e` 标记，支持 `-m unit` 等过滤运行。代码注释里特别提到用 `as_posix()` 而非 `str()`——在 Windows 上 `str()` 返回反斜杠路径会导致标记静默失效，测试被静默跳过。这种跨平台细节正是成熟测试体系的标志。

## 2.2 Layer 2: Contract 契约测试

契约测试是 QwenPaw 测试金字塔中最有特色的一层，详见 Part 4。它解决的核心问题是：**多态子类家族的接口回归**。覆盖四类对象：

- `channels/`：所有 IM 渠道子类必须遵守 `BaseChannel` 契约
- `providers/`：模型提供商适配器的接口契约
- `security/`：安全组件契约
- `browser/`：浏览器组件契约
- `runner/`：运行时控制命令契约（`test_control_command_contract.py`）

## 2.3 Layer 3: Integration 集成测试

### 核心机制：真实子进程 + 随机端口 + 隔离工作区

集成测试的官方定义（`tests/integration/README.md`）：

> HTTP smoke tests that exercise the QwenPaw FastAPI app end-to-end via a **real subprocess**. Each test file owns its own QwenPaw app subprocess on a **random port**, with **isolated workspace directories** — no real API keys or external services required.

三个设计决策：

1. **真实子进程**：不是 TestClient 内存调用，而是 `python -m qwenpaw app` 真起一个进程。这能捕获进程级问题（端口占用、启动时序、信号处理）。
2. **每个测试文件独占一个子进程**（module-scoped `app_server` fixture）：文件内共享、文件间隔离，配合 `pytest-xdist --dist=loadscope` 可以并行跑而不冲突。
3. **无真实 API Key**：外部服务全部用本地 Mock 替代。

### 10 个 IM 协议 Mock 服务

这是集成层最重的资产——为每一个支持的 IM 平台写了协议级模拟器：

| Mock 文件 | 模拟平台 |
|-----------|---------|
| `mock_dingtalk_im.py` | 钉钉 |
| `mock_feishu_im.py` | 飞书 |
| `mock_wecom_gateway.py` | 企业微信 |
| `mock_qq_im.py` | QQ（OneBot 协议） |
| `mock_telegram_api.py` | Telegram Bot API |
| `mock_wechat_ilink.py` | 微信 iLink |
| `mock_matrix_hs.py` | Matrix 协议 |
| `mock_mattermost.py` | Mattermost |
| `mock_xiaoyi.py` | 小艺 |
| `mock_yuanbao.py` | 元宝 |

对应的集成测试（如 `test_dingtalk_mock_im.py`、`test_onebot_reverse_ws.py`）会启动 Mock 服务端 + QwenPaw 应用，走通"收到消息 → Agent 处理 → 回复消息"的完整链路。这比 Mock 掉整个 Channel 的单元测试置信度高一个量级。

### 并行执行策略

```bash
pytest tests/integration -n auto --dist=loadscope
```

`loadscope` 按模块分组——与 module-scoped 的 `app_server` fixture 精确匹配（一个子进程/测试文件，文件内共享）。README 还建议 `--no-cov` 跳过父进程覆盖率采集（子进程覆盖率另有专门机制）。

## 2.4 Layer 4: E2E 端到端测试

前端 E2E 是整个金字塔的塔尖，独立成 `e2e/` 工程（详见姊妹篇《QwenPaw_E2E评测基建与CI验证_深度解析》）。这里给出它在金字塔中的定位：

- **技术栈**：Playwright + pytest + Page Object 模式
- **规模**：23 个测试文件、26 个页面对象、172 个用例
- **两种运行模式**：
  - `ui_smoke`：Playwright route 拦截所有 API，纯前端冒烟，**不需要后端**
  - `integration`：真实后端 + 真实浏览器，需要隔离工作目录（`QWENPAW_WORKING_DIR`）
- **安全护栏**：如果 `QWENPAW_WORKING_DIR` 未设置或指向真实 `~/.qwenpaw`，pytest **拒绝启动**并抛出 RuntimeError——防止 E2E 测试的种子数据污染用户真实数据。

---

# Part 3: 优先级基准（p0/p1/p2）设计哲学

## 3.1 判定标准：用户影响面，而非技术复杂度

这是 QwenPaw 评测基准设计中最值得学习的一点。`tests/integration/README.md` 给出了明确的判定问句：

> Tests are tagged by **user-facing impact**, not technical complexity. When adding a test, ask:
> *"If this fails, can users still send a message and get a reply?"*
> Yes → `p1` or `p2`. No → `p0`.

**一条消息的收发是产品的生命线**，这个判据直接锚定了 p0 的边界。

## 3.2 三级基准定义

### p0 — Critical（PR 冒烟门禁）

**定义**：失败即产品基本不可用。每个 PR 必须通过。

覆盖范围：
- **消息主路径**：`/api/messages/send` 核心流程、默认 Agent 路由
- **Agent / Chat / Skills 核心 CRUD**：列表/创建/读取/删除、启停开关、系统提示词文件
- **全局配置**：channels、heartbeat、MCP CRUD、workspace running config
- **安全护栏（全局）**：file guard、tool guard、skill scanner
- **工具开关**：影响 Agent 运行时能力
- **API 版本**：基础健康检查

**运行成本**：`pytest -m p0` ≈ 22 个测试 ≈ 2 分钟。

### p1 — Supported（夜间 / 合并回归）

**定义**：失败导致功能降级，但默认配置下用户仍可完成基本任务。

覆盖范围：
- **设置与作用域覆盖**：语言、音频模式、时区、转录提供商，以及 channel/heartbeat/guards 的 scoped 版本
- **工作区文件**：working/memory 文件 CRUD、zip 上传下载、作用域一致性
- **ACP / LLM 路由**：开发者向功能
- **计划 / Cron**：辅助功能
- **统计类**：token usage、plugins/backups 列表、agent stats、auth 状态
- **辅助 API**：文件预览、Agent 排序、批量操作

**运行成本**：`pytest -m p1` ≈ 53 个测试。

### p2 — Contracts（广度覆盖）

**定义**：不影响主流程的边界行为。

覆盖范围：
- **校验拒绝路径**：`*_rejected` 测试（重名、非法载荷、非 zip 上传）
- **404 处理**：`*_returns_404`、`missing_*`
- **部分成功分支**：批量操作中的部分失败
- **隔离边界**：`*_isolated_*`、跨 Agent 边缘情况
- **HEAD 请求与最小契约**：`*_minimal_contract`
- **版本元数据**：包版本、PEP 440 合规

**运行成本**：`pytest -m p2` ≈ 30 个测试。

## 3.3 三级基准的运行编排

优先级标记的真正价值在于**按触发时机分层运行**：

| 触发时机 | 运行集合 | 目标 |
|---------|---------|------|
| **每个 PR** | p0（集成层）+ 全量单测 + 契约测试 | 2 分钟级反馈，阻断明显回归 |
| **合并 / 夜间** | p0 + p1（`full-tests-nightly.yml`） | 降级路径回归 |
| **发布前** | p0 + p1 + p2 全量 | 边界行为全覆盖 |

CI workflow（`.github/workflows/tests.yml`）还支持手动触发时自定义集成层标记表达式：

```yaml
workflow_dispatch:
  inputs:
    integration_marker:
      description: >-
        Pytest marker expression for the integration tier. Leave blank
        to keep the event-based default (PR=p0, push/dispatch=p0+p1).
```

即：PR 默认只跑 `integration and p0`，push/手动默认跑 `p0 or p1`，需要时可一键全量。**基准不是静态的测试集合，而是可调度的运行策略**。

## 3.4 E2E 层的平行基准

前端 E2E 采用同样的三级划分（`e2e/pytest.ini`）：

```ini
markers =
    p0: P0 - core functionality (must pass)
    p1: P1 - important functionality
    p2: P2 - minor functionality
```

且每个用例带 `test_id` 元数据（如 `@pytest.mark.test_id("P0-001")`），与 `E2E_COVERAGE_REPORT.md` 台账一一对应：

| 优先级 | 用例数 | 定位 |
|--------|--------|------|
| P0 | 67 | 核心功能，必须通过 |
| P1 | 72 | 重要功能 |
| P2 | 35 | 边缘场景 |
| **合计** | **172** | 覆盖 23 个模块 |

---

# Part 4: 契约测试框架

## 4.1 要解决的问题

`tests/contract/README.md` 开篇给出了一个真实场景：

> Developer fixes DingTalk file upload by modifying `BaseChannel.send_media()`:
> - DingTalk tests pass (tested locally)
> - Feishu, Discord, Telegram break in production!

**修一个子类，崩一片兄弟**——这是所有多态家族的通病。契约测试的答案是：把"基类接口契约"显式化为可执行的测试，任何子类都必须通过同一套契约验证。

## 4.2 BaseContractTest 机制

框架核心只有一个抽象类（`tests/contract/__init__.py`）：

```python
class BaseContractTest(ABC):
    @abstractmethod
    def create_instance(self) -> Any:
        """Create and return an instance of the class under test."""

    @pytest.fixture
    def instance(self) -> Any:
        return self.create_instance()
```

工作原理：

1. 定义一个契约类（如 `ChannelContractTest`），在其中写若干 `test_*` 方法，断言契约要求（方法存在、属性存在、签名兼容、行为正确）。
2. 每个具体实现写一个测试子类，只需实现 `create_instance()` 返回自己的实例。
3. pytest 的继承机制让**契约测试方法自动在每个子类上重跑**——新增一个渠道，继承契约类即自动获得全部契约检查。

```
BaseContractTest (抽象基)
    ↓
ChannelContractTest (定义渠道契约)
    ↓
TestDingTalkChannel   TestFeishuChannel   TestConsoleChannel...
    (各自实现 create_instance())
```

## 4.3 Channel 契约的四类验证

| 类别 | 验证内容 | 能抓到的典型问题 |
|------|---------|-----------------|
| **Abstract Methods** | `start()` / `stop()` / `send()` 已实现 | 基类新增抽象方法，某子类没实现 |
| **Attributes** | `channel`、`uses_manager_queue` 等属性存在 | 构造函数遗漏属性初始化 |
| **Signatures** | 参数类型兼容 | `send()` 签名被悄悄改动 |
| **Behavior** | `resolve_session_id` 返回 str | 返回类型退化 |

## 4.4 契约覆盖率检查

Makefile 提供了独立的契约覆盖率检查命令：

```makefile
check-contracts:
	$(PYTHON) scripts/check_channel_contracts.py
```

它检查"是否每个渠道子类都有对应的契约测试"——防止新渠道接入时绕过契约层。这是对契约测试体系的**元测试**。

## 4.5 扩展新渠道的标准流程

契约框架把"接入新渠道"变成了填空题：

```python
# tests/contract/channels/test_slack_contract.py
class TestSlackChannelContract(ChannelContractTest):
    def create_instance(self):
        return SlackChannel(process=mock_process, ...)

    # 可选：Slack 特有契约
    def test_has_webhook_url(self, instance):
        assert hasattr(instance, '_webhook_url')
```

继承 → 实现 `create_instance()` → （可选）补充专属契约。三步完成，全部契约检查自动生效。

---

# Part 5: 运行编排与设计哲学

## 5.1 Makefile 命令矩阵

`Makefile` 是测试体系的统一入口，按层级和场景封装：

| 命令 | 作用 |
|------|------|
| `make test` | 全量测试 |
| `make test-unit` | 仅单元测试 |
| `make test-contract` | 仅契约测试 |
| `make test-integration` | 仅集成测试（约 3 分钟） |
| `make quick` | **快速反馈**：临时工作目录 + `-x`（首个失败即停）+ `-q --tb=line` |
| `make coverage-full` | 全量 + coverage（term-missing + html） |
| `make check-contracts` | 渠道契约覆盖率检查 |
| `make test-channel` / `test-channel-contract` | 渠道专项 |

`quick` 目标值得注意：

```makefile
quick:
	@qp_test_workdir=$$(mktemp -d); \
	trap 'rm -rf "$$qp_test_workdir"' EXIT; \
	QWENPAW_WORKING_DIR="$$qp_test_workdir" \
	$(PYTEST) tests/unit/ -x -q --tb=line
```

用 `mktemp -d` 开临时工作目录、`trap` 保证清理、`-x` 快速失败——这是给开发者"改一行代码 30 秒内得到反馈"的通道。**测试体系的可用性取决于最快那条通道的速度**。

## 5.2 CI 质量门禁链

`tests.yml` 定义了五段式门禁：

```
spam-gate（PR 垃圾过滤）
    ↓
approval-gate（maintainer 批准环境）
    ↓
unit-tests（矩阵：py3.11/3.13 × ubuntu/macos/windows）
contract-tests
integrated-tests
coverage-report（合并覆盖率 artifact）
    ↓
test-summary（汇总判定，任一失败即红）
```

单测矩阵覆盖三个操作系统——因为 QwenPaw 的沙箱在三平台有不同实现（macOS Seatbelt / Linux Bubblewrap / Windows AppContainer），Linux 上还要专门安装 `bubblewrap` 并关闭 AppArmor 的 user namespace 限制：

```yaml
- name: Install Linux isolation dependency
  if: runner.os == 'Linux' && matrix.python-version == '3.11'
  run: |
    sudo apt-get install -y bubblewrap
    sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
```

覆盖率采集也有讲究：只在单一矩阵组合（ubuntu + py3.13，tracer 开销最低）上采集 `.coverage.*` 文件，作为短生命周期 artifact 上传，`coverage-report` job 只做合并渲染、不重跑测试——**覆盖率不增加矩阵时间**。

## 5.3 六条设计哲学总结

1. **优先级锚定用户影响面**：`"If this fails, can users still send a message?"` 一个问题定生死，避免优先级标注沦为拍脑袋。
2. **隔离优先于便利**：清除敏感 Key、重置全局单例、临时工作目录、拒绝污染真实 `~/.qwenpaw`——四层防线保证测试可重复、零副作用。
3. **契约层专治多态回归**：多态家族越大，契约层的杠杆率越高。新增子类"继承即合规"，基类变更"一改全查"。
4. **Mock 的深度分级**：单测层 Mock 接口（MagicMock），集成层 Mock 协议（10 个模拟服务端），E2E 层 Mock 前端 API（Playwright route）——每一层用与该层职责匹配的 Mock 粒度。
5. **基准是运行策略而非测试集合**：p0/p1/p2 标记 × 触发时机（PR/nightly/release）× 手动覆盖表达式，构成可调度的质量门禁矩阵。
6. **最快通道决定开发体验**：`make quick` 的 30 秒反馈与 nightly 的全量回归并存——快慢两条腿，缺一不可。

---

## 附：与 Kilo Code 评测体系的对照视角

| 维度 | QwenPaw | Kilo Code（参照） |
|------|---------|------------------|
| 语言生态 | Python（pytest） | TypeScript（vitest） |
| 金字塔层数 | 4 层（unit/contract/integration/e2e） | 类似分层 |
| 契约测试 | 显式 `BaseContractTest` 框架 | 类型系统承担部分契约职能 |
| 优先级体系 | p0/p1/p2 按用户影响面 | 按套件划分 |
| 协议级 Mock | 10 个 IM 模拟器 | — |
| 跨平台矩阵 | ubuntu/macos/windows × py3.11/3.13 | 多平台 |

QwenPaw 的独特性在于**契约测试框架的显式化**和**IM 协议模拟器的厚度**——这是由它"多渠道 Agent 网关"的产品形态决定的。
