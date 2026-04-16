# API Client 架构分析

## 一、概述

API 客户端层位于 `src/services/api/` 目录（20 个文件），实现了与 Anthropic 兼容 API 的完整通信层。

### 1.1 核心文件

| 文件 | 职责 |
|------|------|
| `client.ts` | SDK 工厂，创建 Anthropic 客户端实例 |
| `claude.ts` | 核心查询引擎，流式/非流式请求 |
| `withRetry.ts` | 重试包装器，指数退避 |
| `errors.ts` | 错误分类与用户友好消息 |
| `errorUtils.ts` | 连接错误详情提取 |
| `bootstrap.ts` | 启动时获取配置 |
| `usage.ts` | 速率限制使用量查询 |
| `sessionIngress.ts` | 会话日志持久化 |

## 二、Provider 支持

### 2.1 支持的后端

| Provider | 环境变量 | SDK |
|----------|----------|-----|
| **firstParty** (默认) | 无 (或 `ANTHROPIC_BASE_URL`) | `@anthropic-ai/sdk` |
| **Bedrock** | `CLAUDE_CODE_USE_BEDROCK=1` | `@anthropic-ai/bedrock-sdk` |
| **Vertex AI** | `CLAUDE_CODE_USE_VERTEX=1` | `@anthropic-ai/vertex-sdk` |
| **Foundry (Azure)** | `CLAUDE_CODE_USE_FOUNDRY=1` | `@anthropic-ai/foundry-sdk` |

### 2.2 客户端工厂 (`client.ts`)

`getAnthropicClient()` 动态导入适当的 SDK：

| Provider | 认证方式 |
|----------|----------|
| firstParty | OAuth access token 或 API key |
| Bedrock | AWS credentials 或 `AWS_BEARER_TOKEN_BEDROCK` |
| Vertex | `GoogleAuth` 自动 GCP 凭证刷新 |
| Foundry | `ANTHROPIC_FOUNDRY_API_KEY` 或 Azure AD |

### 2.3 自定义端点验证

`isFirstPartyAnthropicBaseUrl()` 验证：

- 仅允许 `api.anthropic.com`
- 内部用户允许 `api-staging.anthropic.com`

## 三、流式实现

### 3.1 核心流式 (`claude.ts`)

`queryModel()` 异步生成器 (~1000-2100+ 行)：

```
1. 原始流: anthropic.beta.messages.create({ stream: true }).withResponse()
2. 块处理: for await 循环处理 BetaRawMessageStreamEvent
3. 状态机: 处理不同事件类型
4. 资源清理: releaseStreamResources() 取消 Response body
```

### 3.2 状态机处理

| 事件 | 处理 |
|------|------|
| `message_start` | 捕获 `partialMessage`, TTFT, usage |
| `content_block_start` | 初始化 text/tool_use/thinking 块 |
| `content_block_delta` | 累积 text/input_json/thinking deltas |
| `content_block_stop` | 完成块 |
| `message_delta` | 更新 usage, 检测 stop_reason |

### 3.3 流式停滞检测

| 配置 | 默认值 |
|------|--------|
| 警告阈值 | 30 秒无块 |
| 空闲看门狗 | `CLAUDE_STREAM_IDLE_TIMEOUT_MS` (默认 90s) |
| 环境变量 | `CLAUDE_ENABLE_STREAM_WATCHDOG` |

### 3.4 非流式回退

如果流式失败且可重试：

- `executeNonStreamingRequest()` 重试 `stream: false`
- 独立超时 (默认 300s，远程会话 120s)

## 四、重试逻辑

### 4.1 核心重试 (`withRetry.ts`)

`withRetry<T>()` 异步生成器是中心重试机制：

| 参数 | 默认值 |
|------|--------|
| `DEFAULT_MAX_RETRIES` | 10 |
| `baseDelay` | 500ms |
| `maxDelay` | 32s (普通) / 5min (持久模式) |
| 抖动 | 25% |

### 4.2 重试策略

| 策略 | 说明 |
|------|------|
| **指数退避** | `baseDelay * 2^attempt + jitter` |
| **Retry-After 头** | 优先使用 API 返回的延迟 |
| **529 超载** | 最多 3 次连续重试，然后回退到 `fallbackModel` |
| **前台 vs 后台** | 仅前台查询源重试 529 |
| **持久重试** | `CLAUDE_CODE_UNATTENDED_RETRY` 启用无限重试 |

### 4.3 特殊处理

| 场景 | 处理 |
|------|------|
| **401 认证错误** | `handleOAuth401Error()` 强制刷新令牌 |
| **403 令牌吊销** | 清除并重试 |
| **ECONNRESET/EPIPE** | 禁用 keep-alive 池 |
| **Fast Mode 冷却** | 429/529 时进入冷却 |
| **Max Tokens 溢出** | 动态调整 `max_tokens` |
| **云 Provider 认证** | AWS/GCP 凭证刷新 |

### 4.4 529 处理

```
最多 3 次连续 529 重试
    → 如果配置了 fallbackModel
        → Opus → Sonnet 回退（仅订阅者）
    → 前台查询源才重试
    → 后台查询立即退出
```

### 4.5 持久重试模式

```
CLAUDE_CODE_UNATTENDED_RETRY=1
    → 无限重试
    → 30s 心跳间隔（防止宿主标记空闲）
    → 最大延迟 5 分钟
```

## 五、错误处理

### 5.1 错误分类 (`errors.ts`)

`getAssistantMessageFromError()` (~400-688 行) 映射错误：

