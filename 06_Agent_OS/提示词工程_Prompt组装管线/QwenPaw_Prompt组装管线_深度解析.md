# QwenPaw Prompt 组装管线 深度解析

> **来源**：基于 QwenPaw 代码仓 `src/qwenpaw/runtime/prompt_manager.py`（146 行）、`runtime/prompt_contributors.py`（401 行）、`agents/prompt.py`（565 行）、`agents/prompt_builder.py` 全量源码分析
> **分析范围**：`PromptContributor` 贡献者模型、`PromptManager` 按优先级装配、内置七贡献者清单、工作区提示文件（AGENTS.md / SOUL.md / PROFILE.md）与条件段落标记、`PromptBuilder` 宿主锚点与插件段落插入、动态上下文注入
> **本文档包含四部分内容**：
> 1. 贡献者模型：系统提示词的分段责任制
> 2. 工作区提示文件：磁盘即人格
> 3. 条件段落与能力感知：heartbeat / memory / multimodal 标记
> 4. 插件段落与动态注入：锚点模型

---

## 目录

### Part 1: 贡献者模型
- PromptContributor / SyncPromptContributor
- PromptManager：优先级排序、故障隔离、分隔符契约
- 内置七贡献者与优先级分布

### Part 2: 工作区提示文件
- 三文件默认集与 UI 可配置顺序
- 路径穿越防护与 frontmatter 剥离
- None 与空列表的语义区分

### Part 3: 条件段落
- `<!-- heartbeat:start -->` / `<!-- memory:start -->` 标记处理
- 多模态提示与活跃模型探测
- Coding Mode / Scroll 上下文的模式级注入

### Part 4: 插件与动态注入
- PromptBuilder：宿主锚点 + 插件段落
- HookContext.inject_context：每请求动态提示
- 设计哲学总结

---

# Part 1: 贡献者模型：系统提示词的分段责任制

## 1.1 单一职责的贡献者

`prompt_manager.py` 的模块注释定义了管线形态：

> Each contributor declares one fragment of the system prompt; `PromptManager` orders them by priority and joins non-empty results with `PROMPT_SEPARATOR`.

贡献者基类只有三个构件：

```python
class PromptContributor:
    name: str          # 管理器内唯一，用于替换/禁用
    priority: int = 100  # 升序：数字小者先出现
    async def contribute(self, ctx) -> str | None   # 返回 None/空 = 本次请求弃权
```

`SyncPromptContributor` 为无异步工作的贡献者提供便捷基类。**弃权是一等公民**——贡献者可以根据当前请求上下文决定不产出任何内容（如 Coding Mode 未启用时 `CodingModeContributor` 返回 None），最终提示词只包含实际生效的段落。

## 1.2 PromptManager 的三条纪律

`build()` 的实现藏着三条纪律：

**① 故障隔离**："A contributor raising is logged and skipped — one broken contributor must not take down the prompt assembly for the whole request." 一个贡献者坏了只记日志跳过，不拖垮整个请求的提示词装配。

**② 同步/异步双兼容**：`await raw if inspect.isawaitable(raw) else raw`——无条件 await 检查，纯字符串与协程返回值通吃；另有 `build_sync` 直通 `contribute_sync`，无事件循环的场景也能装配（内置贡献者全是同步的，注释明示）。

**③ 分隔符是契约**：

> `PROMPT_SEPARATOR` is intentionally a module-level constant so parity tests can pin the exact separator (**any change is a contract break**).

`PROMPT_SEPARATOR = "\n\n"` 被刻意升格为模块级常量——一致性测试会钉死它，改动即违约。系统提示词的拼接细节被当成对外契约对待，因为下游（评测基准、提示词缓存、行为回放）依赖它稳定。

注册面同样严格：重名注册直接报错（"already registered"），注册后按优先级重排。名字是替换/禁用的句柄——运行时可以精确摘除某个段落来源。

## 1.3 内置七贡献者与优先级分布

`build_default_prompt_manager` 预载的贡献者与优先级：

| 贡献者 | priority | 段落内容 |
|--------|----------|----------|
| `AgentIdentityContributor` | 5 | Agent 身份头（"Your agent id is …"） |
| `WorkspacePromptFilesContributor` | 10 | 工作区提示文件（AGENTS.md 等） |
| `MultimodalHintContributor` | 80 | 多模态能力感知提示 |
| `CodingModeContributor` | 85 | Coding Mode 人格块 |
| `ScrollContextContributor` | 86 | scroll 上下文策略的记忆/召回指引 |
| `DriverPolicyHintContributor` | 88 | Driver 策略指引（工具暴露时） |
| `EnvContextContributor` | 90 | 环境上下文（时间/会话/OS） |

优先级分布本身就是叙事：**身份与工作区人格在前（5–30），能力与模式在中段（80–88），环境事实收尾（90）**——提示词的阅读顺序被设计成"我是谁 → 我在哪工作 → 我能干什么 → 此刻的环境事实"。

# Part 2: 工作区提示文件：磁盘即人格

## 2.1 三文件默认集

`DEFAULT_SYSTEM_PROMPT_FILES = ("AGENTS.md", "SOUL.md", "PROFILE.md")`——人格拆成三个关注点：**AGENTS.md 是操作纪律，SOUL.md 是性格内核，PROFILE.md 是身份档案**。`_system_prompt_files` 对配置的三态语义处理得极精细：

> `None` means the profile predates the field, so use the historical defaults. **An empty list is meaningful** and disables workspace prompt files.

