# Plugin (插件) 系统架构分析

## 一、概述

Claude Code 的插件系统实现了完整的扩展机制，支持从市场安装、内置插件、会话级插件到组织管理插件的多种类型。

### 1.1 核心目录

| 目录 | 职责 |
|------|------|
| `src/services/plugins/` | 插件服务层（安装管理、CLI 命令） |
| `src/utils/plugins/` | 插件工具函数（加载、解析、更新） |
| `src/plugins/` | 内置插件定义 |

### 1.2 核心文件

| 文件 | 职责 |
|------|------|
| `pluginOperations.ts` | 核心操作（安装、卸载、启用、禁用、更新） |
| `PluginInstallationManager.ts` | 后台启动协调 |
| `pluginCliCommands.ts` | CLI 命令封装 |
| `pluginLoader.ts` | 插件加载器 |
| `marketplaceManager.ts` | 市场管理 |
| `dependencyResolver.ts` | 依赖解析 |
| `pluginPolicy.ts` | 策略执行 |
| `installedPluginsManager.ts` | 已安装插件管理 |
| `reconciler.ts` | 市场协调器 |
| `refresh.ts` | 刷新系统 |
| `pluginAutoupdate.ts` | 自动更新 |

## 二、插件类型

### 2.1 四种插件类型

| 类型 | ID 格式 | 来源 | 持久化 |
|------|---------|------|--------|
| **Marketplace** | `{name}@{marketplace}` | URL、GitHub、git、本地 | 全局 (`~/.claude/plugins/`) |
| **Builtin** | `{name}@builtin` | 硬编码在 `src/plugins/builtinPlugins.ts` | N/A（随 CLI 分发） |
| **Session-only (Inline)** | `{name}@inline` | `--plugin-dir` CLI 标志 | 仅会话，不持久化 |
| **Managed** | `{name}@{marketplace}` | 组织策略通过 `policySettings` | 全局，策略强制执行 |

### 2.2 Builtin 插件

- 通过 `registerBuiltinPlugin()` 在启动时注册
- 出现在 `/plugin` UI 中
- 用户可启用/禁用
- 可提供 skills、hooks 和 MCP 服务器

### 2.3 Session-only 插件

- 使用合成市场 `inline`
- 不能有意义地被其他插件依赖
- 裸依赖仅按名称匹配

### 2.4 Managed 插件

- 由组织策略在 `policySettings.enabledPlugins` 中锁定
- `isPluginBlockedByPolicy()` 检查是否被强制禁用
- 用户无法安装/卸载

## 三、安装流程与依赖解析

### 3.1 安装流程

```
1. 解析插件标识符: parsePluginIdentifier() 拆分 name@marketplace
2. 市场查找: getPluginById() 从市场缓存解析
3. 依赖解析: resolveDependencyClosure() DFS 遍历传递依赖
4. 策略检查: isPluginBlockedByPolicy() 检查组织策略
5. 写入设置: 将整个闭包写入 enabledPlugins
6. 缓存插件: cachePlugin() 处理 npm、GitHub、git、本地路径
7. 注册: addPluginInstallation() 写入 installed_plugins.json (V2)
8. 清除缓存: 清除所有插件缓存
```

### 3.2 依赖解析 (`dependencyResolver.ts`)

**安装时解析**：`resolveDependencyClosure()`

| 特性 | 说明 |
|------|------|
| **DFS 遍历** | 深度优先搜索所有传递依赖 |
| **循环检测** | 返回 `{ ok: false, reason: 'cycle', chain }` |
| **跨市场阻止** | 市场 A 的插件不能从市场 B 自动安装（安全边界） |
| **已启用跳过** | 避免对已安装的依赖进行意外设置写入 |

**加载时验证**：`verifyAndDemote()`

- 固定点循环检查所有启用插件的 manifest 依赖
- 如果依赖未启用，则将依赖插件降级（仅会话级禁用，不写入设置）

### 3.3 反向依赖

`findReverseDependents()`：卸载/禁用时警告其他插件依赖于此插件。

## 四、市场系统

### 4.1 市场来源 (`MarketplaceSource`)

| 类型 | 说明 |
|------|------|
| `url` | HTTP(S) 端点返回 `marketplace.json` |
| `github` | `{ repo: "owner/repo", ref?, sha? }` |
| `git` | `{ url: "https://..." }` |
| `git-subdir` | `{ url, path, ref?, sha? }` 用于 monorepo |
| `file` | 本地 JSON 文件 |
| `directory` | 本地目录包含 `.claude-plugin/marketplace.json` |
| `npm` | npm 包注册表 |
| `pip` | pip 包注册表（尚未实现） |

