# 本地插件

## 放置位置

- **全局**：`~/.config/opencode/plugin/` 或 `~/.config/opencode/plugins/`（所有项目可用）
- **项目级**：`.opencode/plugin/` 或 `.opencode/plugins/`（仅当前项目）

单数 `plugin/` 与复数 `plugins/` **两种目录名都会自动加载**（opencode 按 `{plugin,plugins}/*.{ts,js}` 扫描这两个目录）。

**放进插件目录即自动加载，不要再把它写进 `opencode.json` 的 `plugin` 数组** —— 配置里的 `plugin` 数组是给 **npm 包 / 远程 spec** 用的（见 [npm-plugin.md](npm-plugin.md)）。本地文件重复登记虽然会按文件 URL 去重（不会加载两次），但纯属多余，而且一旦路径写错就会让 opencode 报「找不到模块 / 加载失败」。

### ⚠️ 踩坑反例（真实事故）

1. **产物没生成到插件目录**：插件源不是手写文件，而是由模板 / 构建脚本生成，只改了源、**没执行生成步骤** ⇒ 插件目录下没有该文件 ⇒ 自动发现不到；若配置数组里又写了它的路径，opencode 会直接报加载失败。
2. **又写进 `plugin` 数组**：文件已在插件目录下自动加载，再往数组里加一遍是多余的（本地文件只靠目录加载，数组只留给 npm 包）。

> 去重规则：同一**本地文件**重复登记会按精确 `file://` URL 去重；同一 **npm 包名**也只加载一次。但一个本地文件与同名 npm 包会被各自加载。

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

**OpenCode 的服务端进程和 TUI 进程是两个独立 OS 进程**，各自维护自己的 Bun 模块缓存，插件模块会被两个进程各 `import()` 一次。

```
服务端进程 ── import(插件模块) ──► 模块顶层代码执行 ──► 调用插件函数（每目录一次）──► hooks
TUI 进程   ── import(插件模块) ──► 模块顶层代码执行 ──► 调用 default.tui()（每目录一次）
```

**先理解"目录"指什么：** 这里的"目录"是 **opencode 当前打开的项目工作目录**（`worktree`），不是聊天里的"文件夹概念"。插件实例绑定在打开的项目上——你在 `/a` 目录启动 opencode，就有一个绑定 `/a` 的插件实例；切换到 `/b` 目录，opencode 会新建一个绑定 `/b` 的插件实例。同一个 `/a` 下的所有会话（含子 agent）都共享 `/a` 这一个实例。

插件状态按**三层缓存维度**，理解这层决定变量放哪里：

| 内容 | 缓存维度 | 行为 |
|------|---------|------|
| **模块顶层变量 / 副作用** | **进程级** | Bun 模块缓存保证整个进程只 `import()` 一次（无论打开多少个目录都不重复）；server 和 TUI 各有一份，互不相通 |
| **插件函数体 + hooks 闭包变量** | **目录级** | `InstanceState` 按工作目录缓存（`ScopedCache.get(cache, directory)`）：每个打开的目录对应一份独立闭包，同目录所有会话共享；切换到另一个目录 → 插件函数重新执行、生成新闭包 |
| **hook 回调**（event / tool.execute.*） | **事件级** | 每次事件/调用触发，可访问所在闭包的共享变量 |

**实际含义：**
- 在一个项目目录里开多个会话 → **共享同一份插件闭包变量**
- opencode 打开另一个项目目录 → **插件函数重新执行**，产生新的 hooks 闭包（目录间隔离）
- 进程内打开多个目录**不会**重复执行模块顶层代码（模块缓存是进程级）
- server 和 TUI 两进程之间，无论顶层还是闭包都互不可见

### 顶层该放什么

```ts
import createLogger from "@xiaoqiong0v0/opencode-plugin-logger"

// ✅ 顶层放无副作用的纯定义（函数、常量、logger 实例）——进程级，跨目录共享
const log = createLogger("my-plugin")
const MAX_RETRY = 3

// ⚠️ 顶层放有副作用的操作 —— 会在 server 和 TUI 各执行一次，且跨目录共享状态
// try { loadCfg() } catch (e) { log.error("初始化失败", e) }
// 如果操作不幂等（重复注册、重复连接），会产生两次副作用

// ✅ 有副作用的初始化放进插件函数内部 —— 目录级，每个打开的项目目录执行一次（同目录会话共享）
export const MyPlugin = async () => {
  try { loadCfg() } catch (e) { log.error("初始化失败", e) }
  return { /* hooks */ }
}
```

**原则：**
- 顶层只放**无副作用的纯定义**（函数、常量、logger 实例），进程级共享
- 所有**有副作用的初始化**（连接、注册、写状态）放进插件函数或钩子内部，目录级执行
- 若必须在顶层做有副作用操作，要保证**幂等**（重复执行无额外影响）

### 幂等的保证方式

幂等的本质是**重复执行的结果与执行一次相同**。插件场景按层次分三种做法：

**① 进程内防重复（内存标志）** — 只防同一进程内重复，无法跨进程：

```ts
let initialized = false

export const MyPlugin = async () => {
  if (initialized) return {}   // 已初始化直接返回
  initialized = true
  // 真正初始化...
  return { /* hooks */ }
}
```

**② 防重复注册（去重 / 先解绑再绑定）** — 应对 hook 回调被 `reload()` 多次触发：

```ts
const registered = new Set<string>()

async function registerOnce(name: string, fn: () => void) {
  if (registered.has(name)) return
  registered.add(name)
  // 注册逻辑
}
```

**③ 跨进程幂等（文件锁 / 原子操作 / 状态检查）** — **保证真正幂等的关键**，server/TUI 不同进程内存不共享，必须用外部状态：

