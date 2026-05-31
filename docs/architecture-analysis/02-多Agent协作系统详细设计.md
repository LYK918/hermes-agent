# 02 - 多 Agent 协作系统详细设计

> 本文档基于 hermes-agent 项目源码深度分析，面向高级工程师面试准备，涵盖子Agent委派、MoA多模型协作、ACP协议适配三大协作模式的完整设计细节。

---

## 1. 多Agent协作架构总览

### 1.1 ASCII架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户请求入口                                  │
│              (CLI / Gateway / ACP / Cron)                           │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      主 Agent (AIAgent)                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ 对话管理      │  │ 工具调度      │  │ 回调系统                  │  │
│  │ run_conver-  │  │ _run_tool()  │  │ tool_progress_callback   │  │
│  │ sation()     │  │              │  │ thinking_callback        │  │
│  └──────────────┘  └──────┬───────┘  │ step_callback            │  │
│                           │          │ message_callback         │  │
│                           │          │ status_callback          │  │
│                           │          │ clarify_callback         │  │
│                           │          │ stream_delta_callback    │  │
│                           │          └──────────────────────────┘  │
└───────────────────────────┼────────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────────┐
│  委派模式      │  │  MoA模式       │  │  ACP协议模式       │
│  (Delegation) │  │  (Multi-Agent)│  │  (ACP Adapter)    │
│               │  │               │  │                   │
│ delegate_task │  │ mixture_of_   │  │ HermesACPAgent    │
│               │  │ agents_tool   │  │ SessionManager    │
└───────┬───────┘  └───────┬───────┘  └─────────┬─────────┘
        │                  │                     │
        ▼                  ▼                     ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────────┐
│ 子Agent池      │  │ 参考模型池     │  │ 外部ACP客户端      │
│               │  │               │  │ (VS Code等)       │
│ child-0       │  │ Claude Opus   │  │                   │
│ child-1       │  │ Gemini 3 Pro  │  │ ThreadPool        │
│ child-2       │  │ GPT-5.4 Pro   │  │ Executor(4)       │
│               │  │ DeepSeek v3.2 │  │                   │
└───────┬───────┘  └───────┬───────┘  └─────────┬─────────┘
        │                  │                     │
        ▼                  ▼                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          工具层 (Tools)                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ terminal │ │ file     │ │ web      │ │ browser  │ │ code_exec│ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 三种协作模式对比

| 维度 | 委派模式 (Delegation) | MoA模式 (Mixture-of-Agents) | ACP模式 (Protocol) |
|------|----------------------|----------------------------|-------------------|
| **核心思想** | 任务拆分，并行执行 | 多模型投票，聚合最优 | 标准化协议，互操作 |
| **适用场景** | 独立子任务并行处理 | 复杂推理问题 | IDE/编辑器集成 |
| **上下文** | 完全隔离（新对话） | 共享用户prompt | 独立会话持久化 |
| **并发模型** | ThreadPoolExecutor | asyncio.gather | ThreadPoolExecutor(4) |
| **容错机制** | 子Agent崩溃不影响父Agent | 单模型失败继续 | 会话级隔离 |
| **深度限制** | MAX_DEPTH=2 | 固定2层 | N/A |
| **典型用例** | 并行代码审查、多路研究 | 数学证明、算法设计 | VS Code Copilot集成 |

### 1.3 每种模式的适用场景

**委派模式**适用于：
- 任务可以拆分为多个独立子任务
- 子任务之间无数据依赖
- 需要隔离中间过程（只返回最终摘要）
- 避免上下文窗口被中间数据淹没

**MoA模式**适用于：
- 极其困难的推理问题（数学、算法、多步分析）
- 单一模型容易出错或产生偏见的场景
- 需要多角度审视同一问题
- 质量优先于延迟的场景（5次API调用）

**ACP模式**适用于：
- IDE/编辑器需要标准化Agent通信协议
- 需要会话持久化和恢复
- 需要工具进度实时推送到客户端
- 多客户端共享同一Agent后端

---

## 2. 子Agent委派机制 (delegate_tool.py) 深度剖析

### 2.1 层级结构设计

#### 2.1.1 MAX_DEPTH=2 的设计理由

```python
# 文件: tools/delegate_tool.py, 第53行
MAX_DEPTH = 2  # parent (0) -> child (1) -> grandchild rejected (2)
```

**为什么是2而不是3或无限：**

1. **指数爆炸防护**：如果允许无限深度，一个父Agent可以派生3个子Agent，每个子Agent又派生3个，到第3层就是27个并发Agent，到第5层就是243个。这会迅速耗尽API配额、内存和线程资源。

2. **上下文质量递减**：每层委派都会丢失父Agent的完整上下文。子Agent只能通过`goal`和`context`参数获取信息，到第3层时信息已经严重碎片化，任务描述的质量会大幅下降。

3. **调试可观测性**：2层结构（父→子）足够扁平，人类可以理解整个执行链。3层以上的嵌套会让调试和日志分析变得极其困难。

4. **实际需求覆盖**：绝大多数任务拆分场景只需要一层委派。需要递归拆分的任务通常说明任务本身需要重新设计。

#### 2.1.2 深度检查的代码实现

```python
# 文件: tools/delegate_tool.py, 第646-654行
def delegate_task(..., parent_agent=None) -> str:
    if parent_agent is None:
        return tool_error("delegate_task requires a parent agent context.")

    # Depth limit
    depth = getattr(parent_agent, '_delegate_depth', 0)
    if depth >= MAX_DEPTH:
        return json.dumps({
            "error": (
                f"Delegation depth limit reached ({MAX_DEPTH}). "
                "Subagents cannot spawn further subagents."
            )
        })
```

深度值存储在每个`AIAgent`实例的`_delegate_depth`属性上：

```python
# 文件: run_agent.py, 第780行
self._delegate_depth = 0  # 0 = top-level agent, incremented for children
```

构建子Agent时递增：

```python
# 文件: tools/delegate_tool.py, 第380行
child._delegate_depth = getattr(parent_agent, '_delegate_depth', 0) + 1
```

**关键设计点**：深度检查发生在`delegate_task()`函数入口处，而不是`_build_child_agent()`中。这意味着即使构建成功，如果深度超限也会被拒绝。这是一种防御性编程——检查前置，构建后置。

#### 2.1.3 父子关系的建立和维护

父子关系通过`_active_children`列表和`_active_children_lock`互斥锁维护：

```python
# 文件: run_agent.py, 第781-782行
self._active_children = []      # Running child AIAgents (for interrupt propagation)
self._active_children_lock = threading.Lock()
```

子Agent注册到父Agent：

```python
# 文件: tools/delegate_tool.py, 第389-396行
# Register child for interrupt propagation
if hasattr(parent_agent, '_active_children'):
    lock = getattr(parent_agent, '_active_children_lock', None)
    if lock:
        with lock:
            parent_agent._active_children.append(child)
    else:
        parent_agent._active_children.append(child)
```

子Agent完成后注销：

```python
# 文件: tools/delegate_tool.py, 第603-612行
# Unregister child from interrupt propagation
if hasattr(parent_agent, '_active_children'):
    try:
        lock = getattr(parent_agent, '_active_children_lock', None)
        if lock:
            with lock:
                parent_agent._active_children.remove(child)
        else:
            parent_agent._active_children.remove(child)
    except (ValueError, UnboundLocalError) as e:
        logger.debug("Could not remove child from active_children: %s", e)
```

### 2.2 上下文隔离

#### 2.2.1 子Agent获得全新上下文的原因

