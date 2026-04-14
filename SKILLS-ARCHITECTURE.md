# Hermes Skills 系统架构与原理

> 基于源码分析得出的 Hermes Skills 工作机制

## 目录结构

```
~/.hermes/skills/
├── .bundled_manifest          # 内置 skills 的哈希清单
├── [category]/                # 按领域分类的 skills
│   ├── [skill-name]/
│   │   ├── SKILL.md          # 核心元数据文件（必需）
│   │   ├── _meta.json        # 机器可读配置
│   │   ├── scripts/          # 可执行脚本
│   │   ├── references/       # 参考文档
│   │   ├── templates/        # 模板文件
│   │   └── tests/            # 测试
│   └── another-skill/
└── openclaw-imports/         # 特殊目录：桥接 OpenClaw skills
```

## 核心文件

### SKILL.md（必需）

```yaml
---
name: skill-name
description: "一句话描述 skill 用途"
version: 1.0.0
author: 作者
license: MIT
metadata:
  hermes:
    tags: [标签1, 标签2]
    related_skills: [相关skill]
  openclaw:              # 可选：OpenClaw 兼容配置
    requires:
      bins: [python3]
---

# Skill 名称

## 概述
...

## 触发条件
...

## 使用方法
...
```

### _meta.json（可选）

```json
{
  "name": "skill-name",
  "version": "1.0.0",
  "category": "category-name",
  "hermes": {
    "tags": ["tag1"],
    "related_skills": ["other-skill"]
  }
}
```

## Skill 发现机制

Hermes 启动时扫描 `~/.hermes/skills/` 目录：

1. 扫描所有一级子目录作为 category
2. 扫描所有二级子目录作为 skill name
3. 读取 `SKILL.md` 作为主要元数据
4. 读取 `_meta.json` 作为补充配置
5. 生成 `.bundled_manifest` 缓存

```
hermes skills list
```

这个命令展示所有已安装的 skills，来源分为：

| Source | 说明 |
|--------|------|
| `builtin` | Hermes 内置 skills（74个） |
| `local` | 本地 `~/.hermes/skills/` 目录（35个） |
| `hub` | 从 skills.sh 等 registry 安装 |

## Skill 的分类（按 source）

### 1. builtin skills

位于 Hermes 二进制文件中，加载到 `~/.hermes/skills/` 时不带完整目录结构。

查看：
```bash
hermes skills list | grep builtin
```

### 2. local skills

直接放在 `~/.hermes/skills/[category]/[skill-name]/` 下的 skill。

- `openclaw-imports/` 目录：桥接 OpenClaw 格式的 skills（单向）
- 其他目录：Hermes 原生格式

### 3. hub skills

从 remote registry 安装的 skills。

```bash
hermes skills search <keyword>    # 搜索
hermes skills install <name>       # 安装
hermes skills check               # 检查更新
```

## Skill 的内部结构复杂度

| 复杂度 | 示例 | 说明 |
|--------|------|------|
| **简单** | `diagramming/` | 只有 `DESCRIPTION.md`，无 SKILL.md |
| **标准** | `systematic-debugging/` | 只有 `SKILL.md` |
| **复杂** | `hawk-bridge/` | SKILL.md + README多语言 + scripts + tests + node_modules |

复杂 skill 可以包含：
- 完整的 Python/Node.js 项目
- 依赖管理（package.json, requirements.txt）
- 安装脚本（install.sh）
- 多语言文档
- 测试套件

## Skill 的用途分类

| 用途 | 示例 | 说明 |
|------|------|------|
| **工作流封装** | `systematic-debugging` | 把复杂流程模板化 |
| **工具包装** | `github-pr-workflow` | 封装 CLI 工具的使用 |
| **知识库** | `project-standard` | 项目规范和检查清单 |
| **跨系统桥接** | `hawk-bridge` | OpenClaw Hook ↔ Hermes |
| **系统扩展** | `hermes-agent` | Hermes 自身文档 |

## Skill 生命周期

```
创建 → 发现 → 加载 → 使用 → （可选：更新）
  │       │       │       │
  │       │       │       └── 执行任务时调用
  │       │       └── skill_view(name) 加载到上下文
  │       └── hermes skills list 显示
  └── skill_manage 创建或 skill_view 加载
```

## 触发 skill 的方式

1. **自动加载**：Hermes 根据讨论内容自动匹配相关 skill
2. **手动加载**：`skill_view(name)` 调用
3. **预加载**：启动时 `hermes --skills skill1,skill2`

## Skill 与 Tool 的区别

| 维度 | Skill | Tool |
|------|-------|------|
| 本质 | 知识模板/工作流 | 可执行函数 |
| 触发 | 对话上下文匹配 | tool_call |
| 内容 | Markdown 文档 | 代码函数 |
| 持久化 | 文件系统 | 代码注册 |
| 用途 | 指导怎么做 | 执行具体操作 |

## 创建建议

1. **简单优先**：先用 `SKILL.md` 满足需求
2. **脚本独立**：复杂逻辑放 `scripts/`，SKILL.md 只做编排
3. **描述清晰**：明确 `触发条件`，让 Hermes 知道什么时候用
4. **版本控制**：记录 `version` 和 `CHANGELOG.md`
5. **测试友好**：包含 `tests/` 目录

## 相关文档

- [SKILLS.md](./SKILLS.md) - 创建机制详解
- [官方文档](https://hermes-agent.nousresearch.com/docs/)
- [hermes-agent skill](../openclaw-imports/hermes-agent/) - 官方指南
