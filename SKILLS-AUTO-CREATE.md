# Hermes Skills 自动创建：原理与策略

> 基于 `~/.hermes/hermes-agent/tools/skill_manager_tool.py` 源码分析

## 核心结论

**Hermes 没有自动创建 skill 的后台进程或调度器。** Skills 的创建完全依赖 Agent **主动决策**——由 LLM 根据 System Prompt 中的指导原则，决定是否调用 `skill_manage` 工具来创建/更新 skill。

---

## 一、原理：Skill 是纯文件系统的发现机制

### 1.1 Skills 的本质

Skills 不是什么特殊的运行时对象，**本质上就是 `~/.hermes/skills/[category]/[name]/SKILL.md` 文件**。

```
~/.hermes/skills/
├── my-skill/                 # category 可选
│   └── SKILL.md             # 必需的核心文件
├── devops/
│   └── docker-debug/
│       ├── SKILL.md
│       ├── references/       # 支持文件
│       └── scripts/          # 脚本
└── openclaw-imports/         # 桥接 OpenClaw skills
```

### 1.2 SKILL.md 结构

```yaml
---
name: docker-debug
description: "Docker 容器调试工作流"
version: 1.0.0
author: hermes-agent
---

# Docker Debug Skill

## 触发条件
容器无法启动或频繁重启时使用。

## 排查步骤
1. `docker ps -a` 查看容器状态
2. `docker logs <container>` 检查日志
...
```

### 1.3 发现机制

Hermes 启动时通过 `prompt_builder.py` 中的 `build_skills_system_prompt()` 扫描 skills 目录：

1. 扫描 `~/.hermes/skills/` 所有子目录
2. 读取每个 skill 的 `SKILL.md` 提取 frontmatter
3. 生成 `<available_skills>` 索引块注入 System Prompt
4. Agent 根据描述判断是否需要加载（`skill_view`）

**Skills 的"注册"是纯目录扫描，无需显式注册表。**

---

## 二、策略：System Prompt 中的四层行为指导

Hermes 在 System Prompt 中通过**四层叠加**来驱动 Agent 主动创建 skill：

### 2.1 第一层：MEMORY_GUIDANCE 中嵌入的一句提示

**文件：** `prompt_builder.py` 第 144-156 行

```python
MEMORY_GUIDANCE = (
    "You have persistent memory across sessions. Save durable facts using the memory "
    "tool: user preferences, environment details, tool quirks, and stable conventions. "
    "Memory is injected into every turn, so keep it compact and focused on facts that "
    "will still matter later.\n"
    "Prioritize what reduces future user steering — the most valuable memory is one "
    "that prevents the user from having to correct or remind you again. "
    "User preferences and recurring corrections matter more than procedural task details.\n"
    "Do NOT save task progress, session outcomes, completed-work logs, or temporary TODO "
    "state to memory; use session_search to recall those from past transcripts. "
    "If you've discovered a new way to do something, solved a problem that could be "
    "save it as a skill with the skill tool."   # <-- 嵌入在这里
)
```

# 中文说明：
# 1. 记忆是跨会话持久的，不要担心忘记
# 2. 用 memory tool 保存：用户偏好、环境细节、工具特性、稳定惯例
# 3. 记忆要精简，只保存长期有用的信息
# 4. 最有价值的记忆：减少用户重复指导（用户纠正过的不要再问）
# 5. 不要存：任务进度、会话结果、已完成的工作日志、临时 TODO
#    → 这些用 session_search 查过往记录
# 6. 如果发现了新方法/解决了问题 → 用 skill tool 保存（不是 memory）
#    → memory 存"事实"，skill 存"流程/经验"

**作用：** 将 skill 创建与 general memory 区分开来——程序化工作流用 skill，事实性信息用 memory。

---

### 2.2 第二层：独立的 SKILLS_GUIDANCE 常量

**文件：** `prompt_builder.py` 第 164-171 行

```python
SKILLS_GUIDANCE = (
    "After completing a complex task (5+ tool calls), fixing a tricky error, "
    "or discovering a non-trivial workflow, save the approach as a "
    "skill with skill_manage so you can reuse it next time.\n"
    "When using a skill and finding it outdated, incomplete, or wrong, "
    "patch it immediately with skill_manage(action='patch') — don't wait to be asked. "
    "Skills that aren't maintained become liabilities."
)
```