子Agent不继承父Agent的对话历史，这是刻意设计：

1. **上下文窗口保护**：父Agent的对话历史可能已经很长，继承会导致子Agent的上下文窗口被无关信息填满。

2. **任务专注度**：子Agent只需要完成特定任务，不需要知道父Agent之前的对话内容。所有必要信息通过`goal`和`context`参数传递。

3. **防止信息泄露**：父Agent可能包含敏感的用户对话，子Agent不应该看到这些内容。

4. **并行安全**：多个子Agent同时运行时，如果共享上下文，会产生竞态条件。

#### 2.2.2 System Prompt 构建

```python
# 文件: tools/delegate_tool.py, 第90-122行
def _build_child_system_prompt(
    goal: str,
    context: Optional[str] = None,
    *,
    workspace_path: Optional[str] = None,
) -> str:
    parts = [
        "You are a focused subagent working on a specific delegated task.",
        "",
        f"YOUR TASK:\n{goal}",
    ]
    if context and context.strip():
        parts.append(f"\nCONTEXT:\n{context}")
    if workspace_path and str(workspace_path).strip():
        parts.append(
            "\nWORKSPACE PATH:\n"
            f"{workspace_path}\n"
            "Use this exact path for local repository/workdir operations..."
        )
    parts.append(
        "\nComplete this task using the tools available to you. "
        "When finished, provide a clear, concise summary of:\n"
        "- What you did\n"
        "- What you found or accomplished\n"
        "- Any files you created or modified\n"
        "- Any issues encountered\n\n"
        "Important workspace rule: Never assume a repository lives at "
        "/workspace/... or any other container-style path...\n\n"
        "Be thorough but concise -- your response is returned to the "
        "parent agent as a summary."
    )
    return "\n".join(parts)
```

System Prompt 结构：
1. **角色定义**："You are a focused subagent..."
2. **任务目标**：`YOUR TASK:` + goal
3. **上下文信息**：`CONTEXT:` + context（可选）
4. **工作空间路径**：`WORKSPACE PATH:` + workspace_path（可选，自动检测）
5. **输出格式要求**：明确要求返回摘要格式

#### 2.2.3 工具限制：DELEGATE_BLOCKED_TOOLS

```python
# 文件: tools/delegate_tool.py, 第32-38行
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # no recursive delegation
    "clarify",         # no user interaction
    "memory",          # no writes to shared MEMORY.md
    "send_message",    # no cross-platform side effects
    "execute_code",    # children should reason step-by-step, not write scripts
])
```

每个工具被禁止的原因：

| 工具 | 禁止原因 |
|------|---------|
| `delegate_task` | 防止递归委派。如果允许，子Agent可以继续派生子Agent，导致无限递归和资源耗尽。虽然有MAX_DEPTH检查，但直接禁止更安全。 |
| `clarify` | 子Agent无法与用户交互。`clarify`工具用于向用户提问，但子Agent运行在后台线程中，没有用户界面。允许它会导致工具调用失败或阻塞。 |
| `memory` | 防止写入共享的MEMORY.md文件。多个子Agent同时写入会产生竞态条件，导致数据损坏。父Agent负责决定哪些信息值得保存。 |
| `send_message` | 防止跨平台副作用。子Agent不应该直接向Telegram/Discord等平台发送消息，这会绕过父Agent的消息路由逻辑。 |
| `execute_code` | 强制子Agent逐步推理。`execute_code`允许执行任意Python脚本，这会让子Agent跳过工具调用的逐步推理过程，降低可调试性和可追踪性。 |

#### 2.2.4 终端会话隔离

每个子Agent获得独立的`task_id`（即`session_id`），用于隔离终端环境：

```python
# 文件: tools/delegate_tool.py, 第366行（AIAgent构造参数）
session_db=getattr(parent_agent, '_session_db', None),
parent_session_id=getattr(parent_agent, 'session_id', None),
```

子Agent的`session_id`在`AIAgent.__init__()`中自动生成（UUID），确保每个子Agent有独立的：
- 终端沙箱环境
- 文件操作缓存
- 进程注册表

#### 2.2.5 迭代预算独立

```python
# 文件: tools/delegate_tool.py, 第376行
iteration_budget=None,  # fresh budget per subagent
```

每个子Agent获得独立的迭代预算，不受父Agent的迭代消耗影响。默认50次迭代（`DEFAULT_MAX_ITERATIONS = 50`），可通过配置覆盖：

```python
# 文件: tools/delegate_tool.py, 第80行
DEFAULT_MAX_ITERATIONS = 50
```

### 2.3 构建流程 (_build_child_agent)

#### 2.3.1 完整的构建步骤

```python
# 文件: tools/delegate_tool.py, 第238-397行
def _build_child_agent(
    task_index: int,
    goal: str,
    context: Optional[str],
    toolsets: Optional[List[str]],
    model: Optional[str],
    max_iterations: int,
    parent_agent,
    # Credential overrides from delegation config
    override_provider: Optional[str] = None,
    override_base_url: Optional[str] = None,
    override_api_key: Optional[str] = None,
    override_api_mode: Optional[str] = None,
    # ACP transport overrides
    override_acp_command: Optional[str] = None,
    override_acp_args: Optional[List[str]] = None,
):
```

**步骤1：确定工具集继承逻辑（第270-291行）**

```python
parent_enabled = getattr(parent_agent, "enabled_toolsets", None)
if parent_enabled is not None:
    parent_toolsets = set(parent_enabled)
elif parent_agent and hasattr(parent_agent, "valid_tool_names"):
    # enabled_toolsets is None (all tools) — derive from loaded tool names
    import model_tools
    parent_toolsets = {
        ts for name in parent_agent.valid_tool_names
        if (ts := model_tools.get_toolset_for_tool(name)) is not None
    }
else:
    parent_toolsets = set(DEFAULT_TOOLSETS)
```

工具集继承规则：
- 如果父Agent指定了`enabled_toolsets`，子Agent继承这些工具集
- 如果父Agent启用所有工具（`enabled_toolsets=None`），从已加载的工具名反推工具集
- 否则使用默认工具集：`["terminal", "file", "web"]`

然后与请求的工具集取交集，再移除blocked工具集：

```python
if toolsets:
    child_toolsets = _strip_blocked_tools([t for t in toolsets if t in parent_toolsets])
```

**步骤2：构建子Agent的System Prompt（第293-294行）**

```python
workspace_hint = _resolve_workspace_hint(parent_agent)
child_prompt = _build_child_system_prompt(goal, context, workspace_path=workspace_hint)
```

**步骤3：提取父Agent的API Key（第296-298行）**

```python
parent_api_key = getattr(parent_agent, "api_key", None)
if (not parent_api_key) and hasattr(parent_agent, "_client_kwargs"):
    parent_api_key = parent_agent._client_kwargs.get("api_key")
```

**步骤4：构建进度回调（第301行）**

```python
child_progress_cb = _build_child_progress_callback(task_index, parent_agent)
```

**步骤5：解析有效凭证（第321-327行）**

```python
effective_model = model or parent_agent.model
effective_provider = override_provider or getattr(parent_agent, "provider", None)
effective_base_url = override_base_url or parent_agent.base_url
effective_api_key = override_api_key or parent_api_key
effective_api_mode = override_api_mode or getattr(parent_agent, "api_mode", None)
```

优先级：配置覆盖 > 父Agent继承

**步骤6：解析推理配置（第330-346行）**