```ts
import { mkdirSync } from "node:fs"
import os from "node:os"
import path from "node:path"

const lockPath = path.join(os.tmpdir(), "my-plugin-init.lock")

// mkdir 是原子操作——谁先创建成功谁就拿到锁
try {
  mkdirSync(lockPath)           // 已存在会抛错
  await doExpensiveInit()       // 只有拿到锁的进程执行
} catch {
  // 另一个进程已初始化，跳过
}
```

其他跨进程手段：
- **写文件**：先 `existsSync` 检查，或用 `writeFileSync(flag: "wx")`（不存在才创建）
- **连接服务**：先探测端口/连接状态，已连接则复用
- **数据库**：`INSERT ... ON CONFLICT DO NOTHING`（upsert）
- **启动子进程**：先检查进程是否已存在

**操作幂等性判断表：**

| 操作 | 幂等？ | 处理 |
|------|--------|------|
| 读配置/读文件 | ✅ 天然幂等 | 顶层放也没事 |
| 定义常量/函数 | ✅ 天然幂等 | 顶层随便放 |
| 注册 hook 回调 | ⚠️ 重复会叠加 | 去重 / 先解绑 |
| 写文件/建目录 | ⚠️ 重复会冲突 | 原子标志 / 先检查 |
| 连接服务/启动进程 | ❌ 重复会重复连 | 锁 / 状态检查 |
| 发通知/埋点 | ❌ 重复会重复发 | dedupe |

**结论：** 大部分场景把有副作用的初始化从顶层挪进插件函数内部（目录级，同目录一次）就够；只有当同一操作可能被**跨进程**（server/TUI）或**跨目录**触发时，才需要文件锁/状态检查这类真正幂等的机制。

## 启动性能：不要阻塞

OpenCode 启动时**串行 `await`** 加载每个插件（`bootstrap.run` → `applyPlugin`）。插件函数体里任何耗时操作都会**直接累加到启动时间里**——多个插件各慢 1 秒，启动就慢好几秒。

```ts
// ❌ 错误：阻塞启动
export const MyPlugin = async () => {
  const cfg = await loadHugeConfig()      // 读大文件，慢
  await connectToRemoteServer()           // 网络请求，更慢
  return { /* hooks */ }
}

// ✅ 正确：函数体只做同步注册，耗时任务丢后台
export const MyPlugin = async () => {
  void init()                             // 不 await，后台异步跑，不阻塞启动
  return { /* hooks */ }
}

let cfg: Config | undefined
async function init() {
  try {
    cfg = await loadHugeConfig()
    await connectToRemoteServer()
  } catch (e) {
    log.error("后台初始化失败", e)
  }
}
```

**要点：**
- **不要 `await` 耗时操作**：用 `void asyncFn()` 或 `queueMicrotask(...)` 让它后台跑
- **延迟初始化（lazy）**：非必需的东西等第一次真正用到时再初始化（比如在 tool 的 `execute` 里或 hook 回调里）
- **用事件代替启动时探测**：需要感知运行状态的，监听 `session.idle` / `session.created` 等事件，而不是启动时同步等待
- **后台初始化要幂等 + 容错**：后台任务失败不能让插件崩溃，用 `try/catch` 记日志（见上文"幂等的保证方式"）
- **注意时序**：后台初始化未完成时，hook 回调可能已到达——要么在回调里检查就绪状态，要么把依赖项也 lazy 化

**慢操作清单（都别在启动路径上 `await`）：**
- 读大文件 / 递归扫描目录
- 网络请求、API 调用
- 起子进程 / `$` shell 命令
- 数据库连接、大查询
- 加载重量级依赖（把 `import` 放函数体内做动态引入）

### 反例：加载期 `await` 一个会超时的操作

最常见的启动阻塞不是"读大文件"，而是**在插件函数体里 `await` 一个可能超时的操作**（等端口、等网络、等另一个进程就绪）。一旦目标没就绪，超时逻辑会让**每次都白等十几秒**；又因为加载是串行的，这段等待直接叠加到 opencode 启动时间上。

```ts
// ❌ 反例：加载期同步等待一个可能超时的操作
export const MyPlugin = async () => {
  const port = await waitForPort(4096, { timeout: 15_000 })  // 目标没起来就白等 15s
  return { /* hooks */ }
}

// ✅ 修法：懒加载 + 异步 + 短超时 + 失败即弃
let conn: Conn | undefined
async function ensureConn(): Promise<Conn | undefined> {
  if (conn) return conn
  try {
    conn = await connect({ timeout: 500 })   // 短超时，快速失败
  } catch {
    return undefined                          // 失败即弃，不缓存错误状态，下次再试
  }
  return conn
}

export const MyPlugin = async () => {
  void ensureConn()                           // 后台预热，不阻塞启动
  return {
    "tool.execute.before": async () => {
      const c = await ensureConn()            // 真正用到时才等待
      // ...
    },
  }
}
```

**通用模式：** 加载期不做任何"可能等待外部就绪"的 `await`；把这类操作改成**懒加载**（首次用到才做）、**异步**（不 `await`）、**短超时**（快速失败）、**失败即弃**（不缓存错误状态、下次重试），并配合就绪检查或事件触发。

## 注意

- 修改插件文件后**重启 OpenCode** 或重新加载配置即可生效
- 文件后缀可以是 `.js` 或 `.ts`（OpenCode 自动处理 TypeScript）。但 `.ts` 文件的依赖必须在 `.opencode/package.json` 中声明，否则类型/模块解析会失败。复杂项目建议先编译为 `.js` 再部署。
- 插件中的 `console.log`/`console.error` 输出会显示在 OpenCode 日志中
