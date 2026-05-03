# 03 - Gateway 消息平台网关深度分析

> 学习顺序：第 3 篇 | 前置知识：异步编程、网络协议、适配器模式

---

## 一、模块概述

Hermes Agent Gateway 是一个**统一多平台消息网关**，核心设计目标：

- **平台无关性**：将 20+ 消息平台的消息统一抽象为 `MessageEvent`
- **会话持久化**：跨重启的会话状态管理
- **实时交互**：消息中断、打字指示器、流式输出
- **安全准入**：基于配对码的用户授权体系
- **生产级可靠性**：自动重连、背压处理、优雅关停

核心文件规模：
- `run.py` (9795 行) -- 主循环、消息分发
- `session.py` (1090 行) -- 会话持久化
- `platforms/base.py` (2133 行) -- 适配器基类
- `platforms/telegram.py` (2879 行) -- Telegram 适配器
- `platforms/discord.py` (3165 行) -- Discord 适配器
- `platforms/slack.py` (1677 行) -- Slack 适配器

---

## 二、Gateway 主循环

### 2.1 启动流程

`GatewayRunner.start()` 执行顺序：
1. SSL 证书自动检测、`.env` 加载
2. Hook 发现（扫描 `~/.hermes/hooks/`）
3. 崩溃恢复（从检查点恢复后台进程）
4. 会话悬挂（防止重启后恢复损坏状态）
5. 适配器初始化（遍历配置的平台，创建实例，注入 handler，调用 connect）
6. 后台任务启动（会话过期监视器、平台重连监视器）

### 2.2 消息分发机制

7 步处理管线：
1. 用户授权检查
2. 命令检测（`/new`, `/reset` 等）
3. 运行中 Agent 检测和中断
4. 获取或创建会话
5. 构建上下文
6. 运行 Agent 对话
7. 返回响应

### 2.3 统一消息事件

```python
@dataclass
class MessageEvent:
    text: str
    message_type: MessageType = MessageType.TEXT
    source: SessionSource = None
    raw_message: Any = None
    message_id: Optional[str] = None
    media_urls: List[str] = field(default_factory=list)
    reply_to_message_id: Optional[str] = None
    auto_skill: Optional[str | list[str]] = None
    channel_prompt: Optional[str] = None
```

---

## 三、平台适配器模式

### 3.1 适配器接口

```python
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def connect(self) -> bool: ...
    @abstractmethod
    async def disconnect(self) -> None: ...
    @abstractmethod
    async def send(self, chat_id, content, reply_to, metadata) -> SendResult: ...
    @abstractmethod
    async def get_chat_info(self, chat_id) -> Dict[str, Any]: ...
```

### 3.2 各平台 API 差异统一

| 特性 | Telegram | Discord | Slack |
|------|----------|---------|-------|
| 连接方式 | 长轮询/Webhook | WebSocket | Socket Mode WebSocket |
| 消息长度限制 | 4096 UTF-16 码元 | 2000 字符 | 40000 字符 |
| 格式化 | MarkdownV2 | 标准 Markdown | mrkdwn |
| 输入指示器 | `sendChatAction` 5秒过期 | `typing()` 上下文管理器 | 不支持 |

### 3.3 Telegram 适配器关键特性

1. **MarkdownV2 转义**：18 个特殊字符必须转义
2. **媒体组缓冲**：0.8 秒延迟等待后合并相册
3. **文本分片聚合**：0.6-2.0 秒延迟聚合客户端分片
4. **UTF-16 长度计算**：emoji 等 BMP 外字符占 2 个码元
5. **网络容错**：指数退避重连（5s-60s），最多 10 次

---

## 四、会话管理

### 4.1 双层存储架构

| 数据 | 存储位置 | 写入策略 |
|------|---------|---------|
| Session key → ID 映射 | `sessions.json` | 原子写入 |
| 消息记录 | SQLite + JSONL 双写 | 同时写 DB 和 .jsonl |
| 会话元数据 | SQLite sessions 表 | create/end 时写入 |

### 4.2 Session Key 设计

格式：`agent:main:{platform}:{chat_type}:{chat_id}[:{thread_id}][:{user_id}]`

设计考量：
1. 确定性：相同来源总是生成相同 key
2. 层级性：platform → chat_type → chat_id → thread_id → user_id
3. 可配置隔离度：`group_sessions_per_user` 控制群组内隔离

### 4.3 会话过期策略

```python
@dataclass
class SessionResetPolicy:
    mode: str = "both"        # "daily", "idle", "both", "none"
    at_hour: int = 4
    idle_minutes: int = 1440  # 24 小时
    notify: bool = True
```

过期前触发**记忆冲刷**：用临时 Agent 回顾对话历史，保存重要信息到 memory。

---

## 五、安全机制

### 5.1 DM 配对验证

```python
ALPHABET = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"  # 无歧义字母表
CODE_LENGTH = 8
CODE_TTL_SECONDS = 3600          # 1 小时过期
RATE_LIMIT_SECONDS = 600         # 每用户每 10 分钟 1 次
LOCKOUT_SECONDS = 3600           # 5 次失败后锁定 1 小时
```

