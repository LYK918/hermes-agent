# Hermes Agent 项目概述

## 项目简介

Hermes Agent 是由 **Nous Research** 构建的一个**自我改进的通用 AI 智能体**。它是一个可以执行工具（运行终端命令、浏览网页、读写文件、生成图像等）、从经验中学习、并跨多个消息平台运行的对话式 AI 助手。

其核心特点是**闭环学习机制**：从经验中创建技能、在使用中改进技能、跨会话持久化记忆、搜索历史对话、并逐步构建用户画像。

---

## 核心交互模型

```
用户发送消息 → LLM 响应（可能调用工具）→ 工具执行并返回结果 → 循环直到 LLM 返回文本响应
```

默认最大迭代次数为 90 次。消息遵循 OpenAI 格式（`system/user/assistant/tool` 角色）。

---

## 架构概览

项目采用分层架构，职责清晰分离：

```
┌─────────────────────────────────────────────────────┐
│                   入口层 (Entry Points)               │
│  hermes CLI │ hermes-agent │ hermes-acp              │
├─────────────────────────────────────────────────────┤
│               CLI / Gateway / ACP Adapter            │
│  交互式终端 │ 17+ 消息平台 │ 编辑器集成              │
├─────────────────────────────────────────────────────┤
│                  核心 Agent 循环                      │
│  run_agent.py (AIAgent) → model_tools.py            │
├─────────────────────────────────────────────────────┤
│                  工具层 (Tools)                       │
│  Tool Registry → Toolset System → 57 个工具模块      │
├─────────────────────────────────────────────────────┤
│               Agent 内部子系统                        │
│  Prompt 构建 │ 上下文压缩 │ 记忆管理 │ 重试/缓存     │
└─────────────────────────────────────────────────────┘
```

### 入口点

| 入口 | 模块 | 用途 |
|------|------|------|
| `hermes` | `hermes_cli/main.py` | 主 CLI，包含 chat、gateway、setup、model 等子命令 |
| `hermes-agent` | `run_agent.py` | 直接运行 Agent |
| `hermes-acp` | `acp_adapter/entry.py` | ACP 服务器，用于编辑器集成 |

### 核心模块

| 模块 | 路径 | 职责 |
|------|------|------|
| **AIAgent** | `run_agent.py` | 核心对话循环，调用 LLM、分发工具调用 |
| **Tool Orchestration** | `model_tools.py` | 工具发现、schema 收集、调用分发、async 桥接 |
| **Tool Registry** | `tools/registry.py` | 单例注册中心，AST 扫描自动发现工具模块 |
| **Toolset System** | `toolsets.py` | 将工具组合为可复用的集合（web、terminal、browser 等） |

---

## 目录结构

```
hermes-agent/
├── run_agent.py              # AIAgent 核心对话循环
├── cli.py                    # HermesCLI 交互式终端
├── model_tools.py            # 工具编排层
├── toolsets.py               # 工具集定义与解析
├── hermes_state.py           # SQLite 会话存储（FTS5 全文搜索）
├── hermes_constants.py       # 共享常量、路径解析、Profile 支持
├── batch_runner.py           # 并行批处理
├── trajectory_compressor.py  # 训练轨迹后处理
├── mcp_serve.py              # MCP 服务器
│
├── agent/                    # Agent 内部子系统（30 个文件）
│   ├── prompt_builder.py     # 系统 Prompt 组装
│   ├── context_compressor.py # 自动上下文窗口压缩
│   ├── memory_manager.py     # 记忆编排
│   ├── auxiliary_client.py   # 辅助 LLM 调用（视觉、摘要）
│   ├── prompt_caching.py     # Anthropic Prompt 缓存
│   ├── error_classifier.py   # API 错误分类与故障转移
│   └── ...
│
├── hermes_cli/               # CLI 子命令与设置（48 个文件）
│   ├── main.py               # 入口 -- 所有 hermes 子命令
│   ├── config.py             # 默认配置、环境变量、迁移
│   ├── setup.py              # 交互式设置向导
│   ├── skin_engine.py        # CLI 主题引擎
│   ├── skills_hub.py         # 技能浏览与安装
│   ├── doctor.py             # 配置诊断
│   └── ...
│
├── tools/                    # 工具实现（57 个文件）
│   ├── registry.py           # 中央工具注册中心
│   ├── terminal_tool.py      # 终端执行（6 种后端）
│   ├── file_tools.py         # 文件读写/搜索/补丁
│   ├── web_tools.py          # 网页搜索/提取
│   ├── browser_tool.py       # 浏览器自动化
│   ├── delegate_tool.py      # 子智能体委派
│   ├── mcp_tool.py           # MCP 客户端
│   ├── memory_tool.py        # 持久化记忆
│   └── environments/         # 终端后端（10 个文件）
│
├── gateway/                  # 消息平台网关
│   ├── run.py                # 主循环、消息分发
│   ├── session.py            # 会话持久化
│   └── platforms/            # 17 个平台适配器
│
├── skills/                   # 26 个内置技能类别
├── optional-skills/          # 14 个可选技能类别
├── cron/                     # 内置定时任务调度器
├── acp_adapter/              # Agent Client Protocol 服务器
├── environments/             # RL 训练环境（Atropos）
├── plugins/                  # 插件系统
├── tests/                    # ~3000 个测试
├── scripts/                  # 安装脚本、WhatsApp 桥接
├── web/                      # Web 仪表板（Vue.js/Vite）
├── website/                  # 文档站点
└── docker/                   # Docker 配置
```

