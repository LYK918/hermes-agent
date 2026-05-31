# 03 - MCP 协议实现详细设计

> 本文档基于 `hermes-agent` 项目源码深度分析，覆盖 MCP Server (`mcp_serve.py`)、MCP Client (`tools/mcp_tool.py`)、OAuth 2.1 PKCE (`tools/mcp_oauth.py`) 及 CLI 管理 (`hermes_cli/mcp_config.py`) 四大模块的完整实现细节。

---

## 1. MCP 协议概述

### 1.1 什么是 MCP (Model Context Protocol)

MCP（Model Context Protocol）是由 Anthropic 主导提出的开放协议，旨在为 AI 模型与外部工具/数据源之间建立标准化的通信桥梁。其核心思想是：

- **统一接口**：无论底层是本地文件系统、数据库、API 还是远程服务，MCP 都提供一致的 `tools/list`（发现）、`tools/call`（调用）、`resources/list`（资源列表）、`prompts/list`（提示词列表）等标准方法。
- **传输无关**：协议层与传输层解耦，支持 stdio（子进程管道）和 HTTP/StreamableHTTP（远程流式）两种传输方式。
- **双向通信**：不仅 Client 可以调用 Server 工具，Server 还可以通过 `sampling/createMessage` 请求 Client 端的 LLM 进行推理（反向调用）。
- **动态能力发现**：Server 可以在运行时通过 `notifications/tools/list_changed` 通知 Client 工具列表已变更，Client 据此动态刷新已注册的工具集。

### 1.2 MCP 在 AI Agent 生态中的定位

在 Hermes Agent 架构中，MCP 处于以下位置：

```
用户消息 --> Hermes Agent (LLM Brain)
                  |
                  |--> 内置工具 (tools/*.py) -- 直接函数调用
                  |--> MCP 工具 (tools/mcp_tool.py) -- 通过 MCP 协议调用外部 MCP Server
                  |     |
                  |     |--> stdio 传输: 本地子进程 (如 npx @modelcontextprotocol/server-filesystem)
                  |     |--> HTTP 传输: 远程服务 (如 https://mcp.example.com/mcp)
                  |
                  |--> Hermes 自身作为 MCP Server (mcp_serve.py)
                        |
                        |--> 暴露 10 个消息桥接工具给外部 MCP Client (Claude Code, Cursor 等)
```

MCP 的定位是让 Hermes Agent 既能**消费**外部 MCP Server 提供的工具（作为 Client），也能**暴露**自身能力给其他 AI 工具（作为 Server），形成双向互联的 Agent 生态。

### 1.3 Server vs Client 的角色

| 角色 | 模块 | 职责 |
|------|------|------|
| **MCP Server** | `mcp_serve.py` | 将 Hermes 的消息会话能力封装为 10 个标准 MCP 工具，通过 stdio 传输暴露给任何 MCP Client（Claude Desktop、Cursor、Codex 等） |
| **MCP Client** | `tools/mcp_tool.py` | 连接外部 MCP Server，发现其工具列表，将这些工具注册到 Hermes 的内部工具注册表中，使 LLM Agent 可以像调用内置工具一样调用 MCP 工具 |

---

## 2. MCP Server 实现 (mcp_serve.py)

### 2.1 架构设计

#### 2.1.1 FastMCP 框架的使用

Hermes MCP Server 基于 `mcp` Python SDK 提供的 `FastMCP` 框架构建。`FastMCP` 是一个装饰器驱动的 MCP Server 开发框架，开发者只需用 `@mcp.tool()` 装饰器标注 Python 函数，框架自动完成以下工作：

- 根据函数签名和 docstring 生成 JSON Schema（工具的输入/输出描述）
- 处理 MCP 协议握手（`initialize` / `initialized`）
- 路由 `tools/call` 请求到对应的 Python 函数
- 序列化返回值为 MCP 标准响应格式

**创建 MCP Server 实例的关键代码**（`mcp_serve.py` 第 431-446 行）：

```python
mcp = FastMCP(
    "hermes",
    instructions=(
        "Hermes Agent messaging bridge. Use these tools to interact with "
        "conversations across Telegram, Discord, Slack, WhatsApp, Signal, "
        "Matrix, and other connected platforms."
    ),
)
```

`instructions` 字段在 MCP 的 `initialize` 握手阶段返回给 Client，作为该 Server 的全局上下文说明，帮助 LLM 理解如何使用这些工具。

#### 2.1.2 EventBridge 事件桥接机制

`EventBridge` 是 MCP Server 的核心组件（第 185-425 行），它是一个后台轮询器，职责是：

1. **轮询 SessionDB**：以 200ms 间隔检查 SQLite 数据库中的新消息
2. **维护内存事件队列**：将新消息转换为 `QueueEvent` 对象，存储在内存队列中（最大 1000 条）
3. **支持等待者唤醒**：通过 `threading.Event` 实现 `wait_for_event` 长轮询机制
4. **追踪审批请求**：在内存中维护 `_pending_approvals` 字典

**EventBridge 的数据结构**：

```python
@dataclass
class QueueEvent:
    cursor: int                                    # 递增游标，用于增量拉取
    type: str                                      # "message" | "approval_requested" | "approval_resolved"
    session_key: str = ""                          # 会话标识
    data: dict = field(default_factory=dict)        # 事件负载
```

**线程模型**：

```python
class EventBridge:
    def __init__(self):
        self._queue: List[QueueEvent] = []         # 事件队列
        self._cursor = 0                            # 全局游标
        self._lock = threading.Lock()               # 线程锁
        self._new_event = threading.Event()          # 唤醒信号
        self._running = False                        # 运行标志
        self._thread: Optional[threading.Thread] = None  # 后台线程
        self._last_poll_timestamps: Dict[str, float] = {}  # 上次轮询时间戳
        self._pending_approvals: Dict[str, dict] = {}      # 审批跟踪
        self._sessions_json_mtime: float = 0.0      # mtime 缓存
        self._state_db_mtime: float = 0.0           # mtime 缓存
        self._cached_sessions_index: dict = {}       # 缓存的会话索引
```

#### 2.1.3 200ms 轮询 + mtime 优化

**核心设计问题**：200ms 轮询间隔对于数据库查询来说过于频繁，直接查询会导致大量无效 I/O。

**解决方案：mtime 检查优化**（`_poll_once` 方法，第 327-425 行）：

```python
def _poll_once(self, db):
    # 1. 检查 sessions.json 的 mtime（约 1 微秒）
    sj_mtime = sessions_file.stat().st_mtime if sessions_file.exists() else 0.0
    if sj_mtime != self._sessions_json_mtime:
        self._sessions_json_mtime = sj_mtime
        self._cached_sessions_index = _load_sessions_index()

    # 2. 检查 state.db 的 mtime
    db_mtime = db_file.stat().st_mtime if db_file.exists() else 0.0

    # 3. 如果两个文件都没变化，跳过本轮
    if db_mtime == self._state_db_mtime and sj_mtime == self._sessions_json_mtime:
        return  # 无变化，完全跳过

    # 4. 只有文件变化时才执行数据库查询
    self._state_db_mtime = db_mtime
    entries = self._cached_sessions_index
    # ... 遍历会话，查找新消息
```

**优化效果**：

| 场景 | 无 mtime 优化 | 有 mtime 优化 |
|------|-------------|-------------|
| 无新消息时 | 每 200ms 一次完整 DB 查询 | 每 200ms 两次 `stat()` 调用（~2μs） |
| 有新消息时 | 一次完整 DB 查询 | 两次 `stat()` + 一次 DB 查询 |
| CPU 开销 | 持续高 | 接近零 |

**消息时间戳比较**（`_ts_float` 内部函数）：

时间戳可能以 `int`、`float`、`str`（Unix epoch）或 ISO 格式字符串存储，需要统一转换：

