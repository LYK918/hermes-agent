# 04 - CLI 交互式终端系统深度分析

> 学习顺序：第 4 篇 | 前置知识：prompt_toolkit、命令模式、配置管理

---

## 一、模块概述

CLI 系统采用分层架构：

| 层次 | 文件 | 职责 |
|------|------|------|
| 入口层 | `hermes_cli/main.py` | argparse 子命令注册、profile 覆盖、env 加载 |
| 交互层 | `cli.py` | `HermesCLI` 类，基于 prompt_toolkit 的 TUI 主循环 |
| 命令注册 | `hermes_cli/commands.py` | `CommandDef` 数据结构、自动补全/建议 |
| 配置层 | `hermes_cli/config.py` | YAML 配置加载/合并、环境变量桥接 |
| 主题层 | `hermes_cli/skin_engine.py` | `SkinConfig` 数据结构、动态切换 |
| 模型层 | `hermes_cli/models.py` / `model_switch.py` | 多提供商模型目录 |
| 认证层 | `hermes_cli/auth.py` | OAuth 设备码流、API Key 管理 |

---

## 二、交互式终端实现

### 2.1 prompt_toolkit 的使用

项目构建了完整的 `Application` 对象（非简单的 `prompt()`），实现固定底部输入区、状态栏、spinner 等 TUI 元素。

### 2.2 多行编辑与自动补全

`SlashCommandCompleter` 支持六层补全：
1. 斜杠命令补全
2. 子命令补全
3. 技能命令补全
4. 文件路径补全
5. @ 上下文补全（`@file:`, `@folder:`, `@diff`）
6. 模型别名补全

**Ghost text 建议**：输入 `/upd` 时显示灰色的 `ate` 提示。

### 2.3 流式输出与动画

`_stream_delta()` 实现：
- **行缓冲**：缓存到 `_stream_buf`，遇到完整行才输出
- **推理标签抑制**：`<think>` 等标签路由到推理显示框或丢弃
- **ANSI 渲染**：通过 `prompt_toolkit.formatted_text.ANSI` 解析

**Spinner 动画**：braille 字符序列 `"⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏"`。

### 2.4 TUI 布局

- 底部固定输入区
- 状态栏（上下文使用率、模型名称、时长）
- 自适应宽度（52/76 字符阈值切换紧凑/完整布局）
- 节流重绘（0.25 秒间隔防止 SSH 闪烁）

---

## 三、命令系统设计

### 3.1 CommandDef 数据结构

```python
@dataclass(frozen=True)
class CommandDef:
    name: str                          # 规范名称
    description: str                   # 描述
    category: str                      # 分类
    aliases: tuple[str, ...] = ()      # 别名
    args_hint: str = ""                # 参数提示
    subcommands: tuple[str, ...] = ()  # 子命令
    cli_only: bool = False
    gateway_only: bool = False
    gateway_config_gate: str | None = None
```

**单一事实源**：CLI 帮助、网关分发、Telegram BotCommands、自动补全都从 `COMMAND_REGISTRY` 派生。

---

## 四、配置管理

### 4.1 配置加载流程

1. 查找优先级：`~/.hermes/config.yaml` > `./cli-config.yaml`
2. 深度合并：文件配置深度合并到默认值
3. 环境变量展开：`${ENV_VAR}` 引用
4. 环境变量桥接：配置值映射到环境变量

### 4.2 配置安全

- 目录权限 0700，文件权限 0600
- 原子写入：`atomic_yaml_write()` 确保中断不会截断

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

8 个内置主题：default, ares, mono, slate, daylight, warm-lightmode, poseidon, sisyphus, charizard。

用户可在 `~/.hermes/skins/<name>.yaml` 定义自定义主题。`_SkinAwareAnsi` 实现惰性 ANSI 转义，`/skin` 切换后清除缓存。

---

## 六、模型管理

`ProviderConfig` 支持 20+ 个提供商，认证类型包括：
- OAuth 设备码流（Nous Portal）
- OAuth 外部（OpenAI Codex、Qwen）
- API Key（OpenRouter、Anthropic、Gemini 等）
- AWS SDK（Bedrock）
- 外部进程（GitHub Copilot ACP）

---

## 七、设计模式

| 模式 | 应用 |
|------|------|
| 命令模式 | `CommandDef` 数据类 |
| 模板方法 | `cmd_chat()` 定义骨架流程 |
| 建造者模式 | `HermesCLI.__init__()` 逐步构建状态 |
| 策略模式 | `ProviderConfig` 注册表 |
| 观察者模式 | 流式输出回调机制 |
| 单例模式 | 皮肤引擎的全局 `_active_skin` |

---

## 八、面试八股文

### 8.1 prompt_toolkit vs readline

| 特性 | prompt_toolkit | readline |
|------|---------------|----------|
| 纯 Python | 是 | C 扩展 |
| 多行编辑 | 原生支持 | 有限 |
| 自定义补全 | Completer 类 | 完成函数 |
| 异步 | 原生 asyncio | 同步 |
| TUI 布局 | Application + Layout | 无 |

### 8.2 配置管理方案对比

| 格式 | 优点 | 缺点 |
|------|------|------|
| YAML | 人类可读、支持注释 | 解析慢、布尔陷阱 |
| JSON | 标准格式、解析快 | 不支持注释 |
| TOML | 人类可读 | 嵌套深时冗长 |
| ENV | 简单、容器友好 | 扁平键值 |

### 8.3 命令模式的实现

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

### 8.4 流式输出实现原理

SSE → 增量回调 → 行缓冲 → ANSI 渲染 → 标签过滤

### 8.5 自动补全算法

前缀匹配 + 模糊匹配评分（精确 > 前缀 > 子串 > 路径包含 > 缩写匹配）

---

## 九、面试官可能问的问题

### Q1: 为什么选择 prompt_toolkit 而不是 curses？

prompt_toolkit 是纯 Python，原生 asyncio 支持，丰富的补全/建议 API，`patch_stdout` 解决异步输出与输入区冲突，更好的跨平台兼容性。

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

---

## 十、学习路径建议

1. `hermes_cli/commands.py` -- 命令系统心智模型
2. `hermes_cli/skin_engine.py` -- 数据驱动设计
3. `hermes_cli/config.py` 的 `DEFAULT_CONFIG` -- 配置全景
4. `hermes_cli/auth.py` 的 `PROVIDER_REGISTRY` -- 提供商抽象
5. `cli.py` 的 `HermesCLI` 类 -- 最复杂，需要前四个模块的知识
