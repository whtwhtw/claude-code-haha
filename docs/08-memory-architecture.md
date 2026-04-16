# Memory (记忆) 系统架构分析

## 一、概述

Claude Code 实现了双轨记忆系统：

1. **Session Memory（会话内存）**：单会话内的持久化摘要
2. **Extract Memories（提取记忆）**：跨会话持久化的记忆

### 1.1 核心目录

| 目录 | 职责 |
|------|------|
| `src/services/SessionMemory/` | 会话内存（3 个文件） |
| `src/services/extractMemories/` | 记忆提取（2 个文件） |
| `src/memdir/` | 记忆目录管理 |
| `src/services/teamMemorySync/` | 团队记忆同步 |

### 1.2 核心文件

| 文件 | 职责 |
|------|------|
| `sessionMemory.ts` | 会话内存提取逻辑 |
| `sessionMemoryUtils.ts` | 会话内存工具函数 |
| `extractMemories.ts` | 记忆提取执行 |
| `prompts.ts` (SessionMemory) | 会话内存提示词 |
| `prompts.ts` (extractMemories) | 记忆提取提示词 |

## 二、Session Memory (会话内存)

### 2.1 概述

Session Memory 是一个**后台运行的摘要机制**，在对话期间周期性地提取关键信息并维护一个 markdown 文件。

### 2.2 提取触发条件

| 条件 | 默认值 | 说明 |
|------|--------|------|
| `minimumMessageTokensToInit` | 10000 | 初始化阈值 |
| `minimumTokensBetweenUpdate` | 5000 | Token 增长阈值 |
| `toolCallsBetweenUpdates` | 3 | Tool Call 阈值 |

**触发逻辑**：
```
(token 阈值 && tool_call 阈值) || (token 阈值 && 最后无 tool_calls)
```

**安全条件**：最后一条 assistant 消息不能有 tool calls（避免提取时模型仍在工作中）

### 2.3 提取执行流程

```
1. 注册为 PostSamplingHook（在 initSessionMemory 中）
2. 仅在 REPL 主线程运行（querySource === 'repl_main_thread'）
3. 通过 runForkedAgent 创建分叉子代理执行提取
4. 与主对话共享 prompt cache
5. 使用 createMemoryFileCanUseTool 限制子代理只能对 memory 文件执行 Edit 操作
6. 提取完成后更新 lastSummarizedMessageId 和记录 token 使用量
```

### 2.4 手动提取

`manuallyExtractSessionMemory()`：

- 绕过阈值检查
- 供 `/summary` 命令使用

## 三、Session Memory 持久化

### 3.1 存储路径

```typescript
function getSessionMemoryPath(): string
```

路径通过 `findCanonicalGitRoot` 获取，确保同一 repo 的不同 worktree 共享记忆。

### 3.2 文件格式

Markdown 格式，包含预定义模板（9 个章节）：

| 章节 | 说明 |
|------|------|
| `# Session Title` | 5-10 字标题 |
| `# Current State` | 当前正在做什么 |
| `# Task specification` | 用户要求构建什么 |
| `# Files and Functions` | 重要文件 |
| `# Workflow` | 常用命令和工作流 |
| `# Errors & Corrections` | 遇到的错误和修正 |
| `# Codebase and System Documentation` | 系统组件 |
| `# Learnings` | 经验教训 |
| `# Key results` | 关键输出结果 |
| `# Worklog` | 逐步操作摘要 |

### 3.3 模板自定义

**默认模板位置**：`~/.claude/session-memory/config/template.md`

用户可自定义模板结构。

## 四、状态管理工具函数

### 4.1 提取状态追踪

| 函数 | 说明 |
|------|------|
| `markExtractionStarted()` | 标记提取开始（带时间戳） |
| `markExtractionCompleted()` | 标记提取完成 |
| `waitForSessionMemoryExtraction()` | 等待进行中的提取完成（15 秒超时） |
| `recordExtractionTokenCount()` | 记录提取时的上下文大小 |
| `hasMetUpdateThreshold()` | 测量自上次提取后的上下文增长 |

### 4.2 截断机制

`truncateSessionMemoryForCompact()`：

| 限制 | 值 |
|------|-----|
| 每章节 | ~2000 tokens |
| 总文件 | ~12000 tokens |

超限时生成 condense 提醒并自动截断。

## 五、Extract Memories (提取记忆)

### 5.1 概述

