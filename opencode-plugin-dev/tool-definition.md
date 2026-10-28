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

`execute` 的返回值**必须是字符串**。宿主（Bun/JSC）把返回内容按字符串处理，返回普通对象会直接报错：`undefined is not an object (evaluating 'c.split')`。

```ts
// ✅ 返回字符串
return "处理完成"

// ✅ 结构化结果：自行 JSON.stringify 或格式化成文本
return JSON.stringify({ rowCount: 42, rows }, null, 2)

// ❌ 返回普通对象 → 报 undefined is not an object (evaluating 'c.split')
return { title: "查询结果", output: "表格数据..." }
```

例外：带附件的**特殊结果对象**可以返回（如截图工具返回的 image attachment），宿主会识别并按附件处理。

标题 / 附加元数据不要塞进返回对象，用 `context.metadata({ title, metadata })` 设置。

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

### help 的记忆特性

模型执行一次 `help` 后，输出**保留在会话上下文中**，后续调用可直接使用，**不需要每次先查用法**。这是单工具多命令模式省 token 的关键。

**但 help 记忆会失效：**

| 场景 | 原因 |
|------|------|
| 会话被压缩（compaction） | help 细节可能被摘要掉 |
| 子 agent | 子会话独立，父会话的 help 不共享 |
| 新会话 | 上下文清空 |
| 长时间间隔 | help 被挤出上下文窗口 |

**因此：**
- `description` 里也要写**精简的子命令列表**，让模型在第一次调用前就有初步概念，减少"先 help 再操作"的往返：
  ```
  description: "统一命令行工具。子命令: list / add --name / remove / help。用 'help' 看完整用法。"
  ```
- help 输出要**精炼但完整**——它是模型长期记忆的"手册"
- 若插件关键用法被压缩后容易遗忘，可在 `experimental.session.compacting` 钩子中把用法注入压缩摘要（见 [hooks.md](hooks.md)）

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

## 工具描述（description）规范

`description` 是模型**唯一的**调用依据：模型只看描述决定调不调用、怎么传参。描述里混进实现细节，模型就会把「现象」当「线索」——转而去找文件、翻源码，而不是修正参数重试。

### 只写 5 类

| # | 内容 | 例子 |
|---|------|------|
| ① | 做什么（一句话） | `查询指定城市的当前天气` |
| ② | 何时用 / 何时别用 | `已知城市名时直接用；不确定时先用 city_search` |
| ③ | 参数语义（类型 + 取值范围 + 必填） | `unit: "c" / "f"，可选，默认 "c"` |
| ④ | 返回什么 | `返回 Markdown 表格，每行一个城市` |
| ⑤ | 关键副作用 / 约束（写库、删文件、幂等、是否需要凭据） | `会写入配置目录；幂等，重复调用不产生重复项；需已登录凭据` |

### 禁止写

- 实现机制（怎么排队、用什么锁、内部几个阶段）
- 内部钩子 / API 名（如 `session.idle`、`session.summarize`）
- 文件路径、函数名、源码位置（`src/foo.ts`、`handleCommand()`）
- 「见某文件 / 某 doc」这类指向
- 调试建议、维护者备注（TODO、坑位备注）

⇒ 以上内容归 `docs/` 与代码注释，不进描述字符串。

### 反考古条款

描述里确实要交代失败语义时，写成**可执行的下一步**，而不是指路：

- ✅ `失败时按 error 处理，修正参数后重试`
- ✅ `参数非法时返回 help 文本，照其修正后重新调用`
- ❌ `失败时可读 src/xxx.ts 了解原因`
- ❌ `详见 docs/xxx.md`

理由：错误信息会诱导模型去翻仓库，把「调用出错」变成「考古」，白白消耗上下文且通常找不到答案。**公开仓尤其注意**——描述会随包分发出去，读者拿不到你的源码上下文。

### 自检（可 grep）

描述字符串（`description` 与 `.describe()`）中**不得**出现 `.ts` / `.js` 源码路径、`src/`、`session.`、`实现`、`源码`、`见 `：

```powershell
Get-ChildItem plugins -Recurse -Include *.ts,*.js | Select-String 'description:\s*"[^"]*(\.ts|\.js|src/|session\.|实现|源码|见 )'
```

命中即需重写该条描述。

