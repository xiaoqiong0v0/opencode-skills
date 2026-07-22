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

| 字段 | 类型 | 说明 |
|------|------|------|
| `sessionID` | `string` | 当前会话 ID |
| `messageID` | `string` | 触发消息 ID |
| `callID` | `string` | 工具调用 ID |
| `agent` | `string` | 代理类型（`build`/`general`/`pro`/`explore`） |
| `directory` | `string` | 项目工作目录 |
| `worktree` | `string` | Git 工作树路径 |
| `abort` | `AbortSignal` | 取消信号，支持超时/手动取消 |
| `extra` | `object` | 模型信息（`model.id`、`model.providerID` 等） |
| `time` | `number` | 时间戳 |
| `metadata({ title })` | `function` | 设置工具结果标题（在 UI 中显示） |

**注意：** `execute` 的 context 中**没有** `parentID`。需要跟踪父子会话关系，请通过 `session.created` 事件监听（见 [hooks.md](hooks.md) 的会话管理章节）。

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

## 注意事项

- `description` 是模型决定是否调用的关键——写清楚工具做什么、什么时候用
- 函数的返回值必须是字符串，复杂结构需自行序列化（如 `JSON.stringify`）
- 错误处理：`execute` 内抛出的异常会被 OpenCode 捕获并展示给用户
- 工具拦截（`tool.execute.before` / `tool.execute.after`）的 `input`/`output` 参数见 [hooks.md](hooks.md)