与 Session Memory **不同的系统**——它提取的是**跨会话持久化的记忆**，存储在 `~/.claude/projects/<git-root>/memory/` 目录。

### 5.2 关键架构特征

| 特征 | 说明 |
|------|------|
| **闭包状态管理** | 所有可变状态封闭在 `initExtractMemories()` 创建的闭包内 |
| **互斥机制** | 主代理和后台代理互斥执行 |
| **节流控制** | 通过 `tengu_bramble_lintel` 功能标志控制每 N 轮提取一次 |
| **Coalesced 调用** | 提取进行中时，新调用存入 pendingContext |
| **Drain 机制** | 响应刷新后等待所有进行中的提取完成 |

### 5.3 互斥机制

`hasMemoryWritesSince`：

- 如果主代理已经写了记忆文件，分叉提取跳过该轮
- 光标前进到最后一条消息，避免重复处理
- 主代理和后台代理**互斥执行**

### 5.4 工具权限

`createAutoMemCanUseTool()`：

| 允许 | 说明 |
|------|------|
| Read/Grep/Glob | 无限制 |
| Bash | 仅只读命令 |
| Edit/Write | 仅限 auto-memory 目录内 |
| REPL | 内部调用仍受上述规则约束 |
| **拒绝** | 其他所有工具 |

### 5.5 记忆写入流程

```
1. 预注入 memory directory manifest（避免浪费一轮做 ls）
2. 运行 forked agent（最多 5 轮）
3. 提取写入的文件路径
4. 更新光标
5. 创建 createMemorySavedMessage 系统消息通知主代理
```

## 六、记忆提示词

### 6.1 提取记忆提示词

**两种变体**：

| 变体 | 说明 |
|------|------|
| `buildExtractAutoOnlyPrompt` | 仅私有记忆 |
| `buildExtractCombinedPrompt` | 私有 + 团队记忆 |

**提示词结构**：

1. **Opener**：说明是 memory extraction subagent
2. **Types Section**：四种记忆类型
3. **What NOT to save**：排除规则
4. **How to save**：两步保存流程

### 6.2 记忆类型

| 类型 | 描述 | Scope |
|------|------|-------|
| **user** | 用户角色、目标、偏好 | 始终私有 |
| **feedback** | 用户给出的指导 | 默认私有，项目级约定才团队 |
| **project** | 工作、目标、决策 | 强烈偏向团队 |
| **reference** | 外部系统指针 | 通常团队 |

### 6.3 排除规则

**不保存的内容**：

- 代码模式、架构、git 历史（可从项目状态推导）
- 调试方案（修复在代码里）
- CLAUDE.md 已有内容
- 当前会话的临时状态
- PR 列表/活动摘要

### 6.4 两步保存流程

```
Step 1: 写记忆文件（带 frontmatter）
Step 2: 更新 MEMORY.md 索引
```

**MEMORY.md**：

- 是索引不是记忆本身
- 每行 <150 字符
- 超过 200 行会被截断

**Frontmatter 格式**：`name`, `description`, `type`

## 七、回忆信任机制

`TRUSTING_RECALL_SECTION`：

- 记忆声称的文件/函数存在性必须在推荐前验证
- 过时记忆应与当前代码状态比对
- 记忆是时间点快照，不是实时状态

## 八、记忆目录结构

### 8.1 目录路径

```
{memoryBase}/projects/{sanitized-git-root}/memory/
```

| 变量 | 说明 |
|------|------|
| `memoryBase` | `CLAUDE_CODE_REMOTE_MEMORY_DIR` 或 `~/.claude` |
| `git-root` | 通过 `findCanonicalGitRoot` 获取 |

