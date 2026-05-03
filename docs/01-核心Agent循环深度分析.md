# 01 - 核心 Agent 循环深度分析

> 学习顺序：第 1 篇 | 基础模块，建议最先学习

---

## 一、模块概述

### 1.1 `run_agent.py` -- 核心 Agent 运行时

这个文件（11,462 行）是整个 Hermes Agent 的心脏，实现了完整的 **ReAct (Reasoning + Acting) 循环**。核心职责：

1. **管理 LLM 对话循环**：用户消息 -> API 调用 -> 工具执行 -> 再次调用，直到模型给出最终回答
2. **工具编排**：并行/串行调度数十种工具
3. **错误恢复**：多层重试、provider 故障转移、上下文压缩、凭证轮换
4. **上下文管理**：自动压缩过长的对话历史，维护 token 预算

### 1.2 `model_tools.py` -- 工具编排层

这个文件（563 行）是薄薄的编排层：

1. **工具发现**：导入 `tools/` 目录下的所有工具模块，触发自注册
2. **工具定义分发**：根据 toolset 过滤，返回 OpenAI 格式的工具 schema
3. **函数调用路由**：将 LLM 返回的 tool_calls 分发到对应的 handler
4. **sync/async 桥接**：统一处理同步/异步工具的执行

---

## 二、核心类与函数详解

### 2.1 `AIAgent` 类

核心类，一个 Agent 实例代表一个完整的对话会话。构造函数参数 50+ 个，分为几类：

| 类别 | 参数示例 | 说明 |
|------|----------|------|
| 模型配置 | `model`, `base_url`, `api_key`, `provider` | LLM 接入参数 |
| 行为控制 | `max_iterations=90`, `tool_delay=1.0` | 循环上限、工具间延迟 |
| 工具过滤 | `enabled_toolsets`, `disabled_toolsets` | 按 toolset 启用/禁用 |
| 回调钩子 | `tool_progress_callback`, `stream_delta_callback` | UI/网关集成 |
| 会话管理 | `session_id`, `session_db`, `platform` | 持久化与多平台适配 |
| 容错 | `fallback_model`, `credential_pool` | 故障转移链 |

### 2.2 `IterationBudget` 类

```python
class IterationBudget:
    def __init__(self, max_total: int):
        self.max_total = max_total
        self._used = 0
        self._lock = threading.Lock()

    def consume(self) -> bool:    # 消耗一次迭代，返回是否允许
    def refund(self) -> None:     # 退还一次（execute_code 专用）
```

关键设计：`execute_code` 工具的迭代会被退还，因为它是程序化的工具调用，不应消耗宝贵的迭代预算。

### 2.3 `run_conversation()` 方法

整个 Agent 的入口方法，返回结构化的结果 dict：

```python
result = {
    "final_response": final_response,
    "last_reasoning": last_reasoning,
    "messages": messages,
    "api_calls": api_call_count,
    "completed": completed,
    "interrupted": interrupted,
    "input_tokens": self.session_input_tokens,
    "output_tokens": self.session_output_tokens,
    "estimated_cost_usd": self.session_estimated_cost_usd,
    ...
}
```

### 2.4 `_execute_tool_calls()` 方法

工具执行的总入口，根据工具特性决定并行还是串行：

```python
def _execute_tool_calls(self, assistant_message, messages, effective_task_id, api_call_count):
    tool_calls = assistant_message.tool_calls
    if not _should_parallelize_tool_batch(tool_calls):
        return self._execute_tool_calls_sequential(...)
    return self._execute_tool_calls_concurrent(...)
```

### 2.5 `_run_async()` -- sync/async 桥接核心

