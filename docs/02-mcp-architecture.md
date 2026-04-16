# MCP (Model Context Protocol) 架构分析

## 一、概述

MCP (Model Context Protocol) 是 Claude Code 用于连接外部工具和服务的标准化协议。项目 `src/services/mcp/` 目录包含 **23 个文件**，实现了完整的 MCP 客户端功能。

### 1.1 核心文件

| 文件 | 职责 |
|------|------|
| `config.ts` | MCP 配置定义与解析 |
| `client.ts` | MCP 客户端核心逻辑 |
| `MCPConnectionManager.tsx` | 连接管理器（React 组件） |
| `auth.ts` | MCP 认证处理 |
| `InProcessTransport.ts` | 进程内传输 |
| `SdkControlTransport.ts` | SDK 控制传输 |
| `types.ts` | 类型定义 |
| `channelAllowlist.ts` | 频道白名单 |
| `channelPermissions.ts` | 频道权限 |
| `oauthPort.ts` | OAuth 端口分配 |
| `xaa.ts` | XAA 代理 |
| `xaaIdpLogin.ts` | 企业 IdP OIDC 登录 |
| `elicitationHandler.ts` | 唤起处理器 |
| `normalization.ts` | 标准化 |

## 二、MCP 架构

### 2.1 传输层 (Transports)

MCP 支持 **7 种传输方式**：

| 传输类型 | 说明 | 文件 |
|----------|------|------|
| **Stdio** | 标准输入/输出（进程通信） | 默认传输 |
| **In-Process** | 进程内传输（同进程） | `InProcessTransport.ts` |
| **SDK Control** | SDK 控制传输 | `SdkControlTransport.ts` |
| **HTTP** | HTTP 传输（远程 MCP 服务器） | `client.ts` |
| **WebSocket** | WebSocket 传输 | `client.ts` |
| **SSE** | Server-Sent Events | `client.ts` |
| **Claude.ai Proxy** | 通过 claude.ai 代理 | `auth.ts` |

### 2.2 架构组件

```
MCPConnectionManager (连接管理器)
    ├── MCPClient (客户端)
    │       ├── StdioTransport
    │       ├── InProcessTransport
    │       ├── SdkControlTransport
    │       └── HTTP/WebSocket/SSE Transports
    ├── AuthHandler (认证处理)
    │       ├── OAuth (AccessToken)
    │       └── API Key
    ├── ChannelManager (频道管理)
    │       ├── Allowlist (白名单)
    │       └── Permissions (权限)
    └── ServerRegistry (服务器注册表)
```

### 2.3 连接管理

`MCPConnectionManager.tsx` 负责：

1. **连接生命周期**：建立、维护、断开 MCP 服务器连接
2. **重连策略**：自动重连、指数退避
3. **健康检查**：定期检查连接状态
4. **资源清理**：优雅关闭、资源释放

## 三、认证机制

### 3.1 认证方式

MCP 服务器支持两种认证方式：

| 方式 | 说明 |
|------|------|
| **OAuth 2.0** | 通过 OAuth 令牌访问 MCP 代理 (`mcp-proxy.anthropic.com`) |
| **API Key** | 使用 API Key 直接认证 |

### 3.2 OAuth 集成

- **端口管理** (`oauthPort.ts`)：
  - Windows: `39152-49151`（避开系统保留端口）
  - 其他平台: `49152-65535`（IANA 动态端口）
  - 回退端口: `3118`
  - 支持 `MCP_OAUTH_CALLBACK_PORT` 环境变量

- **自动重试**：`withOAuth401Retry` 包装请求，401 时自动刷新令牌

### 3.3 XAA (External Agent Authorization)

企业级 IdP 集成 (`xaaIdpLogin.ts`)：

```
OIDC Discovery → PKCE + authorization_code → id_token 缓存
```