**中文说明：**
- **什么时候该创建 Skill？**
  1. 复杂任务（5+ 工具调用）成功完成
  2. 克服了棘手的错误
  3. 发现了 nontrivial 的工作流程
- **什么时候该更新 Skill？**
  1. 发现内容过时/错误/不完整
  2. 遇到 OS 特异性失败
  3. 发现了新的陷阱或坑
  4. 使用时发现缺失步骤

**作用：** 独立常量，在 System Prompt 的 Guidance 段被引用。明确给出"5+ tool calls"的量化阈值，并强调**维护责任**——过时 skill 是负担。

---

### 2.3 第三层：Skill Index 说明段中的行为指导

**文件：** `prompt_builder.py` 第 757-772 行

这是在 `<available_skills>` 索引块之前的一段说明，被直接注入 System Prompt：

```python
result = (
    "## Skills (mandatory)\n"
    "Before replying, scan the skills below. If a skill matches or is even partially relevant "
    "to your task, you MUST load it with skill_view(name) and follow its instructions. "
    "Err on the side of loading — it is always better to have context you don't need "
    "than to miss critical steps, pitfalls, or established workflows. "
    ...
    "If a skill has issues, fix it with skill_manage(action='patch').\n"
    "After difficult/iterative tasks, offer to save as a skill. "          # <-- 创建建议
    "If a skill you loaded was missing steps, had wrong commands, or needed "
    "pitfalls you discovered, update it before finishing.\n"              # <-- 更新建议
    ...
)
```

**作用：** 在 Agent 每次响应前都会读到这段话——扫描 skill → 加载相关 skill → 如有缺失则更新/创建。

---

### 2.4 第四层：skill_manage Tool Schema 的 description

**文件：** `tools/skill_manager_tool.py` 第 653-673 行

Tool 的 description 是 LLM 看到的**最终触发条件清单**：

```
"Manage skills (create, update, delete). Skills are your procedural "
"memory — reusable approaches for recurring task types. "
"New skills go to ~/.hermes/skills/; existing skills can be modified wherever they live.\n\n"
"Actions: create (full SKILL.md + optional category), "
"patch (old_string/new_string — preferred for fixes), "
"edit (full SKILL.md rewrite — major overhauls only), "
"delete, write_file, remove_file.\n\n"
"Create when: complex task succeeded (5+ calls), errors overcome, "
"user-corrected approach worked, non-trivial workflow discovered, "
"or user asks you to remember a procedure.\n"        # <-- 明确触发条件
"Update when: instructions stale/wrong, OS-specific failures, "
"missing steps or pitfalls found during use. "
"If you used a skill and hit issues not covered by it, patch it immediately.\n\n"
"After difficult/iterative tasks, offer to save as a skill. "
"Skip for simple one-offs. Confirm with user before creating/deleting.\n\n"
"Good skills: trigger conditions, numbered steps with exact commands, "
"pitfalls section, verification steps. Use skill_view() to see format examples."
```

**中文说明：**
- **创建 Skill 的时机**
  1. 复杂任务成功（5+ 工具调用）—— 需要多次操作才能完成的任务
  2. 克服了错误 —— 解决了棘手的 bug 或问题
  3. 发现了 nontrivial 工作流 —— 找到了意想不到但有效的解决方案
  4. 用户纠正生效 —— 用户纠正了你的方法且有效，说明原本流程有问题
  5. 用户要求记住流程 —— 主动要求记住的操作步骤
- **更新 Skill 的时机**
  1. 内容过时/错误/不完整 —— 发现 Skill 中的信息已经不对了
  2. OS 特异性失败 —— 遇到了特定操作系统的问题
  3. 发现新坑/陷阱 —— 使用时发现了原本没写进去的陷阱
  4. 缺失步骤 —— 使用时发现步骤不完整
- **什么是"好的 Skill"**
  - 触发条件：明确说明在什么情况下使用这个 Skill
  - 带命令的编号步骤：每个步骤都有具体命令，方便复制执行
  - 陷阱说明：提醒可能出问题的地方
  - 验证步骤：确认任务完成的方法

**作用：** Tool 调用时 LLM 必读，是最终的"操作手册"。

---

## 三、创建流程：Agent 主动决策链

