# LSP (Language Server Protocol) 架构分析

## 一、概述

Claude Code 实现了完整的 Language Server Protocol (LSP) 客户端-服务器管理系统。LSP 服务位于 `src/services/lsp/` 目录，包含 **7 个核心文件**。

### 1.1 核心文件

| 文件 | 职责 |
|------|------|
| `config.ts` | LSP 配置定义与解析 |
| `LSPClient.ts` | JSON-RPC 通信客户端 |
| `LSPServerInstance.ts` | 单个服务器实例管理 |
| `LSPServerManager.ts` | 服务器集群管理器（单例） |
| `manager.ts` | 应用初始化入口 |
| `LSPDiagnosticRegistry.ts` | 诊断异步注册中心 |
| `passiveFeedback.ts` | 通知处理器注册与诊断格式转换 |

## 二、LSP 架构

### 2.1 整体架构

```
LSPServerManager (中枢 - 单例)
    ├── LSPClient (协议通信层)
    ├── LSPServerInstance (服务器实例层, 多个)
    ├── LSPDiagnosticRegistry (诊断注册中心)
    ├── passiveFeedback.ts (被动反馈/诊断处理)
    └── config.ts (配置层)
```

### 2.2 初始化流程

```
应用启动
    → manager.ts: initializeLSPServers()
        → LSPServerManager.getInstance()
            → manager.initLSPServers()
                ├── 解析配置 (config.ts)
                ├── 创建服务器实例 (LSPServerInstance)
                ├── 启动每个实例 (LSPClient)
                └── 注册通知处理器 (passiveFeedback.ts)
```

## 三、配置层

### 3.1 配置接口 (`config.ts`)

| 接口 | 说明 |
|------|------|
| `LSPClientConfig` | 客户端配置（初始化选项、客户端信息、根路径） |
| `LSPServerConfig` | 服务器配置（命令、参数、环境、超时） |
| `LSPConfig` | 顶层配置（服务器映射、全局设置） |

### 3.2 默认值

| 配置 | 默认值 |
|------|--------|
| 启动超时 | 30 秒 |
| 请求超时 | 10 秒 |

### 3.3 服务器发现

**发现方式**：

1. `package.json` 的 `contributes.languageServers` 声明
2. `lspServers` 配置项手动指定

### 3.4 配置解析

| 函数 | 说明 |
|------|------|
| `resolveLSPConfig()` | 从 `.env.json` 解析配置 |
| `resolveLSPServerConfig()` | 根据服务器名解析具体配置 |
| `getServerEnv()` | 构建服务器环境变量 |

## 四、LSP 客户端

### 4.1 核心架构 (`LSPClient.ts`)

```
LSPClient
    ├── ChildProcess (child_process)
    ├── JSON-RPC 2.0 (手动实现)
    ├── pendingRequests Map (请求追踪)
    └── nextRequestId (自增 ID)
```

### 4.2 关键方法

| 方法 | 说明 |
|------|------|
| `start()` | 启动子进程，设置 stdout/stderr 监听 |
| `sendRequest(method, params)` | 发送请求并返回 Promise |
| `sendNotification(method, params)` | 发送通知（无需响应） |
| `initialize(params)` | 发送 LSP initialize 请求 |
| `shutdown()` | 优雅关闭 |
| `kill()` | 强制终止子进程 |

### 4.3 消息处理

```
stdout 按行解析 JSON-RPC 消息
    ├── 有 id 字段 → 请求响应 → 分发到 pendingRequests resolver
    └── 无 id 字段 → 通知 → 分发到 onNotification 回调
```

### 4.4 错误处理

| 场景 | 处理 |
|------|------|
| 进程崩溃 | 清理所有 pending 请求（reject with error） |
| 请求超时 | 自动清理并 reject |
| stderr 输出 | 静默处理（不干扰主流程） |

## 五、LSP 服务器实例

### 5.1 状态管理 (`LSPServerInstance.ts`)

```
LSPServerState 枚举:
    IDLE → STARTING → READY → SHUTTING_DOWN → STOPPED
```

| 方法 | 说明 |
|------|------|
| `isUsable()` | 只有 `READY` 状态才可用 |

### 5.2 核心字段

