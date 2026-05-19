# 07 - Agent 内部子系统深度分析

> 学习顺序：第 7 篇 | 前置知识：Prompt Engineering、缓存策略、错误处理

---

## 一、模块概述

Agent 内部子系统由 12 个核心模块组成，形成从 Prompt 构建到错误恢复的完整调用链：

```
┌─────────────────────────────────────────────────────┐
│               AIAgent (run_agent.py)                 │  ← 主循环
├──────────┬──────────┬──────────┬──────────┬──────────┤
│ prompt_  │ smart_   │ context_ │ error_   │ retry_   │  ← 决策层
│ builder  │ model_   │ engine   │ classi-  │ utils    │
│          │ routing  │ .compress│ fier     │          │
├──────────┼──────────┼──────────┼──────────┼──────────┤
│ prompt_  │ auxilia- │ model_   │ usage_   │ display  │  ← 基础层
│ caching  │ ry_      │ metadata │ pricing  │          │
│          │ client   │          │          │          │
├──────────┴──────────┴──────────┴──────────┴──────────┤
│                    trajectory                         │  ← 持久化层
└─────────────────────────────────────────────────────┘
```

### 核心模块职责：
- **prompt_builder**: 系统提示词组装，包括身份、环境信息、平台适配、技能索引、上下文文件加载
- **context_engine**: 上下文管理抽象基类，定义压缩生命周期接口
- **context_compressor**: 默认上下文压缩实现，采用工具输出预修剪+LLM结构化摘要+迭代更新策略
- **prompt_caching**: Anthropic提示词缓存实现，system_and_3策略降低输入成本约75%
- **error_classifier**: API错误分类器，13种错误类型+7级分类流水线，提供精准恢复策略
- **retry_utils**: 抖动退避算法实现，使用黄金比例哈希避免惊群效应
- **smart_model_routing**: 智能模型路由，简单请求自动分配到廉价模型
- **auxiliary_client**: 辅助LLM调用客户端，统一接口适配不同提供商
- **model_metadata**: 模型元数据管理，上下文长度查询、Token估算
- **usage_pricing**: 成本精确计算，使用Decimal避免浮点误差，多数据源价格查询链
- **display**: 输出展示适配不同平台
- **trajectory**: 会话轨迹持久化

---

## 二、Prompt 构建系统

### 2.1 系统 Prompt 组装顺序

`_build_system_prompt()` 按以下顺序组装约 15 层（某些层有条件注入）：

```
Layer 1:  Agent identity — SOUL.md（优先）或 DEFAULT_AGENT_IDENTITY
Layer 2:  Tool behavioral guidance — memory/session_search/skills_manage 引导
          （仅当对应工具被加载时注入，用空格连接为单一段落）
Layer 3:  Nous subscription prompt — 为订阅路由提供模型定制化提示词
          （build_nous_subscription_prompt()，仅在对应工具加载时注入）
Layer 4:  Tool-use enforcement guidance — 强制模型实际调用工具而非描述意图
          （配置控制："auto"/true/false/模型名列表）
Layer 5:  Google model operational guidance — 简洁输出、绝对路径、并行工具
          调用、先验后编、非交互命令（仅 Gemini/Gemma 模型）
Layer 6:  OpenAI/GPT model execution discipline — 工具持久化、前置检查、
          验证反幻觉（仅 GPT/Codex 模型）
Layer 7:  User system message — 用户或网关传入的自定义系统提示词（可选）
Layer 8:  Built-in memory store — memory 和 USER.md profile
          （由 _memory_store.format_for_system_prompt() 生成）
Layer 9:  External memory provider block — 外部记忆管理器产出的系统提示词
          （_memory_manager.build_system_prompt()，附加到内置记忆之上）
Layer 10: Skills system prompt — 技能索引（仅当 skills_list/skill_view/
          skill_manage 中至少一个工具被加载）
Layer 11: Context files — .hermes.md / AGENTS.md / CLAUDE.md / .cursorrules
          （优先级互斥，SOUL.md 已加载时排除 .hermes.md）
Layer 12: Conversation timestamp — 冻结的会话开始时间 + Session ID + Model +
          Provider 元数据，以 "Conversation started: ..." 开头
Layer 13: Alibaba model identity workaround — 阿里 Coding Plan API 总返回
          "glm-4.7" 为模型名，此处强制注入正确的模型标识（仅阿里 provider）
Layer 14: Environment hints — WSL、Termux 等执行环境检测
Layer 15: Platform-specific formatting hints — WhatsApp/Telegram/Discord/...
          的输出格式适配
```

**关键规则**：
- **SOUL.md 优先**：存在时替换默认身份（非追加）。
- **条件注入**：工具引导和模型特定层仅在条件匹配时注入，避免浪费 token。
- **一次性构建**：`_cached_system_prompt` 在会话首轮构建后缓存，跨轮次复用以保证 Anthropic prefix cache 命中。仅在上下文压缩事件后重建。
- **ephemeral_system_prompt 不入缓存**：临时提示词在 API 调用时以追加方式注入到 effective system prompt，不进入 `_cached_system_prompt` 也不持久化到轨迹数据。

### 2.2 上下文文件的优先级互斥

```python
project_context = (
    _load_hermes_md(cwd_path)      # 优先级 1
    or _load_agents_md(cwd_path)   # 优先级 2
    or _load_claude_md(cwd_path)   # 优先级 3
    or _load_cursorrules(cwd_path) # 优先级 4
)
```

只加载第一个匹配的项目上下文类型。

### 2.3 Prompt Injection 防护

`_scan_context_content()` 在加载上下文文件前扫描：
- 不可见 Unicode 字符（零宽字符、BiDi 控制符）
- 10 种注入模式（`ignore previous instructions`、`do not tell the user` 等）
- 发现威胁返回 `"[BLOCKED: ...]"`

