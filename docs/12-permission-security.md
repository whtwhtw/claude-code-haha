# 权限安全模型架构分析

## 一、概述

Claude Code 实现了多层级的权限安全模型，确保工具执行的安全性和可控性。

### 1.1 整体架构

```
Trust Model (信任模型)          Permission Engine (权限引擎)          Sandbox System (沙箱系统)
├── User Whitelist (用户白名单)   ├── Ask Mode (询问模式)                ├── Isolated Environment (隔离环境)
├── Domain Allowlist (域名许可)   ├── Auto Mode (自动模式)                ├── Resource Limits (资源限制)
├── Behavior Analysis (行为分析)  └── Bypass Mode (旁路模式)              ├── Network Policy (网络策略)
├── Reputation Score (信誉评分)                                       └── Process Namespace (进程命名空间)
└── Reputation (信誉)
```

### 1.2 工具请求流程

```
Tool Request (工具请求)
    → Hook (钩子)
        → Permission Gate (权限网关)
            ├── allow → Bash Security (5层) → Sandbox (沙箱) → Execute (执行)
            └── deny  → Reject (拒绝)
```

## 二、信任模型

### 2.1 用户白名单

允许特定用户自动执行某些工具：

- 基于用户身份
- 可配置的白名单规则
- 支持通配符

### 2.2 域名许可名单

允许访问特定域名：

- Web 搜索域名
- API 端点域名
- MCP 服务器域名

### 2.3 行为分析

分析工具使用模式：

- 频率检测
- 异常检测
- 模式识别

### 2.4 信誉评分

基于历史行为计算信誉：

- 成功执行率
- 错误率
- 用户反馈

## 三、权限引擎

### 3.1 三种模式

| 模式 | 说明 | 使用场景 |
|------|------|----------|
| **Ask Mode (询问模式)** | 每次工具执行前询问用户 | 默认模式，高安全性 |
| **Auto Mode (自动模式)** | 白名单工具自动执行 | 可信工具，提高效率 |
| **Bypass Mode (旁路模式)** | 跳过权限检查 | 自动化场景，脚本模式 |

### 3.2 权限网关

`Permission Gate` 是权限决策的核心：

```typescript
interface PermissionGate {
  tool: string;
  input: Record<string, unknown>;
  context: ToolUseContext;
  
  check(): PermissionResult;
}

type PermissionResult = 
  | { type: 'allow' }
  | { type: 'deny' }
  | { type: 'ask', prompt: string };
```

### 3.3 权限决策流程

```
工具请求
    → 检查 Bypass Mode
        → 是 → 允许
        → 否 → 检查 Auto Mode 白名单
            → 在白名单 → 允许
            → 不在 → Ask Mode
                → 用户确认 → 允许
                → 用户拒绝 → 拒绝
```

## 四、Bash 安全 (5 层)

### 4.1 Validation (验证)

命令验证层：

- 语法检查
- 命令白名单
- 参数验证

### 4.2 Sanitization (清理)

命令清理层：

- 转义特殊字符
- 移除危险序列
- 规范化路径

### 4.3 Restriction (限制)

命令限制层：

- 禁止危险命令（rm -rf, mkfs 等）
- 路径限制
- 资源限制

### 4.4 Monitoring (监控)

执行监控层：

- 超时监控
- 资源监控
- 行为监控

### 4.5 Audit (审计)

审计日志层：

- 命令记录
- 结果记录
- 时间戳

## 五、沙箱系统

### 5.1 隔离环境

```
Isolated Environment (隔离环境)
    ├── 文件系统隔离
    ├── 进程隔离
    ├── 网络隔离
    └── 环境变量隔离
```

### 5.2 资源限制

| 资源 | 限制 |
|------|------|
| CPU | 核心数限制 |
| 内存 | 内存上限 |
| 磁盘 | 磁盘配额 |
| 网络 | 带宽限制 |
| 时间 | 执行超时 |

### 5.3 网络策略

| 策略 | 说明 |
|------|------|
| 允许列表 | 允许访问的网络地址 |
| 拒绝列表 | 禁止访问的网络地址 |
| 代理配置 | 网络代理设置 |

### 5.4 进程命名空间

进程隔离：

- 独立 PID 命名空间
- 限制进程数量
- 防止进程逃逸

## 六、Hook 系统

### 6.1 钩子类型

| 钩子 | 时机 | 用途 |
|------|------|------|
| **Pre-execution Hook** | 工具执行前 | 权限检查、参数验证 |
| **Post-execution Hook** | 工具执行后 | 结果处理、审计日志 |
| **Error Hook** | 工具执行失败 | 错误处理、重试 |

### 6.2 钩子注册

```typescript
interface Hook {
  name: string;
  priority: number;
  handler: (context: HookContext) => HookResult;
}
```

## 七、权限配置

### 7.1 配置文件

权限配置存储在：

- `~/.claude/settings.json` (用户级)
- `.claude/settings.json` (项目级)
- `managed-settings.json` (组织级)

### 7.2 权限规则

```json
{
  "permissions": {
    "tools": {
      "Bash": {
        "mode": "ask",
        "allowCommands": ["ls", "cat", "grep"],
        "denyCommands": ["rm -rf", "mkfs", "dd"]
      },
      "FileWrite": {
        "mode": "auto",
        "allowedPaths": ["./src/**", "./tests/**"]
      }
    }
  }
}
```

## 八、安全最佳实践

### 8.1 最小权限原则

- 默认拒绝所有
- 按需授权
- 定期审查

### 8.2 防御深度

- 多层防护
- 独立验证
- 失败安全

### 8.3 审计追踪

- 完整日志
- 不可篡改
- 定期审查

## 九、文件索引

| 目录/文件 | 路径 |
|-----------|------|
| 权限系统 | `src/utils/permissions/` |
| 权限组件 | `src/components/permissions/` |
| 权限钩子 | `src/hooks/toolPermission/` |
| Bash 安全 | `src/tools/BashTool/` |
| 沙箱 | `src/utils/sandbox/` |
| Hook 系统 | `src/utils/hooks/` |
