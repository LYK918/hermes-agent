# 03 - Gateway 消息平台网关深度分析

> 学习顺序：第 3 篇 | 前置知识：异步编程、网络协议、适配器模式

---

## 一、模块概述

Hermes Agent Gateway 是一个**统一多平台消息网关**，核心设计目标：
- **平台无关性**：将 19+ 消息平台的消息统一抽象为 `MessageEvent`
- **会话持久化**：跨重启的会话状态管理
- **实时交互**：消息中断、打字指示器、流式输出
- **安全准入**：基于配对码的用户授权体系
- **生产级可靠性**：自动重连、并发控制与优雅降级、优雅关停

核心文件规模：
- `run.py` (9795 行) -- 主循环、消息分发
- `session.py` (1090 行) -- 会话持久化
- `platforms/base.py` (2133 行) -- 适配器基类
- `platforms/telegram.py` (2879 行) -- Telegram 适配器
- `platforms/discord.py` (3165 行) -- Discord 适配器
- `platforms/slack.py` (1677 行) -- Slack 适配器

### 核心架构分层
```
┌─────────────────────────────────────────────────┐
│                 GatewayRunner                   │
│  (主循环、消息分发、生命周期管理、后台任务)      │
├─────────────────────────────────────────────────┤
│              Platform Adapters                  │
│  (Telegram/Discord/Slack/... 平台消息适配)      │
├─────────────────────────────────────────────────┤
│                  Session Store                  │
│  (会话持久化、过期策略、记忆冲刷)                │
├─────────────────────────────────────────────────┤
│              Security & Delivery                │
│  (用户授权、SSRF防护、消息路由、通知分发)        │
└─────────────────────────────────────────────────┘
```

### 消息处理数据流

```
┌──────────┐    ┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│ Platform │───▶│ Message      │───▶│ TextBatch    │───▶│ Authorization│
│ Receive  │    │ Deduplicator │    │ Aggregator   │    │ Check        │
└──────────┘    └─────────────┘    └──────────────┘    └──────┬──────┘
                                                               │
┌──────────┐    ┌─────────────┐    ┌──────────────┐    ┌──────▼──────┐
│ Response │◀───│ _send_with  │◀───│ GatewayStream│◀───│ Command      │
│ Delivery │    │ _retry()    │    │ Consumer     │    │ Detection    │
└──────────┘    └─────────────┘    └──────────────┘    └──────┬──────┘
                                                               │
                                              ┌────────────────▼──────┐
                                              │ Session Lookup / Agent│
                                              │ Run (thread pool)     │
                                              └───────────────────────┘
```

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

#### 启动核心代码实现
```python
async def start(self) -> bool:
    # SSL证书自动检测 - 必须在HTTP库导入前执行
    _ensure_ssl_certs()
    
    # 加载环境变量和配置
    load_hermes_dotenv(hermes_home=_hermes_home, project_env=Path(__file__).resolve().parents[1] / '.env')
    
    # 加载配置文件并注入环境变量
    _config_path = _hermes_home / 'config.yaml'
    if _config_path.exists():
        with open(_config_path, encoding="utf-8") as _f:
            _cfg = _yaml.safe_load(_f) or {}
        _cfg = _expand_env_vars(_cfg)
        # 注入基础配置到环境变量
        for _key, _val in _cfg.items():
            if isinstance(_val, (str, int, float, bool)) and _key not in os.environ:
                os.environ[_key] = str(_val)
    
    # 初始化会话存储
    self.session_store = SessionStore(self.config)
    
    # 初始化各平台适配器
    for platform, platform_config in self.config.platforms.items():
        adapter = self._create_adapter(platform, platform_config)
        if adapter:
            adapter.set_message_handler(self._handle_message)
            adapter.set_fatal_error_handler(self._handle_adapter_fatal_error)
            adapter.set_session_store(self.session_store)
            success = await adapter.connect()
            if success:
                self.adapters[platform] = adapter
    
    # 启动后台任务
    asyncio.create_task(self._session_expiry_watcher())  # 会话过期监视器
    asyncio.create_task(self._platform_reconnect_watcher())  # 平台重连监视器
    
    logger.info("Gateway started successfully")
    return True
```

### 2.2 消息分发机制

7 步处理管线：
1. 用户授权检查
2. 命令检测（`/new`, `/reset` 等）
3. 运行中 Agent 检测和中断
4. 获取或创建会话
5. 构建上下文
6. 运行 Agent 对话
7. 返回响应

