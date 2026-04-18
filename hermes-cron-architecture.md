# Hermes Cron 架构原理与路线图

> 本文档分析 Hermes Agent 的 Cron 调度系统内部原理，并解答「cron 巡检是否经过 hawk-bridge 进入 LanceDB」这一问题。

---

## 一、核心结论（先行回答）

**Cron 巡检任务不经过 hawk-bridge，不会进入 LanceDB。**

原因：Cron Job 由 `AIAgent` 直接执行，绕过了 Hermes Gateway，因此 Hook 的 `agent:start` / `agent:end` 不会被调用。

---

## 二、整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Hermes Gateway                            │
│                                                                  │
│  ┌──────────────┐   ┌─────────────┐   ┌──────────────────────┐  │
│  │  HTTP API    │   │  Platform   │   │  HookRegistry         │  │
│  │  (Feishu/    │   │  Adapters   │   │  - hawk-bridge        │  │
│  │   Discord..) │   │             │   │  - agent:start/end    │  │
│  └──────┬───────┘   └──────┬──────┘   └──────────┬───────────┘  │
│         │                   │                       │              │
│         │            ┌──────▼──────┐                │              │
│         │            │  Message    │                │              │
│         └───────────►│  Handler    │◄───────────────┘              │
│                      │             │                                │
│                      │ context     │  HookRegistry                   │
│                      │   + Hook    │  触发点：                      │
│                      │   执行      │  agent:start → /recall        │
│                      └──────┬──────┘  agent:end   → /capture        │
│                             │                                        │
│                  ┌──────────▼──────────┐                            │
│                  │     AIAgent         │                            │
│                  │  (对话式推理 Agent)  │                            │
│                  └─────────────────────┘                            │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Cron Ticker (后台线程, 每 60s 触发一次)                      │  │
│  │  _start_cron_ticker() → cron.scheduler.tick()               │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│                    cron.scheduler.tick()                           │
│                                                                   │
│  1. 获取文件锁 ~/.hermes/cron/.tick.lock（防止重复执行）           │
│  2. get_due_jobs() → 读取 ~/.hermes/cron/jobs.json               │
│  3. 遍历每个 due job:                                             │
│     a. advance_next_run() — 先推进下次运行时间（防 crash 重跑）    │
│     b. run_job(job)                                               │
│     c. save_job_output() → 本地文件                               │
│     d. _deliver_result() → 推送到 feishu/telegram 等             │
│     e. mark_job_run() — 记录执行结果                              │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│                    cron.scheduler.run_job()                        │
│                                                                   │
│  创建全新 AIAgent 实例（独立 session，无历史上下文）:              │
│                                                                   │
│  agent = AIAgent(                                                 │
│      platform="cron",           ← 关键：platform="cron"           │
│      session_id=f"cron_{job_id}_{timestamp}",                     │
│      disabled_toolsets=["cronjob", "messaging", "clarify"],       │
│      skip_memory=True,         ← 跳过 memory hook                │
│      ...                                                            │
│  )                                                                  │
│  agent.run_conversation(prompt)                                     │
│                                                                   │
│  ⚠️ 这里的 AIAgent 是直接实例化，                                  │
│     不经过 Gateway → 没有 HookRegistry 触发                         │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│              hawk-bridge Hook 触发条件                              │
│                                                                   │
│  只在 Hermes Gateway 的消息处理流程中触发:                          │
│                                                                   │
│  Platform Adapter                                                 │
│      │                                                             │
│      ▼                                                             │
│  Message Handler (gateway/run.py)                                 │
│      │                                                             │
│      ▼                                                             │
│  HookRegistry.trigger("agent:start", context)  ← hawk-bridge 在此  │
│      │                                                             │
│      ▼                                                             │
│  AIAgent.run_conversation()                                        │
│      │                                                             │
│      ▼                                                             │
│  HookRegistry.trigger("agent:end", context)    ← hawk-bridge 在此  │
│                                                                   │
│  Cron Job: run_job() 直接实例化 AIAgent，绕过了整个链路            │
└───────────────────────────────────────────────────────────────────┘
```

---

## 三、关键文件说明

| 文件路径 | 作用 |
|---------|------|
| `~/.hermes/hermes-agent/cron/__init__.py` | 导出接口 |
| `~/.hermes/hermes-agent/cron/jobs.py` | Job 存储、CRUD、调度解析（interval/cron/once） |
| `~/.hermes/hermes-agent/cron/scheduler.py` | tick() 主循环、run_job() 执行逻辑、_deliver_result()投递 |
| `~/.hermes/hermes-agent/gateway/run.py:8722` | `_start_cron_ticker()` — Gateway 内后台线程，每60s触发 |
| `~/.hermes/hermes-agent/agent/__init__.py` | AIAgent 主类 |
| `~/.hermes/hooks/hawk-bridge/handler.py` | Hook 实现：`/recall`（start）和 `/capture`（end） |

---

## 四、执行流程时序图

```
[Gateway 启动]
     │
     ▼
_start_cron_ticker()  ──── 后台线程 ───┐
     │                                 │
     │  每 60s:                        │
     ▼                                 │
scheduler.tick()                       │
     │                                 │
     ▼                                 │
[文件锁] ~/.hermes/cron/.tick.lock     │
     │                                 │
     ▼                                 │