| 字段 | 说明 |
|------|------|
| `client` | 关联的 LSPClient 实例 |
| `openedDocuments` | 跟踪已打开的文档集合（Set） |
| `onNotification` | 通知回调函数 |
| `serverInfo` | 服务器元信息 |

### 5.3 生命周期

| 方法 | 说明 |
|------|------|
| `start()` | 创建 Client → 启动 → initialize → initialized → READY |
| `stop()` | 状态检查 → client.shutdown() → 清理 |
| `restart()` | stop() + start() |

### 5.4 文档同步

| 方法 | LSP 通知 |
|------|----------|
| `openDocument()` | `textDocument/didOpen` |
| `closeDocument()` | `textDocument/didClose` |
| `updateDocument()` | `textDocument/didChange` (全量更新) |

**自动去重**：已打开的文档不会重复打开。

### 5.5 LSP 能力查询

所有方法都有 try/catch 保护：

| 方法 | LSP 方法 |
|------|----------|
| `getDefinition()` | `textDocument/definition` |
| `getReferences()` | `textDocument/references` |
| `getHover()` | `textDocument/hover` |
| `getDiagnostics()` | `textDocument/diagnostic` (pull model) |
| `getCodeActions()` | `textDocument/codeAction` |
| `getCompletions()` | `textDocument/completion` |
| `getDocumentSymbols()` | `textDocument/documentSymbol` |
| `getWorkspaceSymbols()` | `workspace/symbol` |
| `formatDocument()` | `textDocument/formatting` |
| `rename()` | `textDocument/rename` |

**能力检查**：调用前检查 `serverCapabilities`。

### 5.6 工厂方法

| 方法 | 说明 |
|------|------|
| `createFromConfig()` | 从配置创建实例 |
| `createMultipleFromConfig()` | 批量创建 |

## 六、LSP 服务器管理器

### 6.1 单例模式 (`LSPServerManager.ts`)

```typescript
class LSPServerManager {
  private static instance: LSPServerManager;
  static getInstance(): LSPServerManager;
}
```

### 6.2 核心字段

| 字段 | 说明 |
|------|------|
| `servers` | Map<serverName, LSPServerInstance> |
| `lspConfig` | 当前配置 |
| `managerPromise` | 预创建的 Promise |
| `activeRequests` | 跟踪进行中的请求 |

### 6.3 初始化流程

1. `initLSPServers()`：解析配置 → 创建实例 → 启动 → 注册通知
2. `registerNotificationHandlers()`：调用 `passiveFeedback.ts`
3. 启动 `cleanupTimer`：定期检查超时请求

### 6.4 服务器管理

| 方法 | 说明 |
|------|------|
| `getServer(name)` | 按名称获取 |
| `getAllServers()` | 获取所有服务器（只读 Map） |
| `addServer()` | 添加新服务器 |
| `removeServer()` | 移除并停止服务器 |
| `updateServerConfig()` | 更新配置并重启 |
| `restartAllServers()` | 批量重启 |

### 6.5 等待就绪

| 方法 | 说明 |
|------|------|
| `waitForReady()` | 返回预创建的 Promise，最多 30 秒 |
| `isReady()` | 快速检查所有服务器是否 READY |

### 6.6 广播操作

| 方法 | 说明 |
|------|------|
| `openDocumentInAllServers()` | 在所有服务器中打开文档 |
| `closeDocumentInAllServers()` | 关闭文档 |
| `updateDocumentInAllServers()` | 更新文档内容 |

### 6.7 统一查询接口

| 方法 | 说明 |
|------|------|
| `queryAllServersCapability()` | 广播查询所有服务器的某个能力 |
| `getDiagnosticsForFile()` | 聚合所有服务器对某文件的诊断 |

### 6.8 请求生命周期

| 方法 | 说明 |
|------|------|
| `executeWithTracking()` | 包装请求，记录到 activeRequests |
| `cleanupTimedOutRequests()` | 定期清理超时请求（默认 60 秒） |
| `cancelRequest()` | 主动取消请求 |

### 6.9 优雅关闭

```
shutdown()
    → 清理定时器
    → 并发停止所有服务器
    → 标记已关闭
```

## 七、诊断注册中心

### 7.1 设计模式 (`LSPDiagnosticRegistry.ts`)

