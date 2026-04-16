# 终端 UI 架构分析 (Ink/React)

## 一、概述

Claude Code 使用 **React + Ink** 构建终端用户界面 (TUI)。Ink 是一个 React 渲染器，可以将 React 组件渲染到终端。

### 1.1 技术栈

| 技术 | 说明 |
|------|------|
| **React** | UI 组件框架 |
| **Ink** | 终端 React 渲染器 |
| **react-reconciler** | React 协调器 |
| **Yoga Layout** | Flexbox 布局引擎 |

## 二、UI 架构分层

```
App.tsx
    ├── Screen Components (屏幕组件)
    │       ├── MainLayout (主布局)
    │       ├── CommandPalette (命令面板)
    │       ├── HistoryPanel (历史面板)
    │       └── StatusBar (状态栏)
    │
    ├── UI Components (界面组件)
    │       ├── PromptInput (提示输入)
    │       ├── Messages (消息列表)
    │       ├── Diff (差异对比)
    │       └── Markdown (Markdown 渲染)
    │
    └── Custom Ink Engine (自定义 Ink 引擎)
            ├── DOM (虚拟 DOM)
            ├── Yoga Layout (Yoga 布局)
            ├── Render (渲染过程)
            ├── Blit/Diff (差异计算/传输)
            └── Terminal (终端输出)
```

## 三、屏幕组件

### 3.1 MainLayout (主布局)

负责整体布局管理：

- 顶部：欢迎信息与模型信息
- 中部：消息列表与输入区域
- 底部：状态栏

### 3.2 CommandPalette (命令面板)

斜杠命令面板：

- `/help` - 帮助
- `/compact` - 压缩上下文
- `/model` - 切换模型
- `/settings` - 设置
- `/plugins` - 插件管理

### 3.3 HistoryPanel (历史面板)

显示对话历史：

- 会话列表
- 会话恢复
- 会话删除

### 3.4 StatusBar (状态栏)

底部状态栏显示：

- 当前模型
- Token 使用量
- 权限模式
- 键盘提示

## 四、UI 组件

### 4.1 PromptInput (提示输入)

用户输入区域：

- 多行输入支持
- 自动补全
- 命令提示
- 附件支持

### 4.2 Messages (消息列表)

消息展示区域：

- 用户消息
- 助手消息
- 工具使用
- 错误提示

### 4.3 Diff (差异对比)

文件差异展示：

- 语法高亮
- 行号显示
- 变更高亮
- 接受/拒绝

### 4.4 Markdown (Markdown 渲染)

Markdown 内容渲染：

- 代码块
- 列表
- 表格
- 链接

## 五、自定义 Ink 引擎

### 5.1 DOM (虚拟 DOM)

Ink 维护一个虚拟 DOM 树：

- 组件树结构
- 属性追踪
- 子节点管理

### 5.2 Yoga Layout (Yoga 布局)

使用 Yoga Layout 引擎进行 Flexbox 布局：

- Flex 方向
- 对齐方式
- 边距与填充
- 尺寸约束

### 5.3 Render (渲染过程)

渲染流程：

```
虚拟 DOM → Yoga Layout → 像素输出 → Blit/Diff → 终端
```

### 5.4 Blit/Diff (差异计算/传输)

差异计算优化：

- 仅渲染变更部分
- 光标位置优化
- ANSI 转义序列

### 5.5 Terminal (终端输出)

最终输出到终端：

- ANSI 颜色
- 光标控制
- 窗口调整

## 六、全局功能

### 6.1 Vim Mode (Vim 模式)

支持 Vim 键绑定：

- 正常模式
- 插入模式
- 视觉模式
- 命令模式

### 6.2 Keybindings (快捷键)

全局快捷键：

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+C` | 中断当前操作 |
| `Ctrl+L` | 清屏 |
| `Ctrl+O` | 展开/折叠 |
| `Ctrl+N/P` | 上下导航 |

### 6.3 Selection (选区)

文本选择功能：

- 鼠标选择
- 键盘选择
- 复制到剪贴板

### 6.4 Focus (焦点管理)

焦点管理：

- 输入焦点
- 组件焦点
- 焦点环显示

## 七、组件目录结构

`src/components/` (389 个文件) 主要子目录：

| 目录 | 说明 |
|------|------|
| `agents/` | 代理相关组件 |
| `CustomSelect/` | 自定义选择组件 |
| `design-system/` | 设计系统组件 |
| `diff/` | 差异对比组件 |
| `FeedbackSurvey/` | 反馈调查 |
| `permissions/` | 权限请求组件 |
| `messages/` | 消息组件 |
| `mcp/` | MCP 相关组件 |
| `memory/` | 记忆相关组件 |
| `skills/` | Skills 组件 |
| `tasks/` | 任务组件 |
| `teams/` | 团队组件 |
| `Settings/` | 设置组件 |
| `shell/` | Shell 组件 |
| `StructuredDiff/` | 结构化差异 |

## 八、Ink 框架

`src/ink/` (96 个文件) 主要子目录：

| 目录 | 说明 |
|------|------|
| `components/` | Ink 基础组件 |
| `events/` | 事件处理 |
| `hooks/` | Ink Hooks |
| `layout/` | 布局管理 |
| `termio/` | 终端 I/O |

## 九、入口点

| 文件 | 说明 |
|------|------|
| `src/entrypoints/cli.tsx` | CLI 入口 |
| `src/main.tsx` | TUI 主逻辑 |
| `src/screens/REPL.tsx` | REPL 界面 |

## 十、文件索引

| 目录 | 路径 |
|------|------|
| 组件 | `src/components/` |
| Ink 框架 | `src/ink/` |
| 入口点 | `src/entrypoints/` |
| 屏幕 | `src/screens/` |
| UI 工具 | `src/utils/ui/` |
