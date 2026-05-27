# 09 - Agent 间交互与协作深度分析

> 学习顺序：第 9 篇 | 前置知识：核心 Agent 循环（doc 01）、工具系统（doc 02）、ACP 协议（doc 06）

---

## 一、模块概述

### 1.1 Agent 间交互的定义

在 Hermes Agent 中，"Agent 间交互"指多个 AIAgent 实例之间的通信与协作，区别于 Agent 与外部用户（通过消息平台）或 Agent 与外部服务（通过 MCP/API）的交互。

Hermes 提供了三种 Agent 间交互机制：

| 机制 | 通信方式 | 进程模型 | 适用场景 |
|------|---------|---------|---------|
| **Delegate/Subagent** | 同进程直接调用 | 同一进程，多线程 | 任务分解、并行执行 |
| **ACP 子进程生成** | JSON-RPC 2.0 over stdio | 父子进程 | 跨 Agent 实现调用（如调用 Claude Code） |
| **Mixture-of-Agents** | OpenRouter HTTP API | 同进程，异步并发 | 多模型协作推理 |

### 1.2 核心源文件

| 文件 | 行数 | 角色 |
|------|------|------|
| `tools/delegate_tool.py` | ~1104 | 子 Agent 委派核心：构建、启动、监控子 Agent |
| `agent/copilot_acp_client.py` | ~260 | OpenAI 兼容 facade，将请求转发到 Copilot ACP |
| `acp_adapter/server.py` | ~728 | ACP Agent 主体实现，暴露 Hermes 为 ACP Server |
| `acp_adapter/events.py` | ~175 | ACP 事件桥接，Agent 内部事件 → ACP 协议事件 |
| `tools/mixture_of_agents_tool.py` | ~350 | MoA 多模型分层协作 |
| `run_agent.py` | ~535+ | AIAgent 主类，委托/中断/回调的承载者 |
| `tools/managed_tool_gateway.py` | ~200 | 托管工具网关，通过 Nous 基础设施路由工具调用 |
| `tools/registry.py` | — | 工具注册表，delegate_task 和 mixture_of_agents 在此注册 |

---

## 二、Delegate/Subagent 架构（同进程多 Agent）

### 2.1 架构总览

Delegate 是 Hermes 最核心的 Agent 间交互机制。父 Agent 通过 `delegate_task` 工具调用，在同进程内创建独立的子 AIAgent 实例来分担工作。

```
┌─────────────────────────────────────────────────────────┐
│                    Parent AIAgent                        │
│  ┌──────────────────────────────────────────────────┐   │
│  │            delegate_task() 工具调用               │   │
│  │  goal="分析 bug", toolsets=["terminal","file"]    │   │
│  └──────────────────┬───────────────────────────────┘   │
│                     │                                    │
│         ┌───────────┼───────────┐                        │
│         ▼           ▼           ▼                        │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│   │ Child 0  │ │ Child 1  │ │ Child 2  │  ThreadPool   │
│   │ AIAgent  │ │ AIAgent  │ │ AIAgent  │               │
│   │ 独立上下文│ │ 独立上下文│ │ 独立上下文│               │
│   │ 独立预算  │ │ 独立预算  │ │ 独立预算  │               │
│   └────┬─────┘ └────┬─────┘ └────┬─────┘               │
│        │            │            │                       │
│        ▼            ▼            ▼                       │
│    summary       summary      summary                    │
│        │            │            │                       │
│        └────────────┼────────────┘                       │
│                     ▼                                    │
│         JSON results (merged, returned to parent)        │
└─────────────────────────────────────────────────────────┘
```

### 2.2 子 Agent 构建流程

`_build_child_agent()` (delegate_tool.py L238-397) 是子 Agent 的工厂函数，运行在主线程上以确保线程安全：

**第一步：解析工具集**

```
parent_enabled = parent_agent.enabled_toolsets
     │
     ├─ 用户指定 toolsets → 与父 Agent 工具集取交集
     ├─ 未指定 → 继承父 Agent 的 enabled_toolsets
     └─ 一律经过 _strip_blocked_tools() 移除禁止工具
```

