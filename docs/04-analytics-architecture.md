# Analytics (分析与遥测) 架构分析

## 一、概述

Claude Code 实现了**双轨遥测架构**：

1. **内部分析 (1P)**：第一方事件日志 + Datadog
2. **客户遥测 (3P)**：OTLP (OpenTelemetry Protocol)

Analytics 服务位于 `src/services/analytics/` 目录（9 个文件），遥测工具位于 `src/utils/telemetry/` 目录。

### 1.1 核心文件

| 文件 | 职责 |
|------|------|
| `index.ts` | 公共 API 入口、事件队列 |
| `config.ts` | 共享配置、禁用检查 |
| `datadog.ts` | Datadog 集成 |
| `firstPartyEventLogger.ts` | 1P 事件日志 |
| `firstPartyEventLoggingExporter.ts` | 1P 导出器 |
| `growthbook.ts` | GrowthBook 功能标志 |
| `metadata.ts` | 元数据收集与脱敏 |
| `sink.ts` | 分析路由 |
| `sinkKillswitch.ts` | Kill Switch |

### 1.2 遥测工具

| 文件 | 职责 |
|------|------|
| `instrumentation.ts` | OTLP 遥测初始化 |
| `sessionTracing.ts` | 会话追踪 |
| `betaSessionTracing.ts` | Beta 详细追踪 |
| `perfettoTracing.ts` | Perfetto 追踪 |
| `events.ts` | OTel 事件记录 |
| `pluginTelemetry.ts` | 插件遥测 |
| `skillLoadedEvent.ts` | 技能加载事件 |

## 二、整体架构

### 2.1 双轨架构

```
用户操作
    ├── logEvent() / logEventAsync()  ← index.ts (公共 API)
    │       └── 事件队列 (sink 未挂载时)
    │
    ├── logOTelEvent()  ← events.ts (客户遥测)
    │       └── OTLP 端点
    │
    └── BigQueryMetricsExporter  ← 定时 5min
            └── /api/claude_code/metrics
```

### 2.2 核心设计原则

| 原则 | 实现 |
|------|------|
| **零依赖队列** | sink 挂载前事件入队，避免导入循环 |
| **隐私分级** | `_PROTO_*` 前缀标记 PII 字段 |
| **Kill Switch** | 按 sink 粒度远程关闭 |
| **采样控制** | GrowthBook 动态配置采样率 |

## 三、事件追踪

### 3.1 公共 API (`index.ts`)

```typescript
// 事件日志主入口
function logEvent(eventName: string, metadata: AnalyticsMetadata): void
function logEventAsync(eventName: string, metadata: AnalyticsMetadata): Promise<void>

// 挂载后端
function attachAnalyticsSink(sink: AnalyticsSink): void

// 脱敏
function stripProtoFields(metadata: Record<string, unknown>): Record<string, unknown>
```

### 3.2 元数据限制

| 类型 | 说明 |
|------|------|
| **允许** | `boolean`, `number` |
| **禁止** | 字符串（需显式类型标记 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`） |

### 3.3 队列机制

```
事件 → 检查 sink 是否挂载
    ├── 已挂载 → 直接发送到 sink
    └── 未挂载 → 入 eventQueue
        └── sink 挂载后 → queueMicrotask 异步排空