- **OIDC Discovery**：从 `{issuer}/.well-known/openid-configuration` 获取元数据
- **id_token 缓存**：使用 `secureStorage` 存储，键为 `mcpXaaIdp[issuerKey]`
- **客户端密钥**：可选 `idpClientSecret`（保密客户端）

## 四、服务器管理

### 4.1 服务器配置

MCP 服务器配置来源：

1. **用户配置**：`~/.claude/settings.json` 中的 `mcpServers`
2. **项目配置**：项目根目录的 `.claude/settings.json`
3. **插件配置**：插件 manifest 中声明的 MCP 服务器
4. **远程管理**：组织策略通过 `managed-settings.json` 下发

### 4.2 服务器生命周期

```
配置解析 → 验证 → 启动连接 → 能力协商 → 工具注册 → 运行中 → 断开连接
```

### 4.3 工具注册

MCP 服务器提供的工具通过以下流程注册到工具系统：

1. 连接建立后，调用 `tools/list` 获取工具列表
2. 将 MCP 工具转换为内部 `Tool` 格式
3. 注册到 `ToolRegistry`
4. 工具调用时路由到对应 MCP 服务器

## 五、频道系统

### 5.1 频道类型

| 类型 | 说明 |
|------|------|
| **Stdio Channel** | 标准输入/输出频道 |
| **HTTP Channel** | HTTP 请求/响应频道 |
| **Notification Channel** | 通知频道（服务器推送） |

### 5.2 权限控制

`channelPermissions.ts` 定义频道级别的权限：

- **读取权限**：允许从频道读取数据
- **写入权限**：允许向频道写入数据
- **管理权限**：允许修改频道配置

### 5.3 白名单机制

`channelAllowlist.ts` 实现白名单过滤：

- 只允许白名单中的频道建立连接
- 防止未授权的外部服务接入
- 支持域名和 IP 白名单

## 六、In-Process 传输

`InProcessTransport.ts` 实现进程内 MCP 通信：

- **用途**：同进程内的 MCP 服务器（如插件提供的 MCP 工具）
- **优势**：无进程间通信开销
- **隔离**：通过命名空间隔离不同服务器

## 七、SDK 控制传输

`SdkControlTransport.ts` 用于 SDK 模式：

- **场景**：Claude Code 作为 SDK 被其他应用调用时
- **功能**：外部应用通过此传输控制 MCP 连接
- **权限**：受 SDK 调用方权限约束

## 八、与工具系统集成

MCP 工具与内部工具的集成流程：

```
MCP 工具请求
    → MCPClient.sendRequest(method="tools/call")
        → Transport.send(payload)
            → MCP 服务器执行
                → Transport.receive(result)
                    → MCPClient.parseResponse()
                        → ToolResult → ToolRegistry
```

### 8.1 错误处理

- **连接错误**：自动重连、降级
- **超时错误**：请求超时、重试
- **认证错误**：401 自动刷新、403 清除令牌
- **协议错误**：JSON-RPC 错误码映射

## 九、关键设计模式

| 模式 | 应用 |
|------|------|
| **传输抽象** | 统一 Transport 接口，支持多种底层协议 |
| **连接池** | 复用已建立的 MCP 连接 |
| **请求追踪** | JSON-RPC request ID 匹配响应 |
| **事件驱动** | 通知异步分发 |
| **单例** | MCPConnectionManager 全局唯一实例 |

## 十、文件索引

| 文件 | 关键导出/功能 |
|------|---------------|
| `config.ts` | MCPConfig 接口、配置解析 |
| `client.ts` | MCPClient 类、请求发送 |
| `MCPConnectionManager.tsx` | 连接管理 React 组件 |
| `auth.ts` | 认证处理、令牌管理 |
| `InProcessTransport.ts` | 进程内传输实现 |
| `SdkControlTransport.ts` | SDK 控制传输实现 |
| `types.ts` | MCP 类型定义 |
| `oauthPort.ts` | OAuth 回调端口分配 |
| `xaaIdpLogin.ts` | 企业 IdP OIDC 登录 |
