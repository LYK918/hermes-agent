# 01 - 核心 Agent 循环深度分析

> 学习顺序：第 1 篇 | 基础模块，建议最先学习

---

## 一、模块概述

### 1.1 `run_agent.py` -- 核心 Agent 运行时

这个文件（11,460+ 行）是整个 Hermes Agent 的心脏，实现了完整的 **ReAct (Reasoning + Acting) 循环**。核心职责：
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

### 1.3 `tools/registry.py` -- 工具注册中心

这个文件（483 行）是工具的元数据管理中心：
1. **自注册机制**：每个工具在模块级别调用 `registry.register()` 声明 schema 和 handler
2. **元数据存储**：维护工具的名称、所属 toolset、依赖环境、可用性检查等信息
3. **动态发现**：支持内置工具、MCP 工具、插件工具的自动发现和注册
4. **并发安全**：线程安全的设计支持 MCP 工具的动态刷新和多线程访问

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

#### 关键实例方法：
- **`run_conversation()`**: 对话入口，协调整个 ReAct 循环的执行
- **`_execute_tool_calls()`**: 工具执行总入口，自动判断并行/串行执行策略
- **`_invoke_tool()`**: 单工具执行，处理 agent 级工具（todo、memory、delegate_task）
- **`_interruptible_api_call()`**: 带中断和超时保护的 API 调用
- **`_compress_context()`**: 上下文压缩，自动总结历史消息以适配模型窗口

### 2.2 `IterationBudget` 类

```python
class IterationBudget:
    def __init__(self, max_total: int):
        self.max_total = max_total
        self._used = 0
        self._lock = threading.Lock()

    def consume(self) -> bool:    # 消耗一次迭代，返回是否允许
        with self._lock:
            if self._used >= self.max_total:
                return False
            self._used += 1
            return True
            
    def refund(self) -> None:     # 退还一次（execute_code 专用）
        with self._lock:
            if self._used > 0:
                self._used -= 1
                
    @property
    def remaining(self) -> int:
        with self._lock:
            return self.max_total - self._used
```

关键设计：`execute_code` 工具的迭代会被退还，因为它是程序化的工具调用，不应消耗宝贵的迭代预算。线程安全的设计支持多工具并行执行时的预算管理。

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
    "partial": False,  # 是否为部分结果（迭代耗尽、错误等）
    "error": None,     # 错误信息
}
```

#### 执行流程：
1. **初始化阶段**：
   - 安装安全 stdio 保护，避免管道断开导致崩溃
   - 设置会话上下文，方便日志过滤
   - 恢复主运行时（如果之前触发了 fallback）
   - 清理用户输入中的 surrogate 字符，避免 JSON 序列化错误
   - 重置各种重试计数器和迭代预算
   - 清理死 TCP 连接，避免 API 调用挂起

2. **准备阶段**：
   - 加载对话历史，反序列化工具结果
   - 从历史中恢复 TodoStore 状态（网关场景）
   - 构建/加载缓存的系统提示（支持前缀缓存优化）
   - 预检查上下文大小，主动压缩避免 API 错误
   - 调用 `pre_llm_call` 插件钩子，注入额外上下文

3. **主循环阶段**：
   - 检查中断请求，支持用户随时取消
   - 消耗迭代预算，支持跨轮次的全局预算控制
   - 构建 API 消息，注入预填充内容和缓存标记
   - 调用 `pre_api_request` 插件钩子
   - 执行带重试和故障转移的 API 调用
   - 解析和验证响应格式
   - 处理工具调用或最终回答

4. **后处理阶段**：
   - 保存会话轨迹到数据库
   - 清理任务资源（终端、浏览器会话等）
   - 触发后台记忆和技能审查
   - 调用 `post_llm_call` 插件钩子

### 2.4 `_execute_tool_calls()` 方法

工具执行的总入口，根据工具特性决定并行还是串行：

```python
def _execute_tool_calls(self, assistant_message, messages, effective_task_id, api_call_count):
    tool_calls = assistant_message.tool_calls
    if not _should_parallelize_tool_batch(tool_calls):
        return self._execute_tool_calls_sequential(...)
    return self._execute_tool_calls_concurrent(...)
