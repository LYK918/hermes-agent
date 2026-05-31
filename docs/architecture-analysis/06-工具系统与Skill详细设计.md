# 06 - 工具系统与 Skill 详细设计

## 1. 工具系统架构

### 1.1 ToolRegistry 单例模式

工具系统的核心是 `tools/registry.py` 中的 `ToolRegistry` 类，在模块级别创建全局单例：

```python
registry = ToolRegistry()
```

**设计要点：**

- **线程安全**：内部使用 `threading.RLock()` 保护所有读写操作。RLock（可重入锁）允许同一线程多次获取锁，避免递归调用死锁。
- **快照隔离**：提供 `_snapshot_state()` 方法，在锁内复制一份完整的工具条目和 toolset 检查函数的快照，确保读者（如 `get_definitions()`）拿到的是一致的视图，不会被并发的 MCP 动态刷新所干扰。
- **影子保护（Shadow Protection）**：`register()` 方法在注册时检查是否会发生跨 toolset 的覆盖。如果一个工具已经注册在 toolset A 中，新注册请求试图将其放入 toolset B，会被**拒绝**并记录错误日志。唯一的例外是 MCP-to-MCP 的覆盖（两个 toolset 名都以 `mcp-` 开头），这是合法的服务器刷新行为。

```python
# 影子保护逻辑
if existing and existing.toolset != toolset:
    both_mcp = existing.toolset.startswith("mcp-") and toolset.startswith("mcp-")
    if both_mcp:
        logger.debug("MCP toolset overwrite: allowed")
    else:
        logger.error("Tool registration REJECTED: would shadow existing tool")
        return
```

### 1.2 ToolEntry 数据结构

使用 `__slots__` 优化内存占用，每个工具注册为一个 `ToolEntry`：

```python
class ToolEntry:
    __slots__ = (
        "name",           # 工具名称（如 "web_search"）
        "toolset",        # 所属工具集（如 "web"、"mcp-github"）
        "schema",         # OpenAI 函数调用格式的 JSON Schema
        "handler",        # 实际执行函数（同步或异步）
        "check_fn",       # 可用性检查函数（返回 bool）
        "requires_env",   # 依赖的环境变量列表
        "is_async",       # 是否为异步处理器
        "description",    # 描述文本
        "emoji",          # UI 显示用的 emoji
        "max_result_size_chars,  # 结果大小上限
    )
```

### 1.3 自注册机制

每个工具文件（如 `web_tools.py`、`terminal_tool.py`）在**模块导入时**自动调用 `registry.register()`：

```python
# web_tools.py 模块级别
registry.register(
    name="web_search",
    toolset="web",
    schema={...},         # OpenAI 函数调用格式
    handler=web_search_tool,
    check_fn=lambda: _is_backend_available(_get_backend()),
    requires_env=["FIRECRAWL_API_KEY", "TAVILY_API_KEY", ...],
    is_async=True,
    description="Search the web for information",
    emoji="🔍",
)
```

这种自注册模式的关键优势：
- **零配置添加**：只需在 `tools/` 目录下新建一个 `.py` 文件并调用 `register()`，无需修改任何中央配置。
- **关注点内聚**：工具的 schema、handler、可用性检查定义在同一文件中。
- **循环导入安全**：导入链为 `tools/registry.py`（无工具文件导入）-> `tools/*.py`（导入 registry）-> `model_tools.py`（导入所有工具模块）。

### 1.4 AST 自动发现

`discover_builtin_tools()` 使用 Python AST 模块静态分析工具文件，而非盲目导入所有 `.py` 文件：

```python
def _module_registers_tools(module_path: Path) -> bool:
    """只在模块顶层包含 registry.register(...) 调用时返回 True"""
    source = module_path.read_text(encoding="utf-8")
    tree = ast.parse(source, filename=str(module_path))
    return any(_is_registry_register_call(stmt) for stmt in tree.body)
```

**AST 节点匹配逻辑**：
- 检查节点是否为 `ast.Expr` 包裹的 `ast.Call`
- 函数调用的 `func` 必须是 `ast.Attribute`（属性访问），且 `attr == "register"`
- 属性的目标必须是 `ast.Name`，且 `id == "registry"`

这确保了只有真正注册工具的模块才会被导入，避免辅助模块（如工具函数库）被意外触发。

---

## 2. 工具调度

### 2.1 dispatch() 执行流程

`model_tools.py` 中的 `handle_function_call()` 是工具调度的主入口，执行流程如下：

```
1. coerce_tool_args() — 参数类型强制转换（LLM 经常返回 "42" 而非 42）
2. 检查是否为 _AGENT_LOOP_TOOLS（todo, memory, session_search, delegate_task）
   → 这些工具由 agent loop 直接拦截，返回 stub 错误
3. 触发 pre_tool_call 插件钩子（可返回 block 指令阻止执行）
4. 通知 read-loop tracker（非 read/search 工具调用时重置连续读取计数）
5. registry.dispatch() — 实际执行
6. 触发 post_tool_call 插件钩子
7. 返回 JSON 字符串结果
```