#### 消息处理核心代码
```python
async def _handle_message(self, event: MessageEvent) -> None:
    # 1. 用户授权检查
    if not await self._authorize_user(event.source):
        await self._send_authorization_required(event)
        return
    
    # 2. 命令检测和处理
    command_result = await self._try_handle_command(event)
    if command_result.handled:
        return
    
    # 3. 运行中Agent检测和中断
    session_key = build_session_key(event.source, self.config)
    if session_key in self._running_agents:
        await self._handle_active_session_busy_message(event, session_key)
        return
    
    # 4. 获取或创建会话
    session = self.session_store.get_or_create(event.source)
    
    # 5. 构建上下文
    context = build_session_context(event.source, self.config, session)
    
    # 6. 运行Agent对话
    self._running_agents[session_key] = _AGENT_PENDING_SENTINEL
    try:
        agent = self._get_or_create_agent(session, context)
        self._running_agents[session_key] = agent
        
        # 处理消息并获取响应
        response = await agent.run(event.text, context=context)
        
        # 7. 返回响应
        adapter = self.adapters.get(event.source.platform)
        if adapter:
            await adapter.send(
                chat_id=event.source.chat_id,
                content=response,
                reply_to=event.message_id,
                metadata={"thread_id": event.source.thread_id}
            )
    finally:
        self._running_agents.pop(session_key, None)
```

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

#### 消息事件使用说明
- `text`: 消息文本内容，已处理特殊字符转义
- `source`: 消息来源信息，包含平台、聊天ID、用户ID等完整元数据
- `media_urls`: 消息中的媒体文件URL，已通过SSRF安全检查
- `reply_to_message_id`: 回复的消息ID，用于上下文关联

### 2.4 后台任务系统

Gateway 包含两个核心后台任务：

#### 会话过期监视器
```python
async def _session_expiry_watcher(self, interval: int = 300):
    """后台任务每5分钟检查一次过期会话，自动冲刷记忆"""
    await asyncio.sleep(60)  # 初始延迟，等待网关完全启动
    while self._running:
        self.session_store._ensure_loaded()
        expired_entries = []
        for key, entry in list(self.session_store._entries.items()):
            if not entry.memory_flushed and self.session_store._is_session_expired(entry):
                expired_entries.append((key, entry))
        
        if expired_entries:
            logger.info(f"Session expiry: {len(expired_entries)} sessions to flush")
            for key, entry in expired_entries:
                await self._async_flush_memories(entry.session_id, key)
                entry.memory_flushed = True
                self.session_store._save()
        
        # 分块睡眠，支持快速停止
        for _ in range(interval):
            if not self._running:
                break
            await asyncio.sleep(1)
```

#### 平台重连监视器
```python
async def _platform_reconnect_watcher(self) -> None:
    """后台任务自动重试连接失败的平台，使用指数退避策略"""
    _MAX_ATTEMPTS = 20
    _BACKOFF_CAP = 300  # 最长5分钟重试间隔
    
    await asyncio.sleep(10)  # 初始延迟，等待启动完成
    while self._running:
        if not self._failed_platforms:
            await asyncio.sleep(30)
            continue
            
        now = time.monotonic()
        for platform in list(self._failed_platforms.keys()):
            info = self._failed_platforms[platform]
            if now < info["next_retry"]:
                continue
                
            if info["attempts"] >= _MAX_ATTEMPTS:
                logger.warning(f"Giving up reconnecting {platform.value} after {_MAX_ATTEMPTS} attempts")
                del self._failed_platforms[platform]
                continue
                
            attempt = info["attempts"] + 1
            logger.info(f"Reconnecting {platform.value} (attempt {attempt}/{_MAX_ATTEMPTS})...")
            
            try:
                adapter = self._create_adapter(platform, info["config"])
                if adapter:
                    success = await adapter.connect()
                    if success:
                        self.adapters[platform] = adapter
                        del self._failed_platforms[platform]
                        logger.info(f"✓ {platform.value} reconnected successfully")
                    else:
                        backoff = min(30 * (2 ** (attempt - 1)), _BACKOFF_CAP)
                        info["attempts"] = attempt
                        info["next_retry"] = time.monotonic() + backoff
                        logger.info(f"Reconnect {platform.value} failed, next retry in {backoff}s")
            except Exception as e:
                backoff = min(30 * (2 ** (attempt - 1)), _BACKOFF_CAP)
                info["attempts"] = attempt
                info["next_retry"] = time.monotonic() + backoff
                logger.warning(f"Reconnect {platform.value} error: {e}, next retry in {backoff}s")
        
        await asyncio.sleep(10)
```

---

