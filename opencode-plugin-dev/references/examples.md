# 实战示例

## 发送通知

事件触发时发送系统通知：

```js
// ~/.config/opencode/plugins/notification.js
export const NotificationPlugin = async ({ $ }) => {
  return {
    event: async ({ event }) => {
      if (event.type === "session.idle") {
        // macOS 系统通知
        await $`osascript -e 'display notification "会话完成！" with title "opencode"'`
      }
      if (event.type === "session.error") {
        await $`osascript -e 'display notification "会话出错！" with title "opencode" sound name "Basso"'`
      }
    },
  }
}
```

**注意：** OpenCode 桌面版自带系统通知功能，此示例适用于 CLI 模式。

## .env 文件保护

阻止 OpenCode 读取 `.env` 文件：

```js
// ~/.config/opencode/plugins/env-protection.js
export const EnvProtection = async () => {
  return {
    "tool.execute.before": async (input, output) => {
      if (input.tool === "read" && output.args.filePath?.includes(".env")) {
        throw new Error("禁止读取 .env 文件")
      }
    },
  }
}
```

## 注入环境变量

将自定义环境变量注入所有 Shell 执行（AI 工具和用户终端）：

```js
// ~/.config/opencode/plugins/inject-env.js
export const InjectEnvPlugin = async () => {
  return {
    "shell.env": async (input, output) => {
      output.env.MY_API_KEY = "your-key-here"
      output.env.PROJECT_ROOT = input.cwd
    },
  }
}
```

## 会话压缩钩子

在 LLM 生成续接摘要前注入自定义上下文：

```ts
// ~/.config/opencode/plugins/compaction.ts
import type { Plugin } from "@opencode-ai/plugin"

export const CompactionPlugin: Plugin = async (ctx) => {
  return {
    "experimental.session.compacting": async (input, output) => {
      // 注入压缩时保留的额外上下文
      output.context.push(`
## 自定义上下文

压缩时应保留的状态：
- 当前任务状态
- 已做的重要决策
- 正在编辑的文件
`)
    },
  }
}
```

### 完全替换压缩提示词

```ts
export const CustomCompactionPlugin: Plugin = async (ctx) => {
  return {
    "experimental.session.compacting": async (input, output) => {
      // 替换整个压缩提示词
      output.prompt = `
你正在生成多 agent 集群会话的续接摘要。

请总结：
1. 当前任务及其状态
2. 哪些文件正在被修改，由谁修改
3. 任何阻塞项或 agent 间依赖
4. 完成工作的下一步

格式化为结构化提示词，供新 agent 恢复工作时使用。
`
    },
  }
}
```

## 耗时操作监控

记录工具执行耗时：

```js
export const PerfMonitor = async ({ client }) => {
  return {
    "tool.execute.before": async (input) => {
      input._startTime = Date.now()
    },
    "tool.execute.after": async (input, output) => {
      const elapsed = Date.now() - (input._startTime || Date.now())
      if (elapsed > 5000) {
        await client.app.log({
          body: {
            service: "perf-monitor",
            level: "warn",
            message: `Slow tool: ${input.tool} took ${elapsed}ms`,
          },
        })
      }
    },
  }
}
```
