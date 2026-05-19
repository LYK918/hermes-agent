# 04 - CLI 交互式终端系统深度分析

> 学习顺序：第 4 篇 | 前置知识：prompt_toolkit、命令模式、配置管理

---

## 一、模块概述

CLI 系统采用分层架构，实现了高内聚低耦合的设计：

| 层次 | 文件 | 职责 | 核心功能 |
|------|------|------|----------|
| 入口层 | `hermes_cli/main.py` | argparse 子命令注册、profile 覆盖、env 加载 | 50+ 子命令定义、参数解析、命令路由、启动前检查 |
| 交互层 | `cli.py` | `HermesCLI` 类，基于 prompt_toolkit 的 TUI 主循环 | 8000+ 行代码，包含混合线程模型、输入处理、流式输出、模态交互、会话管理 |
| 命令注册 | `hermes_cli/commands.py` | `CommandDef` 数据结构、自动补全/建议 | 51 内置命令定义、单源多端分发、命令查找与解析 |
| 配置层 | `hermes_cli/config.py` | YAML 配置加载/合并、环境变量桥接 | 多层级配置合并、环境变量展开、配置验证、原子写入 |
| 主题层 | `hermes_cli/skin_engine.py` | `SkinConfig` 数据结构、动态切换 | 9 个内置主题、自定义主题支持、实时主题切换、惰性 ANSI 渲染 |
| 模型层 | `hermes_cli/models.py` / `model_switch.py` | 多提供商模型目录 | 20+ 模型提供商支持、模型自动发现、配置校验、参数归一化 |
| 认证层 | `hermes_cli/auth.py` | OAuth 设备码流、API Key 管理 | 多认证方式支持、密钥安全存储、自动刷新、权限隔离 |
| 补全层 | `hermes_cli/completion.py` | Shell 补全脚本生成 | 自动从 argparse 生成 bash/zsh/fish 补全脚本、动态 profile 补全 |

---

## 二、交互式终端实现

### 2.1 prompt_toolkit 的使用

项目构建了完整的 `Application` 对象（非简单的 `prompt()`），实现固定底部输入区、状态栏、spinner 等 TUI 元素。

**核心架构特点**：
- 混合并发模型：prompt_toolkit Application 在主线程以 asyncio 事件循环驱动 UI 渲染和按键处理；Agent 执行和输入处理在后台线程（`threading.Thread`）中运行
- `patch_stdout` 代理系统输出，确保输出不会破坏底部输入区
- 自定义布局系统，支持动态显示/隐藏状态栏、模态框等元素
- 可扩展的按键绑定系统，支持多模式交互（命令输入、选择确认、密码输入等）

### 2.1.1 HermesCLI 核心类结构

```python
class HermesCLI:
    def __init__(
        self,
        model: str = None,
        toolsets: List[str] = None,
        provider: str = None,
        api_key: str = None,
        base_url: str = None,
        max_turns: int = None,
        verbose: bool = False,
        compact: bool = False,
        resume: str = None,
        checkpoints: bool = False,
        pass_session_id: bool = False,
    ):
        # 状态管理
        self._agent_running = False
        self._pending_input = queue.Queue()  # 普通输入队列
        self._interrupt_queue = queue.Queue()  # 中断输入队列
        self._should_exit = False
        
        # 模态交互状态
        self._clarify_state = None  # 多选项确认状态
        self._sudo_state = None  # 密码输入状态
        self._approval_state = None  # 命令确认状态
        self._model_picker_state = None  # 模型选择状态
        
        # UI 状态
        self._status_bar_visible = True
        self._attached_images = []  # 剪贴板图片附件
        
        # 线程管理
        self._background_tasks: Dict[str, threading.Thread] = {}
```

### 2.1.2 启动与主循环流程

```
入口 main() → HermesCLI 初始化 → show_banner() → 恢复会话（如果指定）
→ 配置文件监听器启动 → 注册系统回调（sudo/approval/secret）
→ 键盘绑定初始化 → 构建 prompt_toolkit Application 对象
→ 启动 spinner_thread + process_thread（两个 daemon 线程）
→ app.run() 进入主线程 asyncio 事件循环
```

**并发模型详解**：

CLI 采用**混合线程 + asyncio**架构，而非纯异步事件循环：

- **主线程**：运行 prompt_toolkit `Application.run()`，持有 asyncio 事件循环，负责 UI 渲染和按键事件处理
- **process_thread**（daemon threading.Thread）：运行 `process_loop()` 同步函数，处理输入队列、配置监控、Agent 调用
- **agent_thread**（daemon threading.Thread）：chat() 中启动，执行实际的 Agent 调用和结果处理
- **spinner_thread**（daemon threading.Thread）：运行 spinner 动画帧更新

**process_loop 核心逻辑**（同步函数，运行在后台线程）：

```python
def process_loop():
    while not self._should_exit:
        try:
            user_input = self._pending_input.get(timeout=0.1)
        except queue.Empty:
            # 定期检查 MCP 配置变更（仅监控 mcp_servers 节）
            if not self._agent_running:
                self._check_config_mcp_changes()
                # 检查后台进程完成通知
                ...
            continue
        
        if not user_input:
            continue
            
        # 解包图片附件
        if isinstance(user_input, tuple):
            user_input, submit_images = user_input
        
        # 处理斜杠命令 / 正常对话
        if user_input.startswith('/'):
            if not self.process_command(user_input):
                self._should_exit = True
                break
            continue
            
        self.chat(user_input, images=submit_images)
```

