---
名称: subagent-driven-development
描述: >
  当执行具有独立任务实施计划时使用。每个任务派发全新的 delegate_task，
  进行两阶段审查（规范合规然后代码质量）。
版本: 1.1.0
作者: Hermes Agent (改编自 obra/superpowers)
许可证: MIT
元数据:
  hermes:
    标签: [委托, 子Agent, 实现, 工作流, 并行]
    相关技能: [writing-plans, requesting-code-review, test-driven-development]
---

# 子 Agent 驱动开发

## 概述

通过为每个任务派发全新子 Agent 并进行系统化两阶段审查来执行实施计划。

**核心原则：** 每个任务全新子 Agent + 两阶段审查（先规范后质量）= 高质量、快速迭代。

## 何时使用

以下情况使用本技能：
- 有实施计划时（来自 writing-plans 技能或用户需求）
- 任务大部分是独立的
- 质量和规范合规重要
- 想要任务间的自动化审查

**vs. 手动执行：**
- 每个任务全新上下文（无累积状态混淆）
- 自动化审查流程早期抓住问题
- 所有任务一致的质量检查
- 子 Agent 可以在开始工作前提问

## 流程

### 1. 读取并解析计划

读取计划文件。提前提取所有任务的完整文本和上下文。创建 todo 列表：

```python
# 读取计划
read_file("docs/plans/feature-plan.md")

# 创建包含所有任务的 todo 列表
todo([
    {"id": "task-1", "content": "创建带 email 字段的 User 模型", "status": "pending"},
    {"id": "task-2", "content": "添加密码哈希工具", "status": "pending"},
    {"id": "task-3", "content": "创建登录端点", "status": "pending"},
])
```

**关键：** 读一次计划。提取一切。不要让子 Agent 读计划文件——直接把完整任务文本放入上下文。

### 2. 每个任务的工作流

对计划中的每个任务：

#### 步骤 1：派发实现者子 Agent

用完整上下文使用 `delegate_task`：

```python
delegate_task(
    goal="实现任务 1：创建带 email 和 password_hash 字段的 User 模型",
    context="""
    计划中的任务：
    - 创建：src/models/user.py
    - 添加 User 类，含 email (str) 和 password_hash (str) 字段
    - 使用 bcrypt 做密码哈希
    - 包含 __repr__ 用于调试

    遵循 TDD：
    1. 在 tests/models/test_user.py 写失败的测试
    2. 运行：pytest tests/models/test_user.py -v（验证 FAIL）
    3. 写最简实现
    4. 运行：pytest tests/models/test_user.py -v（验证 PASS）
    5. 运行：pytest tests/ -q（验证无回归）
    6. 提交：git add -A && git commit -m "feat: add User model with password hashing"

    项目上下文：
    - Python 3.11，Flask 应用在 src/app.py
    - 现有模型在 src/models/
    - 测试使用 pytest，从项目根目录运行
    - bcrypt 已在 requirements.txt
    """,
    toolsets=['terminal', 'file']
)
```

#### 步骤 2：派发规范合规审查者

实现者完成后，对照原始规范验证：

```python
delegate_task(
    goal="审查实现是否符合计划中的规范",
    context="""
    原始任务规范：
    - 创建含 User 类的 src/models/user.py
    - 字段：email (str)、password_hash (str)
    - 使用 bcrypt 做密码哈希
    - 包含 __repr__

    检查：
    - [ ] 所有规范中的需求都实现了？
    - [ ] 文件路径符合规范？
    - [ ] 函数签名符合规范？
    - [ ] 行为符合预期？
    - [ ] 没有添加额外内容（无范围蔓延）？

    输出：PASS 或具体规范差距列表以便修复。
    """,
    toolsets=['file']
)
```

**如果发现规范问题：** 修复差距，然后重新运行规范审查。只有在规范合规后才能继续。

#### 步骤 3：派发代码质量审查者

规范合规通过后：

```python
delegate_task(
    goal="审查任务 1 实现的代码质量",
    context="""
    要审查的文件：
    - src/models/user.py
    - tests/models/test_user.py

    检查：
    - [ ] 遵循项目约定和风格？
    - [ ] 错误处理得当？
    - [ ] 变量/函数名称清晰？
    - [ ] 测试覆盖充分？
    - [ ] 无明显 bug 或错过的边界情况？
    - [ ] 无安全问题？

    输出格式：
    - 关键问题：[继续前必须修复]
    - 重要问题：[应该修复]
    - 小问题：[可选]
    - 裁决：APPROVED 或 REQUEST_CHANGES
    """,
    toolsets=['file']
)
```

**如果发现质量问题：** 修复问题，重新审查。只有批准后才能继续。

#### 步骤 4：标记完成

```python
todo([{"id": "task-1", "content": "创建带 email 字段的 User 模型", "status": "completed"}], merge=True)
```

### 3. 最终审查

所有任务完成后，派发最终集成审查者：