**关键实现代码**：
```python
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'disregard\s+(your|all|any)\s+(instructions|rules|guidelines)', "disregard_rules"),
    (r'act\s+as\s+(if|though)\s+you\s+(have\s+no|don\'t\s+have)\s+(restrictions|limits|rules)', "bypass_restrictions"),
    (r'<!--[^>]*(?:ignore|override|system|secret|hidden)[^>]*-->', "html_comment_injection"),
    (r'<\s*div\s+style\s*=\s*["\'][\s\S]*?display\s*:\s*none', "hidden_div"),
    (r'translate\s+.*\s+into\s+.*\s+(execute|run|eval)', "translate_execute"),
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL|API)', "exfil_curl"),
    (r'cat\s+[^\n]*(\.env|credentials|\.netrc|\.pgpass)', "read_secrets"),
]

_CONTEXT_INVISIBLE_CHARS = {
    '​', '‌', '‍', '⁠', '﻿',
    '‪', '‫', '‬', '‭', '‮',
}
```

### 2.4 模型特定指导

- **GPT/Codex**：`<tool_persistence>`、`<mandatory_tool_use>`、`<verification>` XML 标签
- **Gemini**：绝对路径、并行工具调用、非交互命令

### 2.5 Skills 索引的两层缓存

```
Layer 1: 进程内 LRU 缓存（OrderedDict, 最大 8 条）
Layer 2: 磁盘快照（.skills_prompt_snapshot.json, mtime/size manifest 验证）
Layer 3: 冷启动 — 全文件系统扫描
```

**缓存机制实现**：
```python
# 进程内LRU缓存
_SKILLS_PROMPT_CACHE_MAX = 8
_SKILLS_PROMPT_CACHE: OrderedDict[tuple, str] = OrderedDict()
_SKILLS_PROMPT_CACHE_LOCK = threading.Lock()

# 磁盘快照验证
def _load_skills_snapshot(skills_dir: Path) -> Optional[dict]:
    """Load the disk snapshot if it exists and its manifest still matches."""
    snapshot_path = _skills_prompt_snapshot_path()
    if not snapshot_path.exists():
        return None
    try:
        snapshot = json.loads(snapshot_path.read_text(encoding="utf-8"))
    except Exception:
        return None
    if not isinstance(snapshot, dict):
        return None
    if snapshot.get("version") != _SKILLS_SNAPSHOT_VERSION:
        return None
    if snapshot.get("manifest") != _build_skills_manifest(skills_dir):
        return None
    return snapshot
```

---

## 三、上下文管理

### 3.1 ContextEngine 生命周期

**基本调用链**：
```
on_session_start → update_from_response → should_compress → compress → on_session_end
```

**完整插件生命周期**（8 个可选的 hook 方法）：

| Hook | 调用时机 | 用途 |
|------|---------|------|
| `on_session_start(session_id)` | 新会话开始 | 加载持久化状态（DAG、store）、接受 hermes_home/platform/model 等参数 |
| `should_compress_preflight(messages)` | API 调用前的快速预算检查 | 无真实 token 数时的廉价预检（默认返回 False） |
| `update_from_response(usage)` | 每次 API 响应后 | 更新 last_prompt_tokens / last_completion_tokens |
| `should_compress(prompt_tokens)` | 决定是否触发压缩 | 比较当前 token 数与 threshold_tokens |
| `compress(messages, current_tokens)` | 执行压缩 | 返回压缩后的消息列表 |
| `on_session_reset()` | /new 或 /reset | 重置 per-session 状态（compression_count、token tracking） |
| `on_session_end(session_id, messages)` | 真实会话边界 | 持久化状态、关闭 DB 连接（非每次轮询触发） |
| `get_tool_schemas()` | 工具注册 | 引擎提供的工具 schema（LCM 返回 lcm_grep/lcm_describe/lcm_expand） |
| `handle_tool_call(name, args)` | 工具调用 | 处理引擎注册的工具调用（仅处理 get_tool_schemas() 返回的工具名） |
| `get_status()` | 状态展示 | 返回标准字段（last_prompt_tokens, threshold_tokens, usage_percent 等） |
| `update_model(model, context_length)` | 模型切换 / fallback 激活 | 更新 context_length 并重算 threshold_tokens |



**ContextEngine抽象基类定义**：
```python
class ContextEngine(ABC):
    """Base class all context engines must implement."""
    
    @property
    @abstractmethod
    def name(self) -> str:
        """Short identifier (e.g. 'compressor', 'lcm')."""
    
    # Token state
    last_prompt_tokens: int = 0
    last_completion_tokens: int = 0
    last_total_tokens: int = 0
    threshold_tokens: int = 0
    context_length: int = 0
    compression_count: int = 0
    
    # Core interface
    @abstractmethod
    def update_from_response(self, usage: Dict[str, Any]) -> None:
        """Update tracked token usage from an API response."""
    
    @abstractmethod
    def should_compress(self, prompt_tokens: int = None) -> bool:
        """Return True if compaction should fire this turn."""
    
    @abstractmethod
    def compress(
        self,
        messages: List[Dict[str, Any]],
        current_tokens: int = None,
    ) -> List[Dict[str, Any]]:
        """Compact the message list and return the new message list."""
```

### 3.2 压缩算法（5 阶段）

1. **工具输出预修剪**（无 LLM 调用）：MD5 去重 + 信息性单行摘要
2. **确定保护边界**：head + token-budget tail
3. **LLM 摘要生成**：结构化模板（Goal/Progress/Decisions/Files）
4. **组装**：head + summary + tail
5. **孤立 tool_call/tool_result 对清理**

**工具输出预修剪实现**：
```python
def _prune_old_tool_results(
    self, messages: List[Dict[str, Any]], protect_tail_count: int,
    protect_tail_tokens: int | None = None,
) -> tuple[List[Dict[str, Any]], int]:
    """Replace old tool result contents with informative 1-line summaries."""
    # 1. 构建tool_call_id到工具信息的映射
    call_id_to_tool: Dict[str, tuple] = {}
    for msg in result:
        if msg.get("role") == "assistant":
            for tc in msg.get("tool_calls") or []:
                # 提取工具名称和参数
    
    # 2. 确定修剪边界（基于Token预算或消息数量）
    if protect_tail_tokens is not None and protect_tail_tokens > 0:
        # 从后向前累计Token，确定保护边界
        accumulated = 0
        boundary = len(result)
        min_protect = min(protect_tail_count, len(result) - 1)
        for i in range(len(result) - 1, -1, -1):
            # 计算消息Token数
            if accumulated + msg_tokens > protect_tail_tokens and (len(result) - i) >= min_protect:
                boundary = i
                break
            accumulated += msg_tokens
            boundary = i
        prune_boundary = max(boundary, len(result) - min_protect)
    
    # 3. MD5去重相同工具结果
    content_hashes: dict = {}  # hash -> (index, tool_call_id)
    for i in range(len(result) - 1, -1, -1):
        # 计算内容哈希，重复的替换为引用
    
    # 4. 替换旧工具结果为摘要
    for i in range(prune_boundary):
        # 生成工具结果摘要：[terminal] ran `npm test` -> exit 0, 47 lines output
```