### 4.2 三层架构

| 层 | 说明 |
|-----|------|
| **Layer 1 (Intent)** | 设置文件声明应存在哪些市场 (`extraKnownMarketplaces`) |
| **Layer 2 (Materialization)** | `~/.claude/plugins/` — `reconcileMarketplaces()` 使 `known_marketplaces.json` 一致 |
| **Layer 3 (Active Components)** | AppState — `refreshActivePlugins()` 交换命令、代理、hooks、MCP 服务器 |

### 4.3 市场协调 (`reconciler.ts`)

```
diffMarketplaces()
    → 比较声明意图与物化状态
        → 产生 missing、sourceChanged、upToDate 列表

reconcileMarketplaces()
    → 安装 missing
    → 更新 sourceChanged
    → 通过 addMarketplaceSource()
```

### 4.4 种子目录

只读插件种子（如容器镜像）通过 `registerSeedMarketplaces()` 注册市场：

- 种子条目始终胜出
- `autoUpdate` 强制为 false

## 五、插件加载

### 5.1 加载流程 (`pluginLoader.ts`)

```
1. 加载内置插件: getBuiltinPlugins()
2. 发现市场插件: 从 settings (enabledPlugins)
3. 解析插件路径: 版本化缓存路径 → 种子缓存探测 → 遗留路径
4. 加载 Manifest: .claude-plugin/plugin.json 或根 plugin.json
5. 组件自动检测（并行）:
   ├── commands/ → commandsPath
   ├── agents/ → agentsPath
   ├── skills/ → skillsPath
   └── output-styles/ → outputStylesPath
6. Manifest 扩展组件: manifest.commands、manifest.agents 等
7. 加载 Hooks: hooks/hooks.json 验证
8. MCP/LSP 服务器: 从 .mcp.json 或 manifest 懒加载
```

### 5.2 LoadedPlugin 结构

| 字段 | 说明 |
|------|------|
| `name` | 插件名称 |
| `manifest` | 插件清单 |
| `path` | 插件路径 |
| `source` | 来源 |
| `repository` | 仓库 |
| `enabled` | 是否启用 |
| `isBuiltin` | 是否内置 |
| `hooksConfig` | Hooks 配置 |
| `mcpServers` | MCP 服务器 |
| `lspServers` | LSP 服务器 |
| `commandsPath/commandsPaths` | 命令路径 |
| `agentsPath/agentsPaths` | 代理路径 |
| `skillsPath/skillsPaths` | Skills 路径 |
| `outputStylesPath` | 输出样式路径 |
| `settings` | 设置 |
| `sha` | Git SHA |

## 六、策略执行

### 6.1 单一检查点 (`pluginPolicy.ts`)

```typescript
function isPluginBlockedByPolicy(pluginId: string): boolean
```

- 读取 `policySettings.enabledPlugins` 从管理设置
- 如果策略显式设置为 `false`，返回 `true`
- 用于安装检查点、启用操作和 UI 过滤的单一事实来源

### 6.2 管理插件名称

`getManagedPluginNames()` 返回策略中声明的插件名称集合（包含 `@` 的 ID）。

## 七、自动更新

### 7.1 更新机制 (`pluginAutoupdate.ts`)

```
1. 跳过检查: shouldSkipPluginAutoupdate()
2. 市场选择: getAutoUpdateEnabledMarketplaces() — 官方 Anthropic 市场默认 autoUpdate: true
3. 市场刷新: refreshMarketplace() (git pull 或重新下载)
4. 插件更新: updatePluginsForMarketplaces() — 遍历 installed_plugins.json
5. 非原地更新: 仅磁盘更新，需要重启生效
6. 通知: onPluginsAutoUpdated() 通知 REPL 显示重启提示
```

### 7.2 版本分离

- 内存会话状态 (`inMemoryInstalledPlugins`) 不被后台操作更新
- `hasPendingUpdates()` / `getPendingUpdatesDetails()` 检测磁盘与内存差异

## 八、刷新系统

### 8.1 刷新流程 (`refresh.ts`)

`refreshActivePlugins()` 是 Layer-3 原语：

