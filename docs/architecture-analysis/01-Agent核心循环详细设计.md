# 01 - Agent 核心循环详细设计

> **文档目的**: 面试准备用技术深度文档，覆盖 Hermes Agent 核心循环的每一个设计细节。
> **核心文件**: `run_agent.py` (11,462行)、`agent/error_classifier.py`、`agent/retry_utils.py`、`agent/display.py`、`agent/trajectory.py`、`model_tools.py`、`hermes_constants.py`、`hermes_logging.py`、`hermes_time.py`

---

## 1. 整体架构概述

### 1.1 Agent 在系统中的位置

Hermes Agent 是一个面向多模型、多平台的 AI 工具调用代理系统。`AIAgent` 类是系统的核心引擎，负责管理完整的对话生命周期：从用户输入到 LLM 调用，到工具执行，再到最终响应生成。

```
+------------------------------------------------------------------+
|                        用户入口层                                  |
|  +-----------+  +--------------+  +-----------+  +------------+  |
|  | CLI 终端   |  | Telegram Bot |  | Discord   |  | WhatsApp   |  |
|  | hermes_cli |  |   Gateway    |  |  Gateway  |  |  Gateway   |  |
|  +-----+-----+  +------+-------+  +-----+-----+  +-----+------+  |
|        |               |               |               |          |
+--------+---------------+---------------+---------------+----------+
         |               |               |               |
         v               v               v               v
+------------------------------------------------------------------+
|                    AIAgent 核心引擎 (run_agent.py)                  |
|                                                                    |
|  +------------------+  +------------------+  +------------------+ |
|  | run_conversation  |  | IterationBudget  |  | Error Classifier| |
|  | (对话生命周期)     |  | (迭代预算控制)    |  | (错误分类恢复)   | |
|  +--------+---------+  +------------------+  +------------------+ |
|           |                                                       |
|           v                                                       |
|  +--------------------------------------------------------------+ |
|  |              主工具调用循环 (Main Loop)                         | |
|  |  LLM调用 -> 解析响应 -> 工具执行 -> 结果追加 -> 循环/退出       | |
|  +--------+----------------------------------------------+------+ |
|           |                                               |       |
|           v                                               v       |
|  +------------------+                          +----------------+ |
|  | Context          |                          | Parallel Tool  | |
|  | Compressor       |                          | Executor       | |
|  | (上下文压缩)      |                          | (并行工具执行)  | |
|  +------------------+                          +----------------+ |
+------------------------------------------------------------------+
         |                                               |
         v                                               v
+------------------------------------------------------------------+
|                      工具与模型层                                   |
|  +------------------+  +------------------+  +------------------+ |
|  | model_tools.py   |  | tools/           |  | agent/           | |
|  | (同步/异步桥接)   |  | (30+ 工具实现)   |  | (辅助模块)       | |
|  +------------------+  +------------------+  +------------------+ |
+------------------------------------------------------------------+
         |                                               |
         v                                               v
+------------------------------------------------------------------+
|                      外部服务层                                     |
|  +-----------+  +------------+  +----------+  +----------------+ |
|  | OpenRouter|  | Anthropic  |  | OpenAI   |  | AWS Bedrock    | |
|  | API       |  | API        |  | API      |  | API            | |
|  +-----------+  +------------+  +----------+  +----------------+ |
+------------------------------------------------------------------+
```

### 1.2 模块依赖关系图

```
run_agent.py (AIAgent)
    |
    +-- model_tools.py          # 工具定义获取、工具调用分发、同步/异步桥接
    |   +-- tools/registry.py   # 工具注册中心
    |   +-- tools/*.py          # 30+ 具体工具实现
    |
    +-- agent/
    |   +-- error_classifier.py  # API错误分类与恢复策略
    |   +-- retry_utils.py      # 抖动退避算法
    |   +-- display.py          # KawaiiSpinner、工具预览、diff渲染
    |   +-- trajectory.py       # 轨迹保存 (ShareGPT格式)
    |   +-- context_compressor.py# 上下文自动压缩
    |   +-- prompt_builder.py   # 系统提示词构建
    |   +-- model_metadata.py   # 模型元数据、上下文长度探测
    |   +-- memory_manager.py   # 记忆管理
    |   +-- anthropic_adapter.py # Anthropic原生API适配
    |   +-- usage_pricing.py    # Token用量与费用估算
    |   +-- prompt_caching.py   # Anthropic Prompt Caching
    |
    +-- hermes_constants.py      # 全局常量 (路径、URL、环境检测)
    +-- hermes_logging.py        # 集中日志系统
    +-- hermes_time.py           # 时区感知时钟
    +-- tools/interrupt.py       # 中断信号管理
    +-- tools/tool_result_storage.py # 大结果持久化
```

---

## 2. AIAgent 类深度剖析

### 2.1 `__init__` 参数详解 (60+ 参数)

`AIAgent.__init__` 位于 `run_agent.py` 第559行，接受60+个参数，可分为以下六大类：

#### 2.1.1 模型配置类

| 参数 | 类型 | 默认值 | 作用 |
|------|------|--------|------|
| `base_url` | str | None | LLM API 的基础 URL。决定使用哪个提供商的端点 |
| `api_key` | str | None | API 认证密钥。未提供时从环境变量自动解析 |
| `provider` | str | None | 提供商标识符 (如 "openrouter", "anthropic", "openai-codex")。用于遥测和路由决策 |
| `api_mode` | str | None | API 模式覆盖。可选值: "chat_completions", "codex_responses", "anthropic_messages", "bedrock_converse" |
| `model` | str | "" | 模型名称 (如 "anthropic/claude-opus-4.6") |
| `max_iterations` | int | 90 | 最大工具调用迭代次数。防止无限循环的安全阀 |
| `max_tokens` | int | None | 模型响应的最大 token 数。None 时使用模型默认值 |
| `reasoning_config` | Dict | None | 推理配置 (如 {"effort": "medium"})。控制思考深度 |
| `service_tier` | str | None | 服务层级 (OpenAI 特有) |
| `request_overrides` | Dict | None | 任意 API 请求参数覆盖 |
| `fallback_model` | Dict/List | None | 备用模型配置。支持单个 dict 或 list 实现链式故障转移 |
| `credential_pool` | object | None | 凭证池对象，支持多 API Key 轮换 |