关键点：`process_loop` 是**同步函数**（`def process_loop():`），使用 `queue.Queue.get(timeout=0.1)` 阻塞等待输入而非 `await asyncio.Queue.get()`。prompt_toolkit 的 asyncio 事件循环仅在主线程运行，用于处理 UI 渲染和按键事件。

```python
# cli.py line 9665-9666: process_loop 运行在 threading.Thread 中
process_thread = threading.Thread(target=process_loop, daemon=True)
process_thread.start()
```

### 2.2 多行编辑与自动补全

#### 2.2.1 多层级补全系统

`SlashCommandCompleter` 支持六层补全：
1. 斜杠命令补全
2. 子命令补全
3. 技能命令补全
4. 文件路径补全
5. @ 上下文补全（`@file:`, `@folder:`, `@diff`）
6. 模型别名补全

**Ghost text 建议**：输入 `/upd` 时显示灰色的 `ate` 提示，基于历史输入和命令使用频率智能推荐。

#### 2.2.2 键盘绑定设计

```python
# 核心按键绑定
kb.add('enter')(handle_enter)  # 提交输入/确认选择
kb.add('escape', 'enter')(handle_alt_enter)  # 插入换行（多行编辑）
kb.add('c-j')(handle_ctrl_enter)  # Ctrl+Enter 插入换行
kb.add('tab', eager=True)(handle_tab)  # 补全/接受建议

# 方向键导航（多选模式）
kb.add('up', filter=Condition(lambda: bool(self._clarify_state)))(clarify_up)
kb.add('down', filter=Condition(lambda: bool(self._clarify_state)))(clarify_down)
```

**智能路由逻辑**：
- 普通模式下 Enter 提交输入
- 多选模式下 Enter 确认选择
- 密码模式下 Enter 提交密码
- 代理运行中 Enter 发送中断信号（可配置为队列模式）

### 2.2.3 系统自动补全生成

项目实现了自动补全脚本生成器，支持 bash/zsh/fish 三种 shell：

```python
def generate_bash(parser: argparse.ArgumentParser) -> str:
    # 递归遍历 argparse 解析器树
    tree = _walk(parser)
    # 生成动态补全规则，包括子命令、flag 和 profile 名称
    return completion_script
```

特性：
- 自动从 argparse 定义生成，无需手动维护
- 支持 profile 名称补全（从 `~/.hermes/profiles` 目录读取）
- 支持多级子命令补全
- 自动提取帮助文本作为补全提示

### 2.3 流式输出与动画

#### 2.3.1 流式响应处理

`_stream_delta()` 方法实现了高效的流式输出处理：
- **推理/思考标签抑制**：识别 `<REASONING_SCRATCHPAD>`、`<think>`、`<reasoning>`、`<THINKING>`、`<thinking>`、`<thought>` 等标签，配合边界检测算法（仅在流起点、换行后或行首空白后匹配才算块边界），防止模型在文中引用标签名时误触发
- **边界检测**：通过 `_stream_last_was_newline` 标记跟踪是否处于行边界，非边界位置的标签被视作普通文本输出
- **预过滤缓冲区**：增量文本先进入 `_stream_prefilt` 缓冲，完成边界判断后再输出；部分标签末尾可能被截断时暂不输出，等待后续增量

**核心实现逻辑**：
```python
def _stream_delta(self, text) -> None:
    """Line-buffered streaming callback for real-time token rendering.
    
    Reasoning/thinking blocks are suppressed during streaming by
    boundary-aware tag matching. Only tags appearing at block boundaries
    (stream start, after newline with optional whitespace) trigger
    suppression — inline mentions in prose are emitted normally.
    """
    # Tag-based reasoning suppression with boundary detection
    _OPEN_TAGS = ("<REASONING_SCRATCHPAD>", "<think>", "<reasoning>",
                   "<THINKING>", "<thinking>", "<thought>")
    _CLOSE_TAGS = ("</REASONING_SCRATCHPAD>", "</think>", "</reasoning>",
                    "</THINKING>", "</thinking>", "</thought>")
    
    self._stream_prefilt = getattr(self, "_stream_prefilt", "") + text
    
    # Check for open tag at block boundary
    if not self._in_reasoning_block:
        for tag in _OPEN_TAGS:
            idx = self._stream_prefilt.find(tag)
            if idx == -1:
                continue
            # Verify block boundary: tag must be at start or after newline
            preceding = self._stream_prefilt[:idx]
            is_boundary = (idx == 0 and self._stream_last_was_newline) or \
                          (preceding.rfind('\n') != -1 and 
                           preceding[preceding.rfind('\n')+1:].strip() == "")
            if is_boundary:
                self._emit_stream_text(preceding)  # emit before-tag text
                self._in_reasoning_block = True
                self._stream_prefilt = self._stream_prefilt[idx + len(tag):]
                break
    
    # If still in reasoning block, look for close tag
    if self._in_reasoning_block:
        # ... accumulate until close tag found, then suppress or route to
        # reasoning display box if show_reasoning enabled
        pass
    
    # Complete line emission
    self._stream_buf += text
    while '\n' in self._stream_buf:
        line, self._stream_buf = self._stream_buf.split('\n', 1)
        _cprint(f"{_STREAM_PAD}{_tc}{line}{_RST}")
```

#### 2.3.2 进度动画系统