## 三、平台适配器模式

### 3.1 适配器接口

```python
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def connect(self) -> bool:
        """连接到平台，返回是否成功"""
        ...
    
    @abstractmethod
    async def disconnect(self) -> None:
        """断开平台连接，清理资源"""
        ...
    
    @abstractmethod
    async def send(self, chat_id, content, reply_to, metadata) -> SendResult:
        """发送消息到指定聊天"""
        ...
    
    @abstractmethod
    async def get_chat_info(self, chat_id) -> Dict[str, Any]:
        """获取聊天信息"""
        ...
```

### 3.2 适配器工厂模式

`_create_adapter()` 使用条件分支模式，对每个平台执行三步：
1. **依赖检查**：调用 `check_<platform>_requirements()` 验证 SDK/环境变量是否就绪
2. **配置注入**：`config.extra.setdefault()` 向适配器注入 `group_sessions_per_user` 和 `thread_sessions_per_user` 全局配置
3. **实例创建**：返回适配器实例，失败则返回 `None`

#### 工厂核心代码
```python
def _create_adapter(self, platform: Platform, config: Any) -> Optional[BasePlatformAdapter]:
    """Create the appropriate adapter for a platform."""
    # 注入全局配置到所有适配器
    if hasattr(config, "extra") and isinstance(config.extra, dict):
        config.extra.setdefault(
            "group_sessions_per_user",
            self.config.group_sessions_per_user,
        )
        config.extra.setdefault(
            "thread_sessions_per_user",
            getattr(self.config, "thread_sessions_per_user", False),
        )

    if platform == Platform.TELEGRAM:
        from gateway.platforms.telegram import TelegramAdapter, check_telegram_requirements
        if not check_telegram_requirements():
            logger.warning("Telegram: python-telegram-bot not installed")
            return None
        return TelegramAdapter(config)
    
    elif platform == Platform.DISCORD:
        from gateway.platforms.discord import DiscordAdapter, check_discord_requirements
        if not check_discord_requirements():
            logger.warning("Discord: discord.py not installed")
            return None
        return DiscordAdapter(config)
    # ... 19 个平台分支 ...
    
    elif platform == Platform.WEBHOOK:
        from gateway.platforms.webhook import WebhookAdapter, check_webhook_requirements
        if not check_webhook_requirements():
            logger.warning("Webhook: aiohttp not installed")
            return None
        adapter = WebhookAdapter(config)
        adapter.gateway_runner = self  # 跨平台投递引用
        return adapter
    # ...
```

### 3.3 完整平台列表

| 平台 | 适配器文件 | 代码行数 | 连接方式 | 简介 |
|------|-----------|---------|---------|------|
| Telegram | telegram.py | 2879 | 长轮询/Webhook | 功能最完整，支持 media_group 合并、MarkdownV2 转义 |
| Discord | discord.py | 3165 | WebSocket (discord.py) | 支持语音接收、Slash Command、通道目录 |
| WhatsApp | whatsapp.py | 989 | Baileys bridge | 通过 Node.js bridge 连接 WhatsApp Web |
| Slack | slack.py | 1677 | Socket Mode WebSocket | 多 workspace 支持、Block Kit 交互 |
| Feishu | feishu.py | 3986 | 长连接 WebSocket | 持久化去重、富文本卡片、图片缓存 |
| WeCom | wecom.py | 1430 | Webhook + 主动推送 | 企业微信机器人消息 |
| WeCom Callback | wecom_callback.py | 401 | HTTP 回调 + 加解密 | 企业微信回调模式，需 wecom_crypto.py 加解密 |
| WeChat | weixin.py | 1829 | HTTP API 推送 | 微信公众号消息处理 |
| DingTalk | dingtalk.py | 333 | Stream 模式 | 钉钉 Stream 模式长连接 |
| Signal | signal.py | 825 | HTTP REST (signal-cli) | 通过 signal-cli REST API 桥接 |
| Matrix | matrix.py | 2023 | 客户端-服务器 API (mautrix) | 联邦协议，支持 E2E 加密 |
| Mattermost | mattermost.py | 740 | WebSocket / REST | 自托管团队协作平台 |
| QQBot | qqbot.py | 1960 | WebSocket (QQ 官方) | QQ 官方机器人平台 |
| BlueBubbles | bluebubbles.py | 918 | HTTP REST | iMessage 桥接（macOS BlueBubbles 服务） |
| HomeAssistant | homeassistant.py | 449 | HTTP REST | Home Assistant 对话代理集成 |
| Email | email.py | 625 | IMAP + SMTP | 邮件收发，支持附件 |
| SMS | sms.py | 373 | Twilio API | 短信收发，Markdown 自动剥离 |
| API Server | api_server.py | 2436 | HTTP REST | 本地 HTTP API 服务，支持 SSE 流式输出 |
| Webhook | webhook.py | 672 | HTTP Webhook | 通用 Webhook + 跨平台消息投递 |

