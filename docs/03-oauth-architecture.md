# OAuth 认证系统架构分析

## 一、概述

Claude Code 实现了两套独立的 OAuth 体系：

1. **Claude.ai OAuth**：面向最终用户的身份认证
2. **XAA (External Agent Authorization) IdP**：企业级 OIDC 集成

OAuth 服务位于 `src/services/oauth/` 目录，包含 **5 个核心文件**。

### 1.1 核心文件

| 文件 | 职责 |
|------|------|
| `index.ts` | OAuthService 类，完整流程编排 |
| `client.ts` | 令牌交换、刷新、profile 获取、角色管理 |
| `auth-code-listener.ts` | 本地 HTTP 回调监听器 |
| `crypto.ts` | PKCE code_verifier/challenge/state 生成 |
| `getOauthProfile.ts` | API key / OAuth token 两种方式的 profile 查询 |

### 1.2 配置文件

| 文件 | 职责 |
|------|------|
| `src/constants/oauth.ts` | OAuth 环境配置（prod/staging/local/custom） |
| `src/services/mcp/oauthPort.ts` | MCP OAuth 端口分配 |
| `src/services/mcp/xaaIdpLogin.ts` | 企业 IdP OIDC 登录 + id_token 缓存 |

## 二、OAuth 架构

### 2.1 整体流程

```
用户触发登录
    → OAuthService.startOAuthFlow()
        → 生成 PKCE 参数 (code_verifier, code_challenge, state)
        → 构建授权 URL
        → AuthCodeListener 启动本地 HTTP 服务器
        → 打开浏览器 → 用户授权
        → 回调接收授权码
        → 交换令牌 (access_token + refresh_token)
        → 获取 Profile 信息
        → 重定向到成功页面
        → 返回 OAuthTokens
```

### 2.2 环境配置

| 环境 | OAuth 端点 | CLIENT_ID |
|------|------------|-----------|
| **Production** | `platform.claude.com` / `claude.com` | `9d1c250a-e61b-44d9-88ed-5944d1962f5e` |
| **Staging** | 仅 `USER_TYPE=ant` 时可启用 | - |
| **Local** | `localhost:8000` (API), `:4000` (claude-ai) | - |
| **Custom** | `CLAUDE_CODE_CUSTOM_OAUTH_URL` | 仅限白名单端点 |

## 三、PKCE 实现

### 3.1 密码学流程 (`crypto.ts`)

```
code_verifier = base64url(randomBytes(32))    // 43 字符
code_challenge = base64url(SHA256(verifier))  // 43 字符
state = base64url(randomBytes(32))            // CSRF 防护
```

- **方法**：S256（SHA-256 哈希）
- **编码**：Base64URL（`+` → `-`, `/` → `_`, 移除 `=` 填充）
- **安全**：每次流程生成新的随机值

### 3.2 Base64URL 编码

```typescript
function base64URLEncode(buffer: Buffer): string {
  return buffer.toString('base64')
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
}
```

## 四、授权码监听器

### 4.1 工作原理 (`auth-code-listener.ts`)

```
本地 HTTP 服务器
    → 监听 localhost:{port}/callback
    → 端口可选指定或由 OS 自动分配（listen(0)）
    → 验证 state 参数防止 CSRF
    → 收到授权码后存储 ServerResponse 对象
    → 后续用于重定向到成功页面
```

### 4.2 关键方法

| 方法 | 说明 |
|------|------|
| `start(port?)` | 启动监听，返回实际端口号 |
| `waitForAuthorization(state, onReady)` | 等待授权码，返回 Promise |
| `handleSuccessRedirect(scopes, customHandler?)` | 根据 scopes 重定向 |
| `close()` | 清理资源 |

### 4.3 安全设计

- **CSRF 防护**：验证 `state` 参数
- **端口随机化**：降低端口预测风险
- **超时机制**：防止无限等待

## 五、OAuth 客户端核心

### 5.1 主要功能 (`client.ts`)

| 功能 | 说明 |
|------|------|
| **buildAuthUrl** | 构建授权 URL |
| **exchangeCodeForTokens** | 授权码交换令牌 |
| **refreshOAuthToken** | 令牌刷新 |
| **fetchProfileInfo** | 获取用户 Profile |
| **fetchAndStoreUserRoles** | 获取用户角色 |
| **createAndStoreApiKey** | 创建 API Key |
| **isOAuthTokenExpired** | 令牌过期检查 |
| **populateOAuthAccountInfoIfNeeded** | 账户信息缓存 |

### 5.2 授权 URL 构建

支持参数：

| 参数 | 说明 |
|------|------|
| `orgUUID` | 组织绑定 |
| `login_hint` | 预填邮箱 |
| `login_method` | 指定登录方式（如 SSO） |
| `inferenceOnly` | 仅请求 `user:inference` scope |

### 5.3 令牌交换

```
POST TOKEN_URL
  grant_type=authorization_code
  code=<授权码>
  code_verifier=<PKCE验证器>
  state=<状态令牌>
  [expires_in=<可选有效期>]
```

### 5.4 令牌刷新

```
POST TOKEN_URL
  grant_type=refresh_token
  refresh_token=<刷新令牌>
  [scope=<请求的scope>]
```

**智能优化**：刷新时智能获取 profile 信息（避免重复查询，每天减少约 700 万次请求）

## 六、Profile 管理

