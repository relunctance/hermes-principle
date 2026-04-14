# Hermes 系统架构

> 基于源码分析得出的 Hermes Agent 完整架构图

## 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户交互层                               │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌────────┐  ┌──────┐ │
│  │  CLI    │  │ Feishu   │  │Telegram │  │Discord │  │ ...  │ │
│  └────┬────┘  └────┬─────┘  └────┬────┘  └───┬────┘  └──┬───┘ │
└───────┼─────────────┼─────────────┼───────────┼──────────┼─────┘
        │             │             │           │          │
        └─────────────┴─────────────┴───────────┴──────────┘
                              │
                    ┌─────────▼─────────┐
                    │  Gateway (网关)    │
                    │  run.py           │
                    │  - 消息路由       │
                    │  - Slash 命令     │
                    │  - 会话管理       │
                    └─────────┬─────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌─────────────────┐   ┌───────────────┐
│  AIAgent      │   │  Cron Scheduler │   │  MCP Client   │
│  run_agent.py │   │  cron/          │   │  tools/mcp   │
│               │   └─────────────────┘   └───────────────┘
│  - 对话循环   │                                   │
│  - Tool 分发  │                                   │
└───────┬───────┘                                   │
        │                                           │
        ▼                                           ▼
┌───────────────┐                         ┌─────────────────┐
│ model_tools   │                         │ Tool Registry   │
│               │                         │ tools/registry  │
│ - Tool 发现   │────────────────────────►│ - 50+ 内置工具 │
│ - 函数调用    │                         │ - MCP 工具     │
└───────┬───────┘                         └────────┬────────┘
        │                                          │
        │    ┌────────────────────────────────────┘
        │    │
        ▼    ▼
┌───────────────────────────────────────────────────────────────┐
│                        Tools 实现层                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │terminal │ │file_tools│ │delegate  │ │  web_tools   │  │
│  │  tool   │ │          │ │  _tool   │ │              │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │  cron    │ │  skill   │ │  mcp     │ │  browser     │  │
│  │  tool    │ │  tool    │ │  tool    │ │  tool        │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘  │
└───────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                        Skills 层                              │
│  ~/.hermes/skills/                                           │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ builtin (74) │ local (35) │ hub (0)                   │  │
│  │ ┌──────────┐ │ ┌────────┐ │                           │  │
│  │ │llm-wiki  │ │ │hawk-   │ │                           │  │
│  │ │whisper   │ │ │bridge  │ │                           │  │
│  │ │github-*  │ │ │soul-   │ │                           │  │
│  │ │...       │ │ │force   │ │                           │  │
│  │ └──────────┘ │ └────────┘ │                           │  │
│  └────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

## 核心组件

### 1. 用户交互层 (Platforms)

| 平台 | 状态 | 说明 |
|------|------|------|
| CLI | ✅ | 终端交互 |
| Feishu | ✅ | 飞书（当前） |
| Telegram | ✅ | |
| Discord | ✅ | |
| Slack | ✅ | |
| WhatsApp | ✅ | |
| Signal | ✅ | |
| Email | ✅ | |
| Matrix | ✅ | |
| Home Assistant | ✅ | |
| DingTalk | ✅ | |
| WeCom | ✅ | |
| Webhook | ✅ | 自定义回调 |
| Open WebUI | ✅ | |

### 2. Gateway (消息网关)

**文件：** `gateway/run.py`

职责：
- 接收来自各平台的消息
- 路由到 AIAgent
- 管理会话状态
- 执行 Slash 命令
- 发送响应回各平台

```
消息入口 → Gateway → AIAgent → 响应 → Gateway → 平台
```

### 3. AIAgent (核心代理)

**文件：** `run_agent.py`

```python
class AIAgent:
    def chat(self, message: str) -> str
    def run_conversation(self, user_message, system_message, ...) -> dict
```

核心循环：
```python
while api_call_count < max_iterations:
    response = client.chat.completions.create(model=model, messages=messages, tools=tool_schemas)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args)
            messages.append(tool_result_message(result))
    else:
        return response.content
```

### 4. Model Tools (工具编排)

**文件：** `model_tools.py`

- `_discover_tools()` — 发现所有可用工具
- `handle_function_call()` — 分发工具调用

### 5. Tool Registry (工具注册表)

**文件：** `tools/registry.py`