```

#### 并行判定规则：
- 工具数量 <= 1 时串行
- 包含交互工具（clarify、delegate_task 等）时串行
- 路径作用域工具（文件读写等）路径重叠时串行
- 只读工具和路径不冲突的文件工具允许并行
- 最多并行 8 个工具，避免资源耗尽

#### 并行执行保障：
- 线程池隔离，避免阻塞主循环
- 结果顺序保证，按原始调用顺序追加到消息
- 超时保护，单个工具执行超时不影响整个批次
- 错误隔离，单个工具失败不影响其他工具执行

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

#### 设计亮点：
- 三层适配策略，兼容各种运行环境（CLI、网关、RL 环境）
- 持久化 event loop，避免 "Event loop is closed" 错误
- 线程本地 loop 隔离，避免多线程并发冲突
- 5 分钟超时保护，防止协程挂起导致死锁

---

## 三、关键设计模式

### 3.1 策略模式 (Strategy Pattern)

多 API 模式适配：根据 `api_mode` 属性选择不同的 API 调用策略：
- `chat_completions` -- OpenAI 标准 Chat Completions API
- `codex_responses` -- OpenAI Responses API (GPT-5+)
- `anthropic_messages` -- Anthropic Messages API
- `bedrock_converse` -- AWS Bedrock Converse API

每个模式有独立的请求构建、响应解析和错误处理逻辑，新增 provider 只需添加对应策略类。

### 3.2 观察者模式 (Observer Pattern)

通过回调钩子实现 UI/网关的无侵入集成：

```python
self.tool_progress_callback = tool_progress_callback
self.tool_start_callback = tool_start_callback
self.tool_complete_callback = tool_complete_callback
self.thinking_callback = thinking_callback
self.stream_delta_callback = stream_delta_callback
self.status_callback = status_callback
self.step_callback = step_callback
```

插件系统也基于此模式，提供 `pre_llm_call`、`pre_api_request`、`pre_tool_call`、`post_tool_call`、`post_llm_call` 等钩子点，支持功能扩展而无需修改核心代码。

### 3.3 责任链模式 (Chain of Responsibility)

错误恢复链：API 调用失败时，按优先级尝试一系列恢复策略：
1. 凭证池轮换
2. 凭证刷新
3. thinking block 签名修复
4. 上下文压缩
5. Provider 故障转移
6. 传输层恢复
7. 降级到 fallback 模型
8. 返回部分结果或友好错误

每个恢复策略独立实现，按顺序尝试，直到成功或全部失败。

### 3.4 注册表模式 (Registry Pattern)

工具自注册：每个工具文件在模块级别调用 `registry.register()`，声明 schema、handler、toolset 归属和可用性检查函数。

```python
# 工具注册示例（tools/web_search.py）
registry.register(
    name="web_search",
    toolset="web",
    schema=WEB_SEARCH_SCHEMA,
    handler=handle_web_search,
    check_fn=check_web_search_available,
    requires_env=["SERPER_API_KEY"],
    is_async=True,
    description="Search the web for real-time information",
    emoji="🔍",
    max_result_size_chars=10000,
)
```

优点：
- 新增工具无需修改核心代码，只需添加新的工具文件
- 工具元数据集中管理，便于查询和过滤
- 支持动态工具发现和注册（MCP、插件）
- 可用性检查集中处理，避免重复代码

### 3.5 模板方法模式 (Template Method)

`run_conversation()` 定义了完整的对话流程骨架：
1. 初始化/清理 -> 2. 构建 system prompt -> 3. 预压缩 -> 4. 主循环 -> 5. 后处理/持久化

各个步骤的具体实现可以被子类重写，支持自定义 Agent 行为而不改变整体流程。

---

## 四、数据流：完整消息流转

```
用户输入 (user_message)
    │
    ▼
