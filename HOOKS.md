# Hermes 事件钩子 (Event Hooks) 完整参考

> 基于源码分析：`gateway/hooks.py`、`gateway/run.py`、`model_tools.py`、`run_agent.py`
> 官方文档：`~/.hermes/hermes-agent/website/docs/user-guide/features/hooks.md`

---

## 一、核心结论：两套 Hook 系统

Hermes 有**两套独立的 Hook 系统**，职责不同：

| 系统 | 注册方式 | 运行范围 | 事件 |
|------|---------|---------|------|
| **Gateway Hooks** | `~/.hermes/hooks/<name>/HOOK.yaml` + `handler.py` | Gateway only（飞书/TG/Discord等） | `gateway:startup`、`session:*`、`agent:*`、`command:*` |
| **Plugin Hooks** | `ctx.register_hook()` in plugin | **CLI + Gateway** | `pre_tool_call`、`post_tool_call`、`pre_llm_call`、`post_llm_call`、`on_session_start`、`on_session_end` |

**关键区别**：
- Gateway Hooks 在消息处理管道中触发（平台消息进 → Agent 处理 → 响应回）
- Plugin Hooks 在工具执行和 LLM 调用管道中触发（更底层，CLI 也能用）
- 两套系统**独立运行**，互不依赖

---

## 二、Gateway Hooks（HOOK.yaml 方式）

### 2.1 目录结构

```
~/.hermes/hooks/
└── my-hook/
    ├── HOOK.yaml      # 声明监听哪些事件
    └── handler.py     # 处理函数
```

### 2.2 事件列表

| 事件名称 | 触发时机 | context 参数 |
|---------|---------|-------------|
| `gateway:startup` | Gateway 进程启动，所有平台连接完成 | `platforms: List[str]` |
| `session:start` | 新会话创建（首次消息或自动重置后） | `platform`, `user_id`, `session_id`, `session_key` |
| `session:end` | 会话结束（/new、/reset、手动重置） | `platform`, `user_id`, `session_key` |
| `session:reset` | 会话重置完成（新会话条目创建） | `platform`, `user_id`, `session_key` |
| `agent:start` | Agent 开始处理消息 | `platform`, `user_id`, `session_id`, `message` |
| `agent:step` | Agent 每轮工具调用循环完成 | `platform`, `user_id`, `session_id`, `iteration`, `tool_names`, `tools` |
| `agent:end` | Agent 处理完成（返回响应前） | `platform`, `user_id`, `session_id`, `message`, `response` |
| `command:<name>` | 任意 slash 命令执行前 | `platform`, `user_id`, `command`, `args` |
| `command:*` | 所有 slash 命令（通配符） | 同上 |

### 2.3 触发位置源码

#### gateway:startup

```python
# gateway/run.py ~1692
await self.hooks.emit("gateway:startup", {
    "platforms": [p.value for p in self.adapters.keys()],
})
```

**时机：** `GatewayRunner.start()` 末尾，所有平台连接成功后

---

#### session:start

```python
# gateway/run.py ~3116
_is_new_session = (
    session_entry.created_at == session_entry.updated_at
    or getattr(session_entry, "was_auto_reset", False)
)
if _is_new_session:
    await self.hooks.emit("session:start", {
        "platform": source.platform.value,
        "user_id": source.user_id,
        "session_id": session_entry.session_id,
        "session_key": session_key,
    })
```

**时机：** `_handle_message_with_agent()` 中，获取/创建会话后，新会话才触发

---

#### agent:start

```python
# gateway/run.py ~3551
hook_ctx = {
    "platform": source.platform.value,
    "user_id": source.user_id,
    "session_id": session_entry.session_id,
    "message": message_text[:500],
}
await self.hooks.emit("agent:start", hook_ctx)
```

**时机：** `_handle_message_with_agent()` 中，调用 `_run_agent()` 之前

---

#### agent:step

```python
# gateway/run.py ~7669-7685
def _step_callback_sync(iteration: int, prev_tools: list) -> None:
    asyncio.run_coroutine_threadsafe(
        _hooks_ref.emit("agent:step", {
            "platform": source.platform.value,
            "user_id": source.user_id,
            "session_id": session_entry.session_id,
            "iteration": iteration,
            "tool_names": _names,
            "tools": prev_tools,
        }),
        _loop_for_step,
    )
```

**时机：** `_run_agent()` 中 `run_sync()` 里，`agent.run_conversation()` 每完成一轮工具调用触发

**prev_tools 格式：** `List[str]` 或 `List[dict]`，dict 包含 `name` 和 `result`

---

#### agent:end

```python
# gateway/run.py ~3640
await self.hooks.emit("agent:end", {
    **hook_ctx,
    "response": (response or "")[:500],
})
```

**时机：** `_handle_message_with_agent()` 中，`_run_agent()` 返回后，发送响应前

