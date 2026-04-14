# Hermes Gateway 内部原理与运行机制

> 基于源码分析：gateway/run.py 全部 ~9000 行

## 架构定位

Gateway 是 Hermes 的**消息中枢**，负责：
- 接收来自 14+ 平台的 inbound 消息
- 管理 AIAgent 生命周期
- 协调 Tool 执行与响应回传

```
用户 → Platform Adapter → GatewayRunner → AIAgent → Tools
                  ↑                                    │
                  └────────── 响应回传 ←───────────────┘
```

## 核心类

### GatewayRunner

```python
# gateway/run.py
class GatewayRunner:
    adapters: Dict[Platform, BasePlatformAdapter]  # 已连接的平台
    session_store: SessionStore                     # 会话存储
    _running_agents: Dict[str, AIAgent]           # 运行中的 Agent（per session）
    _agent_cache: Dict[str, tuple]                 # Agent 实例缓存（保持 prompt caching）
    _pending_messages: Dict[str, str]              # 中断时排队的消息
    _session_model_overrides: Dict[str, Dict]     # /model 命令的 per-session 覆盖
    _pending_approvals: Dict[str, Dict]            # 待审批的命令
    hooks: HookRegistry                            # 事件钩子
```

## 启动流程

```python
# gateway/run.py
async def start_gateway(...) -> bool:
    config = load_gateway_config()
    runner = GatewayRunner(config)
    await runner.start()  # 启动所有已配置的 platform adapters
```

### GatewayRunner.start()

```
1. 日志初始化
2. 检查用户白名单配置（未配置则警告）
3. hooks.discover_and_load() — 发现事件钩子
4. process_registry.recover_from_checkpoint() — 崩溃恢复
5. session_store.suspend_recently_active() — 暂停上次遗留的会话
6. 遍历 config.platforms:
   a. _create_adapter(platform, config)
   b. adapter.set_message_handler(self._handle_message)
   c. adapter.set_fatal_error_handler(...)
   d. adapter.connect() — 连接平台
7. 返回 connected_count
```

### 平台连接流程

```python
# 每个 Platform Adapter
async def connect(self) -> bool:
    if self.config.connection_mode == "websocket":
        await self._connect_websocket()
    else:
        await self._connect_webhook()
```

支持的连接模式：
- **WebSocket 长连接**（优先）：平台主动推送
- **Webhook HTTP 回调**：Hermes 暴露 HTTP 端点

## 消息处理流程

### 入口：_handle_message(event)

```python
async def _handle_message(self, event: MessageEvent) -> Optional[str]:
    """
    核心消息处理管道：
    1. 内部消息跳过授权
    2. 未知用户 → pairing 流程
    3. /update 响应拦截
    4. Staleness 驱逐（超时的 _running_agents 条目）
    5. 有运行中的 Agent → 中断/排队/处理命令
    6. 无运行中的 Agent → 命令处理 → _handle_message_with_agent
    """
```

### 关键机制 1：Sentinel 防止重复 Agent

```python
# 在 _handle_message 中，await 任何操作之前：
self._running_agents[_quick_key] = _AGENT_PENDING_SENTINEL
#                       ↑ 标记"正在启动中"，防止第二条消息也创建 Agent

try:
    return await self._handle_message_with_agent(event, source, _quick_key)
finally:
    # 异常时清理 sentinel，避免会话永久锁死
    if self._running_agents.get(_quick_key) is _AGENT_PENDING_SENTINEL:
        del self._running_agents[_quick_key]
```

**为什么需要这个？**
- `_handle_message` 中有多个 `await` 点（hooks、vision enrichment、STT）
- 如果在 await 期间第二条消息到达，会错误创建第二个 Agent
- Sentinel + `finally` 保证任何退出路径都会清理

### 关键机制 2：Staleness 驱逐