> **添加新平台指南**：完整的 16 项检查清单见 `gateway/platforms/ADDING_A_PLATFORM.md`，覆盖了适配器核心实现、Platform 枚举、适配器工厂、授权映射、SessionSource、System Prompt 提示词、工具集、定时投递、send_message 工具、Cronjob 工具、通道目录、状态显示、安装向导、手机号脱敏、文档、测试。每项缺漏都可能导致功能异常或行为不一致。

### 3.4 各平台 API 差异统一

| 特性 | Telegram | Discord | Slack |
|------|----------|---------|-------|
| 连接方式 | 长轮询/Webhook | WebSocket | Socket Mode WebSocket |
| 消息长度限制 | 4096 UTF-16 码元 | 2000 字符 | 40000 字符 |
| 格式化 | MarkdownV2 | 标准 Markdown | mrkdwn |
| 输入指示器 | `sendChatAction` 5秒过期 | `typing()` 上下文管理器 | 不支持 |

### 3.5 Telegram 适配器关键特性

1. **MarkdownV2 转义**：18 个特殊字符必须转义
2. **媒体组缓冲**：0.8 秒延迟等待后合并相册
3. **文本分片聚合**：0.6-2.0 秒延迟聚合客户端分片
4. **UTF-16 长度计算**：emoji 等 BMP 外字符占 2 个码元
5. **网络容错**：指数退避重连（5s-60s），最多 10 次

#### UTF-16 长度计算实现
```python
def utf16_len(s: str) -> int:
    """计算字符串的UTF-16码元长度，Telegram消息长度限制使用此单位"""
    return len(s.encode("utf-16-le")) // 2

def _prefix_within_utf16_limit(s: str, limit: int) -> str:
    """返回不超过UTF-16长度限制的最长前缀，确保不截断多字节字符"""
    if utf16_len(s) <= limit:
        return s
    # 二分查找最长安全前缀
    lo, hi = 0, len(s)
    while lo < hi:
        mid = (lo + hi + 1) // 2
        if utf16_len(s[:mid]) <= limit:
            lo = mid
        else:
            hi = mid - 1
    return s[:lo]
```

---

## 三-A、流式消息桥接 (GatewayStreamConsumer)

`gateway/stream_consumer.py` 是 Agent 同步工作线程与异步事件循环之间的桥接层。

**核心架构**：

```
Agent worker thread (sync)                Async event loop
   │                                          │
   │ on_delta("hello")  ──▶ queue.Queue ──▶   run()  ──▶ adapter.edit_message()
   │ on_delta(" world") ──▶ queue.Queue ──▶   drain ──▶ adapter.edit_message()
   │ on_delta(None)     ──▶ _NEW_SEGMENT──▶   tool boundary ──▶ fresh message
   │ finish()           ──▶ _DONE      ──▶   final edit (strip cursor)
```

**关键特性**：

1. **线程安全队列**：使用 `queue.Queue`（线程安全）将 Agent worker 线程的 `on_delta()` 回调转化为 `run()` 协程中的异步消费。

2. **Think-block 过滤**：使用状态机过滤 6 种推理标签变体（`<REASONING_SCRATCHPAD>`, `<think>`, `<reasoning>`, `<THINKING>`, `<thinking>`, `<thought>`），同时处理块边界处的部分标签缓冲。

3. **自适应洪水控制退避**：
   - 编辑失败且错误为 flood/rate 时，编辑间隔翻倍（最大 10s）
   - 连续 3 次洪水失败后（`_MAX_FLOOD_STRIKES = 3`），永久禁用渐进式编辑
   - 成功编辑后重置 strike 计数器

4. **Tool-boundary 段分隔**：`on_segment_break()` 在 tool 调用边界触发，结束当前消息并开始新消息，使后续文本出现在 tool 进度消息之下。特殊处理 `__no_edit__` 哨兵防止 GitHub Comment 等平台产生 155 条碎片消息。

5. **光标显示**：默认 `▉` 光标，编辑间隔默认 1 秒，缓冲区阈值 40 字符。仅在消息有足够可见内容（≥4 字符）时创建新消息。

