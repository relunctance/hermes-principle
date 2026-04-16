# Hermes-Agent 工具系统详解

> 来源：`~/.hermes/hermes-agent/`（v0.8.0，1.3GB）
> 本质：**共享的 Agent 运行时**，所有 Agent 共用同一份代码

---

## 核心问题：多个 Agent 共用一个 hermes-agent 吗？

**是的。** `~/.hermes/hermes-agent/` 是**单例运行时**，所有 Agent（tseng、wukong、bajie、bailong、maomao）共享同一个安装实例。

| 概念 | 说明 |
|------|------|
| `~/.hermes/hermes-agent/` | **运行时代码**（Git 克隆），所有 Agent 共用 |
| `~/.hermes/config.yaml` | **配置**（可以用 profiles 隔离） |
| `~/.hermes/skills/` | ** Skills**（可共享也可独立） |
| `~/.hermes/state.db` | **会话状态**（SQLite，按 session_id 隔离） |

**类比**：hermes-agent 像 Python 解释器，所有 Agent 像 Python 脚本——同一份解释器，不同的脚本内容。

---

## 目录结构

```
~/.hermes/hermes-agent/
├── tools/                    # ★ 54 个工具实现
│   ├── registry.py           # 工具注册表（核心）
│   ├── file_tools.py         # read_file, write_file, patch, search_files
│   ├── terminal_tool.py      # terminal, process
│   ├── browser_tool.py       # 浏览器自动化（87KB）
│   ├── web_tools.py          # web_search, web_extract（87KB）
│   ├── delegate_tool.py      # delegate_task（子 Agent）
│   ├── mcp_tool.py           # MCP 协议桥接（84KB）
│   ├── skill_manager_tool.py # skill_manage
│   ├── memory_tool.py        # memory
│   ├── session_search_tool.py # session_search
│   ├── skills_hub.py         # Skills Hub 市场（111KB）
│   └── environments/         # 执行环境后端
│       ├── local.py          # 本地执行
│       ├── docker.py         # Docker 容器
│       ├── ssh.py            # 远程 SSH
│       ├── modal.py          # Modal 云沙箱
│       └── ...
├── run_agent.py              # Agent 主循环（537KB）
├── model_tools.py            # 工具编排层
├── toolsets.py               # 工具集分组定义
├── cli.py                    # CLI 入口（443KB）
├── gateway/                  # 消息平台网关（Telegram/Discord/飞书...）
├── agent/                    # Agent 内部组件
│   ├── prompt_builder.py     # System Prompt 构造
│   ├── context_compressor.py # 上下文压缩
│   ├── skill_commands.py     # Skill slash 命令
│   └── ...
├── hermes_cli/              # CLI 子命令
├── skills/                   # 内置 Skills（28 个）
├── tests/                    # 测试套件
└── venv/                    # Python 虚拟环境
```

---

## 工具注册机制

### 核心文件

| 文件 | 职责 |
|------|------|
| `tools/registry.py`（335 行） | 工具注册表（单例），所有工具在此登记 |
| `model_tools.py`（577 行） | 工具发现、分发、异步调度 |
| `toolsets.py`（655 行） | 工具集分组（哪些工具打包成一组） |

### 工具注册流程

```
tools/xxx.py（模块加载）
    ↓
registry.register(
    name="tool_name",
    toolset="category",
    schema={...},          # LLM 看到的工具定义
    handler=lambda ...,
    check_fn=check_fn,     # 可选：环境检查
    requires_env=[...],    # 可选：需要的 env
)
    ↓
model_tools.py 在启动时扫描所有工具
    ↓
run_agent.py 将工具 schema 发送给 LLM
    ↓
LLM 决定调用哪个工具
    ↓
handle_function_call() 分发到具体 handler
```

### 工具 Schema 示例

LLM 看到的工具定义（OpenAI 格式）：

```json
{
  "name": "search_files",
  "description": "Search file contents or find files by name...",
  "parameters": {
    "type": "object",
    "properties": {
      "pattern": {"type": "string", "description": "Regex pattern..."},
      "target": {
        "enum": ["content", "files"],
        "description": "'content' searches inside files..."
      },
      "path": {"type": "string", "description": "Directory to search in"}
    }
  }
}
```

