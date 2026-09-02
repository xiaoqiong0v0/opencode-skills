# 自定义工具

## 基本定义

使用 `tool()` 辅助函数创建自定义工具，参数使用 Zod schema 做校验：

```ts
import { type Plugin, tool } from "@opencode-ai/plugin"

export const CustomToolsPlugin: Plugin = async (ctx) => {
  return {
    tool: {
      mytool: tool({
        description: "工具描述——模型根据此描述决定是否调用",
        args: {
          foo: tool.schema.string().describe("参数说明（模型据此生成参数值）"),
          bar: tool.schema.number().optional().describe("可选数字参数"),
        },
        async execute(args, context) {
          const { directory, worktree } = context
          return `处理结果: ${args.foo}`
        },
      }),
    },
  }
}
```

## ToolContext

`execute` 函数的第二个参数 `context` 包含以下字段：

| 字段                              | 类型            | 说明                                                                   |
| --------------------------------- | --------------- | ---------------------------------------------------------------------- |
| `sessionID`                     | `string`      | 当前会话 ID                                                            |
| `messageID`                     | `string`      | 触发消息 ID                                                            |
| `callID`                        | `string`      | 工具调用 ID                                                            |
| `agent`                         | `string`      | 代理类型（`build`/`general`/`pro`/`explore`）                  |
| `directory`                     | `string`      | 项目工作目录                                                           |
| `worktree`                      | `string`      | Git 工作树路径                                                         |
| `abort`                         | `AbortSignal` | 取消信号，支持超时/手动取消                                            |
| `extra`                         | `object`      | 模型信息（`model.id`、`model.providerID` 等）                      |
| `time`                          | `number`      | 时间戳                                                                 |
| `metadata({ title, metadata })` | `function`    | 设置工具结果标题和附加元数据                                           |
| `ask(input)`                    | `function`    | 运行时向用户询问权限（`{ permission, patterns, always, metadata }`） |

**注意：**

- `metadata` 在某些 OpenCode 版本中可能为 `undefined`，调用时建议使用可选链：`context.metadata?.({ title: "结果" })`。
- `execute` 的 context 中**没有** `parentID`。需要跟踪父子会话关系，请通过 `session.created` 事件监听（见 [hooks.md](hooks.md) 的会话管理章节）。

## 参数 Schema

`tool.schema` 基于 Zod，支持所有 Zod 类型方法：

```ts
tool.schema.string()           // 字符串
tool.schema.number()           // 数字
tool.schema.boolean()          // 布尔
tool.schema.enum(["a", "b"])   // 枚举
tool.schema.array(tool.schema.string())  // 数组
```

每个类型后可以链式调用：

- `.optional()` — 可选参数
- `.default(val)` — 默认值
- `.describe("说明")` — 参数说明（模型据此理解参数含义）
- `.min()` / `.max()` — 长度/数值限制

## 覆盖规则

- 插件工具与内置工具同名时，**插件工具优先**
- 多插件注册同名工具时，**加载顺序决定优先级**（全局 > 项目 > 先加载者优先）

## ToolResult

`execute` 函数的返回值可以是字符串或对象：

```ts
// 简单字符串
return "处理完成"

// 结构化结果（附加标题和附件）
return {
  title: "查询结果",
  output: "表格数据...",
  metadata: { rowCount: 42 },
  attachments: [{ type: "file", mime: "text/csv", url: "..." }],
}
```

## 单工具多命令模式（CLI 风格）

当插件的操作较多时，推荐**用一个工具承载全部操作，传入完整命令行字符串**（`'list'`、`'add --name 项目A'`），通过 `help` 子命令向模型返回用法。效果类似命令行 CLI，减少工具数量、集中逻辑、省 token。

### 基础结构

**命名约定：** 单工具多命令模式的工具名建议用 `<插件名>_cli`（如 `git_cli`、`db_cli`），让模型一眼识别这是"统一命令行入口"，且与其他工具区分开。

```ts
import stringArgv from "string-argv"   // 字符串 → argv 数组（处理引号/转义）
import { parseArgs } from "node:util"

export const MyPlugin: Plugin = async () => {
  return {
    tool: {
      my_cli: tool({
        description: "统一命令行工具。传入完整命令行字符串，如 'list' 或 'add --name 项目A'；用 'help' 查看全部用法。",
        args: {
          args: tool.schema.string().optional().describe("完整命令行字符串，如 'foo --param'；空时默认 'help'"),
        },
        async execute(args, context) {
          return handleCommand(args.args ?? "help", context)
        },
      }),
    },
  }
}

async function handleCommand(raw: string, context: ToolContext): Promise<string> {
  const tokens = stringArgv(raw)         // 正确分词（保留引号内空格）
  const [cmd, ...rest] = tokens
  if (!cmd) return HELP_TEXT             // 空命令返回 help
  switch (cmd) {
    case "help":
      return HELP_TEXT                   // 返回完整用法，引导模型正确调用
    case "list":
      return listItems()
    case "add":
      return addItem(rest, context)
    case "remove":
      return removeItem(rest)
    default:
      return `未知命令: ${cmd}\n\n${HELP_TEXT}`   // 未知命令也返回 help，引导自愈
  }
}
```

