# Hermes model_tools 工具编排层原理

> 基于 `~/.hermes/hermes-agent/model_tools.py`（577行）源码分析。

---

## 一、核心定位

model_tools 是 Hermes 的**工具编排层**，位于 AIAgent 和 Tool Registry 之间：

```
AIAgent.run_conversation()
       ↓
model_tools.get_tool_definitions()  → 返回 tool schemas 给 LLM
       ↓
LLM 返回 tool_calls
       ↓
model_tools.handle_function_call()  → 分发到具体工具
       ↓
Tool Registry → 工具实现
```

---

## 二、核心 API

| 函数 | 作用 |
|------|------|
| `get_tool_definitions(enabled_toolsets, disabled_toolsets)` | 返回 LLM 可调用的 tool schemas，按 toolset 过滤 |
| `handle_function_call(function_name, function_args, ...)` | 分发工具调用，执行具体工具 |
| `coerce_tool_args(tool_name, args)` | **类型强制转换**：LLM 的 `"42"` → int `42` |

---

## 三、Tool Discovery 机制

### 3.1 导入即注册

```python
def _discover_tools():
    _modules = [
        "tools.web_tools",
        "tools.terminal_tool",
        "tools.file_tools",
        "tools.vision_tools",
        ...
    ]
    for mod_name in _modules:
        importlib.import_module(mod_name)  # 触发 registry.register()

_discover_tools()  # 模块加载时自动执行
```

每个工具文件在 import 时调用 `registry.register()`，将 schema、handler、check_fn 注册进去。

### 3.2 三层工具来源

| 来源 | 方式 |
|------|------|
| 内置工具（22个） | 直接 import tools/ 模块 |
| MCP 工具 | `discover_mcp_tools()` 从外部 MCP 服务器发现 |
| Plugin 工具 | `discover_plugins()` 从用户/项目插件发现 |

---

## 四、Toolset 过滤机制

### 4.1 过滤逻辑

```
enabled_toolsets ≠ None → 只包含这些 toolset
disabled_toolsets         → 排除这些 toolset（全量 - 禁用）
两者都为空               → 全量返回
```

### 4.2 check_fn 二次过滤

Registry 在返回 schema 时，会对每个工具调用 `check_fn()` — 检查环境变量/API key 等是否满足：

```python
filtered_tools = registry.get_definitions(tools_to_include, quiet=quiet_mode)
# 只返回 check_fn 通过的工具
```

### 4.3 动态 Schema 修正

**两个重要的动态修正**，防止模型幻觉调用不存在的工具：

**a) execute_code 的 sandbox 工具列表**
```python
if "execute_code" in available_tool_names:
    sandbox_enabled = SANDBOX_ALLOWED_TOOLS & available_tool_names
    dynamic_schema = build_execute_code_schema(sandbox_enabled)
    # 动态替换 schema 中的 allowed_tools 列表
```

**b) browser_navigate 的 web_search 提示**
```python
if "browser_navigate" in available_tool_names:
    web_tools_available = {"web_search", "web_extract"} & available_tool_names
    if not web_tools_available:
        # 删除 schema 描述中 "prefer web_search or web_extract" 的提示
        desc = desc.replace(" For simple information retrieval, prefer...", "")
```

---

## 五、类型强制转换（coerce_tool_args）

### 问题背景

LLM 返回的参数经常类型错误：
- 数字 `"42"` → 字符串
- 布尔 `"true"` → 字符串

### 解决：Schema 驱动的类型修正

```python
def coerce_tool_args(tool_name, args):
    schema = registry.get_schema(tool_name)
    for key, value in args.items():
        expected = schema["parameters"]["properties"][key]["type"]
        args[key] = _coerce_value(value, expected)
```

支持的转换：
| Schema 类型 | 转换规则 |
|-------------|---------|
| `integer` | `"42"` → `42` |
| `number` | `"3.14"` → `3.14` |
| `boolean` | `"true"` → `True` |
| `["integer", "string"]` | 依次尝试，返回首个成功 |

---

## 六、handle_function_call 分发流程

```
handle_function_call(function_name, function_args, task_id, ...)
         ↓
coerce_tool_args()  ← 参数类型修正
         ↓
pre_tool_call hook  ← 插件钩子（可选）
         ↓
_AGENT_LOOP_TOOLS 拦截检查
         ↓
registry.dispatch()  ← 实际执行
         ↓
post_tool_call hook ← 插件钩子（可选）
         ↓
返回 JSON 字符串
```

