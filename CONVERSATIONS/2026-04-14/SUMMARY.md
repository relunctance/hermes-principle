# 2026-04-14 对话记录

## 对话背景

- 用户：其林
- 时间：2026-04-14 12:36
- 平台：Feishu (DM)
- 主题：Hermes Skills 机制探讨 + 创建公开仓库

## 关键讨论点

### 1. OpenClaw 回退

用户升级到 `2026.4.12` 后遇到问题，决定回退到 `2026.4.11`。

**命令：**
```bash
npm install -g openclaw@2026.4.11
openclaw gateway restart
```

### 2. Hermes 与 OpenClaw Skills 互通性

- Hermes Skills：`~/.hermes/skills/`
- OpenClaw Skills：`~/.openclaw/agents/*/workspace/skills/`

**结论：单向桥接**
- Hermes ✅ 可调用 `openclaw-imports/` 下的 skills
- OpenClaw ❌ 无法使用 Hermes skills（格式不兼容）

### 3. Hermes 可用 Skills 总览

共 **108 个 skills**，分类：
- openclaw-imports (~35): soul-force, hawk-bridge, auto-evolve 等
- mlops (~22): vllm, llama-cpp, gguf, peft, axolotl, unsloth
- autonomous-ai-agents (4): claude-code, codex, opencode, hermes-agent
- creative (7): p5js, manim, excalidraw, ascii-art
- github (6): pr-workflow, code-review, repo-management
- software-development (5): systematic-debugging, plan, writing-plans
- 其他：media, productivity, research, gaming, smart-home, social-media

### 4. Skill 创建机制

**创建时机：**
- 复杂任务成功（5+ 工具调用）
- 发现非平凡的工作流
- 用户纠正方法后
- 棘手的 bug 及解决方案
- 跨会话可能需要

**创建流程：**
1. 评估任务是否值得存档
2. 编写 SKILL.md
3. 组织 scripts/references/tests
4. Hermes 自动发现（目录结构）
5. 验证 skill 可加载

### 5. 创建公开仓库

用户希望创建公开仓库 `hermes-principle`，存放：
- Hermes 原理
- Skills 创建机制
- 日常沟通的细节和原理

**仓库地址：** github.com/relunctance/hermes-principle

**仓库结构：**
```
hermes-principle/
├── README.md
├── SKILLS.md              # Skills 创建机制
├── ARCHITECTURE.md        # (待补充)
├── CONVERSATIONS/
│   └── 2026-04-14/
└── EXAMPLES/
```

## 后续计划

- [ ] 补充 ARCHITECTURE.md（Hermes 系统架构）
- [ ] 创建 EXAMPLES/ 目录
- [ ] 创建 hermes-principle skill（已完成）