**Spinner 动画**：使用 braille 字符序列 `"⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏"`，实现流畅的加载动画。
- 10 帧动画，每秒更新 10 次，视觉流畅
- 支持自定义动画帧和速度
- 不同状态使用不同颜色（蓝色运行中、绿色成功、红色错误）

**工具进度显示**：
```
┊ Running terminal command (0.5s)
┊   $ ls -la
┊   total 20
┊   drwxr-xr-x  2 user user 4096 May 12 10:00 .
┊   drwxr-xr-x 10 user user 4096 May 12 09:00 ..
┊   -rw-r--r--  1 user user  120 May 12 10:00 README.md
```
- 支持实时显示工具执行输出
- 嵌套工具调用层级缩进显示
- 成功/失败状态图标反馈
- 可配置显示级别（off/new/all/verbose）

### 2.4 TUI 布局

**核心布局组件**：
- 底部固定输入区（始终可见，不会被输出内容覆盖）
- 状态栏（上下文使用率、模型名称、运行时长、会话ID）
- 滚动输出区域（历史对话、工具输出、流式响应）
- 模态组件（多选确认框、密码输入框、命令审批框）

**布局特性**：
- 自适应宽度（52/76 字符阈值切换紧凑/完整布局）
- 节流重绘（0.25 秒间隔防止 SSH 连接下闪烁）
- 响应式设计，自动适配终端窗口大小变化
- 可折叠/展开状态信息，支持极简显示模式

### 2.5 模态交互系统

CLI 支持多种模态交互，统一基于状态机实现：

#### 2.5.1 多选项确认（Clarify）
```python
# 状态结构
self._clarify_state = {
    "question": "Choose an option:",
    "choices": ["Option A", "Option B", "Option C"],
    "selected": 0,
    "response_queue": asyncio.Queue()
}
```
- 方向键上下导航选项
- Enter 确认选择
- 支持"Other"选项切换到自由文本输入
- 超时自动选择默认选项

#### 2.5.2 命令审批（Approval）
```python
# 状态结构
self._approval_state = {
    "command": "rm -rf /",
    "description": "This command will delete all files on your system",
    "choices": ["Approve", "Deny", "Always allow for this session"],
    "selected": 0,
    "response_queue": asyncio.Queue()
}
```
- 显示危险命令预览和风险提示
- 支持会话级白名单
- 审批结果回调到工具执行器

#### 2.5.3 密码输入（Sudo）
```python
# 状态结构
self._sudo_state = {
    "prompt": "[sudo] password for user:",
    "response_queue": asyncio.Queue()
}
```
- 输入内容隐藏显示（星号或无回显）
- 密码安全处理，内存中自动清零
- 超时自动取消

### 2.6 输入队列系统

CLI 实现了双队列输入系统，支持中断和排队两种模式：

```python
# 队列定义
self._pending_input = queue.Queue()  # 等待下一轮对话的输入
self._interrupt_queue = queue.Queue()  # 中断当前运行的输入
```

**中断模式（默认）**：
- 代理运行中输入直接发送到中断队列
- 支持 `/stop` 命令强制终止当前运行
- 中断信号实时响应，无需等待当前步骤完成

**队列模式（可配置）**：
- 代理运行中输入添加到待执行队列
- 当前轮次完成后自动执行队列中的输入
- 支持批量提交多个任务
- 队列内容可预览和管理

---

## 三、命令系统设计

### 3.1 CommandDef 数据结构

命令系统采用**单一事实源**设计，所有命令定义集中在 `COMMAND_REGISTRY`，CLI 帮助、网关分发、Telegram BotCommands、自动补全都从此派生。

```python
@dataclass(frozen=True)
class CommandDef:
    name: str                          # 规范名称（无斜杠）
    description: str                   # 人类可读描述
    category: str                      # 分类（Session/Configuration等）
    aliases: tuple[str, ...] = ()      # 别名数组
    args_hint: str = ""                # 参数提示（如 "<prompt>", "[name]"）
    subcommands: tuple[str, ...] = ()  # 可补全的子命令
    cli_only: bool = False             # 仅 CLI 可用
    gateway_only: bool = False         # 仅网关/消息平台可用
    gateway_config_gate: str | None = None  # 配置门控（如 "feature.flag"）
```

### 3.2 命令注册表设计

```python
COMMAND_REGISTRY: list[CommandDef] = [
    # Session 类命令
    CommandDef("new", "Start a new session", "Session", aliases=("reset",)),
    CommandDef("clear", "Clear screen and start new session", "Session", cli_only=True),
    CommandDef("history", "Show conversation history", "Session", cli_only=True),
    
    # Configuration 类命令
    CommandDef("model", "Switch model for this session", "Configuration", 
               args_hint="[model] [--global]"),
    CommandDef("personality", "Set a predefined personality", "Configuration",
               args_hint="[name]"),
    
    # Tools & Skills 类命令
    CommandDef("skills", "Manage skills", "Tools & Skills", cli_only=True,
               subcommands=("search", "browse", "inspect", "install")),
    CommandDef("cron", "Manage scheduled tasks", "Tools & Skills", cli_only=True,
               subcommands=("list", "add", "create", "edit", "pause", "resume")),
]
```

### 3.3 命令处理流程

```
用户输入 "/command args" → 解析命令名 → resolve_command() 查找 CommandDef
→ 验证权限（cli_only/gateway_only）→ 检查配置门控（gateway_config_gate）
→ 路由到对应处理函数 → 执行命令逻辑 → 返回结果
```