### 5.2 用户授权链

```
allow-all flag → pairing store → platform allowlist → global allowlist → default deny
```

### 5.3 SSRF 防护

所有图片/音频下载经过 SSRF 防护，包括重定向链检查。

---

## 六、设计模式

| 模式 | 应用 |
|------|------|
| 适配器模式 | `BasePlatformAdapter` 统一 20+ 平台 |
| 观察者模式 | `HookRegistry` 事件钩子系统 |
| 消息队列模式 | 多层队列（适配器级、文本批次、媒体批次、流式输出、中断信号） |
| 责任链模式 | 消息授权检查链 |
| 策略模式 | 会话重置策略 |

---

## 七、面试八股文

### 7.1 什么是适配器模式？

将一个类的接口转换成客户端期望的另一个接口。本项目中 `BasePlatformAdapter.send()` 统一了 20+ 个平台的消息发送 API。

### 7.2 Webhook vs 长连接的选择

| 模式 | 优点 | 缺点 |
|------|------|------|
| Webhook | 适合高并发 | 需要公网 IP |
| 长轮询/WebSocket | 无需公网暴露 | 需处理重连 |

### 7.3 消息去重策略

基于 TTL 的去重缓存：`Dict[msg_id, timestamp]` + 300 秒 TTL + 最大容量 2000 条。

### 7.4 背压处理

1. 中断机制：新消息中断正在处理的 Agent
2. 忙碌模式：`"interrupt"` 或 `"queue"`
3. 陈旧 Agent 驱逐
4. 流式输出限速

### 7.5 API 限流策略

1. 配对码速率限制（每用户每 10 分钟 1 次）
2. 打字指示器节流（每 2 秒刷新）
3. 发送重试（指数退避 + 随机抖动）
4. 编辑限速（1 秒间隔）

---

## 八、面试官可能问的问题及详细回答

### Q1: 整体架构是什么？

三层：GatewayRunner（核心控制器）→ Platform Adapters（平台适配器）→ AI Agent。消息经过授权检查、命令分派、会话管理后交由 Agent 处理。

### Q2: 如何保证同一会话不会并发处理？

`_active_sessions: Dict[str, asyncio.Event]` 实现会话级互斥。同步设置守卫（grammY sequentialize 模式），避免竞态窗口。

### Q3: Session Key 怎么设计的？

确定性构建、层级性结构、可配置隔离度、人类可读。

### Q4: 流式输出怎么实现的？

Agent（同步线程）→ on_delta → queue.Queue → run()（异步任务）→ adapter.edit_message()。编辑频率限速（1 秒间隔），连续 3 次洪水控制失败后永久降级。

### Q5: 优雅关停怎么做的？

设置 shutdown_event → 取消后台任务 → 断开适配器 → 等待 Agent 完成 → 写入 .clean_shutdown 标记。

### Q6: 平台适配器崩溃了怎么办？

启动时失败：加入重连队列。运行时失败：移除后检查是否所有平台都断了，可配置为触发 Gateway 关闭。

### Q7: 记忆冲刷是什么？

会话过期时用临时 Agent 回顾对话历史，将重要信息保存到 memory 文件。防止用户重复提供之前说过的信息。

### Q8: SSRF 防护怎么做的？

URL 安全检查（私有/内部地址）+ 重定向防护（httpx event hook）+ DNS 解析检查。

### Q9: 会话悬挂机制？

启动时标记 120 秒内活跃的会话为 suspended，下次用户发消息时自动创建新会话。

### Q10: 打字指示器为什么每 2 秒刷新？

Telegram 的 `sendChatAction` 效果只有 5 秒。2 秒间隔确保用户始终看到 "正在输入..." 状态。

### Q11: 消息截断有什么技巧？

代码块感知分割、UTF-16 安全分割、内联代码保护、多段指示。

### Q12: 多 Workspace 支持（Slack）怎么实现？

按 team_id 维护独立的 WebClient 实例，收到消息时根据 channel_id 查找对应的 team_id。

### Q13: VoiceReceiver（Discord）的设计？

RTP 解密 → SSRC-User 映射 → Opus 解码 → 静音检测（1.5 秒）→ 最小时长过滤（0.5 秒）。

### Q14: 代理支持怎么做的？

多层检测：平台特定环境变量 → HTTPS_PROXY → macOS 系统代理。SOCKS5 使用 `aiohttp_socks.ProxyConnector`。

### Q15: 如何设计一个可扩展的命令系统？

基于命令注册表（`COMMAND_REGISTRY`），集中定义所有命令。添加新命令只需在注册表添加条目 + 实现 handler。

---

## 九、学习路径建议

1. `gateway/platforms/base.py` -- 适配器接口和 MessageEvent
2. `gateway/session.py` -- 会话管理和持久化
3. `gateway/platforms/telegram.py` -- 最完整适配器
4. `gateway/run.py` 的 `_handle_message()` -- 消息分发管线
5. `gateway/pairing.py` -- 安全授权
6. `gateway/stream_consumer.py` -- 流式输出桥接