```python
parent_reasoning = getattr(parent_agent, "reasoning_config", None)
child_reasoning = parent_reasoning
try:
    delegation_cfg = _load_config()
    delegation_effort = str(delegation_cfg.get("reasoning_effort") or "").strip()
    if delegation_effort:
        from hermes_constants import parse_reasoning_effort
        parsed = parse_reasoning_effort(delegation_effort)
        if parsed is not None:
            child_reasoning = parsed
```

**步骤7：创建AIAgent实例（第348-377行）**

```python
child = AIAgent(
    base_url=effective_base_url,
    api_key=effective_api_key,
    model=effective_model,
    provider=effective_provider,
    api_mode=effective_api_mode,
    acp_command=effective_acp_command,
    acp_args=effective_acp_args,
    max_iterations=max_iterations,
    max_tokens=getattr(parent_agent, "max_tokens", None),
    reasoning_config=child_reasoning,
    prefill_messages=getattr(parent_agent, "prefill_messages", None),
    enabled_toolsets=child_toolsets,
    quiet_mode=True,
    ephemeral_system_prompt=child_prompt,
    log_prefix=f"[subagent-{task_index}]",
    platform=parent_agent.platform,
    skip_context_files=True,
    skip_memory=True,
    clarify_callback=None,
    thinking_callback=child_thinking_cb,
    session_db=getattr(parent_agent, '_session_db', None),
    parent_session_id=getattr(parent_agent, 'session_id', None),
    providers_allowed=parent_agent.providers_allowed,
    providers_ignored=parent_agent.providers_ignored,
    providers_order=parent_agent.providers_order,
    provider_sort=parent_agent.provider_sort,
    tool_progress_callback=child_progress_cb,
    iteration_budget=None,  # fresh budget per subagent
)
```

关键参数说明：
- `quiet_mode=True`：子Agent不输出到终端
- `skip_context_files=True`：不加载SOUL.md等上下文文件
- `skip_memory=True`：不加载共享记忆
- `clarify_callback=None`：禁止用户交互
- `ephemeral_system_prompt=child_prompt`：使用专门构建的子Agent prompt

**步骤8：设置深度和凭证池（第379-396行）**

```python
child._delegate_depth = getattr(parent_agent, '_delegate_depth', 0) + 1
child._credential_pool = child_pool  # 共享凭证池
```

#### 2.3.2 凭证共享机制（租约）

```python
# 文件: tools/delegate_tool.py, 第816-845行
def _resolve_child_credential_pool(effective_provider, parent_agent):
    if not effective_provider:
        return getattr(parent_agent, "_credential_pool", None)

    parent_provider = getattr(parent_agent, "provider", None) or ""
    parent_pool = getattr(parent_agent, "_credential_pool", None)
    if parent_pool is not None and effective_provider == parent_provider:
        return parent_pool  # 同provider共享池

    try:
        from agent.credential_pool import load_pool
        pool = load_pool(effective_provider)
        if pool is not None and pool.has_credentials():
            return pool
    except Exception as exc:
        logger.debug("Could not load credential pool for child provider '%s': %s", ...)
    return None
```

租约机制在`_run_single_child`中执行：

```python
# 文件: tools/delegate_tool.py, 第421-431行
child_pool = getattr(child, '_credential_pool', None)
leased_cred_id = None
if child_pool is not None:
    leased_cred_id = child_pool.acquire_lease()
    if leased_cred_id is not None:
        try:
            leased_entry = child_pool.current()
            if leased_entry is not None and hasattr(child, '_swap_credential'):
                child._swap_credential(leased_entry)
        except Exception as exc:
            logger.debug("Failed to bind child to leased credential: %s", exc)
```

子Agent完成后释放租约：

```python
# 文件: tools/delegate_tool.py, 第586-590行
if child_pool is not None and leased_cred_id is not None:
    try:
        child_pool.release_lease(leased_cred_id)
    except Exception as exc:
        logger.debug("Failed to release credential lease: %s", exc)
```

### 2.4 执行流程 (_run_single_child)

#### 2.4.1 线程模型

```python
# 文件: tools/delegate_tool.py, 第399-405行
def _run_single_child(
    task_index: int,
    goal: str,
    child=None,
    parent_agent=None,
    **_kwargs,
) -> Dict[str, Any]:
```

单任务模式直接在主线程执行：

```python
# 文件: tools/delegate_tool.py, 第731-735行
if n_tasks == 1:
    _i, _t, child = children[0]
    result = _run_single_child(0, _t["goal"], child, parent_agent)
    results.append(result)
```

批量模式使用ThreadPoolExecutor：

```python
# 文件: tools/delegate_tool.py, 第741-743行
with ThreadPoolExecutor(max_workers=max_children) as executor:
    futures = {}
    for i, t, child in children:
        future = executor.submit(_run_single_child, ...)
```

#### 2.4.2 结果收集：只返回最终摘要

子Agent的结果结构：

```python
# 文件: tools/delegate_tool.py, 第549-566行
entry: Dict[str, Any] = {
    "task_index": task_index,
    "status": status,           # "completed" | "failed" | "interrupted"
    "summary": summary,         # final_response — 子Agent的最终回复
    "api_calls": api_calls,     # API调用次数
    "duration_seconds": duration,
    "model": _model,
    "exit_reason": exit_reason, # "completed" | "max_iterations" | "interrupted"
    "tokens": {
        "input": _input_tokens,
        "output": _output_tokens,
    },
    "tool_trace": tool_trace,   # 工具调用追踪（用于调试）
}
```

父Agent只看到`summary`字段，不暴露中间工具调用的详细结果。`tool_trace`仅包含元数据（工具名、参数大小、结果大小），不包含实际内容。

#### 2.4.3 异常处理：子Agent崩溃不影响父Agent

```python
# 文件: tools/delegate_tool.py, 第568-578行
except Exception as exc:
    duration = round(time.monotonic() - child_start, 2)
    logging.exception(f"[subagent-{task_index}] failed")
    return {
        "task_index": task_index,
        "status": "error",
        "summary": None,
        "error": str(exc),
        "api_calls": 0,
        "duration_seconds": duration,
    }
```

即使子Agent抛出未捕获异常，也会被捕获并转换为错误结果返回给父Agent。父Agent永远不会因子Agent崩溃而中断。

#### 2.4.4 超时控制

子Agent本身的迭代预算就是超时机制：

```python
# 文件: tools/delegate_tool.py, 第80行
DEFAULT_MAX_ITERATIONS = 50
```

每次迭代包含一次API调用+工具执行，50次迭代大约对应5-15分钟的执行时间。没有额外的wall-clock超时，因为不同任务的合理执行时间差异很大。

### 2.5 批量并行模式

#### 2.5.1 max_concurrent_children=3 的选择理由

```python
# 文件: tools/delegate_tool.py, 第52行
_DEFAULT_MAX_CONCURRENT_CHILDREN = 3
```

选择3的原因：

1. **API并发限制**：大多数LLM API提供商对单个API Key有并发请求限制（通常5-10个）。3个子Agent加上父Agent本身，总共4个并发请求，留有余量。

2. **资源平衡**：每个子Agent需要独立的终端环境、文件句柄和内存。3个是性能和资源消耗的平衡点。

3. **实用性**：大多数任务拆分场景不需要超过3个并行子任务。超过3个通常说明任务拆分粒度太细。

可通过配置覆盖：

```python
# 文件: tools/delegate_tool.py, 第56-79行
def _get_max_concurrent_children() -> int:
    cfg = _load_config()
    val = cfg.get("max_concurrent_children")
    if val is not None:
        try:
            return max(1, int(val))
        except (TypeError, ValueError):
            ...
    env_val = os.getenv("DELEGATION_MAX_CONCURRENT_CHILDREN")
    if env_val:
        try:
            return max(1, int(env_val))
        except (TypeError, ValueError):
            pass
    return _DEFAULT_MAX_CONCURRENT_CHILDREN
```

