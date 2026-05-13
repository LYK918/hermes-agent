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

```
Slot 1: Identity (DEFAULT_AGENT_IDENTITY or SOUL.md)
Slot 2: Environment hints (WSL, Docker, etc.)
Slot 3: Platform hints (WhatsApp/Telegram/Discord/...)
Slot 4: Tool-use enforcement guidance (model-specific)
Slot 5: Model-specific operational guidance
Slot 6: Memory guidance
Slot 7: Session search guidance
Slot 8: Skills guidance
Slot 9: Skills system prompt (index)
Slot 10: Context files (.hermes.md, AGENTS.md, CLAUDE.md, .cursorrules)
```

**SOUL.md 优先**：存在时替换默认身份（非追加）。

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

```
on_session_start → update_from_response → should_compress → compress → on_session_end
```

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

### 3.4 Token 估算

```python
def estimate_tokens_rough(text: str) -> int:
    return (len(text) + 3) // 4  # ~4 chars/token
```

上下文长度解析链（10 级）：配置 → 缓存 → 端点元数据 → 本地服务器 → API → 注册表 → 默认值。

---

## 四、Prompt 缓存

### 4.1 Anthropic Prompt Caching

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

## 五、错误处理

### 5.1 错误分类体系

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

### 5.2 402 错误消歧义

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

### 5.3 Jittered Backoff

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

---

## 六、模型路由

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

## 七、成本管理

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

## 八、辅助 LLM 调用

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

## 九、设计模式

| 模式 | 应用 |
|------|------|
| 模板方法 | ContextEngine 定义压缩骨架 |
| 策略模式 | 上下文引擎选择、模型路由、Provider 解析 |
| 适配器模式 | Codex/Anthropic → OpenAI 接口 |
| 装饰器模式 | `apply_anthropic_cache_control()` |
| 责任链 | 错误分类流水线、上下文长度解析链、Provider 解析链 |
| 快照模式 | `LocalEditSnapshot` 文件系统快照 |

---

## 十、面试八股文

### 10.1 BPE 分词原理

1. 初始化：文本分解为单个字节
2. 迭代合并：统计相邻 token 对频率，最高频合并为新 token
3. 终止：达到目标词表大小

粗略估算：英文 ~4 字符/token，中文 ~1-2 字符/token。

### 10.2 上下文窗口管理策略

Head 保护 + Tail 保护 + 中间区域（工具输出预修剪 → LLM 摘要 → 迭代更新）+ 反抖动。

### 10.3 指数退避算法

```python
delay = min(base_delay * 2^(attempt-1), max_delay)
```

### 10.4 抖动的作用

纯指数退避导致"惊群效应"。添加随机抖动自然错开竞争者。

### 10.5 缓存策略

- **LRU**：`_SKILLS_PROMPT_CACHE`（OrderedDict，最大 8 条）
- **TTL**：OpenRouter 模型元数据 3600 秒，端点模型元数据 300 秒
- **Manifest 验证**：mtime/size 对比

### 10.6 如何降低 LLM API 调用成本？

1. Prompt Caching（~75% 输入成本降低）
2. 智能模型路由（简单消息 → 廉价模型）
3. 上下文压缩（减少 input token）
4. 工具输出预修剪
5. 精确成本追踪

### 10.7 结构化 Prompt 设计

1. 分层结构：身份 → 环境 → 平台 → 行为 → 技能 → 上下文
2. XML 标签隔离
3. 摘要模板（Goal/Progress/Decisions/Files）
4. Handoff 语义（`"[CONTEXT COMPACTION — REFERENCE ONLY]"`）

---

## 十一、面试官可能问的问题

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

## 十二、学习路径建议

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