```python
def _ts_float(ts) -> float:
    if isinstance(ts, (int, float)):
        return float(ts)
    if isinstance(ts, str) and ts:
        try:
            return float(ts)
        except ValueError:
            from datetime import datetime
            return datetime.fromisoformat(ts).timestamp()
    return 0.0
```

### 2.2 十个暴露工具详解

#### 2.2.1 conversations_list

**功能**：列出所有活跃的消息会话。

| 项目 | 详情 |
|------|------|
| **输入参数** | `platform: Optional[str]` - 按平台过滤; `limit: int = 50` - 最大返回数; `search: Optional[str]` - 文本搜索过滤 |
| **输出格式** | `{count: int, conversations: [{session_key, session_id, platform, chat_type, display_name, chat_name, user_name, updated_at}]}` |
| **数据源** | `_load_sessions_index()` -> `sessions.json` 文件 |
| **排序** | 按 `updated_at` 降序 |
| **搜索逻辑** | 匹配 `display_name`、`chat_name`、`session_key` 的子串（大小写不敏感） |

#### 2.2.2 conversation_get

**功能**：获取单个会话的详细信息。

| 项目 | 详情 |
|------|------|
| **输入参数** | `session_key: str` - 必需，会话标识 |
| **输出格式** | 包含 `session_key, session_id, platform, chat_type, display_name, user_name, chat_name, chat_id, thread_id, updated_at, created_at, input_tokens, output_tokens, total_tokens` |
| **错误处理** | 会话不存在时返回 `{error: "Conversation not found: ..."}` |

#### 2.2.3 messages_read

**功能**：读取会话中的消息历史。

| 项目 | 详情 |
|------|------|
| **输入参数** | `session_key: str` - 会话标识; `limit: int = 50` - 最大消息数 |
| **输出格式** | `{session_key, count, total_in_session, messages: [{id, role, content, timestamp}]}` |
| **过滤逻辑** | 只返回 `role` 为 `user` 或 `assistant` 的消息；内容截断为 2000 字符 |
| **数据源** | `SessionDB.get_messages(session_id)` |
| **消息格式处理** | `_extract_message_content()` 处理多部分内容（`list` 类型的 content 中提取 text 部分并拼接） |

#### 2.2.4 attachments_fetch

**功能**：获取消息中的非文本附件。

| 项目 | 详情 |
|------|------|
| **输入参数** | `session_key: str`, `message_id: str` |
| **输出格式** | `{message_id, count, attachments: [{type, url} 或 {type, data} 或 {type, path}]}` |
| **附件识别逻辑** | `_extract_attachments()` 函数通过三种方式识别附件：<br>1. 多部分内容块中的 `image_url`/`image` 类型<br>2. 文本中的 `MEDIA:` 标签（正则匹配）<br>3. 其他非 text 类型的内容块 |

#### 2.2.5 events_poll

**功能**：轮询获取自某个游标以来的新事件。

| 项目 | 详情 |
|------|------|
| **输入参数** | `after_cursor: int = 0` - 起始游标; `session_key: Optional[str]` - 会话过滤; `limit: int = 20` - 最大事件数 |
| **输出格式** | `{events: [{cursor, type, session_key, ...}], next_cursor: int}` |
| **事件类型** | `message`（新消息）、`approval_requested`（审批请求）、`approval_resolved`（审批响应） |
| **实现** | 委托给 `bridge.poll_events()` |

#### 2.2.6 events_wait（长轮询 5 分钟）

**功能**：长轮询等待下一个事件，避免短轮询的资源浪费。

| 项目 | 详情 |
|------|------|
| **输入参数** | `after_cursor: int = 0`, `session_key: Optional[str]`, `timeout_ms: int = 30000` |
| **超时上限** | `min(timeout_ms, 300000)` = 5 分钟 |
| **输出格式** | 有事件时：`{event: {cursor, type, ...}}`; 超时：`{event: null, reason: "timeout"}` |
| **等待机制** | `wait_for_event()` 使用 `threading.Event.wait()` 实现，每次最多等待 `POLL_INTERVAL`(200ms) 然后重新检查 |

**长轮询实现细节**（第 249-275 行）：

```python
def wait_for_event(self, after_cursor, session_key, timeout_ms):
    deadline = time.monotonic() + (timeout_ms / 1000.0)
    while time.monotonic() < deadline:
        with self._lock:
            for e in self._queue:
                if e.cursor > after_cursor and (...):
                    return {...}
        remaining = deadline - time.monotonic()
        self._new_event.clear()
        self._new_event.wait(timeout=min(remaining, POLL_INTERVAL))
    return None
```

关键设计：使用 `threading.Event.clear()` + `wait()` 模式，既能在新事件到达时立即唤醒（通过 `_enqueue` 中的 `self._new_event.set()`），又能在超时后优雅退出。

#### 2.2.7 messages_send

**功能**：向指定平台的会话发送消息。

| 项目 | 详情 |
|------|------|
| **输入参数** | `target: str` - 格式为 `"platform:chat_id"`; `message: str` - 消息文本 |
| **示例** | `target="telegram:6308981865"`, `target="discord:#general"` |
| **实现** | 委托给 `tools/send_message_tool.py` 的 `send_message_tool()` 函数 |
| **错误处理** | target 或 message 为空时返回错误；导入失败或发送异常时返回错误 JSON |

#### 2.2.8 channels_list

**功能**：列出所有可用的消息通道和目标。

| 项目 | 详情 |
|------|------|
| **输入参数** | `platform: Optional[str]` - 按平台过滤 |
| **输出格式** | `{count: int, channels: [{target, platform, name, chat_type}]}` |
| **数据源优先级** | 1. `channel_directory.json`（缓存的通道目录）; 2. `sessions.json`（回退方案） |
| **target 格式** | `"platform:chat_id"`，可直接用于 `messages_send` |

#### 2.2.9 permissions_list_open

**功能**：列出当前桥接会话中观察到的待审批请求。

| 项目 | 详情 |
|------|------|
| **输入参数** | 无 |
| **输出格式** | `{count: int, approvals: [sorted by created_at]}` |
| **限制** | 只返回桥接启动后的审批请求（实时会话限制） |

#### 2.2.10 permissions_respond

**功能**：响应一个待审批请求。

| 项目 | 详情 |
|------|------|
| **输入参数** | `id: str` - 审批 ID; `decision: str` - 必须为 `"allow-once"`, `"allow-always"`, 或 `"deny"` |
| **输出格式** | `{resolved: true, approval_id, decision}` 或 `{error: "..."}` |
| **实现** | 从 `_pending_approvals` 字典中移除对应条目，并向事件队列添加 `approval_resolved` 事件 |

### 2.3 SessionDB 集成

#### 2.3.1 读取会话数据

SessionDB 是 Hermes 的会话持久化层，基于 SQLite 实现。MCP Server 通过以下方式集成：

```python
def _get_session_db():
    """Get a SessionDB instance for reading message transcripts."""
    from hermes_state import SessionDB
    return SessionDB()
```

主要使用的 API：
- `db.get_messages(session_id)` - 获取指定会话的所有消息

#### 2.3.2 FTS5 全文搜索

SessionDB 底层使用 SQLite 的 FTS5（Full-Text Search 5）扩展实现高效的全文搜索。这使得 `conversations_list` 的 `search` 参数可以快速匹配大量消息内容，而不需要逐条扫描。

### 2.4 启动入口

```python
def run_mcp_server(verbose: bool = False) -> None:
    bridge = EventBridge()
    bridge.start()
    server = create_mcp_server(event_bridge=bridge)
    asyncio.run(server.run_stdio_async())
```

启动流程：
1. 创建 `EventBridge` 实例并启动后台轮询线程
2. 调用 `create_mcp_server()` 注册所有 10 个工具
3. 通过 `FastMCP.run_stdio_async()` 在 stdio 传输上运行 MCP Server
4. 关闭时停止 EventBridge