---

#### command:<name>

```python
# gateway/run.py ~2658
if command and command in GATEWAY_KNOWN_COMMANDS:
    await self.hooks.emit(f"command:{command}", {
        "platform": source.platform.value,
        "user_id": source.user_id,
        "command": command,
        "args": event.get_command_args().strip(),
    })
```

**时机：** `_handle_message()` 中，识别为 slash 命令后，命令实际处理之前

**通配符支持：** `command:*` 可匹配所有命令

---

### 2.4 内置钩子：boot-md

```python
# gateway/builtin_hooks/boot_md.py
async def handle(event_type: str, context: dict) -> None:
    # 读取 ~/.hermes/BOOT.md
    # 启动后台 AIAgent 执行启动清单
```

**功能：** Gateway 启动时执行 `~/.hermes/BOOT.md` 中的指令

**BOOT.md 示例：**
```markdown
# Startup Checklist
1. 检查昨夜 cron 任务是否有失败
2. 向 Discord #general 发送状态更新
3. 检查 /opt/app/deploy.log 是否有错误
```

---

### 2.5 hawk-bridge 实战案例

**需求：**
- `agent:start` → 从 LanceDB 召回记忆，注入到 Agent 上下文
- `agent:end` → 将本次对话内容存储到 LanceDB

**目录结构：**

```
~/.hermes/hooks/hawk-bridge/
├── HOOK.yaml
└── handler.py
```

**HOOK.yaml：**

```yaml
name: hawk-bridge
description: "OpenClaw Hook Bridge + context-hawk Memory Engine"
events:
  - agent:start   # Agent 开始处理 → 从 LanceDB 召回记忆
  - agent:end     # Agent 完成 → 存储记忆到 LanceDB
```

**handler.py：**

```python
import asyncio
import logging
from pathlib import Path

logger = logging.getLogger("hooks.hawk-bridge")

LANCEDB_PATH = Path.home() / ".hermes" / "hawk-memory" / "memory.db"

async def handle(event_type: str, context: dict) -> None:
    if event_type == "agent:start":
        session_id = context.get("session_id")
        if not session_id:
            return

        try:
            from hawk_memory import RecallEngine
            recall = RecallEngine(LANCEDB_PATH)
            memories = recall.recall(session_id)

            if memories:
                logger.info(f"hawk-bridge: 召回 {len(memories)} 条记忆 for {session_id}")
                _inject_memories(session_id, memories)
            else:
                logger.debug(f"hawk-bridge: 无召回记忆 for {session_id}")

        except Exception as e:
            logger.warning(f"hawk-bridge recall failed: {e}")

    elif event_type == "agent:end":
        session_id = context.get("session_id")
        response = context.get("response", "")
        message = context.get("message", "")

        if not session_id or not response:
            return

        try:
            from hawk_memory import CaptureEngine
            capture = CaptureEngine(LANCEDB_PATH)
            capture.capture(
                session_id=session_id,
                user_message=message,
                agent_response=response,
            )
            logger.info(f"hawk-bridge: 存储记忆 for {session_id}")

        except Exception as e:
            logger.warning(f"hawk-bridge capture failed: {e}")

def _inject_memories(session_id: str, memories: list) -> None:
    # 实现方式取决于如何与 AIAgent 交互
    pass
```

---

## 三、Plugin Hooks（ctx.register_hook 方式）

Plugin Hooks 通过 `ctx.register_hook()` 注册，在**工具执行管道**和 **LLM 调用管道**中触发，**CLI 和 Gateway 都生效**。

### 3.1 注册方式

```python
# 在 plugin 的 register() 函数中
def register(ctx):
    ctx.register_hook("pre_tool_call", my_tool_observer)
    ctx.register_hook("post_tool_call", my_tool_logger)
    ctx.register_hook("pre_llm_call", my_memory_callback)
    ctx.register_hook("post_llm_call", my_sync_callback)
    ctx.register_hook("on_session_start", my_init_callback)
    ctx.register_hook("on_session_end", my_cleanup_callback)
```

### 3.2 完整事件列表

| Hook | 触发时机 | 可否注入上下文 | 源码位置 |
|------|---------|-------------|---------|
| `pre_tool_call` | 每个工具执行前 | ❌ | `model_tools.py` handle_function_call() |
| `post_tool_call` | 每个工具返回后 | ❌ | `model_tools.py` handle_function_call() |
| `pre_llm_call` | 每轮工具循环开始前 | ✅ **唯一可注入** | `run_agent.py` run_conversation() |
| `post_llm_call` | 每轮工具循环结束后 | ❌ | `run_agent.py` run_conversation() |
| `on_session_start` | 新会话创建时（首轮） | ❌ | `run_agent.py` run_conversation() |
| `on_session_end` | 每次 run_conversation 结束时 | ❌ | `run_agent.py` + `cli.py` atexit |