```
1. 清除所有缓存: clearAllCaches(), clearPluginCacheExclusions()
2. 完整插件重载: loadAllPlugins()
3. 组件重载（并行）: getPluginCommands(), getAgentDefinitionsWithOverrides()
4. MCP/LSP 加载: 填充 mcpServers/lspServers
5. AppState 更新: 设置 enabled/disabled/plugins.commands/agentDefinitions
6.  bumped mcp.pluginReconnectKey: 触发 MCP 重连
7. LSP 重新初始化: reinitializeLspServerManager()
8. Hook 加载: loadPluginHooks() 带错误隔离
```

### 8.2 调用点

| 调用点 | 说明 |
|--------|------|
| `/reload-plugins` | 用户命令 |
| `print.ts` | 无头自动刷新 |
| `performBackgroundPluginInstallations()` | 后台安装完成 |

## 九、已安装插件注册表

### 9.1 文件格式 (V2)

**文件**：`~/.claude/plugins/installed_plugins.json`

```json
{
  "version": 2,
  "plugins": {
    "plugin-name@marketplace": [
      {
        "scope": "user" | "managed" | "project" | "local",
        "installPath": "/absolute/path/to/versioned/cache/dir",
        "version": "1.2.3" | "sha-hash",
        "installedAt": "2024-01-15T10:30:00.000Z",
        "lastUpdated": "2024-01-15T10:30:00.000Z",
        "gitCommitSha": "abc123...",
        "projectPath": "/path/to/project"
      }
    ]
  }
}
```

### 9.2 关键设计决策

| 决策 | 说明 |
|------|------|
| **数组每插件** | 支持多范围安装（user + project + local） |
| **范围相关性** | `isInstallationRelevantToCurrentProject()` — user/managed 始终相关 |
| **迁移** | V1 格式在启动时自动转换为 V2 |
| **内存与磁盘分离** | `inMemoryInstalledPlugins` 是会话快照 |
| **孤儿追踪** | 移除的插件路径通过 `.orphaned_at` 标记清理 |

### 9.3 版本化缓存

```
~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/
```

- 版本从 git SHA 或内容哈希计算
- **Zip 缓存模式**：`PLUGIN_ZIP_CACHE=1` 存储 `.zip` 文件

## 十、插件扩展点

插件可贡献以下扩展点：

| 扩展点 | 说明 |
|--------|------|
| **Slash 命令** | `commands/` 目录中的 Markdown 文件或 manifest 声明 |
| **Agents** | `agents/` 目录中的 AI 代理定义 |
| **Skills** | `skills/` 目录中可发现的技能 |
| **Hooks** | `hooks/hooks.json` 或 manifest 中定义的生命周期拦截器 |
| **MCP 服务器** | 内联、JSON 文件或 `.mcpb` 包 |
| **LSP 服务器** | Language Server Protocol 集成 |
| **输出样式** | 工具结果的自定义格式 |
| **用户配置** | 启用时基于提示的配置 |

## 十一、CLI 命令

### 11.1 命令封装 (`pluginCliCommands.ts`)

| 命令 | 说明 |
|------|------|
| `installPlugin(plugin, scope)` | 非交互安装 |
| `uninstallPlugin(plugin, scope, keepData)` | 非交互卸载 |
| `enablePlugin(plugin, scope)` | 启用插件 |
| `disablePlugin(plugin, scope)` | 禁用插件 |
| `disableAllPlugins()` | 禁用所有插件 |
| `updatePluginCli(plugin, scope)` | 更新插件 |

### 11.2 遥测

所有 CLI 命令发出遥测事件：

- `tengu_plugin_installed_cli`
- `tengu_plugin_uninstalled_cli`
- `tengu_plugin_enabled_cli`
- `tengu_plugin_disabled_cli`

## 十二、文件索引

| 文件 | 路径 |
|------|------|
| 插件操作 | `src/services/plugins/pluginOperations.ts` |
| 安装管理 | `src/services/plugins/PluginInstallationManager.ts` |
| CLI 命令 | `src/services/plugins/pluginCliCommands.ts` |
| 插件加载 | `src/utils/plugins/pluginLoader.ts` |
| 市场管理 | `src/utils/plugins/marketplaceManager.ts` |
| 依赖解析 | `src/utils/plugins/dependencyResolver.ts` |
| 策略执行 | `src/utils/plugins/pluginPolicy.ts` |
| 已安装管理 | `src/utils/plugins/installedPluginsManager.ts` |
| 市场协调 | `src/utils/plugins/reconciler.ts` |
| 刷新系统 | `src/utils/plugins/refresh.ts` |
| 自动更新 | `src/utils/plugins/pluginAutoupdate.ts` |