```python
# 检测泄漏的锁（Agent 崩溃但未清理）
_stale_age = time.time() - self._running_agents_ts[_quick_key]
_stale_idle = agent.get_activity_summary()["seconds_since_activity"]

if _stale_idle >= HERMES_AGENT_TIMEOUT or _stale_age > max(10*TIMEOUT, 7200):
    del self._running_agents[_quick_key]  # 驱逐
```

### 关键机制 3：Agent 缓存（Prompt Caching）

```python
# AIAgent 重建系统提示很昂贵（含 memory、skills）
# 缓存保证同一 session_key 复用同一个 AIAgent 实例

_sig = self._agent_config_signature(model, runtime, toolsets, ephemeral_prompt)
with _agent_cache_lock:
    cached = _agent_cache.get(session_key)
    if cached and cached[1] == _sig:
        agent = cached[0]  # 复用
    else:
        agent = AIAgent(...)
        _agent_cache[session_key] = (agent, _sig)
```

## 消息预处理（_prepare_inbound_message_text）

```
1. 回复上下文：/[Replying to: "..."] 注入
2. @ 上下文引用：@file → 文件内容展开
3. 图像富媒体：vision enrichment → 图片描述
4. 语音转文本：STT 转换
5. 文档处理：text 文档内容注入
```

## 命令处理

### Slash 命令路由

```python
# 优先于 Agent 执行
if command in GATEWAY_KNOWN_COMMANDS:
    canonical = resolve_command(command).name

    if canonical == "new":       → _handle_reset_command
    if canonical == "help":     → _handle_help_command
    if canonical == "status":   → _handle_status_command
    if canonical == "restart":  → _handle_restart_command
    if canonical == "stop":     → _handle_stop_command
    if canonical == "model":    → _handle_model_command
    if canonical == "approve":  → _handle_approve_command
    if canonical == "deny":     → _handle_deny_command
    if canonical == "background":→ _handle_background_command
    if canonical == "queue":    → 排队不中断
```

### /new、/reset 特殊处理

```python
# 这两个命令必须绕过 running-agent 守卫
# 否则会被当作普通文本注入 Agent，产生错误的上下文
if canonical == "new":
    running_agent.interrupt("Session reset")
    adapter.get_pending_message(_quick_key)  # 消费并丢弃
    del self._running_agents[_quick_key]
    return await self._handle_reset_command(event)
```

## Agent 运行：_run_agent()

```python
async def _run_agent(message, context_prompt, history, source, session_id, session_key):
    """
    在线程池中运行（不阻塞事件循环）
    支持新消息中断
    """
    def run_sync():
        # 1. 解析配置
        model, runtime_kwargs = self._resolve_session_agent_runtime(...)

        # 2. 检查 Agent 缓存
        agent = self._get_cached_agent(session_key, model, runtime_kwargs, ...)
        if not agent:
            agent = AIAgent(model=model, **runtime_kwargs, ...)
            self._cache_agent(session_key, agent)

        # 3. 设置回调
        agent.tool_progress_callback = progress_callback    # 工具进度
        agent.step_callback = _step_callback_sync           # agent:step 钩子
        agent.stream_delta_callback = _stream_delta_cb      # token 流式
        agent.interim_assistant_callback = _interim_assistant_cb  # 中间注释
        agent.status_callback = _status_callback_sync      # 状态回调
        agent.background_review_callback = _bg_review_send  # Memory 更新通知

        # 4. 注册审批回调（危险命令）
        register_gateway_notify(session_key, _approval_notify_sync)

        try:
            result = agent.run_conversation(message, conversation_history=agent_history)
        finally:
            unregister_gateway_notify(session_key)

        # 5. 流式消费结束
        _stream_consumer.finish()
        return result
```

### 进度消息机制

```
Agent 执行工具 → progress_callback("tool.started", tool_name, args)
                    │
                    ▼
               progress_queue.put(msg)
                    │
                    ▼
            send_progress_messages() (async)
                    │
                    ├─► adapter.send() — 第一条工具消息
                    │
                    └─► adapter.edit_message() — 编辑更新后续工具
```