### 2.2 同步/异步桥接 (_run_async 三种场景)

`_run_async()` 是整个工具系统同步->异步桥接的**唯一真相来源**，处理三种场景：

**场景 1：当前线程已有运行中的事件循环**（如 gateway 的 async 栈、Atropos 的 RL 环境）
```python
if loop and loop.is_running():
    # 在新线程中运行 asyncio.run()，避免与已有循环冲突
    with concurrent.futures.ThreadPoolExecutor(max_workers=1) as pool:
        future = pool.submit(asyncio.run, coro)
        return future.result(timeout=300)
```

**场景 2：当前在工作线程**（如 delegate_task 的 ThreadPoolExecutor 线程）
```python
if threading.current_thread() is not threading.main_thread():
    # 使用线程本地的持久事件循环
    worker_loop = _get_worker_loop()
    return worker_loop.run_until_complete(coro)
```

**场景 3：主 CLI 线程**（最常见路径）
```python
# 使用全局持久事件循环，避免 asyncio.run() 的 create-and-destroy 生命周期
tool_loop = _get_tool_loop()
return tool_loop.run_until_complete(coro)
```

**为什么不用 `asyncio.run()`？**
`asyncio.run()` 每次创建新循环、执行协程、然后**关闭**循环。但缓存的 httpx/AsyncOpenAI 客户端仍然绑定到已关闭的循环，在垃圾回收时会触发 "Event loop is closed" 错误。持久循环方案避免了这个问题。

### 2.3 并行执行

代码执行工具（`execute_code`）内部使用 `ThreadPoolExecutor` 实现工具调用的并行执行。系统维护一个 `_PARALLEL_SAFE_TOOLS` 集合，标记哪些工具可以安全并行调用（如 `web_search`、`read_file`），哪些必须串行（如 `terminal`，因为可能修改共享状态）。

### 2.4 结果大小控制

每个工具可以通过 `max_result_size_chars` 参数声明结果大小上限。`registry.get_max_result_size()` 方法按优先级查找：
1. 工具自身声明的上限
2. 调用方传入的默认值
3. 全局配置 `DEFAULT_RESULT_SIZE_CHARS`

超大结果会被截断，防止撑爆 LLM 的上下文窗口。

---

## 3. 终端执行后端

### 3.1 BaseEnvironment 统一接口

`tools/environments/base.py` 定义了所有后端的抽象基类：

```python
class BaseEnvironment(ABC):
    def __init__(self, cwd: str, timeout: int, env: dict = None):
        self._session_id = uuid.uuid4().hex[:12]
        self._snapshot_path = f"/tmp/hermes-snap-{self._session_id}.sh"
        self._cwd_file = f"/tmp/hermes-cwd-{self._session_id}.txt"
        self._cwd_marker = _cwd_marker(self._session_id)

    def _run_bash(self, cmd_string, *, login=False, timeout=120, stdin_data=None) -> ProcessHandle:
        raise NotImplementedError  # 子类必须实现

    @abstractmethod
    def cleanup(self): ...  # 释放后端资源

    def execute(self, command, cwd="", *, timeout=None, stdin_data=None) -> dict:
        # 统一执行流程：prepare → wrap → spawn → wait → update_cwd
```

**统一执行流程（execute 方法）：**
1. `_before_execute()` — 钩子，远程后端用于触发文件同步
2. `_prepare_command()` — sudo 命令转换（添加 `-S` 标志，注入密码到 stdin）
3. `_wrap_command()` — 包装为完整的 bash 脚本（source snapshot → cd → 执行 → 重新导出 env → 输出 CWD 标记）
4. `_run_bash()` — 子类实现的实际进程创建
5. `_wait_for_process()` — 基于轮询的等待，处理中断、超时、活动心跳
6. `_update_cwd()` — 从输出中提取 CWD 标记，更新工作目录状态

### 3.2 会话快照机制

会话快照是终端系统的核心创新，解决了"每次 spawn 新进程丢失环境状态"的问题：

**初始化时（init_session）：**
```bash
export -p > /tmp/hermes-snap-{session_id}.sh       # 导出所有环境变量
declare -f | grep -vE '^_[^_]' >> /tmp/hermes-snap-{session_id}.sh  # 导出 shell 函数
alias -p >> /tmp/hermes-snap-{session_id}.sh        # 导出别名
echo 'shopt -s expand_aliases' >> ...               # 启用别名展开
echo 'set +e' >> ...                                # 关闭 errexit
echo 'set +u' >> ...                                # 关闭 nounset
```

