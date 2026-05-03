# 06 - MCP 与 ACP 协议集成深度分析

> 学习顺序：第 6 篇 | 前置知识：Python asyncio、JSON-RPC、网络编程基础

---

## 一、模块概述

### 1.1 设计目标

Hermes Agent 在此代码库中实现了两种协议的完整集成：

**MCP (Model Context Protocol)** -- 两个角色：
- **MCP 客户端** (`tools/mcp_tool.py`, ~2274 行)：连接外部 MCP 服务器，发现其工具并注册到本地工具注册表，让 Hermes Agent 可以像调用内置工具一样调用远程 MCP 服务器暴露的工具。
- **MCP 服务器** (`mcp_serve.py`, ~868 行)：将 Hermes 的消息对话能力（跨 Telegram、Discord、Slack 等平台）暴露为 MCP 工具，使任何 MCP 客户端（Claude Code、Cursor、Codex 等）可以读取对话、发送消息、轮询事件。

**ACP (Agent Client Protocol)** -- 适配器 (`acp_adapter/`, ~7 个文件)：将 Hermes Agent 的完整对话能力通过 ACP 协议暴露给编辑器客户端（如 VS Code 的 AI 插件），支持会话管理、实时事件流、权限审批和斜杠命令。

### 1.2 架构总览

```
  ┌──────────────────────────────────────────────────────────────┐
  │                     Hermes Agent 核心                        │
  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐ │
  │  │  AIAgent     │  │ ToolRegistry │  │  SessionDB (SQLite) │ │
  │  └──────┬──────┘  └──────┬───────┘  └──────────┬──────────┘ │
  └─────────┼───────────────┼───────────────────────┼────────────┘
            │               │                       │
    ┌───────┴───┐    ┌──────┴──────┐         ┌──────┴──────┐
    │ MCP 客户端 │    │ MCP 服务器  │         │ ACP 适配器  │
    │(mcp_tool.py)│   │(mcp_serve.py)│        │(acp_adapter/)│
    └───────┬───┘    └──────┬──────┘         └──────┬──────┘
            │               │                       │
    ┌───────┴───┐    ┌──────┴──────┐         ┌──────┴──────┐
    │ stdio/HTTP │    │   stdio     │         │    stdio    │
    │ 传输层     │    │   传输层    │         │   传输层    │
    └───────┬───┘    └──────┬──────┘         └──────┬──────┘
            │               │                       │
    ┌───────┴───┐    ┌──────┴──────┐         ┌──────┴──────┐
    │ 外部 MCP   │    │ MCP 客户端  │         │  ACP 客户端 │
    │ 服务器     │    │(Claude Code)│         │  (编辑器)   │
    └───────────┘    └─────────────┘         └─────────────┘
```

---

## 二、MCP 客户端详解 (`tools/mcp_tool.py`)

### 2.1 MCP 协议核心概念

MCP (Model Context Protocol) 是 Anthropic 提出的开放协议，定义了 AI 模型与外部工具/数据源交互的标准方式。协议基于 JSON-RPC 2.0，核心概念包括：

| 概念 | 说明 | Hermes 中的实现 |
|------|------|----------------|
| **Tools** | 可被 AI 调用的函数 | `_make_tool_handler()` |
| **Resources** | 可被读取的数据资源 | `_make_list_resources_handler()` / `_make_read_resource_handler()` |
| **Prompts** | 可复用的提示模板 | `_make_list_prompts_handler()` / `_make_get_prompt_handler()` |
| **Sampling** | 服务器请求 LLM 补全 | `SamplingHandler` 类 |

Hermes 的 MCP 客户端实现了完整的四层支持，并且为 Resources 和 Prompts 自动创建了对应的工具（`mcp_{server}_list_resources`、`mcp_{server}_read_resource` 等），使得 LLM Agent 可以通过工具调用来访问这些能力。

### 2.2 stdio 传输 vs HTTP 传输

**stdio 传输** (`_run_stdio`)：
- 通过 `subprocess` 启动子进程，使用 stdin/stdout 管道通信
- 安全性：通过 `_build_safe_env()` 过滤环境变量，只传递 `PATH`、`HOME`、`USER` 等安全变量和用户显式指定的变量
- 命令解析：`_resolve_stdio_command()` 解析命令路径，特别处理 `npx`/`npm`/`node` 的路径查找
- 恶意软件检查：启动前通过 `osv_check` 检查包是否存在于 OSV 恶意数据库

**HTTP/StreamableHTTP 传输** (`_run_http`)：
- 使用 httpx 异步 HTTP 客户端
- 支持 OAuth 2.1 PKCE 认证
- 兼容新旧两版 SDK API：`_MCP_NEW_HTTP` 标志控制使用 `streamable_http_client` 还是 `streamablehttp_client`
- 传输选择逻辑在 `_is_http()`：通过配置中是否有 `url` 字段判断