---

## 3. MCP Client 实现 (tools/mcp_tool.py) 深度剖析

### 3.1 整体架构

`tools/mcp_tool.py` 是 Hermes Agent 的 MCP Client 实现，共约 2274 行代码。其核心架构如下：

```
                            主线程 (Agent)
                                |
                    register_mcp_servers() / discover_mcp_tools()
                                |
                    _ensure_mcp_loop() -- 启动后台事件循环线程
                                |
                    _run_on_mcp_loop(_discover_all())
                                |
                    ┌───────────┼───────────┐
                    v           v           v
            MCPServerTask  MCPServerTask  MCPServerTask
            (server-1)     (server-2)     (server-3)
                    |           |           |
            [asyncio Task] [asyncio Task] [asyncio Task]
                    |           |           |
            stdio/HTTP     stdio/HTTP     HTTP
                    |           |           |
            MCP Server   MCP Server   MCP Server
            (子进程)      (子进程)      (远程)
```

#### 3.1.1 MCPServerTask: 每个 MCP 服务器一个 asyncio Task

**设计核心**：每个 MCP Server 的完整生命周期（连接、发现、服务、断开）都在一个专用的 `asyncio.Task` 中运行。这样设计的原因是：

- MCP SDK 使用 `anyio` 的 cancel-scope 进行资源清理
- `anyio` 要求 cancel-scope 的进入和退出必须在同一个 Task 中
- 如果多个 Server 共享一个 Task，一个 Server 的断开会影响其他 Server

**MCPServerTask 类结构**（第 774-1161 行）：

```python
class MCPServerTask:
    __slots__ = (
        "name", "session", "tool_timeout",
        "_task", "_ready", "_shutdown_event", "_tools", "_error", "_config",
        "_sampling", "_registered_tool_names", "_auth_type", "_refresh_lock",
    )
```

关键状态：
- `session`: MCP `ClientSession` 实例，用于调用 `list_tools()`、`call_tool()` 等方法
- `_ready`: `asyncio.Event`，当连接成功并完成工具发现后 set，让外部知道 Server 已就绪
- `_shutdown_event`: `asyncio.Event`，当需要关闭时 set，触发 Task 退出
- `_tools`: 从 Server 发现的工具列表
- `_registered_tool_names`: 已注册到 Hermes 工具注册表的工具名称列表
- `_refresh_lock`: `asyncio.Lock`，防止并发刷新导致的竞态条件

#### 3.1.2 长连接管理

每个 MCPServerTask 是一个**长连接**——连接建立后一直保持，直到：
1. 显式调用 `shutdown()` 方法
2. 连接断开且重试次数用尽
3. 主进程退出

#### 3.1.3 工具发现与注册

工具发现通过 `_discover_tools()` 方法完成：

```python
async def _discover_tools(self):
    tools_result = await self.session.list_tools()
    self._tools = tools_result.tools if hasattr(tools_result, "tools") else []
```

工具注册通过 `_register_server_tools()` 函数完成，它将 MCP 工具转换为 Hermes 工具注册表的格式并注册：

```python
def _register_server_tools(name, server, config):
    for mcp_tool in server._tools:
        schema = _convert_mcp_schema(name, mcp_tool)
        registry.register(
            name=schema["name"],      # 前缀格式: mcp_{server}_{tool}
            toolset=f"mcp-{name}",
            schema=schema,
            handler=_make_tool_handler(name, mcp_tool.name, server.tool_timeout),
            check_fn=_make_check_fn(name),
            is_async=False,
            description=schema["description"],
        )
```

**工具命名规则**：`mcp_{server_name}_{tool_name}`，其中 server_name 和 tool_name 中的特殊字符（包括连字符）被转换为下划线。

### 3.2 两种传输协议

#### 3.2.1 stdio: 命令+参数，子进程通信

**配置示例**：
```yaml
mcp_servers:
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_..."
    timeout: 120
    connect_timeout: 60
```

**实现**（`_run_stdio` 方法，第 890-939 行）：

```python
async def _run_stdio(self, config):
    command = config.get("command")
    args = config.get("args", [])
    user_env = config.get("env")

    safe_env = _build_safe_env(user_env)              # 环境变量白名单过滤
    command, safe_env = _resolve_stdio_command(command, safe_env)  # 命令解析

    # OSV 恶意软件检查
    from tools.osv_check import check_package_for_malware
    malware_error = check_package_for_malware(command, args)
    if malware_error:
        raise ValueError(f"MCP server '{self.name}': {malware_error}")

    server_params = StdioServerParameters(command=command, args=args, env=safe_env)

    pids_before = _snapshot_child_pids()
    async with stdio_client(server_params) as (read_stream, write_stream):
        new_pids = _snapshot_child_pids() - pids_before
        if new_pids:
            _stdio_pids.update(new_pids)
        async with ClientSession(read_stream, write_stream, **kwargs) as session:
            await session.initialize()
            self.session = session
            await self._discover_tools()
            self._ready.set()
            await self._shutdown_event.wait()
```

**关键细节**：
- 使用 `_snapshot_child_pids()` 追踪子进程 PID，用于关闭时强制清理
- `_resolve_stdio_command()` 解析命令路径，特别处理 `npx`/`npm`/`node` 等 Node.js 命令
- `_build_safe_env()` 进行环境变量白名单过滤（安全机制）

**适用场景**：本地工具（文件系统操作、Git 操作、本地数据库等）

#### 3.2.2 HTTP/StreamableHTTP: URL+headers，远程通信

**配置示例**：
```yaml
mcp_servers:
  remote_api:
    url: "https://my-mcp-server.example.com/mcp"
    headers:
      Authorization: "Bearer ${API_KEY}"
    timeout: 180
    connect_timeout: 60
    auth: oauth
    oauth:
      client_id: "pre-registered-id"
      scope: "read write"
```

**实现**（`_run_http` 方法，第 941-1015 行）：

```python
async def _run_http(self, config):
    url = config["url"]
    headers = dict(config.get("headers") or {})
    connect_timeout = config.get("connect_timeout", _DEFAULT_CONNECT_TIMEOUT)

    # OAuth 处理
    if self._auth_type == "oauth":
        from tools.mcp_oauth import build_oauth_auth
        _oauth_auth = build_oauth_auth(self.name, url, config.get("oauth"))

    if _MCP_NEW_HTTP:
        # mcp >= 1.24.0 新 API
        async with httpx.AsyncClient(**client_kwargs) as http_client:
            async with streamable_http_client(url, http_client=http_client) as (read, write, _):
                async with ClientSession(read, write, **kwargs) as session:
                    await session.initialize()
                    # ...
    else:
        # mcp < 1.24.0 旧 API
        async with streamablehttp_client(url, **_http_kwargs) as (read, write, _):
            async with ClientSession(read, write, **kwargs) as session:
                await session.initialize()
                # ...
```

**版本兼容性**：代码同时支持 `mcp >= 1.24.0`（新 API `streamable_http_client`）和旧版本（`streamablehttp_client`），通过 `_MCP_NEW_HTTP` 标志位自动选择。

**适用场景**：远程 API 服务、云端工具、需要 OAuth 认证的服务

#### 3.2.3 每种传输的适用场景对比

| 特性 | stdio | HTTP/StreamableHTTP |
|------|-------|---------------------|
| 通信方式 | 子进程 stdin/stdout 管道 | HTTP 请求/响应 + SSE 流 |
| 延迟 | 极低（本地进程） | 较高（网络往返） |
| 部署 | 需要本地安装命令 | 只需 URL |
| 认证 | 环境变量 | Headers、OAuth 2.1 |
| 安全性 | 环境变量白名单 | HTTPS + Token |
| 典型用途 | 本地文件系统、Git | 远程 API、云端服务 |