get_due_jobs()                         │
读取 ~/.hermes/cron/jobs.json          │
     │                                 │
     ▼                                 │
对每个 due job:                         │
  advance_next_run()                   │
     │                                 │
     ▼                                 │
  run_job(job)                         │
    │                                 │
    ├── AIAgent(platform="cron", skip_memory=True)  ◄── 绕过 Hook
    │                                 │
    ├── agent.run_conversation(prompt) │
    │                                 │
    ▼                                 │
  save_job_output() → 本地文件         │
     │                                 │
     ▼                                 │
  _deliver_result() → 飞书/本地        │
     │                                 │
     ▼                                 │
  mark_job_run()                       │
```

---

## 五、Cron Job 的 AIAgent 配置（关键差异）

```python
agent = AIAgent(
    model=...,
    platform="cron",              # ← 标记为 cron 类型，非 feishu/discord
    session_id=f"cron_{job_id}_{timestamp}",  # ← 独立 session ID
    disabled_toolsets=["cronjob", "messaging", "clarify"],  # ← 禁用 cronjob tool
    skip_memory=True,              # ← 跳过 memory hook（防止用户表征被污染）
    skip_context_files=True,       # ← 跳过 SOUL.md/AGENTS.md
    quiet_mode=True,
    ...
)
```

**对比 Gateway 中的 AIAgent：**

```python
# Gateway 消息处理中的 AIAgent
agent = AIAgent(
    platform=platform,             # ← feishu / discord / telegram
    session_id=session_id,         # ← 真实用户 session
    ...
)
# 然后：
HookRegistry.trigger("agent:start", context)  # ← hawk-bridge 触发
agent.run_conversation()
HookRegistry.trigger("agent:end", context)   # ← hawk-bridge 触发
```

---

## 六、为什么 Cron 不过 hawk-bridge？

1. **调用路径不同**
   - Gateway 对话：`Message Handler → HookRegistry → AIAgent`
   - Cron Job：`scheduler.tick() → AIAgent`（直接）

2. **HookRegistry 只在 Gateway 消息处理中调用**
   - `HookRegistry.trigger("agent:start", ...)` 只在 `gateway/run.py` 的消息处理流程中调用
   - `scheduler.run_job()` 直接 `AIAgent()` 实例化，没有任何 Hook 调用

3. **skip_memory=True**
   - Cron 的 AIAgent 显式设置 `skip_memory=True`，明确禁用 memory hook

4. **独立的 session_id**
   - Cron session ID 格式为 `cron_{job_id}_{timestamp}`，与用户 session 完全隔离

---

## 七、如果希望 Cron 结果进入 LanceDB？

### 方案 A：在 prompt 中调用 API（推荐）
在 Cron job 的 prompt 中让 agent 主动调用 `hawk-memory-api` 的 `/capture` 接口：

```python
# 在 prompt 中添加：
# 请将本次巡检结果 capture 到记忆系统：
# curl -X POST http://127.0.0.1:18360/capture \
#   -H "Content-Type: application/json" \
#   -d '{"session_id": "...", "user_id": "...", "message": "...", "response": "..."}'
```

### 方案 B：改造 run_job() 事后写入
在 `scheduler.run_job()` 的 `run_job()` 函数末尾，追加对 `/capture` 的调用。需修改 `~/.hermes/hermes-agent/cron/scheduler.py`。

### 方案 C：改造为走 Gateway 消息流
将 Cron 改为通过 `HookRegistry.trigger("agent:end", context)` 触发 hawk-bridge，需要较大的架构调整。

---

## 八、调度器详细参数

| 参数 | 值 | 说明 |
|------|-----|------|
| tick 间隔 | 60s | Gateway 后台线程控制 |
| 文件锁 | `~/.hermes/cron/.tick.lock` | 防重复执行 |
| Job 存储 | `~/.hermes/cron/jobs.json` | JSON 文件 |
| 输出存储 | `~/.hermes/cron/output/{job_id}/{timestamp}.md` | 本地审计 |
| 单次超时 | 默认 600s（可通过 `HERMES_CRON_TIMEOUT` 环境变量修改） | 静默超时 |
| 捕获超时 | `HERMES_CRON_SCRIPT_TIMEOUT` / `script_timeout_seconds` | Cron 预执行脚本 |

---

## 九、schedule 格式支持

| 格式 | 示例 | 类型 |
|------|------|------|
| 间隔 | `every 5m` / `every 2h` | recurring interval |
| Cron 表达式 | `*/5 * * * *` | recurring cron |
| 持续时间 | `30m` / `2h` | one-shot（从现在起） |
| ISO 时间戳 | `2026-04-20T09:00` | one-shot（固定时间） |

---

## 十、已知限制

1. **Cron 不过 Hook**：如本文档所述，Cron Job 绕过了 Hook 系统
2. **独立 session**：Cron 的 session_id 与用户会话隔离，无法做跨任务关联记忆
3. **60s 精度**：tick 间隔为 60s，最小调度精度为 1 分钟
4. **文件锁竞争**：在高频场景下文件锁可能成为瓶颈

---

*文档创建时间: 2026-04-18*
*源码版本: Hermes Agent v0.8.0*
*源码路径: ~/.hermes/hermes-agent/*
