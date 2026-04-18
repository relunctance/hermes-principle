# Hermes 消息队列与并发处理原理

> 本文档分析 Hermes Gateway 的消息队列机制，解答「飞书发 1 条 vs 连续发 5 条消息有何不同处理效果」。

---

## 一、核心结论

| 场景 | 处理行为 | 结果 |
|------|---------|------|
| **发 1 条** | 正常流程：消息 → AIAgent → 回复 | ✅ 正常 |
| **发 5 条（连续快速）** | 第 1 条正常处理；第 2-5 条**全部替换覆盖**为最后一条 | ⚠️ 只有最后一条被处理 |
| **发 5 条（最后一条含图片）** | 文本被覆盖为最后一条，但图片会被**合并保留**（如果是 PHOTO 类型的 burst） | ⚠️ 只有最后一条文本 + 所有图片 |

---

## 二、架构总览

```
飞书用户 ───────────────────────────────────────────────────────────►

  消息 A (文本)
    │
    ▼
  FeishuAdapter.handle_message()
    │
    ▼
  [session_key = f"feishu_{chat_id}"]
    │
    ├─── 如果 session 无 active agent ──────────────────────────────► 走正常流程
    │        AIAgent.run_conversation() → HookRegistry → 回复
    │
    └─── 如果 session 已有 agent 在跑 ──────────────────────────────►
              │
              ▼
         _pending_messages[session_key] = event   ← 覆盖写入（替换）
              │
              ▼
         _active_sessions[session_key].set()     ← 触发 interrupt 信号
              │
              ▼
         Agent 被 interrupt 后，检查 _pending_messages
              │
              ▼
         发现有 pending，取出 → 作为下一轮输入
              │
              ▼
         递归调用 _run_agent(message=pending)      ← 继续处理
```

---

## 三、关键数据结构

### 1. `_pending_messages`（Adapter 级别）

```python
# gateway/platforms/base.py

self._pending_messages: Dict[str, MessageEvent] = {}
# Key: session_key (e.g., "feishu_ou_xxx")
# Value: MessageEvent（只有最新的一条）
```

### 2. `_running_agents`（Gateway 级别）

```python
# gateway/run.py

self._running_agents: Dict[str, AIAgent] = {}
# Key: session_key
# Value: AIAgent 实例 或 _AGENT_PENDING_SENTINEL（正在启动中）
```

---

## 四、处理流程详解

### 场景 1：发 1 条消息

```
用户发送: "你好"
  │
  ▼
FeishuAdapter.handle_message(event)
  │
  ▼
检查 _active_sessions[session_key]?
  → 不存在（session 空闲）
  │
  ▼
spawn _process_message_background(event, session_key)
  │
  ▼
_run_agent() → AIAgent → HookRegistry → 回复
  │
  ▼
_mark_session_done() → 清理 _active_sessions
```

**结果**：正常处理，正常回复。

---

### 场景 2：连续发 5 条文本消息

```
时刻 T0: 用户发送 "第1条"
  │
  ▼
  AIAgent 开始处理第1条（假设需要 30s）
  │
  ▼

时刻 T1: 用户发送 "第2条"（AIAgent 还在处理第1条）
  │
  ▼
  _pending_messages["feishu_xxx"] = "第2条"  ← 覆盖！
  → 之前存的 "第1条" 被替换掉
  │
  ▼
  _active_sessions["feishu_xxx"].set()         ← 触发 interrupt

时刻 T2: 用户发送 "第3条"
  │
  ▼
  _pending_messages["feishu_xxx"] = "第3条"  ← 再次覆盖！
  │
  ▼

时刻 T3: 用户发送 "第4条"
  │
  ▼
  _pending_messages["feishu_xxx"] = "第4条"  ← 再次覆盖！
  │
  ▼

时刻 T4: 用户发送 "第5条"
  │
  ▼
  _pending_messages["feishu_xxx"] = "第5条"  ← 再次覆盖！
  │
  ▼

时刻 T30: AIAgent 处理完第1条，准备回复前检查 pending
  │
  ▼
  _dequeue_pending_event() → 返回 "第5条"
  │
  ▼
  递归 _run_agent(message="第5条")           ← 只有第5条被处理
```

**结果**：只有第 5 条被处理，第 1-4 条全部丢失。

---

### 场景 3：发文本 + 多张图片（PHOTO burst）

