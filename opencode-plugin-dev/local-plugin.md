# 本地插件

## 放置位置

- **全局**：`~/.config/opencode/plugins/*.js`（所有项目可用）
- **项目级**：`.opencode/plugins/*.js`（仅当前项目）

自动加载，无需在 `opencode.json` 中注册。

## 依赖管理

如果插件需要外部包，在插件所在目录下创建 `package.json`：

```json
{
  "dependencies": {
    "shescape": "^2.1.0"
  }
}
```

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

## 文件组织

简单插件的所有逻辑写在一个文件里。复杂插件可以拆分为多个文件，入口文件导出插件：

```
~/.config/opencode/plugins/
├── package.json          # 依赖声明（可选）
├── my-plugin.js          # 入口，export const MyPlugin = ...
└── helpers/              # 辅助模块（可选）
    └── utils.js
```

## 注意

- 修改插件文件后**重启 OpenCode** 或重新加载配置即可生效
- 文件后缀可以是 `.js` 或 `.ts`（OpenCode 自动处理 TypeScript）
- 插件中的 `console.log`/`console.error` 输出会显示在 OpenCode 日志中