| 错误类型 | 用户消息 |
|----------|----------|
| `APIConnectionTimeoutError` | "Request timed out" |
| Prompt too long | 解析 token gap 用于反应式压缩 |
| Rate limits (429) | 处理统一速率限制头 |
| Media errors | 图片大小、PDF 页数限制 |
| Auth errors | API key 无效、令牌吊销 |

### 5.2 媒体错误

| 检测函数 | 说明 |
|----------|------|
| `isMediaSizeError()` | 检测媒体大小错误用于反应式压缩 |
| 图片大小 | 检测并提示 |
| PDF 页数 | 超限处理 |
| 密码保护 PDF | 提示输入密码 |

### 5.3 连接错误详情 (`errorUtils.ts`)

| 函数 | 说明 |
|------|------|
| 提取 SSL/超时详情 | 从连接错误提取 |
| HTML 清理 | 清理错误页面 HTML |
| 嵌套错误消息 | 从反序列化 JSONL 错误提取 |

## 六、速率限制

### 6.1 客户端处理

| 机制 | 说明 |
|------|------|
| `withRetry.ts` | 遵守 `Retry-After` 头 |
| **指数退避** | 自动延迟重试 |

### 6.2 配额追踪 (`usage.ts`)

从 `/api/oauth/usage` 获取使用量：

| 配额类型 | 说明 |
|----------|------|
| 5 小时配额 | 短期使用量 |
| 7 天配额 | 长期使用量 |
| Opus/Sonnet 配额 | 模型特定配额 |
| 额外使用量 | overage credits |

### 6.3 Claude AI 限制

`claudeAiLimits.ts`：

- 从错误头提取配额状态
- 追踪 overage 状态

### 6.4 模拟速率限制

`/mock-limits` 命令用于内部测试。

### 6.5 Fast Mode 超额

`anthropic-ratelimit-unified-overage-disabled-reason` 头存在时，fast mode 永久禁用。

## 七、使用量追踪

### 7.1 Token 计数

| 函数 | 说明 |
|------|------|
| `tokenCountFromLastAPIResponse()` | 从最后响应提取 usage |
| `getTokenUsage()` | 提取 input/output/cache tokens |

### 7.2 成本追踪

`addToTotalSessionCost()` 在 `claude.ts` 中计算 USD 成本。

### 7.3 分析事件

| 事件 | 说明 |
|------|------|
| `tengu_api_retry` | API 重试 |
| `tengu_api_opus_fallback_triggered` | Opus 回退 |
| `tengu_compact` | 压缩 |
| `tengu_streaming_stall` | 流式停滞 |

### 7.4 提示缓存指标

- 缓存创建/读取 tokens 单独追踪
- `promptCacheBreakDetection.ts` 检测缓存中断

## 八、Bootstrap 启动

### 8.1 启动流程 (`bootstrap.ts`)

```
启动时
    → 调用 /api/claude_cli/bootstrap
    → 获取客户端数据
    → 获取额外模型选项
    → 优先使用 OAuth（需 user:profile scope）
```

### 8.2 Beta 头

`OAUTH_BETA_HEADER = 'oauth-2025-04-20'`

## 九、会话日志

### 9.1 Session Ingress (`sessionIngress.ts`)

| 操作 | 说明 |
|------|------|
| PUT append | 乐观并发（`Last-Uuid` 头） |
| GET | 会话 hydration |
| CCR v2 | teleport-events 分页 API |

## 十、关键设计模式

### 10.1 异步生成器模式

`withRetry()` 和 `queryModel()` 都是异步生成器：

```
yield SystemAPIErrorMessage (进度更新)
    → 最终 return 结果
```

### 10.2 Beta 头锁存

Beta 头在首次发送后**会话级锁存**：

- fast mode
- AFK mode
- 缓存编辑
- 上下文管理

防止中途切换导致提示缓存中断。

### 10.3 提示缓存

| 特性 | 说明 |
|------|------|
| 临时缓存控制标记 | 系统提示和消息上 |
| 1 小时 TTL | 合格用户通过 GrowthBook 功能标志 |

### 10.4 VCR 支持

| 函数 | 说明 |
|------|------|
| `withStreamingVCR()` | 流式请求/响应录制和回放 |
| `withVCR()` | 非流式录制和回放 |

### 10.5 额外 Body 参数

`CLAUDE_CODE_EXTRA_BODY` 环境变量允许任意 JSON 注入到 API 请求 body。

## 十一、文件索引

| 文件 | 路径 |
|------|------|
| 客户端工厂 | `src/services/api/client.ts` |
| 查询引擎 | `src/services/api/claude.ts` |
| 重试逻辑 | `src/services/api/withRetry.ts` |
| 错误处理 | `src/services/api/errors.ts` |
| 错误工具 | `src/services/api/errorUtils.ts` |
| Bootstrap | `src/services/api/bootstrap.ts` |
| 使用量 | `src/services/api/usage.ts` |
| 会话日志 | `src/services/api/sessionIngress.ts` |
| 提示缓存检测 | `src/services/api/promptCacheBreakDetection.ts` |
| 首次 Token 时间 | `src/services/api/firstTokenDate.ts` |
| 日志 | `src/services/api/logging.ts` |
| 指标退出 | `src/services/api/metricsOptOut.ts` |
| Overage 信用 | `src/services/api/overageCreditGrant.ts` |
| 推荐 | `src/services/api/referral.ts` |
| Ultra Review 配额 | `src/services/api/ultrareviewQuota.ts` |
| 空 Usage | `src/services/api/emptyUsage.ts` |
| 文件 API | `src/services/api/filesApi.ts` |
| Grove | `src/services/api/grove.ts` |
| 管理请求 | `src/services/api/adminRequests.ts` |
| 转储提示 | `src/services/api/dumpPrompts.ts` |