**每次命令执行时（_wrap_command）：**
```bash
source /tmp/hermes-snap-{session_id}.sh 2>/dev/null || true  # 恢复环境
cd {cwd} || exit 126                                          # 切换目录
eval '{escaped_command}'                                      # 执行命令
__hermes_ec=$?                                                # 捕获退出码
export -p > /tmp/hermes-snap-{session_id}.sh 2>/dev/null || true  # 重新保存环境
pwd -P > /tmp/hermes-cwd-{session_id}.txt                     # 写入 CWD 文件
printf '\n__HERMES_CWD_{session_id}__%s__HERMES_CWD_{session_id}__\n' "$(pwd -P)"  # 输出 CWD 标记
exit $__hermes_ec
```

**CWD 持久化的两种方式：**
- **本地后端**：直接读取 `_cwd_file`（文件 I/O，无网络开销）
- **远程后端**（Docker/SSH/Modal）：从 stdout 中解析 `__HERMES_CWD_{session_id}__` 标记

### 3.3 ProcessHandle 协议

```python
class ProcessHandle(Protocol):
    def poll(self) -> int | None: ...
    def kill(self) -> None: ...
    def wait(self, timeout: float | None = None) -> int: ...
    @property
    def stdout(self) -> IO[str] | None: ...
    @property
    def returncode(self) -> int | None: ...
```

`subprocess.Popen` 天然满足此协议。对于 SDK 后端（Modal、Daytona），提供 `_ThreadedProcessHandle` 适配器，将阻塞的 `exec_fn()` 包装在后台线程中，通过 `os.pipe()` 模拟 stdout 流。

### 3.4 六种后端

| 后端 | 文件 | 特点 |
|------|------|------|
| **Local** | `local.py` | 直接在宿主机执行，最快。Spawn-per-call 模式，每次命令创建新 bash 进程。通过 `_make_run_env()` 过滤敏感环境变量（API Key 等），防止泄露到子进程。 |
| **Docker** | `docker.py` | 安全加固容器：`--cap-drop ALL` + 最小能力回加（DAC_OVERRIDE, CHOWN, FOWNER）、`--security-opt no-new-privileges`、`--pids-limit 256`、size-limited tmpfs。支持持久化文件系统（bind mount）和非持久化（tmpfs）。 |
| **Modal** | `modal.py` | 云端沙箱，通过 Modal SDK 创建。支持自定义 Dockerfile、GPU 资源。使用 `_ThreadedProcessHandle` 适配。 |
| **SSH** | `ssh.py` | 通过 SSH 连接远程机器执行。支持持久化 shell 会话和文件同步（`file_sync.py`）。 |
| **Singularity** | `singularity.py` | HPC 容器运行时，适用于超算环境。支持 overlay 文件系统和 SIF 缓存。 |
| **Daytona** | `daytona.py` | 云开发环境，通过 Daytona API 创建沙箱。 |

**Docker 后端安全细节：**
```python
_SECURITY_ARGS = [
    "--cap-drop", "ALL",
    "--cap-add", "DAC_OVERRIDE",   # root 写入 bind-mounted 目录
    "--cap-add", "CHOWN",          # 包管理器需要
    "--cap-add", "FOWNER",         # 包管理器需要
    "--security-opt", "no-new-privileges",
    "--pids-limit", "256",
    "--tmpfs", "/tmp:rw,nosuid,size=512m",
    "--tmpfs", "/var/tmp:rw,noexec,nosuid,size=256m",
    "--tmpfs", "/run:rw,noexec,nosuid,size=64m",
]
```

---

## 4. 浏览器自动化

### 4.1 架构概览

浏览器工具通过 `agent-browser` CLI 统一封装，支持多种后端：

```
browser_tool.py
├── CloudBrowserProvider (抽象基类)
│   ├── BrowserbaseProvider  — Browserbase 云浏览器
│   ├── BrowserUseProvider   — Browser Use 云浏览器（Nous 订阅者默认）
│   └── FirecrawlProvider    — Firecrawl 云浏览器
└── 本地 Chromium 后端
    └── agent-browser CLI（headless Chromium）
```

### 4.2 云后端选择逻辑

```python
def _get_cloud_provider() -> Optional[CloudBrowserProvider]:
    # 1. 读取 config["browser"]["cloud_provider"]
    #    "local" → 返回 None（强制本地模式）
    #    "browserbase"/"browser-use"/"firecrawl" → 实例化对应 Provider
    # 2. 未配置时自动探测：
    #    优先 Browser Use（Nous 网关或直接 API Key）
    #    回退 Browserbase（直接凭证）
    # 3. 都未配置 → 返回 None（本地 Chromium 模式）
```

### 4.3 SSRF 防护

云浏览器后端（Browserbase、BrowserUse）运行在远程机器上，Agent 可以通过浏览器访问该机器的内部网络。SSRF 防护阻止导航到私有/内部地址：

```python
def _allow_private_urls() -> bool:
    # 读取 config["browser"]["allow_private_urls"]
    # 默认 False（SSRF 防护激活）

def _is_local_backend() -> bool:
    # 本地后端不需要 SSRF 防护（用户已有完全的终端和网络访问权限）
    return _is_camofox_mode() or _get_cloud_provider() is None
```