### 3.3 MCPServerTask 生命周期

#### 3.3.1 连接建立

```python
async def run(self, config: dict):
    self._config = config
    self.tool_timeout = config.get("timeout", _DEFAULT_TOOL_TIMEOUT)
    self._auth_type = (config.get("auth") or "").lower().strip()

    # 设置采样处理器
    if sampling_config.get("enabled", True) and _MCP_SAMPLING_TYPES:
        self._sampling = SamplingHandler(self.name, sampling_config)

    while True:
        try:
            if self._is_http():
                await self._run_http(config)
            else:
                await self._run_stdio(config)
            break  # 正常退出（shutdown 请求）
        except Exception as exc:
            # 重连逻辑...
```

#### 3.3.2 工具发现 (tools/list)

```python
async def _discover_tools(self):
    tools_result = await self.session.list_tools()
    self._tools = tools_result.tools if hasattr(tools_result, "tools") else []
```

调用 MCP 协议的 `tools/list` 方法，获取 Server 暴露的所有工具。

#### 3.3.3 工具调用 (tools/call)

工具调用通过 `_make_tool_handler` 工厂函数生成的同步处理器完成：

```python
def _make_tool_handler(server_name, tool_name, tool_timeout):
    def _handler(args, **kwargs) -> str:
        async def _call():
            result = await server.session.call_tool(tool_name, arguments=args)
            if result.isError:
                return json.dumps({"error": _sanitize_error(error_text)})
            # 收集 content blocks 中的文本
            parts = [block.text for block in result.content if hasattr(block, "text")]
            # 处理 structuredContent
            structured = getattr(result, "structuredContent", None)
            return json.dumps({"result": text_result, "structuredContent": structured})
        return _run_on_mcp_loop(_call(), timeout=tool_timeout)
    return _handler
```

**关键设计**：处理器是同步的（`is_async=False`），内部通过 `_run_on_mcp_loop()` 将异步调用调度到 MCP 后台事件循环上。

#### 3.3.4 动态刷新 (tools/list_changed)

当 MCP Server 发送 `notifications/tools/list_changed` 通知时，触发 `_refresh_tools()` 方法：

```python
async def _refresh_tools(self):
    async with self._refresh_lock:
        old_tool_names = set(self._registered_tool_names)
        # 1. 获取新工具列表
        tools_result = await self.session.list_tools()
        # 2. 注销旧工具
        for prefixed_name in self._registered_tool_names:
            registry.deregister(prefixed_name)
        # 3. 注册新工具
        self._tools = new_mcp_tools
        self._registered_tool_names = _register_server_tools(self.name, self, self._config)
        # 4. 记录变更
        added = new_tool_names - old_tool_names
        removed = old_tool_names - new_tool_names
```

#### 3.3.5 断开连接

```python
async def shutdown(self):
    self._shutdown_event.set()  # 信号退出
    if self._task and not self._task.done():
        await asyncio.wait_for(self._task, timeout=10)  # 等待优雅退出
    for tool_name in self._registered_tool_names:
        registry.deregister(tool_name)  # 注销所有工具
    self.session = None
```

### 3.4 SamplingHandler（服务端发起的 LLM 请求）

#### 3.4.1 MCP sampling/createMessage 协议

MCP 协议允许 Server 通过 `sampling/createMessage` 方法请求 Client 端的 LLM 进行推理。这在以下场景中非常有用：

- Server 需要 LLM 处理用户请求的一部分
- Server 需要 LLM 生成工具调用的参数
- Server 需要 LLM 对结果进行总结

#### 3.4.2 消息格式转换: MCP -> OpenAI

`SamplingHandler._convert_messages()` 方法负责将 MCP 消息格式转换为 OpenAI 兼容格式：

```python
def _convert_messages(self, params) -> List[dict]:
    messages = []
    for msg in params.messages:
        blocks = msg.content_as_list if hasattr(msg, "content_as_list") else (
            msg.content if isinstance(msg.content, list) else [msg.content]
        )

        # 分离不同类型的块
        tool_results = [b for b in blocks if hasattr(b, "toolUseId")]
        tool_uses = [b for b in blocks if hasattr(b, "name") and hasattr(b, "input") ...]
        content_blocks = [b for b in blocks if not hasattr(b, "toolUseId") ...]

        # 转换为 OpenAI 格式
        for tr in tool_results:
            messages.append({"role": "tool", "tool_call_id": tr.toolUseId, "content": ...})
        if tool_uses:
            tc_list = [{"id": ..., "type": "function", "function": {"name": ..., "arguments": ...}}]
            messages.append({"role": msg.role, "tool_calls": tc_list})
        elif content_blocks:
            messages.append({"role": msg.role, "content": ...})
    return messages
```

**支持的 MCP 内容类型**：
- `TextContent` - 文本内容
- `ImageContent` - 图片（转为 `data:` URI）
- `ToolUseContent` - 工具调用请求
- `ToolResultContent` - 工具调用结果

#### 3.4.3 滑动窗口速率限制

```python
def _check_rate_limit(self) -> bool:
    """滑动窗口速率限制器"""
    now = time.time()
    window = now - 60  # 60 秒窗口
    self._rate_timestamps[:] = [t for t in self._rate_timestamps if t > window]
    if len(self._rate_timestamps) >= self.max_rpm:
        return False
    self._rate_timestamps.append(now)
    return True
```

配置项 `max_rpm`（默认 10）控制每分钟最大请求数。

#### 3.4.4 工具循环治理

防止 Server 通过 sampling 请求无限循环调用工具：

```python
def _build_tool_use_result(self, choice, response):
    if self.max_tool_rounds == 0:
        return self._error("Tool loops disabled")
    self._tool_loop_count += 1
    if self._tool_loop_count > self.max_tool_rounds:
        self._tool_loop_count = 0
        return self._error(f"Tool loop limit exceeded (max {self.max_tool_rounds} rounds)")
```

**采样配置示例**：
```yaml
sampling:
  enabled: true
  model: "gemini-3-flash"       # 覆盖模型
  max_tokens_cap: 4096          # 最大 token 数
  timeout: 30                   # LLM 调用超时（秒）
  max_rpm: 10                   # 每分钟最大请求数
  allowed_models: []            # 模型白名单（空 = 允许所有）
  max_tool_rounds: 5            # 工具循环限制（0 = 禁用）
  log_level: "info"             # 审计日志级别
```

### 3.5 动态工具发现与刷新

#### 3.5.1 tools/list_changed 通知处理

MCP Server 可以通过发送 `notifications/tools/list_changed` 通知来告知 Client 工具列表已变更。Hermes 的处理流程：

```python
def _make_message_handler(self):
    async def _handler(message):
        if isinstance(message, ServerNotification):
            match message.root:
                case ToolListChangedNotification():
                    await self._refresh_tools()
                case PromptListChangedNotification():
                    pass  # 未来支持
                case ResourceListChangedNotification():
                    pass  # 未来支持
    return _handler
```

**通知处理能力检测**（第 141-157 行）：

```python
def _check_message_handler_support() -> bool:
    """检查 ClientSession 是否支持 message_handler 参数"""
    return "message_handler" in inspect.signature(ClientSession).parameters

_MCP_MESSAGE_HANDLER_SUPPORTED = _check_message_handler_support()
```

只有当 MCP SDK 版本支持 `message_handler` 参数时，才会注册通知处理器。

#### 3.5.2 _refresh_tools 原子操作（删旧+注册新）