---

## 技术栈

### 语言与运行时

- **Python 3.11+**（主要语言）
- **Node.js**（浏览器自动化、WhatsApp 桥接）

### 核心 Python 依赖

| 依赖 | 用途 |
|------|------|
| `openai` (>=2.21.0) | LLM API 客户端（OpenAI 兼容，所有提供商通用） |
| `anthropic` (>=0.39.0) | Anthropic 专用适配器 |
| `httpx` | 异步 HTTP 客户端（支持 SOCKS 代理） |
| `rich` | 终端格式化、进度条、面板 |
| `prompt_toolkit` | 交互式 CLI、自动补全、多行编辑 |
| `pydantic` | 数据验证 |
| `pyyaml` | 配置文件解析 |
| `fire` | CLI 参数解析 |
| `tenacity` | 重试逻辑 |
| `jinja2` | 模板渲染 |
| `exa-py` / `firecrawl-py` / `parallel-web` | 网页搜索/提取后端 |
| `fal-client` | 图像生成 |
| `edge-tts` | 免费文本转语音 |

### 可选依赖组

modal（云沙箱）、daytona（无服务器）、messaging（Telegram/Discord/Slack）、cron、matrix、voice（faster-whisper）、pty、honcho（用户建模）、mcp、homeassistant、acp、mistral、bedrock、web（FastAPI）、rl（Atropos/Tinker）等。

### 构建系统

- **setuptools** + **uv**（推荐的包管理器）

---

## 核心功能

### 1. 内置工具（40+）

| 工具集 | 工具 | 说明 |
|--------|------|------|
| **Web** | 搜索、内容提取、爬取 | 支持 Exa、Firecrawl、Parallel、Tavily |
| **Terminal** | 命令执行 | 6 种后端：local、Docker、SSH、Modal、Daytona、Singularity |
| **Files** | 读写、搜索、补丁 | 路径安全保护 |
| **Browser** | 完整浏览器自动化 | 通过无障碍树快照，支持 Chromium、Browserbase、Browser Use |
| **Vision** | 图像分析 | 通过辅助 LLM |
| **Image Gen** | 图像生成 | fal.ai 集成 |
| **TTS** | 文本转语音 | Edge TTS（免费）、ElevenLabs（高级） |
| **Code Exec** | 代码执行 | LLM 编写 Python 脚本通过 RPC 调用工具 |
| **Delegation** | 子智能体委派 | 最大深度 2，可配置并发数 |
| **MCP** | MCP 客户端 | stdio 和 HTTP 传输，采样支持，自动重连 |
| **Memory** | 持久化记忆 | MEMORY.md/USER.md，可选 Honcho 用户建模 |
| **Skills** | 技能系统 | 渐进式披露（元数据 → 完整指令 → 链接文件） |
| **Session Search** | 会话搜索 | FTS5 全文搜索 + LLM 摘要 |
| **Cron** | 定时任务 | 自然语言调度，投递到任意消息平台 |
| **Home Assistant** | 智能家居 | 设备控制 |
| **Messaging** | 跨平台消息 | send_message 工具跨所有已连接平台 |

### 2. 消息平台集成（17 个）

Telegram、Discord、Slack、WhatsApp（桥接）、Signal、Home Assistant、Matrix、Mattermost、钉钉、飞书、企业微信、微信、QQ Bot、BlueBubbles、Email、SMS、Webhook