### 正反例

✅ 正例（只有契约）：

```ts
description: "查询指定城市的当前天气。已知城市名时直接用，不确定时先用 city_search。返回 Markdown 表格。只读，不写任何文件。"
```

❌ 反例（把实现机制、内部钩子名、源码位置写进契约）：

```ts
description:
  "天气查询。插件只排队标记并在 session.idle 时运行 session.summarize() 汇总，" +
  "实现在 src/weather.ts 的 handleCommand()，失败时可去读源码了解原因。"
```

问题：模型读不到「输入什么、返回什么、有什么副作用」，却拿到了钩子名和文件路径；一旦出错只会去翻仓库。正例提供了同样的调用能力，且不含任何实现线索。

### 长度与 help 分工

| 位置 | 硬上限 | 要求 |
|------|--------|------|
| 单命令工具 `description` | **≤ 100 字符**（中文按字计） | 一句话说清「做什么 + 何时用」 |
| 多命令 CLI 风格工具：主描述 | **≤ 60 字符** | 只写「是个什么 CLI + 用 help 看用法」 |
| 多命令 CLI 风格工具：每个子命令描述 | **≤ 60 字符** | 一个子命令一句话 |
| `schema.describe()` 每个参数 | **一行 ≤ 40 字符** | 类型 + 取值 + 默认值，不写示例故事 |

**超出即拆分**：详细用法、枚举取值、示例、边界与限制 → `help` 子命令（CLI 风格工具）或 `docs/`，**不进 `description`**。参数细节写进参数自己的 `.describe()`，不要塞进工具描述——模型按参数取用时能直接对上，塞进描述反而和参数定义脱节。

**为什么限长**：描述越长，模型越容易漏掉关键约束、把示例当规则、在报错时按描述里的细节「自由发挥」。索引卡写得再长也不会变成说明书，只会稀释重点。

**为什么细节该放 help**：`help` 是**按需展开**的——模型只在需要时才读，输出再保留在上下文中复用（见上文「help 的记忆特性」）。常驻的 `description` 每次调用都要付 token，按需的 `help` 只在真需要时付。

✅ 精简（一句话 + 何时用，约 40 字）：

```ts
description: "查询指定城市天气。已知城市名直接用，否则先 city_search。只读。"
```

❌ 超标（200+ 字符、多段堆细节——换个温度单位都要读一遍）：

```ts
description:
  "天气查询工具。本工具支持指定城市查询当前天气，返回 Markdown 表格，每行一个城市。" +
  "支持 unit 参数取 c 或 f，默认 c；支持一次查询多个城市，用逗号分隔，最多 5 个。" +
  "如果不确定城市名，请先调用 city_search 得到标准城市名，再调用本工具查询天气。" +
  "失败时返回错误码：404 表示城市不存在，应改用 city_search 重新确认；" +
  "500 表示上游超时，应稍后重试；429 表示被限流，应减少查询城市数量后再试。"
```

问题：真正的契约只有「做什么 / 何时用 / 只读」，其余全是本该由 `help` 承接的用法说明；模型每次调用都要重读一遍，反而容易在报错时按描述里的错误码清单自由发挥。

**自检（按字符数统计）**：

```powershell
Get-ChildItem plugins -Recurse -Include *.ts,*.js |
  Select-String 'description:\s*"([^"]*)"' |
  ForEach-Object { [pscustomobject]@{ File = $_.Filename; Line = $_.LineNumber
                                      Len  = $_.Matches[0].Groups[1].Value.Length } } |
  Where-Object { $_.Len -gt 60 } | Format-Table -AutoSize
```

> 命中 > 60 即需人工确认：CLI 风格工具直接判超标；单命令工具再看是否 > 100。多行字符串拼接的描述脚本逐行计数会漏算，需把拼接段合起来数。

一句话：**`description` = 索引卡；`help` / `docs` = 说明书。**

## 注意事项

- `description` 是模型决定是否调用的关键——写清楚工具做什么、什么时候用、有什么副作用（写法见上节，尤其别漏写副作用与「何时别用」）
- 错误处理：`execute` 内抛出的异常会被 OpenCode 捕获并展示给用户
- 工具拦截（`tool.execute.before` / `tool.execute.after`）的 `input`/`output` 参数见 [hooks.md](hooks.md)