`is_safe_url()` 函数检查目标 URL 是否指向私有 IP 段（10.x、172.16-31.x、192.168.x、127.x、::1 等），仅在云后端模式下生效。

### 4.4 CDP 覆盖

通过 `BROWSER_CDP_URL` 环境变量，可以连接到用户指定的 Chrome DevTools Protocol 端点，支持：
- 完整 WebSocket 端点：`ws://host:port/devtools/browser/...`
- HTTP 发现端点：`http://host:port`（自动获取 `/json/version` 中的 `webSocketDebuggerUrl`）
- 裸 WebSocket 地址：`ws://host:port`

---

## 5. 审批系统 (approval.py)

### 5.1 危险命令模式库

`DANGEROUS_PATTERNS` 包含 40+ 条正则规则，覆盖以下类别：

| 类别 | 示例模式 | 描述 |
|------|----------|------|
| **文件系统破坏** | `rm -rf /`、`mkfs`、`dd if=` | 递归删除、格式化、磁盘覆写 |
| **权限篡改** | `chmod 777`、`chown -R root` | 不安全权限、递归所有权变更 |
| **SQL 注入** | `DROP TABLE`、`DELETE FROM`（无 WHERE）、`TRUNCATE` | 数据库破坏操作 |
| **系统服务** | `systemctl stop/restart/disable` | 系统服务操控 |
| **进程攻击** | `kill -9 -1`、fork bomb | 杀死所有进程、资源耗尽 |
| **远程代码执行** | `curl|sh`、`bash <(curl)`、`python -c`、heredoc 执行 | 管道执行、脚本注入 |
| **Git 破坏** | `git reset --hard`、`git push --force`、`git clean -f` | 丢失未提交工作、重写远程历史 |
| **系统配置** | `>/etc/`、`sed -i /etc/`、`cp/mv /etc/` | 覆写系统配置文件 |
| **自我终止** | `pkill hermes`、`kill $(pgrep -f hermes)`、`hermes gateway stop` | 杀死 Agent 自身或 Gateway |
| **Shell 注入** | `bash -c`、`python -e`、`node -e` | 通过标志执行任意代码 |

### 5.2 命令规范化与反混淆

在模式匹配之前，命令经过三层规范化处理：

```python
def _normalize_command_for_detection(command: str) -> str:
    # 1. 剥离 ANSI 转义序列（完整 ECMA-48：CSI、OSC、DCS、8-bit C1 等）
    command = strip_ansi(command)
    # 2. 剥离 null 字节
    command = command.replace('\x00', '')
    # 3. Unicode NFKC 规范化（全角拉丁→半角、半角片假名→全角等）
    command = unicodedata.normalize('NFKC', command)
    return command
```

这三层处理可以防御以下混淆攻击：
- **ANSI 注入**：在 `rm` 命令中间插入不可见的 ANSI 转义序列，绕过正则匹配
- **Null 字节注入**：在关键词中插入 `\x00`，使正则匹配失败
- **Unicode 全角**：使用全角字母（如 `ｒｍ`）绕过 ASCII 正则

### 5.3 审批队列阻塞

Gateway 模式下，审批使用队列化阻塞机制：

```python
class _ApprovalEntry:
    __slots__ = ("event", "data", "result")
    def __init__(self, data: dict):
        self.event = threading.Event()  # 阻塞等待
        self.data = data                # 命令信息
        self.result: Optional[str] = None  # 用户选择
```

**流程：**
1. Agent 线程创建 `_ApprovalEntry`，加入 session 队列
2. 通过 `notify_cb` 回调通知 Gateway（sync→async 桥接）
3. Agent 线程调用 `entry.event.wait(timeout=300)` 阻塞
4. 用户通过 `/approve` 或 `/deny` 命令响应
5. Gateway 调用 `resolve_gateway_approval()` 设置 result 并 `event.set()`
6. Agent 线程解除阻塞，继续执行或返回 BLOCKED

支持 `/approve all` 批量解决同一 session 中的所有待审批项。

### 5.4 智能审批 (Smart Approval)

当 `approvals.mode` 配置为 `smart` 时，系统使用辅助 LLM 评估命令风险：

```python
def _smart_approve(command: str, description: str) -> str:
    prompt = f"""You are a security reviewer for an AI coding agent.
Command: {command}
Flagged reason: {description}

Rules:
- APPROVE if the command is clearly safe
- DENY if the command could genuinely damage the system
- ESCALATE if you're uncertain

Respond with exactly one word: APPROVE, DENY, or ESCALATE"""

    response = client.chat.completions.create(model=model, messages=[...], temperature=0)
    answer = response.choices[0].message.content.strip().upper()

    if "APPROVE" in answer: return "approve"
    elif "DENY" in answer: return "deny"
    else: return "escalate"  # 不确定→回退到人工审批
```

**设计灵感**：来自 OpenAI Codex 的 Smart Approvals guardian subagent（openai/codex#13860）。