run_conversation() 入口
    │
    ├─ 1. 清理/初始化
    │     ├─ 安装安全 stdio 保护
    │     ├─ 设置会话上下文
    │     ├─ 恢复主运行时
    │     ├─ 清理用户输入特殊字符
    │     └─ 重置重试计数器和迭代预算
    │
    ├─ 2. 构建 system prompt
    │     ├─ 加载缓存的系统提示（或重新构建）
    │     ├─ 身份/平台提示 + 工具使用指引
    │     ├─ 记忆系统注入相关上下文
    │     └─ 技能系统注入可用技能描述
    │
    ├─ 3. 预压缩检查
    │     ├─ 估算当前上下文 token 用量
    │     └─ 超过阈值时主动压缩中间历史
    │
    └─ 4. 主循环 ─────────────────────────────
         │
         ├─ while api_call_count < max_iterations and iteration_budget.remaining > 0 or _budget_grace_call:
         │    ├─ 检查中断请求（用户取消、超时等）
         │    ├─ 消耗迭代预算（execute_code 会退还）
         │    ├─ 调用 step_callback 钩子
         │    ├─ 构建 API 消息
         │    │     ├─ 注入记忆预取结果
         │    │     ├─ 注入插件上下文
         │    │     ├─ 提取 reasoning 内容到单独字段
         │    │     ├─ 清理内部字段（reasoning、finish_reason 等）
         │    │     └─ 应用 prompt 缓存标记（Claude 模型）
         │    ├─ 消息标准化和清理
         │    │     ├─ 清理多余空白字符
         │    │     └─ 标准化工具调用 JSON 格式（提升缓存命中率）
         │    ├─ API 调用（优先流式，带重试循环 max_retries=3）
         │    │     ├─ Nous 速率限制预检查
         │    │     ├─ 调用 pre_api_request 插件钩子
         │    │     ├─ 执行带中断保护的 API 调用
         │    │     ├─ 响应格式验证和错误分类
         │    │     ├─ 重试逻辑（带抖动退避，最长 2 分钟）
         │    │     ├─ 故障转移（自动切换到 fallback 模型）
         │    │     └─ 错误处理（按错误类型应用恢复策略）
         │    ├─ 响应解析
         │    │     ├─ 提取消息内容和 reasoning
         │    │     ├─ 解析工具调用（支持多 provider 格式）
         │    │     └─ 处理 finish_reason（stop、length、tool_calls 等）
         │    ├─ 如果有 tool_calls:
         │    │     ├─ 工具名称验证和自动修复（幻觉纠正）
         │    │     ├─ 工具参数 JSON 验证和类型强制转换
         │    │     ├─ 参数截断检测（finish_reason=length 时）
         │    │     ├─ 工具调用防护（限制 delegate 深度、去重等）
         │    │     ├─ 并行/串行执行决策
         │    │     ├─ _execute_tool_calls() 执行工具
         │    │     ├─ 工具结果处理（大小限制、截断、错误包装）
         │    │     ├─ 结果按顺序追加到消息列表
         │    │     ├─ 上下文压缩检查（工具返回后可能超过阈值）
         │    │     ├─ 增量保存会话日志
         │    │     └─ 继续下一轮循环
         │    └─ 如果没有 tool_calls（最终回答）:
         │         ├─ 剥离 <think> 标签（如果存在）
         │         ├─ 空响应检测和恢复（多种重试策略）
         │         ├─ 部分流内容恢复（连接中断时使用已接收内容）
         │         └─ 跳出循环
         │
         └─ 5. 后处理
              ├─ 预算耗尽 → 请求摘要或返回部分结果
              ├─ 保存完整 trajectory 到会话数据库
              ├─ 清理任务资源（终端、浏览器、MCP 会话等）
              ├─ 持久化会话元数据（token 用量、成本、耗时等）
              ├─ 后台 memory/skill 审查（异步，不阻塞响应）
              └─ 调用 post_llm_call 插件钩子
```

### 消息格式（OpenAI 格式，内部统一表示）

```python
# 系统消息
{"role": "system", "content": "You are Hermes Agent..."}

# 用户消息
{"role": "user", "content": "帮我写一个 Python 脚本"}