**选择决策树**：
```
配置中有 "url"? ──→ HTTP/StreamableHTTP 传输
       │
       否
       ↓
配置中有 "command"? ──→ stdio 传输
       │
       否
       ↓
报错：缺少传输配置
```

### 2.3 服务器发现与连接管理

**配置加载** (`_load_mcp_config`)：
```python
config = load_config()
servers = config.get("mcp_servers")
# 支持 ${ENV_VAR} 占位符插值
return {name: _interpolate_env_vars(cfg) for name, cfg in servers.items()}
```

配置文件位置：`~/.hermes/config.yaml` 的 `mcp_servers` 键。支持环境变量插值（`${GITHUB_TOKEN}`）。

**并行发现** (`register_mcp_servers`)：
```python
# 所有服务器并行连接
results = await asyncio.gather(
    *(_discover_one(name, cfg) for name, cfg in new_servers.items()),
    return_exceptions=True,
)
```

关键设计点：
- 幂等性：已连接的服务器不会重复连接
- `enabled: false` 配置项可跳过服务器而不删除配置
- 外层超时 120 秒，内层每个服务器有独立的 `connect_timeout`（默认 60 秒）
- 失败隔离：单个服务器连接失败不影响其他服务器

**MCPServerTask 生命周期**：

每个 MCP 服务器由一个 `MCPServerTask` 管理，运行在专用的 asyncio Task 中：
```
start() → 创建 Task → run() 循环:
    while True:
        try:
            _run_stdio() 或 _run_http()
            break  # 正常退出（shutdown 请求）
        except Exception:
            if 首次连接失败:
                指数退避重试 (最多 3 次)
            elif 运行中断开:
                指数退避重连 (最多 5 次)
```

### 2.4 工具代理机制

**工具注册流程** (`_register_server_tools`)：
```python
schema = _convert_mcp_schema(name, mcp_tool)
registry.register(
    name=tool_name_prefixed,        # "mcp_{server}_{tool}"
    toolset=toolset_name,           # "mcp-{server}"
    schema=schema,
    handler=_make_tool_handler(name, mcp_tool.name, server.tool_timeout),
    check_fn=_make_check_fn(name),  # 检查连接是否存活
    is_async=False,
)
```

**工具名称转换**：MCP 工具名 `read_file` 在服务器 `filesystem` 上注册为 `mcp_filesystem_read_file`。通过 `sanitize_mcp_name_component()` 确保名称安全。

**工具调用代理** (`_make_tool_handler`)：
```python
def _handler(args: dict, **kwargs) -> str:
    async def _call():
        result = await server.session.call_tool(tool_name, arguments=args)
        # 处理 isError、content blocks、structuredContent
        ...
    return _run_on_mcp_loop(_call(), timeout=tool_timeout)
```

关键设计：
- 同步 handler 通过 `run_coroutine_threadsafe()` 将异步调用桥接到 MCP 事件循环
- 支持用户中断：检查 `is_interrupted()`
- 错误清洗：`_sanitize_error()` 使用正则剔除凭据信息

**选择性工具加载**：
```yaml
mcp_servers:
  filesystem:
    tools:
      include: [read_file, write_file]  # 白名单
      exclude: [dangerous_tool]          # 黑名单
      resources: false                   # 禁用 resources 工具
      prompts: false                     # 禁用 prompts 工具
```

- `include` 优先于 `exclude`
- 与内置工具的名称冲突检测：MCP 工具不会覆盖内置工具

### 2.5 自动重连与错误恢复

**指数退避重连**：

两层重试：
1. **初始连接重试**：最多 3 次（`_MAX_INITIAL_CONNECT_RETRIES`），用于处理启动时的瞬态 DNS/网络问题
2. **运行中断线重连**：最多 5 次（`_MAX_RECONNECT_RETRIES`），用于处理运行中连接意外断开

```python
initial_retries += 1
if initial_retries > _MAX_INITIAL_CONNECT_RETRIES:  # 3
    # 放弃，标记错误
    return
await asyncio.sleep(backoff)
backoff = min(backoff * 2, _MAX_BACKOFF_SECONDS)  # 最大 60 秒
```

**动态工具发现**：
```python
case ToolListChangedNotification():
    logger.info("MCP server '%s': received tools/list_changed notification", self.name)
    await self._refresh_tools()
```

当 MCP 服务器发送 `notifications/tools/list_changed` 通知时，自动重新获取工具列表，注销旧工具、注册新工具，并记录变更差异。使用 `asyncio.Lock` 防止并发刷新。

**子进程管理**：
通过 `_snapshot_child_pids()` 跟踪 stdio 子进程 PID，在事件循环关闭后强制杀死未正常退出的孤儿进程。

### 2.6 Sampling 支持

`SamplingHandler` 实现了 MCP 的 `sampling/createMessage` 能力，允许 MCP 服务器请求 LLM 补全：

