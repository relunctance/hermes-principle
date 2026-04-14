# Hermes 用户交互层与网关通信机制

> 基于源码分析：gateway/run.py、platforms/base.py、platforms/feishu.py

## 通信架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户 (Feishu App)                        │
└──────────────────────────┬────────────────────────────────────┘
                           │ HTTP/WebSocket
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Feishu Platform Adapter                          │
│                  (platforms/feishu.py)                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - WebSocket 长连接 或 Webhook 接收消息                   │   │
│  │ - 消息标准化 → MessageEvent                              │   │
│  │ - 发送响应回 Feishu                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
└──────────────────────────┬────────────────────────────────────┘
                           │
                    MessageEvent
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    GatewayRunner (run.py)                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ async def _handle_message(event: MessageEvent)           │   │
│  │     1. 验证用户授权                                      │   │
│  │     2. 检查 Slash 命令 (/new, /reset, /model...)        │   │
│  │     3. 中断正在运行的 Agent (如需要)                     │   │
│  │     4. 获取或创建会话                                    │   │
│  │     5. 构建 Agent 上下文                                 │   │
│  │     6. 调用 AIAgent.run_conversation()                   │   │
│  │     7. 返回响应                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
└──────────────────────────┬────────────────────────────────────┘
                           │
                    AIAgent.run_conversation()
                           │
            ┌──────────────┴──────────────┐
            │                             │
            ▼                             ▼
    LLM API 调用                  Tool Registry 执行
    (返回响应)                    (工具结果注入消息)
                                         │
                                         ▼
                                  继续循环直到完成
                                         │
                                         ▼
                           返回最终响应字符串
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Feishu Platform Adapter                       │
│                  发送响应文本回 Feishu                           │
└─────────────────────────────────────────────────────────────────┘
```

## 核心组件

### 1. BasePlatformAdapter (平台适配器基类)

```python
# gateway/platforms/base.py
class BasePlatformAdapter(ABC):
    def __init__(self, config: PlatformConfig, platform: Platform):
        self._message_handler: Optional[MessageHandler] = None
        self._running = False

    @abstractmethod
    async def send_message(self, chat_id: str, text: str, ...) -> SendResult: ...

    @abstractmethod
    async def handle_message(self, event: MessageEvent) -> None:
        """接收消息，转换为 MessageEvent，路由到 Gateway"""
```

### 2. Feishu Platform Adapter

```python
# gateway/platforms/feishu.py
class FeishuAdapter(BasePlatformAdapter):
    # 支持两种模式：
    # 1. WebSocket 长连接 (优先)
    # 2. Webhook HTTP 回调

    async def _handle_ws_message(self, ws_message):
        # WebSocket 接收 Feishu 事件
        event = self._parse_ws_message(ws_message)
        await self._gateway.route_message(event)

    async def _handle_webhook(self, request):
        # Webhook 模式接收
        event = self._parse_webhook(request)
        await self._gateway.route_message(event)
```

### 3. MessageEvent (消息标准化)

```python
# gateway/platforms/base.py
@dataclass
class MessageEvent:
    platform: Platform
    chat_id: str
    user_id: str
    text: str
    message_type: MessageType  # text / image / audio / command / ...
    source: SessionSource
    metadata: dict  # 平台特定元数据
```

### 4. GatewayRunner (消息网关核心)

```python
# gateway/run.py
class GatewayRunner:
    def __init__(self, config: GatewayConfig):
        self.adapters: Dict[Platform, BasePlatformAdapter] = {}
        self.session_store: SessionStore
        self._running_agents: Dict[str, AIAgent]  # per-session agent 缓存

    async def _handle_message(self, event: MessageEvent) -> Optional[str]:
        """核心消息处理流程"""
        # 1. 授权检查
        if not self._is_user_authorized(event.source):
            return self._handle_unauthorized(event)

        # 2. 检查 Slash 命令
        command = event.get_command()
        if command in KNOWN_COMMANDS:
            return await self._handle_command(event, command)

        # 3. 检查并中断运行中的 Agent
        if event.session_key in self._running_agents:
            await self._interrupt_agent(event)

        # 4. 获取/创建会话
        session = self.session_store.get_or_create_session(event.source)

        # 5. 构建上下文
        context = build_session_context(event.source, self.config, session)
        context_prompt = build_session_context_prompt(context)

        # 6. 调用 AIAgent
        return await self._handle_message_with_agent(event, session, context_prompt)