子 Agent 工具集 = 父 Agent 工具集 ∩ 请求工具集 - DELEGATE_BLOCKED_TOOLS

**第二步：构建隔离的系统提示词**

`_build_child_system_prompt()` (delegate_tool.py L90-122) 生成简洁的专注提示词：

```
You are a focused subagent working on a specific delegated task.

YOUR TASK:
{goal}

CONTEXT:                          ← 可选，传递文件路径、错误信息等
{context}

WORKSPACE PATH:                   ← 可选，从父 Agent 环境检测
{workspace_path}

Complete this task using the tools available to you.
When finished, provide a clear, concise summary of:
- What you did
- What you found or accomplished
- Any files you created or modified
- Any issues encountered
```

**第三步：实例化 AIAgent**

```python
child = AIAgent(
    base_url=effective_base_url,
    api_key=effective_api_key,
    model=effective_model,
    provider=effective_provider,
    acp_command=effective_acp_command,      # ACP 传输覆盖
    acp_args=effective_acp_args,
    max_iterations=max_iterations,           # 默认 50
    enabled_toolsets=child_toolsets,
    quiet_mode=True,                         # 静默模式
    ephemeral_system_prompt=child_prompt,    # 临时提示词，不影响父 Agent
    skip_context_files=True,                 # 不加载项目上下文文件
    skip_memory=True,                        # 不加载记忆
    clarify_callback=None,                   # 不能与用户交互
    parent_session_id=parent.session_id,     # 链路追踪
    iteration_budget=None,                   # 独立预算
)
```

### 2.3 安全隔离机制

**五层安全防护：**

```
Layer 1: 工具阻止列表 (DELEGATE_BLOCKED_TOOLS)
         ├─ delegate_task   ← 禁止递归委派
         ├─ clarify         ← 禁止用户交互
         ├─ memory          ← 禁止写入共享 MEMORY.md
         ├─ send_message    ← 禁止跨平台副作用
         └─ execute_code    ← 子 Agent 应逐步推理而非写脚本

Layer 2: 深度限制 (MAX_DEPTH = 2)
         parent(0) → child(1) → grandchild REJECTED

Layer 3: 上下文隔离
         ├─ ephemeral_system_prompt 不入父 Agent 上下文
         ├─ 独立消息历史（全新 conversation）
         ├─ 独立终端会话（独立 working directory）
         └─ skip_memory=True（不加载共享记忆）

Layer 4: 预算隔离
         ├─ iteration_budget=None（独立 IterationBudget，默认 50 次）
         └─ 子 Agent 迭代不消耗父 Agent 预算

Layer 5: 工具集交集限制
         子 Agent 不能获得父 Agent 没有的工具（取交集）
```

### 2.4 单任务 vs 批量并行

`delegate_task()` (delegate_tool.py L623-813) 支持两种执行模式：

**单任务模式** (n_tasks == 1)：
```python
# 直接在当前线程执行，无线程池开销
result = _run_single_child(0, task["goal"], child, parent_agent)
```

**批量并行模式** (n_tasks > 1)：
```python
# 使用 ThreadPoolExecutor，默认最多 3 个并发子 Agent
with ThreadPoolExecutor(max_workers=max_children) as executor:
    futures = {}
    for i, t, child in children:
        future = executor.submit(_run_single_child, ...)
        futures[future] = i

    for future in as_completed(futures):
        entry = future.result()
        results.append(entry)
```

并发数由 `delegation.max_concurrent_children` 配置控制（默认 3），可通过 `config.yaml` 或 `DELEGATION_MAX_CONCURRENT_CHILDREN` 环境变量调整。

### 2.5 进度回传机制

`_build_child_progress_callback()` (delegate_tool.py L158-235) 构建回传回调，将子 Agent 的工具调用状态实时反馈给父 Agent：

**双路径显示：**

```
CLI 路径 (spinner.print_above):
  ├─ 💭 "thinking text..."              ← thinking 事件
  ├─ 🔧 read  "file.py"                ← tool.started 事件
  └─ ✓ [1/3] Analyze bug pattern  (12s) ← 完成通知

Gateway 路径 (parent_callback):
  └─ 批量收集工具名（每 5 个一批），通过 parent_cb("subagent_progress", ...) 发送
```

