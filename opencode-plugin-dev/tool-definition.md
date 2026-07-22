# 自定义工具

## 定义

```js
import { tool } from "@opencode-ai/plugin"

tool: {
  mytool: tool({
    description: "工具描述（模型决策用）",
    args: {
      foo: tool.schema.string().optional().describe("参数说明"),
    },
    async execute(args, context) {
      // args: 用户传入的参数
      // context.sessionID: 当前会话 ID
      // context.directory: 工作目录
      return "结果字符串"
    },
  }),
}
```

## ToolContext

| 字段 | 说明 |
|------|------|
| `sessionID` | 当前会话 ID |
| `messageID` | 触发消息 ID |
| `agent` | 代理类型（default/pro/multi） |
| `directory` | 项目工作目录 |
| `worktree` | Git 工作树路径 |
| `abort` | AbortSignal，支持取消 |
| `metadata({ title })` | 设置工具结果标题 |

## 覆盖规则

- 插件工具与内置工具同名时，**插件工具优先**
- 多插件注册同名工具时，加载顺序决定优先级

参考文档：https://opencode.ai/docs/zh-cn/plugins/
