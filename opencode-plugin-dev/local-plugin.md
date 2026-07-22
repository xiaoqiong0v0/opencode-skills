# 本地插件

## 位置

- 全局：`~/.config/opencode/plugins/*.js`
- 项目：`.opencode/plugins/*.js`

自动加载，无需注册。

## 依赖

在 `~/.config/opencode/package.json` 中添加：

```json
{
  "dependencies": {
    "shescape": "^2.1.0"
  }
}
```

OpenCode 启动时自动 `bun install`。

## 基本结构

```js
export const PluginName = async ({ project, client, $, directory, worktree }) => {
  return {
    // 钩子或工具
  }
}
```

参数 `{ project, client, $, directory, worktree }` 全部可选，按需解构。

参考文档：https://opencode.ai/docs/zh-cn/plugins/
