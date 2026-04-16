# Hermes vs OpenClaw：任务拆解能力对比分析

## 先说结论

**Hermes 任务拆解更优秀，不是因为它更"聪明"，而是因为它有更完善的工程机制强制模型按正确方式工作。**

---

## 核心差异

### OpenClaw：依赖模型自身能力

```
用户任务 → 模型自主拆解 → 执行
```

OpenClaw 的 `task-decomposer` skill 是可选的——模型**可以**用，也可以不用。如果任务描述足够清晰，模型可能直接跳到执行，漏掉关键步骤。

### Hermes：强制性的结构化流程

```
用户任务 → [强制 Skills Scan] → [强制 Prerequisite Checks] → [可选 Plan Mode] → 拆解执行
```

Hermes 在 System Prompt 里注入了**强制性指令**，要求模型在行动前必须完成某些步骤。

---

## Hermes 的四大强制机制

### 1. Skills Mandatory Rule（Skills 强制扫描）

**源码：** `prompt_builder.py` 第 768-790 行

```
## Skills (mandatory)
Before replying, scan the skills below. If a skill matches or is even partially
relevant to your task, you MUST load it with skill_view(name) and follow its
instructions.
```

**效果：** 每次回复前，模型必须检查是否有适用的 Skills。如果任务涉及 debug → 必须加载 `systematic-debugging`；如果需要规划 → 必须加载 `plan`；如果复杂多步 → 必须加载 `subagent-driven-development`。

**关键创新：** 这是**强制性的**，不是建议。模型如果不扫描就行动，会被 System Prompt 明确指出"你没有遵守 Skills mandatory rule"。

---

### 2. Prerequisite Checks（前置检查指令块）

**源码：** `prompt_builder.py` 第 242-247 行

```
<prerequisite_checks>
- Before taking an action, check whether prerequisite discovery, lookup, or
  context-gathering steps are needed.
- Do not skip prerequisite steps just because the final action seems obvious.
- If a task depends on output from a prior step, resolve that dependency first.
</prerequisite_checks>
```

**效果：** 模型被强制要求在行动前检查"我有没有遗漏前置步骤"。这直接防止了"看到任务就动手"的冲动行为。

---

### 3. Verification（验证指令块）

**源码：** `prompt_builder.py` 第 249-256 行

```
<verification>
Before finalizing your response:
- Correctness: does the output satisfy every stated requirement?
- Grounding: are factual claims backed by tool outputs or provided context?
- Formatting: does the output match the requested format or schema?
- Safety: if the next step has side effects (file writes, commands, API calls),
  confirm scope before executing.
</verification>
```

**效果：** 模型在交付结果前必须自检四个维度，减少了"做完了但没做对"的返工。

---

### 4. Missing Context（上下文缺失处理）

**源码：** `prompt_builder.py` 第 258-264 行

```
<missing_context>
- If required context is missing, do NOT guess or hallucinate an answer.
- Use the appropriate lookup tool when missing information is retrievable.
- Ask a clarifying question only when the information cannot be retrieved by tools.
```

**效果：** 防止模型在信息不足时瞎猜，强制它先查文件、搜代码。

---

## 对比：OpenClaw 的 task-decomposer skill

OpenClaw 有 `task-decomposer` skill，功能类似：

| 能力 | Hermes | OpenClaw task-decomposer |
|------|--------|-------------------------|
| 识别子组件 | ✅ (via mandatory skill scan) | ✅ |
| 确定依赖关系 | ✅ (via prerequisite_checks) | ✅ |
| 估算工作量 | ❌ 无 | ❌ |
| 创建执行计划 | ✅ (via plan skill) | ✅ |
| **执行** | ✅ **并行子 Agent** | ⚠️ 串行 |

**关键差异：** Hermes 有 `delegate_task` 的并行执行能力，OpenClaw 的 task-decomposer 只负责拆解，不负责并行执行。

---

## Hermes 任务拆解的完整流程

```
1. 用户输入复杂任务
         │
         ▼
2. Skills Mandatory Scan
   → 检测到复杂任务，加载 plan skill
   → 检测到需要调试，加载 systematic-debugging
   → 检测到多子任务，加载 subagent-driven-development
         │
         ▼
3. Plan Mode（可选，由 plan skill 触发）
   → 将任务写入 .hermes/plans/YYYY-MM-DD_HHMMSS-<slug>.md
   → 不执行，只规划
         │
         ▼
4. Prerequisite Checks
   → 列出所有需要的前置步骤
   → 确定依赖关系
         │
         ▼
5. 拆解 + 并行执行
   → delegate_task 分配给多个子 Agent（最多 3 个并行）
   → 每个子 Agent 有独立 iteration budget（默认 50 次）
         │
         ▼
6. Verification
   → 每个子 Agent 完成后自检
   → 主 Agent 汇总验证
         │
         ▼
7. 结果交付
```

---

## 拆解能力的三个层次

### L1：口头拆解（大多数 Agent）

```
"好的，我来拆解成三步：1. 读文件，2. 修改，3. 提交"
```

只说不做，模型可能跳过第一步直接做第三步。

### L2：结构化拆解（OpenClaw task-decomposer）

```
任务：A
  ├─ 子任务 A1（依赖：无）
  ├─ 子任务 A2（依赖：A1）
  └─ 子任务 A3（依赖：A2）
```

有结构，但可能不执行或串行执行。

### L3：结构化拆解 + 强制执行（Hermes）

```
任务：A
  ├─ 子任务 A1 → Agent 1（并行，50 次迭代）
  ├─ 子任务 A2 → Agent 2（并行，50 次迭代）
  └─ 子任务 A3 → Agent 3（并行，50 次迭代）
         │
         ▼
  主 Agent 收集结果，Verification 自检后交付
```

**Hermes 的优势：**
- 并行执行，不浪费时间
- 每个子 Agent 有独立 iteration budget，不会互相干扰
- 主 Agent 负责协调，不被工具调用消耗

---

## 为什么会感觉 Hermes 更优秀？

| 感受 | 原因 |
|------|------|
| "拆解得很全面" | Skills mandatory + prerequisite_checks 强制检查 |
| "不会漏掉步骤" | 强制 Verification |
| "执行很快" | 并行子 Agent |
| "不会瞎猜" | missing_context 强制查文件 |
| "任务完成度高" | 90 次 iteration 允许充分执行 |

**本质：Hermes 不是让模型更聪明，而是让模型按正确的方式工作。**

---

## OpenClaw 可以改进的方向

1. **将 task-decomposer 从可选改为强制** — 在 System Prompt 里加入类似 `<task_decomposition>` 的强制性指令
2. **增加并行执行机制** — 给 OpenClaw Agent 增加 `parallel_tasks` 执行模式
3. **增加 Verification 指令块** — 让模型在交付前自检
4. **增加 subagent 支持** — 让 OpenClaw 也能并行执行多个子任务

---

## 总结

| | Hermes | OpenClaw |
|---|---|---|
| **拆解机制** | 强制性 Skills + Prerequisite Checks | 可选 skill |
| **执行模式** | 并行子 Agent（最多 3 个） | 串行执行 |
| **Iteration Budget** | 90 次主 + 50 次子 | 不明确 |
| **Verification** | 强制性自检 | 无 |
| **Plan Mode** | 内置 skill | 可选 skill |
| **missing_context** | 强制性查文件 | 无 |

Hermes 优秀的核心原因：**它用 System Prompt 的强制性指令 + 工程化的并行执行机制，弥补了模型自身的"冲动行事"倾向。**