**结构化摘要模板**：
```
## Goal
[用户目标]
## Constraints & Preferences
[用户偏好、约束、重要决策]
## Completed Actions
[已完成动作的编号列表，包含工具、目标、结果]
## Active State
[当前工作状态：目录、分支、修改文件、测试状态等]
## In Progress
[进行中的工作]
## Blocked
[未解决的阻塞问题]
## Key Decisions
[重要技术决策及原因]
## Resolved Questions
[已回答的用户问题]
## Pending User Asks
[未回答的用户请求]
## Relevant Files
[相关文件列表及说明]
## Remaining Work
[剩余工作]
## Critical Context
[需要保留的关键上下文：错误信息、配置值等]
```

### 3.3 反抖动保护

```python
if self._ineffective_compression_count >= 2:
    return False  # 最近两次压缩各自节省不到 10%
```

**压缩有效性检查逻辑**：
```python
def should_compress(self, prompt_tokens: int = None) -> bool:
    """Check if context exceeds the compression threshold.
    
    Includes anti-thrashing protection: if the last two compressions
    each saved less than 10%, skip compression to avoid infinite loops
    where each pass removes only 1-2 messages.
    """
    tokens = prompt_tokens if prompt_tokens is not None else self.last_prompt_tokens
    if tokens < self.threshold_tokens:
        return False
    # Anti-thrashing: back off if recent compressions were ineffective
    if self._ineffective_compression_count >= 2:
        if not self.quiet_mode:
            logger.warning(
                "Compression skipped — last %d compressions saved <10%% each. "
                "Consider /new to start a fresh session, or /compress <topic> "
                "for focused compression.",
                self._ineffective_compression_count,
            )
        return False
    return True
```

### 3.4 迭代摘要更新 (Iterative Summary)

首次压缩从零生成摘要；后续压缩**增量更新**已有摘要：

```python
# _generate_summary() 检测 _previous_summary 存在时：
if self._previous_summary:
    prompt = f"""PREVIOUS SUMMARY: {self._previous_summary}
    NEW TURNS TO INCORPORATE: {content_to_summarize}
    Update the summary... PRESERVE... ADD... Move items..."""
```

新轮次被合并到已有摘要中，而非替代。保持编号连续性、动态移动 In Progress→Completed Actions。

### 3.5 聚焦主题压缩 (Focus Topic)

通过 `focus_topic` 参数实现 Claude Code `/compact <topic>` 风格的主题聚焦压缩：

```python
FOCUS TOPIC: "{focus_topic}"
The user has requested that this compaction PRIORITISE preserving all
information related to the focus topic above... The focus topic sections
should receive roughly 60-70% of the summary token budget.
```

非焦点内容被更激进地压缩（一行摘要或直接省略），焦点内容保留完整细节。

### 3.6 摘要失败容错

**三层容错机制**：

1. **冷却期**：摘要生成失败后进入冷却（RuntimeError=600s，一般异常=60s），在此期间跳过摘要生成。
2. **模型降级**：独立的 `summary_model` 失败后，尝试回退到主模型。若回退成功，重置冷却期。
3. **静态 fallback**：摘要完全失败时插入 handoff 标记 `"[CONTEXT COMPACTION — REFERENCE ONLY]"`，告知模型上下文已丢失。

**冷却期区分**：
- 瞬时错误（网络、超时）：60 秒短冷却
- 硬性错误（模型未找到、503、无可用 channel、`summary_model` 回退失败）：600 秒长冷却

### 3.7 工具输出优化细节

**MD5 去重**：相同工具结果的 MD5（前 12 位）被追踪，重复项替换为引用标记：
```python
h = hashlib.md5(content.encode("utf-8", errors="replace")).hexdigest()[:12]
if h in content_hashes:
    msg["content"] = f"[Same result as earlier tool call..."
```

**信息性单行摘要**：工具结果被压缩为描述性摘要而非通用占位符：
- `terminal: ran 'npm test' -> exit 0, 47 lines output`
- `read_file: read config.py from line 1 (3,400 chars)`

**3-pass 工具输出修剪**：
1. 从后向前扫描，按 token 预算确定修剪边界
2. 在修剪边界内执行 MD5 去重
3. 将修剪边界外的旧工具结果替换为信息性单行摘要

### 3.8 Token 估算

```python
def estimate_tokens_rough(text: str) -> int:
    return (len(text) + 3) // 4  # ~4 chars/token
```

上下文长度解析链（10 级）：配置 → 缓存 → 端点元数据 → 本地服务器 → API → 注册表 → 默认值。

---

## 四、对话历史管理 (Message Lifecycle)

### 4.1 预调用消毒器 (`_sanitize_api_messages()`)

每次 LLM API 调用前**无条件执行**的消息清洗函数：

**Role 白名单过滤**：`_VALID_API_ROLES = frozenset({"system", "user", "assistant", "tool", "function", "developer"})`。任何不在该集合中的角色消息被直接丢弃。

**孤立 tool_call/tool_result 对清理**：
1. 收集所有 assistant 消息中的 tool_call id → 构成"存活"集合
2. 收集所有 tool 消息的 tool_call_id → 构成"结果"集合
3. 删除存活集合中没有的工具结果（step 1）
4. 删除未完成（无响应工具结果）的工具调用（step 2）
5. 截断末尾的未完成工具调用（step 3）

**调用时机**：在代码内联转化前（`api_messages = self._sanitize_api_messages(api_messages)`），确保历史回放、会话加载、手动消息注入等所有路径都被清洗。

