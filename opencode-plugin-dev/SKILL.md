---
name: opencode-plugin-dev
description: OpenCode 插件开发：创建自定义工具、事件钩子、本地插件和 npm 插件发布。当用户提到开发/创建/修改/修复 OpenCode 插件、自定义工具（tool）、注册或使用事件钩子（hook）、拦截工具行为（tool.execute.before/after）、将插件发布到 npm、本地插件文件的放置位置（~/.config/opencode/plugins/ 或 .opencode/plugins/）、插件安装方式、opencode.json 中 plugin 数组配置、Plugin 类型导入、tool() 辅助函数、Zod schema 参数校验、ToolContext 使用、插件依赖管理时，必须使用此技能。也包括用户想扩展 OpenCode 功能、添加自定义命令或工具、监听 OpenCode 事件（会话/消息/文件变更/LSP/权限）、修改 LLM 参数或 shell 环境等场景。如果用户问的是"如何使用"现有插件而不是"开发"插件，则不需要触发此技能。
---

# OpenCode 插件开发

## 工作流程

开发一个插件通常按以下顺序进行：

1. **确定类型**：本地插件（文件）还是 npm 插件（包）
2. **创建文件**：在对应目录下创建 `.js`/`.ts` 文件
3. **实现功能**：注册工具或钩子
4. **测试验证**：重启或重载 OpenCode 测试
5. **（可选）发布**：发布到 npm

## 快速示例

```ts
// ~/.config/opencode/plugins/my-plugin.js
import { tool } from "@opencode-ai/plugin"

export const MyPlugin = async ({ project, client, $, directory, worktree }) => {
  return {
    tool: {
      hello: tool({
        description: "向用户打招呼",
        args: {
          name: tool.schema.string().optional().describe("用户名"),
        },
        async execute(args, context) {
          const { directory } = context
          return `Hello ${args.name || "world"} from ${directory}`
        },
      }),
    },
  }
}
```

## 目录结构

| 路径 | 用途 | 优先级 |
|------|------|--------|
| `~/.config/opencode/plugins/*.js` | 全局本地插件（自动加载） | 高 |
| `.opencode/plugins/*.js` | 项目级本地插件（自动加载） | 低 |
| `opencode.json` 的 `plugin` 数组 | npm 插件引用 | — |

本地插件自动加载，无需注册。npm 插件在 `opencode.json` 中添加包名后自动安装。

## 深入阅读

根据具体需求选择子文档：

| 场景 | 参考文件 |
|------|---------|
| 首次创建本地插件、依赖管理 | [local-plugin.md](local-plugin.md) |
| 发布到 npm、包结构、安装方式 | [npm-plugin.md](npm-plugin.md) |
| 工具定义 API、参数、context | [tool-definition.md](tool-definition.md) |
| 事件钩子、工具拦截、会话管理 | [hooks.md](hooks.md) |

## 重要原则

1. **加载顺序**：全局配置 → 项目配置 → 全局插件 → 项目插件。同名工具时**插件优先**于内置工具。
2. **TypeScript**：推荐使用 TypeScript，导入 `type Plugin` 获得类型安全。文件后缀使用 `.js`（OpenCode 自动处理）。
3. **依赖管理**：本地插件如需外部包，在配置目录下创建 `package.json`，OpenCode 启动时自动 `bun install`。
4. **调试技巧**：插件中的 `console.log` 输出可以在 OpenCode 的 TUI 或日志中查看。修改插件后重启 OpenCode 或重载插件目录即可生效。

详细文档：https://opencode.ai/docs/plugins/ (English)