配置优先级：config.yaml > 环境变量 > 默认值(3)

#### 2.5.2 线程池大小计算

```python
# 文件: tools/delegate_tool.py, 第741行
with ThreadPoolExecutor(max_workers=max_children) as executor:
```

线程池大小等于`max_children`（即`max_concurrent_children`）。每个线程运行一个子Agent的`_run_single_child`。

#### 2.5.3 结果合并策略

```python
# 文件: tools/delegate_tool.py, 第753-793行
for future in as_completed(futures):
    try:
        entry = future.result()
    except Exception as exc:
        idx = futures[future]
        entry = {
            "task_index": idx,
            "status": "error",
            "summary": None,
            "error": str(exc),
            "api_calls": 0,
            "duration_seconds": 0,
        }
    results.append(entry)
    ...

# Sort by task_index so results match input order
results.sort(key=lambda r: r["task_index"])
```

结果按`task_index`排序，确保输出顺序与输入顺序一致。`as_completed`用于实时获取完成的任务，但最终结果按原始顺序排列。

#### 2.5.4 部分失败处理

部分子Agent失败不影响其他子Agent：

```python
# 文件: tools/delegate_tool.py, 第765-766行
completed_count = 0
...
for future in as_completed(futures):
    try:
        entry = future.result()
    except Exception as exc:
        entry = {"task_index": idx, "status": "error", ...}
    results.append(entry)
    completed_count += 1
```

每个任务的结果独立收集，一个任务的异常不会影响其他任务。父Agent收到的结果数组中，每个条目都有独立的`status`字段。

### 2.6 心跳机制

#### 2.6.1 为什么需要心跳

Gateway有不活跃超时检测机制。当Agent长时间没有活动时，Gateway会认为Agent卡死并终止它。但是子Agent执行期间，父Agent的`_last_activity_ts`会冻结（因为父Agent在等待子Agent完成），导致Gateway误判为不活跃。

```python
# 文件: tools/delegate_tool.py, 第433-437行
# Heartbeat: periodically propagate child activity to the parent so the
# gateway inactivity timeout doesn't fire while the subagent is working.
# Without this, the parent's _last_activity_ts freezes when delegate_task
# starts and the gateway eventually kills the agent for "no activity".
_heartbeat_stop = threading.Event()
```

#### 2.6.2 心跳间隔（30秒）

```python
# 文件: tools/delegate_tool.py, 第81行
_HEARTBEAT_INTERVAL = 30  # seconds between parent activity heartbeats during delegation
```

30秒间隔与Gateway的不活跃检测周期匹配。Gateway通常每5秒轮询一次，30秒的心跳间隔确保父Agent的活动时间戳始终保持新鲜。

#### 2.6.3 心跳线程的生命周期

```python
# 文件: tools/delegate_tool.py, 第439-468行
def _heartbeat_loop():
    while not _heartbeat_stop.wait(_HEARTBEAT_INTERVAL):
        if parent_agent is None:
            continue
        touch = getattr(parent_agent, '_touch_activity', None)
        if not touch:
            continue
        desc = f"delegate_task: subagent {task_index} working"
        try:
            child_summary = child.get_activity_summary()
            child_tool = child_summary.get("current_tool")
            child_iter = child_summary.get("api_call_count", 0)
            child_max = child_summary.get("max_iterations", 0)
            if child_tool:
                desc = (f"delegate_task: subagent running {child_tool} "
                        f"(iteration {child_iter}/{child_max})")
        except Exception:
            pass
        try:
            touch(desc)
        except Exception:
            pass

_heartbeat_thread = threading.Thread(target=_heartbeat_loop, daemon=True)
_heartbeat_thread.start()
```

心跳线程是daemon线程，随主线程退出而终止。在`finally`块中显式停止：

```python
# 文件: tools/delegate_tool.py, 第581-584行
finally:
    _heartbeat_stop.set()
    _heartbeat_thread.join(timeout=5)
```

#### 2.6.4 父子Agent活动传播

心跳不仅更新父Agent的`_last_activity_ts`，还携带子Agent的详细活动信息：

```python
child_summary = child.get_activity_summary()
child_tool = child_summary.get("current_tool")
child_iter = child_summary.get("api_call_count", 0)
child_max = child_summary.get("max_iterations", 0)
if child_tool:
    desc = (f"delegate_task: subagent running {child_tool} "
            f"(iteration {child_iter}/{child_max})")
```

这使得Gateway的超时诊断信息能够显示子Agent正在执行的具体工具和迭代进度。

### 2.7 中断传播

#### 2.7.1 Ctrl+C 如何传播到子Agent

中断传播路径：

```
用户 Ctrl+C
    → Gateway/CLI 检测到中断
    → parent_agent.interrupt(message)
    → 遍历 _active_children 列表
    → 每个 child.interrupt(message)
    → 子Agent 设置 _interrupt_requested = True
    → 子Agent 在下一个检查点退出
```

父Agent的`interrupt()`方法（run_agent.py, 第3040-3087行）：

```python
def interrupt(self, message: str = None) -> None:
    self._interrupt_requested = True
    self._interrupt_message = message

    # Signal all tools to abort
    if self._execution_thread_id is not None:
        _set_interrupt(True, self._execution_thread_id)

    # Propagate interrupt to any running child agents
    with self._active_children_lock:
        children_copy = list(self._active_children)
    for child in children_copy:
        try:
            child.interrupt(message)
        except Exception as e:
            logger.debug("Failed to propagate interrupt to child agent: %s", e)
```

#### 2.7.2 递归传播逻辑

中断是递归传播的：父Agent中断所有子Agent，子Agent再中断它们的子Agent（虽然MAX_DEPTH=2限制了深度，但代码是通用的）。每个Agent的`interrupt()`方法都会遍历自己的`_active_children`并调用它们的`interrupt()`。

#### 2.7.3 清理顺序

子Agent的清理在`_run_single_child`的`finally`块中执行：

```python
finally:
    # 1. 停止心跳线程
    _heartbeat_stop.set()
    _heartbeat_thread.join(timeout=5)

    # 2. 释放凭证租约
    if child_pool is not None and leased_cred_id is not None:
        child_pool.release_lease(leased_cred_id)

    # 3. 恢复父Agent的工具名（全局状态）
    saved_tool_names = getattr(child, "_delegate_saved_tool_names", None)
    if isinstance(saved_tool_names, list):
        model_tools._last_resolved_tool_names = list(saved_tool_names)

    # 4. 从父Agent的活跃子Agent列表中注销
    if hasattr(parent_agent, '_active_children'):
        with lock:
            parent_agent._active_children.remove(child)

    # 5. 关闭子Agent的所有资源
    try:
        if hasattr(child, 'close'):
            child.close()
    except Exception:
        logger.debug("Failed to close child agent after delegation")
```

`close()`方法（run_agent.py, 第3182-3231行）会清理：
1. 后台进程（ProcessRegistry）
2. 终端沙箱环境
3. 浏览器daemon会话
4. 活跃的子Agent（递归关闭）
5. OpenAI/httpx客户端连接

---

## 3. MoA多模型协作 (mixture_of_agents_tool.py)

### 3.1 算法原理

#### 3.1.1 基于论文 arXiv:2406.04692v1

