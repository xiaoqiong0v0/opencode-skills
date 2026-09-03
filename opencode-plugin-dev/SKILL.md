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

### 技能发现路径

除了本地技能目录，还可以在 `opencode.json` 中配置额外路径：

```json
{
  "skills": {
    "paths": ["~/community-skills", "/shared/team-skills"],
    "urls": ["https://example.com/skills/my-skill"]
  }
}
```

### 加载顺序

所有来源的插件按以下顺序加载，同名钩子全部执行：

1. **全局配置文件** `~/.config/opencode/opencode.json` 中的 `plugin` 数组
2. **项目配置文件** `opencode.json` 中的 `plugin` 数组
3. **全局插件目录** `~/.config/opencode/plugins/`
4. **项目插件目录** `.opencode/plugins/`

同名 npm 包只加载一次；本地插件和同名的 npm 插件各自独立加载。插件可通过 V2 API 在运行时动态注册/卸载技能和命令（见 [v2-plugin.md](references/v2-plugin.md)）。

## 深入阅读

根据具体需求选择子文档：

| 场景 | 参考文件 |
|------|---------|
| 首次创建本地插件、依赖管理 | [local-plugin.md](local-plugin.md) |
| 发布到 npm、包结构、安装方式 | [npm-plugin.md](npm-plugin.md) |
| 工具定义 API、参数、context | [tool-definition.md](tool-definition.md) |
| 事件钩子、工具拦截、会话管理 | [hooks.md](hooks.md) |
| V2 Plugin API（运行时注册技能/命令/参考） | [v2-plugin.md](references/v2-plugin.md) |
| 实战示例（通知、.env保护、注入变量、压缩） | [examples.md](references/examples.md) |
| 结构化日志（client.app.log） | [logging.md](references/logging.md) |

## 开发原则

- **不知道怎么实现时，去看源码**：OpenCode 是开源项目（[github.com/anomalyco/opencode](https://github.com/anomalyco/opencode)），type 定义在 `node_modules/@opencode-ai/plugin/dist/` 和 `@opencode-ai/sdk/dist/` 中。不确定 API 签名、事件字段、context 类型时，直接读对应版本的 `.d.ts` 文件，不要凭记忆或猜测写代码。
- **了解进程/实例模型**：插件状态分三层缓存维度——**进程级**（模块顶层变量，Bun 模块缓存保证同进程只 import 一次）、**目录级**（插件函数体 + hooks 闭包，`InstanceState` 按工作目录缓存，同目录多会话共享、切目录产生新闭包）、**事件级**（hook 回调每次触发执行）。server 和 TUI 是两个独立进程，各自执行一遍顶层代码。顶层只放无副作用定义，有副作用的初始化放函数内（见 [local-plugin.md](local-plugin.md) 的"模块加载与进程模型"章节）。
- 插件代码中使用 `console.log` 调试，输出在 TUI 或日志中可见。

## 重要原则

1. **加载顺序**：全局配置 → 项目配置 → 全局插件 → 项目插件。同名工具时**插件优先**于内置工具。
2. **TypeScript**：推荐使用 TypeScript，从 `@opencode-ai/plugin` 导入类型获得类型安全：
   ```ts
   import type { Plugin } from "@opencode-ai/plugin"

   export const MyPlugin: Plugin = async ({ project, client, $, directory, worktree }) => {
     return {
       // 类型安全的钩子实现
     }
   }
   ```
   文件后缀使用 `.js` 或 `.ts` 均可，OpenCode 自动处理。
3. **依赖管理**：本地插件如需外部包，在配置目录下创建 `package.json`，OpenCode 启动时自动 `bun install`。
4. **调试技巧**：插件中的 `console.log` 输出可以在 OpenCode 的 TUI 或日志中查看。修改插件后重启 OpenCode 或重载插件目录即可生效。
5. **config 钩子**：V1 插件的 `config` 钩子可修改运行时的配置对象，常用于动态注册命令（`config.command`），见 [v2-plugin.md](references/v2-plugin.md) 的 CommandHooks 章节。

详细文档：https://opencode.ai/docs/plugins/ (English)