所有工具在导入时通过 `registry.register()` 注册：

| 类别 | 工具 |
|------|------|
| **文件** | read_file, write_file, patch, search_files |
| **终端** | terminal, process |
| **Web** | web_search, web_extract, browser_* |
| **代码** | execute_code, delegate_task |
| **系统** | cronjob, skill_view, skill_manage |
| **MCP** | mcp_tool (支持 1050+ 工具) |
| **其他** | memory, vision_analyze, text_to_speech, ... |

### 6. Skills 系统

**位置：** `~/.hermes/skills/`

```
~/.hermes/skills/
├── .bundled_manifest     # 内置 skills 哈希
├── [category]/
│   └── [skill-name]/
│       ├── SKILL.md      # 必需
│       └── ...
└── openclaw-imports/     # 桥接 OpenClaw skills
```

**发现机制：** 目录扫描 + SKILL.md 解析

### 7. Cron 调度器

**目录：** `cron/`

- 定时任务创建/管理
- 消息触发
- 跨会话执行

## 配置文件

| 文件 | 说明 |
|------|------|
| `~/.hermes/config.yaml` | 主配置 |
| `~/.hermes/.env` | API Keys |
| `~/.hermes/sessions/` | 会话历史 |
| `~/.hermes/logs/` | 日志 |

## 源码结构

```
~/.hermes/hermes-agent/
├── run_agent.py          # AIAgent — 核心对话循环
├── model_tools.py        # Tool 发现与分发
├── toolsets.py           # Toolset 定义
├── cli.py                # 交互式 CLI
├── hermes_state.py       # SQLite 会话存储 (FTS5)
├── agent/                # Agent 内部模块
│   ├── prompt_builder.py     # 系统提示组装
│   ├── context_compressor.py # 上下文压缩
│   ├── prompt_caching.py     # Anthropic 缓存
│   └── display.py            # KawaiiSpinner
├── hermes_cli/           # CLI 子命令
│   ├── main.py           # 入口
│   ├── config.py         # 配置管理
│   └── commands.py       # Slash 命令注册
├── tools/                # 工具实现
│   ├── registry.py       # 中心注册表
│   ├── terminal_tool.py
│   ├── file_tools.py
│   ├── web_tools.py
│   ├── delegate_tool.py
│   └── mcp_tool.py
├── gateway/              # 消息网关
│   ├── run.py
│   └── platforms/        # 平台适配器
└── cron/                 # 调度器
```

## 消息流程

```
用户消息
    │
    ▼
Gateway (run.py)
    │
    ▼
AIAgent.run_conversation()
    │
    ├─► 构建系统提示 (prompt_builder.py)
    │
    ├─► 压缩上下文 (context_compressor.py)
    │
    ├─► LLM API 调用
    │       │
    │       ├─► 无 tool_calls → 返回响应
    │       │
    │       └─► 有 tool_calls
    │               │
    │               ▼
    │           Model Tools (model_tools.py)
    │               │
    │               ▼
    │           Tool Registry → 执行工具
    │               │
    │               ▼
    │           结果注入消息 → 继续循环
    │
    ▼
返回响应
    │
    ▼
Gateway → 用户
```

## 与 OpenClaw 对比

| 维度 | Hermes | OpenClaw |
|------|--------|----------|
| **消息网关** | Gateway (多平台) | Claw |
| **Tool 注册** | Registry (中心化) | Hook 系统 |
| **Skills** | 目录扫描 | Agent 专属 |
| **记忆** | 可插拔 (Honcho/Mem0) | SOUL.md/USER.md |
| **会话** | SQLite (FTS5) | 文件 |
| **定时任务** | Cron 内置 | 依赖外部 |

## 配置示例

```yaml
model:
  default: MiniMax-M2.7-highspeed
  provider: custom
  base_url: https://api.minimaxi.com/anthropic

agent:
  max_turns: 90
  gateway_timeout: 1800
  tool_use_enforcement: auto

toolsets:
  - hermes-cli

gateway:
  platforms:
    feishu:
      enabled: true
```

## 相关文档

- [SKILLS.md](./SKILLS.md) - Skills 创建机制
- [SKILLS-ARCHITECTURE.md](./SKILLS-ARCHITECTURE.md) - Skills 系统架构
- [官方文档](https://hermes-agent.nousresearch.com/docs/)