---

## 54 个工具分类

### 1. 文件操作（File）

| 工具 | 功能 |
|------|------|
| `read_file` | 读文件（支持 offset/limit，大文件安全） |
| `write_file` | 写文件（覆盖） |
| `patch` | 模糊匹配修改文件（9 种策略） |
| `search_files` | 内容搜索 + 文件名搜索 |

**关键保护机制**（`file_tools.py`）：
- `/dev/*` 设备路径黑名单（防止无限输出）
- 敏感路径保护（`/etc/`、`/boot/`、`/usr/lib/systemd/`）
- 100K 字符读取上限（大文件分段读）
- 路径白名单验证

### 2. 终端执行（Terminal）

| 工具 | 功能 |
|------|------|
| `terminal` | 执行命令（本地/Docker/SSH/Modal） |
| `process` | 管理后台进程 |

**执行环境后端**（`tools/environments/`）：

| 后端 | 说明 |
|------|------|
| `local.py` | 本地执行（最快） |
| `docker.py` | Docker 容器隔离 |
| `ssh.py` | 远程 SSH 执行 |
| `modal.py` | Modal 云沙箱 |
| `singularity.py` | Singularity 容器 |
| `daytona.py` | Daytona 云 IDE |
| `managed_modal.py` | 托管 Modal |

**安全机制**：
- 危险命令检测（`tools/approval.py`）
- sudo 密码缓存 + 交互式提示
- workdir 路径白名单验证
- 磁盘使用警告（>500GB）

### 3. 浏览器自动化（Browser）

| 工具 | 功能 |
|------|------|
| `browser_navigate` | 打开 URL |
| `browser_snapshot` | 获取页面快照 |
| `browser_click` | 点击元素 |
| `browser_type` | 输入文本 |
| `browser_scroll` | 滚动页面 |
| `browser_back` | 返回上一页 |
| `browser_press` | 按键 |
| `browser_get_images` | 获取图片列表 |
| `browser_vision` | 视觉分析截图 |
| `browser_console` | 获取控制台日志 |

**支持后端**：
- Browserbase（云端浏览器）
- Camofox（本地 Firefox）
- Local（本地 Chrome/Firefox）

### 4. Web 搜索提取

| 工具 | 功能 |
|------|------|
| `web_search` | 搜索（支持 Google/Serper/Tavily） |
| `web_extract` | 抓取网页内容（Firecrawl/Trafilatura） |

### 5. 子 Agent 委托

| 工具 | 功能 |
|------|------|
| `delegate_task` | 派发独立子 Agent（Claude Code/Codex/OpenCode） |

**关键机制**：
- 每个子 Agent 独立上下文
- 支持并行（最多 3 个）
- 子 Agent 不能调用 delegate_task/clarify/memory

### 6. Skills 管理

| 工具 | 功能 |
|------|------|
| `skills_list` | 列出可用 Skills |
| `skill_view` | 查看 Skill 内容 |
| `skill_manage` | 创建/修改/删除 Skill |

**Skills Hub**（`skills_hub.py`，111KB）：
- 从 GitHub 安装 Skills
- 安全扫描（quarantine、audit log）
- 支持的源：GitHub、ClawHub、Claude Marketplace、LobeHub

### 7. 记忆系统

| 工具 | 功能 |
|------|------|
| `memory` | 持久化记忆（跨会话） |
| `session_search` | 搜索历史会话 |

**会话状态**（`hermes_state.py`）：
- SQLite WAL 模式
- FTS5 全文搜索
- 按 session_id 隔离

### 8. 其他工具

| 工具 | 功能 |
|------|------|
| `todo` | 任务列表管理 |
| `cronjob` | 定时任务管理 |
| `clarify` | 向用户提问 |
| `text_to_speech` | 文字转语音 |
| `vision_analyze` | 图片分析 |
| `image_generate` | 图片生成 |
| `send_message` | 跨平台发消息 |
| `ha_*` | Home Assistant 控制 |