```python
def _run_async(coro):
    try:
        loop = asyncio.get_running_loop()
    except RuntimeError:
        loop = None

    if loop and loop.is_running():
        # 已有运行中的 event loop → 启动新线程
        with concurrent.futures.ThreadPoolExecutor(max_workers=1) as pool:
            future = pool.submit(asyncio.run, coro)
            return future.result(timeout=300)

    if threading.current_thread() is not threading.main_thread():
        # 工作线程 → 使用线程本地的持久 loop
        worker_loop = _get_worker_loop()
        return worker_loop.run_until_complete(coro)

    # 主线程 → 使用全局持久 loop
    tool_loop = _get_tool_loop()
    return tool_loop.run_until_complete(coro)
```

---

## 三、关键设计模式

### 3.1 策略模式 (Strategy Pattern)

多 API 模式适配：根据 `api_mode` 属性选择不同的 API 调用策略：
- `chat_completions` -- OpenAI 标准 Chat Completions API
- `codex_responses` -- OpenAI Responses API (GPT-5+)
- `anthropic_messages` -- Anthropic Messages API
- `bedrock_converse` -- AWS Bedrock Converse API

### 3.2 观察者模式 (Observer Pattern)

通过回调钩子实现 UI/网关的无侵入集成：

```python
self.tool_progress_callback = tool_progress_callback
self.tool_start_callback = tool_start_callback
self.tool_complete_callback = tool_complete_callback
self.thinking_callback = thinking_callback
self.stream_delta_callback = stream_delta_callback
self.status_callback = status_callback
```

### 3.3 责任链模式 (Chain of Responsibility)

错误恢复链：API 调用失败时，按优先级尝试一系列恢复策略：
1. 凭证池轮换
2. 凭证刷新
3. thinking block 签名修复
4. 上下文压缩
5. Provider 故障转移
6. 传输层恢复

### 3.4 注册表模式 (Registry Pattern)

工具自注册：每个工具文件在模块级别调用 `registry.register()`，声明 schema、handler、toolset 归属和可用性检查函数。

### 3.5 模板方法模式 (Template Method)

`run_conversation()` 定义了完整的对话流程骨架：
1. 初始化/清理 -> 2. 构建 system prompt -> 3. 预压缩 -> 4. 主循环 -> 5. 后处理/持久化

---

## 四、数据流：完整消息流转

```
用户输入 (user_message)
    │
    ▼
run_conversation() 入口
    │
    ├─ 1. 清理/初始化
    │
    ├─ 2. 构建 system prompt
    │     └─ 身份/平台提示 + 工具使用指引 + 上下文文件 + 内存 + 技能
    │
    ├─ 3. 预压缩检查
    │
    └─ 4. 主循环 ─────────────────────────────
         │
         ├─ while api_call_count < max_iterations:
         │    ├─ 检查中断请求
         │    ├─ 消耗迭代预算
         │    ├─ 构建 API 消息（注入预填充、缓存标记）
         │    ├─ API 调用（优先流式，带重试循环 max_retries=3）
         │    ├─ 响应验证
         │    ├─ 错误处理（分类器 → 恢复策略链）
         │    ├─ 如果有 tool_calls:
         │    │    ├─ 验证工具名称（修复幻觉名称）
         │    │    ├─ 验证 JSON 参数
         │    │    ├─ _execute_tool_calls()
         │    │    └─ continue
         │    └─ 如果没有 tool_calls（最终回答）:
         │         ├─ 剥离 <think> 标签
         │         └─ break
         │
         └─ 后处理
              ├─ 预算耗尽 → 请求摘要
              ├─ 保存 trajectory
              ├─ 清理任务资源
              ├─ 持久化会话
              └─ 后台 memory/skill 审查
```

### 消息格式（OpenAI 格式）

```python
# 系统消息
{"role": "system", "content": "You are Hermes Agent..."}

# 用户消息
{"role": "user", "content": "帮我写一个 Python 脚本"}

# 助手消息（带工具调用）
{
    "role": "assistant",
    "content": "我来帮你写...",
    "tool_calls": [
        {
            "id": "call_abc123",
            "type": "function",
            "function": {
                "name": "write_file",
                "arguments": '{"path": "script.py", "content": "print(1)"}'
            }
        }
    ]
}

# 工具结果
{
    "role": "tool",
    "tool_call_id": "call_abc123",
    "content": '{"success": true, "message": "File written"}'
}
```