```python
async def __call__(self, context, params):
    if not self._check_rate_limit():     # 滑动窗口限流
        return self._error(...)
    model = self._resolve_model(...)      # 模型解析
    messages = self._convert_messages(params)  # MCP → OpenAI 格式转换
    response = await asyncio.to_thread(_sync_call)  # 异步化同步 LLM 调用
```

安全控制：
- 滑动窗口限流（`max_rpm`，默认 10 次/分钟）
- 模型白名单（`allowed_models`）
- token 上限（`max_tokens_cap`，默认 4096）
- 工具循环治理（`max_tool_rounds`，默认 5 轮）
- 超时控制（`timeout`，默认 30 秒）

### 2.7 安全机制

**环境变量过滤**：
```python
_SAFE_ENV_KEYS = frozenset({
    "PATH", "HOME", "USER", "LANG", "LC_ALL", "TERM", "SHELL", "TMPDIR",
})
```
只传递安全变量，防止凭据泄露给 MCP 子进程。

**凭据清洗**：
```python
_CREDENTIAL_PATTERN = re.compile(
    r"(?:ghp_[A-Za-z0-9_]{1,255}|sk-[A-Za-z0-9_]{1,255}|Bearer\s+\S+|...)"
)
```
在错误消息返回给 LLM 前，自动替换匹配的凭据为 `[REDACTED]`。

**Prompt 注入扫描**：
```python
_MCP_INJECTION_PATTERNS = [
    (re.compile(r"ignore\s+(all\s+)?previous\s+instructions", re.I),
     "prompt override attempt"),
    (re.compile(r"you\s+are\s+now\s+a", re.I),
     "identity override attempt"),
    ...
]
```
对 MCP 工具描述进行 prompt 注入模式扫描，发现可疑内容时记录警告日志（但不阻断，避免误报）。

---

## 三、MCP 服务器详解 (`mcp_serve.py`)

### 3.1 将 Hermes 对话暴露为 MCP 工具

使用 `FastMCP` 框架，暴露 10 个工具：

| 工具名 | 功能 |
|--------|------|
| `conversations_list` | 列出活跃对话 |
| `conversation_get` | 获取单个对话详情 |
| `messages_read` | 读取消息历史 |
| `attachments_fetch` | 获取消息附件 |
| `events_poll` | 轮询新事件 |
| `events_wait` | 长轮询等待事件 |
| `messages_send` | 发送消息到平台 |
| `channels_list` | 列出可用渠道 |
| `permissions_list_open` | 列出待审批请求 |
| `permissions_respond` | 响应审批请求 |

### 3.2 EventBridge 事件桥

`EventBridge` 是核心组件，用后台线程轮询 SQLite 数据库：

```python
def _poll_loop(self):
    db = _get_session_db()
    while self._running:
        self._poll_once(db)
        time.sleep(POLL_INTERVAL)  # 200ms
```

**mtime 优化**：通过检查 `sessions.json` 和 `state.db` 的文件修改时间，跳过无变化时的昂贵操作，使 200ms 轮询几乎零开销。

**事件队列**：
- 容量限制：`QUEUE_LIMIT = 1000`
- 线程安全：使用 `threading.Lock` 保护
- 唤醒机制：`threading.Event` 用于 `wait_for_event()` 的阻塞等待

### 3.3 数据提取

`_extract_message_content()` 和 `_extract_attachments()` 处理多平台消息格式：
- 多部分内容块（`list` 格式）
- `MEDIA:` 标签提取
- 图片 URL 和文件引用

---

## 四、ACP 适配器详解 (`acp_adapter/`)

### 4.1 ACP 协议概述

ACP (Agent Client Protocol) 是一个面向编辑器集成的协议，允许 IDE/编辑器与 AI Agent 通信。核心特性：
- 基于 stdio 的 JSON-RPC 传输
- 会话生命周期管理（创建、加载、恢复、分叉、取消）
- 实时事件推送（工具调用进度、思考过程、消息流）
- 权限审批流程
- 模型/模式切换

### 4.2 服务器实现 (`server.py`)

`HermesACPAgent` 继承 `acp.Agent`，实现完整的 ACP 协议：

**初始化** (`initialize`)：
```python
return InitializeResponse(
    protocol_version=acp.PROTOCOL_VERSION,
    agent_info=Implementation(name="hermes-agent", version=HERMES_VERSION),
    agent_capabilities=AgentCapabilities(
        load_session=True,
        session_capabilities=SessionCapabilities(
            fork=SessionForkCapabilities(),
            list=SessionListCapabilities(),
            resume=SessionResumeCapabilities(),
        ),
    ),
    auth_methods=auth_methods,  # 检测到的运行时凭据
)
```

**Prompt 核心流程** (`prompt`)：
```python
# 绑定回调
tool_progress_cb = make_tool_progress_cb(conn, session_id, loop, tool_call_ids)
thinking_cb = make_thinking_cb(conn, session_id, loop)
step_cb = make_step_cb(conn, session_id, loop, tool_call_ids)
message_cb = make_message_cb(conn, session_id, loop)
approval_cb = make_approval_callback(conn.request_permission, loop, session_id)
```

