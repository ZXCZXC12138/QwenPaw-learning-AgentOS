# QwenPaw 消息通道（Channels / CLI / ACP）深度解析

> **来源**：基于 QwenPaw 代码仓 `src/qwenpaw/app/channels/`（约 48000 行，20+ 渠道）与 `cli/`、`agents/acp/` 接入面源码分析
> **分析范围**：`BaseChannel` 渠道基类（2367 行）、`UnifiedQueueManager` 三元组队列、`ChannelManager` 生命周期、消息渲染器、统一访问控制、重复身份检测、渠道注册表（18 个内置渠道）
> **本文档包含五部分内容**：
> 1. 渠道层定位：同一个内核，二十张脸
> 2. 队列工程：三元组隔离与按需消费者
> 3. BaseChannel：入站归一、出站渲染、流式钩子
> 4. 安全与秩序：访问控制、去抖、重复身份检测
> 5. 注册与接入面：内置 + 插件渠道、CLI 与 ACP

---

## 目录

### Part 1: 渠道层定位
- 20+ 渠道清单与代码量分布
- 渠道 = 协议适配器，不是逻辑宿主

### Part 2: 队列工程
- QueueKey 三元组：channel × session × priority
- 按需消费者与空闲队列回收

### Part 3: BaseChannel
- 入站：原生负载识别、合并、去抖
- 出站：渲染器与显示配置
- 流式：三段式钩子

### Part 4: 安全与秩序
- 统一访问控制存储（白名单/黑名单/待审批）
- 群聊 @ 提及与拒绝消息
- 跨渠道重复 Bot 身份检测

### Part 5: 注册与接入面
- 惰性加载注册表与必需渠道
- CLI 与 ACP 作为另外两条腿
- 设计哲学总结

---

# Part 1: 渠道层定位：同一个内核，二十张脸

## 1.1 渠道清单

`app/channels/` 是代码仓最大的单一目录（约 4.8 万行），内置渠道覆盖中外主流 IM 与开放协议：

| 梯队 | 渠道 | 单渠道代码量 |
|------|------|--------------|
| 国内 IM | 钉钉（3723 行）、飞书（2823）、QQ（2318）、微信（1823）、企业微信（1687）、小艺（1731）、元宝（1757）、OneBot（1663） | 千行级 |
| 开放协议 | Matrix（3488）、Telegram（1593）、Discord（1066）、Slack（901）、Mattermost（1106）、MQTT | 千行级 |
| 系统/语音 | iMessage、Console（736）、Voice、SIP（822） | — |

代码量差异本身就是信息：钉钉 3700 行不是框架不公，而是钉钉生态的协议复杂度（Stream 回调 + sessionWebhook + AI 卡片流式 + Open API 四条投递通路）——**渠道适配器的厚度由上游平台决定，框架只保证厚度落在正确的位置**。

## 1.2 渠道是协议适配器，不是逻辑宿主

`base.py` 模块注释一句话定位：

> Base Channel: bound to AgentRequest/AgentResponse, **unified by process**.

渠道只干两件事：把平台原生消息翻译成 `AgentRequest`、把内核的 `Event` 流翻译回平台消息。中间的处理函数签名是统一的：

```python
# process: accepts AgentRequest, streams Event
ProcessHandler = Callable[[Any], AsyncIterator["Event"]]
```

每个渠道拿到的都是同一个 `process`——即 Runtime 的请求处理面。渠道不认识 agent、不认识工具、不认识模型，**它只是翻译官**。

# Part 2: 队列工程：三元组隔离与按需消费者

## 2.1 QueueKey 三元组

`unified_queue_manager.py`（498 行）的模块文档把并发模型说得很透：

> QueueKey = (channel_id, session_id, priority_level)
> 1. 不同会话并发处理
> 2. 同一会话内不同优先级并发处理
> 3. 相同 QueueKey 的消息严格串行
> 4. 按需创建消费者（没有固定工作池）
> 5. 空闲队列自动清理

这个三元组直接编码了消息系统的正确性约束：**同一用户同一优先级的消息必须按序处理**（否则前后轮次乱序），而不同会话之间没有任何理由排队等待。每个 QueueKey 拥有独立的 `asyncio.Queue` + 消费者 `Task`，首条消息到达时按需创建，空闲 600 秒（`idle_timeout`）后由清理循环（60 秒一次）回收——**工作池不预分配，队列不为想象中的流量买单**。