### 3.3 pre_tool_call

**触发位置：** `model_tools.py` → `handle_function_call()` → 工具 handler 执行前

**回调签名：**
```python
def my_callback(tool_name: str, args: dict, task_id: str, **kwargs):
    # tool_name: 工具名（如 "terminal"、"read_file"）
    # args: 模型传入的参数
    # task_id: session/task 标识符
```

**用途：** 审计日志、危险工具警告、限流、工具调用计数

**示例 — 审计日志：**
```python
import json, logging
logger = logging.getLogger(__name__)

def audit_tool_call(tool_name, args, task_id, **kwargs):
    logger.info("TOOL_CALL session=%s tool=%s args=%s",
                task_id, tool_name, json.dumps(args)[:200])

def register(ctx):
    ctx.register_hook("pre_tool_call", audit_tool_call)
```

**示例 — 危险工具警告：**
```python
DANGEROUS = {"terminal", "write_file", "patch"}

def warn_dangerous(tool_name, **kwargs):
    if tool_name in DANGEROUS:
        print(f"⚠ Executing potentially dangerous tool: {tool_name}")

def register(ctx):
    ctx.register_hook("pre_tool_call", warn_dangerous)
```

---

### 3.4 post_tool_call

**触发位置：** `model_tools.py` → `handle_function_call()` → 工具 handler 返回后

**回调签名：**
```python
def my_callback(tool_name: str, args: dict, result: str, task_id: str, **kwargs):
    # tool_name: 工具名
    # args: 模型传入的参数
    # result: 工具返回的 JSON 字符串
    # task_id: session/task 标识符
```

**注意：** 如果工具抛出未处理异常，错误会被捕获并作为 JSON 字符串返回，`post_tool_call` 仍会触发（result 是错误 JSON）。

**用途：** 记录工具结果、追踪成功率、发送特定工具完成通知

**示例 — 工具使用指标：**
```python
from collections import Counter
import json

_tool_counts = Counter()
_error_counts = Counter()

def track_metrics(tool_name, result, **kwargs):
    _tool_counts[tool_name] += 1
    try:
        parsed = json.loads(result)
        if "error" in parsed:
            _error_counts[tool_name] += 1
    except (json.JSONDecodeError, TypeError):
        pass

def register(ctx):
    ctx.register_hook("post_tool_call", track_metrics)
```

---

### 3.5 pre_llm_call

**触发位置：** `run_agent.py` → `run_conversation()` → tool loop 开始前

**这是唯一可以注入上下文的 Hook**——返回值会追加到当前 turn 的 user message 中。

**回调签名：**
```python
def my_callback(session_id: str, user_message: str, conversation_history: list,
                is_first_turn: bool, model: str, platform: str, **kwargs):
    # session_id: 当前会话 ID
    # user_message: 用户原始消息
    # conversation_history: 完整消息列表（OpenAI 格式）
    # is_first_turn: 是否新会话第一轮
    # model: 模型标识符
    # platform: 运行平台（"cli"、"telegram"、"feishu" 等）
```

**返回值：** 返回 dict `{"context": "文本"}` 或纯字符串 → 追加到 user message
返回 `None` → 不注入

**注入位置：** 永远是 **user message**，不是 system prompt（保持 prompt caching）

**示例 — 记忆召回：**
```python
import httpx

MEMORY_API = "https://your-memory-api.example.com"

def recall(session_id, user_message, is_first_turn, **kwargs):
    try:
        resp = httpx.post(f"{MEMORY_API}/recall", json={
            "session_id": session_id,
            "query": user_message,
        }, timeout=3)
        memories = resp.json().get("results", [])
        if not memories:
            return None
        text = "Recalled context:\n" + "\n".join(f"- {m['text']}" for m in memories)
        return {"context": text}
    except Exception:
        return None

def register(ctx):
    ctx.register_hook("pre_llm_call", recall)
```

---

### 3.6 post_llm_call

**触发位置：** `run_agent.py` → `run_conversation()` → tool loop 结束，产生最终响应后

**注意：** 只在**成功**的 turn 触发；用户中断或迭代超限不触发。

**回调签名：**
```python
def my_callback(session_id: str, user_message: str, assistant_response: str,
                conversation_history: list, model: str, platform: str, **kwargs):
```

**用途：** 同步到外部记忆系统、响应质量指标、触发后续动作

---

### 3.7 on_session_start

**触发位置：** `run_agent.py` → `run_conversation()` → 首轮且无历史消息时

**只触发一次**——会话续续（用户发第二条消息）不触发。

**回调签名：**
```python
def my_callback(session_id: str, model: str, platform: str, **kwargs):
```

**用途：** 初始化会话级缓存、注册外部服务、记录会话开始

---

### 3.8 on_session_end