### 2.6 心跳机制

子 Agent 执行期间，父 Agent 的 `_last_activity_ts` 不再更新，网关可能因"无活动"而超时杀死父 Agent。心跳机制解决此问题：

```python
# delegate_tool.py L437-469
_heartbeat_stop = threading.Event()

def _heartbeat_loop():
    while not _heartbeat_stop.wait(_HEARTBEAT_INTERVAL):  # 每 30 秒
        touch = getattr(parent_agent, '_touch_activity', None)
        desc = f"delegate_task: subagent running {child_tool} (iteration {n}/{max})"
        touch(desc)  # 更新父 Agent 的活动时间戳

_heartbeat_thread = threading.Thread(target=_heartbeat_loop, daemon=True)
_heartbeat_thread.start()
```

心跳消息包含子 Agent 的当前工具名和迭代进度，便于日志追踪。

### 2.7 中断传播

父 Agent 被中断时，中断信号会传播到所有活跃的子 Agent：

```python
# delegate_tool.py L388-395
# 注册子 Agent 到父 Agent 的活跃列表
if hasattr(parent_agent, '_active_children'):
    with lock:
        parent_agent._active_children.append(child)

# 父 Agent 的中断处理会遍历 _active_children 并调用 child.set_interrupt()
```

子 Agent 完成后从活跃列表移除（delegate_tool.py L602-611）。

### 2.8 结果汇总

每个子 Agent 返回结构化的 JSON 结果：

```json
{
  "task_index": 0,
  "status": "completed|interrupted|failed",
  "summary": "子 Agent 的最终回复内容",
  "api_calls": 12,
  "duration_seconds": 34.5,
  "error": null,
  "tool_trace": [
    {"tool": "bash", "args_preview": "git log --oneline", "result_preview": "abc1234 ..."},
    {"tool": "read", "args_preview": "src/main.py", "result_preview": "..."}
  ]
}
```

结果通过 `json.dumps()` 序列化后返回给父 Agent，作为单条工具返回消息出现在父 Agent 的上下文中。**子 Agent 的中间工具调用和推理过程永不出现在父 Agent 上下文中**——这是 delegate 最关键的上下文隔离保证。

---

## 三、ACP 子进程生成（跨进程 Agent 调用）

### 3.1 设计动机

同进程的 delegate 受限于同一 Python 运行时。当父 Agent（运行在 Telegram/Discord/CLI 中）需要调用一个完全不同的 Agent 实现（如 Claude Code、GitHub Copilot）时，需要通过 ACP 协议启动独立子进程。

```
┌──────────────────────────────────────────────────┐
│            Hermes Parent Process                  │
│  ┌────────────────────────────────────────────┐  │
│  │  AIAgent (provider=openrouter)             │  │
│  │  delegate_task(                            │  │
│  │    goal="review PR #42",                   │  │
│  │    acp_command="claude",                   │  │
│  │    acp_args=["--acp", "--stdio"]            │  │
│  │  )                                          │  │
│  └──────────────────┬─────────────────────────┘  │
└────────────────────┼────────────────────────────┘
                     │ subprocess.Popen
                     ▼
         ┌───────────────────────────┐
         │   claude --acp --stdio    │  ← 独立进程
         │   (Claude Code ACP)       │
         │                           │
         │   JSON-RPC 2.0 over stdio │
         │   ├─ initialize           │
         │   ├─ session/new          │
         │   ├─ session/prompt       │
         │   └─ session/update 事件流│
         └───────────────────────────┘
```

### 3.2 ACP 传输覆盖

在 `_build_child_agent()` (delegate_tool.py L326-327) 中：

```python
effective_acp_command = override_acp_command or getattr(parent_agent, "acp_command", None)
effective_acp_args = list(override_acp_args or getattr(parent_agent, "acp_args", []) or [])
```

当 `acp_command` 被设置时，子 Agent 使用 ACP 子进程传输（而非继承父 Agent 的 provider/base_url/api_key），由 `AIAgent.__init__()` 中的逻辑处理：