### 4.2 Todo 状态水合 (`_hydrate_todo_store()`)

Gateway 模式下每个消息创建新的 AIAgent 实例，内存中的 TodoStore 为空。此方法从对话历史中恢复状态：

- 逆序扫描历史消息，查找最近一条包含 `"todos"` 键的 tool 消息
- 解析 JSON 并以 `replace` 模式重放到 `_todo_store`
- 确保跨请求的 todo 状态不丢失

### 4.3 会话持久化 (`_flush_messages_to_session_db()`)

- 将 `_session_messages` 写入 SQLite 会话数据库
- 压缩后的消息列表自动更新到存储（`_flush_messages_to_session_db writes compressed`）
- 上下文溢出错误（HTTP 400 + 大会话）跳过持久化，避免增长死循环
- 支持 `persist_session=False` 的临时会话（不写入数据库）

### 4.4 消息字段清理

API 调用前自动清洗内部字段：
- 移除 `_conversation_history_index`（内部索引标记）
- 移除 `_thinking_prefill`（预填充标记）
- 调用 `_sanitize_tool_calls_for_strict_api()` 确保工具调用符合严格格式要求
- 历史保留字段：`reasoning_details`（OpenRouter 多轮推理上下文）、`codex_reasoning_items`（Codex 加密推理项）

---

## 五、Prompt 缓存

### 5.1 Anthropic Prompt Caching

`system_and_3` 策略：最多 4 个 cache_control 断点：
1. System prompt（跨所有回合稳定）
2-4. 最后 3 条非 system 消息（滚动窗口）

热门前缀读取成本降低 ~75%。

**核心实现代码**：
```python
def apply_anthropic_cache_control(
    api_messages: List[Dict[str, Any]],
    cache_ttl: str = "5m",
    native_anthropic: bool = False,
) -> List[Dict[str, Any]]:
    """Apply system_and_3 caching strategy to messages for Anthropic models.
    
    Places up to 4 cache_control breakpoints: system prompt + last 3 non-system messages.
    """
    messages = copy.deepcopy(api_messages)
    if not messages:
        return messages

    marker = {"type": "ephemeral"}
    if cache_ttl == "1h":
        marker["ttl"] = "1h"

    breakpoints_used = 0

    if messages[0].get("role") == "system":
        _apply_cache_marker(messages[0], marker, native_anthropic=native_anthropic)
        breakpoints_used += 1

    remaining = 4 - breakpoints_used
    non_sys = [i for i in range(len(messages)) if messages[i].get("role") != "system"]
    for idx in non_sys[-remaining:]:
        _apply_cache_marker(messages[idx], marker, native_anthropic=native_anthropic)

    return messages
```

---

## 六、错误处理

### 6.1 错误分类体系

13 种 `FailoverReason` 枚举，按优先级排序的 7 级分类流水线：
1. Provider 特定模式（最高优先级）
2. HTTP 状态码分类 + 消息感知细化
3. 结构化错误码分类
4. 消息模式匹配
5. 服务器断开 + 大会话 → 上下文溢出
6. 传输/超时启发式
7. 回退：unknown（可重试）

**FailoverReason枚举定义**：
```python
class FailoverReason(enum.Enum):
    """Why an API call failed — determines recovery strategy."""
    # Authentication / authorization
    auth = "auth"                        # Transient auth (401/403) — refresh/rotate
    auth_permanent = "auth_permanent"    # Auth failed after refresh — abort
    # Billing / quota
    billing = "billing"                  # 402 or confirmed credit exhaustion — rotate immediately
    rate_limit = "rate_limit"            # 429 or quota-based throttling — backoff then rotate
    # Server-side
    overloaded = "overloaded"            # 503/529 — provider overloaded, backoff
    server_error = "server_error"        # 500/502 — internal server error, retry
    # Transport
    timeout = "timeout"                  # Connection/read timeout — rebuild client + retry
    # Context / payload
    context_overflow = "context_overflow"  # Context too large — compress, not failover
    payload_too_large = "payload_too_large"  # 413 — compress payload
    # Model
    model_not_found = "model_not_found"  # 404 or invalid model — fallback to different model
    # Request format
    format_error = "format_error"        # 400 bad request — abort or strip + retry
    # Provider-specific
    thinking_signature = "thinking_signature"  # Anthropic thinking block sig invalid
    long_context_tier = "long_context_tier"    # Anthropic "extra usage" tier gate
    # Catch-all
    unknown = "unknown"                  # Unclassifiable — retry with backoff
```

### 6.2 402 错误消歧义

区分真正计费耗尽和伪装的瞬时限速：
```python
has_usage_limit = any(p in error_msg for p in _USAGE_LIMIT_PATTERNS)
has_transient_signal = any(p in error_msg for p in _USAGE_LIMIT_TRANSIENT_SIGNALS)
if has_usage_limit and has_transient_signal:
    return FailoverReason.rate_limit  # 瞬时配额，非计费
```

**模式匹配规则**：
```python
# Patterns that indicate billing exhaustion (not transient rate limit)
_BILLING_PATTERNS = [
    "insufficient credits",
    "insufficient_quota",
    "credit balance",
    "credits have been exhausted",
    "top up your credits",
    "payment required",
    "billing hard limit",
    "exceeded your current quota",
    "account is deactivated",
    "plan does not include",
]

# Patterns that indicate rate limiting (transient, will resolve)
_RATE_LIMIT_PATTERNS = [
    "rate limit",
    "rate_limit",
    "too many requests",
    "throttled",
    "requests per minute",
    "tokens per minute",
    "requests per day",
    "try again in",
    "please retry after",
    "resource_exhausted",
]

# Usage-limit patterns that need disambiguation (could be billing OR rate_limit)
_USAGE_LIMIT_PATTERNS = [
    "usage limit",
    "quota",
    "limit exceeded",
    "key limit exceeded",
]

# Patterns confirming usage limit is transient (not billing)
_USAGE_LIMIT_TRANSIENT_SIGNALS = [
    "try again",
    "retry",
    "resets at",
    "reset in",
    "wait",
    "requests remaining",
    "periodic",
    "window",
]
```

### 6.3 Jittered Backoff