```

## 四、Datadog 集成

### 4.1 配置 (`datadog.ts`)

| 配置项 | 值 |
|--------|-----|
| **端点** | `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` |
| **批量间隔** | 15 秒 |
| **批量大小** | 最多 100 条 |
| **网络超时** | 5 秒 |

### 4.2 事件白名单

`DATADOG_ALLOWED_EVENTS` 包含约 50+ 个事件：

| 类别 | 示例 |
|------|------|
| Chrome Bridge | `chrome_bridge_*` |
| API 错误/成功 | `api_error`, `api_success` |
| OAuth | `oauth_*` |
| 工具使用 | `tool_*` |
| 异常 | `exception_*` |
| 团队同步 | `team_sync_*` |

### 4.3 基数控制 (Cardinality Reduction)

| 脱敏策略 | 说明 |
|----------|------|
| MCP 工具名 | 统一为 `"mcp"` |
| 模型名 | 非 ant 用户规范化 (`getCanonicalName`) |
| 版本号 | 开发版本截断（去除时间戳和 SHA） |

### 4.4 用户分桶

```typescript
function getUserBucket(userId: string): number {
  const hash = sha256(userId);
  return parseInt(hash.slice(0, 8), 16) % 30; // 30 个桶
}
```

### 4.5 标签字段

18 个高基数过滤字段：

- `arch`, `model`, `platform`, `provider`, `userType`
- `status` → `http_status`, `http_status_range`

## 五、第一方事件日志 (1P)

### 5.1 架构 (`firstPartyEventLogger.ts`)

使用 OpenTelemetry SDK Logs API：

```
1P 事件 → LoggerProvider → FirstPartyEventLoggingExporter → /api/event_logging/batch
```

### 5.2 关键函数

| 函数 | 说明 |
|------|------|
| `logEventTo1P()` | 发送事件到 API |
| `logGrowthBookExperimentTo1P()` | 记录 GB 实验分配 |
| `initialize1PEventLogging()` | 创建 LoggerProvider |
| `reinitialize1PEventLoggingIfConfigChanged()` | 配置变更时重建 |
| `shutdown1PEventLogging()` | 进程退出前 flush |

### 5.3 事件采样

通过 GrowthBook 动态配置 `tengu_event_sampling_config`：

```typescript
function shouldSampleEvent(eventName: string): boolean {
  const config = getFeatureValue('tengu_event_sampling_config');
  const rate = config?.[eventName] ?? 1.0; // 默认 100%
  return Math.random() < rate;
}
```

### 5.4 批次配置

动态配置键 `tengu_1p_event_batch_config`：

| 参数 | 默认值 |
|------|--------|
| `scheduledDelayMillis` | 5000 |
| `maxExportBatchSize` | 100 |
| `maxQueueSize` | 1000 |
| `skipAuth` | false |
| `path` | `/api/event_logging/batch` |

## 六、1P 导出器

### 6.1 弹性设计 (`firstPartyEventLoggingExporter.ts`)

| 机制 | 实现 |
|------|------|
| **失败持久化** | 写入 `~/.claude/telemetry/1p_failed_events.<sessionId>.<batchUuid>.json` |
| **二次退避重试** | `baseBackoffDelayMs * attempts²`，最大 30s，最多 8 次 |
| **大请求分片** | 按 `maxBatchSize` (默认 200) 切分 |
| **Auth 回退** | 401 时自动无认证重试 |
| **启动恢复** | 启动时自动重试上一进程的失败批次 |

### 6.2 重试流程

```
POST 失败
    → 写入磁盘
    → 计算退避延迟 (base * attempts²)
    → 等待延迟
    → 检查 Kill Switch
    → 重试 (最多 8 次)
        ├── 成功 → 删除磁盘文件
        └── 失败 → 继续退避