- `acp_command` 非空 → 启动 ACP 子进程，所有 LLM 调用通过 ACP 协议转发
- `acp_command` 为空 → 使用标准 HTTP API 调用（与父 Agent 相同）

### 3.3 delegate_task 的 ACP 参数

```json
{
  "acp_command": "claude",
  "acp_args": ["--acp", "--stdio"],
  "tasks": [
    {
      "goal": "Review the authentication middleware",
      "context": "Files: src/auth/*.py",
      "acp_command": "claude",
      "acp_args": ["--acp", "--stdio", "--model", "claude-opus-4-6"]
    }
  ]
}
```

支持顶层的 `acp_command`/`acp_args`（应用于所有 tasks）和每个 task 独立的覆盖。

### 3.4 Copilot ACP Client

`agent/copilot_acp_client.py` 提供了 OpenAI 兼容的 facade，将 Hermes 的 LLM 请求转发到 `copilot --acp`：

```
AIAgent (OpenAI client 接口)
     │
     ▼
CopilotACPClient (OpenAI 兼容 facade)
     │  _format_messages_as_prompt()
     │  JSON-RPC 2.0: initialize → session/new → session/prompt
     ▼
copilot --acp --stdio (子进程)
```

**消息格式化** (_format_messages_as_prompt, L57-129)：
- 将 OpenAI 格式的 messages 列表转换为纯文本 prompt
- 注入工具定义（以 JSON 形式嵌入 prompt）
- 使用 `<tool_call>{...}</tool_call>` XML 块标记工具调用

**工具调用提取** (_extract_tool_calls_from_text, L156-227)：
- 正则匹配 `<tool_call>{...}</tool_call>` XML 块
- Fallback：匹配裸 JSON 对象（含 id/type/function 字段）
- 清洗：移除工具调用块，保留纯文本回复

**路径安全** (_ensure_path_within_cwd, L230-240)：
- 所有文件系统路径必须为绝对路径
- 限制在会话 CWD 内，防止路径遍历攻击

### 3.5 ACP 事件桥接

`acp_adapter/events.py` 将 AIAgent 内部事件转换为 ACP 协议事件：

| AIAgent 事件 | ACP 事件 |
|-------------|---------|
| `thinking_callback(text)` | `agent_thought_chunk` |
| `step_callback(step_data)` | `agent_message_chunk` |
| `tool_progress_callback("tool.started", ...)` | `tool_call_start` |
| `tool_progress_callback("tool.completed", ...)` | `tool_call_complete` |

这些事件通过 `conn.session_update()` 发送到连接的 ACP 客户端，实现实时状态同步。

---

## 四、Mixture-of-Agents (MoA) — 多 LLM 协作

### 4.1 架构原理

MoA 实现了论文 "Mixture-of-Agents Enhances Large Language Model Capabilities" (arXiv:2406.04692) 中的分层协作架构：

```
┌──────────────────────────────────────────────────┐
│                 MoA 处理流程                      │
│                                                   │
│  Layer 1: Reference Models (并行)                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────┐│
│  │ Claude   │ │ Gemini   │ │ GPT-5.4  │ │DeepSeek│
│  │ Opus 4.6 │ │ Pro 3    │ │ Pro      │ │ V3.2  ││
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └───┬──┘│
│       │            │            │           │     │
│       └────────────┼────────────┼───────────┘     │
│                    │            │                  │
│                    ▼            ▼                  │
│  Layer 2: Aggregator                              │
│  ┌──────────────────────────────────────────────┐ │
│  │            Claude Opus 4.6                   │ │
│  │  合成所有 reference 响应，生成最终答案        │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

### 4.2 并行参考模型调用

```python
# mixture_of_agents_tool.py L311-313
async def _get_reference_responses(prompt, model_list, temperature):
    tasks = [_call_model(model, prompt, temperature) for model in model_list]
    responses = await asyncio.gather(*tasks, return_exceptions=True)
    return responses
