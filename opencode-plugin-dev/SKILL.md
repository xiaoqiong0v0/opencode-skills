# OpenCode 插件开发

## 快速开始

插件是 OpenCode 的扩展模块，可以注册自定义工具、监听事件、修改行为。

```js
// ~/.config/opencode/plugins/my-plugin.js
import { tool } from "@opencode-ai/plugin"

export const MyPlugin = async ({ project, client, $, directory, worktree }) => {
  return {
    tool: {
      hello: tool({
        description: "说 hello",
        args: { name: tool.schema.string().optional() },
        execute: async (args, context) => `Hello ${args.name || "world"}`
      })
    }
  }
}
```

## 目录结构

| 路径 | 用途 |
|------|------|
| `~/.config/opencode/plugins/*.js` | 本地插件（自动加载） |
| `.opencode/plugins/*.js` | 项目级插件 |
| `opencode.json` 的 `plugin` 数组 | npm 插件引用 |

详细文档：https://opencode.ai/docs/zh-cn/plugins/ (中文) / https://opencode.ai/docs/plugins/ (English)

## 深入阅读

| 文件 | 内容 |
|------|------|
| [local-plugin.md](local-plugin.md) | 本地插件开发（依赖、文件结构） |
| [npm-plugin.md](npm-plugin.md) | npm 插件发布（包结构、依赖处理） |
| [tool-definition.md](tool-definition.md) | 自定义工具 API（参数、context） |
| [hooks.md](hooks.md) | 事件钩子与工具拦截 |
