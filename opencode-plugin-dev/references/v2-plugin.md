# V2 Plugin API

OpenCode 1.18+ 引入了 V2 插件 API，相比 V1 提供了更强大的能力：运行时管理插件、动态注册技能和命令、管理引用文档等。

## 基本结构

```ts
import { define } from "@opencode-ai/plugin/v2/promise"
import type { Plugin } from "@opencode-ai/plugin/v2/promise"

export const MyPlugin: Plugin = {
  id: "my-plugin",
  setup: async (context) => {
    // 通过 context 注册/管理功能
  },
}

// 或使用 define 辅助函数
export default define({
  id: "my-plugin",
  setup: async (context) => {},
})
```

## PluginContext

`setup` 接收的 `context` 包含以下 hooks：

| 属性 | 类型 | 说明 |
|------|------|------|
| `options` | `PluginOptions` | 插件配置选项 |
| `agent` | `AgentHooks & Reload` | 管理 agent（list/get/update/remove/default） |
| `aisdk` | `AISDKHooks` | AI SDK 集成 |
| `catalog` | `CatalogHooks & Reload` | 管理 provider/model 目录 |
| `command` | `CommandHooks & Reload` | 管理自定义命令 |
| `integration` | `IntegrationHooks & Reload` | 管理外部集成 |
| `plugin` | `PluginDomain` | 动态添加/移除插件 |
| `reference` | `ReferenceHooks & Reload` | 管理引用文档 |
| `skill` | `SkillHooks & Reload` | 动态注册技能 |

## PluginDomain — 动态插件管理

```ts
export const MyPlugin: Plugin = {
  id: "my-plugin",
  setup: async ({ plugin }) => {
    // 添加一个插件（另一个 V2 插件对象）
    await plugin.add({
      id: "sub-plugin",
      setup: async (ctx) => { /* ... */ },
    })

    // 移除一个插件
    await plugin.remove("sub-plugin")
  },
}
```

## SkillHooks — 动态注册技能

插件可以在运行时向 OpenCode 注册技能，支持三种来源：

```ts
export const MyPlugin: Plugin = {
  id: "my-plugin",
  setup: async ({ skill }) => {
    // 监听技能 transform 事件
    await skill.transform(async (draft) => {
      // 1. 从目录添加
      draft.source({ type: "directory", path: "/path/to/skill" })

      // 2. 从 URL 添加
      draft.source({ type: "url", url: "https://example.com/skill" })

      // 3. 内嵌技能定义
      draft.source({
        type: "embedded",
        skill: {
          name: "my-embedded-skill",
          description: "内嵌技能示例",
          content: "技能内容（SKILL.md 正文）",
          location: "embedded",
        },
      })
    })

    // 列出已注册的技能来源
    const sources = skill.list()
  },
}
```

## CommandHooks — 动态注册命令

命令有多种注册方式，按需选择：

### 方式一：V1 config 钩子（推荐，简洁）

V1 插件的 `config` 钩子可直接修改 `config.command`：

```ts
export const MyPlugin = async () => {
  return {
    config: async (config) => {
      config.command = config.command ?? {}
      config.command["my-cmd"] = {
        template: "执行 {{input}} 并返回结果",
        description: "自定义命令示例",
        agent: "build",
      }
    },
  }
}
```

命令在 OpenCode TUI 中通过 `/my-cmd` 触发，`{{input}}` 为用户输入的参数。

### 方式二：V2 command.transform（可增删改查）

```ts
export const MyPlugin: Plugin = {
  id: "my-plugin",
  setup: async ({ command }) => {
    await command.transform(async (draft) => {
      // 列出已有命令
      const all = draft.list()

      // 获取特定命令
      const cmd = draft.get("my-cmd")

      // 修改已有命令
      if (cmd) {
        draft.update("my-cmd", (cmd) => {
          cmd.description = "更新后的描述"
        })
      }

      // V2 无法直接添加命令，需要结合 V1 config 钩子
      // 或通过 reload 重新加载配置

      // 移除命令
      draft.remove("obsolete-cmd")
    })
  },
}
```