6. **多阶段回退**：
   ```
   渐进式编辑 → strip cursor → fallback final send
   ```
   - 正常：`send_or_edit()` 渐进更新
   - 洪水：`_try_strip_cursor()` 移除光标 → `_send_fallback_final()` 发送缺失的尾部文本（分块，每块一次重试）
   - `already_sent` 标志防止 Gateway 基路径重复发送完整响应

7. **评论消息**：`on_commentary()` 发送中间助手评论（如 "I'll inspect the repo first"），不标记 `already_sent`，允许后续最终响应正常发送。

8. **超时处理**：被 `CancelledError` 取消时执行最佳努力最终编辑，如果已发送内容则标记 `_final_response_sent` 防止重复。

---

## 三-B、消息批量聚合 (TextBatchAggregator)

`gateway/platforms/helpers.py` 中的 `TextBatchAggregator` 被 Telegram、Discord、Matrix、WeCom、Feishu 等多平台适配器使用，聚合快速连续的文本事件。

```python
class TextBatchAggregator:
    """Aggregates rapid-fire text events into single messages."""
    
    def __init__(self, handler, *, batch_delay=0.6, split_delay=2.0, split_threshold=4000):
```

**工作原理**：
- 每个 `(platform, chat_id, user_id)` 组合维护独立队列
- 新消息追加到 `existing.text`，取消前一个 flush timer
- 使用**自适应延迟**：常规消息使用 `batch_delay`（默认 0.6s），看起来是长消息分片的最后一块时使用 `split_delay`（默认 2.0s）
- 分片判断标准：最后一块文本长度 >= `split_threshold`（默认 4000 字符）

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

#### Session Key 构建实现
```python
def build_session_key(source: SessionSource, config: GatewayConfig) -> str:
    """构建会话唯一标识Key"""
    parts = ["agent", "main", source.platform.value, source.chat_type, source.chat_id]
    
    # 群组会话按用户隔离（如果配置开启）
    if source.chat_type in ["group", "supergroup"] and config.group_sessions_per_user:
        parts.append(source.user_id)
    
    # 线程会话按用户隔离（如果配置开启）
    if source.thread_id:
        parts.append(source.thread_id)
        if config.thread_sessions_per_user:
            parts.append(source.user_id)
    
    return ":".join(parts)
```

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

#### 会话过期检测实现
```python
def _is_session_expired(self, entry: SessionEntry) -> bool:
    """检查会话是否已过期"""
    policy = self.config.session_reset_policy
    
    if policy.mode == "none":
        return False
    
    now = _now()
    expired = False
    
    # 每日重置检查
    if policy.mode in ["daily", "both"]:
        # 上次重置是在今天的重置时间之前吗？
        reset_today = now.replace(hour=policy.at_hour, minute=0, second=0, microsecond=0)
        if entry.last_reset < reset_today and now > reset_today:
            expired = True
    
    # 空闲超时检查
    if policy.mode in ["idle", "both"]:
        idle_time = now - entry.last_activity
        if idle_time > timedelta(minutes=policy.idle_minutes):
            expired = True
    
    return expired
```

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

#### 用户授权检查实现
```python
async def _authorize_user(self, source: SessionSource) -> bool:
    """检查用户是否有权限使用服务"""
    # 1. 全局允许所有用户标志检查
    if self.config.allow_all_users:
        return True
    
    # 2. 配对存储检查
    if self.pairing_store.is_paired(source):
        return True
    
    # 3. 平台级允许列表检查
    platform_allowlist = self.config.platform_allowlists.get(source.platform, [])
    if source.user_id in platform_allowlist or source.chat_id in platform_allowlist:
        return True
    
    # 4. 全局允许列表检查
    if source.user_id in self.config.global_allowlist or source.chat_id in self.config.global_allowlist:
        return True
    
    # 默认拒绝
    return False
```

### 5.3 SSRF 防护

所有图片/音频下载经过 SSRF 防护，包括重定向链检查。

