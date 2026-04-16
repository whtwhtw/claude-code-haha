# 请求生命周期与工具系统架构分析

## 一、请求生命周期

### 1.1 整体流程

```
User Input (用户输入)
    → Message Parse (消息解析)
        → Context Assembly (上下文组装)
            → API Client (Streaming) (API 调用)
                → Tool Detection (工具检测)
                    → Permission Check (权限校验)
                        → Tool Execute (工具执行)
                            → Ink Render (终端渲染)
                                ↖ Multi-turn (多轮) ↙
```

### 1.2 多轮循环

请求生命周期是一个**多轮循环**，每一轮包含：

1. 用户输入 → 消息解析
2. 上下文组装（包含历史、记忆、系统提示）
3. API 调用（流式响应）
4. 工具检测（模型返回 tool_use）
5. 权限校验（用户确认或自动允许）
6. 工具执行（调用实际工具）
7. 终端渲染（展示结果）
8. 回到步骤 3，继续多轮直到完成

### 1.3 查询引擎核心

`src/query.ts` 实现了核心查询循环 `queryLoop()`：

```typescript
async function* queryLoop(params, consumedCommandUuids): AsyncGenerator<...> {
  // 可变状态
  let state: State = {
    messages,
    toolUseContext,
    autoCompactTracking,
    maxOutputTokensRecoveryCount,
    ...
  };

  while (true) {
    // 1. 消息处理（micro-compact, snip, collapse, auto-compact）
    // 2. 上下文组装
    // 3. API 调用（流式）
    // 4. 工具执行（流式工具执行器）
    // 5. 状态更新
    // 6. continue 或 return
  }
}
```

### 1.4 关键状态

| 状态字段 | 说明 |
|----------|------|
| `messages` | 消息历史 |
| `toolUseContext` | 工具使用上下文 |
| `autoCompactTracking` | 自动压缩追踪状态 |
| `maxOutputTokensRecoveryCount` | 最大输出 token 恢复计数 |
| `hasAttemptedReactiveCompact` | 是否已尝试反应式压缩 |
| `turnCount` | 轮次计数 |
| `transition` | 上一轮继续原因 |

### 1.5 查询配置

`buildQueryConfig()` 在入口快照不可变环境状态：

- 功能标志（feature gates）
- 会话 ID
- 模型配置
- 工具选项

### 1.6 预算追踪

| 预算类型 | 说明 |
|----------|------|
| Token Budget | 每轮 token 预算（+500k 自动继续） |
| Task Budget | API `task_budget`（整个 agent turn 的预算） |

### 1.7 错误恢复

| 错误 | 恢复策略 |
|------|----------|
| Prompt too long | 反应式压缩重试 |
| Max output tokens | 最多 3 次恢复 |
| 529 超载 | 模型回退（Opus → Sonnet） |
| 401 认证 | OAuth 令牌刷新 |

## 二、工具系统架构

### 2.1 工具注册中心

```
Tool Registry (工具注册中心)
    ├── File Tools (文件工具)
    │       Read/Edit/Write/Glob/Grep
    ├── Shell Tools (命令工具)
    │       Bash/PowerShell/REPL
    ├── System Tools (系统工具)
    │       Process/Env/Info/Config
    ├── Agent Tools (智能体工具)
    │       Agent/Task
    ├── External (外部)
    │       WebSearch/WebFetch/MCP/LSP
    └── Communication (通信)
            AskUser/SendMessage
```

### 2.2 工具执行流程

```
Input → Validate → Permission Gate → Sandbox → Execute → Render
         验证        权限网关           沙箱       执行       渲染
```

### 2.3 工具分类

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件工具** | Read, Edit, Write, Glob, Grep | 文件读写与搜索 |
| **命令工具** | Bash, PowerShell, REPL | 命令执行 |
| **系统工具** | Process, Env, Info, Config | 系统信息 |
| **智能体工具** | Agent, Task | 子代理与任务 |
| **外部工具** | WebSearch, WebFetch, MCP, LSP | 外部服务集成 |
| **通信工具** | AskUser, SendMessage | 用户交互 |

### 2.4 工具实现

工具位于 `src/tools/` 目录（187 个文件），主要子目录：