## 2.2 ChannelManager：框架侧的所有权

`manager.py` 文件头注释宣示所有权：

> ChannelManager is the framework owner of BaseChannel and must call `_is_native_payload` and `_consume_one_request` as part of the contract.

队列活在 ChannelManager 里，渠道只定义"如何消费一条"（`consume_one`）。批量到达时 `_process_batch` 分四路处理：原生负载多条 → `merge_native_items` 合并；请求多条 → `merge_requests` 合并（合并失败退回逐条）；单条直接消费。**合并是渠道的选择性实现，串行是框架的强制保证**。管理器还跟踪入队任务与启动任务两组 `Task` 集合，为优雅关闭服务；每渠道一把重启锁防止并发重启。

# Part 3: BaseChannel：入站归一、出站渲染、流式钩子

## 3.1 入站：原生负载、合并、去抖

基类的方法面构成入站流水线：

```
_is_native_payload          区分平台原生负载与已归一的请求
merge_native_items          原生消息合并（钉钉连发多句合一）
_debounce_payload           去抖：短时间连发消息合并处理
_apply_no_text_debounce     纯媒体消息（无文本）的专项去抖
build_agent_request_from_native / from_user_content   两路构建请求
resolve_session_id          渠道特异的会话身份解析
```

去抖的存在是 IM 场景的现实主义：用户习惯三秒发四句话，每句触发一次模型调用既贵又割裂上下文。

## 3.2 出站：渲染器与显示配置

`renderer.py` 模块定位：**"Pluggable message renderer: Message -> sendable parts (runtime Content). Style/capabilities control markdown, emoji, code fence."** 同一段模型输出，在 Telegram 可以发 Markdown，在短信式渠道得剥成纯文本——渲染器按渠道能力裁剪。`ChannelDisplayConfig` 是渠道级的展示旋钮：

```
show_tool_details / show_thinking / show_tool_calls / show_tool_results
tool_call_max_length: 200     tool_result_max_length: 500   （0 = 不截断）
```

工具调用的中间过程默认截断到几百字符——**开发者要看的过程，普通用户渠道不硬塞**。一个纪律细节：`_sanitize_surrogate_text` 专门清洗代理对之外的孤立代理字符——IM SDK 的 JSON 序列化遇到这类字符会直接崩，渠道层在源头清掉。

## 3.3 流式：三段式钩子

`streaming_enabled` 开关决定渠道是否参与实时流式。开启后，流式增量事件走三段式钩子：

```
on_streaming_start   →  on_streaming_delta  →  on_streaming_end
```

且是**在已完成消息路径之外的附加路径**（"in addition to"）——不支持流式的渠道照常收到完整消息，支持的渠道额外获得增量事件。钉钉渠道是流式的重用户：AI 卡片流式更新正是靠这组钩子驱动。钉钉的模块注释还点出其异步投递架构："The handler ACKs the DingTalk Stream callback immediately. All actual replies are delivered asynchronously via sessionWebhook, AI Card streaming updates, or Open API."——**立即确认 + 异步投递**是 webhook 型平台的标准姿态。

# Part 4: 安全与秩序

## 4.1 统一访问控制

`access_control.py` 是跨渠道的统一门禁存储，每渠道三类名单：

```
白名单（whitelist）      直接放行
黑名单（blacklist）      直接拒绝
待审批（PendingEntry）   发过消息但尚未归入任何名单的用户
```

`PendingEntry` 记录首条消息、用户名、时间戳与备注——**陌生人第一次发消息不是被静默丢弃，而是进入可审批的待办**。存储持久化到工作目录的 `access_control.json`，原子写入（`write_json_atomic`）+ 线程锁。基类侧的 `_access_control_gate` 是入站第一道闸，旧的 `dm_policy` / `group_policy` / `allow_from` 字段保留但注释明示"Legacy fields — stored for backward compat but not used for filtering"——**迁移到新门禁，老配置不炸但不再生效，且写明**。

## 4.2 群聊纪律与重复身份检测