### 8.2 路径解析优先级

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` (Cowork/SDK)
2. `settings.json` 的 `autoMemoryDirectory` (仅受信任源)
3. 默认计算路径

### 8.3 安全验证

`validateMemoryPath()` 拒绝：

- 相对路径
- 根路径
- Windows 盘符
- UNC 路径
- null 字节

### 8.4 启用控制

`isAutoMemoryEnabled` 检查：

| 条件 | 说明 |
|------|------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 环境变量 |
| `CLAUDE_CODE_SIMPLE` | --bare 模式 |
| CCR 无持久存储 | 远程环境 |
| `settings.json` 的 `autoMemoryEnabled` | 用户设置 |

### 8.5 KAIROS 每日日志模式

当 `feature('KAIROS')` 启用时，长会话使用 append-only 日志：

```
memory/logs/YYYY/MM/YYYY-MM-DD.md
```

夜间 `/dream` skill 将日志蒸馏为 topic 文件 + MEMORY.md。

### 8.6 目录大小控制

| 常量 | 值 |
|------|-----|
| `ENTRYPOINT_NAME` | `MEMORY.md` |
| `MAX_ENTRYPOINT_LINES` | 200 |
| `MAX_ENTRYPOINT_BYTES` | 25,000 |
| `MAX_MEMORY_FILES` | 200（扫描上限） |
| `FRONTMATTER_MAX_LINES` | 30 |

## 九、记忆年龄感知

`memoryAge.ts` 提供：

| 函数 | 说明 |
|------|------|
| `memoryAgeDays()` | 天数计算 |
| `memoryFreshnessText()` | 超过 1 天添加"可能过时"警告 |
| `memoryFreshnessNote()` | 包装为 `<system-reminder>` 标签 |

## 十、记忆在 Context Assembly 中的使用

### 10.1 System Prompt 集成

`loadMemoryPrompt()` 通过 `systemPromptSection('memory', ...)` 注入系统提示。

### 10.2 查询时记忆召回

`findRelevantMemories()`：

```
1. 扫描 memory 目录获取 frontmatter
2. 调用 Sonnet 模型（sideQuery）选择最多 5 个相关记忆
3. 过滤已展示过的路径，避免重复选择
4. 排除 MEMORY.md（已在系统提示中加载）
```

### 10.3 Session Memory 在 Compact 中的使用

`getSessionMemoryContent()` 被：

| 模块 | 用途 |
|------|------|
| `awaySummary.ts` | 用户离开后摘要 |
| `sessionMemoryCompact.ts` | 上下文压缩时注入 |
| `skillify.ts` | skillify skill 中获取会话上下文 |

## 十一、Team Memory Sync (团队记忆同步)

### 11.1 概述

团队记忆是 auto-memory 的 `team/` 子目录，跨 repo 协作者共享。

### 11.2 API 契约

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/api/claude_code/team_memory?repo={owner/repo}` | 完整数据 |
| GET | `/api/claude_code/team_memory?repo={owner/repo}&view=hashes` | 仅 checksums |
| PUT | `/api/claude_code/team_memory?repo={owner/repo}` | 上传 (upsert 语义) |

### 11.3 同步语义

| 操作 | 语义 |
|------|------|
| **Pull** | 服务器获胜（按 key 覆盖本地） |
| **Push** | 仅上传 content hash 不同的 key（增量上传） |
| **删除** | 不传播（本地删除不影响服务器） |

### 11.4 SyncState 状态管理

```typescript
interface SyncState {
  lastKnownChecksum: string | null;
  serverChecksums: Map<string, string>;
  serverMaxEntries: number | null;
}
```

### 11.5 文件 Watcher 机制

| 机制 | 说明 |
|------|------|
| 监听 | `fs.watch({recursive: true})` |
| macOS | FSEvents（O(1) fd） |
| Linux | inotify（O(目录数) fd） |
| Debounce | 2 秒等待写入稳定后推送 |

**Push 抑制**：永久失败后抑制重试，直到文件删除或会话重启。

### 11.6 安全机制

#### Secret 扫描

上传前扫描，使用 gitleaks 规则子集：

- 覆盖 30+ 规则
- AWS/GCP/Azure、GitHub/GitLab、Slack、OpenAI/Anthropic、Stripe 等
- 发现秘密的文件**完全跳过**上传

#### 路径遍历防护

双重验证：

1. `path.resolve()` 快速拒绝
2. `realpath()` 解析 symlink

拒绝：null 字节、URL 编码遍历、Unicode 规范化攻击、dangling symlink、symlink loop。

#### 写入时拦截

FileWriteTool/FileEditTool 在写入 team memory 前调用 `checkTeamMemSecrets`：

- 发现秘密直接拒绝写入

## 十二、文件索引

| 文件 | 路径 |
|------|------|
| 会话内存 | `src/services/SessionMemory/sessionMemory.ts` |
| 会话内存工具 | `src/services/SessionMemory/sessionMemoryUtils.ts` |
| 会话内存提示词 | `src/services/SessionMemory/prompts.ts` |
| 记忆提取 | `src/services/extractMemories/extractMemories.ts` |
| 记忆提取提示词 | `src/services/extractMemories/prompts.ts` |
| 记忆目录 | `src/memdir/` |
| 团队记忆同步 | `src/services/teamMemorySync/` |
