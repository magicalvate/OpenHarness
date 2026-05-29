# OpenHarness 代码结构分析

## 项目定位

OpenHarness 是 Claude Code 的开源 Python 移植版本，核心目标是提供一套完整的 AI Agent 基础设施。包含两个产品：
- **openharness**：核心 agent 框架库（`oh` / `openharness` 命令）
- **ohmo**：基于框架构建的个人 AI 助手，支持 Slack/Telegram/Discord/Feishu

---

## 顶层目录结构

```
OpenHarness/
├── src/openharness/     # 核心库
├── ohmo/                # 个人 agent 应用
├── frontend/            # React + Ink 终端 UI（TypeScript）
├── tests/               # 测试（114+ 用例）
├── scripts/             # E2E 测试脚本
├── docs/                # 文档
└── pyproject.toml       # 入口：oh / openh / openharness / ohmo
```

---

## 核心模块分层

### 1. 入口层 `ui/`

| 文件 | 职责 |
|------|------|
| `app.py` | `run_repl()` / `run_print_mode()` / `run_task_worker()` 三种运行模式 |
| `runtime.py` | 组装所有组件（engine、tools、hooks、memory 等）的工厂函数 |
| `backend_host.py` | JSON-lines 协议宿主，驱动 React TUI 的双向通信 |
| `textual_app.py` | Textual 备用 TUI |
| `protocol.py` | 前后端 JSON-RPC 消息类型定义 |

### 2. 引擎层 `engine/`

核心执行管道：用户输入 → 流式 API 调用 → 解析 tool_call → 执行工具 → 循环

| 文件 | 职责 |
|------|------|
| `query_engine.py` | `QueryEngine` 类，持有对话历史和所有组件 |
| `query.py` | 实际的 agent 循环，处理流式事件、工具执行、上下文压缩、重试 |
| `messages.py` | `ConversationMessage`、`TextBlock`、`ToolUseBlock` 等消息类型 |
| `stream_events.py` | `AssistantTextDelta`、`ToolExecutionStarted` 等流式事件 |
| `cost_tracker.py` | Token 用量追踪 |

### 3. API 层 `api/`

统一抽象多家 LLM 提供商，核心 Protocol 是 `SupportsStreamingMessages`（`client.py:80`）

| 文件 | 职责 |
|------|------|
| `client.py` | `AnthropicApiClient` + `SupportsStreamingMessages` Protocol |
| `openai_client.py` | OpenAI 兼容客户端（格式互转） |
| `copilot_client.py` | GitHub Copilot 客户端 |
| `codex_client.py` | Codex 订阅客户端 |
| `registry.py` | 40+ 提供商元数据（OpenRouter、Ollama、DeepSeek 等） |
| `provider.py` | `detect_provider()` 自动识别提供商 |

### 4. 工具层 `tools/`（45+ 工具）

| 分类 | 工具 |
|------|------|
| 文件操作 | `file_read`, `file_write`, `file_edit`, `glob`, `grep`, `notebook_edit` |
| Shell | `bash` |
| 搜索/Web | `web_search`, `web_fetch`, `tool_search`, `lsp` |
| 任务管理 | `task_create/get/list/update/stop/output` |
| 调度 | `cron_create/list/delete`, `remote_trigger` |
| Agent | `agent_tool`, `send_message` |
| 工作流 | `enter/exit_plan_mode`, `enter/exit_worktree` |
| MCP | `mcp_tool`, `list/read_mcp_resource` |
| 交互 | `ask_user_question`, `config_tool`, `skill_tool` |

`base.py` 定义 `BaseTool` ABC，每个工具有 `name`、`input_model`（Pydantic）、`execute()` 异步方法。

### 5. 权限层 `permissions/`

三种模式：
- **FULL_AUTO**：全部放行
- **CONFIRM**（默认）：敏感操作需用户确认
- **PLAN**：禁止所有写操作（plan mode）

内置敏感路径保护：`.ssh/`、`.aws/`、`.azure/` 等。

### 6. Hooks 系统 `hooks/`

11 个生命周期事件：`SESSION_START/END`、`PRE/POST_TOOL_USE`、`PRE/POST_COMPACT`、`USER_PROMPT_SUBMIT`、`NOTIFICATION`、`STOP`、`SUBAGENT_STOP`

支持 4 种 hook 类型：Shell 命令、HTTP 请求、Prompt（调用 LLM）、Agent 模式。

### 7. 记忆系统 `memory/`

跨会话持久化知识，基于 `MEMORY.md` 文件，支持语义搜索和相关性评分。

### 8. 服务层 `services/`

| 服务 | 职责 |
|------|------|
| `compact/` | 上下文超出时 LLM 自动压缩摘要 |
| `cron.py` | 定时任务调度（croniter） |
| `session_storage.py` | 会话快照保存/恢复 |
| `lsp/` | 语言服务协议集成 |
| `oauth/` | OAuth 认证流程 |

### 9. 多 Agent 协作 `coordinator/` + `swarm/`

- **coordinator**：检测 agent 委托模式，管理 agent 定义
- **swarm**：完整多 agent 编排，含进程内/子进程两种 backend、mailbox 通信、worktree 隔离、权限同步

### 10. MCP 集成 `mcp/`

支持 stdio、HTTP、WebSocket 三种传输方式连接 MCP 服务器。

