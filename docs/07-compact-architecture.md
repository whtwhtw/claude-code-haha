# Compact (上下文压缩) 架构分析

## 一、概述

Claude Code 实现了多层级的上下文压缩系统，通过不同的压缩策略和触发条件来管理对话上下文的大小。

### 1.1 核心目录

| 目录 | 职责 |
|------|------|
| `src/services/compact/` | 压缩服务（11 个文件） |

### 1.2 核心文件

| 文件 | 职责 |
|------|------|
| `autoCompact.ts` | 自动压缩触发与执行 |
| `microCompact.ts` | 微压缩（工具结果级） |
| `apiMicrocompact.ts` | API 级微压缩 |
| `postCompactCleanup.ts` | 压缩后清理 |
| `grouping.ts` | 消息分组逻辑 |
| `prompt.ts` | 压缩提示词模板 |
| `timeBasedMCConfig.ts` | 基于时间的配置 |
| `compactWarningState.ts` | 警告状态管理 |
| `compactWarningHook.ts` | 警告钩子 |

## 二、压缩层级

### 2.1 三层压缩架构

```
Micro-Compact (微压缩)
    → 工具结果级压缩
    → 在 API 调用前执行
    → 减少单次调用的 token 使用

Auto-Compact (自动压缩)
    → 消息级压缩
    → 基于 token 阈值触发
    → 生成摘要替换历史消息

Session Memory Compact (会话内存压缩)
    → 注入 Session Memory 内容
    → 在压缩后提供上下文
```

### 2.2 压缩触发条件

| 类型 | 触发条件 | 压缩范围 |
|------|----------|----------|
| **Micro-Compact** | 每次 API 调用前 | 工具结果 |
| **Auto-Compact** | Token 阈值、时间、消息数 | 历史消息 |
| **Reactive Compact** | API 返回 prompt-too-long | 紧急压缩 |

## 三、Micro-Compact (微压缩)

### 3.1 概述 (`microCompact.ts`)

微压缩在每次 API 调用前执行，通过压缩工具结果来减少 token 使用。

### 3.2 执行流程

```
API 调用前
    → 检查是否需要微压缩
        → 分析工具结果大小
        → 对大型结果进行压缩
            → 保留关键信息
            → 移除冗余内容
    → 返回压缩后的消息
```

### 3.3 缓存微压缩 (`apiMicrocompact.ts`)

**缓存编辑模式**：

- 使用 `CACHED_MICROCOMPACT` 功能标志
- 边界消息延迟到 API 响应后
- 使用实际 `cache_deleted_input_tokens`

## 四、Auto-Compact (自动压缩)

### 4.1 概述 (`autoCompact.ts`)

自动压缩基于多个阈值条件触发，生成摘要替换历史消息。

### 4.2 触发条件

#### Token 阈值

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `minimumMessageTokensToInit` | 10000 | 初始化压缩的最小 token 数 |
| `minimumTokensBetweenUpdate` | 5000 | 两次压缩间的最小 token 增长 |

#### 时间阈值

通过 `timeBasedMCConfig.ts` 配置：

| 参数 | 说明 |
|------|------|
| 时间间隔 | 基于时间的压缩触发间隔 |
| 冷却期 | 两次压缩间的最小时间间隔 |

#### 消息计数阈值

| 参数 | 说明 |
|------|------|
| 消息数量 | 达到一定消息数量时触发 |

### 4.3 触发逻辑

```
(新 token >= 阈值)
    → 检查冷却期
        → 冷却期已过
            → 执行压缩
        → 冷却期中
            → 跳过本次压缩
```

### 4.4 压缩执行

```typescript
interface AutoCompactTrackingState {
  compacted: boolean;
  turnId: string;
  turnCounter: number;
  consecutiveFailures: number;
}
```

**追踪状态**：

- `compacted`：是否已压缩
- `turnId`：压缩轮次 ID
- `turnCounter`：压缩计数器
- `consecutiveFailures`：连续失败次数