```

使用 `asyncio.gather()` 并行调用 4 个参考模型（通过 OpenRouter API），任一模失败不影响其他模型。

### 4.3 聚合器合成

```python
# mixture_of_agents_tool.py L343-353
aggregator_prompt = f"""
You are an expert aggregator. Synthesize the following responses
from multiple AI models into a single high-quality answer.

USER QUERY: {user_prompt}

RESPONSES FROM REFERENCE MODELS:
{formatted_responses}

Provide a comprehensive, well-structured answer that captures
the best insights from all responses.
"""
final_response = await _call_model(AGGREGATOR_MODEL, aggregator_prompt, AGGREGATOR_TEMPERATURE)
```

聚合器接收所有参考模型的响应，合成统一的最终答案。

### 4.4 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `REFERENCE_MODELS` | 4 个模型 | 参考模型列表 |
| `AGGREGATOR_MODEL` | claude-opus-4.6 | 聚合模型 |
| `REFERENCE_TEMPERATURE` | 0.7 | 参考模型采样温度 |
| `AGGREGATOR_TEMPERATURE` | 0.5 | 聚合器采样温度 |
| `MIN_SUCCESSFUL_REFERENCES` | 2 | 最少成功参考模型数 |

---

## 五、Agent 生命周期管理

### 5.1 子 Agent 完整生命周期

```
                    ┌──────────────────┐
                    │  delegate_task() │
                    │  被父 Agent 调用  │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │ _build_child()   │  ← 主线程构建
                    │ 创建 AIAgent     │
                    │ 注册到活跃列表   │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ 单任务   │  │ 批量并行 │  │ 批量并行 │
        │ (主线程) │  │ (线程1)  │  │ (线程2)  │
        └────┬─────┘  └────┬─────┘  └────┬─────┘
             │              │              │
             │    ┌─────────┼─────────┐    │
             │    ▼         ▼         ▼    │
             │  heartbeat  heartbeat  heartbeat
             │  (daemon 线程，30s 间隔)    │
             │    │         │         │    │
             │    ▼         ▼         ▼    │
             │  child.run_conversation()   │
             │    │         │         │    │
             │    ▼         ▼         ▼    │
             │  收集结果  收集结果  收集结果 │
             │    │         │         │    │
             └────┼─────────┼─────────┼────┘
                  │         │         │
                  └─────────┼─────────┘
                            │
                   ┌────────▼─────────┐
                   │ 合并结果 (JSON)  │
                   │ 通知 memory      │
                   └────────┬─────────┘
                            │
                   ┌────────▼─────────┐
                   │ 清理子 Agent     │
                   │ 从活跃列表移除   │
                   │ child.close()    │
                   └──────────────────┘
```

### 5.2 资源清理

每个子 Agent 完成后执行清理 (delegate_tool.py L600-621)：

```python
# 1. 从父 Agent 活跃列表移除
parent_agent._active_children.remove(child)

# 2. 关闭子 Agent 资源
child.close()  # 释放终端沙箱、浏览器 daemon、后台进程、httpx 客户端
```

### 5.3 全局状态保护

子 Agent 构建过程中，`AIAgent.__init__()` 会调用 `get_tool_definitions()` 修改全局的 `_last_resolved_tool_names`。delegate 通过 save/restore 模式保护此全局状态：

```python
# delegate_tool.py L702-728
_parent_tool_names = list(_model_tools._last_resolved_tool_names)

children = []
try:
    for i, t in enumerate(task_list):
        child = _build_child_agent(...)
        child._delegate_saved_tool_names = _parent_tool_names
        children.append((i, t, child))
finally:
    _model_tools._last_resolved_tool_names = _parent_tool_names  # 权威恢复
```

---

## 六、凭证与资源配置

### 6.1 凭证继承策略

子 Agent 的凭证解析优先级：

```
1. delegation.provider 配置（config.yaml）
   └─ _resolve_delegation_credentials() → runtime_provider 系统
      获取完整的 base_url + api_key + api_mode

2. delegation.base_url 配置（config.yaml）
   └─ 直接使用指定的 base_url + api_key

3. 无配置 → 继承父 Agent 全部凭证
   └─ provider, base_url, api_key, api_mode 全部继承