**设计思考**: `api_mode` 参数的存在是因为不同提供商使用完全不同的 API 协议。Anthropic 用 Messages API，OpenAI Codex 用 Responses API，传统 OpenAI 用 Chat Completions，AWS Bedrock 用 Converse API。系统需要在运行时自动检测或手动指定使用哪种协议。

#### 2.1.2 工具系统类

| 参数 | 类型 | 默认值 | 作用 |
|------|------|--------|------|
| `enabled_toolsets` | List[str] | None | 白名单: 仅启用指定工具集 |
| `disabled_toolsets` | List[str] | None | 黑名单: 禁用指定工具集 |
| `tool_delay` | float | 1.0 | 工具调用间的延迟 (秒)。防止速率限制 |

#### 2.1.3 回调系统类 (11个回调)

这是 AIAgent 最重要的设计模式之一 -- **回调驱动的平台无关架构**。同一个 AIAgent 核心可以在 CLI、Telegram、Discord、WhatsApp 等不同平台上运行，只需要注入不同的回调函数。

| 参数 | 类型 | 触发时机 | 用途 |
|------|------|----------|------|
| `tool_progress_callback` | callable | 工具开始/完成时 | Gateway 进度推送 (如 Telegram typing indicator) |
| `tool_start_callback` | callable | 工具开始执行前 | 传递 tool_call_id 和参数给平台层 |
| `tool_complete_callback` | callable | 工具执行完成后 | 传递结果和耗时给平台层 |
| `thinking_callback` | callable | LLM 思考期间 | 更新 thinking 状态 (CLI TUI spinner) |
| `reasoning_callback` | callable | 推理内容流式到达时 | 显示推理过程 |
| `clarify_callback` | callable | Agent 需要用户澄清时 | 交互式问答 (返回用户选择) |
| `step_callback` | callable | 每次迭代开始时 | 通知平台层当前步骤 |
| `stream_delta_callback` | callable | 每个 token 到达时 | 流式输出 (TTS pipeline) |
| `interim_assistant_callback` | callable | 中间 assistant 消息时 | 显示中间回复 |
| `tool_gen_callback` | callable | 工具调用生成时 | 工具调用参数的流式预览 |
| `status_callback` | callable | 生命周期事件时 | 状态通知 (如压缩警告) |

#### 2.1.4 状态管理类

| 参数 | 类型 | 默认值 | 作用 |
|------|------|--------|------|
| `session_id` | str | None | 会话 ID。未提供时自动生成 `{timestamp}_{uuid6}` 格式 |
| `save_trajectories` | bool | False | 是否保存对话轨迹到 JSONL 文件 (训练数据) |
| `verbose_logging` | bool | False | 启用详细日志 |
| `quiet_mode` | bool | False | 静默模式 (抑制进度输出) |
| `ephemeral_system_prompt` | str | None | 临时系统提示词 (不保存到轨迹) |
| `platform` | str | None | 平台标识 ("cli", "telegram", "discord", "whatsapp") |
| `user_id` | str | None | 平台用户 ID (用于 Gateway 会话) |
| `iteration_budget` | IterationBudget | None | 共享迭代预算 (父子 Agent 共享) |
| `session_db` | object | None | SQLite 会话存储 (用于 session_search) |
| `parent_session_id` | str | None | 父会话 ID (子 Agent 场景) |
| `prefill_messages` | List[Dict] | [] | 预填充消息 (few-shot priming) |
| `skip_context_files` | bool | False | 跳过自动注入 AGENTS.md, .cursorrules 等 |
| `skip_memory` | bool | False | 跳过记忆系统初始化 |
| `checkpoints_enabled` | bool | False | 启用文件系统检查点 |
| `pass_session_id` | bool | False | 是否在系统提示词中暴露 session_id |
| `persist_session` | bool | True | 是否持久化会话到 SQLite |

#### 2.1.5 性能与日志类

| 参数 | 类型 | 默认值 | 作用 |
|------|------|--------|------|
| `log_prefix_chars` | int | 100 | 日志中工具参数预览的字符数 |
| `log_prefix` | str | "" | 日志前缀 (用于并行处理时区分不同 Agent) |

#### 2.1.6 OpenRouter 提供商路由类

| 参数 | 类型 | 默认值 | 作用 |
|------|------|--------|------|
| `providers_allowed` | List[str] | None | OpenRouter 允许的提供商白名单 |
| `providers_ignored` | List[str] | None | OpenRouter 忽略的提供商黑名单 |
| `providers_order` | List[str] | None | OpenRouter 提供商优先级顺序 |
| `provider_sort` | str | None | 提供商排序方式 (price/throughput/latency) |
| `provider_require_parameters` | bool | False | 是否要求提供商支持特定参数 |
| `provider_data_collection` | str | None | 数据收集策略 |

### 2.2 API 模式自动检测逻辑 (`_detect_api_mode`)

API 模式检测逻辑位于 `__init__` 的第688-707行，采用**优先级链**模式：

```python
# run_agent.py:688-707
if api_mode in {"chat_completions", "codex_responses", "anthropic_messages", "bedrock_converse"}:
    self.api_mode = api_mode                    # 1. 显式指定 (最高优先级)
elif self.provider == "openai-codex":
    self.api_mode = "codex_responses"            # 2. 提供商标识匹配
elif "chatgpt.com/backend-api/codex" in self._base_url_lower:
    self.api_mode = "codex_responses"            # 3. URL 模式匹配 (Codex)
    self.provider = "openai-codex"
elif self.provider == "anthropic" or "api.anthropic.com" in self._base_url_lower:
    self.api_mode = "anthropic_messages"         # 4. Anthropic 原生
elif self._base_url_lower.rstrip("/").endswith("/anthropic"):
    self.api_mode = "anthropic_messages"         # 5. 第三方 Anthropic 兼容端点
elif self.provider == "bedrock" or "bedrock-runtime" in self._base_url_lower:
    self.api_mode = "bedrock_converse"           # 6. AWS Bedrock
else:
    self.api_mode = "chat_completions"           # 7. 默认: OpenAI Chat Completions
```

此外，在第729-743行还有**二次升级逻辑**：GPT-5.x 模型和直接 OpenAI URL 会从 `chat_completions` 自动升级到 `codex_responses`。