关键设计：AIAgent 运行在 `ThreadPoolExecutor` (`max_workers=4`) 的工作线程中，通过 `asyncio.run_coroutine_threadsafe()` 桥接回主事件循环发送 ACP 通知。

**斜杠命令系统**：支持 `/help`、`/model`、`/tools`、`/context`、`/reset`、`/compact`、`/version` 命令，通过 `AvailableCommandsUpdate` 动态广播给客户端。

**ACP MCP 服务器转发**：ACP 客户端可以在 `new_session`/`load_session`/`resume_session` 中传入 `mcp_servers` 列表，服务器自动调用 `register_mcp_servers()` 注册并刷新 Agent 的工具面。

### 4.3 会话管理 (`session.py`)

`SessionManager` 实现双层持久化：

```python
@dataclass
class SessionState:
    session_id: str
    agent: Any           # AIAgent 实例
    cwd: str = "."       # 编辑器工作目录
    model: str = ""
    history: List[Dict]  # 对话历史
    cancel_event: Any    # threading.Event 取消信号
```

**内存 + SQLite 双层**：
- 内存层：`Dict[str, SessionState]`，快速访问
- 持久层：`SessionDB` (`~/.hermes/state.db`)，进程重启后可恢复

**透明恢复** (`_restore`)：
```python
def get_session(self, session_id):
    state = self._sessions.get(session_id)
    if state is not None:
        return state
    return self._restore(session_id)  # 从 DB 恢复
```

**Agent 工厂** (`_make_agent`)：
- 使用 `resolve_runtime_provider()` 获取 LLM 提供商凭据
- 创建 `AIAgent` 实例并设置 `platform="acp"`, `enabled_toolsets=["hermes-acp"]`, `quiet_mode=True`
- 将 `_print_fn` 重定向到 stderr，保持 stdout 用于 JSON-RPC

### 4.4 事件系统 (`events.py`)

四个回调工厂，将 AIAgent 事件桥接到 ACP 通知：

| 工厂 | 信号 | ACP 更新类型 |
|------|------|-------------|
| `make_tool_progress_cb` | `tool.started` | `ToolCallStart` |
| `make_thinking_cb` | 思考文本 | `update_agent_thought_text` |
| `make_step_cb` | 步骤完成 | `ToolCallProgress` (completed) |
| `make_message_cb` | 消息文本 | `update_agent_message_text` |

**线程桥接模式**：
```python
def _send_update(conn, session_id, loop, update):
    future = asyncio.run_coroutine_threadsafe(
        conn.session_update(session_id, update), loop
    )
    future.result(timeout=5)  # 最多等 5 秒
```

**工具调用 ID 管理**：使用 FIFO 队列 (`Deque[str]`) 跟踪同名并行工具调用的 ID，确保 `tool.started` 和 `tool.completed` 正确配对。

### 4.5 权限与认证

**认证** (`auth.py`)：检测 Hermes 配置的运行时 LLM 提供商（如 `openai`、`anthropic`、`openrouter`），在 `initialize` 响应中作为 `auth_methods` 广告。

**权限审批** (`permissions.py`)：
```python
def make_approval_callback(request_permission_fn, loop, session_id, timeout=60.0):
    def _callback(command, description) -> str:
        options = [
            PermissionOption(option_id="allow_once", kind="allow_once"),
            PermissionOption(option_id="allow_always", kind="allow_always"),
            PermissionOption(option_id="deny", kind="reject_once"),
        ]
        future = asyncio.run_coroutine_threadsafe(
            request_permission_fn(session_id, tool_call, options), loop
        )
        response = future.result(timeout=timeout)
```

将终端命令的审批请求转发到 ACP 客户端（编辑器），用户在编辑器 UI 中选择允许/拒绝，结果映射回 Hermes 的审批回调。

### 4.6 工具映射 (`tools.py`)

**工具类型映射**：
```python
TOOL_KIND_MAP = {
    "read_file": "read",
    "write_file": "edit",
    "terminal": "execute",
    "web_search": "fetch",
    "browser_navigate": "fetch",
    "_thinking": "think",
    ...
}
```

**工具事件构建**：`build_tool_start()` 根据工具类型生成富内容：
- `patch` 工具：生成 diff 内容
- `write_file`：显示新文件内容
- `terminal`：显示命令文本
- `read_file`/`search_files`：显示操作描述

`build_tool_complete()` 截断超过 5000 字符的结果，防止 UI 过载。

---

## 五、设计模式

### 5.1 代理模式 (Proxy Pattern)

**MCP 客户端工具代理**：`_make_tool_handler()` 返回的闭包是一个同步代理，将对本地工具的调用转发到远程 MCP 服务器：

```
Agent 调用 mcp_fs_read_file(args)
  → _handler(args)                    # 同步代理
    → _run_on_mcp_loop(_call())       # 桥接到异步事件循环
      → server.session.call_tool()    # 实际的 MCP RPC 调用
```