```python
def jittered_backoff(attempt, base_delay=5.0, max_delay=120.0, jitter_ratio=0.5):
    delay = min(base_delay * 2^(attempt-1), max_delay)
    seed = (time.time_ns() ^ (tick * 0x9E3779B9)) & 0xFFFFFFFF
    rng = random.Random(seed)
    jitter = rng.uniform(0, jitter_ratio * delay)
    return delay + jitter
```

`0x9E3779B9` 是黄金比例的 32 位定点表示（Fibonacci Hashing），提供良好的哈希分布，避免惊群效应。
- **指数退避**：重试延迟随尝试次数指数增长，减少服务器压力
- **随机抖动**：在基础延迟上添加随机偏移，避免大量请求同时重试
- **黄金比例种子**：确保即使在纳秒级时间精度下，不同请求也能生成不同的随机序列

### 6.4 错误恢复循环 (Full Recovery Loop)

错误发生后，Agent 进入多层恢复循环，每个 `continue` 对应一种恢复策略：

```python
# ClassifiedError 包含恢复决策字段：
class ClassifiedError:
    reason: FailoverReason    # 错误类型
    retryable: bool           # 是否可重试
    should_compress: bool     # 是否应触发上下文压缩
    should_rotate_credential: bool  # 是否应切换凭证
    should_fallback: bool     # 是否应切换到备用 provider
```

**恢复优先级链**（按尝试顺序）：

1. **Credential pool 恢复** — 429 (rate limit) 或 billing 错误时从凭证池获取下一个 key
2. **Codex OAuth 刷新** — 401 + codex_responses 模式下强制刷新 Codex 凭证
3. **Nous 凭证刷新** — 401 + nous provider 下强制刷新 Nous agent key
4. **Anthropic OAuth 刷新** — 401 + anthropic_messages 模式下刷新 Bearer token
5. **Thinking signature 恢复** — thinking_signature 错误，从所有消息中剥离 `reasoning_details`
6. **上下文压缩** — `should_compress=True` 时触发压缩（最多 2 次）
7. **超长上下文特殊处理** — 无法进一步压缩时返回 partial + `compression_exhausted=True`
8. **Provider fallback** — 非可重试错误或重试耗尽后，尝试 `_try_activate_fallback()` 切换到备用 provider
9. **Primary transport 恢复** — 重建 HTTP 客户端（过期连接池、TCP reset 等传输层问题）

**重试跟踪变量**：
- `retry_count` — 同一 provider 下的重试次数
- `compression_attempts` — 压缩尝试次数（最多 2）
- `anthropic_auth_retry_attempted` / `codex_auth_retry_attempted` / `nous_auth_retry_attempted` — 各 provider 认证重试标记
- `thinking_sig_retry_attempted` — thinking signature 恢复标记（单次）
- `primary_recovery_attempted` — transport 层恢复标记（单次）

---

## 七、多 Provider 消息格式转换 (Message Format Adapters)

Agent 内部统一使用 OpenAI chat.completions 格式存储消息，但实际 LLM 调用涉及三种不同的 API 协议。消息格式转换在 API 边界完成，内部表示始终保持一致。

### 7.1 三种 API 路径

| API 模式 | Provider | 转换方向 | 关键模块 |
|---------|----------|---------|---------|
| `chat_completions` | OpenAI / OpenRouter / Nous / 自定义 | OpenAI 原生格式，无需转换 | - |
| `anthropic_messages` | Anthropic (Direct / OpenCode) | OpenAI → Messages API | `agent/anthropic_adapter.py` |
| `bedrock_converse` | AWS Bedrock | OpenAI → Converse API | `agent/bedrock_adapter.py` (1098 行) |
| `codex_responses` | OpenAI Codex CLI | 特殊 Responses API 调度 | `run_agent.py` 内联 Codex 路由 |

### 7.2 Anthropic Messages API 适配

`anthropic_adapter.py` 负责完整的双向转换：

- **消息格式**：system 消息提取为独立的 `system` 参数；user/assistant 消息转为 `content` 块列表（text + tool_use + tool_result 块）
- **工具格式**：OpenAI function 格式 → Anthropic tool_use 格式（input_schema JSON Schema）
- **Cache Control**：在消息和 system prompt 内容块末尾注入 `{"type": "ephemeral"}` 标记，实现 `system_and_3` 缓存策略
- **Thinking 配置**：Claude 4.6+ 使用 adaptive thinking (`effort: "max"/"high"/"medium"/"low"`)；旧模型使用 manual thinking (`budget_tokens`)
- **输出限制**：按模型维护 `_ANTHROPIC_OUTPUT_LIMITS` 表（16K-128K），确保 max_tokens 不低于 thinking budget

### 7.3 Bedrock Converse API 适配

`bedrock_adapter.py` 是最大的适配器（1098 行），核心能力：

- `convert_messages_to_converse()` — 将 OpenAI 消息列表转为 Converse 格式（role + content 块）
- `convert_tools_to_converse()` — 工具 schema 转换（含自动检测不支持工具的模型并跳过 toolConfig）
- `build_converse_kwargs()` — 组装完整的 Converse API 参数（system/messages/inferenceConfig/toolConfig/guardrailConfig）
- `normalize_converse_response()` — 将 Converse 响应转回 OpenAI 兼容格式（含 reasoning token 提取、toolUse 块转换）
- `stream_converse_with_callbacks()` — 流式版本，支持实时 delta 回调 + 中断检查

### 7.4 Developer Role 角色交换 (GPT-5/Codex)

```python
DEVELOPER_ROLE_MODELS = ("gpt-5", "codex")
```

对于 GPT-5 和 Codex 模型，system 角色在 API 边界被映射为 `developer`：

```python
if (
    sanitized_messages
    and sanitized_messages[0].get("role") == "system"
    and any(p in _model_lower for p in DEVELOPER_ROLE_MODELS)
):
    sanitized_messages[0] = {**sanitized_messages[0], "role": "developer"}
```

**原理**：GPT-5/Codex 模型遵循 OpenAI 的 "developer" 角色语义——`system` 用于平台级控制，`developer` 用于应用级指令。内部始终用 `system` 保持统一，仅在 API 边界做浅拷贝替换（不修改原始消息对象）。`_VALID_API_ROLES` 中包含 `"developer"` 以允许该角色在消息历史中存在。