MoA（Mixture-of-Agents）是一种多模型协作方法论，通过分层架构利用多个LLM的集体优势，在复杂推理任务上实现最先进的性能。

核心思想：不同LLM在不同类型的任务上有各自的优势。通过让多个模型独立生成响应，然后由一个强模型综合这些响应，可以产生比任何单个模型更高质量的输出。

#### 3.1.2 为什么多模型协作比单一模型好

1. **多样性降低偏差**：每个模型都有自己的训练偏差和知识盲区。多个模型的响应覆盖了更广泛的知识空间。

2. **互相验证**：当多个模型对同一问题给出相似答案时，答案的可信度显著提高。当答案不一致时，聚合模型可以识别并选择最合理的答案。

3. **互补优势**：Claude擅长长文本理解和代码生成，GPT-5.4擅长数学推理，Gemini擅长多模态理解，DeepSeek擅长中文和代码。协作可以发挥各自优势。

4. **降低随机性**：单个模型的输出受temperature影响有随机性。多模型聚合可以平滑这种随机性。

#### 3.1.3 参考模型多样性的重要性

```python
# 文件: tools/mixture_of_agents_tool.py, 第63-68行
REFERENCE_MODELS = [
    "anthropic/claude-opus-4.6",
    "google/gemini-3-pro-preview",
    "openai/gpt-5.4-pro",
    "deepseek/deepseek-v3.2",
]
```

选择4个来自不同公司的模型，确保：
- 不同的训练数据（减少共同偏差）
- 不同的架构（Transformer变体、MoE等）
- 不同的对齐方式（RLHF、DPO等）
- 不同的推理风格（保守vs激进、详细vs简洁）

### 3.2 参考模型配置

#### 3.2.1 四个参考模型的特点

| 模型 | 提供商 | 特点和优势 |
|------|--------|-----------|
| Claude Opus 4.6 | Anthropic | 最强的长文本理解能力，优秀的代码生成，安全的输出 |
| Gemini 3 Pro | Google | 强大的多模态理解，优秀的数学推理，支持长上下文 |
| GPT-5.4 Pro | OpenAI | 广泛的世界知识，优秀的指令遵循，强大的推理能力 |
| DeepSeek v3.2 | DeepSeek | 优秀的中文理解，强大的代码能力，高性价比 |

#### 3.2.2 为什么选这4个模型

1. **顶级性能**：都是各自提供商的旗舰模型，在各类基准测试上名列前茅。
2. **API可用性**：都通过OpenRouter统一接入，简化了调用逻辑。
3. **成本平衡**：虽然都是高端模型，但通过OpenRouter的统一定价，成本可控。
4. **覆盖全面**：覆盖了中英文、代码、数学、推理等多个维度。

### 3.3 并行生成

#### 3.3.1 asyncio.gather 并行调用

```python
# 文件: tools/mixture_of_agents_tool.py, 第311-314行
model_results = await asyncio.gather(*[
    _run_reference_model_safe(model, user_prompt, REFERENCE_TEMPERATURE)
    for model in ref_models
])
```

`asyncio.gather`将4个模型的API调用并行执行，总耗时约等于最慢的那个模型的响应时间，而不是4个模型的总和。

#### 3.3.2 容错：单个模型失败不影响整体

```python
# 文件: tools/mixture_of_agents_tool.py, 第320-329行
for model_name, content, success in model_results:
    if success:
        successful_responses.append(content)
    else:
        failed_models.append(model_name)
```

每个模型的结果独立判断成功/失败，失败的模型被跳过。

#### 3.3.3 重试策略：指数退避，最多6次/模型

```python
# 文件: tools/mixture_of_agents_tool.py, 第104-176行
async def _run_reference_model_safe(
    model: str,
    user_prompt: str,
    temperature: float = REFERENCE_TEMPERATURE,
    max_tokens: int = 32000,
    max_retries: int = 6
) -> tuple[str, str, bool]:
    for attempt in range(max_retries):
        try:
            # ... API调用 ...
            response = await _get_openrouter_client().chat.completions.create(**api_params)
            content = extract_content_or_reasoning(response)
            if not content:
                # Reasoning-only response — retry
                if attempt < max_retries - 1:
                    await asyncio.sleep(min(2 ** (attempt + 1), 60))
                    continue
            return model, content, True
        except Exception as e:
            if attempt < max_retries - 1:
                # Exponential backoff: 2s, 4s, 8s, 16s, 32s, 60s
                sleep_time = min(2 ** (attempt + 1), 60)
                await asyncio.sleep(sleep_time)
            else:
                return model, error_msg, False
```

指数退避策略：
- 第1次重试：等待2秒
- 第2次重试：等待4秒
- 第3次重试：等待8秒
- 第4次重试：等待16秒
- 第5次重试：等待32秒
- 第6次重试：等待60秒（上限）

这种策略对rate limiting特别有效，给API提供商足够的恢复时间。

#### 3.3.4 最少1个成功的要求

```python
# 文件: tools/mixture_of_agents_tool.py, 第79行
MIN_SUCCESSFUL_REFERENCES = 1  # Minimum successful reference models needed to proceed
```

检查逻辑：

```python
# 文件: tools/mixture_of_agents_tool.py, 第335-336行
if successful_count < MIN_SUCCESSFUL_REFERENCES:
    raise ValueError(f"Insufficient successful reference models ({successful_count}/{len(ref_models)}).")
```

只要至少1个参考模型成功返回，就可以继续聚合。这提供了极高的容错性——即使3个模型都失败，只要有1个成功就能产出结果。

### 3.4 聚合策略

#### 3.4.1 Claude Opus 4.6 作为聚合模型

```python
# 文件: tools/mixture_of_agents_tool.py, 第72行
AGGREGATOR_MODEL = "anthropic/claude-opus-4.6"
```

选择Claude Opus 4.6作为聚合模型的原因：
1. **最强的综合能力**：在各类基准测试中综合排名最高
2. **优秀的长文本处理**：可以同时处理4个模型的完整响应
3. **批判性思维**：能够识别其他模型响应中的错误和偏见
4. **结构化输出**：能够生成高质量、结构化的综合回复

#### 3.4.2 temperature=0.4 的选择理由

```python
# 文件: tools/mixture_of_agents_tool.py, 第76行
AGGREGATOR_TEMPERATURE = 0.4  # Focused synthesis for consistency
```

- 参考模型使用`temperature=0.6`（较高，鼓励多样性）
- 聚合模型使用`temperature=0.4`（较低，追求一致性）

聚合模型的任务是综合已有响应，而不是创造新内容。较低的temperature确保输出更确定、更一致，减少幻觉风险。

#### 3.4.3 聚合Prompt的设计

```python
# 文件: tools/mixture_of_agents_tool.py, 第82-84行
AGGREGATOR_SYSTEM_PROMPT = """You have been provided with a set of responses from various open-source models to the latest user query. Your task is to synthesize these responses into a single, high-quality response. It is crucial to critically evaluate the information provided in these responses, recognizing that some of it may be biased or incorrect. Your response should not simply replicate the given answers but should offer a refined, accurate, and comprehensive reply to the instruction. Ensure your response is well-structured, coherent, and adheres to the highest standards of accuracy and reliability.

Responses from models:"""
```

Prompt的关键要素：
1. **明确任务**：综合多个响应为一个高质量响应
2. **批判性评估**：不要简单复制，要识别偏见和错误
3. **质量标准**：well-structured, coherent, accurate, reliable
4. **响应格式**：每个响应编号列出

构建聚合prompt的函数：

