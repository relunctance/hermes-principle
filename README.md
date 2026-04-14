# Hermes 原理与实践

> Hermes Agent 架构、Skills 机制、原理细节的沉淀仓库。

## 目录结构

```
hermes-principle/
├── SKILLS.md              # Hermes Skills 创建机制与规范
├── ARCHITECTURE.md        # Hermes 系统架构
├── CONVERSATIONS/         # 典型对话案例
│   └── 2026-04-14/       # 今天的对话记录
└── EXAMPLES/              # 实践示例
```

## 关于 Hermes

Hermes 是基于大模型的 AI Agent 系统，具备：
- **Skills 系统** — 可扩展的能力模块
- **MCP 协议** — 工具发现与调用
- **记忆系统** — 持久化上下文
- **定时任务** — Cron 调度
- **子 Agent 委托** — Delegate tasks

## Skills 创建机制

详见 [SKILLS.md](./SKILLS.md)

## License

MIT