---

## 五、错误处理与重试

### 5.1 错误分类体系

`error_classifier.py` 定义了 13 种 `FailoverReason` 枚举：

| 分类 | HTTP 状态 | 恢复策略 |
|------|-----------|----------|
| `auth` / `auth_permanent` | 401/403 | 刷新凭证 or 中止 |
| `rate_limit` | 429 | 退避 + 凭证轮换 |
| `billing` | 402 | 立即轮换凭证 |
| `overloaded` | 503/529 | 退避重试 |
| `server_error` | 500/502 | 重试 |
| `context_overflow` | 400 + 大会话 | 压缩上下文 |
| `payload_too_large` | 413 | 压缩历史 |
| `thinking_signature` | 400 (Anthropic) | 剥离 thinking blocks |
| `long_context_tier` | 429 (Anthropic) | 降低 context_length |

### 5.2 Jittered Backoff 重试策略

```python
def jittered_backoff(attempt, base_delay=5.0, max_delay=120.0, jitter_ratio=0.5):
    delay = min(base_delay * (2 ** (attempt - 1)), max_delay)
    jitter = random.uniform(0, jitter_ratio * delay)
    return delay + jitter
```

使用去相关抖动避免多会话同时重试的"惊群效应"。

### 5.3 特殊恢复机制

- **Unicode 恢复**：surrogate 字符和 ASCII 编码系统的两阶段清洗
- **上下文压缩**：超长对话自动压缩中间历史
- **空响应恢复**：thinking-only prefill、post-tool nudge、prior-turn fallback 三重恢复
- **截断工具调用检测**：通过参数是否以 `}` 结尾检测截断

---

## 六、关键代码片段分析

### 6.1 主循环终止条件

```python
while (api_call_count < self.max_iterations and self.iteration_budget.remaining > 0) or self._budget_grace_call:
```

两个独立的终止条件：`api_call_count` 是本 turn 内的计数，`iteration_budget` 是跨 turn 的全局预算。

### 6.2 工具并行判定

```python
def _should_parallelize_tool_batch(tool_calls) -> bool:
    if len(tool_calls) <= 1:
        return False
    tool_names = [tc.function.name for tc in tool_calls]
    if any(name in _NEVER_PARALLEL_TOOLS for name in tool_names):
        return False  # clarify 等交互工具永远串行
    # 路径作用域工具检查路径不重叠
    reserved_paths = []
    for tool_call in tool_calls:
        if tool_name in _PATH_SCOPED_TOOLS:
            scoped_path = _extract_parallel_scope_path(...)
            if any(_paths_overlap(scoped_path, existing) for existing in reserved_paths):
                return False  # 路径冲突，退回串行
```

### 6.3 类型强制转换

```python
def coerce_tool_args(tool_name, args):
    schema = registry.get_schema(tool_name)
    properties = (schema.get("parameters") or {}).get("properties")
    for key, value in args.items():
        if isinstance(value, str):
            prop_schema = properties.get(key)
            expected = prop_schema.get("type")
            coerced = _coerce_value(value, expected)
            if coerced is not value:
                args[key] = coerced
```

解决 LLM 常见的类型错误：`"42"` → `42`，`"true"` → `True`。

---

## 七、面试八股文

### 7.1 LLM Agent 的 ReAct 循环是什么？实现中有哪些关键考量？

**ReAct (Reasoning + Acting)** 是一种让 LLM 交替进行推理和行动的范式。模型先"思考"（reasoning），再"行动"（调用工具），观察工具结果后继续推理，直到完成任务。