**关键点：**
- 工具只暴露**一个 `args` 字符串参数**，模型像写命令行一样调用，`help` 会引导它学会所有用法
- 用 `string-argv` 把字符串正确分词（`--name "项目 A"` 的引号内空格不会丢），再交给子命令处理
- 命令从分词结果的**第一个 token** 提取，剩余作为子命令参数
- `string-argv` 是 npm 依赖，需在插件目录的 `package.json` 中声明（见 [local-plugin.md](local-plugin.md) 依赖管理）

### 关键设计要点

1. **`help` 必须返回完整、结构化的用法**（所有子命令、参数、示例），这是模型"自学"的主要途径：

   ```ts
   const HELP_TEXT = `
   用法: my_cli <命令行字符串>

   子命令:
     list                    列出所有项
     add --name <n>          添加项
     remove <id>             删除项
     help                    显示此帮助

   示例:
     my_cli list
     my_cli add --name "项目A"
     my_cli help
   `
   ```

2. **未知命令时返回 help** —— 让模型从错误中自愈，而不是抛异常。

### 命令解析库选择

**不要手写参数解析**（`split(/\s+/)` 会漏掉引号、转义、`--key=value` 等情况）。标准做法分两步：**先用分词库把字符串转成数组，再用解析库处理数组**。

**① 字符串 → 数组（分词）**：主流解析库都要求数组（遵循 Unix argv 约定），没有"直接解析字符串"的库，所以需要分词器：

| 分词库 | 特点 |
|--------|------|
| `string-argv` | ⭐ 轻量，专为"字符串 → argv 数组"设计，正确处理引号/转义 |
| `shell-quote` | 更完整，支持通配符展开等 shell 语义 |

**② 数组 → 参数（解析）**：

| 需求 | 推荐 | 特点 |
|------|------|------|
| 简单 flag 解析，零依赖 | **`node:util.parseArgs`** ⭐ | Node 内置，Bun 原生兼容，够用 |
| 极简轻量、只要 flag | `arg` | 300 字节，无依赖 |
| 复杂子命令结构 | `clipanion` | 真正的子命令框架，类型安全 |
| 完整 CLI 框架 | `commander`/`yargs` | 重，倾向绑定 `process.argv`，插件场景不推荐 |

**完整组合**（字符串 → 数组 → 解析）：

```ts
import stringArgv from "string-argv"
import { parseArgs } from "node:util"

function run(raw: string) {
  const tokens = stringArgv(raw)        // 1. 字符串 → 数组（处理引号）
  const { values, positionals } = parseArgs({  // 2. 用成熟库解析
    args: tokens,
    options: {
      name: { type: "string", short: "n" },
      verbose: { type: "boolean", short: "v" },
    },
    allowPositionals: true,             // 允许位置参数
  })
  return { values, positionals }
}
// run(`add --name "项目 A"`)
// → { values: { name: "项目 A" }, positionals: ["add"] }  ✅ 引号内空格保留
```

### 何时用 / 何时不用

| 场景                                               | 建议                        |
| -------------------------------------------------- | --------------------------- |
| 操作多（>4 个）且共享状态/资源                     | ✅ 单工具多命令             |
| 操作少且相互独立                                   | ❌ 拆成多个工具更清晰       |
| 需要模型并行调用不同操作                           | ⚠️ 单工具会互相排队       |
| 参数差异大（如 add 需要 5 个参数，list 只要 1 个） | ✅ 单工具 + help 引导最合适 |

**原则：** 工具数量控制在 2 个以内（一个主工具 + 一个可选辅助）。多于 2 个时，考虑合并为单工具多命令。

## 注意事项

- `description` 是模型决定是否调用的关键——写清楚工具做什么、什么时候用
- 错误处理：`execute` 内抛出的异常会被 OpenCode 捕获并展示给用户
- 工具拦截（`tool.execute.before` / `tool.execute.after`）的 `input`/`output` 参数见 [hooks.md](hooks.md)
