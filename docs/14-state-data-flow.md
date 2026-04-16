# 状态管理与数据流架构分析

## 一、概述

Claude Code 使用自定义 Store 结合 React Context Providers 进行状态管理。

### 1.1 整体架构

```
Custom Store (自定义 Store - 34行)
    ├── getState
    ├── setState
    └── subscribe

AppState (100+ fields)
    └── 全局应用状态

React Context Providers:
    ├── AppStateProvider
    ├── StatsProvider
    ├── FpsMetrics
    ├── Modal
    ├── Voice
    └── Notification
```

### 1.2 数据流

```
User Input → processInput → Query Engine → API Stream → State Update → React Render → Terminal
```

## 二、Custom Store

### 2.1 核心 API

```typescript
interface Store<T> {
  getState(): T;
  setState(updater: Partial<T> | ((state: T) => Partial<T>)): void;
  subscribe(listener: (state: T) => void): Unsubscribe;
}
```

### 2.2 实现特点

- **极简设计**：仅 34 行代码
- **发布-订阅**：状态变更通知订阅者
- **不可变更新**：支持函数式更新

### 2.3 使用模式

```typescript
// 获取状态
const state = store.getState();

// 更新状态
store.setState({ loading: true });

// 函数式更新
store.setState(state => ({ count: state.count + 1 }));

// 订阅变更
const unsubscribe = store.subscribe(state => {
  console.log('State changed:', state);
});
```

## 三、AppState

### 3.1 字段概览 (100+ fields)

AppState 包含以下主要类别：

| 类别 | 字段示例 |
|------|----------|
| **会话信息** | sessionId, userId, organizationId |
| **模型配置** | model, mainLoopModel, permissionMode |
| **消息历史** | messages, assistantMessages |
| **工具状态** | tools, toolPermissionContext |
| **插件状态** | plugins, enabledPlugins |
| **MCP 状态** | mcpServers, mcpConnections |
| **内存状态** | sessionMemory, memories |
| **UI 状态** | loading, error, modal |
| **统计信息** | tokenUsage, cost, turns |
| **设置** | settings, preferences |

### 3.2 关键字段

```typescript
interface AppState {
  // 会话
  sessionId: string;
  userId: string;
  
  // 模型
  model: string;
  mainLoopModel: string;
  permissionMode: 'ask' | 'auto' | 'bypass';
  
  // 消息
  messages: Message[];
  
  // 工具
  tools: Tool[];
  toolPermissionContext: PermissionContext;
  
  // 插件
  plugins: PluginState;
  
  // MCP
  mcpServers: MCPServer[];
  mcp: { pluginReconnectKey: number };
  
  // UI
  loading: boolean;
  error: Error | null;
  
  // 统计
  tokenUsage: TokenUsage;
  cost: number;
}
```

## 四、React Context Providers

### 4.1 AppStateProvider

提供全局应用状态：

```tsx
<AppStateProvider>
  <App />
</AppStateProvider>
```

### 4.2 StatsProvider

提供统计信息：

- Token 使用量
- 成本追踪
- 轮次计数

### 4.3 FpsMetrics

性能指标：

- FPS 监控
- 渲染时间
- 内存使用

### 4.4 Modal

模态框管理：

- 模态框栈
- 打开/关闭
- 焦点管理

### 4.5 Voice

语音功能：

- 语音状态
- 录音状态
- 转录结果

### 4.6 Notification

通知系统：

- 通知队列
- 显示/隐藏
- 自动消失

## 五、数据流详细分析

### 5.1 完整数据流

```
1. User Input (用户输入)
   ↓
2. processInput (处理输入)
   - 解析用户输入
   - 识别命令
   - 提取附件
   ↓
3. Query Engine (查询引擎)
   - 上下文组装
   - API 调用
   - 工具检测
   ↓
4. API Stream (API 流式响应)
   - 流式接收
   - 内容解析
   - 工具调用
   ↓
5. State Update (状态更新)
   - 消息追加
   - 工具结果
   - 错误处理
   ↓
6. React Render (React 渲染)
   - 虚拟 DOM 更新
   - 差异计算
   ↓
7. Terminal (终端输出)
   - Ink 渲染
   - 终端输出
```

### 5.2 状态更新模式

```typescript
// 直接更新
store.setState({ messages: [...messages, newMessage] });

// 函数式更新
store.setState(state => ({
  tokenUsage: {
    ...state.tokenUsage,
    input: state.tokenUsage.input + delta.input,
    output: state.tokenUsage.output + delta.output,
  }
}));

// 批量更新
store.setState({
  loading: false,
  messages: newMessages,
  tokenUsage: newUsage,
});
```

## 六、状态持久化

### 6.1 会话持久化

会话状态持久化到：

- `~/.claude/sessions/` - 会话数据
- `~/.claude/config.json` - 配置
- `~/.claude/settings.json` - 设置

### 6.2 恢复机制

```
启动时
    → 读取会话数据
    → 恢复 AppState
    → 继续对话
```

## 七、性能优化

### 7.1 选择订阅

组件只订阅需要的状态：

```typescript
// 只订阅 messages
const messages = useAppState(state => state.messages);

// 只订阅 loading
const loading = useAppState(state => state.loading);
```

### 7.2 批量更新

合并多个状态变更：

```typescript
store.setState({
  loading: false,
  messages: newMessages,
  error: null,
});
```

### 7.3 防抖与节流

- 输入防抖
- 渲染节流
- 批量渲染

## 八、调试工具

### 8.1 状态检查

```
/state - 查看当前状态
/stats - 查看统计信息
/cost - 查看成本
```

### 8.2 性能监控

- FpsMetrics 实时监控
- 内存使用追踪
- 渲染时间分析

## 九、状态目录

### 9.1 状态管理文件

| 文件 | 说明 |
|------|------|
| `src/state/` | 状态管理目录 |
| `src/context.ts` | 上下文定义 |
| `src/context/` | Context 组件 |

### 9.2 Store 实现

| 文件 | 说明 |
|------|------|
| `src/ink/store.ts` | Ink Store 实现 |
| `src/hooks/` | 状态相关 Hooks |

## 十、文件索引

| 目录/文件 | 路径 |
|-----------|------|
| 状态管理 | `src/state/` |
| 上下文 | `src/context/`, `src/context.ts` |
| Hooks | `src/hooks/` |
| Ink Store | `src/ink/store.ts` |
| 组件状态 | `src/components/` |
| 统计 Provider | `src/ink/components/StatsProvider/` |