**关键考量**：
1. 终止条件多样化：不只有 `max_iterations`，还有 `IterationBudget`、`_budget_grace_call`、中断信号等
2. 无限循环防护：多种重试计数器各有上限
3. 空响应恢复：模型可能返回只有 thinking 没有 content 的响应
4. 上下文膨胀控制：实时监控 token 使用量，自动触发压缩
5. 多 provider 兼容：同一个循环需要处理不同 API 格式

### 7.2 Function Calling / Tool Use 的实现原理

**定义阶段**：工具通过 `tools/registry.py` 自注册，声明 JSON Schema。`get_tool_definitions()` 根据 toolset 过滤后返回 OpenAI 格式的 `tools` 参数。

**调用阶段**：
1. API 返回 `tool_calls`
2. Agent 验证工具名称
3. 验证 JSON 参数
4. 分发到 handler
5. 结果以 `tool` role 消息追加到消息列表
6. 循环继续

### 7.3 如何处理 LLM 返回的 tool_calls？

1. 名称验证：遍历 `tool_calls`，检查是否在 `valid_tool_names` 中
2. 自动修复：`_repair_tool_call()` 尝试模糊匹配幻觉工具名
3. 参数验证：空字符串 → `{}`，截断检测
4. Guardrails：限制 delegate 数量，去重
5. 执行：根据工具特性选择并行/串行
6. 结果注入：以 `tool` role 消息追加

### 7.4 Agent 循环的终止条件有哪些？

至少有 **12 种**终止条件：
1. `api_call_count >= max_iterations`
2. `iteration_budget.remaining <= 0`
3. `_interrupt_requested`（用户中断）
4. `final_response is not None`（最终回答）
5. 非可重试的客户端错误
6. 重试次数耗尽且无 fallback
7. 上下文压缩耗尽
8. 空响应重试耗尽
9. 无效工具调用重试耗尽
10. 不完整 scratchpad 重试耗尽
11. 思考预算耗尽
12. `_budget_grace_call` 完成

### 7.5 如何防止 Agent 陷入无限循环？

1. `IterationBudget`：线程安全的计数器
2. 多重试计数器归零机制：成功执行工具后重置
3. `_budget_grace_call`：预算耗尽后再给一次机会
4. `execute_code` 退还：程序化工具调用不消耗迭代预算
5. 上下文压缩后重置重试计数器
6. 重复内容检测

### 7.6 同步与异步代码如何桥接？

`_run_async()` 的三层策略：
1. 已有 event loop 运行中 → 在新线程中调用 `asyncio.run()`
2. 工作线程 → 使用线程本地的持久 event loop
3. 主线程 → 使用全局持久 loop

持久 loop 避免了 `asyncio.run()` 每次创建/销毁循环导致的 "Event loop is closed" 错误。

### 7.7 Token 管理与上下文窗口限制

1. 粗略估计：字符数 / 4 的启发式
2. 精确统计：API 响应的 `usage` 字段
3. 压缩阈值：默认 50%，到达时自动压缩
4. 压缩策略：保留首尾 N 条消息，中间用辅助模型总结
5. 上下文探测：API 返回 overflow 时逐步降低 context_length
6. Prompt Caching：对 Claude 模型自动注入 cache_control breakpoints

---

## 八、面试官可能问的问题及详细回答

### Q1: 这个 Agent 的架构和 ChatGPT 的 Function Calling 有什么区别？

本实现是 OpenAI Function Calling 的**完整 Agent 运行时封装**。ChatGPT 只返回一次 tool_calls，调用方需要自己实现循环。而 `AIAgent.run_conversation()` 实现了完整的 ReAct 循环，包括：重试、错误恢复、多 provider 故障转移、上下文压缩、并行工具执行、迭代预算管理等。

### Q2: 为什么要区分 agent-level tools 和 registry tools？

某些工具（`todo`, `memory`, `session_search`, `delegate_task`）需要访问 Agent 实例的状态。如果走 `registry.dispatch()`，这些状态无法传递。因此 `_invoke_tool()` 对这些工具直接处理，其余走注册表统一分发。

### Q3: 工具并行执行如何保证线程安全？