### 3.4 跨平台兼容性设计

```python
# 网关可用命令集合（包含配置门控的 CLI 命令）
GATEWAY_KNOWN_COMMANDS: frozenset[str] = frozenset(
    name
    for cmd in COMMAND_REGISTRY
    if not cmd.cli_only or cmd.gateway_config_gate
    for name in (cmd.name, *cmd.aliases)
)

# 运行时检查配置门控是否开启
def _is_gateway_available(cmd: CommandDef) -> bool:
    if not cmd.cli_only:
        return True
    if cmd.gateway_config_gate:
        # 读取配置检查是否开启该功能
        return get_config_value(cmd.gateway_config_gate, False)
    return False
```

特性：
- 同一命令可以同时支持 CLI 和网关
- 配置门控实现功能灰度发布
- 自动生成各平台的命令列表和帮助信息
- 命令别名自动同步到所有平台

---

## 四、配置管理

### 4.1 配置加载流程

配置系统由**两套独立的配置加载路径**组成，分别服务于不同的运行时上下文：

#### 4.1.1 核心配置路径（`hermes_cli/config.py` 中的 `load_config()`）

应用于 Agent 行为、安全策略、MCP 服务器等核心功能的配置：

```
1. 内置默认配置（代码中 DEFAULT_CONFIG 字典）
   ↓
2. 用户主目录配置（~/.hermes/config.yaml）→ 深度合并覆盖
   ↓
3. 环境变量展开（${ENV_VAR} 语法替换为运行时环境变量值）
```

**特点**：两层深度合并（DEFAULT_CONFIG + 用户 YAML）+ 环境变量展开。没有项目级本地配置、没有环境变量覆盖层。

#### 4.1.2 CLI 显示配置路径（`cli.py` 中的 `load_cli_config()`）

应用于终端显示、模型路由、快捷键等交互层配置：

```
1. cli.py 内联默认值（defaults 字典）
   ↓
2. EITHER ~/.hermes/config.yaml（用户配置，优先）
   OR ./cli-config.yaml（项目级配置，回退）
   ↓
3. 环境变量桥接（将配置值映射到对应的系统环境变量）
```

**特点**：从用户配置或项目配置中二选一（用户配置优先），然后桥接到系统环境变量供底层模块使用。两个文件不会合并。

**核心处理步骤**：
1. 核心路径：DEFAULT_CONFIG → 深度合并用户 YAML → `${ENV_VAR}` 展开
2. CLI 路径：内联 defaults → 选用户或项目 YAML → 环境变量桥接
3. 配置安全：目录权限 0700，文件权限 0600，原子写入防止截断

### 4.2 配置安全

为确保配置文件的安全性，系统实现了多重保障：
- 目录权限强制设置为 0700（仅所有者可读可写可执行）
- 配置文件权限强制设置为 0600（仅所有者可读可写）
- 原子写入：`atomic_yaml_write()` 先写入临时文件，fsync 后再原子替换原文件，确保中断不会导致配置文件截断
- 敏感字段自动加密：API Key、密码等敏感信息在存储时自动加密

### 4.3 运行时配置重载

CLI 支持配置文件热重载，但**仅监控 MCP 服务器配置变更**，而非完整配置重载：

```python
def _check_config_mcp_changes(self) -> None:
    """Detect mcp_servers changes in config.yaml and auto-reload MCP."""
    CONFIG_WATCH_INTERVAL = 5.0  # seconds between config.yaml stat() calls
    
    now = time.monotonic()
    if now - self._last_config_check < CONFIG_WATCH_INTERVAL:
        return
    self._last_config_check = now
    
    # Check config file mtime
    mtime = cfg_path.stat().st_mtime
    if mtime == self._config_mtime:
        return  # File unchanged
    
    self._config_mtime = mtime
    new_cfg = yaml.safe_load(f) or {}
    new_mcp = new_cfg.get("mcp_servers") or {}
    
    # Only reload if mcp_servers section actually changed
    if new_mcp == self._config_mcp_servers:
        return  # Some OTHER section was edited — ignore
    
    self._config_mcp_servers = new_mcp
    # Reload MCP in separate thread with 30s timeout
    _reload_thread = threading.Thread(target=self._reload_mcp, daemon=True)
    _reload_thread.start()
    _reload_thread.join(timeout=30)
```

关键点：该方法名是 `_check_config_mcp_changes()`，仅比较 `mcp_servers` 节是否变更。如果用户修改了 model、display 或其他配置节，方法会检测到 mtime 变化但最终跳过（因为 mcp_servers 未变）。其他配置变更需要重启 CLI 才能生效。

### 4.4 Profile 管理系统

支持多 profile 配置隔离，实现多环境无缝切换：
```
~/.hermes/
├── config.yaml          # 默认 profile
└── profiles/
    ├── work/            # 工作环境 profile
    │   ├── config.yaml
    │   ├── auth.json
    │   └── sessions/
    └── personal/        # 个人环境 profile
        ├── config.yaml
        ├── auth.json
        └── sessions/
```

特性：
- 每个 profile 独立的配置、认证信息和会话历史
- 命令行通过 `-p/--profile <name>` 切换 profile
- 支持 profile 别名和快速切换
- 配置继承：子 profile 可以继承父 profile 的配置

---

## 五、主题引擎