```
用户请求任务
    │
    ▼
AIAgent.run_conversation()
    │
    ├─► 构建 System Prompt（含 SKILLS_GUIDANCE + available_skills index）
    │
    ▼
LLM 思考是否值得创建 skill
    │
    ├─► 判断：任务复杂度（5+ tool calls）？
    ├─► 判断：发现新的工作流？
    ├─► 判断：用户纠正了错误？
    └─► 判断：未来可能再次需要？
    │
    ▼（满足任一条件）
LLM 调用 skill_manage tool
    │
    ▼
skill_manage(action='create', name='...', content='...')
    │
    ├─► 验证 name/description/frontmatter
    ├─► 原子写入 ~/.hermes/skills/[category]/[name]/SKILL.md
    ├─► 安全扫描（skills_guard）
    └─► 成功：清除 skills 缓存
    │
    ▼
Skill 在下次启动时自动被发现
```

---

## 四、Skill 管理的 6 个操作

`skill_manage` 工具支持 6 个 action：

| Action | 说明 | 使用场景 |
|--------|------|----------|
| `create` | 创建新 skill（含 SKILL.md） | 沉淀新工作流 |
| `patch` | 目标字符串替换 | 修复过时内容 |
| `edit` | 完整重写 SKILL.md | 重大重构 |
| `delete` | 删除 skill | 删除无用 skill |
| `write_file` | 添加支持文件 | 添加 reference/script |
| `remove_file` | 删除支持文件 | 清理废弃文件 |

---

## 五、安全机制

### 5.1 写入安全

- **原子写入**：先写临时文件，再 `os.replace()`，防止崩溃后文件损坏
- **路径验证**：禁止 `..` 路径遍历，支持文件必须在 `references/templates/scripts/assets` 下
- **大小限制**：SKILL.md ≤ 100,000 字符，支持文件 ≤ 1 MiB

### 5.2 安全扫描

创建/修改后自动执行 `skills_guard.scan_skill()`：
- 被阻止（`allowed=False`）→ 回滚文件变更
- 警告（`allowed=None`）→ 允许但记录日志

---

## 六、缓存机制

### 6.1 Skills Prompt 缓存

`build_skills_system_prompt()` 实现了两层缓存：

1. **进程内 LRU 缓存**：按 `(skills_dir, external_dirs, tools, toolsets, platform)` 生成 cache key
2. **磁盘快照**：`.skills_prompt_snapshot.json` 跨进程持久化

创建 skill 后调用 `clear_skills_system_prompt_cache()` 使缓存失效。

### 6.2 Skill 位置优先级

- **本地 skills**（`~/.hermes/skills/`）：可读写
- **外部 dirs**（`skills.external_dirs` 配置）：只读，显示在索引中

同名时本地优先。

---

## 七、与 OpenClaw soul-force 的区别

| 维度 | Hermes skill_manage | OpenClaw soul-force |
|------|-------------------|---------------------|
| **创建触发** | Agent 主动决策（LLM 判断） | 层级化的 L1-L6 巡检体系 |
| **执行机制** | Tool call → 文件写入 | Hook 事件 → 记忆提取 → 验证 → 进化 |
| **存储形式** | Markdown 文件 | LanceDB + SOUL.md |
| **更新粒度** | Skill 级别 | 记忆片段级别 |
| **自动化程度** | 半自动（需 LLM 决策） | 全自动（Hook 驱动） |

---

## 八、总结

**Hermes 自动创建 skills 的"自动化"是 Prompt 引导的 Agent 主动行为，而非后台进程。**

```
System Prompt (SKILLS_GUIDANCE)
         │
         ▼
    LLM Agent 感知到：
    - 任务复杂（5+ tool calls）
    - 发现新工作流
    - 用户纠正了错误
         │
         ▼
    主动调用 skill_manage(action='create', ...)
         │
         ▼
    写入 ~/.hermes/skills/[name]/SKILL.md
         │
         ▼
    下次启动 → 扫描发现 → 自动进入 available_skills 索引
```

**核心本质**：Skills 是**程序化记忆（Procedural Memory）**的载体，Hermes 通过让 Agent 自己判断并调用 `skill_manage` 工具来实现"从经验中学习"——这是一种**由 LLM 驱动的自我改进机制**，而不是预先编程的自动化流程。
