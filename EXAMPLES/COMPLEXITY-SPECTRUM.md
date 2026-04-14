# Hermes Skills 复杂度光谱实例

> 从极简到项目级，不同复杂度的真实 Skill 案例

## 复杂度光谱总览

```
极简 ──────────────────────────────────────────────────── 项目级
  │                                                          │
DESCRIPTION.md                                    完整 TypeScript/Python 项目
SKILL.md                                          SKILL.md + 多语言 README
                                                  scripts/ + node_modules + tests
```

---

## Level 1：极简 (1 文件)

### 示例：`diagramming/`

```
~/.hermes/skills/diagramming/
└── DESCRIPTION.md          # 仅 1 个文件，159 字节
```

**DESCRIPTION.md 内容：**
```yaml
---
description: Diagram creation skills for generating visual diagrams, flowcharts, architecture diagrams, and illustrations using tools like Excalidraw.
---
```

**分析：**
| 维度 | 说明 |
|------|------|
| 文件数 | 1 |
| 核心内容 | 1 行 YAML 描述 |
| 脚本 | 无 |
| 触发条件 | 用户请求生成图表时 Hermes 自动联想 |

**适用场景：** 只做备忘录，不需要详细说明的场景。

---

## Level 2：标准 (SKILL.md 文档)

### 示例：`hermes-principle/`

```
~/.hermes/skills/hermes-principle/
└── SKILL.md               # 2144 字节
```

**SKILL.md 核心结构：**

```yaml
---
name: hermes-principle
description: "Hermes Agent 原理沉淀与知识库构建..."
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [hermes, 原理, 知识库, 沉淀]
    related_skills: [hermes-agent, soul-force]
---

# hermes-principle

## 概述
## 触发条件
## 仓库结构
## 沉淀流程
## 提交规范
```

**分析：**
| 维度 | 说明 |
|------|------|
| 文件数 | 1 |
| 内容 | 完整的 Markdown 文档 |
| 结构 | YAML frontmatter + 多级标题 + 代码块 |
| 特点 | 描述清晰，有触发条件，有流程，有提交规范 |

---

### 示例：`systematic-debugging/`

```
~/.hermes/skills/software-development/systematic-debugging/
└── SKILL.md               # 10,538 字节
```

**结构：**
```markdown
# Systematic Debugging

## Overview
## The Iron Law
## When to Use
## The Four Phases
  - Phase 1: Root Cause Investigation
  - Phase 2: Pattern Analysis
  - Phase 3: Hypothesis and Testing
  - Phase 4: Implementation
## Red Flags
## Common Rationalizations
## Quick Reference
## Hermes Agent Integration
## Real-World Impact
```

**分析：**
| 维度 | 说明 |
|------|------|
| 文件数 | 1 |
| 行数 | ~350 行 |
| 内容深度 | 包含流程图、检查清单、代码示例、工具调用方式 |
| 特点 | 工作流完整，细节丰富，可直接指导行动 |

---

## Level 3：完整 (多文件 + 脚本)

### 示例：`soul-force/`

```
~/.hermes/skills/openclaw-imports/soul-force/
├── SKILL.md                # 主文档
├── _meta.json              # 机器可读配置
├── README.md / README.zh-CN.md
├── CHANGELOG.md
├── TODO.md
├── CONTRIBUTING.md
├── .git/                   # 独立 Git 仓库
├── references/
│   ├── ARCHITECTURE.md
│   ├── help-en.md
│   └── help-zh.md
├── scripts/
│   └── soulforge.py        # 核心脚本
├── soulforge/              # Python 包
│   ├── __init__.py
│   ├── analyzer.py
│   ├── config.py
│   ├── evolver.py
│   ├── memory_reader.py
│   └── schema.py
└── tests/
    └── test_soulforge.py
```

**_meta.json 结构：**
```json
{
  "name": "soul-force",
  "version": "1.0.0",
  "category": "openclaw-imports",
  "hermes": {
    "tags": ["memory", "evolution", "openclaw"],
    "related_skills": ["hawk-bridge", "tangseng-brain"]
  }
}
```

**分析：**
| 维度 | 说明 |
|------|------|
| 文件数 | ~20+ |
| 编程语言 | Python (soulforge 包) |
| 文档 | 多语言 (EN, ZH-CN, DE, FR...) |
| 版本控制 | 独立 Git 仓库 |
| 测试 | 有单元测试 |

---

## Level 4：项目级 (完整可运行项目)

### 示例：`hawk-bridge/`