**触发位置：**
1. `run_agent.py` → 每次 `run_conversation()` 结束时
2. `cli.py` → atexit 处理函数（如果 CLI 运行时被中断）

**回调签名：**
```python
def my_callback(session_id: str, completed: bool, interrupted: bool,
                model: str, platform: str, **kwargs):
    # completed: True = 产生了最终响应
    # interrupted: True = 被用户中断（发新消息、/stop、Ctrl+C）
```

**用途：** 刷新缓冲区、关闭连接、持久化会话状态

---

## 四、两套 Hook 系统对比

| 对比维度 | Gateway Hooks | Plugin Hooks |
|--|--|--|
| **注册方式** | `~/.hermes/hooks/<name>/HOOK.yaml` + `handler.py` | `ctx.register_hook()` in plugin |
| **运行范围** | Gateway only | **CLI + Gateway** |
| **触发层级** | 消息处理层（Platform → Agent） | 工具/LLM 执行层 |
| **事件数量** | 8 种 | 6 种 |
| **可注入上下文** | ❌ | ✅ 仅 `pre_llm_call` |
| **典型用途** | 日志记录、webhook、启动清单 | 审计、指标、记忆、限流 |
| **hawk-bridge 挂载点** | ✅ `agent:start` / `agent:end` | ✅ `pre_llm_call` / `post_llm_call` |

---

## 五、错误处理

两套系统统一规则：**Hook 错误不阻塞主管道**

```python
for fn in handlers:
    try:
        result = fn(event_type, context)
        if asyncio.iscoroutine(result):
            await result
    except Exception as e:
        print(f"[hooks] Error in handler for '{event_type}': {e}")
```

---

## 六、HookRegistry 实现

```python
# gateway/hooks.py
class HookRegistry:
    def __init__(self):
        self._handlers: Dict[str, List[Callable]] = {}
        self._loaded_hooks: List[dict] = []

    def discover_and_load(self) -> None:
        # 1. _register_builtin_hooks() — 注册 boot-md
        # 2. 扫描 ~/.hermes/hooks/<hook-dir>/
        #    ├─ 读取 HOOK.yaml
        #    └─ 加载 handler.py

    async def emit(self, event_type: str, context: dict = None) -> None:
        # 精确匹配 + 通配符匹配 (command:*)
        # 异常被捕获并记录，不阻塞主管道
```

### 加载流程

```
GatewayRunner.start()
     │
     ├─► hooks.discover_and_load()
     │       │
     │       ├─► _register_builtin_hooks() — 注册 boot-md
     │       │
     │       └─► 扫描 ~/.hermes/hooks/<hook-dir>/
     │               ├─► 读取 HOOK.yaml
     │               └─► 加载 handler.py
     │
     └─► await hooks.emit("gateway:startup", {...})
```

---

## 七、事件流完整时间线

```
用户消息
     │
     ▼
gateway:startup (首次连接时一次)
     │
     ▼
session:start (新会话时)
     │
     ▼
agent:start ─────────────── ① Gateway Hook: hawk-bridge /recall
     │                           Plugin Hook: pre_llm_call (可注入上下文)
     │
     ├─► tool: step 1 → agent:step
     ├─► tool: step 2 → agent:step
     ├─► tool: step 3 → agent:step
     │    ...
     │     │
     │     ├─► pre_tool_call  (每个工具前)
     │     └─► post_tool_call (每个工具后)
     │
     ▼
agent:end ───────────────── ② Gateway Hook: hawk-bridge /capture
     │                           Plugin Hook: post_llm_call
     ▼
响应发送
     │
     ▼
session:end
session:reset
     │
     ▼
on_session_end ──────────── ③ Plugin Hook: 会话清理
```

① = Gateway Hook `agent:start` + Plugin Hook `pre_llm_call`（可注入上下文）
② = Gateway Hook `agent:end` + Plugin Hook `post_llm_call`
③ = Plugin Hook `on_session_end`

---

## 八、Cron Job 的 Hook 差异

**关键结论：Cron Job 绕过了 Gateway Hooks，不触发 hawk-bridge。**

```
正常消息: Platform → GatewayRunner → HookRegistry → AIAgent  (Gateway Hooks 生效)
Cron Job:  scheduler → AIAgent(platform="cron", skip_memory=True)  (Gateway Hooks 不生效)
```

如果希望 Cron 结果进入 LanceDB，有两个方案：
1. **在 prompt 中主动调用** `hawk-memory-api` 的 `/capture` 接口
2. **改造 scheduler.run_job()**，在末尾追加对 `/capture` 的调用

---

*文档创建时间: 2026-04-18*
*源码路径: gateway/hooks.py, gateway/run.py, model_tools.py, run_agent.py*
*官方文档: ~/.hermes/hermes-agent/website/docs/user-guide/features/hooks.md*