```

### 6.2 凭证池共享

`_resolve_child_credential_pool()` (delegate_tool.py L816-845) 处理凭证池的共享与隔离：

```
子 Agent provider == 父 Agent provider
  └─ 共享父 Agent 的凭证池（冷却状态和轮换保持同步）

子 Agent provider != 父 Agent provider
  └─ 加载子 Agent 独立的凭证池

无可用池
  └─ 回退到固定凭证（继承模式）
```

### 6.3 凭证租约

```python
# delegate_tool.py L422-431
child_pool = getattr(child, '_credential_pool', None)
if child_pool is not None:
    leased_cred_id = child_pool.acquire_lease()
    if leased_cred_id is not None:
        leased_entry = child_pool.current()
        child._swap_credential(leased_entry)
```

凭证池租约机制确保并行子 Agent 使用不同的凭证，避免 API 限流冲突。

### 6.4 推理配置继承

```python
# delegate_tool.py L330-346
parent_reasoning = getattr(parent_agent, "reasoning_config", None)
child_reasoning = parent_reasoning

delegation_cfg = _load_config()
delegation_effort = str(delegation_cfg.get("reasoning_effort") or "").strip()
if delegation_effort:
    parsed = parse_reasoning_effort(delegation_effort)
    if parsed is not None:
        child_reasoning = parsed
```

`delegation.reasoning_effort` 配置可覆盖子 Agent 的推理预算（如 thinking tokens），未配置时继承父 Agent 级别。

---

## 七、跨平台消息传递（Agent ↔ Agent 通过消息平台）

### 7.1 send_message 工具

虽然 `send_message` 在子 Agent 中被阻止，但在父 Agent 场景中，它提供了 Agent 之间的间接通信路径：

```
Agent A (Discord session)                    Agent B (Telegram session)
        │                                            │
        │  send_message(                             │
        │    platform="telegram",                    │
        │    chat_id="12345",                        │
        │    text="数据已处理完毕"                    │
        │  )                                         │
        │                                            │
        ▼                                            ▼
  Gateway DeliveryRouter ────► Telegram Adapter ──► Agent B 接收