**三种审批模式：**
- `manual`：所有危险命令都需要人工确认（默认）
- `smart`：先用 LLM 评估，只在不确定时才请求人工
- `off`：跳过所有审批（YOLO 模式）

### 5.5 YOLO 模式

```python
# CLI 级别：--yolo 标志设置环境变量
os.getenv("HERMES_YOLO_MODE")

# Gateway 级别：/yolo 命令（session-scoped）
is_current_session_yolo_enabled()
```

YOLO 模式跳过所有审批检查，包括 tirith 安全扫描和危险命令检测。

### 5.6 持久化允许列表

用户选择 "always" 审批时，模式被保存到 `config.yaml` 的 `command_allowlist` 中：

```python
def save_permanent_allowlist(patterns: set):
    config = load_config()
    config["command_allowlist"] = list(patterns)
    save_config(config)
```

下次启动时通过 `load_permanent_allowlist()` 自动加载。

---

## 6. Skill 系统

### 6.1 SKILL.md 格式

每个 Skill 是一个包含 `SKILL.md` 文件的目录，使用 YAML frontmatter 声明元数据：

```yaml
---
name: skill-name              # 必需，最大 64 字符
description: Brief description # 必需，最大 1024 字符
version: 1.0.0                # 可选
license: MIT                  # 可选（agentskills.io 兼容）
platforms: [macos]            # 可选 — 限制特定 OS 平台
                              #   有效值: macos, linux, windows
                              #   省略则在所有平台加载
prerequisites:                # 可选 — 运行时依赖
  env_vars: [API_KEY]         #   会自动标准化为 required_environment_variables
  commands: [curl, jq]        #   命令检查仅为建议性
setup:                        # 可选 — 安装向导
  help: "Visit https://..."   #   帮助文本
  collect_secrets:            #   自动收集密钥
    - env_var: API_KEY
      prompt: "Enter your API key"
      provider_url: "https://..."
metadata:                     # 可选，任意键值对（agentskills.io 兼容）
  hermes:
    tags: [fine-tuning, llm]
    related_skills: [peft, lora]
---

# Skill Title

Full instructions and content here...
```

**agentskills.io 兼容性**：
- `name`、`description`、`version`、`license`、`compatibility`、`metadata` 字段与 agentskills.io 规范兼容
- `platforms` 字段为 Hermes 扩展，用于跨平台过滤

**平台过滤逻辑**：
```python
_PLATFORM_MAP = {"macos": "darwin", "linux": "linux", "windows": "win32"}

def skill_matches_platform(frontmatter: Dict) -> bool:
    platforms = frontmatter.get("platforms")
    if not platforms:
        return True  # 未指定→所有平台
    current = sys.platform  # "darwin", "linux", "win32"
    return any(current.startswith(_PLATFORM_MAP.get(p, p)) for p in platforms)
```

### 6.2 三层渐进式披露

Skill 系统采用受 Anthropic Claude Skills 启发的渐进式披露架构，最小化 token 消耗：

**第 1 层：元数据（skills_list）**
- 返回所有 Skill 的 `name`（≤64 字符）和 `description`（≤1024 字符）
- Token 开销极低，适合在上下文窗口中列出所有可用 Skill

**第 2 层：完整指令（skill_view，不带 file_path）**
- 加载 SKILL.md 的完整内容
- 包含详细的操作步骤、示例、注意事项

**第 3 层：链接文件（skill_view，带 file_path）**
- 按需加载 `references/`、`templates/`、`scripts/`、`assets/` 中的文件
- 支持路径如 `references/api.md`、`templates/template.md`

**目录结构示例**：
```
skills/
├── my-skill/
│   ├── SKILL.md           # 主指令（必需）
│   ├── references/        # 参考文档
│   │   ├── api.md
│   │   └── examples.md
│   ├── templates/         # 输出模板
│   │   └── template.md
│   ├── scripts/           # 辅助脚本
│   └── assets/            # 补充文件
└── category/              # 分类文件夹
    └── another-skill/
        └── SKILL.md
```

### 6.3 Skills Hub

`tools/skills_hub.py` 实现了 Skills Hub 系统，用于从远程仓库发现、下载和安装 Skill。

#### 6.3.1 SkillSource 抽象基类

```python
class SkillSource(ABC):
    @abstractmethod
    def search(self, query: str, limit: int = 10) -> List[SkillMeta]: ...
    @abstractmethod
    def fetch(self, identifier: str) -> Optional[SkillBundle]: ...
    @abstractmethod
    def inspect(self, identifier: str) -> Optional[SkillMeta]: ...
    @abstractmethod
    def source_id(self) -> str: ...
    def trust_level_for(self, identifier: str) -> str: return "community"
```

#### 6.3.2 GitHubSource

从 GitHub 仓库获取 Skill，支持多种认证方式：

```python
class GitHubAuth:
    """认证优先级：
    1. GITHUB_TOKEN / GH_TOKEN 环境变量（PAT）
    2. `gh auth token` 子进程（gh CLI）
    3. GitHub App JWT + installation token
    4. 未认证（60 req/hr，仅公共仓库）
    """
```