对 Agent 而言，MCP 工具与内置工具完全透明。

### 5.2 适配器模式 (Adapter Pattern)

**ACP 适配器**是最典型的适配器模式：

```
AIAgent 接口                     ACP 协议接口
─────────────                    ────────────
tool_progress_callback   ←→     ToolCallStart / ToolCallProgress
thinking_callback        ←→     update_agent_thought_text
step_callback            ←→     step completion events
message_callback         ←→     update_agent_message_text
approval_callback        ←→     request_permission
run_conversation()       ←→     prompt()
```

每个 `make_*_cb()` 工厂函数创建一个适配器，将 AIAgent 的回调签名转换为 ACP 的 `session_update()` 调用。

### 5.3 观察者模式 (Observer Pattern)

**动态工具发现**：MCP 服务器通过 `notifications/tools/list_changed` 通知客户端工具列表变化。`_make_message_handler()` 注册为观察者，收到通知后触发 `_refresh_tools()` 刷新。

**EventBridge 事件队列**：后台轮询线程是事件源，`wait_for_event()` 的调用者是观察者。通过 `threading.Event` 实现发布-订阅。

### 5.4 工厂模式 (Factory Pattern)

- `_make_tool_handler()` / `_make_list_resources_handler()` / `_make_check_fn()` 等都是工厂函数，根据服务器名和工具名生成定制的 handler 闭包。
- `make_tool_progress_cb()` / `make_thinking_cb()` 等是 ACP 回调工厂。

### 5.5 单例模式 (Singleton Pattern)

- MCP 事件循环：`_mcp_loop` 和 `_mcp_thread` 是模块级单例
- `ToolRegistry`：全局单例工具注册表
- `SessionManager`：每个 `HermesACPAgent` 实例持有一个

### 5.6 桥接模式 (Bridge Pattern)

**线程-异步桥接**：`_run_on_mcp_loop()` 和 `_send_update()` 都是桥接模式，连接同步工作线程与异步事件循环。

---

## 六、面试八股文

### 6.1 什么是 MCP（Model Context Protocol）？解决了什么问题？

**MCP** 是 Anthropic 于 2024 年底发布的开放协议，全称 Model Context Protocol。它定义了 AI 模型/Agent 与外部工具、数据源之间的标准化通信接口。

**解决的核心问题**：
1. **M×N 集成问题**：之前每个 AI 应用需要为每个工具写专用集成（M 个应用 × N 个工具 = M×N 个集成）。MCP 将其简化为 M+N（每个应用实现一次 MCP 客户端，每个工具实现一次 MCP 服务器）。
2. **工具发现**：协议内置了 `tools/list`、`resources/list`、`prompts/list` 等发现机制，工具可以动态注册和更新。
3. **安全隔离**：通过 stdio 子进程或 HTTP 远程服务隔离工具执行环境。
4. **跨供应商互操作**：基于 JSON-RPC 2.0 的开放标准，不绑定任何特定 AI 提供商。

**协议四层能力**：
- **Tools**：可被 AI 调用的函数（如读文件、搜索）
- **Resources**：可被读取的数据源（如数据库、文件系统）
- **Prompts**：可复用的提示模板
- **Sampling**：服务器可以反向请求 LLM 补全

### 6.2 什么是 ACP（Agent Client Protocol）？

**ACP** 是面向编辑器/IDE 集成的 AI Agent 通信协议，专注于：

1. **会话管理**：创建、恢复、分叉、列举、取消会话
2. **实时事件流**：工具调用进度、思考过程、消息流式输出
3. **权限审批**：编辑器 UI 中的交互式审批
4. **模型/模式切换**：运行时切换 LLM 模型和工作模式

与 MCP 的区别：
- MCP 面向 **工具/数据源集成**（Agent 作为客户端调用外部能力）
- ACP 面向 **编辑器集成**（Agent 作为服务端被编辑器调用）
- MCP 基于请求-响应 + 通知；ACP 增加了会话状态管理和实时推送

### 6.3 stdio vs HTTP 传输的选择

| 维度 | stdio | HTTP/StreamableHTTP |
|------|-------|-------------------|
| **延迟** | 极低（管道 IPC） | 较高（网络往返） |
| **部署** | 本地子进程 | 远程服务 |
| **安全** | 进程隔离，环境变量可控 | 需要认证（OAuth、API Key） |
| **生命周期** | 随客户端进程启动/销毁 | 独立运行 |
| **扩展性** | 单客户端 | 多客户端并发 |
| **适用场景** | 本地工具（文件系统、CLI） | 远程服务（API、数据库） |

**Hermes 的选择**：两种都支持。stdio 用于本地 MCP 服务器（如 `npx @modelcontextprotocol/server-filesystem`），HTTP 用于远程服务。配置中有 `url` 字段则用 HTTP，有 `command` 字段则用 stdio。