### 7.5 Anthropic OAuth 凭证管理

`anthropic_adapter.py` 支持三种认证方式，自动检测并路由：

| 认证方式 | Token 前缀 | Auth Header | 来源 |
|---------|-----------|-------------|------|
| API Key | `sk-ant-api` | `x-api-key` | Console API key |
| OAuth Setup Token | `sk-ant-oat` | `Bearer` + beta header | Claude Code 初始化 |
| Claude Code Credentials | JWT (`eyJ`) | `Bearer` + version header | `~/.claude/.credentials.json` |

**OAuth 刷新流程** (`_resolve_claude_code_token_from_credentials()`)：

1. 读取 `~/.claude/.credentials.json` 中的 `claudeAiOauth` 对象（含 `accessToken`、`refreshToken`、`expiresAt`）
2. 验证 token 是否有效（过期时间预留 60 秒缓冲）：`now_ms < (expiresAt - 60_000)`
3. 如已过期，调用 Anthropic OAuth token 端点刷新：
   - 端点：`platform.claude.com/v1/oauth/token`（主）→ `console.anthropic.com/v1/oauth/token`（备用）
   - Client ID：`9d1c250a-e61b-44d9-88ed-5944d1962f5e`
   - 支持 JSON 和 URL-encoded 两种请求格式
4. 刷新成功后将新凭证写回 `~/.claude/.credentials.json`（权限 600），保留已有的 `scopes` 字段
5. 优先使用 refreshable credential file 而非静态环境变量（当两者都存在且 ENV token 是 OAuth 类型时，优先文件源以便后续 refresh）

**Claude Code 身份**：OAuth 请求需要携带 Claude Code 版本号的 User-Agent，Anthropic 基础设施根据版本号验证和路由 OAuth 流量。

---

## 八、模型路由

`choose_cheap_model_route()` 的判断条件（全部满足才路由到廉价模型）：
```python
len(text) <= 160          # max_simple_chars
len(text.split()) <= 28   # max_simple_words
text.count("\n") <= 1
"```" not in text
"`" not in text
not _URL_RE.search(text)
not (words & _COMPLEX_KEYWORDS)  # 36 个复杂关键词
```

---

## 九、成本管理

使用 `Decimal` 精确计算（避免浮点误差）。价格查询链：
1. 订阅路由 → 零价格
2. OpenRouter → 实时 API
3. 自定义端点 → endpoint /models metadata
4. 官方文档快照 → 硬编码表

**成本计算特点**：
- **精确算术**：使用Python Decimal类型进行货币计算，避免浮点精度问题
- **多层降级**：价格查询失败时自动降级到下一个数据源
- **实时更新**：支持从OpenRouter等平台获取实时价格数据
- **离线可用**：内置常用模型价格快照，无网络时也能正常估算成本

---

## 十、辅助 LLM 调用

**Provider 解析链**：
- 文本任务：OpenRouter → Nous Portal → 自定义端点 → Codex OAuth → Anthropic → None
- 视觉任务：主 provider → OpenRouter → Nous Portal → Codex OAuth → Anthropic → None

**适配器模式**：`_CodexCompletionsAdapter` 和 `_AnthropicCompletionsAdapter` 将各自 API 适配为 OpenAI chat.completions 接口。

**设计优势**：
- **统一接口**：所有辅助LLM调用使用相同的OpenAI兼容接口，上层业务无需关心底层提供商
- **自动降级**：某个提供商不可用时自动尝试下一个，提高可用性
- **任务适配**：不同类型任务（文本/视觉）使用最优的解析链
- **可扩展性**：新增提供商只需实现对应的适配器，无需修改上层代码

---

## 十一、流式处理 (Streaming)

### 11.1 回调体系

Agent 注册 4 种流式回调，在 API 调用时逐 token 传递实时数据：

| 回调 | 触发时机 | 用途 |
|------|---------|------|
| `thinking_callback` | 收到推理 token（Thinking） | 展示模型思考过程（默认隐藏） |
| `reasoning_callback` | 完整推理文本提取完毕 | CLI 后处理展示推理摘要 |
| `stream_delta_callback` | 收到文本 token（Delta） | 实时逐字输出、TUI 更新 |
| `interim_assistant_callback` | 流式响应的中间快照 | 网关中间消息推送 |

此外还有 `tool_gen_callback`（工具参数开始生成时触发一次，用于显示 spinner）。

### 11.2 Think 块剥离 (`_strip_think_blocks()`)

处理 **6 种标签变体**（支持大小写不敏感匹配）：

```python
def _strip_think_blocks(self, content: str) -> str:
    content = re.sub(r'<think>.*?</think>', '', content, flags=re.DOTALL)
    content = re.sub(r'<thinking>.*?</thinking>', '', content, flags=re.DOTALL | re.IGNORECASE)
    content = re.sub(r'<reasoning>.*?</reasoning>', '', content, flags=re.DOTALL)
    content = re.sub(r'<REASONING_SCRATCHPAD>.*?</REASONING_SCRATCHPAD>', '', content, flags=re.DOTALL)
    content = re.sub(r'<thought>.*?</thought>', '', content, flags=re.DOTALL | re.IGNORECASE)
    content = re.sub(r'</?(?:think|thinking|reasoning|thought|REASONING_SCRATCHPAD)>\s*', '', content, flags=re.IGNORECASE)
    return content