### 11. Skills 系统 `skills/`

通过 `.md` 文件（YAML frontmatter）按需加载领域知识，兼容 anthropics/skills 生态。

### 12. 插件系统 `plugins/`

支持自定义 agent、工具、hooks，兼容 claude-code 插件格式。

---

## ohmo 个人 Agent（`ohmo/`）

```
ohmo/
├── cli.py              # 入口：memory / soul / user / gateway 命令
├── runtime.py          # 基于 openharness 组装的专用 runtime
├── workspace.py        # ~/.ohmo 工作区（soul.md / user.md / memory/）
├── memory.py           # 专用记忆管理
├── session_storage.py  # 会话后端
└── gateway/            # 多通道网关（Slack / Telegram / Discord / Feishu）
```

---

## 关键架构特征

1. **异步优先**：全链路 `asyncio`
2. **流式驱动**：`StreamEvent` 实时反馈，前后端 JSON-RPC 解耦
3. **依赖注入**：`runtime.py` 统一组装所有组件
4. **多 Provider 抽象**：`SupportsStreamingMessages` Protocol 屏蔽底层差异
5. **上下文自动压缩**：超出 token 阈值时 LLM 自动摘要历史
6. **三层权限**：路径规则 + 命令模式 + 交互确认
7. **插件生态**：skills / plugins / hooks 三套扩展机制并行

---

## 对 ADR-005（Seekseek Platform）的适配评估

### 结论

OpenHarness 覆盖 ADR-005 中 `runtime/` 层的核心骨架，`packages/skills/` 与 `packages/tool-registry/` 高度匹配，基础设施层（storage、gateway、observability）完全超出框架边界。

### 逐层判断

| ADR-005 层 | 适配程度 | 说明 |
|---|---|---|
| `runtime/loop/` | **直接可用** | `engine/query.py` 实现了完整的 Plan-and-Execute / ReAct 循环 |
| `runtime/hooks/` | **直接可用** | 11 个生命周期事件，PreToolUse/PostToolUse 可扩展 |
| `runtime/context/` | **部分可用** | 自动压缩有，session 生命周期管理较轻量 |
| `runtime/dreaming/` | **不存在** | 无异步批处理引擎，需自建 |
| `packages/skills/` + `tool-registry/` | **最契合** | `BaseTool` + `ToolRegistry` 与 ADR-004 工具体系同构 |
| `packages/ai_providers/llm/` | **部分覆盖** | 40+ provider registry，但无熔断/重试机制 |
| `packages/ai_providers/asr/tts/` | **不覆盖** | voice/ 目录极轻量，需自建 |
| `packages/llm_router/` | **不覆盖** | 仅有提供商识别，无动态路由 |
| `packages/ai_capability/memory/`（三层） | **差距最大** | 基于 MEMORY.md 文件，无向量 DB，三层记忆需完全重建 |
| `packages/storage/` | **完全空白** | 无 PG/Redis/ES/Vector/OSS 封装 |
| `packages/api-guard/` | **完全空白** | 无 JWT/OAuth/租户配额 |
| `packages/observability/` | **完全空白** | 无分布式 tracing SDK |
| `gateway/` | **超出边界** | 框架为本地 TUI 设计，非生产级 API Gateway |
| `agent-configs/` → LangGraph | **设计理念不同** | OpenHarness 代码驱动，非声明式编译执行图 |

### 建议

- **直接借鉴**：`engine/`（agent loop）→ `runtime/loop/`；`hooks/`（生命周期）→ `runtime/hooks/`；`tools/base.py` → `packages/tool-registry/`
- **需自建**：Storage 层、ASR/TTS vendor 封装 + circuit breaker、三层记忆、LangGraph pipeline、Gateway、OpenTelemetry SDK

---

## 录音卡多用户场景下的 Agent 架构结论

### 场景描述

用户使用录音卡硬件录音，音频上传云端实时 ASR 转写，每攒够 N 个段落触发一次 agent 处理（摘要/实体提取），同时用户可随时追问。多个用户并发使用。

### 结论：Session Pool 设计足以承载 agent 层

「多用户」问题分散在三层，session pool 只负责中间层：

| 层 | 多用户问题 | 解法 |
|---|---|---|
| `gateway/ws/` | 鉴权、连接管理、限流 | api-guard（JWT + 配额） |
| `runtime/` session pool | 多用户 agent context 并发隔离 | session_key 隔离 + asyncio |
| `storage/` | 多用户数据不互串 | DB 层按 user_id 分区 |

### 为什么 session pool 够用

录音场景负载特征适合 asyncio session pool：
- ASR 是外部服务，不占 agent 进程 CPU
- 每个 session 大部分时间在等待（批处理触发间隔 + 用户无追问）
- LLM 调用是 I/O bound，asyncio 并发效率高
- 单进程撑不住时横向加实例 + sticky routing，不需要换架构

### 需要额外设计的部分

session pool 解决不了、也不该解决的：
- **api-guard**：用户认证 + 配额限制（gateway 层）
- **storage user_id 分区**：数据持久化隔离（DB 层）
- **优先级队列**：用户追问优先于批处理触发（session 内部）
- **auto-compact**：长录音 context 持续增长，需压缩控制窗口大小

三层合在一起才是完整的多用户方案，缺任何一层都不完整。