### 6.4 JSON-RPC 协议详解

JSON-RPC 2.0 是轻量级远程过程调用协议：

**请求格式**：
```json
{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {"name": "read_file", "arguments": {"path": "/tmp/foo"}},
    "id": 1
}
```

**响应格式**：
```json
{
    "jsonrpc": "2.0",
    "result": {"content": [{"type": "text", "text": "..."}]},
    "id": 1
}
```

**通知**（无 id，不期望响应）：
```json
{
    "jsonrpc": "2.0",
    "method": "notifications/tools/list_changed"
}
```

MCP 和 ACP 都基于 JSON-RPC 2.0，MCP 在其上定义了 `initialize`、`tools/list`、`tools/call`、`resources/list`、`resources/read`、`prompts/list`、`prompts/get`、`sampling/createMessage` 等方法。

### 6.5 SSE（Server-Sent Events）的原理

SSE 是 HTTP 协议上的单向服务器推送机制：

**协议格式**：
```
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"event": "message", "data": "hello"}

data: {"event": "update", "data": "world"}
```

**核心特性**：
- 基于 HTTP 长连接，`Content-Type: text/event-stream`
- 单向：服务器 → 客户端
- 自动重连：浏览器 `EventSource` API 内置重连
- 文本格式：`data:` 前缀，双换行分隔事件

**在 MCP 中**：StreamableHTTP 传输使用 SSE 流式传输 JSON-RPC 消息，实现双向通信（请求通过 HTTP POST，响应/通知通过 SSE 流）。

### 6.6 WebSocket vs SSE vs 长轮询

| 特性 | WebSocket | SSE | 长轮询 |
|------|-----------|-----|--------|
| **方向** | 全双工 | 单向（S→C） | 模拟双向 |
| **协议** | ws:// 独立协议 | HTTP | HTTP |
| **连接** | 持久化升级 | 持久化 | 反复建立/断开 |
| **二进制** | 支持 | 仅文本 | 仅文本 |
| **自动重连** | 需手动实现 | 浏览器内置 | 需手动实现 |
| **代理兼容** | 可能有问题 | 完全兼容 | 完全兼容 |
| **复杂度** | 高 | 低 | 中 |
| **适用场景** | 聊天、游戏、协同编辑 | 通知流、日志流 | 兼容性兜底 |

Hermes 的 ACP 使用 stdio（类比 WebSocket 的持久双向通道），MCP 服务器使用 stdio + StreamableHTTP（HTTP POST + SSE 响应流）。

### 6.7 协议设计的最佳实践

从 Hermes 的实现中可以总结：

1. **渐进式能力协商**：`initialize` 阶段交换 `capabilities`，按需启用功能（如 sampling、fork、resume）
2. **幂等性**：`register_mcp_servers` 对已连接服务器不重复连接
3. **优雅降级**：MCP SDK 未安装时模块成为 no-op，不影响核心功能
4. **安全默认**：环境变量白名单、凭据清洗、prompt 注入扫描
5. **超时治理**：连接超时、工具调用超时、采样超时分层控制
6. **版本兼容**：通过 try/except 和 feature flag 支持多版本 MCP SDK
7. **资源清理**：anyio cancel-scope 要求在同一 Task 中 enter/exit，MCPServerTask 确保了这一点

### 6.8 如何设计一个可扩展的协议系统？

从 Hermes 的架构中提取的原则：

1. **传输层抽象**：MCP 将传输（stdio/HTTP）与协议逻辑分离，`ClientSession` 接收 `read_stream`/`write_stream`，不关心底层传输。
2. **插件化工具注册**：`ToolRegistry` 是统一接口，MCP 工具和内置工具通过相同的 `register()` API 注册。
3. **能力发现**：使用 `tools/list`、`resources/list` 等标准端点，新能力可被自动发现。
4. **通知机制**：`notifications/tools/list_changed` 等通知实现了松耦合的变更传播。
5. **中间件链**：安全检查（恶意软件扫描、注入检测、凭据清洗）作为处理管道的中间件。
6. **向后兼容**：feature flag（`_MCP_HTTP_AVAILABLE`、`_MCP_SAMPLING_TYPES`）确保旧 SDK 版本不会崩溃。

---

## 七、面试官可能问的问题及详细回答

### Q1: 请描述 Hermes MCP 客户端的整体架构

**答**：Hermes MCP 客户端采用"专用事件循环 + 长连接任务"架构。核心是一个运行在守护线程中的 asyncio 事件循环 (`_mcp_loop`)，每个 MCP 服务器由一个 `MCPServerTask` 管理，作为长生命周期的 asyncio Task 运行在该循环上。工具调用通过 `run_coroutine_threadsafe()` 从 Agent 的工作线程调度到 MCP 事件循环。这种设计确保了 anyio cancel-scope 在同一 Task 中创建和销毁，同时避免阻塞 Agent 的主执行流。

### Q2: MCP 工具调用的完整链路是什么？