```python
@dataclass
class SkinConfig:
    name: str
    colors: Dict[str, str]
    spinner: Dict[str, Any]
    branding: Dict[str, str]
    tool_prefix: str = "┊"
    tool_emojis: Dict[str, str]
    banner_logo: str = ""
```

9 个内置主题：default, ares, mono, slate, daylight, warm-lightmode, poseidon, sisyphus, charizard。

用户可在 `~/.hermes/skins/<name>.yaml` 定义自定义主题。`_SkinAwareAnsi` 类（定义在 `cli.py` 中，非 `skin_engine.py`）实现惰性 ANSI 转义：首次 `__str__()` 调用时从皮肤引擎解析颜色并缓存，`/skin` 切换后调用 `.reset()` 清除缓存强制重新解析。

---

## 六、模型管理

### 6.1 多提供商支持架构

`ProviderConfig` 数据类抽象了不同模型提供商的差异，支持 20+ 个提供商：
- **OAuth 设备码流**：Nous Portal、OpenAI Codex、Qwen 等
- **API Key**：OpenRouter、Anthropic、Gemini、OpenAI、DeepSeek 等
- **AWS SDK**：Amazon Bedrock 全系列模型
- **外部进程**：GitHub Copilot ACP、本地模型服务等
- **自定义端点**：兼容 OpenAI 接口的自定义模型服务

```python
@dataclass
class ProviderConfig:
    name: str
    base_url: str
    auth_type: str  # oauth_device, api_key, aws_sigv4, external_process
    models: List[str]
    model_aliases: Dict[str, str]
    default_params: Dict[str, Any]
    required_env_vars: List[str]
```

### 6.2 运行时凭证管理

CLI 实现了安全的凭证管理系统：
- 凭证存储在 `~/.hermes/auth.json`，权限 0600
- 支持多个凭证池，同一提供商可以配置多个 API Key
- 自动负载均衡：请求均匀分发到可用凭证
- 自动降级：失效凭证自动排除，重试时使用其他可用凭证
- 速率限制保护：自动跟踪每个凭证的速率限制，避免触发服务商限制

### 6.3 模型自动发现与切换

```python
def _auto_detect_local_model(base_url: str) -> Optional[str]:
    """自动探测本地运行的模型类型和版本"""
    try:
        response = requests.get(f"{base_url}/v1/models")
        models = response.json()["data"]
        # 识别模型类型，返回最优适配的模型名称
        return _best_match_model(models)
    except Exception:
        return None
```

特性：
- 本地模型服务自动探测，无需手动配置模型名称
- 模型参数自动适配，根据模型类型自动调整 temperature、max_tokens 等参数
- 实时模型切换，`/model` 命令无需重启即可切换当前使用的模型
- 模型别名支持，用户可以自定义短名称映射到完整模型 ID

---

## 七、Markdown 渲染管道

### 7.1 渲染链路

LLM 输出的 Markdown 文本经过完整的渲染管道才出现在用户终端：

```
Agent 流式输出（Markdown 文本）
    ↓
_stream_delta() 行缓冲 + 推理标签过滤
    ↓
_cprint() → print_formatted_text(ANSI(text))
    ↓
prompt_toolkit ANSI 解析器 → 颜色/粗体/下划线渲染
    ↓
Rich 渲染（Panel/Table/Syntax 等富组件）
    ↓
ChatConsole 捕获 Rich ANSI 输出 → 转回 _cprint()
    ↓
StdoutProxy → patch_stdout 安全输出到终端
```

### 7.2 关键组件

**`_rich_text_from_ansi(text)`**：将 ANSI 转义文本转换回 Rich Text 对象，供 Rich Console 使用。

**`_cprint(text)`**：CLI 核心输出函数，通过 `print_formatted_text(ANSI(text))` 将 ANSI 格式文本路由到 prompt_toolkit 的渲染管道。此路由确保输出经过 `patch_stdout` 代理，不会破坏底部固定输入区。

**`ChatConsole` 类**：Rich Console 适配器。内部持有 Rich `Console` 对象渲染到 `StringIO` 缓冲区，然后将渲染后的 ANSI 文本通过 `_cprint()` 逐行输出。实现 `print()` 和 `status()` 接口，可用作 Rich Console 的 drop-in 替换：
```python
class ChatConsole:
    """Rich Console adapter for prompt_toolkit's patch_stdout context."""
    
    def __init__(self):
        self._buffer = StringIO()
        self._inner = Console(
            file=self._buffer,
            force_terminal=True,
            color_system="truecolor",
            highlight=False,
        )
    
    def print(self, *args, **kwargs):
        self._buffer.seek(0)
        self._buffer.truncate()
        self._inner.width = shutil.get_terminal_size((80, 24)).columns
        self._inner.print(*args, **kwargs)
        output = self._buffer.getvalue()
        for line in output.rstrip("\n").split("\n"):
            _cprint(line)
```

---

## 八、ChatConsole 路由

### 8.1 输出路由机制

`_cprint()` 和 `ChatConsole` 是 CLI 输出的**关键路由机制**，解决了一个核心问题：在 prompt_toolkit 的 TUI 环境中，直接调用 `print()` 会破坏底部固定输入区。

所有输出必须通过以下路径：
```
代码中调用 _cprint(text)
    ↓
print_formatted_text(ANSI(text))
    ↓
prompt_toolkit 的 StdoutProxy（patch_stdout 上下文管理器）
    ↓
在底部输入区上方正确绘制，不破坏布局
```

