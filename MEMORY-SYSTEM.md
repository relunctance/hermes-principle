# Hermes 记忆系统 vs OpenClaw soul-force

## 核心区别

| 维度 | Hermes | OpenClaw soul-force |
|------|--------|-------------------|
| **记忆文件** | `~/.hermes/memories/MEMORY.md` + `USER.md` + `SOUL.md` | `~/.openclaw/workspace/MEMORY.md` + `USER.md` + `SOUL.md` |
| **存储形式** | Markdown 文件（纯文本） | Markdown + LanceDB（向量数据库） |
| **进化机制** | **无自动进化**——文件手动更新 | **全自动进化**——Hook 驱动 L1-L6 巡检体系 |
| **记忆来源** | Agent 主动调用 `memory` tool | Hook 事件自动触发 |
| **检索方式** | 文件全文 + session_search | FTS5 向量检索 |
| **Session 持久化** | SQLite sessions 表 | 纯内存 + 可选 SQLite |

---

## Hermes 的 MD 文件体系

### 文件位置

```
~/.hermes/
├── SOUL.md                    # Agent 人格/身份定义
└── memories/
    ├── MEMORY.md             # Agent 的个人笔记（环境/项目/工具惯例）
    └── USER.md               # 用户画像（偏好/交流风格/工作习惯）
```

### 1. SOUL.md — Agent 人格

**位置：** `~/.hermes/SOUL.md`

**用途：** 定义 Agent 的性格和语气。每次消息都会加载，用于 Agent Identity Slot（System Prompt 的第一个部分）。

**内容示例：**
```markdown
# Hermes Agent Persona

<!--
This file defines the agent's personality and tone.
The agent will embody whatever you write here.
Edit this to customize how Hermes communicates with you.

Examples:
  - "You are a warm, playful assistant who uses kaomoji occasionally."
  - "You are a concise technical expert. No fluff, just facts."
  ...
-->
```

**加载机制：**
- 每次会话启动时加载
- 通过 `load_soul_md()` 在 `prompt_builder.py` 中读取
- 内容注入到 System Prompt 的 **Agent Identity Slot**（slot #1）
- 如果文件为空或不存在，使用默认人格

**会不会自动更新？**
- ❌ **不会自动更新**——需要人工编辑
- Agent 不会自动修改 SOUL.md

---

### 2. MEMORY.md — Agent 个人笔记

**位置：** `~/.hermes/memories/MEMORY.md`

**用途：** Agent 自己积累的笔记——环境细节、项目惯例、工具特性、学会的东西。

**内容示例（你当前的）：**
```markdown
### 核心项目（GitHub 仓库）

| 项目名 | GitHub | 定位 | 状态 |
|--------|--------|------|------|
| **auto-evolve** | relunctance/auto-evolve | 19 视角巡检引擎 | **私有** |
...

### 🔒 进程保护规则（禁止 kill）
...

### Git 提交身份：maomao <maomao@gql.ai>
...
```

**加载机制：**
- 通过 `memory_tool.py` 的 `SessionMemoryStore` 类管理
- `load_from_disk()` 读取文件，解析 `§` 分隔的条目
- **会话开始时**注入 System Prompt（作为 frozen snapshot）
- **会话中写入**直接落盘，但不更新 System Prompt（保护 prefix cache）

**会不会自动更新？**
- ✅ **会更新**——Agent 调用 `memory` tool 写入
- 但不是"进化"，只是记录（soul-force 才有真正的进化）

**与 soul-force 的区别：**
| | Hermes MEMORY.md | OpenClaw soul-force |
|---|---|---|
| 更新方式 | Agent 主动调用 `memory` tool | Hook 自动触发 |
| 更新内容 | 人工决定写什么 | 系统分析后自动生成 |
| 进化能力 | 无——只是备忘 | 有——从错误中学习 |
| 检索 | 全文扫描 | 向量 + 关键词混合 |

---

### 3. USER.md — 用户画像