# 助手消息（带工具调用）
{
    "role": "assistant",
    "content": "我来帮你写...",
    "reasoning": "<think>用户需要Python脚本，我需要先确认需求...</think>",  # 内部字段，存储思考内容
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
| `invalid_request` | 400 | 参数修正或重试 |
| `timeout` | N/A | 重试或切换 provider |
| `connection_error` | N/A | 重试或切换 provider |
| `unknown` | N/A | 降级到 fallback |

### 5.2 Jittered Backoff 重试策略

```python
def jittered_backoff(attempt, base_delay=5.0, max_delay=120.0, jitter_ratio=0.5):
    delay = min(base_delay * (2 ** (attempt - 1)), max_delay)
    jitter = random.uniform(0, jitter_ratio * delay)
    return delay + jitter
```

使用去相关抖动避免多会话同时重试的"惊群效应"。重试间隔从 5s 开始指数增长，最长 2 分钟。

### 5.3 特殊恢复机制

- **Unicode 恢复**：surrogate 字符和 ASCII 编码系统的两阶段清洗
- **上下文压缩**：超长对话自动压缩中间历史，保留首尾关键信息
- **空响应恢复**：thinking-only prefill、post-tool nudge、prior-turn fallback 三重恢复
- **截断工具调用检测**：通过参数是否以 `}` 结尾检测截断，避免执行不完整的工具调用
- **思考预算耗尽检测**：模型用完输出 token 但没有生成可见内容时，返回友好提示
- **部分流恢复**：连接中断时使用已接收的部分内容，避免完全重试
- **MCP 工具动态刷新**：MCP 服务器工具变更时自动更新注册表，无需重启

---

## 六、关键代码片段分析

### 6.1 主循环终止条件

```python
while (api_call_count < self.max_iterations and self.iteration_budget.remaining > 0) or self._budget_grace_call:
```

两个独立的终止条件：`api_call_count` 是本 turn 内的计数，`iteration_budget` 是跨 turn 的全局预算。`_budget_grace_call` 允许预算耗尽后再执行一次迭代，用于生成最终回答。

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
            reserved_paths.append(scoped_path)
    return True
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

解决 LLM 常见的类型错误：`"42"` → `42`，`"true"` → `True`，`"1.23"` → `1.23`。支持联合类型（如 `["integer", "string"]`）的自动适配。

### 6.4 工具注册机制

```python
# registry.py 核心注册逻辑
def register(
    self,
    name: str,
    toolset: str,
    schema: dict,
    handler: Callable,
    check_fn: Callable = None,
    requires_env: list = None,
    is_async: bool = False,
    description: str = "",
    emoji: str = "",
    max_result_size_chars: int | float | None = None,
):
    with self._lock:
        existing = self._tools.get(name)
        if existing and existing.toolset != toolset:
            # 允许 MCP 工具覆盖，禁止内置工具被覆盖
            both_mcp = existing.toolset.startswith("mcp-") and toolset.startswith("mcp-")
            if not both_mcp:
                logger.error(f"Tool {name} registration rejected: shadowing existing tool from {existing.toolset}")
                return
        self._tools[name] = ToolEntry(...)
```

### 6.5 工具调度逻辑

```python
def dispatch(self, name: str, args: dict, **kwargs) -> str:
    entry = self.get_entry(name)
    if not entry:
        return json.dumps({"error": f"Unknown tool: {name}"})
    try:
        if entry.is_async:
            from model_tools import _run_async
            return _run_async(entry.handler(args, **kwargs))
        return entry.handler(args, **kwargs)
    except Exception as e:
        logger.exception(f"Tool {name} dispatch error: {e}")
        return json.dumps({"error": f"Tool execution failed: {type(e).__name__}: {e}"})
```

自动处理同步/异步工具的执行，统一错误格式，避免工具异常导致整个 Agent 崩溃。

---

## 七、面试八股文

### 7.1 LLM Agent 的 ReAct 循环是什么？实现中有哪些关键考量？

**ReAct (Reasoning + Acting)** 是一种让 LLM 交替进行推理和行动的范式。模型先"思考"（reasoning），再"行动"（调用工具），观察工具结果后继续推理，直到完成任务。

**关键考量**：
1. 终止条件多样化：不只有 `max_iterations`，还有 `IterationBudget`、`_budget_grace_call`、中断信号等
2. 无限循环防护：多种重试计数器各有上限，成功执行工具后重置
3. 空响应恢复：模型可能返回只有 thinking 没有 content 的响应，需要多重恢复策略
4. 上下文膨胀控制：实时监控 token 用量，自动触发压缩
5. 多 provider 兼容：同一个循环需要处理不同 API 格式和错误类型
6. 工具执行安全：并行执行的线程安全、路径冲突检测、资源限制
7. 用户体验：流式输出、进度反馈、中断支持、错误友好提示

### 7.2 Function Calling / Tool Use 的实现原理

**定义阶段**：工具通过 `tools/registry.py` 自注册，声明 JSON Schema。`get_tool_definitions()` 根据 toolset 过滤后返回 OpenAI 格式的 `tools` 参数。

**调用阶段**：
1. API 返回 `tool_calls`
2. Agent 验证工具名称，自动修复幻觉名称
3. 验证 JSON 参数，强制类型转换
4. Guardrails：限制 delegate 数量，去重，路径冲突检测
5. 执行：根据工具特性选择并行/串行
6. 结果处理：大小限制，错误包装，格式标准化
7. 结果以 `tool` role 消息追加到消息列表
8. 循环继续

### 7.3 如何处理 LLM 返回的 tool_calls？

1. **名称验证**：遍历 `tool_calls`，检查是否在 `valid_tool_names` 中
2. **自动修复**：`_repair_tool_call()` 尝试模糊匹配幻觉工具名（编辑距离 <= 2）
3. **参数验证**：空字符串 → `{}`，截断检测（参数是否以 `}`/`]` 结尾）
4. **类型转换**：`coerce_tool_args()` 将字符串参数转换为 schema 声明的类型
5. **Guardrails**：限制 delegate 深度 <= 2，去重重复工具调用，检查资源权限
6. **执行**：根据工具特性选择并行/串行执行策略
7. **结果注入**：以 `tool` role 消息追加到对话历史，保持调用顺序

### 7.4 Agent 循环的终止条件有哪些？

至少有 **12 种** 终止条件：
1. `api_call_count >= max_iterations`（本轮迭代次数耗尽）
2. `iteration_budget.remaining <= 0`（全局迭代预算耗尽且无 grace call）
3. `_interrupt_requested`（用户中断、超时、会话过期）
4. `final_response is not None`（模型返回最终回答，无工具调用）
5. 非可重试的客户端错误（参数错误、权限不足等）
6. 重试次数耗尽且无 fallback 可用
7. 上下文压缩耗尽（多次压缩后仍超过窗口限制）
8. 空响应重试耗尽（3 次空响应后放弃）
9. 无效工具调用重试耗尽（3 次无效工具后放弃）
10. 不完整 scratchpad 重试耗尽（3 次截断后放弃）
11. 思考预算耗尽（模型用完输出 token 但无可见内容）
12. `_budget_grace_call` 完成（预算耗尽后的最后一次迭代结束）

### 7.5 如何防止 Agent 陷入无限循环？

1. **`IterationBudget`**：线程安全的全局计数器，跨轮次限制总迭代次数
2. **多重试计数器归零机制**：成功执行工具后重置空响应、无效工具等计数器
3. **`_budget_grace_call`**：预算耗尽后再给一次机会生成最终回答，避免卡在工具调用循环
4. **`execute_code` 退还机制**：程序化工具调用不消耗迭代预算，避免被恶意循环利用
5. **上下文压缩后重置重试计数器**：压缩后模型获得新的上下文，给予更多重试机会
6. **重复内容检测**：检测到连续 3 次相同的工具调用或回答时，终止循环并提示
7. **最长执行时间限制**：单个会话总执行时间不超过 30 分钟，超时自动终止

### 7.6 同步与异步代码如何桥接？

`_run_async()` 的三层策略：
1. **已有 event loop 运行中**：在新线程中调用 `asyncio.run()`，避免阻塞现有 loop
2. **工作线程**：使用线程本地的持久 event loop，避免 loop 关闭错误
3. **主线程**：使用全局持久 loop，复用连接池，提升性能

持久 loop 避免了 `asyncio.run()` 每次创建/销毁循环导致的 "Event loop is closed" 错误，同时复用 HTTP 连接，提升 API 调用性能。

### 7.7 Token 管理与上下文窗口限制

1. **粗略估计**：字符数 / 4 的启发式，用于预检查和快速估算
2. **精确统计**：API 响应的 `usage` 字段，用于准确的成本计算和压缩决策
3. **压缩阈值**：默认 50% 上下文窗口大小，到达时自动触发压缩
4. **压缩策略**：保留首尾 N 条消息（首 3 条系统/关键信息，尾 20 条最近对话），中间用辅助模型总结为单条摘要消息
5. **上下文探测**：API 返回 overflow 错误时逐步降低 context_length，自适应模型窗口
6. **Prompt Caching**：对 Claude 模型自动注入 cache_control breakpoints，缓存系统提示和历史消息，降低 token 成本 30-70%
7. **工具结果截断**：大输出结果自动截断，超出部分保存到临时文件，消息中只保留摘要和文件引用

---

## 八、面试官可能问的问题及详细回答

### Q1: 这个 Agent 的架构和 ChatGPT 的 Function Calling 有什么区别？

本实现是 OpenAI Function Calling 的 **完整 Agent 运行时封装**。ChatGPT 只返回一次 tool_calls，调用方需要自己实现循环。而 `AIAgent.run_conversation()` 实现了完整的 ReAct 循环，包括：重试、错误恢复、多 provider 故障转移、上下文压缩、并行工具执行、迭代预算管理、记忆系统、技能系统、插件系统等。

关键差异：
- 完整的对话状态管理，支持多轮工具调用自动循环
- 内置错误恢复和故障转移，无需调用方处理 API 错误
- 上下文自动管理，自动压缩适配模型窗口
- 支持并行工具执行，提升多任务处理效率
- 内置记忆和技能系统，支持长期记忆和工具抽象复用
- 插件扩展机制，支持自定义功能扩展
- 多平台适配（CLI、网关、API、Telegram 等）

### Q2: 为什么要区分 agent-level tools 和 registry tools？

某些工具（`todo`、`memory`、`session_search`、`delegate_task`）需要访问 Agent 实例的状态（会话数据、记忆存储、任务上下文等）。如果走 `registry.dispatch()`，这些状态无法传递，因为 registry 是全局单例。因此 `_invoke_tool()` 对这些工具直接处理，其余走注册表统一分发。

优点：
- 状态隔离：每个 Agent 实例有独立的状态，避免全局状态污染
- 性能优化：避免在工具调用时传递大量上下文参数
- 安全隔离：agent-level 工具可以访问敏感会话数据，registry 工具只能访问传入的参数
- 扩展性：新增 agent-level 工具无需修改 registry 代码

### Q3: 工具并行执行如何保证线程安全？

三重保障：
1. **并行判定**：只有只读工具和路径不冲突的文件工具才允许并行，避免竞态条件
2. **线程池限制**：最多 8 个并发，避免资源耗尽和 API 限流
3. **结果顺序保证**：按原始调用顺序写入结果，最终按顺序追加到消息列表，避免上下文混乱

额外保护：
- 每个工具执行有独立的超时限制（默认 5 分钟）
- 工具异常被捕获并包装为错误结果，不影响其他工具执行
- 线程本地存储隔离，避免全局变量污染
- 文件操作使用原子写入，避免文件损坏

### Q4: 上下文压缩是怎么工作的？

当 token 用量超过阈值（默认 50% 模型上下文窗口），`ContextCompressor` 将中间消息（保留首 3 条 + 尾 20 条）用辅助 LLM 生成摘要，替换为一条 summary 消息。最多尝试 3 次压缩。

压缩流程：
1. 保留前 3 条消息（系统提示、关键初始信息）
2. 保留最后 20 条消息（最近对话上下文）
3. 中间的消息批量发送给辅助 LLM 生成摘要
4. 摘要消息插入到保留消息中间，替换原中间消息
5. 重新估算 token 用量，如果仍超过阈值，重复压缩过程
6. 压缩后重置重试计数器，模型获得"干净"的上下文继续执行

压缩策略配置：
- 可自定义保留的首尾消息数量
- 可自定义压缩阈值（0.3-0.9 窗口大小）
- 可自定义辅助 LLM 模型（默认使用 GPT-3.5-turbo 降低成本）
- 可禁用压缩（适合短对话场景）

### Q5: delegate_task 子 Agent 是怎么工作的？

创建新的 `AIAgent` 实例，继承父 Agent 的 model/tools/config 但有独立的 `IterationBudget`（默认 50 次迭代）。子 Agent 的 `_delegate_depth` 递增，最大深度 2，防止无限递归。

子 Agent 特性：
- 上下文隔离：子 Agent 有独立的消息历史，不污染父 Agent 上下文
- 预算独立：子 Agent 有独立的迭代预算，避免消耗父 Agent 预算
- 结果汇总：子 Agent 完成任务后，结果被汇总为单条工具返回消息
- 资源限制：子 Agent 无法访问父 Agent 的敏感数据，只能访问传入的任务描述
- 深度限制：最大嵌套深度 2，防止代理套代理的无限递归

### Q6: 为什么 streaming 比 non-streaming 更受欢迎？

streaming 提供更细粒度的健康检查（90s  stale-stream 检测，60s 读超时），而非 streaming 模式在 provider 不返回响应时可能无限挂起。

其他优点：
- 用户体验更好：实时显示思考过程和回答，无需等待完整响应
- 内存效率更高：不需要一次性加载完整响应，适合大输出场景
- 中断支持更好：可以随时取消正在生成的响应
- 错误恢复更快：可以在流中断时使用已接收的部分内容，无需完全重试
- 下游处理更快：TTS、实时翻译等下游系统可以并行处理流式输出

### Q7: 中断机制如何实现？

`_interrupt_requested` 标志位通过 `set_interrupt()` 设置，绑定到执行线程 ID。主循环在每个迭代开始、工具执行前、重试等待期间检查此标志。

中断流程：
1. 用户发送中断信号（Ctrl+C、API 取消请求、新消息覆盖等）
2. `set_interrupt()` 被调用，设置对应线程的中断标志
3. 主循环在下一个检查点检测到标志，终止执行
4. 清理正在执行的工具（终端、浏览器等资源）
5. 返回已生成的部分结果和中断状态

设计亮点：
- 线程级中断隔离，多个 Agent 实例同时运行时互不影响
- 多检查点设计，响应延迟 < 1 秒
- 资源自动清理，避免僵尸进程和资源泄漏
- 支持优雅降级，返回已生成的部分结果，而非完全丢弃

### Q8: 如何处理 LLM 幻觉出的工具名？

1. **`_repair_tool_call()`** 尝试模糊匹配已知工具名，编辑距离 <= 2 时自动修复
2. 如果无法修复，返回工具错误消息让模型自行纠正，最多重试 3 次
3. 3 次失败后终止循环，返回部分结果和错误提示

模糊匹配示例：
- "web_sarch" → "web_search"
- "wite_file" → "write_file"
- "termial" → "terminal"

### Q9: Credential Pool 是怎么用的？

当主 API key 被限流时，从 pool 中取出下一个可用凭证，重建 client 并重试。Pool 追踪每个凭证的可用状态（限流、失效、余额不足等），自动轮换到健康凭证。

凭证池特性：
- 支持无限多凭证，轮询使用
- 自动标记失效凭证，避免重复使用
- 支持凭证健康检查，定期重试失效凭证
- 支持按优先级使用，优先使用低成本/高配额凭证
- 完全透明，上层代码无需感知凭证轮换

### Q10: Session 持久化的双写策略？

每个 turn 结束时双写：JSON 日志（完整原始消息） + SQLite 数据库（结构化查询）。

持久化内容：
- JSON 日志：完整的消息历史、API 请求/响应、工具调用详情、错误栈等，用于调试和审计
- SQLite 数据库：结构化元数据（会话 ID、用户 ID、平台、时间、token 用量、成本、完成状态等），用于统计分析和查询

设计亮点：
- 异步写入，不阻塞主响应路径
- 写入失败不影响主流程，仅记录警告日志
- 支持自定义存储后端（MySQL、PostgreSQL、S3 等）
- 自动清理旧会话，支持保留策略配置

### Q11: Tool result storage 是什么？

超大工具输出保存到临时文件，在消息中只保留文件路径引用，避免上下文膨胀。工具结果超过 `max_result_size_chars`（默认 10000 字符）时自动触发。

优点：
- 避免工具大输出导致上下文窗口溢出
- 支持后续工具调用读取文件内容，实现大结果的分块处理
- 临时文件自动清理，会话结束后删除
- 可配置阈值，适应不同模型的上下文窗口大小

### Q12: Checkpoint Manager 的作用？

在执行破坏性操作前自动创建文件系统快照，允许用户通过 `/undo` 回滚。支持多层撤销，最多保留 50 个历史快照。

支持的操作：
- 文件写入、删除、修改
- 终端命令执行（如 git commit、npm install 等）
- 数据库修改操作
- 系统配置变更

实现原理：
- 操作前创建当前工作目录的快照（增量备份，仅保存变更文件）
- 记录操作前后的文件哈希和内容
- 用户调用 `/undo` 时恢复到对应快照
- 会话结束后自动清理所有快照

### Q13: Background Memory/Skill Review 是怎么工作的？

Agent 循环结束后，在守护线程中创建临时 Agent fork，让模型决定是否保存到 memory/skill。完全异步，不影响主响应。

记忆审查流程：
1. 提取本次对话的关键信息和有用知识
2. 判断是否值得长期保存（重要性评分 >= 0.7）
3. 生成记忆条目，包括内容、关键词、时间戳等
4. 保存到向量数据库，支持后续语义检索

技能审查流程：
1. 提取本次对话中使用的复杂工具序列和工作流
2. 抽象为可复用的技能模板，参数化可变部分
3. 生成技能描述和使用示例
4. 保存到技能库，支持后续直接调用

优点：
- 完全异步，不影响用户响应时间
- 自动总结，无需用户手动保存
- 增量学习，知识和技能随使用自动增长
- 可配置审查频率和阈值，避免噪音

### Q14: 如何处理模型的 "thinking budget exhausted"？

直接返回用户友好的提示消息，建议降低 reasoning effort 或增加 max_tokens。不浪费重试次数。

检测逻辑：
- 响应 finish_reason = length
- 内容以 <think> 开头，且没有闭合标签
- 或者内容中只有 thinking 标签，没有可见内容
- 或者 thinking 标签占了全部输出 token

用户提示示例：
```
⚠️ **思考预算耗尽**

模型用完了所有输出 token 来思考，但还没来得及生成实际回答。

建议：
1. 降低思考强度：使用 `/think low` 或 `/think minimal`
2. 增加输出 token 限制：在配置中调整 `model.max_tokens` 参数
3. 简化问题：将复杂问题拆分为多个小问题逐步解决
```

### Q15: `_SafeWriter` 解决了什么问题？

当 Agent 作为 systemd 服务运行时，stdout/stderr pipe 可能断开。`_SafeWriter` 透明包装所有 stdio 操作，静默吞掉 `OSError`，避免进程意外退出。

实现原理：
- 拦截所有 stdout/stderr 写入操作
- 捕获 BrokenPipeError、OSError 等写入错误
- 静默忽略错误，不抛出异常
- 可选记录错误日志到文件，便于调试
- 自动降级为 no-op 当管道完全断开时

---

## 九、学习路径建议

### 阶段 1：理解骨架（1-2 小时）
1. `model_tools.py`（全文 563 行）-- 工具注册和分发核心
2. `tools/registry.py`（前 100 行）-- 工具自注册机制和核心接口
3. `run_agent.py:535-665` -- `AIAgent.__init__` 方法，理解核心参数

### 阶段 2：核心循环（2-3 小时）
4. `run_agent.py:8102-8470` -- `run_conversation()` 前半部分，初始化和准备逻辑
5. `run_agent.py:8471-9200` -- 主循环和 API 调用逻辑
6. `run_agent.py:10360-10589` -- tool_calls 处理和验证逻辑
7. `run_agent.py:10590-10899` -- 最终回答处理和空响应恢复
8. `run_agent.py:10900-11200` -- 后处理和资源清理逻辑

### 阶段 3：工具执行（1-2 小时）
9. `run_agent.py:7158-7267` -- `_execute_tool_calls` 入口和并行判定
10. `run_agent.py:7268-7530` -- 并行执行路径和线程池管理
11. `run_agent.py:7531-7909` -- 串行执行路径和单工具调用逻辑
12. `model_tools.py:421-534` -- `handle_function_call` 分发逻辑

### 阶段 4：错误恢复（2-3 小时）
13. `agent/error_classifier.py` -- 错误分类体系和 FailoverReason 定义
14. `agent/retry_utils.py` -- 退避策略和重试工具函数
15. `run_agent.py:8671-9200` -- API 调用重试和故障转移逻辑
16. `run_agent.py:9201-10113` -- 异常处理和恢复策略实现

### 阶段 5：高级特性（可选，3-5 小时）
17. `agent/context_compressor.py` -- 上下文压缩实现
18. `memory/` 目录 -- 记忆系统实现
19. `skills/` 目录 -- 技能系统实现
20. `plugins/` 目录 -- 插件系统实现
21. `tools/mcp_tool.py` -- MCP 工具集成实现