**答**：
1. Agent 决定调用 `mcp_fs_read_file`
2. 调用 `_handler(args)` — 这是 `_make_tool_handler()` 生成的同步闭包
3. handler 构造异步协程 `_call()`，调用 `server.session.call_tool("read_file", arguments=args)`
4. 通过 `_run_on_mcp_loop()` 将协程调度到 MCP 事件循环
5. MCP SDK 通过 stdio/HTTP 传输发送 JSON-RPC `tools/call` 请求
6. MCP 服务器处理请求并返回结果
7. SDK 解析响应，handler 提取 `content` blocks 的文本
8. 结果经过 `_sanitize_error()` 清洗后返回 JSON 字符串给 Agent

### Q3: 如何处理 MCP 服务器的连接断开？

**答**：`MCPServerTask.run()` 实现了两层重连：
- **首次连接失败**：最多 3 次重试，指数退避（1s → 2s → 4s），处理启动时的瞬态故障
- **运行中断线**：最多 5 次重连，指数退避（最大 60s），处理网络波动
- 如果在退避等待期间收到 shutdown 信号，立即退出
- 超过重试次数后放弃并记录警告

### Q4: MCP 的 Sampling 机制是什么？Hermes 如何实现？

**答**：Sampling 允许 MCP 服务器反过来请求客户端的 LLM 进行补全。这是"反向调用"——通常客户端调用服务器的工具，但 Sampling 让服务器请求客户端的 LLM。

Hermes 通过 `SamplingHandler` 实现：
- 作为 `sampling_callback` 传递给 `ClientSession`
- 将 MCP 的 `SamplingMessage` 转换为 OpenAI 格式
- 通过 `asyncio.to_thread()` 调用 LLM（不阻塞事件循环）
- 返回 `CreateMessageResult` 或 `CreateMessageResultWithTools`
- 安全控制：限流、模型白名单、token 上限、工具循环治理

### Q5: ACP 和 MCP 有什么本质区别？

**答**：
- **视角不同**：MCP 中 Agent 是客户端（调用外部工具），ACP 中 Agent 是服务端（被编辑器调用）
- **关注点不同**：MCP 关注工具/数据源集成，ACP 关注编辑器 UI 集成
- **状态管理**：MCP 是无状态请求-响应（+通知），ACP 有完整的会话状态机（创建→活跃→暂停→恢复）
- **实时性**：ACP 有丰富的实时事件流（工具进度、思考过程、消息流），MCP 主要是请求-响应
- **交互**：ACP 支持权限审批等交互式操作，MCP 的 Sampling 是最接近交互的能力

### Q6: Hermes 的线程-异步桥接是如何工作的？

**答**：Hermes 存在两个执行上下文：
1. **Agent 工作线程**（同步）：运行 `AIAgent.run_conversation()`
2. **MCP/ACP 事件循环**（异步）：管理网络连接

桥接方式：
- Agent → MCP：`_run_on_mcp_loop()` 使用 `asyncio.run_coroutine_threadsafe()` + 轮询 `future.result(timeout=0.1)` 支持用户中断
- Agent → ACP：`_send_update()` 使用 `asyncio.run_coroutine_threadsafe()` + `future.result(timeout=5)` 防火即忘
- ACP prompt：`loop.run_in_executor(_executor, _run_agent)` 将同步 Agent 调用提交到线程池

### Q7: 如何保证 MCP 子进程的安全性？

**答**：Hermes 实现了多层安全措施：
1. **环境变量白名单**：`_build_safe_env()` 只传递 PATH/HOME/USER 等安全变量
2. **恶意软件检查**：启动前通过 OSV 数据库检查 npm 包
3. **凭据清洗**：错误消息中的 token/key 自动替换为 `[REDACTED]`
4. **Prompt 注入扫描**：检测工具描述中的注入模式
5. **子进程 PID 跟踪**：确保进程退出时清理孤儿子进程
6. **名称冲突防护**：MCP 工具不会覆盖内置工具

### Q8: ACP 会话是如何持久化的？

**答**：`SessionManager` 实现了内存 + SQLite 双层持久化：
- 内存层：`Dict[str, SessionState]`，快速访问
- 持久层：`SessionDB` (`~/.hermes/state.db`)
- 每次 prompt 完成、模型切换、斜杠命令后调用 `save_session()`
- 进程重启后 `get_session()` 自动从 DB 透明恢复
- 恢复时重建 `AIAgent` 实例，恢复 provider/cwd/model 等配置

### Q9: MCP 的动态工具发现是如何实现的？

**答**：通过 MCP 的 `notifications/tools/list_changed` 通知机制：
1. `_make_message_handler()` 注册为 `ClientSession` 的消息处理器
2. 收到 `ToolListChangedNotification` 时触发 `_refresh_tools()`
3. `_refresh_tools()` 获取新工具列表，注销旧工具，注册新工具
4. 使用 `asyncio.Lock` 防止并发刷新
5. 记录变更差异（added/removed）用于审计

