# Hermes Iteration Budget（迭代预算）机制

## 问题现象

执行复杂任务时，经常看到 Agent 显示：

```
23/90  →  24/90  →  ...  →  90/90  →  ⚠️ Iteration budget exhausted
```

任务突然中断，未完成的操作就挂了。

---

## 原理：Iteration Budget 是什么？

### 核心概念

**Iteration Budget** 是 Hermes 用来防止 Agent 进入**无限循环**的安全机制。

每次 LLM 调用工具（tool call），就算一次 iteration。默认上限是 **90 次**，超过就强制终止。

### 源码位置

```
run_agent.py
├── IterationBudget 类（第 170-211 行）
├── AIAgent.__init__（第 527 行 max_iterations=90）
└── 主循环条件（第 7870 行）
```

### IterationBudget 类

```python
class IterationBudget:
    """Thread-safe iteration counter for an agent."""

    def __init__(self, max_total: int):
        self.max_total = max_total  # 默认 90
        self._used = 0
        self._lock = threading.Lock()

    def consume(self) -> bool:
        """Try to consume one iteration. Returns True if allowed."""
        with self._lock:
            if self._used >= self.max_total:
                return False  # 预算耗尽
            self._used += 1
            return True

    def refund(self) -> None:
        """退回一次 iteration（例如 execute_code 的循环）。"""
        with self._lock:
            if self._used > 0:
                self._used -= 1

    @property
    def remaining(self) -> int:
        with self._lock:
            return max(0, self.max_total - self._used)
```

### 主循环条件

```python
# run_agent.py 第 7870 行
while (api_call_count < self.max_iterations and self.iteration_budget.remaining > 0) \
      or self._budget_grace_call:
    # ... 执行工具调用
```

---

## 为什么是 90 次？

90 次是 **默认值**，不是硬编码。你可以在 `config.yaml` 中调整：

```yaml
agent:
  max_turns: 90  # 主 Agent 的 iteration 上限
```

### 90 次能做什么？

| 操作 | 平均消耗 |
|------|---------|
| 简单问答（无需工具） | 1-2 次 |
| 读文件 + 写文件 | 3-5 次 |
| Git 操作（读日志 + 提交 + 推送） | 5-8 次 |
| 复杂 debug（多次 search + read + patch） | 15-30 次 |
| 多文件重构（跨多个仓库） | 40-80 次 |

**复杂任务很容易在 90 次内完不成。**

---

## 耗尽时会发生什么？（Grace Period 机制）

Hermes **不会**直接杀掉任务，而是有一套 Grace Period：

### 流程图

```
api_call_count = 89（最后一次正常调用）
    │
    ▼
工具返回结果，api_call_count 变成 90
    │
    ▼
iteration_budget.remaining == 0 → consume() 返回 False
    │
    ▼
检测到 _budget_exhausted_injected == False
    │
    ▼
注入一条 "summarise" 消息给 LLM
"⚠️ Iteration budget exhausted — please summarise what you've done"
    │
    ▼
_budget_exhausted_injected = True
_budget_grace_call = True  ← 开启 grace period
    │
    ▼
允许再调用一次 LLM（grace call）
    │
    ▼
如果这次返回了文字 → 正常结束
    │
    ▼
如果这次还是 tool_calls → 强制终止
```

### 源码注释（第 750-757 行）

```python
# Iteration budget: the LLM is only notified when it actually exhausts
# the iteration budget (api_call_count >= max_iterations).  At that
# point we inject ONE message, allow one final API call, and if the
# model doesn't produce a text response, force a user-message asking
# it to summarise.  No intermediate pressure warnings — they caused
# models to "give up" prematurely on complex tasks (#7915).
self._budget_exhausted_injected = False
self._budget_grace_call = False
```

---

## 为什么模型会"放弃"？

注释里提到 **#7915 issue**：如果中间给模型压力警告，模型会**提前放弃**。

所以 Hermes 的策略是：
- ❌ **不告诉模型** "你只剩 X 次机会了"
- ✅ **只在耗尽时** 才通知一次，让它总结

---

## execute_code 的特殊处理：Refund

`execute_code` 工具内部可能有大量循环工具调用，但这些**不计入**主 iteration 预算：

```python
# execute_code 的循环内每次工具调用后：
iteration_budget.refund()  # 退回一次
```

这样 `execute_code` 内部的循环不会耗尽 Agent 的 90 次预算。

---

## 常见场景

### 场景 1：复杂重构（经常在 40-70 次后耗尽）

```
[1/90] 分析项目结构
[2/90] 读取第一个文件
[3/90] 读取第二个文件
...
[50/90] 修改代码
[51/90] 修改代码
...
[90/90] ⚠️ Iteration budget exhausted
```

**原因**：多文件修改 + Git 操作 + 飞书汇报，加起来超过 90。

**解决**：用 `delegate_task` 把任务拆分给子 Agent，每个子 Agent 有自己的 50 次预算。

### 场景 2：多次 search_files + read_file 循环

```
[1/90] search_files 找某个模式
[2/90] read_file 读第一个结果
[3/90] search_files 找另一个模式
[4/90] read_file 读另一个结果
...
```

**原因**：每个工具调用都消耗 1 次，复杂任务轻松超过 90。

### 场景 3：子 Agent 任务

```
主 Agent [23/90]
    │
    ├─► 子 Agent A [45/50]  → 可能自己耗尽
    │
    └─► 子 Agent B [50/50]  → 强制终止
```

子 Agent 有独立的 `IterationBudget`（默认 50），不会消耗主 Agent 的预算。

---

## 如何避免 Iteration Budget 耗尽？

### 1. 提高上限（简单粗暴）

```yaml
agent:
  max_turns: 200  # 适合复杂任务
```

### 2. 拆分任务（推荐）

用 `delegate_task` 拆分成多个子任务：

```
复杂任务
    ├─► 子任务 A（子 Agent 1，50 次）
    ├─► 子任务 B（子 Agent 2，50 次）
    └─► 子任务 C（子 Agent 3，50 次）
```

### 3. 优化工具调用

- 批量操作：一次 `search_files` 找多个模式，而不是多次搜索
- 减少 read_file：读大文件时用 offset/limit
- 避免循环：用 patch 的 `replace_all=True`

### 4. 监控进度

如果看到 `70/90`、`80/90`，说明任务快耗尽了，可以主动：
- 减少不必要的工具调用
- 让模型先总结当前进度
- 准备"未完成但有进度"的结果交付

---

## 配置参数

| 参数 | 默认值 | 位置 | 说明 |
|------|--------|------|------|
| `max_iterations` | 90 | AIAgent.__init__ | 主 Agent 上限 |
| `delegation.max_iterations` | 50 | delegate_task tool | 子 Agent 上限 |
| `_budget_grace_call` | False | 内部状态 | Grace period 标志 |

---

## 总结

| 问题 | 答案 |
|------|------|
| **是什么** | 防止无限循环的安全机制 |
| **默认上限** | 90 次 tool call |
| **耗尽后** | Grace period：给一次机会总结，然后强制终止 |
| **为什么会"挂"** | 任务复杂超过 90 次，或子 Agent 超过 50 次 |
| **如何避免** | 提高上限 / 拆分任务 / 优化工具调用 |