### 2.3 四种 API 模式对比

| API 模式 | 适用场景 | 流式响应结构 | 工具调用格式 |
|----------|----------|-------------|-------------|
| `chat_completions` | 大多数 OpenAI 兼容端点、OpenRouter | `choices[0].delta` | `choices[0].message.tool_calls` |
| `codex_responses` | OpenAI Codex/GPT-5、直接 OpenAI | `response.output[]` | output items 中的 function_call |
| `anthropic_messages` | Anthropic 原生、Bedrock+Anthropic SDK | `content_block_start/delta/stop` | content blocks 中的 tool_use |
| `bedrock_converse` | AWS Bedrock (非 Anthropic SDK) | `contentBlockDelta` | Converse API 的 toolUse 格式 |

**设计决策**: 为什么不用统一的适配器层？因为每种 API 的流式协议差异太大（Anthropic 用 SSE content block events，OpenAI 用 delta chunks，Codex 用 output items），强行统一会丢失每种协议的特性（如 Anthropic 的 thinking blocks、Codex 的 response items）。因此选择在主循环中用 `if/elif` 分支处理不同 API 模式的响应解析。

---

## 3. IterationBudget 机制

### 3.1 设计动机

`IterationBudget` 位于 `run_agent.py` 第170-211行，是防止 Agent 进入无限工具调用循环的安全机制。

**为什么需要迭代预算？** LLM 在工具调用循环中可能出现以下问题：
- 反复调用同一个工具但参数略有不同（死循环）
- 在复杂任务中无限展开子任务
- 工具返回错误但 LLM 不断重试

### 3.2 核心实现

```python
# run_agent.py:170-211
class IterationBudget:
    """Thread-safe iteration counter for an agent."""

    def __init__(self, max_total: int):
        self.max_total = max_total       # 预算上限 (默认90)
        self._used = 0                    # 已使用次数
        self._lock = threading.Lock()     # 线程安全锁

    def consume(self) -> bool:
        """Try to consume one iteration. Returns True if allowed."""
        with self._lock:
            if self._used >= self.max_total:
                return False              # 预算耗尽
            self._used += 1
            return True

    def refund(self) -> None:
        """Give back one iteration (e.g. for execute_code turns)."""
        with self._lock:
            if self._used > 0:
                self._used -= 1

    @property
    def remaining(self) -> int:
        with self._lock:
            return max(0, self.max_total - self._used)
```

### 3.3 线程安全设计

使用 `threading.Lock` 保护所有读写操作。这是因为：
- 工具并行执行时，多个线程可能同时调用 `consume()`
- 子 Agent (delegate_task) 在独立线程中运行，也需要操作预算
- `_used` 的递增和检查必须是原子操作，否则两个线程可能同时通过检查

### 3.4 execute_code 退还机制

`execute_code` 工具（程序化工具调用）的迭代次数会通过 `refund()` 退还。原因是 `execute_code` 本质上是一个自动化工具，不应该消耗用户可见的迭代预算。这保证了自动化脚本不会意外耗尽 Agent 的工具调用配额。

### 3.5 父子 Agent 独立预算

```
父 Agent: IterationBudget(90)  ←── 用户配置 max_iterations
    |
    +-- 子 Agent 1: IterationBudget(50)  ←── delegation.max_iterations
    +-- 子 Agent 2: IterationBudget(50)  ←── delegation.max_iterations
```

**关键设计**: 每个 Agent（父或子）获得**独立的** IterationBudget。父 Agent 的预算上限由 `max_iterations` 控制（默认90），每个子 Agent 的预算由 `delegation.max_iterations` 控制（默认50）。这意味着总迭代次数可以超过父 Agent 的上限——这是有意为之的设计，因为子任务的复杂度不可预测。

在 `run_conversation` 中（第8195行），每个对话轮次开始时会创建新的 `IterationBudget`：

```python
# run_agent.py:8195
self.iteration_budget = IterationBudget(self.max_iterations)
```

---

## 4. 对话生命周期 (`run_conversation`)

`run_conversation` 方法位于 `run_agent.py` 第8102行，是整个 Agent 的核心入口。它接受用户消息，执行完整的工具调用循环，返回最终响应。

### 4.1 八个步骤详解

#### 步骤 1: 初始化与清理 (行8130-8170)

```python
# 行8130-8137: 安装安全 stdio 包装器
_install_safe_stdio()

# 行8136-8137: 设置日志会话上下文
from hermes_logging import set_session_context
set_session_context(self.session_id)

# 行8142: 恢复主运行时 (如果上一轮启用了 fallback)
self._restore_primary_runtime()

# 行8147-8150: 清理用户输入中的代理字符
if isinstance(user_message, str):
    user_message = _sanitize_surrogates(user_message)
```

**为什么需要 `_sanitize_surrogates`？** 富文本编辑器（Google Docs、Word）的剪贴板复制可能注入孤立的 surrogate 字符（U+D800-U+DFFF），这些字符在 UTF-8 中是无效的，会导致 OpenAI SDK 的 `json.dumps()` 崩溃。函数将其替换为 U+FFFD（替换字符）。

#### 步骤 2: 重置轮次状态 (行8159-8171)

```python
self._invalid_tool_retries = 0
self._invalid_json_retries = 0
self._empty_content_retries = 0
self._incomplete_scratchpad_retries = 0
self._codex_incomplete_retries = 0
self._thinking_prefill_retries = 0
self._post_tool_empty_retried = False
self._mute_post_response = False
self._unicode_sanitization_passes = 0
```

每种重试计数器对应一种特定的错误模式，确保上一轮的重试状态不会泄漏到下一轮。

#### 步骤 3: 连接健康检查 (行8176-8185)

```python
if self.api_mode != "anthropic_messages":
    try:
        if self._cleanup_dead_connections():
            self._emit_status("🔌 Detected stale connections...")
    except Exception:
        pass
```

在非 Anthropic 模式下，检测并清理上次会话遗留的死 TCP 连接。Anthropic SDK 有自己的连接管理，不需要此步骤。

#### 步骤 4: 系统提示词构建 (行8248-8298)

系统提示词采用**分层构建 + 缓存**策略：