### Q10: EventBridge 为什么用轮询而不是文件监听？

**答**：EventBridge 轮询 SQLite 数据库（200ms 间隔），而不是使用 `inotify`/`FSEvents` 等文件系统通知，原因：
1. **跨平台**：SQLite 的 WAL 模式在不同 OS 上行为一致，文件通知 API 不一致
2. **mtime 优化**：检查文件修改时间仅需 ~1μs，无变化时跳过所有工作
3. **可靠性**：文件系统通知可能丢失事件，轮询保证不遗漏
4. **简单性**：不需要管理 watcher 生命周期和回调队列
5. **足够实时**：200ms 间隔对消息桥接场景足够（人类感知延迟约 100ms）

### Q11: ACP 中工具调用的开始和完成事件是如何配对的？

**答**：使用 `tool_call_ids: Dict[str, Deque[str]]` 实现 FIFO 队列配对：
- `tool.started` 事件时：生成 `tc_id`，追加到该工具名的队列尾部
- `step_callback` 触发时：从队列头部弹出 `tc_id`，构建 `ToolCallProgress` 完成事件
- 队列结构确保即使同名工具并行调用，配对也不会混乱
- 如果队列为空（不应该发生），优雅降级

### Q12: `_run_on_mcp_loop` 为什么要轮询而不是直接 await？

**答**：`_run_on_mcp_loop()` 从同步线程调用，不能直接 await。它使用 `future.result(timeout=0.1)` 短间隔轮询，核心原因是**支持用户中断**：

```python
while True:
    if is_interrupted():       # 检查用户是否发送了新消息
        future.cancel()
        raise InterruptedError("User sent a new message")
    return future.result(timeout=0.1)  # 短间隔轮询
```

如果用 `future.result()` 无超时阻塞，Agent 线程无法响应用户中断。100ms 的轮询间隔确保了中断响应的及时性。

### Q13: ACP 的斜杠命令是如何广播给客户端的？

**答**：通过 `AvailableCommandsUpdate` 会话更新：
1. 在 `new_session`/`load_session`/`resume_session` 响应发送后
2. 调用 `_schedule_available_commands_update()`
3. 使用 `loop.call_soon(asyncio.create_task, ...)` 确保在响应之后发送
4. 通过 `conn.session_update()` 推送 `AvailableCommandsUpdate`
5. 客户端在 UI 中显示可用的斜杠命令

### Q14: 如何理解 Hermes 的"工具集"(toolset) 概念？

**答**：工具集是工具的逻辑分组：
- 内置工具集：如 `hermes-core`、`hermes-browser`
- MCP 工具集：如 `mcp-filesystem`、`mcp-github`（格式 `mcp-{server_name}`）
- ACP 工具集：`hermes-acp`

通过 `enabled_toolsets`/`disabled_toolsets` 配置控制哪些工具集对 Agent 可见。MCP 服务器还支持 `tools.include`/`tools.exclude` 细粒度过滤。注册表中每个工具有 `toolset` 属性，支持按工具集查询和管理。

### Q15: 这个系统有哪些可以改进的地方？

**答**（展示批判性思维）：
1. **MCP 客户端**：可以增加连接池和负载均衡，支持同一 MCP 服务器的多个实例
2. **EventBridge**：可以使用 SQLite 的 WAL 模式 notification 替代 mtime 检查
3. **ACP 会话**：可以实现会话过期和自动清理策略（当前是全量保留）
4. **Sampling**：可以增加缓存层，对相同请求返回缓存结果以减少 LLM 调用
5. **安全**：prompt 注入扫描目前只记录警告，可以增加阻断模式
6. **可观测性**：可以增加 OpenTelemetry traces，追踪跨 MCP/ACP 的工具调用链路
7. **类型安全**：大量使用 `dict` 和 `Any`，可以引入更强的类型定义

---

## 八、学习路径建议

### 初级（理解基础）
1. 阅读 JSON-RPC 2.0 规范
2. 理解 asyncio 的事件循环、Task、Future 概念
3. 学习 Python 的 `threading` 与 `asyncio` 交互方式
4. 阅读 MCP 官方文档 (modelcontextprotocol.io)

### 中级（深入实现）
1. 研读 `tools/mcp_tool.py` 的 `MCPServerTask` 生命周期管理
2. 理解 `run_coroutine_threadsafe()` 的使用场景和陷阱
3. 学习 anyio cancel-scope 的工作原理（为什么必须在同一 Task 中 enter/exit）
4. 阅读 `acp_adapter/server.py` 理解协议适配的完整流程

### 高级（架构设计）
1. 研究 MCP 的 Sampling 协议及其安全模型
2. 理解 StreamableHTTP 传输的 SSE + POST 双向通信机制
3. 设计自己的协议扩展（如自定义 MCP 通知类型）
4. 对比 MCP/ACP 与 LSP (Language Server Protocol) 的架构异同