### 4.5 压缩结果

```typescript
interface CompactionResult {
  preCompactTokenCount: number;
  postCompactTokenCount: number;
  truePostCompactTokenCount: number;
  compactionUsage: TokenUsage;
  summaryMessages: Message[];
  attachments: Attachment[];
  hookResults: HookResult[];
}
```

### 4.6 遥测事件

```
logEvent('tengu_auto_compact_succeeded', {
  originalMessageCount,
  compactedMessageCount,
  preCompactTokenCount,
  postCompactTokenCount,
  compactionInputTokens,
  compactionOutputTokens,
  ...
})
```

## 五、Session Memory Compact

### 5.1 概述

在压缩后注入 Session Memory 内容，为模型提供持久化的上下文信息。

### 5.2 注入时机

| 时机 | 说明 |
|------|------|
| 压缩后 | 在 `buildPostCompactMessages()` 后注入 |
| 上下文组装 | 在 `getContext()` 中加载 |

### 5.3 截断机制

`truncateSessionMemoryForCompact()`：

| 限制 | 值 |
|------|-----|
| 每章节 | ~2000 tokens |
| 总文件 | ~12000 tokens |

## 六、压缩后清理

### 6.1 概述 (`postCompactCleanup.ts`)

压缩完成后执行清理操作，释放资源。

### 6.2 清理内容

| 清理项 | 说明 |
|--------|------|
| 临时文件 | 删除压缩过程中创建的临时文件 |
| 缓存 | 清除压缩相关的缓存 |
| 状态重置 | 重置压缩追踪状态 |

## 七、消息分组

### 7.1 概述 (`grouping.ts`)

消息分组将相关的消息组合在一起，便于压缩处理。

### 7.2 分组策略

| 策略 | 说明 |
|------|------|
| 轮次分组 | 按对话轮次分组 |
| 工具调用分组 | 将工具调用和结果分组 |
| 主题分组 | 按主题相关性分组 |

## 八、压缩提示词

### 8.1 提示词模板 (`prompt.ts`)

压缩提示词指导模型如何生成摘要：

```
你是一个对话压缩助手。请将以下对话历史压缩为一个简洁的摘要。

要求：
1. 保留所有关键决策和结论
2. 保留所有重要的代码变更
3. 保留所有用户明确要求的信息
4. 移除冗余的中间步骤
5. 保持摘要的可读性

对话历史：
{history}

摘要：
```

## 九、警告系统

### 9.1 警告状态 (`compactWarningState.ts`)

管理压缩警告的显示状态。

### 9.2 警告钩子 (`compactWarningHook.ts`)

在适当时机显示压缩警告。

## 十、查询循环中的压缩

### 10.1 压缩在查询循环中的位置

```
查询循环
    → 消息过滤 (getMessagesAfterCompactBoundary)
    → Snip Compact (可选)
    → Micro-Compact
    → Context Collapse (可选)
    → Auto-Compact
    → API 调用
```

### 10.2 关键代码

```typescript
// src/query.ts
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  { systemPrompt, userContext, systemContext, ... },
  querySource,
  tracking,
  snipTokensFreed,
);
```

## 十一、文件索引

| 文件 | 路径 |
|------|------|
| 自动压缩 | `src/services/compact/autoCompact.ts` |
| 微压缩 | `src/services/compact/microCompact.ts` |
| API 微压缩 | `src/services/compact/apiMicrocompact.ts` |
| 压缩后清理 | `src/services/compact/postCompactCleanup.ts` |
| 消息分组 | `src/services/compact/grouping.ts` |
| 压缩提示词 | `src/services/compact/prompt.ts` |
| 时间配置 | `src/services/compact/timeBasedMCConfig.ts` |
| 警告状态 | `src/services/compact/compactWarningState.ts` |
| 警告钩子 | `src/services/compact/compactWarningHook.ts` |