```
系统提示词层次:
1. Agent 身份 (SOUL.md 或 DEFAULT_AGENT_IDENTITY)
2. 用户/Gateway 自定义系统消息
3. 持久化记忆 (MEMORY.md + USER.md)
4. 外部记忆提供商内容
5. Skills 指导
6. 上下文文件 (AGENTS.md, .cursorrules)
7. 当前日期时间
8. 平台特定格式化提示
```

**缓存策略**: 系统提示词在每个会话中只构建一次（第8259行），后续轮次复用缓存。只在上下文压缩事件后才重建。这对于 Anthropic 的 prompt caching 至关重要——系统提示词的稳定性直接影响缓存命中率。

#### 步骤 5: 预压缩检查 (行8300-8366)

```python
# 行8307-8311: 检查是否需要预压缩
if (
    self.compression_enabled
    and len(messages) > self.context_compressor.protect_first_n
                        + self.context_compressor.protect_last_n + 1
):
    _preflight_tokens = estimate_request_tokens_rough(
        messages, system_prompt=active_system_prompt, tools=self.tools
    )
    if _preflight_tokens >= self.context_compressor.threshold_tokens:
        # 最多执行3轮压缩
        for _pass in range(3):
            messages, active_system_prompt = self._compress_context(...)
```

**为什么需要预压缩？** 当用户切换到上下文窗口更小的模型时，已有的对话历史可能已经超出新模型的限制。预压缩在进入主循环前主动压缩，而不是等到 API 报错后再处理。

#### 步骤 6: 插件钩子 - pre_llm_call (行8368-8402)

```python
_pre_results = _invoke_hook(
    "pre_llm_call",
    session_id=self.session_id,
    user_message=original_user_message,
    conversation_history=list(messages),
    is_first_turn=(not bool(conversation_history)),
    model=self.model,
    platform=getattr(self, "platform", None) or "",
    sender_id=getattr(self, "_user_id", None) or "",
)
```

插件可以返回上下文内容，这些内容会被注入到用户消息中（不是系统提示词），以保持系统提示词的缓存前缀不变。

#### 步骤 7: 主工具调用循环 (行8444-9999+)

这是整个方法的核心，详见第5节。

#### 步骤 8: 清理与返回 (循环结束后)

循环结束后执行：
- 检查是否需要触发记忆审查 nudge
- 检查是否需要触发 skill 创建 nudge
- 持久化会话到 SQLite
- 清理任务资源 (终端沙箱、浏览器等)
- 保存轨迹 (如果启用了 trajectory saving)
- 返回结果 dict

---

## 5. 主工具调用循环

### 5.1 完整循环流程图

```
                    +---------------------------+
                    |     run_conversation()    |
                    +-------------+-------------+
                                  |
                                  v
                    +---------------------------+
                    | 初始化: messages, budget,  |
                    | system_prompt, prefetch    |
                    +-------------+-------------+
                                  |
                                  v
              +-----------------------------------+
              |  while (api_call_count < max AND  |
              |    budget.remaining > 0) OR       |
              |    _budget_grace_call:            |
              +----------------+------------------+
                               |
                               v
              +-----------------------------------+
              | 1. 检查 interrupt_requested       |
              | 2. api_call_count++               |
              | 3. budget.consume()               |
              | 4. fire step_callback             |
              +----------------+------------------+
                               |
                               v
              +-----------------------------------+
              | 5. 构建 api_messages:             |
              |    - 注入临时上下文               |
              |    - 处理 reasoning_content       |
              |    - 注入系统提示词               |
              |    - 注入 prefill_messages        |
              |    - 应用 prompt caching          |
              |    - 清理孤立 tool_call/result    |
              |    - 规范化 whitespace/JSON       |
              +----------------+------------------+
                               |
                               v
              +-----------------------------------+
              | 6. 内部重试循环 (max_retries=3):  |
              |    +---------------------------+  |
              |    | 构建 api_kwargs           |  |
              |    | 调用 LLM (streaming)      |  |
              |    | 验证响应 shape            |  |
              |    | 处理 finish_reason        |  |
              |    | 更新 token 用量统计       |  |
              |    | break (成功) 或 retry     |  |
              |    +---------------------------+  |
              +----------------+------------------+
                               |
                               v
              +-----------------------------------+
              | 7. 解析 assistant 响应:           |
              |    - 提取 tool_calls              |
              |    - 提取 text content            |
              |    - 处理 reasoning blocks        |
              +----------------+------------------+
                               |
                    +----------+----------+
                    |                     |
                    v                     v
          +------------------+  +------------------+
          | 有 tool_calls?   |  | 无 tool_calls?   |
          |                  |  | (纯文本响应)      |
          +--------+---------+  +--------+---------+
                   |                     |
                   v                     v
          +------------------+  +------------------+
          | _execute_tool_   |  | final_response   |
          | calls()          |  | = content        |
          | 追加 tool 结果   |  | 退出循环          |
          +------------------+  +------------------+
                   |
                   v
          +------------------+
          | 检查上下文压力    |
          | 检查是否需要压缩  |
          | 继续循环...       |
          +------------------+
```

### 5.2 LLM 调用与流式处理

LLM 调用始终优先使用流式路径（第8784行）：

```python
# 行8770-8789
_use_streaming = True
if getattr(self, "_disable_streaming", False):
    _use_streaming = False
elif not self._has_stream_consumers():
    from unittest.mock import Mock
    if isinstance(getattr(self, "client", None), Mock):
        _use_streaming = False

if _use_streaming:
    response = self._interruptible_streaming_api_call(
        api_kwargs, on_first_delta=_stop_spinner
    )
else:
    response = self._interruptible_api_call(api_kwargs)
```

**为什么总是优先流式？** 流式响应提供了细粒度的健康检查能力（90秒无数据检测、60秒读取超时）。非流式路径缺乏这种检测，子 Agent 和其他静默模式的调用者可能无限挂起。

### 5.3 响应解析逻辑

根据 `api_mode` 的不同，响应解析逻辑完全不同：