**Git Trees API 优化**：

传统的 Contents API 需要逐目录请求，一次安装可能消耗 12+ 次 API 调用（占未认证限额的 20%）。优化方案使用 Git Trees API：

```python
def _get_repo_tree(self, repo: str) -> Optional[Tuple[str, List[dict]]]:
    # 1. GET /repos/{repo} → 获取 default_branch
    # 2. GET /repos/{repo}/git/trees/{branch}?recursive=1 → 获取完整文件树
    # 结果缓存在 self._tree_cache 中，同一次搜索/安装流程内复用
```

一次 Trees API 调用即可获取仓库的完整文件结构，后续的 `_download_directory_via_tree()` 和 `_find_skill_in_repo_tree()` 直接在内存中查找，无需额外网络请求。

**速率限制处理**：
```python
def _check_rate_limit_response(self, resp):
    if resp.status_code == 403:
        remaining = resp.headers.get("X-RateLimit-Remaining", "")
        if remaining == "0":
            self._rate_limited = True
            logger.warning("GitHub API rate limit exhausted. Set GITHUB_TOKEN to raise to 5,000/hr.")
```

#### 6.3.3 WellKnownSkillSource

支持从任意域名的 `/.well-known/skills/index.json` 端点发现 Skill：

```python
class WellKnownSkillSource(SkillSource):
    BASE_PATH = "/.well-known/skills"
    # index.json 格式：
    # {
    #   "skills": [
    #     {"name": "my-skill", "description": "...", "files": ["SKILL.md", "references/api.md"]}
    #   ]
    # }
```

#### 6.3.4 HubLockFile

`HubLockFile` 追踪所有从 Hub 安装的 Skill 的来源信息：

```python
class HubLockFile:
    """管理 skills/.hub/lock.json"""

    def record_install(self, name, source, identifier, trust_level,
                       scan_verdict, skill_hash, install_path, files, metadata):
        data["installed"][name] = {
            "source": source,           # "github", "well-known", "skills-sh"
            "identifier": identifier,   # "openai/skills/skill-creator"
            "trust_level": trust_level, # "builtin", "trusted", "community"
            "scan_verdict": scan_verdict,  # "safe", "caution", "dangerous"
            "content_hash": skill_hash,    # MD5 哈希，用于变更检测
            "install_path": install_path,
            "files": files,
            "installed_at": datetime.now(timezone.utc).isoformat(),
            "updated_at": datetime.now(timezone.utc).isoformat(),
        }
```

Hub 状态目录结构：
```
~/.hermes/skills/.hub/
├── lock.json          # 安装追踪
├── quarantine/        # 隔离区（下载但未扫描的 Skill）
├── audit.log          # 安装/卸载审计日志
├── taps.json          # 自定义 GitHub 仓库源
└── index-cache/       # 远程索引缓存（TTL=1 小时）
```

### 6.4 安全扫描 (skills_guard.py)

#### 6.4.1 四级信任模型

```python
TRUSTED_REPOS = {"openai/skills", "anthropics/skills"}

INSTALL_POLICY = {
    #                  safe      caution    dangerous
    "builtin":       ("allow",  "allow",   "allow"),     # 内置 Skill，完全信任
    "trusted":       ("allow",  "allow",   "block"),     # OpenAI/Anthropic 官方
    "community":     ("allow",  "block",   "block"),     # 社区 Skill，严格审查
    "agent-created": ("allow",  "allow",   "ask"),        # Agent 自己创建的，需确认
}
```

#### 6.4.2 100+ 正则规则覆盖 13 大威胁类别

`THREAT_PATTERNS` 列表包含 100+ 条正则规则，每条规则包含：`(regex, pattern_id, severity, category, description)`

| 威胁类别 | 规则数 | 典型检测 |
|----------|--------|----------|
| **exfiltration** | 20+ | curl/wget 注入环境变量、读取 .ssh/.aws/.gnupg 目录、os.environ 泄露、DNS 隐蔽通道、Markdown 图片/链接注入 |
| **injection** | 15+ | "ignore previous instructions"、角色劫持、隐藏信息、DAN 模式越狱、developer mode、HTML 注释注入 |
| **destructive** | 8 | rm -rf /、chmod 777、mkfs、dd 磁盘覆写、shutil.rmtree |
| **persistence** | 10+ | crontab、shell rc 文件修改、authorized_keys 后门、systemd service、launchd、sudoers 修改 |
| **network** | 10+ | 反向 shell（nc/socat）、隧道服务（ngrok/cloudflared）、绑定所有接口、webhook.site 外泄 |
| **obfuscation** | 15+ | base64 解码管道执行、eval()/exec() 字符串参数、hex 编码、chr() 拼接、Unicode 转义链 |
| **execution** | 6 | subprocess.run、os.system、child_process.exec、Runtime.exec |
| **traversal** | 5 | 深度路径穿越（../../../）、/etc/passwd 访问、/proc/self |
| **mining** | 2 | xmrig、stratum+tcp、monero、coinhive |
| **supply_chain** | 8 | curl\|sh、unpinned pip/npm install、git clone、PEP 723 内联依赖 |
| **privilege_escalation** | 5 | sudo、setuid/setgid、NOPASSWD、SUID 位 |
| **credential_exposure** | 6 | 硬编码 API Key、GitHub PAT、OpenAI/Anthropic Key、AWS Access Key |
| **agent_config_mod** | 3 | 修改 AGENTS.md/CLAUDE.md/.cursorrules、Hermes 配置文件 |