#### SSRF 防护实现
```python
async def _ssrf_redirect_guard(response):
    """重定向防护，检查每个重定向目标的安全性"""
    if response.is_redirect and response.next_request:
        redirect_url = str(response.next_request.url)
        from tools.url_safety import is_safe_url
        
        if not is_safe_url(redirect_url):
            logger.warning(f"Blocked SSRF redirect to unsafe URL: {safe_url_for_log(redirect_url)}")
            raise httpx.TransportError(f"Unsafe redirect target: {redirect_url}")

def is_safe_url(url: str) -> bool:
    """检查URL是否安全，防止SSRF攻击"""
    try:
        parsed = urlparse(url)
        
        # 只允许HTTP/HTTPS协议
        if parsed.scheme not in ("http", "https"):
            return False
        
        # 解析主机名
        hostname = parsed.hostname
        if not hostname:
            return False
        
        # 检查是否是内网地址
        try:
            ip = ipaddress.ip_address(hostname)
            if (ip.is_private 
                or ip.is_loopback 
                or ip.is_link_local 
                or ip.is_multicast 
                or ip.is_reserved):
                return False
        except ValueError:
            # 是域名，解析所有IP地址
            try:
                ips = set(ipaddress.ip_address(a[4][0]) for a in socket.getaddrinfo(hostname, None))
                for ip in ips:
                    if (ip.is_private 
                        or ip.is_loopback 
                        or ip.is_link_local 
                        or ip.is_multicast 
                        or ip.is_reserved):
                        return False
            except socket.gaierror:
                # 域名解析失败，视为不安全
                return False
        
        # 检查端口是否是常用HTTP端口
        if parsed.port and parsed.port not in (80, 443, 8080, 8443):
            return False
        
        return True
    except Exception:
        return False
```

---

## 六、设计模式

| 模式 | 应用 |
|------|------|
| 适配器模式 | `BasePlatformAdapter` 统一 19+ 平台 |
| 观察者模式 | `HookRegistry` 事件钩子系统 |
| 消息队列模式 | 多层队列（适配器级、文本批次、媒体批次、流式输出、中断信号） |
| 责任链模式 | 消息授权检查链 |
| 策略模式 | 会话重置策略 |

---

## 七、面试八股文

### 7.1 什么是适配器模式？

将一个类的接口转换成客户端期望的另一个接口。本项目中 `BasePlatformAdapter.send()` 统一了 19+ 个平台的消息发送 API。

### 7.2 Webhook vs 长连接的选择

| 模式 | 优点 | 缺点 |
|------|------|------|
| Webhook | 适合高并发 | 需要公网 IP |
| 长轮询/WebSocket | 无需公网暴露 | 需处理重连 |

### 7.3 消息去重策略

各平台可重放/重发的消息由 `MessageDeduplicator`（位于 `gateway/platforms/helpers.py`）统一处理：

**通用去重器**

```python
class MessageDeduplicator:
    """TTL-based message deduplication cache."""
    def __init__(self, max_size: int = 2000, ttl_seconds: float = 300):
```

默认参数：最大容量 2000 条，TTL 300 秒（5 分钟）。

**各平台去重配置**

| 平台 | 去重实现 | TTL | max_size | 备注 |
|------|---------|-----|----------|------|
| Discord | `MessageDeduplicator()` | 300s（默认） | 2000（默认） | 需处理 RESUME 重放事件 |
| Slack | 适配器内建 | — | — | 处理消息重发/重连重播 |
| Mattermost | `MessageDeduplicator()` | 300s（默认） | 2000（默认） | WebSocket 事件重放 |
| DingTalk | `MessageDeduplicator(max_size=1000)` | 300s（默认） | 1000 | Stream 模式消息去重 |
| Feishu | 自定义持久化去重 | 24 小时 | 2048（可配置） | 状态持久化到 `feishu_seen_message_ids.json`，跨重启保留 |
| Feishu Card Action | 自定义 token 去重 | 15 分钟 | — | 卡片交互 token 独立去重窗口 |
| Matrix | 有界 deque 去重 | — | deque bounded | 使用 collections.deque 限制内存 |
| QQBot | 自定义去重 | 300s | 1000 | `DEDUP_WINDOW_SECONDS=300, DEDUP_MAX_SIZE=1000` |
| WeCom Callback | 内联分片去重 | 300s | — | `MESSAGE_DEDUP_TTL_SECONDS = 300` |

**Feishu 跨重启持久化去重**：
- 状态文件：`~/.hermes/feishu_seen_message_ids.json`
- TTL：24 小时（`_FEISHU_DEDUP_TTL_SECONDS = 24 * 60 * 60`）
- 加载时自动清理过期条目，按时间戳排序，保留最近 `dedup_cache_size` 条

### 7.4 并发控制与优雅降级

Gateway 使用会话级互斥锁控制并发，而非消费者端流控（源码中无 "back-pressure" 概念）：

