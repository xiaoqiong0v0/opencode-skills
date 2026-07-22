# npm 插件

## 包结构

```json
{
  "name": "@scope/plugin-name",
  "type": "module",
  "files": ["plugin.js", "README.md"],
  "exports": { ".": "./plugin.js" }
}
```

## 入口导出

```js
export const PluginName = async () => {
  return { /* hooks or tools */ }
}
```

## 安装方式

用户在 `opencode.json` 的 `plugin` 数组添加包名后，OpenCode 自动用 Bun 安装到 `~/.cache/opencode/node_modules/`。

## 依赖处理

npm 插件的 `dependencies` 由 OpenCode 自动安装，无需用户手动操作。

## 发布

```bash
npm publish --access public  # scoped 包必须加 --access public
```

参考文档：https://opencode.ai/docs/plugins/