三重保障：
1. 并行判定：只有只读工具和路径不冲突的文件工具才允许并行
2. 线程池限制：最多 8 个并发
3. 结果顺序保证：按 index 写入，最终按原始顺序追加

### Q4: 上下文压缩是怎么工作的？

当 token 用量超过阈值（默认 50%），`ContextCompressor` 将中间消息（保留首 3 条 + 尾 20 条）用辅助 LLM 生成摘要，替换为一条 summary 消息。最多尝试 3 次压缩。

### Q5: delegate_task 子 Agent 是怎么工作的？

创建新的 `AIAgent` 实例，继承父 Agent 的 model/tools/config 但有独立的 `IterationBudget`（默认 50 次迭代）。子 Agent 的 `_delegate_depth` 递增，最大深度 2。

### Q6: 为什么 streaming 比 non-streaming 更受欢迎？

streaming 提供更细粒度的健康检查（90s stale-stream 检测），而非 streaming 模式在 provider 不返回响应时可能无限挂起。

### Q7: 中断机制如何实现？

`_interrupt_requested` 标志位通过 `set_interrupt()` 设置，绑定到执行线程 ID。主循环在每个迭代开始、工具执行前、重试等待期间检查此标志。

### Q8: 如何处理 LLM 幻觉出的工具名？

1. `_repair_tool_call()` 尝试模糊匹配已知工具名
2. 如果无法修复，返回工具错误消息让模型自行纠正，最多重试 3 次

### Q9: Credential Pool 是怎么用的？

当主 API key 被限流时，从 pool 中取出下一个可用凭证，重建 client 并重试。Pool 追踪每个凭证的可用状态。

### Q10: Session 持久化的双写策略？

每个 turn 结束时双写：JSON 日志（完整原始消息）+ SQLite 数据库（结构化查询）。

### Q11: Tool result storage 是什么？

超大工具输出保存到临时文件，在消息中只保留文件路径引用，避免上下文膨胀。

### Q12: Checkpoint Manager 的作用？

在执行破坏性操作前自动创建文件系统快照，允许用户通过 `/undo` 回滚。

### Q13: Background Memory/Skill Review 是怎么工作的？

Agent 循环结束后，在守护线程中创建临时 Agent fork，让模型决定是否保存到 memory/skill。完全异步，不影响主响应。

### Q14: 如何处理模型的 "thinking budget exhausted"？

直接返回用户友好的提示消息，建议降低 reasoning effort 或增加 max_tokens。不浪费重试次数。

### Q15: `_SafeWriter` 解决了什么问题？

当 Agent 作为 systemd 服务运行时，stdout/stderr pipe 可能断开。`_SafeWriter` 透明包装所有 stdio 操作，静默吞掉 `OSError`。

---

## 九、学习路径建议

### 阶段 1：理解骨架（1-2 小时）
1. `model_tools.py`（全文 563 行）-- 工具注册和分发
2. `tools/registry.py`（前 100 行）-- 工具自注册机制
3. `run_agent.py:535-665` -- `AIAgent.__init__`

### 阶段 2：核心循环（2-3 小时）
4. `run_agent.py:8102-8470` -- `run_conversation()` 前半部分
5. `run_agent.py:10360-10589` -- tool_calls 处理
6. `run_agent.py:10685-10986` -- 最终回答处理
7. `run_agent.py:11038-11200` -- 后处理

### 阶段 3：工具执行（1-2 小时）
8. `run_agent.py:7158-7267` -- `_execute_tool_calls`
9. `run_agent.py:7293-7530` -- 并行执行路径
10. `run_agent.py:7531-7909` -- 串行执行路径

### 阶段 4：错误恢复（2-3 小时）
11. `agent/error_classifier.py` -- 错误分类体系
12. `agent/retry_utils.py` -- 退避策略
13. `run_agent.py:8671-9000` -- 重试循环
14. `run_agent.py:9332-10113` -- 异常处理