### 命令完整结构

```ts
type CommandInfo = {
  name: string        // 命令名称（用于 / 触发）
  template: string    // 命令模板（支持 {{input}} 占位符）
  description?: string // 命令描述
  agent?: string      // 指定执行的 agent
  model?: string      // 指定模型
  variant?: string    // 模型 variant
  subtask?: boolean   // 是否以子任务执行
  hints?: string[]    // 触发提示词
}
```

命令的完整结构：

```ts
type CommandV2Info = {
  name: string        // 命令名称（用于 / 触发）
  template: string    // 命令模板（发送给模型的内容）
  description?: string // 命令描述
  agent?: string      // 指定 agent
  model?: ModelRef    // 指定模型
  subtask?: boolean   // 是否以子任务执行
}
```

## AgentHooks — 管理 Agent

```ts
export const MyPlugin: Plugin = {
  id: "my-plugin",
  setup: async ({ agent }) => {
    await agent.transform(async (draft) => {
      // 列出所有 agent
      const agents = draft.list()

      // 获取特定 agent
      const myAgent = draft.get("build")

      // 修改 agent 配置
      draft.update("build", (agent) => {
        agent.description = "自定义构建 agent"
        agent.steps = 20
      })

      // 设置默认 agent
      draft.default("build")

      // 移除 agent
      draft.remove("legacy-agent")
    })
  },
}
```

## ReferenceHooks — 管理引用文档

```ts
export const MyPlugin: Plugin = {
  id: "my-plugin",
  setup: async ({ reference }) => {
    await reference.transform(async (draft) => {
      // 添加本地引用
      draft.add("my-docs", {
        type: "local",
        path: "/path/to/docs",
      })

      // 添加 Git 仓库引用
      draft.add("framework-docs", {
        type: "git",
        repository: "https://github.com/user/repo.git",
        branch: "main",
      })

      // 移除引用
      draft.remove("obsolete-docs")

      // 列出引用
      const refs = draft.list()
    })
  },
}
```

## Reload

每个 `*Hooks & Reload` 类型都有一个 `reload()` 方法，用于在修改后立即重新加载：

```ts
await command.transform(async (draft) => {
  draft.remove("cmd")
})
await command.reload()
```

**注意：** `reload()` 在某些 OpenCode 版本中可能不生效，建议重启 OpenCode 验证命令是否注册成功。

## V1 vs V2 对比

| 能力 | V1 | V2 |
|------|----|----|
| 注册工具 (tool) | ✅ `tool: { ... }` | 通过 V1 兼容或自定义 |
| 事件钩子 (event) | ✅ `event: async ({ event })` | ❌ V2 无通用 event |
| 工具拦截 | ✅ `tool.execute.before/after` | ❌ 需结合 V1 |
| 动态注册插件 | ❌ | ✅ `plugin.add/remove` |
| 动态注册技能 | ❌ | ✅ `skill.transform` |
| 动态注册命令 | ❌ | ✅ `command.transform` |
| 管理 agent | ❌ | ✅ `agent.transform` |
| 管理引用 | ❌ | ✅ `reference.transform` |
| 结构 | 函数 `(input) => Hooks` | 对象 `{ id, setup }` |
| 额外钩子 | `chat.message/params/headers` | 通过 `aisdk` 等替代 |

## 混合使用

V1 和 V2 可以共存。一个插件可以同时导出 V1 函数和 V2 Plugin 对象，OpenCode 会同时加载。

## 参考

- V2 入口：`@opencode-ai/plugin/v2/promise`（Promise 风格）或 `@opencode-ai/plugin/v2/effect`（Effect 风格）
- SDK 类型：`@opencode-ai/sdk/v2/types`
