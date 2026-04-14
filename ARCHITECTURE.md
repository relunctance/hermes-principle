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
| `~/.hermes/logs/` | 日志目录（agent.log / errors.log / interrupt_debug.log） |

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

### 8. 记忆系统

Hermes 记忆系统由**两层**组成，协同工作实现跨会话持久化记忆。

#### 8.1 原生会话记忆（hermes_state.py）

**文件：** `~/.hermes/hermes-agent/hermes_state.py`

**技术：** SQLite + FTS5 全文搜索

**职责：**
- 存储当前会话的历史消息
- 按 session_id 检索会话上下文
- 提供 `hermes_state.py` 的 `search_sessions()` 方法进行历史搜索

**特点：**
- 会话级别，非跨会话持久化
- 自动加载最近 N 条消息到上下文
- 支持 FTS5 全文检索

#### 8.2 hawk-bridge 跨会话记忆（外部系统）

**概述：**

hawk-bridge 是 Hermes 的记忆增强系统，通过 Hook 机制在每次 `agent:start` 和 `agent:end` 时自动捕获/召回记忆，存储到 LanceDB。

**架构：**

```
                    ┌─────────────────────────────────────────┐
                    │              Hermes Gateway               │
                    │                                          │
                    │  agent:start ──► 召回记忆 ──► 注入上下文  │
用户消息 ──► AIAgent ───────────────────────────────────────────► 响应
                    │              agent:end  ──► 存储记忆    │
                    └──────────────────┬────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────┐
                    │          hawk-memory-api (FastAPI)       │
                    │                                          │
                    │  POST /recall  ──► LanceDB 召回          │
                    │  POST /capture ──► LLM 提取 → 存储       │
                    └──────────────────┬────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────┐
                    │              LanceDB                     │
                    │  ~/.hawk/memory.db                       │
                    │                                          │
                    │  表: hawk_memories                       │
                    │  - id, text, vector, category            │
                    │  - scope, importance, timestamp           │
                    │  - expiresAt                             │
                    └─────────────────────────────────────────┘
```

**四层衰减架构：**

| 层级 | 用途 | 淘汰策略 |
|------|------|---------|
| **Working** | 正在处理的记忆 | 即时淘汰 |
| **Short** | 短期记忆（偏好、事实） | TTL 30天 |
| **Long** | 重要记忆 | 低频访问自动降级 |
| **Archive** | 归档记忆 | 长时间未访问 |

**混合检索管线（6大优化）：**

```
用户 Query
    │
    ▼
1. Query Expansion（查询扩展）
   "openclaw 架构" → ["openclaw 架构", "openclaw design", "openclaw 底层原理", ...]
    │
    ▼
2. Multi-query Vector + BM25
   每个扩展查询独立检索
    │
    ▼
3. RRF Fusion（倒数排名融合）
   合并多路结果
    │
    ▼
4. Noise Filter（噪声过滤）
   过滤"好的""收到"等无意义文本
    │
    ▼
5. Cross-encoder Rerank（交叉编码重排）
   精排相关性
    │
    ▼
6. Confidence Threshold（置信度过滤）
   score < 0.3 丢弃
    │
    ▼
7. MMR Diversity（多样性排序）
   λ × 相关性 - (1-λ) × 最大相似度
    │
    ▼
8. Result Compression（结果压缩）
   超过 200 字符截断到句子边界
    │
    ▼
注入上下文
```

**6大检索优化详解：**

| 优化 | 说明 | 默认值 |
|------|------|--------|
| Query Expansion | 查询扩展成 3-4 个语义相关表述 | 开启，3个扩展 |
| MMR Diversity | λ=0.5 平衡相关性与多样性 | 开启，λ=0.5 |
| Confidence Threshold | 低于 0.3 分的记忆丢弃 | 开启，阈值 0.3 |
| Field-weighted BM25 | category 字段权重 2.0 | 开启 |
| Session Context Awareness | 组合最近对话摘要到 query | 自动 |
| Result Compression | 超过 200 字符自然断句 | 开启，200字符 |

**记忆分类（6类）：**

| 类别 | 说明 |
|------|------|
| `fact` | 客观事实（用户职业、项目名称等） |
| `preference` | 用户偏好（喜欢简洁回复、使用中文等） |
| `decision` | 决策记录（选择了方案A、决定做X等） |
| `entity` | 实体（人名、产品名、工具名等） |
| `context` | 上下文（当前项目、会话背景等） |
| `other` | 其他 |

**接入方式：**

```bash
# Hermes 通过 ~/.hermes/hooks/hawk-bridge/ 接入
~/.hermes/hooks/hawk-bridge/
├── HOOK.yaml      # 声明 agent:start / agent:end
└── handler.py     # 处理函数
```

**HOOK.yaml：**
```yaml
name: hawk-bridge-hermes
description: "Hermes Hook Bridge for hawk-bridge memory system"
events:
  - agent:start   # 召回记忆 → 注入上下文
  - agent:end     # 捕获对话 → 存储记忆
```

**hawk-memory-api 启动：**
```bash
python3 ~/repos/hawk-memory-api/server.py
# 或后台运行
nohup python3 ~/repos/hawk-memory-api/server.py > ~/.hawk/api.log 2>&1 &
```

**配置环境变量：**
```bash
export HAWK__PYTHON__HTTP_MODE=true
export HAWK__PYTHON__HTTP_BASE=http://127.0.0.1:18360
```

**与 OpenClaw 共用：**

hawk-bridge 同时支持 OpenClaw 和 Hermes，共用同一份 LanceDB 数据：
- OpenClaw Hook → `~/.openclaw/hawk/`
- Hermes Hook → `~/.hermes/hooks/hawk-bridge/`

---

### 9. 日志系统

**目录：** `~/.hermes/logs/`

| 文件 | 级别 | 说明 |
|------|------|------|
| `agent.log` | INFO+ | 主日志，记录所有正常运行信息 |
| `errors.log` | WARNING+ | 错误日志，仅记录警告和错误 |
| `interrupt_debug.log` | DEBUG | 中断调试日志（v0.8.0+） |

**日志配置来源：**
- `~/.hermes/config.yaml` 第 270 行 `logging:` 配置段
- `cli.py` 第 526-530 行：启动时通过 `hermes_logging.setup_logging(mode="cli")` 初始化
- v0.8.0 引入集中式日志，取代之前的分散输出

**日志命令：**
```bash
hermes logs        # 查看日志
hermes logs -f     # 实时跟踪
hermes logs --errors  # 仅看错误
```

**日志格式：**
```
2026-04-14 16:11:23,456 INFO  run_agent: Loaded environment variables from ...
2026-04-14 16:11:24,633 ERROR [20260414_094758_d755a4] root: Non-retryable client error: ...
```

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