```
~/.hermes/skills/openclaw-imports/hawk-bridge/
├── SKILL.md
├── _meta.json
├── README.md / README.*.md          # 10 种语言
├── TODO.md
├── CHANGELOG.md
├── manifest.json                    # ClawHub 清单
├── openclaw.plugin.json             # OpenClaw 插件定义
├── package.json / package-lock.json # npm 依赖
├── tsconfig.json
├── install.sh                       # 安装脚本
├── .env.example                     # 环境变量模板
├── .gitignore
├── .clawhub/                        # ClawHub 元数据
│   └── origin.json
├── src/                             # TypeScript 源码
│   ├── index.ts
│   ├── config.ts
│   ├── constants.ts
│   ├── embeddings.ts
│   ├── lancedb.ts
│   ├── retriever.ts
│   ├── seed.ts
│   ├── config/
│   │   ├── defaults.ts
│   │   ├── env.ts
│   │   ├── index.ts
│   │   ├── legacy.ts
│   │   └── yaml.ts
│   ├── hooks/
│   │   ├── hawk-capture/handler.ts
│   │   ├── hawk-decay/handler.ts
│   │   ├── hawk-dream/handler.ts
│   │   └── hawk-recall/handler.ts
│   └── i18n/
│       ├── en.json
│       ├── zh.json
│       └── index.ts
├── dist/                            # 编译输出
│   ├── index.js
│   ├── seed.js
│   ├── cli/
│   │   ├── decay.js
│   │   ├── read-source.js
│   │   ├── verify.js
│   │   └── write.js
│   └── hooks/
│       ├── hawk-capture/
│       ├── hawk-decay/
│       ├── hawk-dream/
│       └── hawk-recall/
└── python/                          # Python 绑定
    └── hawk_memory/__init__.py
```

**openclaw.plugin.json：**
```json
{
  "name": "hawk-bridge",
  "version": "1.2.0",
  "hooks": [
    {
      "name": "hawk-capture",
      "method": "postReply",
      "handler": "./dist/hooks/hawk-capture/hawk-capture.js"
    },
    {
      "name": "hawk-recall",
      "method": "preContext",
      "handler": "./dist/hooks/hawk-recall/hawk-recall.js"
    }
  ]
}
```

**分析：**
| 维度 | 说明 |
|------|------|
| 文件数 | 70+ |
| 编程语言 | TypeScript + Python |
| 构建工具 | TypeScript 编译 |
| 依赖管理 | npm (TypeScript) + pip (Python) |
| 多语言文档 | EN, ZH-CN, ZH-TW, DE, ES, FR, IT, JA, KO, PT-BR, RU |
| 插件系统 | 完整 OpenClaw Hook 插件 |
| 发布 | ClawHub (clawhub.json) |

---

## 复杂度选择指南

| 需求 | 推荐级别 | 理由 |
|------|---------|------|
| 备忘录/提示 | Level 1 | 极简，1 行描述即可 |
| 工作流指导 | Level 2 | 完整 Markdown 文档 |
| 需要脚本执行 | Level 3 | scripts/ + Python/Shell |
| 完整产品/插件 | Level 4 | TypeScript + 构建 + 测试 + 多语言 |

### 决策树

```
需要封装逻辑吗？
  ├── 否 → Level 1 (DESCRIPTION.md)
  │
  └── 是 → 需要可执行代码吗？
              ├── 否 → Level 2 (SKILL.md)
              │
              └── 是 → 需要多语言/完整项目吗？
                          ├── 否 → Level 3 (scripts/ + Python)
                          │
                          └── 是 → Level 4 (TypeScript + 构建 + 多语言)
```

---

## 真实案例对比

| 项目 | 级别 | 主要文件 | 代码量 |
|------|------|---------|--------|
| `diagramming` | L1 | DESCRIPTION.md | 159 字节 |
| `hermes-principle` | L2 | SKILL.md | 2 KB |
| `systematic-debugging` | L2 | SKILL.md | 10 KB |
| `soul-force` | L3 | SKILL.md + soulforge/*.py | ~30 KB |
| `hawk-bridge` | L4 | SKILL.md + src/*.ts + dist/ | ~200 KB+ |

---

## 相关文档

- [SKILLS.md](./SKILLS.md) - 创建机制详解
- [SKILLS-ARCHITECTURE.md](./SKILLS-ARCHITECTURE.md) - 系统架构
- [hermes-principle skill](../openclaw-imports/hermes-agent/) - 官方指南