| 目录 | 说明 |
|------|------|
| `BashTool/` | Bash 命令执行 |
| `FileEditTool/` | 文件编辑 |
| `FileReadTool/` | 文件读取 |
| `FileWriteTool/` | 文件写入 |
| `GlobTool/` | 文件模式匹配 |
| `GrepTool/` | 内容搜索 |
| `WebSearchTool/` | 网页搜索 |
| `WebFetchTool/` | 网页获取 |
| `NotebookTool/` | Jupyter Notebook |
| `RipgrepTool/` | ripgrep 搜索 |

### 2.5 流式工具执行

`StreamingToolExecutor` (`src/services/tools/StreamingToolExecutor.ts`)：

```
API 流式响应
    → 检测到 tool_use 块
        → 流式工具执行器
            → 权限检查
            → 工具执行
            → 结果返回
                → 继续 API 流式
```

### 2.6 工具编排

`runTools()` (`src/services/tools/toolOrchestration.ts`)：

- 并发工具执行
- 结果聚合
- 错误处理

### 2.7 工具结果预算

`applyToolResultBudget()` (`src/utils/toolResultStorage.ts`)：

- 限制工具结果大小
- 在 micro-compact 前执行
- 缓存微压缩按 tool_use_id 操作（不检查内容）

## 三、工具系统详细分析

### 3.1 文件工具

| 工具 | 功能 |
|------|------|
| **Read** | 读取文件内容 |
| **Edit** | 精确编辑文件（查找替换） |
| **Write** | 写入新文件 |
| **Glob** | 文件模式匹配查找 |
| **Grep** | 文件内容搜索 |

### 3.2 命令工具

| 工具 | 功能 |
|------|------|
| **Bash** | 执行 Bash 命令 |
| **PowerShell** | 执行 PowerShell 命令 |
| **REPL** | 交互式命令行 |

### 3.3 智能体工具

| 工具 | 功能 |
|------|------|
| **Agent** | 创建子代理执行任务 |
| **Task** | 任务分配与追踪 |

### 3.4 外部工具

| 工具 | 功能 |
|------|------|
| **WebSearch** | 网页搜索（Exa/Google） |
| **WebFetch** | 获取网页内容 |
| **MCP** | Model Context Protocol 工具 |
| **LSP** | Language Server Protocol 工具 |

### 3.5 通信工具

| 工具 | 功能 |
|------|------|
| **AskUser** | 询问用户获取信息 |
| **SendMessage** | 发送消息给其他代理 |

## 四、工具执行细节

### 4.1 权限网关

工具执行前经过权限检查：

```
Tool Request → Hook → Permission Gate → Allow/Deny
                  钩子      权限网关      允许/拒绝
```

**权限模式**：

| 模式 | 说明 |
|------|------|
| **Ask Mode** | 每次询问用户 |
| **Auto Mode** | 自动允许（白名单） |
| **Bypass Mode** | 旁路模式（跳过检查） |

### 4.2 沙箱系统

沙箱提供多层隔离：

| 层级 | 说明 |
|------|------|
| Isolated Environment | 隔离环境 |
| Resource Limits | 资源限制 |
| Network Policy | 网络策略 |
| Process Namespace | 进程命名空间 |

### 4.3 Bash 安全 (5 层)

| 层 | 说明 |
|-----|------|
| **Validation** | 命令验证 |
| **Sanitization** | 命令清理 |
| **Restriction** | 命令限制 |
| **Monitoring** | 执行监控 |
| **Audit** | 审计日志 |

### 4.4 工具结果渲染

工具执行结果通过 Ink 渲染到终端：

- 结构化输出
- 差异对比 (Diff)
- 错误高亮
- 进度指示

## 五、文件索引

| 文件/目录 | 路径 |
|-----------|------|
| 查询引擎 | `src/query.ts` |
| 工具注册 | `src/Tool.ts`, `src/tools.ts` |
| 工具目录 | `src/tools/` |
| 流式工具执行器 | `src/services/tools/StreamingToolExecutor.ts` |
| 工具编排 | `src/services/tools/toolOrchestration.ts` |
| 工具结果预算 | `src/utils/toolResultStorage.ts` |
| 权限系统 | `src/utils/permissions/` |
| 沙箱 | `src/utils/sandbox/` |
| Hook 系统 | `src/utils/hooks/` |