1. **会话级互斥**：`_active_sessions: Dict[str, asyncio.Event]` 确保同一会话不被并发处理。新消息可中断当前处理（`"interrupt"` 模式）或排队等待（`"queue"` 模式）。
2. **并发度上限**：`_running_agents` 字典跟踪所有活跃 Agent，配合 `asyncio.Event` 实现同步设置守卫（grammY sequentialize 模式），避免竞态窗口。
3. **陈旧 Agent 驱逐**：启动时标记 120 秒内活跃的会话为 suspended，下次用户发消息时自动创建新会话。
4. **优雅关停排泄**：`_drain_active_agents()` 等待活跃 Agent 完成，超时则 `_interrupt_running_agents()` 强制中断，再给 5 秒清理时间。
5. **关停通知**：`_notify_active_sessions_of_shutdown()` 向所有活跃会话发送关停提示。

### 7.5 API 限流与重试策略

| 机制 | 适用范围 | 策略 |
|------|---------|------|
| 配对码速率限制 | 用户授权 | 每用户每 10 分钟 1 次 |
| 打字指示器节流 | 全平台 | 每 2 秒刷新（Telegram `sendChatAction` 5s 过期） |
| `_send_with_retry()` | 所有消息发送 | 最多 2 次重试，指数退避（base=2s） + 随机抖动；非网络错误回退纯文本发送；全部失败发送 delivery-failure 通知 |
| 流式编辑洪水控制 | 流式输出 | 自适应退避：编辑间隔翻倍（最大 10s），连续 3 次失败永久禁用编辑 |
| 平台重连退避 | 失联平台 | 30*(2^(attempt-1))，上限 300s，最多 20 次尝试 |
| Discord 洪水控制 | Discord 发送 | HTTP 429 响应时自动等待 `retry_after` 秒 |
| Telegram Edits 节流 | Telegram 流式编辑 | 默认 1s 间隔（`edit_interval=1.0`） |
| Feishu API 限流 | Feishu 发送 | 持久化去重（24h TTL）减少重复请求，token bucket 由 lark-oapi SDK 处理 |

---

## 七-B、Gateway 命令速查表

命令来源于 `hermes_cli/commands.py` 中的 `COMMAND_REGISTRY`，Gateway 根据 `GATEWAY_KNOWN_COMMANDS` frozenset 识别可用命令。

| 命令 | 分类 | 绕过活跃会话守卫 | Gateway 专有 | 说明 |
|------|------|:---:|:---:|------|
| `/new` (alias: `/reset`) | Session | Yes | No | 启动新会话（重置会话 ID + 历史） |
| `/stop` | Session | No | No | 终止所有运行中的后台进程 |
| `/restart` | Session | No | Yes | 优雅排空后重启 Gateway |
| `/status` | Session | No | No | 显示会话信息 |
| `/profile` | Info | No | No | 显示活动 profile 名称和主目录 |
| `/help` | Info | No | No | 显示可用命令 |
| `/commands` | Info | No | Yes | 浏览所有命令和技能（分页） |
| `/retry` | Session | No | No | 重发上一条消息给 Agent |
| `/undo` | Session | No | No | 移除最后一轮用户/助手对话 |
| `/title` | Session | No | No | 设置当前会话标题 |
| `/branch` (alias: `/fork`) | Session | No | No | 分支当前会话（探索不同路径） |
| `/compress` | Session | No | No | 手动压缩对话上下文 |
| `/rollback` | Session | No | No | 列出或恢复文件系统检查点 |
| `/snapshot` (alias: `/snap`) | Session | No | No | 创建/恢复 Hermes 状态快照 |
| `/background` (alias: `/bg`) | Session | Yes | No | 在后台运行提示词 |
| `/btw` | Session | No | No | 临时侧问（使用会话上下文，无工具，不持久化） |
| `/queue` (alias: `/q`) | Session | No | No | 排队提示词到下一个回合（不打断） |
| `/approve` | Session | No | Yes | 批准待处理危险命令 |
| `/deny` | Session | No | Yes | 拒绝待处理危险命令 |
| `/sethome` (alias: `/set-home`) | Session | No | Yes | 设置当前聊天为 home channel |
| `/resume` | Session | No | No | 恢复之前命名的会话 |
| `/model` | Configuration | No | No | 切换当前会话模型 |
| `/provider` | Configuration | No | No | 显示可用 provider 和当前 provider |
| `/personality` | Configuration | No | No | 设置预定义 personality |
| `/verbose` | Configuration | No | No | 切换工具进度展示（config-gated） |
| `/yolo` | Configuration | No | No | 切换 YOLO 模式（跳过危险命令审批） |
| `/reasoning` | Configuration | No | No | 管理推理力度和展示 |
| `/fast` | Configuration | No | No | 切换 Fast 模式 |
| `/voice` | Configuration | No | No | 切换语音模式 |
| `/reload` | Tools & Skills | No | No | 重新加载 .env 变量 |
| `/reload-mcp` (alias: `/reload_mcp`) | Tools & Skills | No | No | 从配置重新加载 MCP 服务器 |
| `/debug` | Info | No | No | 上传 debug 报告（系统信息 + 日志） |
| `/usage` | Info | No | No | 显示当前会话的 token 用量和速率限制 |
| `/insights` | Info | No | No | 显示用量洞察和分析 |
| `/update` | Info | No | Yes | 更新 Hermes Agent 到最新版本 |