```

### 5. AIAgent (Agent 执行器)

```python
# run_agent.py
class AIAgent:
    def run_conversation(
        self,
        user_message: str,
        system_message: str = None,
        conversation_history: list = None,
    ) -> dict:
        """同步方法，在 _handle_message_with_agent 中被 await"""
        messages = self._build_messages(user_message, system_message, history)

        while self.api_call_count < self.max_iterations:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                tools=self.tool_schemas,
            )

            if response.tool_calls:
                for tool_call in response.tool_calls:
                    result = self._execute_tool(tool_call)
                    messages.append(self._tool_result_message(tool_call, result))
                self.api_call_count += 1
            else:
                return {"final_response": response.content, "messages": messages}

        return {"final_response": "...", "messages": messages}
```

## 完整消息流程

```
1. 用户在 Feishu 发送消息
         │
         ▼
2. Feishu 服务器通过 WebSocket 或 Webhook 推送消息
         │
         ▼
3. FeishuAdapter._handle_ws_message() 或 _handle_webhook()
         │
         ▼
4. 解析消息，转换为 MessageEvent
         │
         ▼
5. GatewayRunner._handle_message(event) 被调用
         │
         ├─► 授权检查
         │
         ├─► Slash 命令处理
         │
         ├─► 会话获取/创建
         │
         └─► 调用 _handle_message_with_agent()
                  │
                  ▼
             6. 构建上下文 (session context + system prompt)
                  │
                  ▼
             7. AIAgent.run_conversation() 循环
                  │
                  ├─► LLM API 调用
                  │
                  ├─► 有 tool_calls → 执行工具 → 继续循环
                  │
                  └─► 无 tool_calls → 返回响应
                       │
                       ▼
                  8. Gateway 返回响应字符串
                       │
                       ▼
             9. FeishuAdapter.send_message(chat_id, response)
                       │
                       ▼
             10. 响应显示在 Feishu
```

## 会话管理

Gateway 为每个用户/聊天维护独立的 AIAgent 实例缓存：

```python
# GatewayRunner.__init__
self._agent_cache: Dict[str, tuple] = {}  # session_key → (AIAgent, config_signature)
```

**为什么需要缓存？**
- 保持 prompt caching (Anthropic)
- 避免每次消息重建系统提示（包含 memory、skills）
- 维护跨消息的上下文

**会话 Key 格式：**
```python
# 基于 platform + user_id + chat_id 生成
session_key = f"{platform}:{user_id}:{chat_id}"
```

## 授权机制

```python
async def _is_user_authorized(self, source: SessionSource) -> bool:
    # 1. 检查 FEISHU_ALLOWED_USERS 白名单
    # 2. 检查 pairing_store 中已授权用户
    # 3. 未授权用户在 DM 模式会收到 pairing code
```

## 中断机制 (Interrupt)

当用户在新消息到达时 Agent 正在处理前一个请求：

```python
# _handle_message 中
if session_key in self._running_agents:
    # 发送中断信号
    await self._interrupt_agent(event)

# AIAgent 内部
async def _interrupt(self):
    self._should_stop = True
    self._interrupted_event.set()
```

## 相关文件

| 文件 | 职责 |
|------|------|
| `gateway/run.py` | GatewayRunner，消息路由 |
| `gateway/platforms/base.py` | BasePlatformAdapter，MessageEvent |
| `gateway/platforms/feishu.py` | Feishu 特定实现 |
| `gateway/session.py` | SessionStore，会话存储 |
| `run_agent.py` | AIAgent，核心对话循环 |
| `model_tools.py` | 工具发现与分发 |
| `tools/registry.py` | 工具注册表 |