支持编辑的平台（Telegram等）：进度消息会**原地编辑**，不产生噪音。
不支持编辑的平台（Webhook等）：每条进度都是独立消息。

### 中断机制

```python
# 新消息到达时，如果 session 有正在运行的 Agent
running_agent.interrupt(event.text)
# interrupt() 设置 _should_stop = True，Agent 在下一个 await 处检查并退出

# 如果 Agent 真的卡死了（threading.Event 阻塞）
# /stop 执行 HARD STOP：强制删除 _running_agents 条目
del self._running_agents[_quick_key]
session_store.suspend_session(_quick_key)  # 下次消息触发自动 reset
```

## 事件钩子系统

```python
self.hooks: HookRegistry

# 支持的钩子：
"session:start"       # 新会话开始
"agent:step"         # Agent 每轮迭代完成
"command:<name>"      # 任意 slash 命令执行前
```

## 会话管理

### SessionStore

```python
session_store.get_or_create_session(source) → SessionEntry
session_store.load_transcript(session_id) → List[Dict]
session_store.save_messages(session_id, messages)
session_store.suspend_session(session_key)    # 暂停（崩溃恢复）
session_store.suspend_recently_active()       # 启动时暂停遗留会话
```

### 自动会话重置

| 原因 | 策略 |
|------|------|
| `idle` | 超过 `idle_minutes` 无活动 |
| `daily` | 每天定时 |
| `suspended` | 崩溃后手动恢复 |

重置时：`[System note: The user's previous session expired...]` 注入上下文。

## 配置体系

```python
# ~/.hermes/config.yaml
model:
  default: MiniMax-M2.7-highspeed
  provider: custom
  base_url: https://api.minimaxi.com/anthropic

agent:
  max_turns: 90
  gateway_timeout: 1800

gateway:
  platforms:
    feishu:
      enabled: true
      connection_mode: websocket  # 或 webhook
      app_id: ...
      app_secret: ...
```

## 与 OpenClaw Claw 对比

| 维度 | Hermes Gateway | OpenClaw Claw |
|------|--------------|--------------|
| 连接模式 | WebSocket + Webhook | WebSocket |
| 消息模型 | MessageEvent 标准化 | 平台原始事件 |
| Agent 管理 | 缓存 + Sentinel | 每次新建 |
| 会话 | SQLite FTS5 | 文件 |
| 中断 | interrupt() | 无（排队） |

## 关键设计决策

### 1. 线程池执行 Agent

```python
# _run_agent 中的 run_sync() 在 ThreadPoolExecutor 执行
# 原因：agent.run_conversation() 是同步的，但 Gateway 是 async 的
loop.run_in_executor(None, run_sync)
```

### 2. 进度消息去重

```python
# 同一工具连续触发（如 execute_code 多次重试）会收到相同 preview
# 合并为 "tool(x) (×3)" 避免刷屏
if msg == last_progress_msg:
    repeat_count[0] += 1
    progress_queue.put(("__dedup__", base_msg, repeat_count))
```

### 3. Clean Shutdown 标记

```python
# 正常退出时写 .clean_shutdown，下次启动跳过 session suspend
# 崩溃退出没有此文件 → 启动时暂停所有遗留会话
_clean_marker = _hermes_home / ".clean_shutdown"
if _clean_marker.exists():
    _clean_marker.unlink()
else:
    session_store.suspend_recently_active()
```

## 源码结构

```
gateway/
├── run.py                    # GatewayRunner (~9000 行)
├── platforms/
│   ├── base.py              # BasePlatformAdapter, MessageEvent (~1600 行)
│   ├── feishu.py            # FeishuAdapter
│   ├── telegram.py
│   ├── discord.py
│   └── ...
├── config.py                 # GatewayConfig
├── session.py                # SessionStore
├── hooks.py                  # HookRegistry
├── display_config.py         # Per-platform 显示配置
├── stream_consumer.py       # Token 流式消费
├── pairing.py               # 用户授权配对
└── status.py                # 运行时状态写入
```