### 3. 编辑器集成（ACP）

通过 Agent Client Protocol 将 Hermes 暴露给 VS Code、Zed、JetBrains 等编辑器。

### 4. MCP 服务器

通过 `mcp_serve.py` 将 Hermes 对话暴露为 MCP 工具，供 Claude Code、Cursor、Codex 等使用。

### 5. 闭环学习系统

- **Agent 策划记忆**：定期提示持久化知识
- **自主技能创建**：复杂任务后自动创建技能
- **技能自我改进**：在使用中持续优化
- **跨会话搜索**：FTS5 全文搜索 + LLM 摘要
- **用户建模**：可选的 Honcho 辩证用户建模

### 6. CLI 特性

- Rich 终端 UI，多行编辑，斜杠命令自动补全
- 对话历史，中断重定向，流式工具输出
- KawaiiSpinner 动画表情
- 皮肤/主题引擎（4 个内置主题 + 用户自定义 YAML 主题）
- 多 Profile 支持（通过 `--profile/-p` 标志）

---

## 多提供商 LLM 支持

支持任何 OpenAI 兼容的 LLM 提供商，运行时可通过 `hermes model` 或 `/model` 切换：

| 提供商 | 说明 |
|--------|------|
| OpenRouter | 200+ 模型 |
| Nous Portal | Nous Research 自有 |
| Google Gemini | Google AI |
| z.ai / GLM | 智谱 AI |
| Kimi / Moonshot | 月之暗面 |
| MiniMax | MiniMax AI |
| Hugging Face | 开源模型 |
| OpenAI | GPT 系列 |
| Anthropic | Claude 系列 |
| Bedrock | AWS 托管模型 |
| Mistral | Mistral AI |
| Arcee | Arcee AI |
| 自定义端点 | 任何 OpenAI 兼容 API |

---

## 配置与部署

### 用户配置目录 `~/.hermes/`

| 文件/目录 | 用途 |
|-----------|------|
| `config.yaml` | 所有设置（模型、工具集、终端后端、显示、MCP 服务器等） |
| `.env` | API 密钥和密钥 |
| `state.db` | SQLite 会话存储（FTS5） |
| `skills/` | 用户创建和安装的技能 |
| `skins/` | 自定义 CLI 主题（YAML） |
| `cron/` | 定时任务定义 |

### 部署方式

| 方式 | 说明 |
|------|------|
| **本地安装** | `curl -fsSL ...install.sh \| bash`，支持 Linux、macOS、WSL2、Termux |
| **Docker** | Debian 基础镜像，包含 Playwright/Chromium、Node.js，非 root 用户 |
| **NixOS** | Nix Flake 打包 |
| **Gateway 服务** | `hermes gateway start/stop/install/uninstall`，支持 systemd/launchd |
| **云沙箱** | Modal（无服务器）、Daytona（无服务器持久化）、SSH 远程 |

---

## 研究能力

| 工具 | 用途 |
|------|------|
| `batch_runner.py` | 并行批处理数据集 |
| `environments/` | Atropos RL 训练环境 |
| `trajectory_compressor.py` | 训练轨迹后处理压缩 |
| `mini_swe_runner.py` | SWE 基准测试运行器 |
| `rl_cli.py` | RL 训练 CLI |

---

## 安全机制

- **命令审批系统**：`tools/approval.py`
- **DM 配对**：消息平台安全配对
- **容器隔离**：终端后端沙箱
- **凭证脱敏**：日志中自动隐藏敏感信息
- **路径安全**：文件操作路径校验
- **URL 安全检查**：网页访问安全验证

---

## 测试

项目包含约 **3000 个测试**，位于 `tests/` 目录下。

---

## 总结

Hermes Agent 是一个功能全面的自我改进 AI 智能体平台，具备以下核心优势：

1. **通用工具能力**：40+ 内置工具覆盖终端、文件、网页、浏览器、图像、语音等场景
2. **广泛平台集成**：17 个消息平台 + 编辑器集成（ACP）+ MCP 服务器
3. **闭环学习**：记忆持久化、技能自创建与自改进、跨会话搜索、用户建模
4. **多模型支持**：兼容 15+ LLM 提供商，运行时热切换
5. **灵活部署**：本地、Docker、NixOS、云沙箱多种部署方式
6. **研究友好**：内置批处理、RL 环境、轨迹压缩等研究工具