```python
# Chat Completions 模式 (行9018-9019)
finish_reason = response.choices[0].finish_reason
assistant_message = response.choices[0].message

# Anthropic Messages 模式 (行9015-9017)
stop_reason_map = {"end_turn": "stop", "tool_use": "tool_calls", "max_tokens": "length"}
finish_reason = stop_reason_map.get(response.stop_reason, "stop")

# Codex Responses 模式 (行9003-9014)
status = getattr(response, "status", None)
incomplete_reason = getattr(response, "incomplete_details", {}).get("reason")
if status == "incomplete" and incomplete_reason in {"max_output_tokens", "length"}:
    finish_reason = "length"
```

### 5.4 循环退出条件

循环在以下情况退出：
1. **`finish_reason == "stop"` 且无 tool_calls**: 正常完成
2. **`api_call_count >= max_iterations`**: 迭代次数达到上限
3. **`iteration_budget.remaining <= 0`**: 预算耗尽
4. **`_interrupt_requested`**: 用户中断
5. **不可恢复的错误**: 所有重试和 fallback 都失败
6. **`_budget_grace_call` 完成**: 预算耗尽后的宽限调用

---

## 6. 并行工具执行机制

### 6.1 工具分类集合

```python
# run_agent.py:214-237

# 绝对不能并行的工具 (交互式/用户面向)
_NEVER_PARALLEL_TOOLS = frozenset({"clarify"})

# 只读工具，无共享可变状态，可安全并行
_PARALLEL_SAFE_TOOLS = frozenset({
    "ha_get_state", "ha_list_entities", "ha_list_services",
    "read_file", "search_files", "session_search",
    "skill_view", "skills_list", "vision_analyze",
    "web_extract", "web_search",
})

# 路径作用域工具: 目标路径不同时可并行
_PATH_SCOPED_TOOLS = frozenset({"read_file", "write_file", "patch"})

# 最大并发 worker 数
_MAX_TOOL_WORKERS = 8
```

**为什么最大 8 个 worker？** 这是一个经验值平衡：
- 太少 (2-4): 无法充分利用 IO 密集型工具的并发性
- 太多 (16+): 线程创建开销大，且可能触发提供商的并发限制
- 8 个: 足够覆盖大多数场景（典型的一批工具调用 2-5 个），同时不会过度消耗资源

### 6.2 并行化判断逻辑

`_should_parallelize_tool_batch` 位于第267行，执行以下检查：

```python
def _should_parallelize_tool_batch(tool_calls) -> bool:
    # 1. 只有1个工具调用，不需要并行
    if len(tool_calls) <= 1:
        return False

    # 2. 包含绝对不能并行的工具 (如 clarify)
    if any(name in _NEVER_PARALLEL_TOOLS for name in tool_names):
        return False

    # 3. 对每个工具调用检查
    reserved_paths = []
    for tool_call in tool_calls:
        # 3a. 解析参数失败 → 降级为串行
        function_args = json.loads(tool_call.function.arguments)

        # 3b. 路径作用域工具: 检查路径冲突
        if tool_name in _PATH_SCOPED_TOOLS:
            scoped_path = _extract_parallel_scope_path(tool_name, function_args)
            if scoped_path is None:
                return False  # 无法提取路径 → 保守降级
            if any(_paths_overlap(scoped_path, existing) for existing in reserved_paths):
                return False  # 路径冲突 → 串行执行
            reserved_paths.append(scoped_path)
            continue

        # 3c. 非安全工具 → 串行
        if tool_name not in _PARALLEL_SAFE_TOOLS:
            return False

    return True
```

### 6.3 路径冲突检测算法

```python
# run_agent.py:328-336
def _paths_overlap(left: Path, right: Path) -> bool:
    """Return True when two paths may refer to the same subtree."""
    left_parts = left.parts
    right_parts = right.parts
    if not left_parts or not right_parts:
        return bool(left_parts) == bool(right_parts) and bool(left_parts)
    common_len = min(len(left_parts), len(right_parts))
    return left_parts[:common_len] == right_parts[:common_len]
```

这是一个**前缀匹配**算法：如果两个路径的前 N 个组成部分相同（N = 较短路径的深度），则认为它们可能指向同一子树。例如：
- `/home/user/a.txt` 和 `/home/user/b.txt` → 冲突（前缀相同）
- `/home/user/a.txt` 和 `/tmp/b.txt` → 不冲突

**设计选择**: 使用前缀匹配而非精确匹配，是因为 `write_file` 操作可能影响目录结构（如创建中间目录）。保守地认为同一目录下的操作可能冲突。

### 6.4 并行执行实现

```python
# run_agent.py:7293-7457
def _execute_tool_calls_concurrent(self, ...):
    # 预解析所有工具调用参数
    parsed_calls = []
    for tool_call in tool_calls:
        function_args = json.loads(tool_call.function.arguments)
        # 检查点: 文件修改前快照
        if function_name in ("write_file", "patch") and self._checkpoint_mgr.enabled:
            self._checkpoint_mgr.ensure_checkpoint(...)
        parsed_calls.append((tool_call, function_name, function_args))

    # 使用 ThreadPoolExecutor 并行执行
    max_workers = min(num_tools, _MAX_TOOL_WORKERS)
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = []
        for i, (tc, name, args) in enumerate(parsed_calls):
            f = executor.submit(_run_tool, i, tc, name, args)
            futures.append(f)

        # 带心跳的等待，防止 Gateway 超时
        while True:
            done, not_done = concurrent.futures.wait(futures, timeout=30.0)
            if not not_done:
                break
            self._touch_activity(f"concurrent tools running ({_conc_elapsed}s, ...)")
```

---

## 7. 同步/异步桥接 (`model_tools.py`)

### 7.1 为什么需要桥接

AIAgent 的主循环是**同步**的（`while` 循环 + 同步函数调用），但工具实现和 Gateway 通信是**异步**的（基于 asyncio + httpx）。这种架构冲突需要一个桥接层。

### 7.2 三种场景处理

`_run_async` 函数位于 `model_tools.py` 第81行，处理三种不同的执行上下文：