借鉴 `AsyncHookRegistry` 的异步附件投递模式。

### 7.2 核心数据结构

```typescript
interface PendingLSPDiagnostic {
  serverName: string;
  files: DiagnosticFile[];
}

pendingDiagnostics: PendingLSPDiagnostic[];
registryLock: AsyncLock; // 并发锁
```

### 7.3 关键方法

| 方法 | 说明 |
|------|------|
| `registerPendingLSPDiagnostic()` | 注册新的诊断到队列 |
| `hasPendingDiagnostics()` | 检查是否有待交付诊断 |
| `consumePendingDiagnostics()` | 原子性地取出所有诊断（清空队列） |
| `clear()` | 清空所有待交付诊断 |

## 八、被动反馈

### 8.1 通知处理 (`passiveFeedback.ts`)

```
registerLSPNotificationHandlers(manager)
    → 遍历所有服务器实例
        → 注册 textDocument/publishDiagnostics 处理器
```

### 8.2 诊断处理流程

```
LSP Server 推送 publishDiagnostics
    → passiveFeedback handler 接收
        → 验证参数结构（uri + diagnostics）
        → formatDiagnosticsForAttachment() 转换格式
        → registerPendingLSPDiagnostic() 注册到队列
        → 跳过空诊断
```

### 8.3 错误隔离

| 机制 | 说明 |
|------|------|
| 独立 try/catch | 每个服务器的 handler 独立处理 |
| 失败追踪 | 连续失败 3 次以上记录警告 |
| 参数验证 | 检查 params 是否为对象、是否包含必需字段 |

### 8.4 格式转换

`formatDiagnosticsForAttachment()` 将 LSP `PublishDiagnosticsParams` 转换为 Claude 的 `DiagnosticFile[]`：

| LSP 严重性 | Claude 级别 |
|------------|-------------|
| 1 | Error |
| 2 | Warning |
| 3 | Info |
| 4 | Hint |

**URI 处理**：支持 `file://` 协议和普通路径，转换失败时 fallback 到原始 URI。

### 8.5 返回结果

```typescript
interface HandlerRegistrationResult {
  totalServers: number;
  successCount: number;
  registrationErrors: Error[];
  diagnosticFailures: Map<string, number>;
}
```

## 九、与主应用的集成

### 9.1 集成点

```
工具调用时
    → manager.openDocumentInAllServers()   // 同步文档
    → server.getDefinition() / getHover()  // 调用 LSP 能力
    → manager.closeDocumentInAllServers()  // 清理文档

诊断异步投递
    → LSP Server 推送 publishDiagnostics
    → passiveFeedback handler 接收
    → 转换为 DiagnosticFile 格式
    → 注册到 LSPDiagnosticRegistry
    → 附件系统 consumePendingDiagnostics() 取出并交付给用户
```

### 9.2 错误处理

- LSP 初始化失败不阻塞主流程
- URI 解析失败 fallback 到原始 URI
- 单个服务器失败不影响其他服务器

## 十、关键设计特点

| 特性 | 实现方式 |
|------|----------|
| **并发安全** | 诊断注册使用 `registryLock` 并发锁 |
| **错误隔离** | 每个服务器独立 try/catch |
| **优雅降级** | LSP 初始化失败不阻塞主流程 |
| **资源管理** | 请求超时自动清理 |
| **状态机** | 服务器生命周期使用明确的状态枚举 |
| **能力检查** | 调用前检查服务器是否声明了对应能力 |
| **异步诊断** | 诊断通过注册中心异步投递 |
| **配置灵活** | 支持环境变量、.env.json、默认配置多层覆盖 |

## 十一、文件索引

| 文件 | 路径 |
|------|------|
| 配置 | `src/services/lsp/config.ts` |
| LSP 客户端 | `src/services/lsp/LSPClient.ts` |
| 服务器实例 | `src/services/lsp/LSPServerInstance.ts` |
| 服务器管理器 | `src/services/lsp/LSPServerManager.ts` |
| 初始化入口 | `src/services/lsp/manager.ts` |
| 诊断注册 | `src/services/lsp/LSPDiagnosticRegistry.ts` |
| 被动反馈 | `src/services/lsp/passiveFeedback.ts` |