---

## 工具集（Toolsets）

### 基础工具集

| toolset | 包含工具 |
|---------|---------|
| `web` | web_search, web_extract |
| `search` | web_search |
| `file` | read_file, write_file, patch, search_files |
| `terminal` | terminal, process |
| `browser` | browser_* + web_search |
| `skills` | skills_list, skill_view, skill_manage |
| `vision` | vision_analyze |
| `image_gen` | image_generate |
| `code_execution` | execute_code |
| `delegation` | delegate_task |
| `memory` | memory |
| `session_search` | session_search |
| `todo` | todo |
| `tts` | text_to_speech |
| `homeassistant` | ha_* |

### 场景工具集

| toolset | 说明 |
|---------|------|
| `debugging` | terminal + web + file（调试专用） |
| `safe` | 无 terminal（web + vision + image_gen） |
| `rl` | 强化学习训练工具 |

### 平台工具集

| toolset | 说明 |
|---------|------|
| `hermes-cli` | 完整 CLI（默认） |
| `hermes-telegram` | Telegram Bot |
| `hermes-discord` | Discord Bot |
| `hermes-feishu` | 飞书 Bot |
| `hermes-gateway` | 所有消息平台并集 |

---

## 执行流程图

```
用户消息
    ↓
run_agent.py（AIAgent 类）
    ↓
model_tools.get_tool_definitions()  ← 从 registry 获取工具 schema
    ↓
LLM API（携带工具定义）
    ↓
LLM 返回 tool_calls 或 text
    ↓
如果有 tool_calls：
    model_tools.handle_function_call()
        ↓
        registry.dispatch()  ← 查找 handler
            ↓
        具体工具执行（tools/*.py）
            ↓
        返回 JSON 结果
    ↓
回到 LLM 继续...
```

---

## 如何扩展工具

### 步骤 1：创建工具文件

```python
# tools/my_tool.py
import json
from tools.registry import registry

def my_tool(param: str, task_id: str = None) -> str:
    """工具逻辑"""
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="my_tool",
    toolset="mytools",
    schema={
        "name": "my_tool",
        "description": "这是我的工具",
        "parameters": {
            "type": "object",
            "properties": {
                "param": {"type": "string", "description": "参数说明"}
            },
            "required": ["param"]
        }
    },
    handler=lambda args, **kw: my_tool(
        param=args.get("param", ""),
        task_id=kw.get("task_id")
    ),
)
```

### 步骤 2：引入工具

在 `model_tools.py` 的 `_discover_tools()` 中添加：
```python
from tools import my_tool  # noqa: F401
```

### 步骤 3：添加工具集

在 `toolsets.py` 的 `_HERMES_CORE_TOOLS` 添加：
```python
"my_tool",
```

### 步骤 4：提交推送

```bash
cd ~/.hermes/hermes-agent
git add -A && git commit -m "feat: add my_tool" && git push
```

---

## 多 Agent 隔离机制

虽然所有 Agent 共用 hermes-agent 代码，但通过以下机制隔离：

| 隔离维度 | 机制 |
|---------|------|
| **会话状态** | `state.db` 按 session_id 隔离 |
| **记忆** | `memory` 工具按 user/memory 分开 |
| **Skills** | 可以共享，也可以每个 Agent 独立 |
| **配置** | `config.yaml` 支持 profiles（`hermes -p <profile>`） |
| **工具可用性** | `check_fn` 可以限制特定工具对特定 Agent 不可用 |
| **API Key** | 不同 Agent 可以用不同的 provider credentials |

---

## 相关文档

- [ARCHITECTURE.md](../ARCHITECTURE.md) — Hermes 整体架构
- [SKILLS-ARCHITECTURE.md](../SKILLS-ARCHITECTURE.md) — Skills 系统
- [MEMORY-SYSTEM.md](../MEMORY-SYSTEM.md) — 记忆系统
- [HOOKS.md](../HOOKS.md) — 生命周期钩子