```python
async def _refresh_tools(self):
    async with self._refresh_lock:  # 防止并发刷新
        # 1. 记录旧工具名称
        old_tool_names = set(self._registered_tool_names)

        # 2. 从 Server 获取新工具列表（唯一的异步操作）
        tools_result = await self.session.list_tools()

        # 3. 注销旧工具（同步操作，原子性）
        for prefixed_name in self._registered_tool_names:
            registry.deregister(prefixed_name)

        # 4. 注册新工具（同步操作，原子性）
        self._tools = new_mcp_tools
        self._registered_tool_names = _register_server_tools(...)

        # 5. 记录变更
        added = new_tool_names - old_tool_names
        removed = old_tool_names - new_tool_names
```

**原子性保证**：
- `asyncio.Lock` 防止并发刷新
- `await session.list_tools()` 是唯一的异步操作
- 注销和注册操作是同步的，从事件循环角度看是原子的

#### 3.5.3 冲突保护（不覆盖内置工具）

```python
# 在 _register_server_tools 中
existing_toolset = registry.get_toolset_for_tool(tool_name_prefixed)
if existing_toolset and not existing_toolset.startswith("mcp-"):
    logger.warning(
        "MCP server '%s': tool '%s' (-> '%s') collides with built-in "
        "tool in toolset '%s' -- skipping to preserve built-in",
        name, mcp_tool.name, tool_name_prefixed, existing_toolset,
    )
    continue
```

**保护逻辑**：如果 MCP 工具的名称与内置工具（toolset 不以 `mcp-` 开头）冲突，则跳过该 MCP 工具，保护内置工具不被覆盖。

### 3.6 自动重连机制

#### 3.6.1 指数退避（首次 3 次 + 运行时 5 次，最大 60s）

重连逻辑在 `MCPServerTask.run()` 方法中实现：

```python
async def run(self, config: dict):
    retries = 0
    initial_retries = 0
    backoff = 1.0

    while True:
        try:
            if self._is_http():
                await self._run_http(config)
            else:
                await self._run_stdio(config)
            break  # 正常退出
        except Exception as exc:
            self.session = None

            # 首次连接重试（3 次）
            if not self._ready.is_set():
                initial_retries += 1
                if initial_retries > _MAX_INITIAL_CONNECT_RETRIES:  # 3
                    self._error = exc
                    self._ready.set()
                    return
                await asyncio.sleep(backoff)
                backoff = min(backoff * 2, _MAX_BACKOFF_SECONDS)  # 最大 60s
                continue

            # 运行时重连（5 次）
            retries += 1
            if retries > _MAX_RECONNECT_RETRIES:  # 5
                return
            await asyncio.sleep(backoff)
            backoff = min(backoff * 2, _MAX_BACKOFF_SECONDS)
```

**退避策略**：
- 首次连接：最多重试 3 次
- 运行时断连：最多重试 5 次
- 退避间隔：1s -> 2s -> 4s -> 8s -> 16s -> 32s -> 60s（最大 60 秒）
- 如果在退避等待期间收到 shutdown 信号，立即退出

#### 3.6.2 连接状态跟踪

```python
# _ready: asyncio.Event
# 初始状态: 未 set
# 连接成功后: set
# 连接失败后: set (同时设置 _error)

# 检查连接是否存活
def _make_check_fn(server_name):
    def _check() -> bool:
        server = _servers.get(server_name)
        return server is not None and server.session is not None
    return _check
```

#### 3.6.3 故障转移

当某个 MCP Server 连接失败时，不影响其他 Server 的正常工作。`register_mcp_servers` 使用 `asyncio.gather(return_exceptions=True)` 并行连接所有 Server，某个 Server 的失败不会阻塞其他 Server。

```python
async def _discover_all():
    results = await asyncio.gather(
        *(_discover_one(name, cfg) for name, cfg in new_servers.items()),
        return_exceptions=True,
    )
    for name, result in zip(server_names, results):
        if isinstance(result, Exception):
            logger.warning("Failed to connect to MCP server '%s': %s", name, ...)
```

### 3.7 安全机制

#### 3.7.1 环境变量白名单过滤（只允许 PATH/HOME 等）

```python
_SAFE_ENV_KEYS = frozenset({
    "PATH", "HOME", "USER", "LANG", "LC_ALL", "TERM", "SHELL", "TMPDIR",
})

def _build_safe_env(user_env: Optional[dict]) -> dict:
    env = {}
    for key, value in os.environ.items():
        if key in _SAFE_ENV_KEYS or key.startswith("XDG_"):
            env[key] = value
    if user_env:
        env.update(user_env)  # 用户显式指定的变量
    return env
```

**安全目的**：防止将主进程的环境变量（可能包含 API Key、Token 等敏感信息）泄露给 MCP Server 子进程。只有安全的基础变量（PATH、HOME 等）和用户显式指定的变量才会传递。

#### 3.7.2 凭证正则脱敏（错误消息中）

```python
_CREDENTIAL_PATTERN = re.compile(
    r"(?:"
    r"ghp_[A-Za-z0-9_]{1,255}"           # GitHub PAT
    r"|sk-[A-Za-z0-9_]{1,255}"           # OpenAI-style key
    r"|Bearer\s+\S+"                      # Bearer token
    r"|token=[^\s&,;\"']{1,255}"         # token=...
    r"|key=[^\s&,;\"']{1,255}"           # key=...
    r"|API_KEY=[^\s&,;\"']{1,255}"       # API_KEY=...
    r"|password=[^\s&,;\"']{1,255}"      # password=...
    r"|secret=[^\s&,;\"']{1,255}"        # secret=...
    r")",
    re.IGNORECASE,
)

def _sanitize_error(text: str) -> str:
    return _CREDENTIAL_PATTERN.sub("[REDACTED]", text)
```

**覆盖的凭证模式**：
- GitHub Personal Access Token (`ghp_...`)
- OpenAI 风格的 API Key (`sk-...`)
- Bearer Token (`Bearer ...`)
- URL 参数中的 token/key/password/secret
- 环境变量风格的 `API_KEY=...`

#### 3.7.3 MCP 工具描述注入扫描（10 种模式）

```python
_MCP_INJECTION_PATTERNS = [
    (re.compile(r"ignore\s+(all\s+)?previous\s+instructions", re.I),
     "prompt override attempt ('ignore previous instructions')"),
    (re.compile(r"you\s+are\s+now\s+a", re.I),
     "identity override attempt ('you are now a...')"),
    (re.compile(r"your\s+new\s+(task|role|instructions?)\s+(is|are)", re.I),
     "task override attempt"),
    (re.compile(r"system\s*:\s*", re.I),
     "system prompt injection attempt"),
    (re.compile(r"<\s*(system|human|assistant)\s*>", re.I),
     "role tag injection attempt"),
    (re.compile(r"do\s+not\s+(tell|inform|mention|reveal)", re.I),
     "concealment instruction"),
    (re.compile(r"(curl|wget|fetch)\s+https?://", re.I),
     "network command in description"),
    (re.compile(r"base64\.(b64decode|decodebytes)", re.I),
     "base64 decode reference"),
    (re.compile(r"exec\s*\(|eval\s*\(", re.I),
     "code execution reference"),
    (re.compile(r"import\s+(subprocess|os|shutil|socket)", re.I),
     "dangerous import reference"),
]
```

**扫描时机**：在工具注册时扫描工具描述

```python
def _scan_mcp_description(server_name, tool_name, description):
    findings = []
    for pattern, reason in _MCP_INJECTION_PATTERNS:
        if pattern.search(description):
            findings.append(reason)
    if findings:
        logger.warning("MCP server '%s' tool '%s': suspicious description content -- %s",
                       server_name, tool_name, "; ".join(findings))
    return findings
```

**设计决策**：只记录警告日志，不阻断工具注册。原因是误报会破坏合法的 MCP Server。

#### 3.7.4 OSV 恶意软件数据库检查

在 stdio 传输启动子进程之前，会检查命令对应的包是否在 OSV（Open Source Vulnerabilities）恶意软件数据库中：

