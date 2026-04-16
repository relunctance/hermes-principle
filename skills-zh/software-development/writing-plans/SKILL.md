---
名称: writing-plans
描述: >
  当有规范或需求的多步任务时使用。创建包含小粒度任务、精确文件路径
  和完整代码示例的综合实施计划。
版本: 1.1.0
作者: Hermes Agent (改编自 obra/superpowers)
许可证: MIT
元数据:
  hermes:
    标签: [规划, 设计, 实现, 工作流, 文档]
    相关技能: [subagent-driven-development, test-driven-development, requesting-code-review]
---

# 编写实施计划

## 概述

写全面的实施计划，假设实现者对代码库零上下文且品味可疑。记录他们需要的一切：改哪些文件、完整代码、测试命令、要查的文档、如何验证。给他们小粒度的任务。DRY。YAGNI。TDD。频繁提交。

假设实现者是个有经验的开发者，但对工具集或问题领域几乎一无所知。假设他们不太懂好的测试设计。

**核心原则：** 好的计划使实现显而易见。如果有人需要猜测，计划就不完整。

## 何时使用

**始终使用，在：**
- 实现多步功能之前
- 分解复杂需求时
- 通过 subagent-driven-development 委托给子 Agent 之前

**不要跳过，当：**
- 功能看起来简单时（假设导致 bug）
- 打算自己实现时（未来的你需要指导）
- 独自工作时（文档很重要）

## 小粒度任务

**每个任务 = 2-5 分钟的专注工作。**

每一步都是一个动作：
- "写失败的测试" — 一步
- "运行它确保它失败" — 一步
- "写最少的代码让测试通过" — 一步
- "运行测试确保它们通过" — 一步
- "提交" — 一步

**太大了：**
```markdown
### 任务 1：构建认证系统
[跨 5 个文件的 50 行代码]
```

**大小合适：**
```markdown
### 任务 1：创建带 email 字段的 User 模型
[10 行，1 个文件]

### 任务 2：给 User 添加 password hash 字段
[8 行，1 个文件]

### 任务 3：创建密码哈希工具
[15 行，1 个文件]
```

## 计划文档结构

### 头部（必需）

每个计划必须以这个开始：

```markdown
# [功能名称] 实施计划

> **给 Hermes：** 使用 subagent-driven-development 技能按任务逐步实现本计划。

**目标：** [一句话描述这个构建什么]

**架构：** [2-3 句话说明方案]

**技术栈：** [关键技术/库]

---
```

### 任务结构

每个任务遵循这个格式：

````markdown
### 任务 N：[描述性名称]

**目标：** 这个任务完成什么（一句话）

**文件：**
- 创建：`exact/path/to/new_file.py`
- 修改：`exact/path/to/existing.py:45-67`（如果知道行号）
- 测试：`tests/path/to/test_file.py`

**步骤 1：写失败的测试**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

**步骤 2：运行测试验证失败**

运行：`pytest tests/path/test.py::test_specific_behavior -v`
预期：FAIL — "function not defined"

**步骤 3：写最简实现**

```python
def function(input):
    return expected
```

**步骤 4：运行测试验证通过**

运行：`pytest tests/path/test.py::test_specific_behavior -v`
预期：PASS

**步骤 5：提交**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 编写流程

### 步骤 1：理解需求

阅读并理解：
- 功能需求
- 设计文档或用户描述
- 验收标准
- 约束

### 步骤 2：探索代码库

用 Hermes 工具理解项目：

```python
# 理解项目结构
search_files("*.py", target="files", path="src/")

# 看类似功能
search_files("similar_pattern", path="src/", file_glob="*.py")

# 检查现有测试
search_files("*.py", target="files", path="tests/")

# 读关键文件
read_file("src/app.py")
```

### 步骤 3：设计方案

决定：
- 架构模式
- 文件组织
- 需要的依赖
- 测试策略

### 步骤 4：写任务

按顺序创建任务：
1. 基础设施/设置
2. 核心功能（每个都 TDD）
3. 边界情况
4. 集成
5. 清理/文档

### 步骤 5：添加完整细节

每个任务要包含：
- **精确的文件路径**（不是"配置文件"而是`src/config/settings.py`）
- **完整的代码示例**（不是"添加验证"而是实际代码）
- **精确的命令**及预期输出
- **验证步骤**证明任务有效

### 步骤 6：审查计划

检查：
- [ ] 任务顺序合理
- [ ] 每个任务小粒度（2-5 分钟）
- [ ] 文件路径精确
- [ ] 代码示例完整（可复制粘贴）
- [ ] 命令精确并有预期输出
- [ ] 无缺失上下文
- [ ] 应用了 DRY、YAGNI、TDD 原则

### 步骤 7：保存计划

```bash
mkdir -p docs/plans
# 保存计划到 docs/plans/YYYY-MM-DD-feature-name.md
git add docs/plans/
git commit -m "docs: add implementation plan for [feature]"
```

## 原则

### DRY（不要重复自己）

**坏：** 在 3 个地方复制粘贴验证
**好：** 提取验证函数，到处使用

### YAGNI（你不需要它）

**坏：** 为未来需求添加"灵活性"
**好：** 只实现现在需要的

```python
# 坏 — YAGNI 违规
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email
        self.preferences = {}  # 现在还不需要！
        self.metadata = {}     # 现在还不需要！

# 好 — YAGNI
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email
```

### TDD（测试驱动开发）

每个产生代码的任务都应包含完整 TDD 循环：
1. 写失败的测试
2. 运行验证失败
3. 写最简代码
4. 运行验证通过

详见 `test-driven-development` 技能。

### 频繁提交

每个任务后提交：
```bash
git add [files]
git commit -m "type: description"
```

## 常见错误

### 模糊任务

**坏：** "添加认证"
**好：** "创建带 email 和 password_hash 字段的 User 模型"

### 不完整代码

**坏：** "步骤 1：添加验证函数"
**好：** "步骤 1：添加验证函数" 后跟完整函数代码

### 缺失验证

**坏：** "步骤 3：测试它能用"
**好：** "步骤 3：运行 `pytest tests/test_auth.py -v`，预期：3 passed"

### 缺失文件路径

**坏：** "创建模型文件"
**好：** "创建：`src/models/user.py`"

## 执行交接

保存计划后，提供执行方案：

**"计划完成并保存。准备好使用 subagent-driven-development 执行——每个任务派发一个全新子 Agent，进行两阶段审查（规范合规然后代码质量）。是否继续？"**

执行时，使用 `subagent-driven-development` 技能：
- 每个任务用全新 `delegate_task`，附完整上下文
- 每个任务后进行规范合规审查
- 规范通过后进行代码质量审查
- 两阶段审查都批准后才能继续

## 记住

```
小粒度任务（每个 2-5 分钟）
精确文件路径
完整代码（可复制粘贴）
精确命令及预期输出
验证步骤
DRY、YAGNI、TDD
频繁提交
```

**好的计划使实现显而易见。**
