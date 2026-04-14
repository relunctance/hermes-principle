# Hermes 事件钩子 (Event Hooks) 完整参考

> 基于源码分析：gateway/hooks.py + gateway/run.py

## 钩子列表

| 事件名称 | 触发时机 | context 参数 |
|---------|---------|-------------|
| `gateway:startup` | Gateway 进程启动，所有平台连接完成 | `platforms: List[str]` |
| `session:start` | 新会话创建（首次消息或自动重置后） | `platform`, `user_id`, `session_id`, `session_key` |
| `session:end` | 会话结束（/new、/reset、手动重置） | `platform`, `user_id`, `session_key` |
| `session:reset` | 会话重置完成（新会话条目创建） | `platform`, `user_id`, `session_key` |
| `agent:start` | Agent 开始处理消息 | `platform`, `user_id`, `session_id`, `message` |
| `agent:step` | Agent 每轮工具调用循环完成 | `platform`, `user_id`, `session_id`, `iteration`, `tool_names` |
| `agent:end` | Agent 处理完成（返回响应前） | `platform`, `user_id`, `session_id`, `response` |
| `command:<name>` | 任意 slash 命令执行前 | `platform`, `user_id`, `command`, `args` |

## 触发位置源码

### gateway:startup

```python
# gateway/run.py ~1692
await self.hooks.emit("gateway:startup", {
    "platforms": [p.value for p in self.adapters.keys()],
})
```

**时机：** `GatewayRunner.start()` 末尾，所有平台连接成功后

---

### session:start

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

### session:end

```python
# gateway/run.py ~3997
await self.hooks.emit("session:end", {
    "platform": source.platform.value,
    "user_id": source.user_id,
    "session_key": session_key,
})
```

**时机：** 会话结束（任何原因导致）

---

### session:reset

```python
# gateway/run.py ~4004
await self.hooks.emit("session:reset", {
    "platform": source.platform.value,
    "user_id": source.user_id,
    "session_key": session_key,
})
```

**时机：** `_handle_reset_command()` 执行后，新会话条目创建

---

### agent:start

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

### agent:step

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

### agent:end

```python
# gateway/run.py ~3640
await self.hooks.emit("agent:end", {
    **hook_ctx,
    "response": (response or "")[:500],
})
```

**时机：** `_handle_message_with_agent()` 中，`_run_agent()` 返回后，发送响应前

---

### command:<name>

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

## 内置钩子 (Built-in)

### boot-md

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

## 自定义钩子开发

### 目录结构

```
~/.hermes/hooks/
└── my-hook/
    ├── HOOK.yaml      # 元数据
    └── handler.py     # 处理函数
```

### HOOK.yaml

```yaml
name: my-hook
description: "我的自定义钩子"
events:
  - agent:start
  - agent:end
  - command:new
```

### handler.py

```python
async def handle(event_type: str, context: dict) -> None:
    if event_type == "agent:start":
        print(f"Agent 开始处理: {context['message'][:50]}")

    elif event_type == "agent:end":
        print(f"Agent 完成: {context['response'][:50]}")

    elif event_type.startswith("command:"):
        print(f"命令: {context['command']} {context['args']}")
```

**注意：** 也支持同步函数，`HookRegistry.emit()` 会自动检测 `asyncio.iscoroutine()`

---

## HookRegistry 实现

```python
# gateway/hooks.py
class HookRegistry:
    def __init__(self):
        self._handlers: Dict[str, List[Callable]] = {}
        self._loaded_hooks: List[dict] = []

    def discover_and_load(self) -> None:
        # 扫描 ~/.hermes/hooks/ 目录
        # 注册 HOOK.yaml 中声明的事件
        # 内置 boot-md 钩子始终注册

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
    │               │
    │               ├─► 读取 HOOK.yaml
    │               └─► 加载 handler.py
    │
    └─► await hooks.emit("gateway:startup", {...})
```

---

## 错误处理

```python
# Hook 错误不阻塞主管道
for fn in handlers:
    try:
        result = fn(event_type, context)
        if asyncio.iscoroutine(result):
            await result
    except Exception as e:
        print(f"[hooks] Error in handler for '{event_type}': {e}")
```

---

## 事件流时间线

```
用户消息
    │
    ▼
_gateway:startup_ (首次连接时一次)
    │
    ▼
session:start (新会话时)
    │
    ▼
agent:start
    │
    ├─► tool: step 1 → agent:step
    ├─► tool: step 2 → agent:step
    ├─► tool: step 3 → agent:step
    │
    ▼
agent:end
    │
    ▼
响应发送
    │
    ▼
session:end
session:reset
```

---

## 工具调用事件对比

| 事件 | 粒度 | 说明 |
|------|------|------|
| `agent:step` | 每轮工具循环 | 包含 `iteration`、`tool_names` |
| `tool.started` | 每个工具调用 | 在 `progress_callback` 中，非 hooks 系统 |

---

## command:* 事件示例

触发 `command:new` 的操作：
- `/new`
- `/reset`（别名）

触发 `command:status` 的操作：
- `/status`

触发 `command:approve` 的操作：
- `/approve`
- `/approve session`
- `/approve always`

---

## context 参数完整类型

```python
# gateway:startup
{
    "platforms": List[str]  # 已连接的 platform 名
}

# session:start / session:end / session:reset
{
    "platform": str,
    "user_id": str,
    "session_id": str,
    "session_key": str,
}

# agent:start
{
    "platform": str,
    "user_id": str,
    "session_id": str,
    "message": str,  # 前500字符
}

# agent:step
{
    "platform": str,
    "user_id": str,
    "session_id": str,
    "iteration": int,
    "tool_names": List[str],
    "tools": List[dict],  # [{"name": str, "result": str}, ...]
}

# agent:end
{
    "platform": str,
    "user_id": str,
    "session_id": str,
    "message": str,
    "response": str,  # 前500字符
}

# command:<name>
{
    "platform": str,
    "user_id": str,
    "command": str,
    "args": str,
}
```