```

此方法在以下场景被调用：检查中间确认消息（Codex ack）、检查空响应、提取可见文本给用户、interim assistant 回调前过滤推理内容。

### 11.3 三种 API 模式下的流式处理

**chat_completions** (OpenAI 兼容)：
- `stream=True`，SSE 增量解析
- `_stream_callback` 逐 token 调用
- 支持中断检查 `_interrupt_requested`

**anthropic_messages** (Anthropic SDK)：
- `client.messages.stream()` 原生流式
- `_on_reasoning` 回调处理推理 delta
- `_fire_stream_delta()` 逐文本 token 推送
- `get_final_message()` 获取完整响应（含 thinking blocks、tool use）

**bedrock_converse** (AWS Bedrock)：
- `client.converse_stream()` + `stream_converse_with_callbacks()`
- 独立线程运行，通过回调传递 delta
- `_on_text` / `_on_tool` / `_on_reasoning` 分别处理
- `on_interrupt_check` 支持中断

### 11.4 截断响应续写 (Truncation Continuation)

当 `finish_reason='length'`（模型因 max_tokens 限制截断输出）：

1. **判定是否可续写**：检测 thinking-budget exhaustion（模型用光了输出 token 在推理上，无可见内容 → 直接报错不续写）
2. **最多 3 次续写重试**：追加 `"[System: Your previous response was truncated...]"` 消息，要求模型从断点继续
3. **3 次后仍截断则放弃**：返回部分响应 + `partial=True`
4. **截断工具调用特殊处理**：只重试 1 次（截断的工具参数不可增量恢复）
5. **代码路径**：`restart_with_length_continuation` 控制外层循环重新发送

### 11.5 Thinking-Prefill 续写

当模型返回结构化推理（API 字段 `reasoning`/`reasoning_content`/`reasoning_details`）但没有可见文本内容时：
- 追加 assistant 消息（标记 `_thinking_prefill=True`）到对话历史
- 最多重试 2 次
- 模型在下一次 API 调用中看到自己的推理内容，继续生成文本

---

## 十二、思考/推理 Token 处理 (Thinking & Reasoning)

### 12.1 Thinking 架构概述

支持三种 thinking/reasoning 路径，在 `_build_assistant_message()` 中统一处理：

1. **结构化推理字段**（Anthropic 原生）：`assistant_message.reasoning` / `reasoning_content` / `reasoning_details` — API 直接返回的推理文本
2. **内联 `<think>` 块**（Gemini、GLM 等模型）：从 `content` 中由 `_strip_think_blocks()` 正则提取
3. **OpenRouter 统一格式**：`reasoning_details` 数组，每项包含 `{type, summary, ...}`

### 12.2 Thinking 块签名管理

Anthropic 对 thinking block 进行加密签名，签名与完整的回合内容绑定。任何上游修改（上下文压缩、会话截断、消息合并）都会使签名失效，导致 HTTP 400 错误。

**恢复策略**（`thinking_signature` 错误处理）：
- 单次恢复尝试：从所有消息中移除 `reasoning_details` 字段
- 失败后不再重试（避免无限循环）
- 日志记录剥离的消息数量

### 12.3 Adaptive Thinking vs Manual Budget

```python
ADAPTIVE_EFFORT_MAP = {"xhigh": "max", "high": "high", "medium": "medium", "low": "low", "minimal": "low"}
THINKING_BUDGET = {"xhigh": 32000, "high": 16000, "medium": 8000, "low": 4000}
```

- **Claude 4.6+ / `_supports_adaptive_thinking()`**：使用 `thinking: {type: "enabled", effort: "max"/"high"/...}` ，让模型自主决定推理深度
- **旧模型**：使用 `thinking: {type: "enabled", budget_tokens: N}` ，预分配固定推理预算
- **Thinking budget 包含在 max_tokens 中**：必须确保 max_tokens > thinking budget，否则响应被截断

### 12.4 Thinking Token 追踪

`session_reasoning_tokens` 累积每轮 API 调用中的推理 token 用量，在会话结果中返回：

```python
self.session_reasoning_tokens += canonical_usage.reasoning_tokens
```

### 12.5 第三方端点 Thinking 块剥离

对于不支持 thinking 块的第三方端点（如部分 OpenRouter 提供商），在发送消息前需要剥离 `reasoning_details` 以避免格式错误。此剥离发生在 thinking_signature 恢复流程中，复用同一逻辑。

---

## 十三、临时系统提示词 (Ephemeral System Prompt)

### 13.1 安全模型

`ephemeral_system_prompt` 是一种**一次性注入**的额外系统提示词，与缓存的系统提示词（`_cached_system_prompt`）完全隔离：

```python
# 构建阶段：ephemeral 显式不包括在 _cached_system_prompt 中
# _build_system_prompt() 注释：
# "Note: ephemeral_system_prompt is NOT included here. It's injected at
#  API-call time only so it stays out of the cached/stored system prompt."

# 使用阶段：仅在 API 调用时追加以构建 effective system prompt
effective_system = self._cached_system_prompt or ""
if self.ephemeral_system_prompt:
    effective_system = (effective_system + "\n\n" + self.ephemeral_system_prompt).strip()
```

### 13.2 安全保障

| 保障维度 | 机制 |
|---------|------|
| **不持久化** | `ephemeral_system_prompt` 不写入 `_session_messages`、不写入轨迹文件 |
| **不入缓存** | 不进入 `_cached_system_prompt`，不影响 Anthropic prompt cache 的稳定性 |
| **不跨轮次** | 每次 API 调用前从实例属性读取，可被 `system_message` 参数覆盖 |
| **不泄露** | gateway 模式下临时注入的上下文（如用户身份、租户信息）不保存到会话数据库 |

### 13.3 典型使用场景

- **Gateway 多租户隔离**：每个租户的身份信息作为 ephemeral prompt 注入，不污染共享的缓存系统提示词
- **临时行为修正**：需要单次模型行为调整而不改变会话基本身份
- **安全注入**：短期的安全策略或审计要求，过后不留痕迹

---

## 十四、设计模式

| 模式 | 应用 |
|------|------|
| 模板方法 | ContextEngine 定义压缩骨架 |
| 策略模式 | 上下文引擎选择、模型路由、Provider 解析 |
| 适配器模式 | Codex/Anthropic → OpenAI 接口 |
| 装饰器模式 | `apply_anthropic_cache_control()` |
| 责任链 | 错误分类流水线、上下文长度解析链、Provider 解析链 |
| 快照模式 | `LocalEditSnapshot` 文件系统快照 |

---

## 十五、面试八股文

### 15.1 Token 估算原理

**实际代码未使用 BPE/tiktoken。** `model_metadata.py` 中的唯一 token 估算是 `estimate_tokens_rough()`：

```python
def estimate_tokens_rough(text: str) -> int:
    if not text:
        return 0
    return (len(text) + 3) // 4  # ~4 chars/token