### 8.2 使用场景

- **流式输出**：`_stream_delta()` 中直接调用 `_cprint()` 输出每一行
- **Rich 组件**：`ChatConsole().print()` 渲染 Rich Panel/Table/Syntax 后，逐行调用 `_cprint()`
- **命令输出**：`_handle_skin_command()`、`_handle_background_command()` 等命令处理函数统一使用 `_cprint()` 输出结果
- **状态消息**：工具执行状态、推理预览、完成通知等均通过 `_cprint()` 路由

---

## 九、CLI-to-Agent 集成

### 9.1 完整 chat() 调用流程

`HermesCLI.chat(message, images)` 是 CLI 与 Agent 集成的核心入口，涵盖多步预处理和安全机制：

```
chat(message, images)
  │
  ├─ 1. 凭证刷新 (_ensure_runtime_credentials)
  │     → 检查 OAuth token 过期、API key rotation
  │
  ├─ 2. 智能模型路由 (_resolve_turn_agent_config)
  │     → 简单查询路由到便宜模型 (smart_model_routing)
  │     → 复杂查询使用当前主模型
  │
  ├─ 3. Agent 初始化 (_init_agent)
  │     → 凭证变化/路由变化/模型变化时重建 Agent
  │     → 应用模型覆盖、运行时覆盖、request_overrides
  │
  ├─ 4. 图片预处理 (_preprocess_images_with_vision)
  │     → 通过 Gemini Flash vision 工具分析图片
  │     → 将图片转换为文本描述注入消息
  │     → 模型无关：任何模型都能"看到"图片
  │
  ├─ 5. @ 引用展开 (preprocess_context_references)
  │     → @file:main.py → 注入文件内容
  │     → @diff → 注入 git diff
  │     → @folder:src/ → 注入目录树
  │     → 上下文长度检查，防止超限
  │
  ├─ 6. 消息清理 (_sanitize_surrogates)
  │     → 清理剪贴板粘贴引入的非法 UTF-8 代理对
  │
  ├─ 7. Agent 执行 (agent_thread 中运行)
  │     → AIAgent.run() 在 daemon 线程中执行
  │     → 主线程监控 _interrupt_queue 检测中断
  │     → 流式回调 _stream_delta() 实时渲染输出
  │
  └─ 8. 后处理
        → 更新 conversation_history
        → TTS 语音输出（如果开启）
        → 状态栏更新
```

### 9.2 智能模型路由

`_resolve_turn_agent_config()` 根据用户消息特征自动选择最优模型：

```python
def _resolve_turn_agent_config(self, user_message: str) -> dict:
    """Resolve model/runtime overrides for a single user turn."""
    from agent.smart_model_routing import resolve_turn_route
    
    route = resolve_turn_route(
        user_message,
        self._smart_model_routing,  # 路由配置
        {  # 当前运行时信息
            "model": self.model,
            "api_key": self.api_key,
            "base_url": self.base_url,
            "provider": self.provider,
            ...
        },
    )
    return route
```

路由规则基于消息长度（`max_simple_chars` 默认 160 字符）和词数（`max_simple_words` 默认 28 词）判断是否为"简单查询"。简单查询自动路由到配置的 `cheap_model`，节省成本。路由结果包含 `model`、`runtime`、`label` 和可选的 `request_overrides`。

---

## 十、多模态输入

### 10.1 图片输入方式

CLI 支持三种图片输入方式：

**`/paste` 命令**：主动检查剪贴板中的图片。适用于终端不支持自动剪贴板检测的场景（如 VSCode 终端）。

**`/image <path>` 命令**：手动指定本地图片文件路径。

**拖放/粘贴检测**：`_detect_file_drop()` 自动识别用户粘贴或拖入的文件路径字符串，提取文件路径和行号信息，区分其与斜杠命令。

### 10.2 图片预处理管道

```python
def _preprocess_images_with_vision(self, text, images):
    """Analyze attached images via the vision tool.
    
    Instead of embedding raw base64 image data in the main conversation
    (which only works with vision-capable models), this pre-processes
    each image through the auxiliary vision model (Gemini Flash)
    and returns enriched text descriptions. This works with any model.
    """
    from tools.vision_tools import vision_analyze_tool
    
    for img_path in images:
        analysis = vision_analyze_tool(
            image_url=str(img_path),
            user_prompt=analysis_prompt
        )
        ...
```

设计理念：图片通过独立 vision model（Gemini Flash）预分析，主模型接收文本描述而非原始图片数据，确保非 vision-capable 模型也能处理图片输入。

### 10.3 剪贴板图片自动检测

`_try_attach_clipboard_image()` 在用户输入开头不为 `/` 时自动检测系统剪贴板中的图片，保存到 `~/.hermes/images/` 并自动附加到下一次对话。`_should_auto_attach_clipboard_image_on_paste()` 判断是否为纯图片粘贴手势，避免意外的图片附件。

---

## 十一、语音模式

### 11.1 状态机设计

语音模式通过 `/voice` 命令控制，是一个多状态切换系统：

```
                  /voice on
   IDLE ─────────────────────────→ VOICE_READY
                                       │
                      Ctrl+B 按下       │    Ctrl+B 释放
                                       ↓         ↓
                                  RECORDING ──→ PROCESSING (STT)
                                       │              │
                                       │              ↓
                                       │         自动发送到 Agent
                                       │              │
                                       └── Ctrl+B 停止录音
```

