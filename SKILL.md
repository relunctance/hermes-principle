---
name: hermes-principle
description: "Hermes Agent 原理沉淀与知识库构建。每次沟通中关于 Hermes 架构、Skills 机制、工具使用等原理性内容时，自动沉淀到 ~/repos/hermes-principle 仓库中。触发时机：讨论 Hermes 原理、Skill 创建、系统架构、工具机制时。"
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [hermes, 原理, 知识库, 沉淀, 学习]
    related_skills: [hermes-agent, soul-force]
---

# hermes-principle

## 概述

Hermes Agent 原理与实践的沉淀仓库。每次沟通中涉及 Hermes 系统架构、Skills 创建机制、工具原理等知识性内容时，自动归档到 `~/repos/hermes-principle/`。

## 文档结构

| 文件 | 说明 |
|------|------|
| `README.md` | 仓库首页 |
| `SKILLS.md` | Hermes Skills 创建机制与规范 |
| `ARCHITECTURE.md` | Hermes 系统架构详解 |
| `HOOKS.md` | Hook 机制原理与实现 |
| `COMMUNICATION.md` | Agent 间通信机制 |
| `GATEWAY-INTERNAL.md` | Gateway 内部原理 |
| `SKILLS-ARCHITECTURE.md` | Skills 系统架构 |
| `CONVERSATIONS/` | 对话记录归档（按日期） |
| `EXAMPLES/` | 实践示例 |

## 触发条件

当讨论以下主题时自动触发：
- Hermes 系统架构
- Skills 创建机制/规范
- 工具调用原理
- 记忆系统机制
- MCP 协议原理
- 子 Agent 委托机制
- Cron 调度机制
- 与 OpenClaw 的关系/区别

## 沉淀流程

1. **识别原理性讨论** — 当发现值得归档的知识时
2. **判断归属** — 属于哪个文档（SKILLS.md/ARCHITECTURE.md/对话记录）
3. **更新文档** — 使用 patch 追加内容
4. **Git 提交** — 注明更新的主题

## 提交规范

```
docs: 补充 XXX 原理说明
feat: 新增对话记录 YYYY-MM-DD
fix: 修正 ARCHITECTURE.md 中关于 XXX 的描述
```

## 对话记录格式

每个对话记录包含：
- 对话背景（时间、平台、主题）
- 关键讨论点（带代码示例）
- 后续计划