```python
from tools.osv_check import check_package_for_malware
malware_error = check_package_for_malware(command, args)
if malware_error:
    raise ValueError(f"MCP server '{self.name}': {malware_error}")
```

#### 3.7.5 子进程安全

**PID 追踪**（第 1183-1206 行）：

```python
def _snapshot_child_pids() -> set:
    my_pid = os.getpid()
    # Linux: /proc 文件系统
    try:
        children_path = f"/proc/{my_pid}/task/{my_pid}/children"
        with open(children_path) as f:
            return {int(p) for p in f.read().split()}
    except (FileNotFoundError, OSError, ValueError):
        pass
    # 回退: psutil
    try:
        import psutil
        return {c.pid for c in psutil.Process(my_pid).children()}
    except Exception:
        pass
    return set()
```

**孤儿进程清理**（第 2231-2253 行）：

```python
def _kill_orphaned_mcp_children():
    """强制杀死在事件循环关闭后仍存活的 MCP stdio 子进程"""
    with _lock:
        pids = list(_stdio_pids)
        _stdio_pids.clear()
    for pid in pids:
        try:
            os.kill(pid, kill_signal)  # SIGKILL 或 SIGTERM
        except (ProcessLookupError, PermissionError, OSError):
            pass
```

### 3.8 模块级状态与线程安全

```python
_servers: Dict[str, MCPServerTask] = {}                    # 已连接的 Server
_mcp_loop: Optional[asyncio.AbstractEventLoop] = None      # 后台事件循环
_mcp_thread: Optional[threading.Thread] = None              # 后台线程
_lock = threading.Lock()                                     # 全局锁
_stdio_pids: set = set()                                     # stdio 子进程 PID
```

**线程安全设计**：所有对 `_servers`、`_mcp_loop`、`_mcp_thread`、`_stdio_pids` 的访问都通过 `_lock` 保护，确保在 Python 3.13+ 的 free-threading 模式下也能安全工作。

### 3.9 工具调用的跨线程调度

```python
def _run_on_mcp_loop(coro, timeout: float = 30):
    """将协程调度到 MCP 后台事件循环上并阻塞等待结果"""
    with _lock:
        loop = _mcp_loop
    future = asyncio.run_coroutine_threadsafe(coro, loop)
    deadline = time.monotonic() + timeout

    while True:
        if is_interrupted():          # 检查用户中断
            future.cancel()
            raise InterruptedError("User sent a new message")
        try:
            return future.result(timeout=0.1)  # 100ms 轮询间隔
        except concurrent.futures.TimeoutError:
            continue
```

**设计亮点**：
1. 使用 `asyncio.run_coroutine_threadsafe()` 跨线程调度协程
2. 100ms 轮询间隔检查用户中断，实现"可中断的阻塞等待"
3. 超时控制防止工具调用无限阻塞

---

## 4. OAuth 2.1 PKCE 流程 (tools/mcp_oauth.py)

### 4.1 PKCE 流程详解

#### 4.1.1 PKCE 概述

PKCE（Proof Key for Code Exchange）是 OAuth 2.1 的强制要求，用于防止授权码拦截攻击。Hermes 的实现基于 MCP Python SDK 的 `OAuthClientProvider`。

#### 4.1.2 构建 OAuth Auth 对象

`build_oauth_auth()` 函数（第 378-482 行）是 OAuth 流程的入口：

```python
def build_oauth_auth(server_name, server_url, oauth_config=None):
    cfg = oauth_config or {}

    # 1. Token 存储
    storage = HermesTokenStorage(server_name)

    # 2. 回调端口
    redirect_port = int(cfg.get("redirect_port", 0))
    if redirect_port == 0:
        redirect_port = _find_free_port()  # 自动选择可用端口

    # 3. Client 元数据
    client_metadata = OAuthClientMetadata.model_validate({
        "client_name": cfg.get("client_name", "Hermes Agent"),
        "redirect_uris": [f"http://127.0.0.1:{redirect_port}/callback"],
        "grant_types": ["authorization_code", "refresh_token"],
        "response_types": ["code"],
        "token_endpoint_auth_method": "none",  # 或 "client_secret_post"
    })

    # 4. 预注册 Client（可选）
    client_id = cfg.get("client_id")
    if client_id:
        client_info = OAuthClientInformationFull.model_validate({...})
        _write_json(storage._client_info_path(), client_info.model_dump(...))

    # 5. 构建 Provider
    provider = OAuthClientProvider(
        server_url=base_url,
        client_metadata=client_metadata,
        storage=storage,
        redirect_handler=_redirect_handler,
        callback_handler=_wait_for_callback,
        timeout=float(cfg.get("timeout", 300)),
    )
    return provider
```

#### 4.1.3 授权 URL 重定向处理

```python
async def _redirect_handler(authorization_url: str) -> None:
    # 打印授权 URL
    print(f"\n  MCP OAuth: authorization required.\n"
          f"  Open this URL in your browser:\n\n    {authorization_url}\n",
          file=sys.stderr)

    # 尝试自动打开浏览器
    if _can_open_browser():
        webbrowser.open(authorization_url)
```

**浏览器检测逻辑**（`_can_open_browser()`）：
- SSH 会话（`SSH_CLIENT`/`SSH_TTY`）-> 不打开
- macOS/Windows -> 打开
- Linux: 需要 `DISPLAY` 或 `WAYLAND_DISPLAY` 环境变量

#### 4.1.4 临时本地 HTTP 服务器接收回调

```python
def _make_callback_handler():
    result = {"auth_code": None, "state": None, "error": None}

    class _Handler(BaseHTTPRequestHandler):
        def do_GET(self):
            params = parse_qs(urlparse(self.path).query)
            result["auth_code"] = params.get("code", [None])[0]
            result["state"] = params.get("state", [None])[0]
            result["error"] = params.get("error", [None])[0]
            # 返回成功/失败页面
    return _Handler, result
```

```python
async def _wait_for_callback() -> tuple[str, str | None]:
    handler_cls, result = _make_callback_handler()
    server = HTTPServer(("127.0.0.1", _oauth_port), handler_cls)
    server_thread = threading.Thread(target=server.handle_request, daemon=True)
    server_thread.start()

    # 轮询等待结果（300 秒超时）
    timeout = 300.0
    elapsed = 0.0
    while elapsed < timeout:
        if result["auth_code"] is not None or result["error"] is not None:
            break
        await asyncio.sleep(0.5)
        elapsed += 0.5

    return result["auth_code"], result["state"]
```

#### 4.1.5 Token 交换

Token 交换由 MCP SDK 的 `OAuthClientProvider` 自动处理，Hermes 只需提供存储和回调接口。

### 4.2 Token 存储

#### 4.2.1 HermesTokenStorage

```python
class HermesTokenStorage:
    """持久化 OAuth tokens 和 client 注册信息到 JSON 文件"""

    def __init__(self, server_name: str):
        self._server_name = _safe_filename(server_name)  # 文件名安全化

    def _tokens_path(self) -> Path:
        return _get_token_dir() / f"{self._server_name}.json"

    def _client_info_path(self) -> Path:
        return _get_token_dir() / f"{self._server_name}.client.json"
```

**文件布局**：
```
HERMES_HOME/mcp-tokens/
    my_server.json           -- OAuth tokens
    my_server.client.json    -- Client 注册信息
```

#### 4.2.2 文件权限 0o600

```python
def _write_json(path: Path, data: dict) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp = path.with_suffix(".tmp")
    try:
        tmp.write_text(json.dumps(data, indent=2, default=str), encoding="utf-8")
        os.chmod(tmp, 0o600)  # 仅所有者可读写
        tmp.rename(path)       # 原子替换
    except OSError:
        tmp.unlink(missing_ok=True)
        raise
```

#### 4.2.3 原子写入

通过"写入临时文件 -> 设置权限 -> 重命名"三步实现原子写入，防止写入过程中断导致文件损坏。