```python
# 文件: tools/mixture_of_agents_tool.py, 第89-101行
def _construct_aggregator_prompt(system_prompt: str, responses: List[str]) -> str:
    response_text = "\n".join([f"{i+1}. {response}" for i, response in enumerate(responses)])
    return f"{system_prompt}\n\n{response_text}"
```

最终发送给聚合模型的消息结构：
- system: 聚合指令 + 编号的参考响应
- user: 原始用户问题

---

## 4. ACP协议 (Agent Client Protocol)

### 4.1 协议概述

#### 4.1.1 ACP标准的目的

ACP（Agent Client Protocol）是一个标准化的Agent通信协议，类似于LSP（Language Server Protocol）之于IDE。它的目的是：

1. **互操作性**：任何支持ACP的客户端（VS Code、Cursor等）可以连接任何支持ACP的Agent
2. **标准化通信**：定义了Agent能力发现、会话管理、消息传递、工具调用等标准接口
3. **流式更新**：支持实时推送工具进度、思考过程、消息文本等

#### 4.1.2 HermesACPAgent 实现 acp.Agent 接口

```python
# 文件: acp_adapter/server.py, 第93行
class HermesACPAgent(acp.Agent):
    """ACP Agent implementation wrapping Hermes AIAgent."""
```

`acp.Agent`是ACP协议定义的抽象基类，`HermesACPAgent`实现了所有必要的协议方法：

- `initialize()` - 能力协商
- `authenticate()` - 认证
- `new_session()` / `load_session()` / `resume_session()` - 会话管理
- `prompt()` - 核心对话处理
- `cancel()` - 取消运行中的任务
- `fork_session()` - 会话分叉
- `list_sessions()` - 列出会话
- `set_session_model()` - 切换模型
- `set_session_mode()` - 切换模式
- `set_config_option()` - 配置更新

### 4.2 会话管理

#### 4.2.1 SessionManager 持久化

```python
# 文件: acp_adapter/session.py, 第70-77行
class SessionManager:
    """Thread-safe manager for ACP sessions backed by Hermes AIAgent instances.

    Sessions are held in-memory for fast access **and** persisted to the
    shared SessionDB so they survive process restarts and are searchable
    via ``session_search``.
    """
```

双层存储架构：
- **内存层**：`Dict[str, SessionState]`，快速访问
- **持久层**：`SessionDB`（SQLite），存储在`~/.hermes/state.db`

会话恢复逻辑：

```python
# 文件: acp_adapter/session.py, 第114-125行
def get_session(self, session_id: str) -> Optional[SessionState]:
    with self._lock:
        state = self._sessions.get(session_id)
    if state is not None:
        return state
    # Attempt to restore from database.
    return self._restore(session_id)
```

当内存中没有会话时，自动从数据库恢复。这对于进程重启后的会话恢复至关重要。

#### 4.2.2 ThreadPoolExecutor(4) 并行ACP会话

```python
# 文件: acp_adapter/server.py, 第70行
_executor = ThreadPoolExecutor(max_workers=4, thread_name_prefix="acp-agent")
```

#### 4.2.3 为什么是4个worker

1. **典型使用场景**：VS Code用户通常同时打开1-2个编辑器窗口，每个窗口可能有1个活跃的ACP会话。4个worker提供了足够的并发能力。

2. **资源限制**：每个ACP会话运行一个完整的AIAgent实例，消耗内存和API并发。4个worker在资源消耗和并发能力之间取得平衡。

3. **Python GIL**：虽然有GIL限制，但AIAgent的主要瓶颈是I/O（API调用），GIL影响不大。

4. **一致性**：与`max_concurrent_children=3`的设计理念一致——留有余量，避免资源耗尽。

ACP会话的执行方式：

```python
# 文件: acp_adapter/server.py, 第441行
result = await loop.run_in_executor(_executor, _run_agent)
```

每个ACP prompt请求在线程池中执行，不阻塞主事件循环。

### 4.3 功能转发

#### 4.3.1 斜杠命令转发

ACP客户端可以通过斜杠命令与Agent交互：

```python
# 文件: acp_adapter/server.py, 第96-104行
_SLASH_COMMANDS = {
    "help": "Show available commands",
    "model": "Show or change current model",
    "tools": "List available tools",
    "context": "Show conversation context info",
    "reset": "Clear conversation history",
    "compact": "Compress conversation context",
    "version": "Show Hermes version",
}
```

斜杠命令在`_handle_slash_command`中本地处理，不发送到LLM：

```python
# 文件: acp_adapter/server.py, 第375-381行
if user_text.startswith("/"):
    response_text = self._handle_slash_command(user_text, state)
    if response_text is not None:
        if self._conn:
            update = acp.update_agent_message_text(response_text)
            await self._conn.session_update(session_id, update)
        return PromptResponse(stop_reason="end_turn")
```

#### 4.3.2 MCP Server配置转发

ACP客户端可以提供MCP Server配置，Agent会注册这些服务器并刷新工具表面：

```python
# 文件: acp_adapter/server.py, 第150-213行
async def _register_session_mcp_servers(self, state, mcp_servers):
    if not mcp_servers:
        return

    # 1. 注册MCP服务器
    config_map = {}
    for server in mcp_servers:
        name = server.name
        if isinstance(server, McpServerStdio):
            config = {"command": server.command, "args": list(server.args), "env": ...}
        else:
            config = {"url": server.url, "headers": ...}
        config_map[name] = config
    await asyncio.to_thread(register_mcp_servers, config_map)

    # 2. 刷新Agent工具表面
    state.agent.tools = get_tool_definitions(
        enabled_toolsets=enabled_toolsets,
        disabled_toolsets=disabled_toolsets,
        quiet_mode=True,
    )
    state.agent.valid_tool_names = {tool["function"]["name"] for tool in state.agent.tools}

    # 3. 使系统提示词缓存失效
    invalidate = getattr(state.agent, "_invalidate_system_prompt", None)
    if callable(invalidate):
        invalidate()
```

---

## 5. 回调通信机制

### 5.1 七种回调的签名、用途

AIAgent支持以下回调（run_agent.py, 第587-597行）：

| 回调 | 签名 | 用途 | 代码位置 |
|------|------|------|---------|
| `tool_progress_callback` | `(event_type, name, preview, args, **kwargs)` | 工具执行进度通知 | 第754行 |
| `tool_start_callback` | `(tool_name, args_preview)` | 工具开始执行 | 第755行 |
| `tool_complete_callback` | `(tool_name, result)` | 工具执行完成 | 第756行 |
| `thinking_callback` | `(text: str)` | 模型思考过程 | 第758行 |
| `reasoning_callback` | `(text: str)` | 推理内容（extended thinking） | 第759行 |
| `clarify_callback` | `(question, choices) -> str` | 向用户提问 | 第760行 |
| `step_callback` | `(api_call_count, prev_tools)` | 每个API调用步骤完成 | 第761行 |
| `stream_delta_callback` | `(text: str)` | 流式文本增量 | 第762行 |
| `interim_assistant_callback` | `(text: str)` | 临时助手消息 | 第763行 |
| `tool_gen_callback` | `(tool_name, args)` | 工具调用生成 | 第764行 |
| `status_callback` | `(event_type, message)` | 生命周期状态通知 | 第764行 |

### 5.2 回调如何实现CLI/Gateway/ACP三种模式的解耦

回调是实现平台解耦的核心机制。AIAgent本身不关心输出到哪里，它只是在适当的时候调用回调。不同的平台层提供不同的回调实现：