#### 6.4.3 结构检查

除了正则扫描，`_check_structure()` 还检查：

```python
MAX_FILE_COUNT = 50           # 文件数上限
MAX_TOTAL_SIZE_KB = 1024      # 总大小上限 1MB
MAX_SINGLE_FILE_KB = 256      # 单文件上限 256KB
SUSPICIOUS_BINARY_EXTENSIONS  # .exe, .dll, .so, .dylib 等二进制文件
```

#### 6.4.4 Unicode 隐形字符检测

检测 17 种 Unicode 隐形字符，防止通过不可见文本隐藏恶意指令：

```python
INVISIBLE_CHARS = {
    '​',  # zero-width space
    '‌',  # zero-width non-joiner
    '‍',  # zero-width joiner
    '⁠',  # word joiner
    '⁢',  # invisible times
    '⁣',  # invisible separator
    '⁤',  # invisible plus
    '﻿',  # zero-width no-break space (BOM)
    '‪',  # left-to-right embedding
    '‫',  # right-to-left embedding
    '‮',  # left-to-right override  ← 可以反转文本显示方向
    # ... 更多
}
```

#### 6.4.5 扫描流程

```python
def scan_skill(skill_path: Path, source: str = "community") -> ScanResult:
    trust_level = _resolve_trust_level(source)
    all_findings = []

    # 1. 结构检查
    all_findings.extend(_check_structure(skill_path))

    # 2. 正则模式扫描每个文件
    for f in skill_path.rglob("*"):
        if f.is_file():
            all_findings.extend(scan_file(f, rel_path))

    # 3. 确定裁决
    verdict = _determine_verdict(all_findings)  # "safe" | "caution" | "dangerous"
    return ScanResult(...)
```

### 6.5 技能同步 (skills_sync.py)

`skills_sync.py` 负责将内置 Skill 从仓库的 `skills/` 目录同步到用户的 `~/.hermes/skills/`。

#### 6.5.1 Manifest 机制

使用 `.bundled_manifest` 文件追踪同步状态，格式为 `skill_name:origin_hash`（v2），其中 `origin_hash` 是内置 Skill 目录的 MD5 哈希。

#### 6.5.2 同步逻辑

```python
def sync_skills(quiet=False) -> dict:
    manifest = _read_manifest()
    bundled_skills = _discover_bundled_skills(bundled_dir)

    for skill_name, skill_src in bundled_skills:
        bundled_hash = _dir_hash(skill_src)

        if skill_name not in manifest:
            # 新 Skill：首次出现
            if dest.exists():
                skipped += 1      # 用户已有同名 Skill，不覆盖
            else:
                shutil.copytree(skill_src, dest)  # 复制到用户目录
                manifest[skill_name] = bundled_hash

        elif dest.exists():
            # 已存在的 Skill
            origin_hash = manifest.get(skill_name, "")
            user_hash = _dir_hash(dest)

            if user_hash != origin_hash:
                # 用户修改过 → 跳过，尊重用户修改
                user_modified.append(skill_name)
                continue

            if bundled_hash != origin_hash:
                # 内置版本有更新，用户未修改 → 安全更新
                shutil.copytree(skill_src, dest)
                manifest[skill_name] = bundled_hash
            else:
                skipped += 1  # 无变化
        else:
            # 在 manifest 中但磁盘上不存在 → 用户删除了，尊重
            skipped += 1
```

**用户修改尊重策略的核心原则：**
1. 通过 origin_hash 对比判断用户是否修改过
2. 用户修改过的 Skill 永远不会被自动覆盖
3. 用户删除的 Skill 不会被重新添加
4. 更新时先创建备份，失败则回滚

### 6.6 技能管理器 (skill_manager_tool.py)

Agent 可以通过 `skill_manage` 工具创建、编辑、删除 Skill：

**支持的操作：**
- `create`：创建新 Skill（SKILL.md + 目录结构）
- `edit`：替换 SKILL.md 内容（全文重写）
- `patch`：定向查找替换（支持 SKILL.md 和任意支持文件）
- `delete`：删除整个 Skill
- `write_file`：添加/覆盖支持文件（reference、template、script、asset）
- `remove_file`：删除支持文件

