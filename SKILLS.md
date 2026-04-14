# Hermes Skills 创建机制

## 什么是 Skill

Skill 是 Hermes Agent 的**能力模块化单元**。每个 Skill 包含：
- 执行逻辑（代码/脚本）
- 元数据（描述、标签、作者）
- 使用场景说明
- 触发条件

本质上：**把「如何做某件事」的经验封装成可复用的模板。**

## Skill 目录结构

```
~/.hermes/skills/
├── category-name/           # 按领域分类
│   ├── skill-name/          # 单个 skill
│   │   ├── SKILL.md         # 核心元数据文件
│   │   ├── _meta.json       # 机器可读的配置
│   │   ├── scripts/         # 可执行脚本
│   │   ├── references/      # 参考文档
│   │   ├── templates/       # 模板文件
│   │   └── tests/           # 测试
│   └── another-skill/
└── openclaw-imports/        # 特殊目录：桥接 OpenClaw skills
    └── soul-force/
        ├── SKILL.md
        ├── CHANGELOG.md
        ├── README.md
        ├── TODO.md
        ├── _meta.json
        ├── scripts/
        ├── soulforge/
        ├── references/
        └── tests/
```

## SKILL.md 结构

```yaml
---
name: skill-name
description: "一句话描述 skill 用途"
version: 1.0.0
author: 作者信息
license: MIT
metadata:
  hermes:
    tags: [标签1, 标签2]
    related_skills: [相关skill]
---

# Skill 名称

## 概述
详细的技能说明...

## 触发条件
什么时候应该使用这个 skill...

## 使用方法
具体的调用方式...

## 示例
使用示例...
```

## _meta.json 结构

```json
{
  "name": "skill-name",
  "version": "1.0.0",
  "category": "category-name",
  "hermes": {
    "tags": ["tag1", "tag2"],
    "related_skills": ["other-skill"]
  },
  "openclaw": {
    "requires": {
      "bins": ["python3"],
      "env": {}
    }
  }
}
```

## 创建时机

| 触发条件 | 说明 |
|---------|------|
| **复杂任务成功** | 5+ 工具调用解决的问题 |
| **发现新工作流** | 找到了更好的方法 |
| **用户纠正** | 纠正后用新方法解决了问题 |
| **非平凡的调试** | 棘手的 bug 及解决方案 |
| **跨会话需要** | 未来可能再次需要 |

## 创建流程

1. **评估** — 任务是否值得存档
2. **编写 SKILL.md** — 描述、触发条件、使用方法
3. **组织文件** — scripts/references/tests
4. **注册** — Hermes 自动发现（通过目录结构）
5. **验证** — 测试 skill 是否可加载

## Skill 发现机制

Hermes 启动时扫描 `~/.hermes/skills/` 目录：
- 每个子目录被视为一个 skill
- `SKILL.md` 是必需的主文件
- `skill_view(name)` 加载 skill 内容

## OpenClaw 桥接

`openclaw-imports/` 目录是单向桥接：
- Hermes 可以调用 OpenClaw 格式的 skills
- OpenClaw Agent 无法使用 Hermes skills
- 格式不兼容，需要手动迁移

## 最佳实践

1. **简洁描述** — 一句话说清用途
2. **明确触发** — 什么情况下用这个 skill
3. **包含示例** — 至少一个可运行的例子
4. **记录依赖** — 需要的工具、环境变量
5. **版本控制** — 记录变更历史

## 相关文档

- [Hermes Agent Skill](../openclaw-imports/hermes-agent/) - 官方指南
- [Awesome OpenClaw Skills](../awesome-openclaw-skills/) - 社区 skill 列表