```

这是一个极为简化的**字符数近似**（ceiling division），将文本长度除以 4 作为 token 数。不进行任何分词操作，不调用 tiktoken 或任何 tokenizer。

**设计权衡**：
- **速度**：O(1) 字符数计算，无分词开销。适合上下文压缩器的预检查（pre-flight）、消息列表 token 估算等高频调用场景。
- **精度**：英文约 4 字符/token，中文约 1-2 字符/token 的粗略假设。对于混合中英文、代码块等场景，误差可达 2-3x。但对于阈值判断（是否超 80% 上下文窗口触发压缩）来说足够。
- **安全性**：ceiling division `(n+3)//4` 保证 1-3 字符的短文本不会估算为 0 token，避免了压缩器系统性地低估大量短工具结果的 token 数。

**完整 token 计数**由 API 响应的 `usage.prompt_tokens` / `usage.completion_tokens` 提供，这才是精确值。`estimate_tokens_rough()` 仅用于 API 调用前的预检查，不求精确。

**BPE 原理（面试题参考，本代码库未实现）**：
1. 初始化：文本分解为单个字节
2. 迭代合并：统计相邻 token 对频率，最高频合并为新 token
3. 终止：达到目标词表大小

BPE 是 tiktoken (OpenAI) 和 Anthropic tokenizer 的底层算法，但本代码库**未直接实现或调用 BPE**。

### 15.2 上下文窗口管理策略

Head 保护 + Tail 保护 + 中间区域（工具输出预修剪 → LLM 摘要 → 迭代更新）+ 反抖动。

### 15.3 指数退避算法

```python
delay = min(base_delay * 2^(attempt-1), max_delay)
```

### 15.4 抖动的作用

纯指数退避导致"惊群效应"。添加随机抖动自然错开竞争者。

### 15.5 缓存策略

- **LRU**：`_SKILLS_PROMPT_CACHE`（OrderedDict，最大 8 条）
- **TTL**：OpenRouter 模型元数据 3600 秒，端点模型元数据 300 秒
- **Manifest 验证**：mtime/size 对比

### 15.6 如何降低 LLM API 调用成本？

1. Prompt Caching（~75% 输入成本降低）
2. 智能模型路由（简单消息 → 廉价模型）
3. 上下文压缩（减少 input token）
4. 工具输出预修剪
5. 精确成本追踪

### 15.7 结构化 Prompt 设计

1. 分层结构：身份 → 环境 → 平台 → 行为 → 技能 → 上下文
2. XML 标签隔离
3. 摘要模板（Goal/Progress/Decisions/Files）
4. Handoff 语义（`"[CONTEXT COMPACTION — REFERENCE ONLY]"`）

---

## 十六、面试官可能问的问题

### Q1: 上下文压缩算法如何工作？

5 阶段：工具输出预修剪 → 确定保护边界 → LLM 摘要 → 组装 → 清理孤立对。支持迭代更新和聚焦压缩。

### Q2: 如何处理信息丢失？

多层保护：Head/tail 永不压缩、结构化模板保留关键信息、迭代更新、摘要失败 fallback、反抖动。

### Q3: 错误分类如何消歧义 402？

检查 usage-limit 模式 + 瞬时信号，两者都满足分类为 rate_limit，否则 billing。

### Q4: 退避种子为什么用 `time.time_ns() ^ (tick * 0x9E3779B9)`？

`0x9E3779B9` 是黄金比例 32 位定点表示。与 tick 相乘后与时间戳异或，确保同纳秒内不同调用产生不同种子。

### Q5: Prompt Caching 为什么选择 3 条非 system 消息？

Anthropic 最多 4 个断点。1 个 system（稳定），3 个最近消息（滚动窗口，命中率最高）。

### Q6: 为什么用 Decimal 而不是 float？

浮点精度问题。`0.1 + 0.2 != 0.3`。对用户账单准确性至关重要。

### Q7: 如何检测 Prompt Injection？

10 种模式 + 14 种不可见 Unicode 字符。检测到返回 `"[BLOCKED]"`。

### Q8: ContextEngine 的生命周期？

6 阶段：on_session_start → update_from_response → should_compress → compress → on_session_end → on_session_reset。

### Q9: 模型路由如何判断消息"简单"？

5 个条件全部满足：字符 ≤ 160、单词 ≤ 28、换行 ≤ 1、无代码/URL、无复杂关键词。

### Q10: 辅助 LLM 客户端如何统一接口？

适配器模式。Codex/Anthropic 适配器暴露 `client.chat.completions.create()` 接口，内部转换为原生 API 格式。

### Q11: 摘要失败时如何处理？

三层容错：冷却机制（60/600 秒）、模型降级、静态 fallback 标记。

### Q12: 价格查询失败时如何降级？

查询链本身就是降级。全部失败返回 `CostResult(amount_usd=None, status="unknown")`。

---

## 十七、学习路径建议

### 入门级（< 1000行）：
1. `context_engine.py`（185 行）-- 抽象基类设计，理解上下文管理接口
2. `retry_utils.py`（57 行）-- 抖动退避算法实现
3. `prompt_caching.py`（73 行）-- 缓存断点注入机制
4. `smart_model_routing.py`（196 行）-- 智能模型路由逻辑
5. `prompt_builder.py`（1046 行）-- Prompt 组装全流程，了解提示工程实践

### 进阶级（1000-2000行）：
6. `error_classifier.py`（830 行）-- 错误分类体系，学习复杂模式匹配
7. `usage_pricing.py`（688 行）-- 成本管理，了解多数据源集成与降级策略
8. `context_compressor.py`（1092 行）-- 压缩算法，理解结构化摘要与迭代更新机制

### 深入级（>2000行以上）：
9. `auxiliary_client.py` -- 辅助LLM调用，了解多提供商适配与统一接口设计
10. `model_metadata.py` -- 模型元数据管理，了解Token估算与上下文长度查询
11. `trajectory.py` -- 会话轨迹持久化，了解数据存储与恢复机制
12. `run_agent.py` -- 主循环实现，理解完整调用链与模块协同