```python
delegate_task(
    goal="审查整个实现的一致性和集成问题",
    context="""
    所有任务都已完成。审查完整实现：
    - 所有组件一起工作吗？
    - 任务之间有任何不一致吗？
    - 所有测试通过吗？
    - 可以合并了吗？
    """,
    toolsets=['terminal', 'file']
)
```

### 4. 验证并提交

```bash
# 运行完整测试套件
pytest tests/ -q

# 审查所有改动
git diff --stat

# 如需要，最后提交
git add -A && git commit -m "feat: complete [feature name] implementation"
```

## 任务粒度

**每个任务 = 2-5 分钟的专注工作。**

**太大了：**
- "实现用户认证系统"

**大小合适：**
- "创建带 email 和 password 字段的 User 模型"
- "添加密码哈希函数"
- "创建登录端点"
- "添加 JWT token 生成"
- "创建注册端点"

## 红旗 — 绝不要做这些

- 没有计划就实现
- 跳过审查（规范合规或代码质量）
- 在有关键/重要问题未修复时继续
- 为涉及相同文件的多个任务同时派发多个实现子 Agent
- 让子 Agent 读计划文件（把完整文本放入上下文）
- 跳过场景设置上下文（子 Agent 需要理解任务在哪里）
- 忽略子 Agent 的问题（在让他们继续前回答）
- 对规范合规接受"差不多就行"
- 跳过审查循环（审查者发现问题 → 实现者修复 → 再次审查）
- 让实现者自审替代实际审查（两者都需要）
- **在规范合规 PASS 之前开始代码质量审查**（顺序错了）
- 在任一审查有关开放问题时进入下一任务

## 处理问题

### 如果子 Agent 提问

- 清晰完整地回答
- 如需要提供额外上下文
- 不要催他们进入实现

### 如果审查者发现问题

- 实现者子 Agent（或新的）修复它们
- 审查者再次审查
- 重复直到批准
- 不要跳过重新审查

### 如果子 Agent 任务失败

- 派发新的修复子 Agent，具体说明哪里出了问题
- 不要在控制器会话中手动修复（上下文污染）

## 效率说明

**为什么每个任务全新子 Agent：**
- 防止累积状态造成的上下文污染
- 每个子 Agent 获得干净、专注的上下文
- 无先前任务的代码或推理混淆

**为什么两阶段审查：**
- 规范审查在早期抓住建设不足/过度建设
- 质量审查确保实现构建良好
- 在问题跨任务叠加之前抓住它们

**成本权衡：**
- 更多子 Agent 调用（每个任务实现者 + 2 个审查者）
- 但早期抓住问题（比后期调试叠加问题便宜）

## 与其他技能集成

### 结合 writing-plans

本技能**执行**由 writing-plans 技能创建的计划：
1. 用户需求 → writing-plans → 实施计划
2. 实施计划 → subagent-driven-development → 可工作代码

### 结合 test-driven-development

实现者子 Agent 应遵循 TDD：
1. 先写失败的测试
2. 实现最简代码
3. 验证测试通过
4. 提交

在每个实现者上下文中包含 TDD 指令。

### 结合 requesting-code-review

两阶段审查流程**就是**代码审查。最终集成审查使用 requesting-code-review 技能的审查维度。

### 结合 systematic-debugging

如果子 Agent 在实现过程中遇到 bug：
1. 遵循 systematic-debugging 流程
2. 找到根因再修复
3. 写回归测试
4. 继续实现

## 示例工作流

```
[读取计划：docs/plans/auth-feature.md]
[创建含 5 个任务的 todo 列表]

--- 任务 1：创建 User 模型 ---
[派发实现者子 Agent]
  实现者："email 需要唯一吗？"
  你："是的，email 必须唯一"
  实现者：已实现，3/3 测试通过，已提交。

[派发规范审查者]
  规范审查者：✅ PASS — 所有需求满足

[派发质量审查者]
  质量审查者：✅ APPROVED — 代码干净，测试良好

[标记任务 1 完成]

--- 任务 2：密码哈希 ---
[派发实现者子 Agent]
  实现者：无问题，已实现，5/5 测试通过。

[派发规范审查者]
  规范审查者：❌ 缺失：密码强度验证（规范说"最少 8 字符"）

[实现者修复]
  实现者：已添加验证，7/7 测试通过。

[再次派发规范审查者]
  规范审查者：✅ PASS

[派发质量审查者]
  质量审查者：重要：魔法数字 8，提取为常量
  实现者：已提取 MIN_PASSWORD_LENGTH 常量
  质量审查者：✅ APPROVED

[标记任务 2 完成]

...（继续所有任务）

[所有任务后：派发最终集成审查者]
[运行完整测试套件：全部通过]
[完成！]
```

## 记住

```
每个任务全新子 Agent
每次两阶段审查
先规范合规
再代码质量
绝不跳过审查
早期抓住问题
```

**质量不是意外。是系统化流程的结果。**