配置字段缺失（旧档案）→ 用历史默认；显式空列表 → 有意识地关闭工作区提示文件。**"没说过"和"说了不要"是两种语义**，不许混为一谈。

## 2.2 读取的三道工序

`_read_prompt_file` 对每个文件做三道处理：

1. **路径穿越防护**：`path.is_relative_to(workspace_root)` 校验——提示文件名可以是配置项，但解析结果必须落在工作区内，越界即跳过并警告
2. **frontmatter 剥离**：以 `---` 开头的 YAML 头被切除，只进正文
3. **编码容错**：走 `decode_text_bytes_with_encoding_fallback` + 文件快照缓存——用户手写文件编码五花八门，解码不许崩

文件读取走 `get_file_snapshot_cache()` 快照缓存——提示文件高频读取、低频变更，缓存是对的。

# Part 3: 条件段落与能力感知

## 3.1 Markdown 里的条件编译

AGENTS.md 支持两组注释标记实现条件段落：

```markdown
<!-- heartbeat:start -->
心跳期间的行为纪律……
<!-- heartbeat:end -->
```

`_process_heartbeat_section` 按 `heartbeat_enabled` 开关：启用时**只剥标记保留内容**，禁用时**整段删除**。`<!-- memory:start -->` 段则是替换语义：标记段落整段移除，换上 `memory_manager.get_memory_prompt()` 生成的真实记忆指引——**模板里的占位段与运行时的真实段落各就各位**。这让工作区提示文件既是用户可编辑的 Markdown，又支持系统托管的条件内容。

## 3.2 能力感知：提示词跟随模型事实

`MultimodalHintContributor` 不硬编码"你能看图"，而是调 `agents/prompt.py` 的 `build_multimodal_hint()`——该模块有完整的探测链：`_get_active_model_info` → `get_active_model_supports_multimodal` → `get_model_supports_image` → `format_multimodal_hint(model_info, model_name)`。**提示词里的能力声明来自当前活跃模型的真实能力面**——模型换了，提示词跟着换，不会出现"告诉模型它能看图但实际不能"的灾难。`build_driver_policy_recheck_hint` 同理：Driver 工具暴露时才注入策略指引段落。

`CodingModeContributor` 展示了模式级注入的工程细节：模板 `_CODING_SYSTEM_PROMPT_TEMPLATE` 需要 `project_dir`，解析顺序是"请求配置优先 → 重新加载磁盘配置"——因为 API 侧切换可能绕过请求体，注释写明 "reload disk config for API switches"。`ScrollContextContributor` 则按 `running.light_context_config.strategy == "scroll"` 决定是否注入记忆/召回指引，且按语言渲染（`build_scroll_system_prompt(language)`）。

# Part 4: 插件段落与动态注入

## 4.1 PromptBuilder：宿主锚点模型

`agents/prompt_builder.py` 是插件生态接入提示词的通道：

```python
HOST_ANCHORS = ("workspace", "multimodal", "env_context")
```

宿主先按锚点顺序发射自己的段落，**插件段落插入到其声明的锚点之后**——插件不决定自己绝对排第几，只声明"跟在工作区段后面"这类相对位置。宿主结构稳定，插件插入点可控，两者互不侵入。`PromptSection`（name + content）是插件注册面 `PluginRegistry.register_prompt_section` 的数据单元——见 Plugin 篇。

## 4.2 每请求动态注入

除系统提示词外，还有两条动态通道（见 Runtime 篇）：

- **`HookContext.inject_context(content, priority, source)`**：钩子在阶段执行中收集动态上下文，`_apply_context_injections` 按优先级排序后合并为一条系统消息插到输入头部——这是**轮次级**的提示注入，不进系统提示词
- **技能正文**：激活的技能以操作手册形式进上下文（见 Skill 篇）

三层各管一时：**系统提示词 = 每请求装配的稳定人格；inject_context = 本轮的临时提醒；技能 = 被激活的操作手册**。

## 4.3 设计哲学总结

1. **提示词是编译出来的，不是写死的**：贡献者按优先级产出片段，弃权、故障、空结果都被优雅处理——最终文本是本次请求上下文的一次求值。
2. **分段责任制**：一个贡献者只管一个关注点；名字是禁用/替换的句柄，优先级是阅读顺序的声明。
3. **拼接细节即契约**：分隔符升格为模块常量，"any change is a contract break"——下游一致性测试钉死它。
4. **磁盘即人格，但读取有门禁**：三文件分工承载纪律/性格/身份；路径穿越防护、frontmatter 剥离、编码容错一道不缺；`None` 与空列表语义严格区分。
5. **能力声明跟随事实**：多模态提示来自活跃模型的真实探测，Driver 指引只在工具暴露时注入——提示词不许替模型吹牛。
6. **Markdown 里的条件编译**：`<!-- heartbeat -->` / `<!-- memory -->` 标记让工作区文件既是用户可编辑文档，又支持系统托管段落。
7. **插件靠锚点插队**：相对位置声明而非绝对排序，宿主结构对插件封闭又友好。

---

## 附：关键文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `runtime/prompt_contributors.py` | 401 | 内置七贡献者与默认管理器工厂 |
| `runtime/prompt_manager.py` | 146 | 贡献者模型与优先级装配 |
| `agents/prompt.py` | 565 | 工作区提示构建、多模态探测、bootstrap 指引 |
| `agents/prompt_builder.py` | — | 宿主锚点 + 插件段落组装 |
| `modes/mission/contributor.py` | — | Mission 模式提示贡献者 |
| `modes/goal/contributor.py` | — | Goal 模式提示贡献者 |