`require_mention`：群聊中必须 @ 机器人才响应——避免机器人在群里抢话。拒绝走 `_acl_msg` 的可模板化拒绝消息。

`conflict.py` 解决一个隐蔽事故：**同一个 Bot 身份配置到多个渠道/实例，消息会被重复处理**。它按渠道声明身份字段：

```python
_CHANNEL_IDENTITY_FIELDS = {
    "discord": ("bot_token",),    "telegram": ("bot_token",),
    "dingtalk": ("client_id",),   "feishu": ("app_id",),
    "matrix": ("homeserver", "user_id"),  ...
}
```

归一化时刻意不暴露原值（"Normalize an identity value without exposing it outside this module"），homeserver/url 去尾斜杠后比对——**令牌类身份只以规范化指纹参与比较**。

# Part 5: 注册与接入面

## 5.1 渠道注册表：内置 + 插件

`registry.py` 维护 18 个内置渠道的惰性加载规格：

```python
_BUILTIN_SPECS = { "dingtalk": (".dingtalk", "DingTalkChannel"), ... }
_REQUIRED_CHANNEL_KEYS = frozenset({"console"})
```

两点纪律：**惰性加载**——未启用的渠道模块不导入（Matrix、SIP 这些重模块不为没配置它们的用户付出启动成本）；**必需渠道不容失败**——`console` 加载失败直接抛异常而非跳过，因为它是兜底交互面。插件渠道通过 `PluginRegistry.register_channel` 进入同一注册表，与内置渠道平权。

每个渠道还可实现 `doctor_connectivity_notes`，为 `copaw doctor --deep` 提供专属连通性检查——诊断体系（见观测诊断篇）的渠道探针就挂在这里。

## 5.2 CLI 与 ACP：另外两条腿

渠道之外还有两条接入路径：**CLI**（`cli/` 目录，命令行交互，与 console 渠道共享渲染与命令面）和 **ACP**（`agents/acp/`，Agent Client Protocol——见 Driver 篇）：前者服务开发者终端，后者让外部编辑器/Agent 以标准协议驱动 QwenPaw。三条腿共享同一个 `ProcessHandler` 面——**接入方式可以无限多，处理内核只有一个**。

## 5.3 设计哲学总结

1. **渠道只是翻译官**：入站翻成 `AgentRequest`，出站翻回平台消息；处理逻辑全部集中在唯一的 `ProcessHandler` 面。
2. **并发模型编码在队列键里**：(渠道 × 会话 × 优先级) 三元组——该串行的严格串行，该并发的绝不排队；消费者按需创建、空闲回收。
3. **适配厚度落在正确位置**：钉钉 3700 行的复杂度是平台给的，框架用统一的钩子面（流式三段、合并、去抖）承接，而不是让每个渠道自造管线。
4. **展示是渠道的权利也是义务**：渲染器按能力裁剪，工具过程默认截断，孤立代理字符在序列化前清洗。
5. **陌生人进待办，不进黑洞**：访问控制三名单 + 待审批记录；老配置字段保留但不再生效且注释写明。
6. **重复身份是可检测的事故**：每渠道声明身份字段，规范化指纹比对，令牌原值不出模块。
7. **加载纪律分明**：未配置渠道惰性加载，兜底渠道（console）加载失败即崩溃——启动期暴露问题好过运行期失去最后一块交互面。

---

## 附：关键文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `base.py` | 2367 | BaseChannel：入站归一/出站渲染/流式钩子 |
| `manager.py` | 852 | ChannelManager：队列所有权与生命周期 |
| `unified_queue_manager.py` | 498 | 三元组队列与按需消费者 |
| `renderer.py` | — | 可插拔消息渲染器与显示配置 |
| `access_control.py` | — | 统一白/黑名单与待审批存储 |
| `conflict.py` | — | 跨渠道重复 Bot 身份检测 |
| `registry.py` | — | 渠道注册表（18 内置 + 插件） |
| `dingtalk/channel.py` | 3723 | 最复杂渠道：四条投递通路 |
| `matrix/channel.py` | 3488 | 开放协议渠道代表 |
| `qrcode_auth_handler.py` | 832 | 扫码登录类渠道的认证辅助 |