**位置：** `~/.hermes/memories/USER.md`

**用途：** Agent 对用户的了解——名字、偏好、交流风格、工作习惯。

**内容示例（你当前的）：**
```markdown
_Learn about the person you're helping. Update this as you go._
§
**团队身份认知：** 其林是 gql-openclaw 团队 Owner，我是 maomao（AI Agent）
§
**What to call them:** 其林
§
**Pronouns:** _(unknown)_
§
**Timezone:** Asia/Shanghai (推测)
§
用户希望使用中文交流，请始终使用中文回复。
...
```

**会不会自动更新？**
- ✅ **会更新**——Agent 发现新信息时会调用 `memory` tool 追加

---

## Hermes 的其他 Context 文件

除了 SOUL.md / MEMORY.md / USER.md，Hermes 还支持：

### 4. `.hermes.md` / `HERMES.md` — 项目级上下文

**位置：** 在当前工作目录或父目录中查找（直到 git root）

**用途：** 项目专用的指令和规范。Agent 在项目目录下工作时自动加载。

**查找顺序：**
1. `.hermes.md`（cwd）
2. `HERMES.md`（cwd）
3. 向上查找父目录直到 git root

**与 AGENTS.md 的关系：**
- 两者互斥，同一时间只加载一个
- `.hermes.md` 优先级高于 `AGENTS.md`

### 5. `AGENTS.md` / `agents.md` — 多 Agent 协作

**位置：** 仅当前工作目录

**用途：** 定义多 Agent 协作时的角色和任务分配。

### 6. `.cursorrules` — Cursor IDE 规则

**位置：** `.cursorrules` 或 `.cursor/rules/*.mdc`

**用途：** 从 Cursor IDE 迁移过来的代码规范。

---

## Hermes vs OpenClaw：记忆系统全景对比

```
┌─────────────────────────────────────────────────────────────────┐
│                         Hermes                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   SOUL.md ──────→ Agent Identity（人格/语气）                    │
│   USER.md ──────→ 用户画像（每次会话开始注入）                   │
│   MEMORY.md ────→ Agent 笔记（调用 memory tool 追加）           │
│                                                                  │
│   .hermes.md ───→ 项目上下文（cwd 或 git root）                  │
│   AGENTS.md ────→ 多 Agent 协作                                 │
│                                                                  │
│   ❌ 无自动进化 ─────────────────────────────────────────────────│
│   Agent 不会自动分析错误、改进决策规则                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        OpenClaw                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   SOUL.md ──────→ Soul Pattern（人格 + 行为模式）                 │
│   USER.md ──────→ 用户画像（偏好/习惯）                          │
│   MEMORY.md ────→ 历史记忆（向量存储）                           │
│                                                                  │
│   soul-force ───→ L5 进化引擎（Hook 驱动）                       │
│   ├── L1 巡检 ──→ 19 视角发现问题                               │
│   ├── L2 决策 ──→ 智能派发任务                                   │
│   ├── L3 执行 ──→ 自动修复代码                                  │
│   ├── L4 验收 ──→ 闭环验收                                      │
│   └── L5 进化 ──→ 从 learnings 分析，更新 SOUL.md               │
│                                                                  │
│   ✅ 自动进化 ──────────────────────────────────────────────────│
│   从错误中学习，主动改进 Soul Pattern                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 总结

**Hermes 的记忆系统是"备忘式"的：**
- MEMORY.md = 人工选择性记录
- USER.md = 用户信息积累
- SOUL.md = 固定人格定义
- **不会自动进化**——Agent 只是"记"，不会"学"

**OpenClaw soul-force 是"进化式"的：**
- 自动从错误中学习
- Hook 驱动 L1-L6 闭环
- SOUL.md 本身会被更新
- Agent 不仅"记"，还会"改进自己"

**如果你想要 Hermes 有进化能力**，可以参考 autoself 的设计思路，把 soul-force 的机制嫁接到 Hermes 上。