**CLI模式**：
- `tool_progress_callback` → 打印树形进度到终端
- `thinking_callback` → 显示思考气泡
- `clarify_callback` → 使用prompt_toolkit交互式提问

**Gateway模式**：
- `tool_progress_callback` → 批量工具名，通过status_callback发送到聊天平台
- `thinking_callback` → 可选转发到聊天平台
- `clarify_callback` → 通过消息平台交互

**ACP模式**（acp_adapter/events.py）：

```python
# 文件: acp_adapter/events.py, 第47-90行
def make_tool_progress_cb(conn, session_id, loop, tool_call_ids) -> Callable:
    def _tool_progress(event_type, name=None, preview=None, args=None, **kwargs):
        if event_type != "tool.started":
            return
        tc_id = make_tool_call_id()
        queue = tool_call_ids.get(name)
        if queue is None:
            queue = deque()
            tool_call_ids[name] = queue
        queue.append(tc_id)
        update = build_tool_start(tc_id, name, args)
        _send_update(conn, session_id, loop, update)
    return _tool_progress
```

ACP模式将回调转换为ACP协议的`session_update`事件，通过`asyncio.run_coroutine_threadsafe`从工作线程发送到主事件循环。

### 5.3 流式输出的回调链

```
API响应流 → stream_delta_callback → TTS管道/终端输出
           → thinking_callback → 思考气泡
           → tool_progress_callback → 工具进度
           → step_callback → 步骤完成
           → message_callback → 最终消息
```

ACP模式下的完整回调链（acp_adapter/events.py）：

```python
# 文件: acp_adapter/server.py, 第394-405行
if conn:
    tool_progress_cb = make_tool_progress_cb(conn, session_id, loop, tool_call_ids)
    thinking_cb = make_thinking_cb(conn, session_id, loop)
    step_cb = make_step_cb(conn, session_id, loop, tool_call_ids)
    message_cb = make_message_cb(conn, session_id, loop)
    approval_cb = make_approval_callback(conn.request_permission, loop, session_id)
```

每个回调工厂函数接收ACP客户端连接和事件循环引用，返回一个可调用对象。回调内部通过`asyncio.run_coroutine_threadsafe`将事件发送到ACP客户端：

```python
# 文件: acp_adapter/events.py, 第27-40行
def _send_update(conn, session_id, loop, update):
    """Fire-and-forget an ACP session update from a worker thread."""
    try:
        future = asyncio.run_coroutine_threadsafe(
            conn.session_update(session_id, update), loop
        )
        future.result(timeout=5)
    except Exception:
        logger.debug("Failed to send ACP update", exc_info=True)
```

---

## 6. Gateway中的Agent状态管理

### 6.1 _running_agents 字典

```python
# 文件: gateway/run.py, 第599-600行
self._running_agents: Dict[str, Any] = {}  # session_key -> AIAgent
self._running_agents_ts: Dict[str, float] = {}  # start timestamp per session
```

`_running_agents`是Gateway的核心状态字典，键是`session_key`（如`telegram:123456`），值是正在运行的`AIAgent`实例。它用于：

1. **防重入**：检查同一会话是否已有Agent在运行
2. **中断支持**：通过Agent实例调用`interrupt()`
3. **状态查询**：获取当前运行的Agent数量和详情
4. **优雅关闭**：遍历所有运行中的Agent发送中断

### 6.2 _AGENT_PENDING_SENTINEL 防竞态设计

```python
# 文件: gateway/run.py, 第312-316行
# Sentinel placed into _running_agents immediately when a session starts
# processing, *before* any await.  Prevents a second message for the same
# session from bypassing the "already running" guard during the async gap
# between the guard check and actual agent creation.
_AGENT_PENDING_SENTINEL = object()
```

**竞态条件场景**：

1. 消息A到达，检查`_running_agents[session]`为空
2. 消息A开始处理，但还没创建AIAgent（async gap）
3. 消息B到达，检查`_running_agents[session]`仍然为空
4. 消息A和B同时处理，导致两个Agent操作同一会话

**解决方案**：在消息处理开始时立即放入哨兵对象：

```python
# 文件: gateway/run.py, 第3229-3231行
self._running_agents[_quick_key] = _AGENT_PENDING_SENTINEL
self._running_agents_ts[_quick_key] = time.time()
```

后续检查时区分哨兵和真实Agent：

```python
# 文件: gateway/run.py, 第1345-1350行
def _snapshot_running_agents(self) -> Dict[str, Any]:
    return {
        session_key: agent
        for session_key, agent in self._running_agents.items()
        if agent is not _AGENT_PENDING_SENTINEL
    }
```

### 6.3 三条中断路径

Gateway实现了三条独立的中断路径，确保中断的可靠性：

**路径1：适配器层中断（Level 1）**

在消息到达时，适配器（如Telegram）检测到同一会话有新消息，直接中断当前Agent。这是最快的路径，延迟最低。

**路径2：运行Agent中断（Level 2）**

```python
# 文件: gateway/run.py, 第1399-1404行
running_agent = self._running_agents.get(session_key)
if running_agent and running_agent is not _AGENT_PENDING_SENTINEL:
    try:
        running_agent.interrupt(event.text)
    except Exception:
        pass
```

在`_handle_active_session_busy_message`中，通过Agent实例直接调用`interrupt()`。

**路径3：备份中断检查**

```python
# 文件: gateway/run.py, 第9175-9190行
if not _interrupt_detected.is_set() and session_key:
    _backup_adapter = self.adapters.get(source.platform)
    _backup_agent = agent_holder[0]
    if (_backup_adapter and _backup_agent
            and hasattr(_backup_adapter, 'has_pending_interrupt')
            and _backup_adapter.has_pending_interrupt(session_key)):
        _backup_agent.interrupt(_bp_text)
        _interrupt_detected.set()
```

在Agent执行的轮询循环中，定期检查适配器是否有待处理的中断。这是为了防止路径1的中断监控任务静默死亡。

### 6.4 不活跃超时检测

```python
# 文件: gateway/run.py, 第9089-9091行
_agent_timeout_raw = float(os.getenv("HERMES_AGENT_TIMEOUT", 1800))
_agent_timeout = _agent_timeout_raw if _agent_timeout_raw > 0 else None
_agent_warning_raw = float(os.getenv("HERMES_AGENT_TIMEOUT_WARNING", 900))
_agent_warning = _agent_warning_raw if _agent_warning_raw > 0 else None
```

- 默认超时：1800秒（30分钟）
- 默认警告：900秒（15分钟）

超时检测逻辑：

```python
# 文件: gateway/run.py, 第9099-9173行
_inactivity_timeout = False
_POLL_INTERVAL = 5.0

while True:
    done, _ = await asyncio.wait({_executor_task}, timeout=_POLL_INTERVAL)
    if done:
        break

    # Check for inactivity
    if _agent_timeout is not None and agent_holder[0]:
        _idle_secs = time.time() - agent_holder[0]._last_activity_ts
        if _idle_secs >= _agent_timeout:
            _inactivity_timeout = True
            break
        # Warning at 15 minutes
        if not _warning_fired and _agent_warning and _idle_secs >= _agent_warning:
            _warning_fired = True
            # Send warning to user
```

关键点：使用`_last_activity_ts`而不是Agent创建时间。只要Agent有活动（API调用、工具执行、心跳），超时计时器就会重置。这使得长时间运行的任务不会被误杀。

### 6.5 Staleness Eviction