```

## 七、GrowthBook 功能标志

### 7.1 客户端配置 (`growthbook.ts`)

| 配置 | 值 |
|------|-----|
| `remoteEval` | true (服务端评估) |
| `cacheKeyAttributes` | `['id', 'organizationUUID']` |
| **ant 用户刷新** | 20 分钟 |
| **外部用户刷新** | 6 小时 |

### 7.2 磁盘持久化

```typescript
syncRemoteEvalToDisk(): void
// 将 remoteEvalFeatureValues 写入全局配置
// 跨进程存活
```

### 7.3 特征值读取 API

| 函数 | 行为 |
|------|------|
| `getFeatureValue_CACHED_MAY_BE_STALE()` | 立即读内存 > 磁盘，不阻塞 |
| `getFeatureValue_DEPRECATED()` | 阻塞等待初始化（已弃用） |
| `checkGate_CACHED_OR_BLOCKING()` | 磁盘为 true 立即返回 |
| `checkSecurityRestrictionGate()` | 安全关键门 |

### 7.4 实验曝光追踪

```typescript
function logExposureForFeature(feature: string): void {
  if (!loggedExposure.has(feature)) {
    loggedExposure.add(feature);
    logGrowthBookExperimentTo1P({ feature });
  }
}
```

### 7.5 覆盖机制（优先级）

1. `CLAUDE_INTERNAL_FC_OVERRIDES` 环境变量（仅 ant）
2. `growthBookOverrides` 全局配置
3. 远程评估结果
4. 磁盘缓存
5. 默认值

### 7.6 关键动态配置

| 配置键 | 用途 |
|--------|------|
| `tengu_event_sampling_config` | 事件采样率 |
| `tengu_1p_event_batch_config` | 1P 批次参数 |
| `tengu_frond_boric` | Sink Kill Switch |
| `tengu_log_datadog_events` | Datadog 功能门 |

## 八、元数据收集

### 8.1 EnvContext (`metadata.ts`)

| 类别 | 字段 |
|------|------|
| 平台信息 | `platform`, `arch`, `nodeVersion`, `terminal` |
| 运行时 | `packageManagers`, `runtimes` |
| 部署模式 | `isCi`, `isClaubbit`, `isClaudeCodeRemote` |
| GitHub | Actions 元数据 |
| Linux | WSL/发行版信息 |
| VCS | 版本控制检测 |

### 8.2 性能指标

| 指标 | 说明 |
|------|------|
| 内存 | `rss`, `heapTotal/Used`, `external`, `arrayBuffers` |
| CPU | 使用率（增量计算） |
| 受限内存 | cgroup 内存限制 |

### 8.3 工具名称脱敏

```typescript
// MCP 工具统一为 "mcp_tool"
function sanitizeToolNameForAnalytics(toolName: string): string

// 记录真实 MCP 服务器/工具名（满足条件时）
function mcpToolDetailsForAnalytics(toolName: string): { server?: string, tool?: string }
```

**允许记录真实名称的条件**：

- 内置服务器
- 官方注册表 URL
- Cowork 模式

### 8.4 其他脱敏

| 函数 | 脱敏规则 |
|------|----------|
| `getFileExtensionForAnalytics()` | 扩展名超 10 字符归为 `"other"` |
| `getFileExtensionsFromBashCommand()` | 从允许命令集合提取 |
| `extractToolInputForTelemetry()` | 字符串 512→128 字符，JSON 4KB，嵌套深度 2 |
| `extractSkillName()` | 从 Skill 工具输入提取 |

### 8.5 Proto 标记

| 标记 | 用途 |
|------|------|
| `_PROTO_skill_name` | 技能名称 |
| `_PROTO_plugin_name` | 插件名称 |
| `_PROTO_marketplace_name` | 市场名称 |

## 九、分析路由

### 9.1 Sink 逻辑 (`sink.ts`)

```
事件 → 采样检查
    ├── 丢弃
    └── 通过
        ├── Datadog? → stripProtoFields() → Datadog HTTP API
        └── 1P? → logEventTo1P() → OTel → /api/event_logging/batch
```

### 9.2 初始化

```typescript
function initializeAnalyticsSink(): void
function initializeAnalyticsGates(): void
```

## 十、Kill Switch

### 10.1 配置 (`sinkKillswitch.ts`)

| 配置键 | 值 |
|--------|-----|
| `tengu_frond_boric` | Kill Switch 开关（混淆名称） |

### 10.2 检查函数

```typescript
function isSinkKilled(sink: 'datadog' | 'firstParty'): boolean
```

### 10.3 检查点

| 位置 | 说明 |
|------|------|
| `sink.ts` | Datadog 分发前检查 |
| `firstPartyEventLogger.ts` | 1P 分发前检查 |
| `FirstPartyEventLoggingExporter` | 每次 POST 前检查 |

## 十一、客户遥测 (3P)

### 11.1 OTLP 初始化 (`instrumentation.ts`)

三层遥测：

| 层 | Provider | Exporter |
|-----|----------|----------|
| **Metrics** | `MeterProvider` | `PeriodicExportingMetricReader` |
| **Logs** | `LoggerProvider` | `BatchLogRecordProcessor` |
| **Traces** | `BasicTracerProvider` | `BatchSpanProcessor` |

**导出器支持**：console, OTLP (grpc/http-json/http-protobuf), Prometheus

### 11.2 BigQuery 指标导出

| 配置 | 值 |
|------|-----|
| **端点** | `https://api.anthropic.com/api/claude_code/metrics` |
| **刷新间隔** | 5 分钟 |
| **启用条件** | API 客户、C4E 企业用户、团队用户 |