```

`gateway/delivery.py` 的 `DeliveryRouter` 支持三种路由目标：
- `origin`：回发到原始会话
- `local`：仅写入本地会话记录
- `platform:chat_id`：定向投递到指定平台和会话

### 7.2 消息镜像

`mirror_to_session()` 在目标会话的 JSONL 转录和 SQLite 数据库中写入投递记录，确保跨平台上下文的一致性。

---

## 八、三种交互机制对比

| 维度 | Delegate/Subagent | ACP 子进程 | Mixture-of-Agents |
|------|-------------------|-----------|-------------------|
| **通信协议** | Python 直接调用 | JSON-RPC 2.0 over stdio | OpenRouter HTTP API |
| **进程模型** | 同进程多线程 | 父子进程 | 同进程异步 |
| **Agent 实现** | 相同 (AIAgent) | 任意 (Claude Code, Copilot 等) | 外部 LLM 模型 |
| **上下文隔离** | 完全隔离 | 完全隔离 | 不适用 |
| **并行度** | ThreadPoolExecutor (默认 3) | 受子进程资源限制 | asyncio.gather (4 路) |
| **工具限制** | DELEGATE_BLOCKED_TOOLS | 由子进程 Agent 决定 | 无工具 |
| **凭证** | 共享池或独立 | 继承 ACP command | 统一 OpenRouter API key |
| **结果格式** | JSON (task_index, status, summary) | 文本流 | 文本 |
| **中断支持** | 传播到子 Agent | 杀死子进程 | asyncio 取消 |
| **适用场景** | 任务分解与并行 | 跨实现 Agent 调用 | 复杂推理增强 |

---

## 九、Agent 交互的完整数据流

```
                           ┌──────────────────┐
                           │   外部 ACP 客户端 │
                           │ (VS Code, etc.)   │
                           └────────┬─────────┘
                                    │ JSON-RPC 2.0 (stdio)
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                       Hermes Gateway                           │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                   GatewayRunner                           │ │
│  │  ┌─────────┐  ┌──────────┐  ┌─────────────┐             │ │
│  │  │Telegram  │  │ Discord  │  │   Slack     │  ...20+     │ │
│  │  │Adapter   │  │ Adapter  │  │   Adapter   │             │ │
│  │  └────┬─────┘  └────┬─────┘  └──────┬──────┘             │ │
│  │       │              │               │                    │ │
│  │       └──────────────┼───────────────┘                    │ │
│  │                      │                                    │ │
│  │          ┌───────────▼───────────┐                        │ │
│  │          │   Session Store       │                        │ │
│  │          │   (SQLite + JSONL)    │                        │ │
│  │          └───────────┬───────────┘                        │ │
│  └──────────────────────┼────────────────────────────────────┘ │
│                         │                                      │
│  ┌──────────────────────┼────────────────────────────────────┐ │
│  │                      ▼                                     │ │
│  │              AIAgent (Parent)                               │ │
│  │  ┌─────────────────────────────────────────────────────┐   │ │
│  │  │ Tool Calling Loop                                   │   │ │
│  │  │  ├─ delegate_task ─────────┐                        │   │ │
│  │  │  ├─ mixture_of_agents ─────┼──── 参考模型 API 调用  │   │ │
│  │  │  ├─ send_message ──────────┼────► 投递到其他平台    │   │ │
│  │  │  └─ 其他工具              │                        │   │ │
│  │  └────────────────────────────┼────────────────────────┘   │ │
│  └───────────────────────────────┼────────────────────────────┘ │
│                                  │                              │
│  ┌───────────────────────────────┼────────────────────────────┐ │
│  │                               ▼                             │ │
│  │  子 Agent 层 (ThreadPoolExecutor, 默认 max 3)              │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │ │
│  │  │ Child AIAgent│  │ Child AIAgent│  │ ACP 子进程   │     │ │
│  │  │ (同进程)     │  │ (同进程)     │  │ (claude --acp)│     │ │
│  │  │              │  │              │  │              │     │ │
│  │  │ 工具循环     │  │ 工具循环     │  │ JSON-RPC 2.0 │     │ │
│  │  │ 独立上下文   │  │ 独立上下文   │  │ 独立进程     │     │ │
│  │  │ 独立预算     │  │ 独立预算     │  │              │     │ │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │ │
│  │         │                 │                 │              │ │
│  │         ▼                 ▼                 ▼              │ │
│  │      summary           summary           text stream       │ │
│  │         │                 │                 │              │ │
│  │         └─────────────────┼─────────────────┘              │ │
│  │                           ▼                                │ │
│  │              JSON results (返回给父 Agent)                  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              外部 LLM API 调用                            │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │   │
│  │  │OpenRouter│  │Anthropic │  │  Nous    │  ...           │   │
│  │  │          │  │          │  │  Portal  │               │   │
│  │  └──────────┘  └──────────┘  └──────────┘               │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 十、设计决策与权衡

### 10.1 为什么子 Agent 是同进程而非独立进程？

**决策**：默认 delegate 在同进程内创建 AIAgent 实例。

**原因**：
- 零序列化开销：子 Agent 的工具调用和结果无需跨进程序列化
- 共享 LLM 客户端连接池：避免每个子 Agent 建立独立 HTTP 连接
- 凭证池共享：同 provider 的子 Agent 共享冷却状态
- 简化生命周期管理：无需处理僵尸进程

**代价**：GIL 限制（Python 线程），但对于 I/O 密集型的 LLM 调用影响不大。

### 10.2 为什么 ACP 子进程是可选的？

**决策**：ACP 子进程作为可选覆盖，而非替代同进程模式。

**原因**：
- 同进程模式更快、更轻量，适合大多数场景
- ACP 子进程仅在需要不同 Agent 实现时使用（如调用 Claude Code 的特定功能）
- 保持向后兼容：无 ACP 配置时行为不变

### 10.3 为什么子 Agent 不能递归委派？

**决策**：`MAX_DEPTH = 2`，`DELEGATE_BLOCKED_TOOLS` 包含 `delegate_task`。