### 6.1 Profile 获取 (`getOauthProfile.ts`)

两种认证方式：

| 方式 | 说明 |
|------|------|
| **API Key** | `getOauthProfileFromApiKey()`，需 `account_uuid` |
| **OAuth Token** | `getOauthProfileFromOauthToken()`，Bearer 认证 |

**端点**：`/api/oauth/profile`
**Header**：`OAUTH_BETA_HEADER = 'oauth-2025-04-20'`

### 6.2 Profile 信息

从 OAuth token 获取的信息：

| 字段 | 说明 |
|------|------|
| `subscriptionType` | 订阅类型（max/pro/enterprise/team） |
| `rateLimitTier` | 速率限制层级 |
| `billingType` | 计费类型 |
| `organization_type` | 组织类型 |
| `organization_role` | 组织角色 |
| `workspace_role` | 工作区角色 |
| `organization_name` | 组织名称 |

## 七、XAA IdP 登录

### 7.1 概述 (`xaaIdpLogin.ts`)

企业 IdP 单点登录 → 多 MCP 服务器静默认证

### 7.2 核心机制

```
OIDC Discovery → PKCE + authorization_code → id_token 缓存
```

| 步骤 | 说明 |
|------|------|
| **Discovery** | 从 `{issuer}/.well-known/openid-configuration` 获取元数据 |
| **PKCE** | 与 Claude OAuth 相同的密码学机制 |
| **id_token 缓存** | `secureStorage` 存储，键为 `mcpXaaIdp[issuerKey]` |
| **令牌有效期** | JWT `exp` claim → `expires_in` → 默认 1 小时 |

### 7.3 关键函数

| 函数 | 说明 |
|------|------|
| `discoverOidc(issuer)` | 获取 OIDC 发现文档 |
| `acquireIdpIdToken(opts)` | 获取 id_token（先查缓存） |
| `getCachedIdpIdToken(issuer)` | 读取缓存，过期前 60 秒视为无效 |
| `saveIdpIdToken`/`clearIdpIdToken` | 缓存管理 |
| `saveIdpClientSecret` | 客户端密钥管理 |

### 7.4 安全设计

- **issuerKey()**：规范化 issuer URL（去除尾部斜杠、小写 host）
- **waitForCallback()**：本地 HTTP 服务器，支持 abortSignal、超时（5 分钟）
- **jwtExp()**：不验证签名直接解析 exp（RFC 8693 subject_token）

## 八、OAuth 端口管理

### 8.1 端口策略 (`oauthPort.ts`)

| 平台 | 端口范围 |
|------|----------|
| **Windows** | `39152-49151`（避开系统保留的 49152-65535） |
| **其他平台** | `49152-65535`（IANA 动态端口） |
| **回退端口** | `3118` |

**环境变量**：`MCP_OAUTH_CALLBACK_PORT` 可固定端口

## 九、令牌生命周期

### 9.1 全生命周期

```
登录 → 获取 access_token + refresh_token + expires_in
      ↓
缓存 → accessToken (内存/memoize)
       refreshToken (secureStorage/globalConfig)
       expiresAt = Date.now() + expires_in * 1000
      ↓
使用 → API 请求携带 Bearer {accessToken}
      ↓
检查 → isOAuthTokenExpired(expiresAt): 提前 5 分钟判定过期
      ↓
刷新 → refreshOAuthToken(refreshToken)
       - 获取新 access_token + refresh_token
       - 智能跳过 profile 查询
       - 更新 globalConfig.oauthAccount
      ↓
清理 → handleOAuth401Error(): 强制刷新、清除 memoize 缓存
```

### 9.2 过期检查

```typescript
function isOAuthTokenExpired(expiresAt: number): boolean {
  const BUFFER = 5 * 60 * 1000; // 5 分钟缓冲
  return Date.now() + BUFFER >= expiresAt;
}
```

## 十、与其他服务的集成

| 服务 | 集成方式 |
|------|----------|
| **MCP 客户端** | `withOAuth401Retry` 包装请求，401 时自动刷新 |
| **Bootstrap** | 优先使用 OAuth（需 `user:profile` scope） |
| **Voice 服务** | Voice 模式必须 OAuth 认证 |
| **Session Ingress** | 通过 OAuth 获取会话日志 |
| **Bridge 远程模式** | 需要 OAuth 令牌才能启用 |

## 十一、OAuth Scopes

| Scope | 说明 |
|-------|------|
| `user:profile` | 用户基本信息 |
| `user:inference` | 推理权限（持久令牌） |
| `user:sessions:claude_code` | Claude Code 会话 |
| `user:mcp_servers` | MCP 服务器访问 |
| `user:file_upload` | 文件上传 |
| `org:create_api_key` | 创建 API Key（Console） |

## 十二、关键文件索引

| 文件 | 路径 |
|------|------|
| OAuth 配置 | `src/constants/oauth.ts` |
| PKCE 实现 | `src/services/oauth/crypto.ts` |
| 回调监听 | `src/services/oauth/auth-code-listener.ts` |
| 客户端核心 | `src/services/oauth/client.ts` |
| Profile 获取 | `src/services/oauth/getOauthProfile.ts` |
| 流程编排 | `src/services/oauth/index.ts` |
| MCP OAuth 端口 | `src/services/mcp/oauthPort.ts` |
| 企业 IdP 登录 | `src/services/mcp/xaaIdpLogin.ts` |