### 11.3 Beta 追踪 (`betaSessionTracing.ts`)

**可见性矩阵**：

| 内容 | 外部用户 | Ant |
|------|---------|-----|
| 系统提示 | ✅ | ✅ |
| 模型输出 | ✅ | ✅ |
| 思考输出 | ❌ | ✅ |
| 工具 | ✅ | ✅ |

**Hash 去重**：系统提示和工具 schema 按 hash 去重，每会话仅记录一次。

### 11.4 Perfetto 追踪 (`perfettoTracing.ts`)

- **格式**：Chrome Trace Event
- **查看工具**：`ui.perfetto.dev`
- **内容**：Agent 层级、API 调用、工具执行、用户等待
- **上限**：100,000 事件（超出淘汰最旧的一半）

## 十二、会话追踪

### 12.1 Span 层级 (`sessionTracing.ts`)

```
Interaction Span（用户请求 → Claude 响应）
├── LLM Request Span（模型调用）
│   └── Tool Span（工具调用）
│       ├── Tool Blocked on User Span（等待用户授权）
│       └── Tool Execution Span（实际执行）
└── Hook Span
```

### 12.2 上下文管理

- **AsyncLocalStorage**：管理追踪上下文
- **WeakRef**：存储活跃 span（30 分钟 TTL）
- **清理间隔**：每分钟

## 十三、隐私控制

### 13.1 总结

| 机制 | 实现 |
|------|------|
| **字符串禁止** | metadata 仅接受 `boolean | number` |
| **PII 分级** | `_PROTO_*` 前缀标记仅限 1P 特权列 |
| **名称脱敏** | MCP 工具名默认 `"mcp_tool"` |
| **用户匿名化** | SHA-256 哈希分桶（30 桶） |
| **组织退出** | `checkMetricsEnabled()` |
| **全局退出** | `isTelemetryDisabled()` |
| **内容截断** | 工具输入 512→128 字符、JSON 4KB |
| **Kill Switch** | 按 sink 粒度远程关闭 |

### 13.2 禁用条件 (`config.ts`)

```typescript
function isAnalyticsDisabled(): boolean {
  return (
    NODE_ENV === 'test' ||
    isThirdPartyCloudProvider() ||
    isPrivacyLevelNoTelemetry() ||
    isPrivacyLevelEssentialTraffic()
  );
}
```

## 十四、文件索引

| 文件 | 路径 |
|------|------|
| 公共 API | `src/services/analytics/index.ts` |
| 配置 | `src/services/analytics/config.ts` |
| Datadog | `src/services/analytics/datadog.ts` |
| 1P 日志 | `src/services/analytics/firstPartyEventLogger.ts` |
| 1P 导出器 | `src/services/analytics/firstPartyEventLoggingExporter.ts` |
| GrowthBook | `src/services/analytics/growthbook.ts` |
| 元数据 | `src/services/analytics/metadata.ts` |
| Sink | `src/services/analytics/sink.ts` |
| Kill Switch | `src/services/analytics/sinkKillswitch.ts` |
| OTLP 初始化 | `src/utils/telemetry/instrumentation.ts` |
| 会话追踪 | `src/utils/telemetry/sessionTracing.ts` |
| Beta 追踪 | `src/utils/telemetry/betaSessionTracing.ts` |
| Perfetto | `src/utils/telemetry/perfettoTracing.ts` |
| 事件记录 | `src/utils/telemetry/events.ts` |