**安全措施：**
- 名称验证：`^[a-z0-9][a-z0-9._-]*$`（文件系统安全、URL 友好）
- 路径遍历防护：禁止 `..` 组件
- 子目录白名单：只允许 `references/`、`templates/`、`scripts/`、`assets/`
- 内容大小限制：SKILL.md ≤ 100,000 字符，单文件 ≤ 1 MiB
- Agent 创建的 Skill 同样经过安全扫描（skills_guard）
- 原子写入：使用 `tempfile.mkstemp()` + `os.replace()` 避免部分写入

---

## 7. 面试技术亮点

### 7.1 工具系统架构亮点

1. **自注册 + AST 发现**：零配置的工具扩展机制。新增工具只需在 `tools/` 下创建文件并调用 `registry.register()`，AST 静态分析确保只有真正注册工具的模块才会被导入，避免副作用。

2. **影子保护**：防止插件/MCP 工具意外覆盖内置工具，是生产级工具注册表的关键安全特性。

3. **快照隔离**：`_snapshot_state()` 在 RLock 内复制完整状态，读者拿到一致视图，写者（如 MCP 动态刷新）不会干扰正在进行的查询。这是读写锁模式的精简实现。

### 7.2 同步/异步桥接亮点

4. **三种场景的事件循环管理**：针对 CLI 主线程、工作线程、已有事件循环三种场景分别优化，使用持久循环避免 `asyncio.run()` 的 create-and-destroy 生命周期问题。这是解决 Python 异步生态中 "Event loop is closed" 经典问题的工程实践。

5. **参数类型强制转换**：LLM 经常将数字返回为字符串（`"42"` 而非 `42`），`coerce_tool_args()` 根据 JSON Schema 自动转换，减少工具调用失败。

### 7.3 终端系统亮点

6. **会话快照**：通过 `export -p` / `source` 机制在进程间传递环境状态，解决了 spawn-per-call 模型的核心痛点。同时支持函数、别名、shell 选项的持久化。

7. **CWD 双通道持久化**：本地后端通过文件 I/O（零网络开销），远程后端通过 stdout 标记解析（带宽友好）。

8. **ProcessHandle 协议**：使用 Python Protocol（结构化子类型）统一 subprocess.Popen 和 SDK 后端的进程接口，`_ThreadedProcessHandle` 适配器通过 `os.pipe()` 模拟 stdout 流。

9. **Docker 安全加固**：`--cap-drop ALL` + 最小能力回加、`no-new-privileges`、PID 限制、size-limited tmpfs，容器即安全边界。

### 7.4 审批系统亮点

10. **命令规范化反混淆**：三层处理（ANSI 剥离 + Null 去除 + Unicode NFKC）防御多种混淆攻击，是安全审查系统的必要预处理。

11. **队列化阻塞审批**：Gateway 模式下多个并行子 Agent 可以同时等待审批，每个调用有独立的 `threading.Event`，支持 `/approve all` 批量解决。

12. **智能审批**：辅助 LLM 评估命令风险，区分真正的恶意命令和误报（如 `python -c "print('hello')"` 被标记为 "script execution via -c flag"），减少用户审批疲劳。

### 7.5 Skill 系统亮点

13. **渐进式披露**：三层加载（元数据→完整指令→链接文件）最小化 token 消耗。Agent 先看到所有可用 Skill 的摘要，按需深入查看。

14. **Git Trees API 优化**：一次 API 调用获取完整仓库文件树，缓存在内存中复用，将安装流程的 API 调用从 12+ 次降低到 2-3 次。

15. **四级信任模型**：builtin/trusted/community/agent-created 四个信任级别，配合 safe/caution/dangerous 三级裁决矩阵，实现差异化的安装策略。

16. **100+ 威胁规则**：覆盖 13 大威胁类别的正则扫描器，包括数据外泄、提示注入、破坏性操作、持久化后门、反向 shell、混淆编码、供应链攻击、权限提升等。

17. **Unicode 隐形字符检测**：检测 17 种 Unicode 隐形字符（zero-width space、directional override 等），防止通过不可见文本隐藏恶意指令。

18. **用户修改尊重策略**：基于 origin_hash 的变更检测，用户修改过的 Skill 永远不会被自动覆盖，用户删除的 Skill 不会被重新添加。原子写入 + 备份回滚确保更新安全。

19. **agent-created Skill 安全扫描**：Agent 自己创建的 Skill 同样经过完整的安全扫描，`should_allow_install()` 对 agent-created 源返回 `None`（需确认）而非自动放行，防止 Agent 被提示注入攻击后创建恶意 Skill 实现持久化。

### 7.6 代码执行工具亮点

20. **双传输协议**：本地后端使用 Unix Domain Socket（零拷贝、低延迟），远程后端使用文件轮询 RPC（跨沙箱兼容）。两种传输共享相同的 `hermes_tools.py` 代码生成器。

21. **沙箱工具白名单**：`SANDBOX_ALLOWED_TOOLS` 限制子进程中可调用的工具（web_search、read_file 等 7 个），防止通过代码执行绕过主 Agent 的工具限制。