### 11.2 核心命令与状态

**`/voice on`**：激活语音模式，启用键盘监听（Ctrl+B 开始/停止录音）
**`/voice off`**：停用语音模式
**`/voice tts`**：切换 TTS（文本转语音）开关
**`/voice continuous`**：切换连续录音模式（每次回答后自动开始下一次录音）

### 11.3 实现细节

- **录音**：`_voice_start_recording()` 通过 `tools/voice_mode` 模块创建音频录制器，支持本地麦克风（sounddevice）和 Termux:API
- **语音转文字（STT）**：支持 faster-whisper（本地免费）、Groq API（免费额度）、OpenAI API
- **文字转语音（TTS）**：Agent 响应通过 TTS 引擎朗读，`text_queue` + `stop_event` 实现流式播放和中断
- **连续录音**：每次对话完成后自动通过 `_restart_recording` 重新开始录音
- **状态栏指示器**：录音中显示 ● REC，转换中显示 ◉ STT，就绪显示 🎤 Voice mode

---

## 十二、后台任务与 /btw

### 12.1 后台任务 (/background)

`/background <prompt>` 命令在独立的 daemon 线程中运行完整 Agent 会话，不影响当前对话：

```python
def _handle_background_command(self, cmd):
    """Spawn a new AIAgent in a background thread with its own session."""
    thread = threading.Thread(target=run_background, daemon=True,
                              name=f"bg-task-{task_id}")
    self._background_tasks[task_id] = thread
    thread.start()
```

- 后台任务具有独立的 session_id，结果存入独立会话
- 用户可继续正常对话，任务完成时通过 process_registry 通知打印结果
- 后台线程是 daemon 线程，终端关闭时自动回收

### 12.2 /btw 命令

`/btw <question>` 创建仅用当前会话上下文的临时查询，不持久化、不使用工具、不修改主会话历史：

```python
def _handle_btw_command(self, cmd):
    """Ephemeral side question using session context.
    Snapshots current conversation history, spawns a no-tools agent
    in a background thread, prints the answer without persisting.
    """
    thread = threading.Thread(target=run_btw, daemon=True,
                              name=f"btw-{task_id}")
    thread.start()
```

---

## 十三、设计模式

| 模式 | 应用 |
|------|------|
| 命令模式 | `CommandDef` 数据类 |
| 模板方法 | `chat()` 定义骨架流程 |
| 建造者模式 | `HermesCLI.__init__()` 逐步构建状态 |
| 策略模式 | `ProviderConfig` 注册表 |
| 观察者模式 | 流式输出回调机制 |
| 单例模式 | 皮肤引擎的全局 `_active_skin` |

---

## 十四、面试八股文

### 14.1 prompt_toolkit vs readline

| 特性 | prompt_toolkit | readline |
|------|---------------|----------|
| 纯 Python | 是 | C 扩展 |
| 多行编辑 | 原生支持 | 有限 |
| 自定义补全 | Completer 类 | 完成函数 |
| 异步 | 原生 asyncio | 同步 |
| TUI 布局 | Application + Layout | 无 |

### 14.2 配置管理方案对比

| 格式 | 优点 | 缺点 |
|------|------|------|
| YAML | 人类可读、支持注释 | 解析慢、布尔陷阱 |
| JSON | 标准格式、解析快 | 不支持注释 |
| TOML | 人类可读 | 嵌套深时冗长 |
| ENV | 简单、容器友好 | 扁平键值 |

### 14.3 命令模式的实现

```python
@dataclass(frozen=True)
class CommandDef:
    name: str
    description: str
    category: str
    aliases: tuple[str, ...] = ()

COMMAND_REGISTRY: list[CommandDef] = [
    CommandDef("new", "Start a new session", "Session", aliases=("reset",)),
    ...
]
```

### 14.4 流式输出实现原理

SSE → 增量回调 → 行缓冲 → 推理标签边界检测过滤 → ANSI 渲染 → prompt_toolkit 输出

### 14.5 自动补全算法

前缀匹配 + 模糊匹配评分（精确 > 前缀 > 子串 > 路径包含 > 缩写匹配）

---

## 十五、面试官可能问的问题

### Q1: 为什么选择 prompt_toolkit 而不是 curses？

prompt_toolkit 是纯 Python 实现，无需编译，跨平台兼容性更好（Windows/macOS/Linux 原生支持）。原生支持 asyncio 事件循环，可以无缝集成到异步架构中。内置丰富的补全、建议、快捷键绑定 API，以及 `patch_stdout` 机制解决了异步输出与底部固定输入区的冲突问题。相比 curses 更易扩展和维护，开发效率更高。

### Q2: 如何实现流式输出不破坏输入区？

`patch_stdout` 上下文管理器将 `sys.stdout` 替换为代理对象，异步输出临时离开渲染区域，在输入区上方输出，然后重新绘制输入区。

### Q3: 配置的深度合并有什么陷阱？

YAML 1.1 布尔陷阱（`off` 被解析为 `False`）、列表不合并直接覆盖、类型不匹配。

### Q4: 如何处理斜杠命令与文件路径的歧义？

检查第一个空格分隔的词中是否包含额外的 `/`。`/help` 不含，`/Users/foo/bar.md` 包含。

### Q5: 主题引擎如何实现实时切换？

