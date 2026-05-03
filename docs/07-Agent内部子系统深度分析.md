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

### 2.4 模型特定指导

- **GPT/Codex**：`<tool_persistence>`、`<mandatory_tool_use>`、`<verification>` XML 标签
- **Gemini**：绝对路径、并行工具调用、非交互命令

### 2.5 Skills 索引的两层缓存

```
Layer 1: 进程内 LRU 缓存（OrderedDict, 最大 8 条）
Layer 2: 磁盘快照（.skills_prompt_snapshot.json, mtime/size manifest 验证）
Layer 3: 冷启动 — 全文件系统扫描
```

---

## 三、上下文管理

### 3.1 ContextEngine 生命周期

```
on_session_start → update_from_response → should_compress → compress → on_session_end
```

### 3.2 压缩算法（5 阶段）

1. **工具输出预修剪**（无 LLM 调用）：MD5 去重 + 信息性单行摘要
2. **确定保护边界**：head + token-budget tail
3. **LLM 摘要生成**：结构化模板（Goal/Progress/Decisions/Files）
4. **组装**：head + summary + tail
5. **孤立 tool_call/tool_result 对清理**

### 3.3 反抖动保护

```python
if self._ineffective_compression_count >= 2:
    return False  # 最近两次压缩各自节省不到 10%
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

```python
marker = {"type": "ephemeral"}
if cache_ttl == "1h":
    marker["ttl"] = "1h"
```

热门前缀读取成本降低 ~75%。

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

### 5.2 402 错误消歧义

区分真正计费耗尽和伪装的瞬时限速：
```python
has_usage_limit = any(p in error_msg for p in _USAGE_LIMIT_PATTERNS)
has_transient_signal = any(p in error_msg for p in _USAGE_LIMIT_TRANSIENT_SIGNALS)
if has_usage_limit and has_transient_signal:
    return FailoverReason.rate_limit  # 瞬时配额，非计费
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

`0x9E3779B9` 是黄金比例的 32 位定点表示（Fibonacci Hashing），提供良好的哈希分布。

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

---

## 八、辅助 LLM 调用

**Provider 解析链**：
- 文本任务：OpenRouter → Nous Portal → 自定义端点 → Codex OAuth → Anthropic → None
- 视觉任务：主 provider → OpenRouter → Nous Portal → Codex OAuth → Anthropic → None

**适配器模式**：`_CodexCompletionsAdapter` 和 `_AnthropicCompletionsAdapter` 将各自 API 适配为 OpenAI chat.completions 接口。

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

1. `context_engine.py`（185 行）-- 抽象基类设计
2. `retry_utils.py`（57 行）-- 抖动退避
3. `prompt_caching.py`（73 行）-- 缓存断点注入
4. `error_classifier.py`（830 行）-- 错误分类体系
5. `smart_model_routing.py`（196 行）-- 模型路由
6. `usage_pricing.py`（688 行）-- 成本管理
7. `prompt_builder.py`（1046 行）-- Prompt 组装全流程
8. `context_compressor.py`（1092 行）-- 压缩算法
