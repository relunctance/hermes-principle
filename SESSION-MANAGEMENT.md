# Hermes Session 管理原理

> 本文档分析 Hermes Gateway 的 Session 管理机制，包括 session key 生成、生命周期、存储和 reset 策略。

---

## 一、核心概念：三层 ID 体系

Hermes 使用三层 session 标识，严格分层：

```
session_key      = "agent:main:feishu:dm:ou_xxx"     # 会话桶标识（路由用）
session_id       = "sess_4a3f8c2d"                   # 独立会话实例（历史隔离）
message_id       = 自增 INTEGER                         # 消息主键
```

| 层级 | 用途 | 持久化 |
|------|------|--------|
| `session_key` | 路由：哪个 session 在跑、interrupt、并发控制 | ✅ 写入 sessions.json |
| `session_id` | 历史：一个 session_key 可以有多个 session_id（reset 后变成新 session） | ✅ 写入 SQLite sessions 表 |
| `message_id` | 消息：自增主键关联 messages 表 | ✅ 写入 SQLite messages 表 |

---

## 二、session_key 的构造规则

`sessioin_key` 由 `build_session_key()` 生成：

```
session_key 格式: agent:main:{platform}:{chat_type}:{chat_id}:{thread_id?}:{user_id?}
```

### DM（私信）

```
feishu:dm:ou_xxx                          → 两个人之间的私信
feishu:dm:ou_xxx:thread_abc               → 私信中的线程
```

### Group（群组）

```
feishu:group:gc_xxx                       → 共享 session（默认）
feishu:group:gc_xxx:u_yyy                 → 按用户隔离的 session
feishu:group:gc_xxx:thread_abc            → 共享线程
feishu:group:gc_xxx:thread_abc:u_yyy     → 用户+线程隔离
```

### 关键规则

- **DM**：按 `chat_id` 隔离，两个人的对话独立一个 session
- **Group**：默认共享；可配置 `group_sessions_per_user=True` 按用户隔离
- **Thread**：默认共享（所有参与者看同一个对话）；可配置 `thread_sessions_per_user=True` 按用户隔离
- **本地 CLI**：`agent:main:local:dm`

---

## 三、Session 生命周期

```
用户发消息
    │
    ▼
build_session_key()        ← 从消息 source 构建 session_key
    │
    ▼
SessionStore.get_or_create_session()
    │
    ├─── 检查 sessions.json 中是否有该 session_key
    │
    ├─── 有 → 检查 reset policy（idle/daily）
    │        │
    │        ├─── 未过期 → 返回现有 SessionEntry
    │        └─── 已过期 → 创建新 session_id（标记 was_auto_reset）
    │
    └─── 无 → 创建新 SessionEntry（session_id = uuid）
             │
             ▼
         写入 SQLite sessions 表（started_at = now）
             │
             ▼
         AIAgent 被创建/复用
             │
             ▼
         用户与 Agent 对话（多轮）
             │
             ▼
         Session 结束（idle 超时 / daily reset / 用户 new）
             │
             ▼
         ended_at = now，end_reason 记录
```

---

## 四、Session 存储：SQLite + JSON

### 两套存储并存

| 存储 | 文件 | 内容 |
|------|------|------|
| **SQLite** | `~/.hermes/state.db` | 完整消息历史、FTS5 搜索、会话元数据 |
| **JSON** | `~/.hermes/sessions/sessions.json` | session_key → SessionEntry 映射（仅索引） |

### SQLite Schema

```sql
sessions 表:
  id                  -- session_id（主键）
  source              -- 'feishu' / 'discord' / 'cli' 等
  user_id             -- 用户标识
  model               -- 使用的模型
  system_prompt       -- 创建时的 system prompt
  parent_session_id   -- reset 后指向前一个 session（链式）
  started_at          -- 开始时间
  ended_at            -- 结束时间
  end_reason          -- 'idle' / 'daily' / 'manual'
  message_count       -- 消息数
  -- token 统计...

messages 表:
  id           -- 自增主键
  session_id   -- 外键
  role         -- 'user' / 'assistant' / 'system'
  content      -- 消息内容
  tool_calls   -- 工具调用 JSON
  timestamp    -- 时间戳

messages_fts 表 (FTS5):
  -- content 列的全文索引，支持 session_search
```

### WAL 模式

```python
state.db 使用 WAL 模式：
- 多个读事务可以并发
- 写事务排队（只有一个写）
- 写锁竞争时 application-level retry（随机 jitter 20-150ms）
- 每 50 次写操作执行一次 passive checkpoint
```

---

## 五、Reset Policy（会话重置策略）

在 `GatewayConfig` 中定义：

```python
class SessionResetPolicy(NamedTuple):
    mode: str           # "none" | "idle" | "daily" | "both"
    idle_minutes: int   # 空闲多少分钟后重置
    at_hour: int        # 每日重置的小时（本地时间）
```

### 四种模式

| 模式 | 说明 | 典型用途 |
|------|------|---------|
| `none` | 永不自动重置 | 永久对话 |
| `idle` | 超过 N 分钟空闲后重置 | 长期任务 |
| `daily` | 每天固定时间重置 | 日报类场景 |
| `both` | 满足任一条件即重置 | 保守策略 |