**原因**：
- 防止委派爆炸：无限制递归会导致指数级 Agent 数量
- 确定性保证：2 层委派足够覆盖绝大多数任务分解场景
- 调试简化：3 层以上的调用链极难追踪和调试

### 10.4 为什么上下文快照复制而非流式传递？

**决策**：子 Agent 获取快照式上下文（goal + context 字符串），而非父 Agent 的实时消息流。

**原因**：
- 上下文隔离是核心设计目标——子 Agent 的中间推理不应污染父 Agent
- 流式传递需要复杂的背压和同步机制
- 快照模式足够满足"传递任务约束和背景信息"的需求

---

## 十一、扩展点与未来方向

### 11.1 自定义 Agent 实现

ACP 协议使得任何实现 ACP 的 Agent 都可以作为 Hermes 的子 Agent：

```yaml
# config.yaml
delegation:
  acp_command: "my-custom-agent"
  acp_args: ["--acp", "--stdio"]
```

### 11.2 ContextEngine 可插拔压缩

`plugins/context_engine/` 支持自定义上下文压缩策略，影响子 Agent 返回结果如何被压缩后合并入父 Agent 上下文。

### 11.3 Multi-Agent Memory 观察者

`honcho-integration-spec.md` 描述了通过 `subagent_spawned` 事件追踪多 Agent 层级的观察者模式，支持外部记忆系统感知 Agent 层级关系。

---

## 十二、与现有文档的关系

| 相关文档 | 关系 |
|---------|------|
| doc 01 (核心循环) | delegate_task 在 `_invoke_tool()` 中特殊处理（不走 registry） |
| doc 02 (工具系统) | delegate_task 和 mixture_of_agents 的工具注册和 schema |
| doc 03 (Gateway) | Agent 会话生命周期、pending message 队列、中断 |
| doc 06 (MCP/ACP) | ACP 协议细节、事件桥接、会话管理 |
| doc 07 (子系统) | 子 Agent 继承的 prompt_builder、context_compressor 等 |
| doc 08 (基础设施) | credential_pool、终端沙箱等共享基础设施 |

---

## 十三、关键代码路径索引

| 功能 | 文件 | 关键函数/类 | 行号 |
|------|------|-----------|------|
| 子 Agent 构建 | `tools/delegate_tool.py` | `_build_child_agent()` | L238-397 |
| 子 Agent 执行 | `tools/delegate_tool.py` | `_run_single_child()` | L399-621 |
| 委派入口 | `tools/delegate_tool.py` | `delegate_task()` | L623-813 |
| 进度回调 | `tools/delegate_tool.py` | `_build_child_progress_callback()` | L158-235 |
| 心跳机制 | `tools/delegate_tool.py` | `_heartbeat_loop()` | L439-466 |
| 凭证池共享 | `tools/delegate_tool.py` | `_resolve_child_credential_pool()` | L816-845 |
| 凭证解析 | `tools/delegate_tool.py` | `_resolve_delegation_credentials()` | L848-935 |
| 深度限制 | `tools/delegate_tool.py` | `MAX_DEPTH`, `DELEGATE_BLOCKED_TOOLS` | L32-53 |
| 工具注册 | `tools/delegate_tool.py` | `registry.register("delegate_task", ...)` | L1088-1103 |
| ACP 子进程客户端 | `agent/copilot_acp_client.py` | `CopilotACPClient` | L256-260 |
| ACP 事件桥接 | `acp_adapter/events.py` | `make_tool_progress_cb()` 等 | L1-175 |
| ACP Server | `acp_adapter/server.py` | `HermesACPAgent` | L93 |
| MoA 工具 | `tools/mixture_of_agents_tool.py` | `mixture_of_agents_tool()` | L300+ |
| Agent 中断 | `run_agent.py` | `set_interrupt()`, `_active_children` | — |
| 消息传递 | `tools/send_message_tool.py` | `send_message()` | L148+ |
| 消息投递 | `gateway/delivery.py` | `DeliveryRouter` | L107 |
| 会话上下文 | `gateway/session_context.py` | `ContextVar` 隔离 | — |