```python
# model_tools.py:81-125
def _run_async(coro):
    """Run an async coroutine from a sync context."""

    # 场景1: 已有运行中的事件循环 (Gateway、RL环境)
    # → 在新线程中运行 asyncio.run()
    try:
        loop = asyncio.get_running_loop()
    except RuntimeError:
        loop = None
    if loop and loop.is_running():
        with concurrent.futures.ThreadPoolExecutor(max_workers=1) as pool:
            future = pool.submit(asyncio.run, coro)
            return future.result(timeout=300)

    # 场景2: 工作线程 (并行工具执行)
    # → 使用 per-thread 持久化事件循环
    if threading.current_thread() is not threading.main_thread():
        worker_loop = _get_worker_loop()
        return worker_loop.run_until_complete(coro)

    # 场景3: 主线程 (CLI)
    # → 使用全局持久化事件循环
    tool_loop = _get_tool_loop()
    return tool_loop.run_until_complete(coro)
```

### 7.3 持久化事件循环 vs asyncio.run()

**核心问题**: `asyncio.run()` 每次调用都会创建一个新的事件循环，运行完毕后**关闭**它。但缓存的 httpx/AsyncOpenAI 客户端仍然绑定到那个已关闭的循环，在垃圾回收时会抛出 "Event loop is closed" 错误。

**解决方案**: 使用持久化事件循环：

```python
# model_tools.py:39-57
_tool_loop = None          # 主线程共享的持久化循环
_tool_loop_lock = threading.Lock()
_worker_thread_local = threading.local()  # per-worker-thread 持久化循环

def _get_tool_loop():
    global _tool_loop
    with _tool_loop_lock:
        if _tool_loop is None or _tool_loop.is_closed():
            _tool_loop = asyncio.new_event_loop()
        return _tool_loop

def _get_worker_loop():
    loop = getattr(_worker_thread_local, 'loop', None)
    if loop is None or loop.is_closed():
        loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)
        _worker_thread_local.loop = loop
    return loop
```

### 7.4 threading.local() 的使用

`_worker_thread_local = threading.local()` 确保每个 worker 线程（来自 ThreadPoolExecutor）拥有自己独立的事件循环。这避免了：
1. **竞争**: 多个 worker 线程共享同一个循环会导致 `run_until_complete` 冲突
2. **关闭问题**: 一个 worker 关闭循环会影响其他 worker

---

## 8. 错误恢复机制

### 8.1 错误分类器 (`agent/error_classifier.py`)

`classify_api_error` 函数是错误恢复的核心，位于 `agent/error_classifier.py` 第242行。它实现了一个**优先级有序的分类管线**：

```
输入: Exception
    |
    v
1. 提取: status_code, error_type, body, error_code, error_msg
    |
    v
2. 优先级1: 提供商特定模式
   - Anthropic thinking signature (400 + "signature" + "thinking")
   - Anthropic long-context tier (429 + "extra usage" + "long context")
    |
    v
3. 优先级2: HTTP 状态码分类
   - 401 → auth (可轮换凭证)
   - 402 → 区分 billing vs 暂时性限额
   - 404 → model_not_found
   - 413 → payload_too_large (可压缩)
   - 429 → rate_limit (可退避+轮换)
   - 400 → 区分 context_overflow vs format_error
   - 500/502 → server_error (可重试)
   - 503/529 → overloaded (可重试)
    |
    v
4. 优先级3: 错误码分类
   - resource_exhausted → rate_limit
   - insufficient_quota → billing
   - context_length_exceeded → context_overflow
    |
    v
5. 优先级4: 消息模式匹配
   - billing patterns (12个模式)
   - rate_limit patterns (13个模式)
   - context_overflow patterns (20+个模式，含中文)
   - auth patterns (8个模式)
    |
    v
6. 优先级5: 传输层错误
   - 大会话 + 断连 → context_overflow
   - 其他断连 → timeout
    |
    v
7. 优先级6: 默认 unknown (可重试)
```

### 8.2 FailoverReason 枚举

```python
class FailoverReason(enum.Enum):
    auth = "auth"                          # 暂时性认证失败 → 刷新/轮换
    auth_permanent = "auth_permanent"      # 永久认证失败 → 终止
    billing = "billing"                    # 额度耗尽 → 立即轮换
    rate_limit = "rate_limit"              # 速率限制 → 退避后轮换
    overloaded = "overloaded"              # 提供商过载 → 退避
    server_error = "server_error"          # 服务端错误 → 重试
    timeout = "timeout"                    # 超时 → 重建客户端+重试
    context_overflow = "context_overflow"  # 上下文溢出 → 压缩
    payload_too_large = "payload_too_large"# 413 → 压缩
    model_not_found = "model_not_found"    # 模型不存在 → 切换模型
    format_error = "format_error"          # 请求格式错误 → 终止或重试
    thinking_signature = "thinking_signature"  # Anthropic thinking签名失效
    long_context_tier = "long_context_tier"    # Anthropic 长上下文层级门控
    unknown = "unknown"                    # 未知 → 退避重试
```

### 8.3 ClassifiedError 恢复建议

```python
@dataclass
class ClassifiedError:
    reason: FailoverReason
    status_code: Optional[int]
    retryable: bool = True                    # 是否可重试
    should_compress: bool = False             # 是否应压缩上下文
    should_rotate_credential: bool = False    # 是否应轮换凭证
    should_fallback: bool = False             # 是否应切换到备用提供商
```

主循环根据这些布尔标志决定恢复策略，而不是重新分析错误。

### 8.4 重试策略 (抖动退避)

```python
# agent/retry_utils.py
def jittered_backoff(
    attempt: int,
    *,
    base_delay: float = 5.0,    # 基础延迟 5 秒
    max_delay: float = 120.0,   # 最大延迟 120 秒
    jitter_ratio: float = 0.5,  # 抖动比例 50%
) -> float:
    # 指数退避: delay = min(base * 2^(attempt-1), max_delay)
    exponent = max(0, attempt - 1)
    delay = min(base_delay * (2 ** exponent), max_delay)

    # 抖动: 使用 time_ns + 单调计数器作为种子
    seed = (time.time_ns() ^ (tick * 0x9E3779B9)) & 0xFFFFFFFF
    rng = random.Random(seed)
    jitter = rng.uniform(0, jitter_ratio * delay)

    return delay + jitter
```

**为什么需要抖动？** 固定的指数退避会导致"惊群效应"——当多个会话同时被同一提供商限流时，它们会在完全相同的时间点重试，再次触发限流。抖动通过引入随机延迟来分散重试时间。

**为什么使用独立的 Random 实例？** 使用 `random.Random(seed)` 而非全局 `random` 模块，确保：
1. 不受全局随机状态影响
2. 线程安全（无需锁）
3. 种子可重现（便于调试）

