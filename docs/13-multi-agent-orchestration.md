# 多智能体编排架构分析

## 一、概述

Claude Code 实现了多智能体编排系统，主智能体可以协调多个子智能体完成复杂任务。

### 1.1 整体架构

```
Main Agent (主智能体 - Coordinator)
    ├── LocalAgent (本地代理)
    ├── RemoteAgent (远程代理)
    ├── Fork (子代理)
    ├── Teammate (队友)
    └── DreamTask (记忆整合)
```

每个智能体都有独立的 **Tool Pool (工具池)**，通过 **SendMessage** 进行通信。

### 1.2 工作树隔离

```
Worktree Isolation (工作树隔离)
    ├── Independent environments (独立环境)
    ├── Branch management (分支管理)
    └── Secure context (安全上下文)
```

## 二、主智能体 (Main Agent)

### 2.1 角色

主智能体是**协调器 (Coordinator)**，负责：

- 任务分解
- 子智能体分配
- 结果聚合
- 错误处理

### 2.2 功能

| 功能 | 说明 |
|------|------|
| **任务规划** | 将复杂任务分解为子任务 |
| **智能体选择** | 根据任务类型选择合适智能体 |
| **通信协调** | 管理智能体间的消息传递 |
| **状态追踪** | 追踪所有子智能体的状态 |

## 三、智能体类型

### 3.1 LocalAgent (本地代理)

在本地环境执行的代理：

| 工具池 | 说明 |
|--------|------|
| 文件操作 | Read/Edit/Write |
| 命令执行 | Bash |
| 搜索 | Glob/Grep |

**使用场景**：本地代码修改、文件操作

### 3.2 RemoteAgent (远程代理)

在远程环境执行的代理：

| 工具池 | 说明 |
|--------|------|
| 远程连接 | SSH |
| 远程命令 | Remote Bash |
| 远程文件 | Remote File Ops |

**使用场景**：远程服务器操作、部署

### 3.3 Fork (子代理)

主智能体的分叉副本：

| 特点 | 说明 |
|------|------|
| **共享上下文** | 与主智能体共享 prompt cache |
| **独立执行** | 可以独立执行任务 |
| **结果返回** | 执行结果返回主智能体 |

**使用场景**：并行任务执行、后台摘要

### 3.4 Teammate (队友)

协作者智能体：

| 工具池 | 说明 |
|--------|------|
| 语音 | Voice |
| 通话 | Call |
| 文件共享 | Shared Files |

**使用场景**：团队协作、语音交互

### 3.5 DreamTask (记忆整合)

记忆整合智能体：

| 工具池 | 说明 |
|--------|------|
| 数据库 | Database |
| 芯片 | Processing |
| 搜索 | Search |
| 时钟 | Time |

**使用场景**：记忆提取、知识整合

## 四、通信机制

### 4.1 SendMessage

智能体间通过 `SendMessage` 通信：

```
Main Agent ←SendMessage→ LocalAgent
Main Agent ←SendMessage→ RemoteAgent
Main Agent ←SendMessage→ Fork
Main Agent ←SendMessage→ Teammate
Main Agent ←SendMessage→ DreamTask
```

### 4.2 消息格式

```typescript
interface AgentMessage {
  from: string;
  to: string;
  type: 'request' | 'response' | 'notification';
  content: unknown;
  messageId: string;
  correlationId?: string;
}
```

### 4.3 通信模式

| 模式 | 说明 |
|------|------|
| **请求-响应** | 主智能体请求，子智能体响应 |
| **发布-订阅** | 主智能体广播，多个子智能体接收 |
| **点对点** | 子智能体间直接通信 |

## 五、工具池

### 5.1 工具池结构

每个智能体有独立的工具池：

```
Tool Pool
    ├── 文件工具 (File Tools)
    ├── 命令工具 (Command Tools)
    ├── 搜索工具 (Search Tools)
    ├── 网络工具 (Network Tools)
    └── 专用工具 (Specialized Tools)
```

### 5.2 工具池隔离

不同智能体的工具池相互隔离：

- 避免工具冲突
- 独立权限
- 独立状态

## 六、工作树隔离

### 6.1 独立环境

每个智能体在独立环境中运行：

| 隔离项 | 说明 |
|--------|------|
| 文件系统 | 独立工作目录 |
| 环境变量 | 独立环境变量 |
| 进程空间 | 独立进程 |

### 6.2 分支管理

支持 Git 分支隔离：

- 每个智能体可以操作独立分支
- 自动分支创建与合并
- 冲突检测与解决

### 6.3 安全上下文

安全隔离：

- 独立权限上下文
- 资源限制
- 网络隔离

## 七、任务编排

### 7.1 任务分解

主智能体将复杂任务分解：

```
复杂任务
    ├── 子任务 1 → LocalAgent
    ├── 子任务 2 → RemoteAgent
    ├── 子任务 3 → Fork
    └── 子任务 4 → Teammate
```

### 7.2 执行流程

```
1. 主智能体接收任务
2. 分析任务需求
3. 选择合适的子智能体
4. 发送任务消息
5. 等待子智能体完成
6. 聚合结果
7. 返回最终结果
```

### 7.3 错误处理

| 错误类型 | 处理策略 |
|----------|----------|
| 子智能体失败 | 重试或切换智能体 |
| 超时 | 终止并报告 |
| 资源不足 | 排队等待 |

## 八、协调器模式

### 8.1 设计模式

使用**协调器模式 (Coordinator Pattern)**：

```
Coordinator (主智能体)
    ├── 注册子智能体
    ├── 分配任务
    ├── 监控状态
    └── 聚合结果
```

### 8.2 状态管理

```typescript
interface AgentState {
  id: string;
  type: AgentType;
  status: 'idle' | 'running' | 'completed' | 'failed';
  currentTask?: Task;
  result?: unknown;
}
```

## 九、文件索引

| 目录/文件 | 路径 |
|-----------|------|
| 协调器 | `src/coordinator/` |
| 代理工具 | `src/tools/AgentTool/` |
| 任务系统 | `src/tasks/`, `src/Task.ts` |
| 远程代理 | `src/remote/` |
| 桥接 | `src/bridge/` |
| 分叉代理 | `src/utils/forkedAgent/` |
| 消息系统 | `src/utils/messages/` |
