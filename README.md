# Claude-Code-Agent

从零实现的本地 Claude Code Agent 系统（mini 版）——一个真正能跑任务、能调工具、能看过程、能管权限、能续上下文、能扩展生态的本地 Agent 运行时。

本项目把 Claude Code 这类 AI 编程 Agent 最核心的运行机制完整拆解并工程化实现：不是"调用一次大模型"的 Demo，而是一套具备守护进程、事件流、权限审批、上下文治理、多客户端架构的完整 Agent 运行时。

## 核心特性

- **ReAct Agent Loop**：用户输入目标 → Agent 自主规划 → 模型思考 → 工具调用 → 结果回填 → 多步执行闭环
- **工具调用安全**：工具调用不直接裸跑，先经 `ToolRegistry` 参数校验 + `PermissionManager` 权限审批 + 失败分类与重试
- **事件流外化**：执行过程不是黑盒，通过 `EventBus` 实时展示到 TUI，events/trace 可持久化、可回放
- **分层会话记忆**：session / thread / notes / context 四层上下文管理，多轮会话可持续
- **上下文治理**：长会话下有 context 水位检测、tool_result 截断、自动/手动 compact 压缩
- **扩展生态**：支持 Skills（工作流）、Subagents（子 Agent）、MCP（外部工具接入）
- **守护进程架构**：`cca-core` daemon + CLI/TUI 多客户端，任务不依赖客户端存活
- **类型化 IPC**：JSON-RPC 2.0 + NDJSON 的类型化通信协议，命令/响应/事件全链路可追踪

## 架构

Claude-Code-Agent 的核心不是一段 prompt，而是一条完整的本地 Agent 运行链路：

```text
用户目标
  → CLI / TUI
  → JSON-RPC over NDJSON
  → cca-core daemon
  → AgentRunner
  → AgentLoop
  → LLM Provider
  → ToolRegistry
  → PermissionManager
  → EventBus
  → Session Store
  → TUI 实时渲染 / events.jsonl 持久化 / trace 回放
```

用户不是直接和一个脚本对话，而是通过 `cca` CLI 或 `cca-tui` 连接到常驻的 `cca-core` 守护进程。真正执行任务的是 Core daemon，CLI 和 TUI 只是客户端——TUI 崩溃不影响 Agent 任务，后续可同时接入 Web 前端。

## 快速开始

```bash
# 1. 克隆项目
git clone https://github.com/gezixuan721-sys/Claude-Code-Agent.git
cd Claude-Code-Agent

# 2. 配置环境变量
cp .env.example .env
# 编辑 .env：填入 LLM API Key（如 ANTHROPIC_API_KEY）

# 3. 安装依赖（Python 3.12）
pip install -e .

# 4. 启动 Core 守护进程
cca-core

# 5. 使用 CLI / TUI
cca --help          # CLI 客户端
cca-tui             # TUI 客户端（实时事件流、工具调用、权限审批）
```

常用环境变量（见 `.env.example`）：

| 变量 | 说明 |
| --- | --- |
| `CCA_HOST` / `CCA_PORT` | Core daemon 监听地址（默认 127.0.0.1:7437） |
| `CCA_LOG_LEVEL` | 日志级别 |
| `CCA_LOG_FILE` | 日志文件路径 |
| `CCA_LOG_FORMAT` | `text`（人工可读）或 `json`（结构化） |
| `CCA_LLM_DEFAULT_MODEL` | 默认 LLM 模型 |
| `CCA_MAX_STEPS` | Agent 最大执行步数 |

## 项目演进（S0 → S7）

项目按 8 个阶段逐步搭建，每一阶段解决一个真实的 Agent 工程问题：

| 阶段 | 主题 | 解决的问题 |
| --- | --- | --- |
| S0 | 骨架与协议契约 | CLI 和 daemon 通过真实 IPC 完成一次 ping/pong |
| S1 | Agent 最小闭环 | 一次 `cca run` 从 goal 到 LLM、工具、事件文件完整跑通 |
| S2 | 事件流外化 | AgentRunner 搬进 daemon，CLI/TUI 通过 IPC 订阅同一份事件流 |
| S3 | 自主规划与 TUI | Agent 能用任务工具拆解复杂目标，TUI 展示完整执行过程 |
| Trace | 系统级时间线 | IPC / EventBus / LLM 三层数据流可追踪、可回放 |
| S4 | 会话与记忆 | 多轮 run 进入同一个 session，thread 和 notes 接住上下文 |
| S5 | 工具安全 | 工具调用前有参数校验、权限审批、失败分类和重试 |
| S6 | 上下文治理 | 长会话下有 context 水位、tool_result 截断和 compact |
| S7 | 扩展边界 | Skills、Subagents、MCP 让 Agent 可组织、可派生、可接外部工具 |

## 目录结构

```text
src/claude_code_agent/
├── cli/            # CLI 客户端（cca 命令）
├── tui/            # TUI 客户端（cca-tui）
├── core/
│   ├── app.py      # Core daemon 入口（cca-core）
│   ├── agents/     # Agent 定义与加载
│   ├── bus/        # 命令/事件/信封模型
│   ├── llm/        # LLM Provider 抽象
│   ├── mcp/        # MCP 客户端/服务端/工具
│   ├── memory/     # 记忆与上下文加载
│   ├── permissions/# 工具权限审批
│   ├── session/    # 会话管理（session/thread/notes）
│   ├── skills/     # Skills 工作流
│   ├── subagent/   # 子 Agent
│   ├── task/       # 任务管理
│   ├── tools/      # 工具注册与内置工具
│   ├── trace/      # trace 记录与回放
│   └── transport/  # IPC 传输（socket/广播）
tests/              # pytest 单元 + 集成测试（mypy strict + ruff）
docs/               # 文档与架构图
```

## 技术栈与质量

- Python 3.12，PEP 621 + Hatch 构建
- pytest、mypy strict、ruff 保证代码质量
- TCP NDJSON + JSON-RPC 2.0 类型化 IPC 协议

## 许可证

MIT License。保留原作者版权声明（详见 [LICENSE](LICENSE)）。
