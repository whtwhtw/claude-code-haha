# Claude Code 项目架构分析

## 一、项目概述

**Claude Code Haha** 是一个基于 Claude Code 泄露源码修复的本地可运行版本，支持接入任意 Anthropic 兼容的 API 端点（如 MiniMax、OpenRouter 等）。项目使用 **Bun** 作为运行时，采用 **TypeScript + React + Ink** 技术栈构建终端 UI 应用。

### 1.1 技术栈总览

| 类别 | 技术选型 |
|------|----------|
| 运行时 | Bun |
| 语言 | TypeScript |
| 终端 UI | React + Ink |
| CLI 解析 | Commander.js |
| 数据验证 | Zod |
| HTTP 客户端 | Axios |
| MCP 协议 | @modelcontextprotocol/sdk |
| 可观测性 | OpenTelemetry |
| A/B 测试 | GrowthBook |

### 1.2 项目规模

- **总计约 1916 个源文件，305 个目录**
- **核心模块分布**：

| 目录 | 文件数 | 职责 |
|------|--------|------|
| `utils/` | 566 | 工具函数库（最大模块） |
| `components/` | 389 | UI 组件（Ink/React 组件） |
| `commands/` | 207 | CLI 命令实现 |
| `tools/` | 187 | 工具实现（Bash、文件操作等） |
| `services/` | 130 | 业务服务层 |
| `hooks/` | 104 | React/Ink Hooks |
| `ink/` | 96 | Ink 终端框架相关 |

## 二、整体架构

### 2.1 架构分层

```
CLI Entry (入口层)
    ├── Terminal UI (终端界面 - Ink/React)
    │       ├── Query Engine (查询引擎)
    │       │       └── Tool System (工具系统 - 30+ Tools)
    │       │               └── Agent/Task (智能体系统)
    │       └── Init/Bootstrap (初始化)
    │               ├── State (状态管理)
    │               ├── Plugin/Skill (扩展系统)
    │               └── External Integrations (外部集成)
    └── Services (服务层: MCP/OAuth/Memory)
```

### 2.2 关键文件路径

| 路径 | 职责 |
|------|------|
| `bin/claude-haha` | CLI 入口脚本 |
| `src/entrypoints/cli.tsx` | CLI 主入口（快速路径路由） |
| `src/main.tsx` | TUI 主逻辑 |
| `src/query.ts` | 核心查询引擎（请求循环） |
| `preload.ts` | Bun preload（设置 MACRO 全局变量） |

### 2.3 快速路径机制

`cli.tsx` 实现了**零模块加载**的快速路径机制，在加载完整 CLI 之前优先处理特殊标志：

- `--version/-v` → 直接输出版本号
- `--dump-system-prompt` → 输出系统提示并退出
- `--claude-in-chrome-mcp` → 启动 Chrome MCP 服务器
- `--daemon-worker=<kind>` → 启动 daemon worker
- `remote-control/rc/remote/sync/bridge` → Bridge 远程模式
- `daemon` → 守护进程
- `ps/logs/attach/kill/--bg` → 后台会话管理
- `new/list/reply` → 模板作业
- `--tmux --worktree` → tmux 工作树
- `--bare` → 简单模式

这种设计确保了常见操作的启动速度。

## 三、模块索引

| 模块 | 说明 | 文档位置 |
|------|------|----------|
| [MCP](./02-mcp-architecture.md) | Model Context Protocol 集成 | 02-mcp-architecture.md |
| [OAuth](./03-oauth-architecture.md) | 认证系统 | 03-oauth-architecture.md |
| [Analytics](./04-analytics-architecture.md) | 分析与遥测 | 04-analytics-architecture.md |
| [LSP](./05-lsp-architecture.md) | Language Server Protocol | 05-lsp-architecture.md |
| [Plugin](./06-plugin-architecture.md) | 插件系统 | 06-plugin-architecture.md |
| [Compact](./07-compact-architecture.md) | 上下文压缩 | 07-compact-architecture.md |
| [Memory](./08-memory-architecture.md) | 记忆系统 | 08-memory-architecture.md |
| [API Client](./09-api-client-architecture.md) | API 客户端 | 09-api-client-architecture.md |
| [Request Lifecycle](./10-request-lifecycle.md) | 请求生命周期与工具系统 | 10-request-lifecycle.md |
| [Terminal UI](./11-terminal-ui-architecture.md) | 终端 UI 架构 | 11-terminal-ui-architecture.md |
| [Permission & Security](./12-permission-security.md) | 权限安全模型 | 12-permission-security.md |
| [Multi-Agent](./13-multi-agent-orchestration.md) | 多智能体编排 | 13-multi-agent-orchestration.md |
| [State & Data Flow](./14-state-data-flow.md) | 状态管理与数据流 | 14-state-data-flow.md |

## 四、服务层总览

服务层 (`src/services/`) 包含 36 项（20 个子目录 + 16 个文件），是连接 Query Engine 与外部系统的桥梁：

```
Query Engine (查询引擎)
    ├── MCP 集成 (7 transports)
    ├── Memory 记忆系统
    ├── OAuth 认证
    ├── API Client
    ├── Analytics 分析
    ├── Compact 压缩 (3层)
    ├── Plugin 插件
    └── LSP
```

### 4.1 主要服务目录

| 服务目录 | 文件数 | 职责 |
|----------|--------|------|
| `analytics/` | 9 | 事件追踪、Datadog、GrowthBook |
| `mcp/` | 23 | MCP 连接管理、认证、传输 |
| `oauth/` | 5 | OAuth 2.0 + PKCE 认证 |
| `lsp/` | 7 | LSP 服务器管理 |
| `compact/` | 11 | 上下文压缩 |
| `api/` | 20 | API 客户端、重试、错误处理 |
| `plugins/` | 3 | 插件安装管理 |
| `SessionMemory/` | 3 | 会话记忆 |
| `extractMemories/` | 2 | 记忆提取 |

## 五、运行时环境

### 5.1 启动流程

```
bin/claude-haha
    → src/entrypoints/cli.tsx (快速路径路由)
        → src/main.tsx (TUI 主逻辑)
            → src/setup.ts (初始化)
                → src/screens/REPL.tsx (交互界面)
                    → src/query.ts (查询循环)
```

### 5.2 环境变量

| 变量 | 说明 |
|------|------|
| `ANTHROPIC_AUTH_TOKEN` | Bearer 认证令牌 |
| `ANTHROPIC_BASE_URL` | 自定义 API 端点 |
| `ANTHROPIC_MODEL` | 默认模型 |
| `API_TIMEOUT_MS` | API 超时（毫秒） |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 禁用非必要网络流量 |
| `DISABLE_TELEMETRY` | 禁用遥测 |

### 5.3 支持的后端

| Provider | 环境变量 | SDK |
|----------|----------|-----|
| firstParty (默认) | 无 (或 ANTHROPIC_BASE_URL) | @anthropic-ai/sdk |
| Bedrock | CLAUDE_CODE_USE_BEDROCK=1 | @anthropic-ai/bedrock-sdk |
| Vertex AI | CLAUDE_CODE_USE_VERTEX=1 | @anthropic-ai/vertex-sdk |
| Foundry (Azure) | CLAUDE_CODE_USE_FOUNDRY=1 | @anthropic-ai/foundry-sdk |