#### 4.2.4 Token 读取与验证

```python
async def get_tokens(self) -> "OAuthToken | None":
    data = _read_json(self._tokens_path())
    if data is None:
        return None
    try:
        return OAuthToken.model_validate(data)  # Pydantic 验证
    except (ValueError, TypeError, KeyError) as exc:
        logger.warning("Corrupt tokens at %s -- ignoring: %s", self._tokens_path(), exc)
        return None
```

### 4.3 非交互环境处理

#### 4.3.1 无浏览器时的处理

```python
def _is_interactive() -> bool:
    try:
        return sys.stdin.isatty()
    except (AttributeError, ValueError):
        return False

# 在 build_oauth_auth 中
if not _is_interactive() and not storage.has_cached_tokens():
    logger.warning(
        "MCP OAuth for '%s': non-interactive environment and no cached tokens found. "
        "The OAuth flow requires browser authorization. Run interactively first "
        "to complete the initial authorization, then cached tokens will be reused.",
        server_name,
    )
```

**策略**：在非交互环境中，如果有缓存的 tokens，可以继续使用；如果没有，记录警告并允许后续失败（Server 连接失败不会阻塞其他 Server）。

#### 4.3.2 超时机制

```python
# _wait_for_callback 中
timeout = 300.0  # 5 分钟超时
```

如果用户在 5 分钟内没有完成授权，回调超时，抛出 `OAuthNonInteractiveError`。

### 4.4 OAuth 配置示例

```yaml
mcp_servers:
  my_server:
    url: "https://mcp.example.com/mcp"
    auth: oauth
    oauth:
      client_id: "pre-registered-id"        # 跳过动态注册
      client_secret: "secret"               # 仅机密客户端
      scope: "read write"                   # 默认: 服务端提供的 scope
      redirect_port: 0                      # 0 = 自动选择空闲端口
      client_name: "My Custom Client"       # 默认: "Hermes Agent"
      timeout: 300                          # 授权超时（秒）
```

---

## 5. MCP CLI 管理 (hermes_cli/mcp_config.py)

### 5.1 命令概览

```
hermes mcp serve                              # 作为 MCP Server 运行
hermes mcp add <name> --url <endpoint>        # 添加 HTTP MCP 服务器
hermes mcp add <name> --command <cmd>         # 添加 stdio MCP 服务器
hermes mcp add <name> --preset <preset>       # 从预设添加
hermes mcp remove <name>                      # 移除服务器
hermes mcp list                               # 列出所有服务器
hermes mcp test <name>                        # 测试连接
hermes mcp configure <name>                   # 配置工具选择
```

### 5.2 add: 添加 MCP 服务器

`cmd_mcp_add()` 函数实现了完整的 MCP 服务器添加流程：

1. **解析参数**：URL/command/preset/auth/env
2. **应用预设**（`_apply_mcp_preset`）
3. **配置认证**：
   - OAuth: 调用 `build_oauth_auth()`
   - API Key: 提示用户输入，存储到 `.env` 文件
4. **发现工具**：临时连接到服务器，获取工具列表
5. **工具选择**：交互式选择要启用的工具（使用 `curses_checklist`）
6. **保存配置**：写入 `~/.hermes/config.yaml`

**工具发现流程**（`_probe_single_server`）：

```python
def _probe_single_server(name, config, connect_timeout=30):
    _ensure_mcp_loop()
    async def _probe():
        server = await asyncio.wait_for(_connect_server(name, config), timeout=connect_timeout)
        for t in server._tools:
            tools_found.append((t.name, getattr(t, "description", "")))
        await server.shutdown()
    _run_on_mcp_loop(_probe(), timeout=connect_timeout + 10)
    _stop_mcp_loop()
    return tools_found
```

### 5.3 remove: 移除

```python
def cmd_mcp_remove(args):
    name = args.name
    if name not in existing:
        _error(f"Server '{name}' not found in config.")
        return
    if not _confirm(f"Remove server '{name}'?", default=True):
        return
    _remove_mcp_server(name)
    # 清理 OAuth tokens
    from tools.mcp_oauth import remove_oauth_tokens
    remove_oauth_tokens(name)
```

### 5.4 list: 列出所有

```python
def cmd_mcp_list(args=None):
    # 表格格式输出
    # Name | Transport | Tools | Status
    for name, cfg in servers.items():
        transport = cfg["url"] if "url" in cfg else cfg["command"]
        tools_str = "all" 或 "N selected" 或 "-N excluded"
        status = "enabled" 或 "disabled"
```

### 5.5 test: 测试连接

```python
def cmd_mcp_test(args):
    # 1. 显示传输信息
    # 2. 显示认证信息（脱敏）
    # 3. 测量连接时间
    start = time.monotonic()
    tools = _probe_single_server(name, cfg)
    elapsed_ms = (time.monotonic() - start) * 1000
    # 4. 显示结果
    _success(f"Connected ({elapsed_ms:.0f}ms)")
    _success(f"Tools discovered: {len(tools)}")
```

### 5.6 configure: 配置参数

```python
def cmd_mcp_configure(args):
    # 1. 连接服务器获取工具列表
    all_tools = _probe_single_server(name, cfg)
    # 2. 确定当前选中状态
    # 3. 交互式 checklist
    chosen = curses_checklist(f"Select tools for '{name}'", labels, pre_selected)
    # 4. 更新配置
    if len(chosen) == total:
        server_entry.pop("tools", None)  # 全选 -> 移除过滤器
    else:
        server_entry["tools"]["include"] = chosen_names
    save_config(config)
```

### 5.7 异常解包

```python
def _unwrap_exception_group(exc: BaseException) -> Exception:
    """从 anyio TaskGroup 包装中提取根因异常"""
    while isinstance(exc, BaseExceptionGroup) and exc.exceptions:
        exc = exc.exceptions[0]
    if isinstance(exc, Exception):
        return exc
    return RuntimeError(str(exc))
```

MCP SDK 使用 anyio task groups，会将错误包装在 `BaseExceptionGroup` 中。此函数解包以暴露真实的错误原因（如 "401 Unauthorized"）。

### 5.8 配置文件格式

```yaml
# ~/.hermes/config.yaml
mcp_servers:
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
    enabled: true
    timeout: 120
    connect_timeout: 60

  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "${GITHUB_TOKEN}"  # 环境变量插值
    tools:
      include: ["create_issue", "list_issues"]  # 只启用部分工具

  remote_api:
    url: "https://my-mcp-server.example.com/mcp"
    headers:
      Authorization: "Bearer ${API_KEY}"
    auth: oauth
    oauth:
      scope: "read write"
    tools:
      exclude: ["dangerous_tool"]  # 排除特定工具

  disabled_server:
    command: "some-command"
    enabled: false  # 禁用但保留配置
```

---

## 6. 面试技术亮点

### 6.1 MCP Client 的线程模型设计（asyncio Task per server）

**设计决策**：每个 MCP Server 对应一个独立的 `asyncio.Task`，运行在专用的后台事件循环线程中。

**为什么不用单线程？**
- MCP Server 可能有不同的生命周期（一个断开不影响其他）
- anyio cancel-scope 要求在同一个 Task 中进入和退出
- 并行连接多个 Server 提高启动速度

**为什么不用多进程？**
- 工具注册表是进程内的，多进程需要 IPC
- asyncio Task 已经足够轻量
- 共享内存更高效

**线程模型示意**：
```
主线程 (Agent)
    |
    v
后台 daemon 线程 (mcp-event-loop)
    |
    ├── asyncio.Task-1 (Server A: stdio)
    │       └── 子进程 A
    ├── asyncio.Task-2 (Server B: HTTP)
    │       └── HTTP 连接
    └── asyncio.Task-3 (Server C: stdio)
            └── 子进程 C
```