### Agent Loop 拦截

这些工具不在 model_tools 分发，由 AIAgent 直接处理：

```python
_AGENT_LOOP_TOOLS = {"todo", "memory", "session_search", "delegate_task"}
```

---

## 七、Async 桥接（_run_async）

model_tools 的 handler 大量使用 async（httpx、MCP 等），但调用层是同步的。

### 三种场景

```python
def _run_async(coro):
    try:
        loop = asyncio.get_running_loop()
    except RuntimeError:
        loop = None

    if loop and loop.is_running():
        # 场景1：在 gateway/RL 等 async 上下文中
        # → 启动新线程，asyncio.run() 创建独立 loop
        with ThreadPoolExecutor(max_workers=1) as pool:
            return pool.submit(asyncio.run, coro).result()

    if threading.current_thread() is not threading.main_thread():
        # 场景2：子 Agent 的 worker 线程
        # → per-thread persistent loop，避免跨线程竞争
        worker_loop = _get_worker_loop()
        return worker_loop.run_until_complete(coro)

    # 场景3：CLI 主线程
    # → 共享 persistent loop，避免 GC 时 "Event loop is closed"
    tool_loop = _get_tool_loop()
    return tool_loop.run_until_complete(coro)
```

### 为什么不用 asyncio.run()？

```python
# asyncio.run() 的问题：
# 1. 创建新 loop
# 2. 执行协程
# 3. 关闭 loop  ← httpx/AsyncOpenAI 缓存的客户端在 GC 时尝试关闭 transport
#                但 loop 已关闭 → RuntimeError: Event loop is closed
```

**解决方案**：persistent loop — 循环复用，不关闭。

---

## 八、工具调用追踪

### _last_resolved_tool_names

```python
_last_resolved_tool_names: List[str] = []  # 全局

def get_tool_definitions(...):
    global _last_resolved_tool_names
    _last_resolved_tool_names = [t["function"]["name"] for t in filtered_tools]
```

用途：
- `execute_code` 工具根据这个列表生成 sandbox 代码
- 子 Agent 继承父 Agent 的可用工具集

### _READ_SEARCH_TOOLS 连续读取计数

```python
_READ_SEARCH_TOOLS = {"read_file", "search_files"}

# 当执行非 read/search 工具时，重置连续读取计数器
# （用于检测"连续读文件不做事"的异常模式）
if function_name not in _READ_SEARCH_TOOLS:
    notify_other_tool_call(task_id)
```

---

## 九、整体架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                         AIAgent                                   │
│  run_conversation()                                             │
│       │                                                          │
│       ▼                                                          │
│  model_tools.get_tool_definitions()                             │
│       │                                                          │
│       ├── _discover_tools() → 导入所有 tools/ 模块              │
│       │          ↓                                               │
│       │    registry.register() 注册每个工具                      │
│       │                                                          │
│       ├── enabled/disabled toolset 过滤                         │
│       ├── check_fn() 二次过滤（环境检查）                        │
│       ├── 动态 schema 修正（execute_code / browser_navigate）     │
│       │                                                          │
│       ▼                                                          │
│  返回 tool_schemas → LLM API                                     │
│       │                                                          │
│       ▼                                                          │
│  LLM 返回 tool_calls                                             │
│       │                                                          │
│       ▼                                                          │
│  model_tools.handle_function_call()                              │
│       │                                                          │
│       ├── coerce_tool_args()  ← 类型修正                         │
│       ├── _run_async()  ← 同步→异步桥接                          │
│       ├── registry.dispatch()                                   │
│       │      ↓                                                   │
│       │    工具实现（terminal/file/web/delegate...）              │
│       │                                                          │
│       ▼                                                          │
│  返回 JSON 字符串                                                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 十、关键设计亮点

| 特性 | 说明 |
|------|------|
| **导入即注册** | 工具不需要集中声明，分散在各文件，import 自动 discover |
| **Schema 驱动类型修正** | LLM 参数类型错误在分发前自动修正 |
| **Persistent Event Loop** | 解决 async 工具的 "Event loop is closed" 问题 |
| **动态 Schema** | execute_code 和 browser_navigate 根据实际可用工具动态修正描述 |
| **三层工具来源** | 内置 + MCP + Plugin 统一注册表 |
| **check_fn 门控** | 每个工具声明自己的环境依赖（API key 等），不满足则不返回给 LLM |

---

*文档创建时间: 2026-04-18*
*源码路径: ~/.hermes/hermes-agent/model_tools.py*