### Reset 后的行为

```python
# 新 session_id 生成，但保留 parent_session_id 链：
# old session:  id = "sess_old", ended_at = now, end_reason = "idle"
# new session:  id = "sess_new", parent_session_id = "sess_old"
```

用户收到提示：
```
[SYSTEM: 之前的会话已自动重置（空闲超时）。]
```

---

## 六、并发控制：_running_agents

Gateway 在内存中追踪正在跑的 agent：

```python
# gateway/run.py

self._running_agents: Dict[str, AIAgent] = {}
# Key: session_key
# Value: AIAgent 实例 或 _AGENT_PENDING_SENTINEL（正在启动中）
```

### 防重入机制

```
消息A 到达
    │
    ▼
session_key = "feishu:dm:ou_xxx"
    │
    ▼
if session_key in _running_agents:
    # 已有 agent 在跑 → interrupt / queue
    pass
else:
    _running_agents[session_key] = _AGENT_PENDING_SENTINEL  ← 先占位
    │
    ▼
    AIAgent 创建
    │
    ▼
    _running_agents[session_key] = agent  ← 真实 agent
    │
    ▼
    agent.run_conversation()
    │
    ▼
    _running_agents.pop(session_key)  ← 完成后删除
```

> **为什么用 sentinel？**
> 占位符在 AIAgent 创建的 **async gap** 中发挥作用——如果第二条消息在 agent 创建过程中到达，会看到 sentinel 而不是 `None`，从而正确进入 queue 逻辑，而不是创建重复 agent。

---

## 七、Session 缓存（Agent 复用）

```python
# gateway/run.py

self._agent_cache: Dict[str, AIAgent] = {}
# Key: session_key
# Value: AIAgent 实例（保留，期望复用）
```

```python
# AIAgent 初始化时检查是否复用：
agent = AIAgent(
    session_id=entry.session_id,
    platform=platform,
    # ...
)

# 如果 session 没有 reset：
# → 复用缓存中的 AIAgent（保留 prompt caching）
# 如果 session 已 reset：
# → 驱逐缓存，创建全新 AIAgent
```

---

## 八、SessionContext（动态 System Prompt 注入）

每次消息都会构建 `SessionContext` 注入 system prompt：

```python
# session.py

@dataclass
class SessionContext:
    source: SessionSource           # 消息来源（平台、用户、群组）
    connected_platforms: List       # 当前连接的平台列表
    home_channels: Dict             # 各平台的 Home Channel
    session_key: str
    session_id: str
    created_at: datetime
    updated_at: datetime
```

然后 `build_session_context_prompt()` 生成：

```
## Current Session Context

**Source:** Feishu (DM with 张三)
**User:** user_4a3b8c2d
**Connected Platforms:** local (files on this machine), feishu: Connected ✓

**Home Channels (default destinations):**
  - feishu: 飞书通知 (ID: oc_xxx)

**Delivery options for scheduled tasks:**
- `"origin"` → Back to this chat (oc_xxx)
- `"local"` → Save to local files only
```

---

## 九、完整消息流

```
飞书消息到达
    │
    ▼
FeishuAdapter.handle_message()
    │
    ▼
build_session_key(source) → "agent:main:feishu:dm:ou_xxx"
    │
    ▼
SessionStore.get_or_create_session()
    │
    ├─── 检查 sessions.json 索引
    ├─── 检查 idle/daily reset policy
    └─── 返回 SessionEntry（session_id, was_auto_reset 等）
    │
    ▼
if session_key in _running_agents:
    → interrupt（中断现有 agent）
    → _pending_messages[session_key] = 新消息
else:
    → _running_agents[session_key] = sentinel
    │
    ▼
AIAgent(session_id=entry.session_id, platform="feishu")
    │
    ▼
build_session_context_prompt() → system prompt 片段
    │
    ▼
HookRegistry.trigger("agent:start") → hawk-bridge /recall
    │
    ▼
agent.run_conversation()
    │
    ▼
HookRegistry.trigger("agent:end") → hawk-bridge /capture
    │
    ▼
SessionDB.append_message() → 写入 SQLite messages 表
    │
    ▼
SessionStore 更新 updated_at + token 统计
    │
    ▼
_pending_messages 检查（是否有排队的消息）
    │
    ├─── 有 → 递归 _run_agent（处理下一条）
    └─── 无 → _running_agents.pop() → 会话结束
```

---

## 十、关键文件索引

| 文件 | 职责 |
|------|------|
| `gateway/session.py` | Session key 生成、SessionStore、SessionEntry、SessionSource、SessionContext |
| `hermes_state.py` | SessionDB（SQLite WAL）、FTS5 搜索 |
| `gateway/run.py` | `_running_agents`、`_agent_cache`、并发控制 |
| `gateway/session_context.py` | `ContextVar` 并发隔离 |

---

*文档创建时间: 2026-04-18*
*源码版本: Hermes Agent v0.8.0*