**说明**：
- "绕过活跃会话守卫" 表示命令可以在 Agent 正在处理时执行（如 `/new`、`/approve`、`/background`）
- Gateway 专有命令（`gateway_only=True`）仅在 Gateway 对话中可用，CLI 中不显示
- Config-gated 命令（`gateway_config_gate`）在 CLI 中为 `cli_only`，但在 Gateway 中通过配置门控可用
- `/plan` 等技能命令也通过 `GATEWAY_KNOWN_COMMANDS` 识别并路由到技能系统

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

#### 优雅关停核心代码
```python
async def stop(self, *, restart: bool = False) -> None:
    """优雅停止网关"""
    self._running = False
    self._draining = True
    
    # 通知所有活跃会话即将关停
    await self._notify_active_sessions_of_shutdown()
    
    # 等待活跃Agent完成，超时则中断
    timeout = self._restart_drain_timeout
    active_agents, timed_out = await self._drain_active_agents(timeout)
    
    if timed_out:
        logger.warning(f"Drain timed out after {timeout}s, interrupting remaining agents")
        self._interrupt_running_agents("Gateway shutting down")
        # 再给5秒时间让中断处理完成
        interrupt_deadline = asyncio.get_running_loop().time() + 5.0
        while self._running_agents and asyncio.get_running_loop().time() < interrupt_deadline:
            await asyncio.sleep(0.1)
    
    # 断开所有平台适配器
    for platform, adapter in list(self.adapters.items()):
        try:
            await adapter.cancel_background_tasks()
            await adapter.disconnect()
            logger.info(f"✓ {platform.value} disconnected")
        except Exception as e:
            logger.error(f"✗ {platform.value} disconnect error: {e}")
    
    # 清理资源
    process_registry.kill_all()  # 终止所有工具子进程
    cleanup_all_environments()  # 清理所有终端环境
    cleanup_all_browsers()  # 清理所有浏览器实例
    
    # 写入干净关停标记
    if not timed_out:
        (_hermes_home / ".clean_shutdown").touch()
    
    logger.info("Gateway stopped successfully")
```

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

#### 代理自动检测实现
```python
def resolve_proxy_url(platform_env_var: str | None = None) -> str | None:
    """解析代理URL，优先级：平台特定环境变量 > 通用环境变量 > macOS系统代理"""
    if platform_env_var:
        value = (os.environ.get(platform_env_var) or "").strip()
        if value:
            return value
    
    # 检查通用代理环境变量
    for key in ("HTTPS_PROXY", "HTTP_PROXY", "ALL_PROXY",
                "https_proxy", "http_proxy", "all_proxy"):
        value = (os.environ.get(key) or "").strip()
        if value:
            return value
    
    # macOS系统代理检测
    return _detect_macos_system_proxy()

def _detect_macos_system_proxy() -> str | None:
    """通过scutil命令读取macOS系统代理设置"""
    if sys.platform != "darwin":
        return None
    try:
        out = subprocess.check_output(
            ["scutil", "--proxy"], timeout=3, text=True, stderr=subprocess.DEVNULL,
        )
    except Exception:
        return None
    
    props = {}
    for line in out.splitlines():
        line = line.strip()
        if " : " in line:
            key, _, val = line.partition(" : ")
            props[key.strip()] = val.strip()
    
    # 优先HTTPS代理，其次HTTP代理
    for enable_key, host_key, port_key in (
        ("HTTPSEnable", "HTTPSProxy", "HTTPSPort"),
        ("HTTPEnable", "HTTPProxy", "HTTPPort"),
    ):
        if props.get(enable_key) == "1":
            host = props.get(host_key)
            port = props.get(port_key)
            if host and port:
                return f"http://{host}:{port}"
    return None
```

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
7. `gateway/platforms/helpers.py` -- MessageDeduplicator、TextBatchAggregator 等共享工具
8. `gateway/platforms/ADDING_A_PLATFORM.md` -- 添加新平台的 16 项完整清单
