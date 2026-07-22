# npm 插件

## 包结构

npm 插件是一个标准 npm 包，入口文件导出一个插件函数：

```json
{
  "name": "@scope/plugin-name",
  "type": "module",
  "files": ["plugin.js", "README.md"],
  "exports": {
    ".": "./plugin.js"
  }
}
```

## 入口导出

```ts
export const PluginName = async () => {
  return {
    /* tools or hooks */
  }
}
```

## 安装方式

用户在 `opencode.json` 的 `plugin` 数组中添加包名：

```json
{
  "plugin": ["@scope/plugin-name"]
}
```

OpenCode 启动时自动用 Bun 安装到 `~/.cache/opencode/node_modules/`。

## 依赖处理

npm 插件的 `dependencies` 在 `package.json` 中声明，由 OpenCode 自动安装，用户无需手动操作。

## 发布

```bash
# scoped 包必须加 --access public
npm publish --access public

# 更新后需要 bump version 再发布
npm version patch
npm publish --access public
```

## 注意事项

- 插件包必须是 ESM（`"type": "module"`）
- 入口文件导出必须使用 `export const` 命名导出（不是 `export default`）
- 确保 `files` 字段只包含需要发布的文件
- 建议在发布前本地测试：先在 `opencode.json` 中用 `"plugin": ["file:./local-path"]` 测试