```python
# 文件: gateway/run.py, 第2778-2782行
# Staleness eviction: detect leaked locks from hung/crashed handlers.
# With inactivity-based timeout, active tasks can run for hours, so
# wall-clock age alone isn't sufficient.  Evict only when the agent
# has been *idle* beyond the inactivity threshold (or when the agent
# object has no activity tracker and wall-clock age is extreme).
```

Staleness eviction处理的是"僵尸锁"场景——Agent进程崩溃但锁没有释放。检测逻辑：

```python
_stale_ts = self._running_agents_ts.get(_quick_key, 0)
if _quick_key in self._running_agents and _stale_ts:
    _stale_age = time.time() - _stale_ts
    _stale_agent = self._running_agents.get(_quick_key)
    # Never evict the pending sentinel
    if _stale_agent is _AGENT_PENDING_SENTINEL:
        pass  # Skip
    elif hasattr(_stale_agent, '_last_activity_ts'):
        # Use inactivity-based timeout
        _idle_secs = time.time() - _stale_agent._last_activity_ts
        if _idle_secs >= _stale_timeout:
            # Evict
    elif _stale_age >= _stale_timeout * 2:
        # No activity tracker, use wall-clock age with 2x timeout
        # Evict
```

两种情况会触发驱逐：
1. Agent有活动追踪器，但空闲时间超过阈值
2. Agent没有活动追踪器（异常情况），且wall-clock时间超过2倍阈值

---

## 7. 面试技术亮点

### 7.1 设计决策背后的思考

#### 全局状态污染问题

`model_tools._last_resolved_tool_names`是一个进程级全局变量。构建子Agent时会覆盖它，导致父Agent的工具名列表被污染。

解决方案（delegate_tool.py, 第706-729行）：

```python
# Save parent tool names BEFORE any child construction mutates the global
_parent_tool_names = list(_model_tools._last_resolved_tool_names)

children = []
try:
    for i, t in enumerate(task_list):
        child = _build_child_agent(...)
        child._delegate_saved_tool_names = _parent_tool_names
        children.append((i, t, child))
finally:
    # Authoritative restore
    _model_tools._last_resolved_tool_names = _parent_tool_names
```

这是一个典型的"保存-修改-恢复"模式，确保全局状态在任何情况下都能恢复。

#### 心跳机制的局限性

当前心跳只更新父Agent的活动时间戳，不更新Gateway的活动追踪。如果Gateway有自己的超时检测（基于Agent实例的`_last_activity_ts`），心跳可以正常工作。但如果Gateway使用其他机制（如HTTP连接活跃度），心跳可能不够。

潜在改进：让心跳也更新Gateway层面的活动追踪，或者使用更细粒度的心跳（如每个工具调用都触发）。

#### 线程安全设计

整个系统大量使用锁和线程安全设计：
- `SessionManager._lock`：保护会话字典
- `_active_children_lock`：保护子Agent列表
- `_client_lock`：保护API客户端
- `threading.Event`：用于心跳停止信号

但有一些地方依赖"尽力而为"的线程安全（如`model_tools._last_resolved_tool_names`的保存/恢复），这在极端情况下可能有竞态风险。

### 7.2 遇到的问题和解决方案

#### 问题1：子Agent构建时的全局状态污染

**问题**：`AIAgent.__init__()`调用`get_tool_definitions()`，这会覆盖`model_tools._last_resolved_tool_names`全局变量。

**解决**：在构建前保存，构建后恢复，使用`try/finally`确保恢复。

#### 问题2：Gateway不活跃超时误杀

**问题**：子Agent执行期间，父Agent的活动时间戳冻结，Gateway误判为不活跃。

**解决**：心跳机制，每30秒更新父Agent的活动时间戳，并携带子Agent的详细活动信息。

#### 问题3：凭证泄露到子Agent

**问题**：子Agent可能访问父Agent的API Key，导致安全风险。

**解决**：通过凭证池和租约机制管理凭证。子Agent通过`acquire_lease()`获取凭证，完成后通过`release_lease()`释放。凭证池支持多凭证轮换和冷却机制。

#### 问题4：ACP会话的stdout污染

**问题**：ACP使用stdout传输JSON-RPC帧，Agent的普通输出会污染协议流。

**解决**：将Agent的`_print_fn`替换为stderr输出：

```python
# 文件: acp_adapter/session.py, 第474行
agent._print_fn = _acp_stderr_print
```

### 7.3 可以深入讲解的技术点

1. **凭证池的轮换和冷却机制**：当某个API Key触发rate limit时，自动切换到下一个可用的Key，并将触发限制的Key放入冷却队列。

2. **MoA的推理增强**：所有模型调用都启用了`reasoning: {enabled: true, effort: "xhigh"}`，强制模型进行深度思考。

3. **ACP会话的持久化和恢复**：会话状态存储在SQLite数据库中，进程重启后可以透明恢复。这包括对话历史、模型配置、工作目录等。

4. **工具调用追踪的ID配对**：ACP模式使用`tool_call_id`正确配对并行的工具调用和结果，避免了传统的"最后调用"匹配方式的错误。

5. **子Agent的进度回调批处理**：为了减少Gateway的消息发送频率，子Agent的工具调用进度被批量收集（每5个工具调用），然后一次性发送给父Agent。

6. **迭代预算独立 vs 共享**：子Agent获得独立的迭代预算（`iteration_budget=None`），不受父Agent的迭代消耗影响。这意味着父Agent + 所有子Agent的总迭代次数可以超过单个Agent的`max_iterations`限制。

7. **斜杠命令的本地处理**：ACP模式下的斜杠命令（如`/model`、`/reset`）在本地处理，不发送到LLM。这减少了不必要的API调用，并提供了即时响应。

8. **Agent缓存和配置签名**：Gateway为每个会话缓存AIAgent实例，通过配置签名判断是否需要重建。当配置不变时复用Agent，保持prompt caching的连续性。

---

## 附录：关键代码路径索引

| 功能 | 文件 | 行号 |
|------|------|------|
| MAX_DEPTH定义 | tools/delegate_tool.py | 53 |
| DELEGATE_BLOCKED_TOOLS | tools/delegate_tool.py | 32-38 |
| _build_child_agent | tools/delegate_tool.py | 238-397 |
| _run_single_child | tools/delegate_tool.py | 399-622 |
| delegate_task主函数 | tools/delegate_tool.py | 623-813 |
| 心跳机制 | tools/delegate_tool.py | 433-468 |
| REFERENCE_MODELS | tools/mixture_of_agents_tool.py | 63-68 |
| AGGREGATOR_MODEL | tools/mixture_of_agents_tool.py | 72 |
| _run_reference_model_safe | tools/mixture_of_agents_tool.py | 104-176 |
| _run_aggregator_model | tools/mixture_of_agents_tool.py | 179-230 |
| mixture_of_agents_tool | tools/mixture_of_agents_tool.py | 233-406 |
| HermesACPAgent | acp_adapter/server.py | 93-728 |
| SessionManager | acp_adapter/session.py | 70-476 |
| ACP回调工厂 | acp_adapter/events.py | 47-176 |
| ACP工具映射 | acp_adapter/tools.py | 1-215 |
| AIAgent类定义 | run_agent.py | 535-834 |
| interrupt方法 | run_agent.py | 3040-3087 |
| close方法 | run_agent.py | 3182-3231 |
| get_activity_summary | run_agent.py | 3125-3141 |
| _running_agents管理 | gateway/run.py | 599-600 |
| _AGENT_PENDING_SENTINEL | gateway/run.py | 316 |
| Staleness eviction | gateway/run.py | 2778-2820 |
| 不活跃超时检测 | gateway/run.py | 9089-9200 |
