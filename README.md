# Hermes 原理与实践

> Hermes Agent 架构、Skills 机制、原理细节的沉淀仓库。

---

## 文档索引

| 文件 | 说明 |
|------|------|
| **[ARCHITECTURE.md](./ARCHITECTURE.md)** | Hermes 系统整体架构：核心组件（Gateway/AIAgent/Cron/MCP）、消息流转、目录结构、模型交互。是理解 Hermes 的起点。 |
| **[COMMUNICATION.md](./COMMUNICATION.md)** | Gateway 与各平台（飞书/Telegram/Discord 等）的通信机制：WebSocket/Webhook 接收、MessageEvent 标准化、响应回传流程。 |
| **[GATEWAY-INTERNAL.md](./GATEWAY-INTERNAL.md)** | Gateway 内部原理：GatewayRunner 核心类、启动流程、Session 管理、消息处理（`_handle_message_with_agent`）、并发控制、平台适配。 |
| **[HOOKS.md](./HOOKS.md)** | Hook 事件系统：全部 12 种事件（`gateway:startup`、`session:start/end`、`agent:start/step/end`、`command:<name>` 等）的触发时机、context 参数、源码位置。hawk-bridge 的 `/recall`（agent:start）和 `/capture`（agent:end）挂载在此。 |
| **[ITERATION-BUDGET.md](./ITERATION-BUDGET.md)** | 迭代预算机制：防止 Agent 进入无限循环的安全阀。IterationBudget 类、90 次上限原理、超时表现、预算耗尽后的行为。 |
| **[MEMORY-SYSTEM.md](./MEMORY-SYSTEM.md)** | Hermes vs OpenClaw soul-force 记忆系统对比：MD 文件体系（SOUL.md/MEMORY.md/USER.md）、加载机制、与 LanceDB 向量进化的核心差异。 |
| **[MESSAGE-QUEUE-AND-BURST.md](./MESSAGE-QUEUE-AND-BURST.md)** | 消息队列与并发处理：飞书连发 5 条消息的覆盖机制（pending slot）、interrupt 信号、burst 场景下的文本覆盖 + 图片合并行为。 |
| **[SESSION-MANAGEMENT.md](./SESSION-MANAGEMENT.md)** | Session 三层 ID 体系（session_key/session_id/message_id）、session_key 构造规则（DM/Group/Thread 隔离）、reset 策略、SQLite 存储。 |
| **[SKILL.md](./SKILL.md)** | Skill 的定义与基本结构：SKILL.md 的 YAML frontmatter + Markdown 正文格式。 |
| **[SKILLS-ARCHITECTURE.md](./SKILLS-ARCHITECTURE.md)** | Skills 系统架构：Skill 加载（`~/.hermes/skills/`）、slash command 注册与注入（作为 user message 而非 system prompt）、CommandDef 规范。 |
| **[SKILLS-AUTO-CREATE.md](./SKILLS-AUTO-CREATE.md)** | Skill 自动创建机制：**无后台调度器**，纯由 LLM 主动决策调用 `skill_manage` 工具。触发条件、分析器、生成器、验证器的四层流程。 |
| **[TASK-DECOMPOSITION.md](./TASK-DECOMPOSITION.md)** | 任务拆解能力对比：Hermes 的四大强制机制（Skills Mandatory Rule、Prerequisite Checks、Plan Mode）如何强制模型按正确方式工作，而非依赖模型自身能力。 |
| **[TOOLS-SYSTEM.md](./TOOLS-SYSTEM.md)** | 工具系统：54 个内置工具（terminal/file/web/delegate/mcp 等）、Tool Registry 注册机制、ToolCall 分发、`environments/` 执行后端（local/docker/ssh/modal）。 |
| **[hermes-cron-architecture.md](./hermes-cron-architecture.md)** | Cron 调度架构：tick 主循环（60s）、run_job 执行（独立 AIAgent，platform="cron"）、deliver 投递。**核心结论：Cron 不过 hawk-bridge，不进 LanceDB**。 |

---

## 目录结构

```
hermes-principle/
├── *.md                    # 各模块原理文档（见上表）
├── CONVERSATIONS/          # 典型对话案例
│   └── 2026-04-14/        # 某一天的对话记录
└── EXAMPLES/               # 实践示例
```

---

## 关于 Hermes

Hermes 是基于大模型的 AI Agent 系统，具备：
- **Skills 系统** — 可扩展的能力模块
- **MCP 协议** — 工具发现与调用
- **记忆系统** — 持久化上下文
- **定时任务** — Cron 调度
- **子 Agent 委托** — Delegate tasks

---

## License

MIT