### 8.5 凭证轮换

当错误分类为 `should_rotate_credential=True` 时，主循环调用 `_recover_with_credential_pool()`：

```python
# 行9487-9494
recovered_with_pool, has_retried_429 = self._recover_with_credential_pool(
    status_code=status_code,
    has_retried_429=has_retried_429,
    classified_reason=classified.reason,
    error_context=error_context,
)
if recovered_with_pool:
    continue  # 使用新凭证重试
```

凭证池轮换流程：
1. 检查凭证池是否有可用的备用凭证
2. 如果有，替换当前 api_key 和 client
3. 重置重试计数器
4. 继续循环（使用新凭证重试）

### 8.6 上下文溢出自动压缩

当错误分类为 `context_overflow` 或 `payload_too_large` 时：

```python
# 行9819-9947 (context_overflow 处理)
# 1. 尝试从错误消息中解析实际限制
parsed_limit = parse_context_limit_from_error(error_msg)

# 2. 更新上下文长度
compressor.update_model(context_length=new_ctx)

# 3. 压缩对话历史
messages, active_system_prompt = self._compress_context(
    messages, system_message, approx_tokens=approx_tokens,
    task_id=effective_task_id,
)

# 4. 最多重试 3 次
compression_attempts += 1
if compression_attempts > max_compression_attempts:
    return {"error": "...", "compression_exhausted": True}
```

---

## 9. 资源清理 (`close` 方法)

`close()` 位于 `run_agent.py` 第3182行，负责释放 Agent 持有的所有资源。

### 9.1 清理步骤

```python
def close(self) -> None:
    """Release all resources held by this agent instance."""

    # 1. 终止后台进程 (ProcessRegistry)
    from tools.process_registry import process_registry
    process_registry.kill_all(task_id=task_id)

    # 2. 清理终端沙箱环境
    from tools.terminal_tool import cleanup_vm
    cleanup_vm(vm_id)

    # 3. 关闭浏览器守护进程
    from tools.browser_tool import cleanup_browser
    cleanup_browser(task_id)

    # 4. 关闭子 Agent (递归)
    with self._active_children_lock:
        children = list(self._active_children)
        self._active_children.clear()
    for child in children:
        child.close()

    # 5. 关闭 OpenAI/httpx 客户端
    if self.client is not None:
        self._close_openai_client(self.client, reason="agent_close", shared=True)
        self.client = None
```

### 9.2 设计要点

1. **幂等性**: 每个清理步骤独立守卫，失败不影响后续步骤
2. **递归关闭**: 子 Agent 递归调用 `close()`，确保整个 Agent 树被清理
3. **资源隔离**: 每种资源类型独立处理，避免单一失败导致全部泄漏
4. **线程安全**: `_active_children` 的读写使用 `_active_children_lock` 保护

---

## 10. 面试技术亮点

### 10.1 设计决策背后的思考

#### 10.1.1 为什么选择回调驱动而非接口继承？

**问题**: 如何让同一个 Agent 核心支持 CLI、Telegram、Discord 等不同平台？

**方案对比**:
- 方案A: 为每个平台创建子类 (CLIAgent, TelegramAgent...) → 类爆炸，代码重复
- 方案B: 定义平台接口，每个平台实现接口 → 需要修改 Agent 核心的调用方式
- **方案C (采用)**: 通过构造函数注入 11 个回调函数 → Agent 核心完全平台无关

**优势**: 新平台只需要实现回调函数，不需要修改 Agent 代码。CLI 的 `_cprint`、Telegram 的 `send_message`、Discord 的 `channel.send` 都可以通过回调注入。

#### 10.1.2 为什么系统提示词注入到 user message 而非 system prompt？

**问题**: 插件需要注入上下文信息，应该放在系统提示词还是用户消息中？

**决策**: 始终注入到用户消息中。原因是系统提示词是 prompt cache 的前缀，任何修改都会导致缓存失效。Anthropic 的 prompt caching 可以节省 ~75% 的输入 token 成本，破坏缓存的代价远大于上下文位置的差异。

#### 10.1.3 为什么使用优先级链而非配置文件来检测 API 模式？

**问题**: 如何自动检测应该使用哪种 API 协议？

**决策**: 使用代码中的优先级链（而非配置文件），因为：
1. 检测逻辑需要实时信息（URL、provider 名称）
2. 配置文件无法处理 "先匹配 provider，再匹配 URL" 的复杂逻辑
3. 用户可以通过 `api_mode` 参数覆盖自动检测（最高优先级）

### 10.2 遇到的问题和解决方案

#### 10.2.1 "Event loop is closed" 错误

**问题**: 使用 `asyncio.run()` 运行异步工具后，缓存的 httpx 客户端在 GC 时崩溃。

**根因**: `asyncio.run()` 创建循环 → 运行 → 关闭循环。但客户端的 `__del__` 尝试在已关闭的循环上运行清理。

**解决**: 使用持久化事件循环（`_get_tool_loop()`），循环在进程生命周期内保持活跃。

#### 10.2.2 孤立 surrogate 字符导致 JSON 序列化崩溃

**问题**: 用户从 Google Docs 复制文本时，剪贴板可能包含孤立的 UTF-16 surrogate 字符，导致 `json.dumps()` 抛出 `UnicodeEncodeError`。

**解决**: 在消息进入主循环前，使用 `_sanitize_surrogates()` 将其替换为 U+FFFD。同时在重试逻辑中处理 ASCII-only 编码系统（LANG=C 环境）。

#### 10.2.3 Thinking budget exhaustion 误判

**问题**: 模型将所有 output tokens 用于推理，没有剩余 token 生成响应。简单的 "truncated" 判断会触发无意义的 continuation 重试。

**解决**: 检测响应中是否存在 think 标签且标签后无可见内容。如果是，直接返回 "Thinking Budget Exhausted" 错误，而不是尝试 continuation。

### 10.3 性能优化点