`set_active_skin()` 更新全局变量 → `_SkinAwareAnsi.reset()` 清除缓存 → `get_prompt_toolkit_style_overrides()` 返回新样式 → 下次重绘生效。

### Q6: OAuth 设备码流的实现原理？

CLI 向 Portal 发起设备授权请求 → 获取 device_code 和 user_code → 用户在浏览器中完成授权 → 轮询返回 access_token → 持久化到 auth.json。

### Q7: 文件路径补全的模糊匹配算法？

多级评分：精确匹配 > 前缀匹配 > 子串匹配 > 路径包含 > 缩写匹配。词边界加分。

### Q8: 如何防止配置文件被截断？

原子写入：写入临时文件 → `fsync()` → `os.replace()` 原子替换。

### Q9: Git Worktree 隔离的实现？

创建隔离的 git worktree 和分支 → `.worktreeinclude` 复制 gitignored 文件 → 退出时清理 → 启动时清理过期 worktree。

### Q10: 托管模式如何影响配置管理？

跳过目录权限修改（使用组可读权限），`managed_error()` 打印升级指导。

### Q11: CLI 如何实现非交互式批量执行？

支持两种非交互式执行模式：
1. **单次查询模式**：`hermes -q "your query"`，执行后自动退出，返回结果到 stdout
2. **脚本模式**：通过管道输入多个查询，`cat queries.txt | hermes`，自动执行所有查询
3. **工作流模式**：`hermes --workflow workflow.yaml`，执行预定义的多步骤任务流

实现原理：
- 检测标准输入是否为管道，自动切换到非交互式模式
- 禁用所有需要用户输入的交互，默认选择安全选项
- 输出格式可以配置为纯文本、JSON、Markdown 等
- 执行状态通过退出码返回（0 成功，非 0 失败）

### Q12: CLI 的会话持久化是如何实现的？

会话数据存储在 SQLite 数据库 `~/.hermes/sessions.db` 中：
- 每条会话包含 ID、标题、创建时间、更新时间、来源平台、消息历史
- 消息历史以 JSON 格式存储，保留完整的角色、内容、工具调用、元数据
- 支持会话分支，基于现有会话创建新分支，不影响原会话历史
- 会话元数据独立存储，支持标签、收藏、归档等功能

实现特性：
- 增量写入，每次消息更新只写入变更部分，性能高效
- 支持全文搜索，基于消息内容快速查找历史会话
- 自动清理超过配置保留期的旧会话，节省存储空间
- 会话导入/导出功能，支持跨设备同步

### Q13: process_loop 为什么使用同步线程而不是纯 asyncio？

采用 **threading.Thread + queue.Queue** 而非 asyncio 的设计决策基于以下考虑：

1. **职责分离**：主线程的 asyncio 事件循环专注于 UI 渲染（prompt_toolkit Application），Agent 执行逻辑在独立线程中运行，互不阻塞
2. **阻塞 API 兼容**：Agent 内部可能调用同步 I/O（文件读写、子进程调用），在 threading.Thread 中可以直接执行，无需 `async`/`await` 包装
3. **中断响应**：`queue.Queue.get(timeout=0.1)` 以 100ms 粒度轮询，既能及时响应新输入，又能处理配置监控等定期任务
4. **简单可靠**：避免了 asyncio 跨线程调度（`call_soon_threadsafe`）的复杂性，queue.Queue 天然线程安全

这是一种**混合线程模型**而非全异步：prompt_toolkit 的 asyncio 事件循环处理 UI 事件 → `process_loop` 的同步线程处理业务逻辑 → 两者通过 `queue.Queue` 通信。

### Q14: 为什么配置系统拆分为两套独立的加载路径？

核心配置（`load_config()`）和 CLI 配置（`load_cli_config()`）分离的原因：

1. **不同生命周期**：核心配置影响模型行为、MCP 工具等持久化选项，CLI 配置处理终端显示、快捷键等交互偏好。前者需要完整的配置验证和版本迁移，后者只需要轻量默认值
2. **不同加载时机**：核心配置在 Agent 初始化时加载，CLI 配置在终端启动时加载
3. **不同合并策略**：核心配置使用深度合并（DEFAULT_CONFIG + 用户 YAML），CLI 配置使用二选一（用户配置优先，回退到项目配置）
4. **环境变量桥接**：CLI 配置需要将终端后端设置桥接到环境变量（DOCKER_HOST 等），核心配置通过 `${ENV_VAR}` 语法直接展开引用

### Q15: 智能模型路由如何节省成本？

`smart_model_routing` 通过消息特征分析自动选择模型：
- 短消息（默认 < 160 字符）和少词数（默认 < 28 词）被判定为"简单查询"
- 简单查询路由到 `cheap_model`（如 Gemini Flash），复杂查询使用主模型（如 Claude Opus）
- 通过在 `cli-config.yaml` 中配置 `smart_model_routing.enabled: true` 启用
- 可自定义 `max_simple_chars`、`max_simple_words`、`cheap_model` 等参数

---

## 十六、学习路径建议

1. `hermes_cli/commands.py` -- 命令系统心智模型
2. `hermes_cli/skin_engine.py` -- 数据驱动设计
3. `hermes_cli/config.py` 的 `DEFAULT_CONFIG` -- 配置全景
4. `hermes_cli/auth.py` 的 `PROVIDER_REGISTRY` -- 提供商抽象
5. `cli.py` 的 `HermesCLI` 类 -- 最复杂，需要前四个模块的知识
