# 本地插件

## 放置位置

- **全局**：`~/.config/opencode/plugins/*.js`（所有项目可用）
- **项目级**：`.opencode/plugins/*.js`（仅当前项目）

自动加载，无需在 `opencode.json` 中注册。

## 依赖管理

如果插件需要外部包，在插件所在目录下创建 `package.json`：

```json
{
  "type": "module",
  "dependencies": {
    "shescape": "^2.1.0"
  }
}
```

**必须添加 `"type": "module"`**，否则 Bun/Node.js 会将 `.js` 文件按 CommonJS 解析，`import`/`export` 语法直接报错。

OpenCode 启动时自动运行 `bun install` 安装依赖。

## 基本结构

插件导出一个异步函数，函数名就是插件名：

```ts
export const MyPlugin = async ({ project, client, $, directory, worktree }) => {
  return {
    // 返回工具（tool）或钩子（event/hook）
  }
}
```

参数 `{ project, client, $, directory, worktree }` 全部可选，按需解构即可：

| 参数 | 说明 |
|------|------|
| `project` | 当前项目信息 |
| `directory` | 当前工作目录 |
| `worktree` | Git 工作树路径 |
| `client` | OpenCode SDK 客户端，用于 AI 交互和结构化日志 |
| `$` | Bun 的 Shell API，用于执行命令 |

TypeScript 版本：

```ts
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async ({ project, client, $, directory, worktree }) => {
  return {
    // 类型安全的钩子实现
  }
}
```

## 模块初始化陷阱：Temporal Dead Zone（TDZ）

模块顶层的 `const`/`let` 声明存在**暂时性死区（TDZ）**，在声明之前引用会抛出 `ReferenceError`。

**错误示例（插件加载静默崩溃）：**

```ts
// ...
loadCfg()       // ❌ 调用时 T 尚未初始化

const TX = { ... }
const T = () => { ... }  // T 定义在 loadCfg 之后
```

**正确做法：所有 `const` 定义必须在任何模块级调用之前：**

```ts
// ...
const TX = { ... }
const T = () => { ... }  // ✅ 先定义

loadCfg()       // ✅ 调用时 T 已就绪
```

**推荐：用 `try/catch` 包裹模块级调用，异常时用 logger 记录：**

```ts
try { loadCfg() } catch (e) { log.error("初始化失败", e) }
```

## 文件组织

简单插件的所有逻辑写在一个文件里。复杂插件可以拆分为多个文件，入口文件导出插件：

```
~/.config/opencode/plugins/
├── package.json          # 依赖声明（可选）
├── my-plugin.js          # 入口，export const MyPlugin = ...
└── helpers/              # 辅助模块（可选）
    └── utils.js
```

## 模块加载与进程模型

**OpenCode 的服务端进程和 TUI 进程是两个独立 OS 进程，各自维护自己的 Bun 模块缓存，插件模块会被两个进程各 `import()` 一次。**

```
服务端进程 ── import(插件模块) ──► 顶层代码执行一次 ──► 调用 default.server()
TUI 进程   ── import(插件模块) ──► 顶层代码执行一次 ──► 调用 default.tui()
```

因此：

| 代码位置 | 执行次数 | 原因 |
|---------|---------|------|
| 模块顶层（副作用代码） | **2 次** | server + TUI 各执行一次 |
| 插件函数内部 | **1 次** | 只在 server 进程调用 |
| 顶层纯变量定义 | 2 份独立实例 | 两个进程各持一份，互不相通 |

### 顶层该放什么

```ts
import createLogger from "@xiaoqiong0v0/opencode-plugin-logger"

// ✅ 顶层放无副作用的纯定义（函数、常量、logger 实例）——正常
const log = createLogger("my-plugin")
const MAX_RETRY = 3

// ⚠️ 顶层放有副作用的操作 —— 会在 server 和 TUI 各执行一次
// try { loadCfg() } catch (e) { log.error("初始化失败", e) }
// 如果操作不幂等（重复注册、重复连接），会产生两次副作用

// ✅ 有副作用的初始化放进插件函数内部，保证只执行一次
export const MyPlugin = async () => {
  try { loadCfg() } catch (e) { log.error("初始化失败", e) }
  return { /* hooks */ }
}
```

**原则：**
- 顶层只放**无副作用的纯定义**（函数、常量、logger 实例）
- 所有**有副作用的初始化**（连接、注册、写状态）放进插件函数或钩子内部
- 若必须在顶层做有副作用操作，要保证**幂等**（重复执行无额外影响）

## 注意

- 修改插件文件后**重启 OpenCode** 或重新加载配置即可生效
- 文件后缀可以是 `.js` 或 `.ts`（OpenCode 自动处理 TypeScript）。但 `.ts` 文件的依赖必须在 `.opencode/package.json` 中声明，否则类型/模块解析会失败。复杂项目建议先编译为 `.js` 再部署。
- 插件中的 `console.log`/`console.error` 输出会显示在 OpenCode 日志中