```
用户发送: [图片A] + [图片B] + [图片C] + "请分析这些"
  │
  ▼
  3 个 PHOTO 事件几乎同时到达
  │
  ▼
  FeishuAdapter.handle_message() 处理每个 PHOTO 事件
  │
  ▼
  merge_pending_message_event() 被调用:
  │
  if existing 是 PHOTO AND 新事件也是 PHOTO:
      existing.media_urls.extend(event.media_urls)  ← 合并所有图片
      existing.text = _merge_caption(existing.text, event.text)
      return  ← 不覆盖，保留之前的
  │
  ▼
  最终 _pending_messages 里有:
  {
    media_urls: [图片A.url, 图片B.url, 图片C.url],
    text: "请分析这些"
  }
```

**结果**：所有图片被合并，只有最后一张图的 caption 文本被保留。

---

## 五、`merge_pending_message_event()` 合并策略

```python
def merge_pending_message_event(pending_messages, session_key, event):
    existing = pending_messages.get(session_key)

    # 条件：两张都是 PHOTO → 合并媒体（图片 burst）
    if (existing
        and existing.message_type == MessageType.PHOTO
        and event.message_type == MessageType.PHOTO):
        existing.media_urls.extend(event.media_urls)
        existing.media_types.extend(event.media_types)
        # 文字取新的（caption 合并）
        existing.text = _merge_caption(existing.text, event.text)
        return

    # 其他所有情况 → 直接覆盖（替换）
    pending_messages[session_key] = event
```

| 消息类型组合 | 行为 |
|-------------|------|
| 文本 → 文本 | ❌ 替换（后者覆盖前者） |
| 文本 → PHOTO | ❌ 替换 |
| PHOTO → PHOTO | ✅ 合并（追加 media_urls） |
| PHOTO → 文本 | ❌ 替换 |

---

## 六、Interrupt（中断）机制

当新消息到达时，如果当前 session 正在处理：

```python
# gateway/platforms/base.py:1546-1551

# 非 PHOTO 消息：立即 interrupt
logger.debug("[%s] New message while session %s is active — triggering interrupt", self.name, session_key)
self._pending_messages[session_key] = event    # 覆盖 pending
self._active_sessions[session_key].set()       # 发出中断信号
```

AIAgent 内部有一个 `interrupt_monitor` 协程，在每个 API 调用前检查 `_interrupt_requested`：

```python
# gateway/run.py:8268-8283

if hasattr(_adapter, 'has_pending_interrupt') and _adapter.has_pending_interrupt(session_key):
    agent.interrupt(pending_text)  # 中断当前 agent
```

被 interrupt 后，agent 会：
1. 丢弃当前正在生成的回复
2. 取出 `pending_message`
3. 递归调用 `_run_agent(message=pending)`

---

## 七、递归深度保护

```python
# gateway/run.py:8587

if _interrupt_depth >= self._MAX_INTERRUPT_DEPTH:
    # 超过最大递归深度，不再处理，直接排队
    merge_pending_message_event(adapter._pending_messages, session_key, pending_event)
    return result
```

防止用户疯狂发消息导致栈溢出。

---

## 八、总结

| 问题 | 答案 |
|------|------|
| Hermes 有命令队列吗？ | ✅ 有，但只是一个**单条覆盖式**的 pending slot，不是真正的 FIFO 队列 |
| 发 1 条 vs 发 5 条有区别吗？ | **有巨大区别**：只有最后一条会被处理，中间 4 条全部丢失 |
| 连续发图片呢？ | 图片 URL 会合并（PHOTO burst 特殊处理），但文本依然只有最后一条 |
| 可以积累多条一起处理吗？ | 不行。`_pending_messages` 是单个 slot，不是 list |
| interrupt 和 queue 是一回事吗？ | 是的。interrupt 触发 → 消息存入 pending → agent 下一轮取出 |

---

## 九、设计意图

这种「替换式 pending」是**有意为之**的设计：

- **用户体验**：用户连续发多条，通常是想替换之前说的内容（如打字打了一半补发）
- **避免重复**：不需要处理用户已经「撤回」的内容
- **图片特殊处理**：PHOTO burst 合并是因为多张图通常是一次性发的

但这也意味着：
- 快速连发 5 条 = 只有最后一条生效
- 如果需要发送多条完整内容，需要等 agent 回复后再发下一条

---

*文档创建时间: 2026-04-18*
*源码版本: Hermes Agent v0.8.0*