**面试要点**：
- 解释为什么需要每个 Server 一个 Task（anyio 的 cancel-scope 语义）
- 解释 `_run_on_mcp_loop` 的跨线程调度和中断检查机制
- 讨论 `_lock` 的必要性（Python 3.13+ free-threading）

### 6.2 动态工具刷新的原子性保证

**挑战**：Server 可能在任意时刻发送 `tools/list_changed` 通知，刷新过程中不能让 Agent 调用到已注销但还未重新注册的工具。

**解决方案**：
1. `asyncio.Lock` 防止并发刷新
2. 刷新操作是"先全删再全注册"的模式
3. 注销和注册是同步操作，在事件循环看来是原子的
4. `asyncio.Lock` 是协程锁，不会阻塞事件循环

**面试要点**：
- 讨论"全删全注册"vs"增量更新"的权衡
- 解释为什么使用 `asyncio.Lock` 而不是 `threading.Lock`
- 讨论工具列表变更时 Agent 正在调用工具的竞态场景

### 6.3 安全机制的多层防御

Hermes MCP Client 实现了**深度防御**策略：

| 层级 | 机制 | 目的 |
|------|------|------|
| **1. 环境隔离** | `_build_safe_env` 白名单过滤 | 防止环境变量泄露 |
| **2. 凭证脱敏** | `_sanitize_error` 正则替换 | 防止错误消息泄露凭证 |
| **3. 注入检测** | `_scan_mcp_description` 10 种模式 | 检测工具描述中的 prompt injection |
| **4. 恶意软件检查** | `osv_check.check_package_for_malware` | 检查 npm 包是否已知恶意 |
| **5. 子进程追踪** | `_snapshot_child_pids` + `_stdio_pids` | 防止孤儿进程 |
| **6. 冲突保护** | 内置工具不被 MCP 工具覆盖 | 保护核心功能 |

**面试要点**：
- 解释为什么 prompt injection 检测只记录警告而不阻断
- 讨论环境变量白名单 vs 黑名单的权衡
- 讨论 OSV 检查的局限性（只检查已知漏洞）

### 6.4 OAuth PKCE 的完整实现

**流程**：
```
1. build_oauth_auth() 构建 OAuthClientProvider
2. MCP SDK 自动处理: 发现 -> 动态注册 -> PKCE 授权 -> Token 交换
3. _redirect_handler() 显示授权 URL + 自动打开浏览器
4. 临时 HTTP 服务器监听 localhost:port/callback
5. _wait_for_callback() 等待授权码（5 分钟超时）
6. HermesTokenStorage 持久化 tokens（0o600 权限）
7. 后续连接自动使用缓存的 tokens
```

**面试要点**：
- 解释 PKCE 的 `code_verifier` / `code_challenge` 生成过程（由 SDK 处理）
- 讨论为什么使用 `0o600` 文件权限
- 讨论原子写入的重要性
- 讨论非交互环境的降级策略

### 6.5 遇到的问题和解决方案

#### 问题 1: MCP SDK 的异常包装

**问题**：MCP SDK 使用 anyio task groups，将异常包装在 `BaseExceptionGroup` 中，导致错误消息不透明（如 "unhandled errors in a TaskGroup"）。

**解决方案**：`_unwrap_exception_group` 函数递归解包异常组，提取根因。

#### 问题 2: MCP SDK 版本兼容性

**问题**：不同版本的 MCP SDK 有不同的 API（如 `streamablehttp_client` vs `streamable_http_client`）。

**解决方案**：通过 `_MCP_NEW_HTTP`、`_MCP_HTTP_AVAILABLE`、`_MCP_SAMPLING_TYPES`、`_MCP_NOTIFICATION_TYPES` 等标志位进行特性检测，优雅降级。

#### 问题 3: 子进程清理

**问题**：当事件循环被强制关闭时，stdio 子进程可能变成孤儿进程。

**解决方案**：
1. 启动时记录子进程 PID（`_snapshot_child_pids`）
2. 正常关闭时通过 SDK context manager 清理
3. 异常关闭时 `_kill_orphaned_mcp_children` 强制清理

#### 问题 4: 跨线程中断

**问题**：Agent 主线程需要能够在 MCP 工具调用期间响应用户新消息（中断）。

**解决方案**：`_run_on_mcp_loop` 以 100ms 间隔轮询 `is_interrupted()` 标志，实现"可中断的阻塞等待"。

#### 问题 5: 事件循环关闭时的噪声

**问题**：httpx/httpcore 的异步传输在事件循环关闭后可能触发 `__del__` finalizer，产生 "Event loop is closed" 错误。

**解决方案**：`_mcp_loop_exception_handler` 自定义异常处理器，静默此特定错误。

---

## 附录 A: 关键常量汇总

| 常量 | 值 | 说明 |
|------|-----|------|
| `QUEUE_LIMIT` | 1000 | EventBridge 内存队列最大长度 |
| `POLL_INTERVAL` | 0.2s (200ms) | EventBridge 轮询间隔 |
| `_DEFAULT_TOOL_TIMEOUT` | 120s | 工具调用默认超时 |
| `_DEFAULT_CONNECT_TIMEOUT` | 60s | 初始连接默认超时 |
| `_MAX_RECONNECT_RETRIES` | 5 | 运行时最大重连次数 |
| `_MAX_INITIAL_CONNECT_RETRIES` | 3 | 首次连接最大重试次数 |
| `_MAX_BACKOFF_SECONDS` | 60s | 指数退避最大间隔 |
| OAuth 回调超时 | 300s (5min) | OAuth 授权等待超时 |
| `events_wait` 最大超时 | 300000ms (5min) | 长轮询最大等待时间 |

## 附录 B: 文件结构与行数

| 文件 | 行数 | 主要类/函数 |
|------|------|------------|
| `mcp_serve.py` | 867 | `EventBridge`, `create_mcp_server()`, 10 个 `@mcp.tool()` 函数 |
| `tools/mcp_tool.py` | ~2274 | `SamplingHandler`, `MCPServerTask`, `register_mcp_servers()`, `discover_mcp_tools()` |
| `tools/mcp_oauth.py` | ~483 | `HermesTokenStorage`, `build_oauth_auth()`, `_wait_for_callback()` |
| `hermes_cli/mcp_config.py` | ~717 | `cmd_mcp_add()`, `cmd_mcp_remove()`, `cmd_mcp_list()`, `cmd_mcp_test()`, `cmd_mcp_configure()` |

## 附录 C: MCP 协议消息流

### C.1 Server 初始化（stdio）

```
Client (Hermes)                      MCP Server (子进程)
     |                                      |
     |--- StdioServerParameters --------->  | (启动子进程)
     |                                      |
     |--- initialize ------------------->  |
     |<-- initialize result -------------  |
     |                                      |
     |--- initialized ------------------>  |
     |                                      |
     |--- tools/list ------------------->  |
     |<-- tools/list result -------------  |
     |                                      |
     |--- tools/call {name, args} ------>  |
     |<-- tools/call result -------------  |
     |                                      |
     |--- notifications/tools/list_changed  |
     |<-- (push notification) -----------  |
     |                                      |
     |--- tools/list (refresh) --------->  |
     |<-- tools/list result -------------  |
```

### C.2 Sampling (Server 发起的 LLM 请求)

```
MCP Server                       Hermes Client                    LLM Provider
     |                                  |                              |
     |--- sampling/createMessage -----> |                              |
     |                                  |--- call_llm(messages) -----> |
     |                                  |<-- LLM response -----------  |
     |<-- CreateMessageResult --------  |                              |
     |                                  |                              |
     |--- sampling/createMessage -----> | (带 tool_calls)
     |                                  |--- call_llm(tools) --------> |
     |                                  |<-- tool_calls response ----- |
     |<-- CreateMessageResultWithTools  |                              |
```