1. **Prompt caching**: Anthropic 模型自动启用 prompt caching，减少 ~75% 输入成本
2. **并行工具执行**: 8-worker ThreadPoolExecutor，IO 密集型工具并发运行
3. **持久化事件循环**: 避免每次工具调用创建/销毁循环的开销
4. **系统提示词缓存**: 每个会话只构建一次，后续轮次复用
5. **后台元数据预取**: OpenRouter 模型元数据在后台线程预热
6. **消息规范化**: 每次 API 调用前规范化 whitespace 和 JSON，提高 KV cache 命中率
7. **Ollama num_ctx 自动注入**: 检测模型最大上下文并注入，避免 Ollama 默认的 2048 限制

### 10.4 可以在面试中深入讲解的技术细节

1. **IterationBudget 的线程安全设计**: 使用 `threading.Lock` 保护原子操作，父子 Agent 独立预算
2. **错误分类管线**: 7 层优先级有序的分类逻辑，从提供商特定模式到通用传输层错误
3. **并行工具执行的路径冲突检测**: 前缀匹配算法，保守但安全
4. **同步/异步桥接**: 三种场景（主线程/工作线程/异步上下文）的不同处理策略
5. **上下文自动压缩**: 预压缩 + 运行时压缩 + 多轮压缩的三层策略
6. **凭证池轮换**: 多 Key 轮换 + 提供商级 fallback 链
7. **抖动退避算法**: 独立 Random 实例 + 单调计数器种子，避免惊群效应

---

## 11. 代码片段引用

### 11.1 IterationBudget - 线程安全的迭代预算

```python
# run_agent.py:170-211
class IterationBudget:
    def __init__(self, max_total: int):
        self.max_total = max_total
        self._used = 0
        self._lock = threading.Lock()

    def consume(self) -> bool:
        with self._lock:                      # 原子操作
            if self._used >= self.max_total:
                return False
            self._used += 1
            return True

    def refund(self) -> None:
        with self._lock:
            if self._used > 0:
                self._used -= 1
```

**设计意图**: 所有读写操作都在锁内完成，确保多线程环境下计数器的一致性。`refund()` 的存在是为了 `execute_code` 工具不消耗用户可见的预算。

### 11.2 _sanitize_surrogates - 输入清理

```python
# run_agent.py:340-353
_SURROGATE_RE = re.compile(r'[\ud800-\udfff]')

def _sanitize_surrogates(text: str) -> str:
    """Replace lone surrogate code points with U+FFFD."""
    if _SURROGATE_RE.search(text):           # 快速路径: 无 surrogate 时直接返回
        return _SURROGATE_RE.sub('�', text)
    return text
```

**设计意图**: 预编译正则 + 快速路径判断，确保在正常输入（无 surrogate）时几乎零开销。只在检测到问题字符时才执行替换。

### 11.3 抖动退避算法

```python
# agent/retry_utils.py:19-57
def jittered_backoff(attempt, *, base_delay=5.0, max_delay=120.0, jitter_ratio=0.5):
    # 1. 指数退避计算
    delay = min(base_delay * (2 ** exponent), max_delay)

    # 2. 独立 RNG 种子 (time_ns + 单调计数器)
    seed = (time.time_ns() ^ (tick * 0x9E3779B9)) & 0xFFFFFFFF
    rng = random.Random(seed)                 # 独立实例，线程安全

    # 3. 添加抖动
    jitter = rng.uniform(0, jitter_ratio * delay)
    return delay + jitter
```

**设计意图**: `0x9E3779B9` 是黄金比例的定点表示（来自斐波那契哈希），用于将单调计数器的值分散到更广的范围，避免连续调用产生相似的种子。

### 11.4 _SafeWriter - 管道断裂保护

```python
# run_agent.py:113-159
class _SafeWriter:
    """Transparent stdio wrapper that catches OSError/ValueError."""
    __slots__ = ("_inner",)                  # 最小内存占用

    def write(self, data):
        try:
            return self._inner.write(data)
        except (OSError, ValueError):
            return len(data) if isinstance(data, str) else 0  # 假装成功
```

**设计意图**: 当 Agent 作为 systemd 服务或 Docker 容器运行时，stdout 管道可能在会话中途断开。`_SafeWriter` 确保 print 调用不会因为管道断裂而崩溃，同时在正常终端上完全透明（零开销代理）。

### 11.5 KawaiiSpinner - 异步动画

```python
# agent/display.py:571-756
class KawaiiSpinner:
    def _animate(self):
        if not self._is_tty:                  # 非 TTY: 跳过动画
            self._write(f"  [tool] {self.message}")
            while self.running:
                time.sleep(0.5)
            return

        if self._is_patch_stdout_proxy():     # prompt_toolkit: 避免冲突
            while self.running:
                time.sleep(0.1)
            return

        while self.running:                    # 正常 TTY: 帧动画
            frame = self.spinner_frames[self.frame_idx % len(self.spinner_frames)]
            elapsed = time.time() - self.start_time
            line = f"\r  {frame} {self.message} ({elapsed:.1f}s)"
            self._write(line, end='', flush=True)
            time.sleep(0.12)                   # ~8 FPS

    def start(self):
        self.thread = threading.Thread(target=self._animate, daemon=True)
        self.thread.start()
```

**设计意图**: 三种运行环境（TTY、非TTY、prompt_toolkit）有不同的渲染策略。daemon 线程确保主进程退出时动画线程自动终止。`\r` 回车实现原地刷新，而非逐行输出。

### 11.6 错误分类管线

```python
# agent/error_classifier.py:242-415
def classify_api_error(error, *, provider, model, approx_tokens, context_length):
    # 优先级1: 提供商特定 (thinking_signature, long_context_tier)
    if status_code == 400 and "signature" in error_msg and "thinking" in error_msg:
        return ClassifiedError(reason=FailoverReason.thinking_signature, retryable=True)

    # 优先级2: HTTP 状态码
    if status_code is not None:
        classified = _classify_by_status(status_code, error_msg, ...)
        if classified: return classified

    # 优先级3: 错误码
    # 优先级4: 消息模式匹配
    # 优先级5: 传输层错误
    # 优先级6: 默认 unknown
```

**设计意图**: 优先级有序确保最精确的分类优先匹配。例如，429 + "extra usage" + "long context" 被分类为 `long_context_tier`（需要压缩），而非普通的 `rate_limit`（需要退避）。

---

> **文档版本**: v1.0 | **最后更新**: 2026-05-31 | **基于 commit**: d9055d02
