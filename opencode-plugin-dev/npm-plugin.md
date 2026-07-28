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

// npm 插件还需要默认导出
export { PluginName as default }
```

**npm 插件需要同时提供命名导出和默认导出**。命名导出供 `.opencode/plugins/` 本地加载使用，默认导出供 npm 加载器使用（`import()` 后读取 `mod.default`）。

用户在 `opencode.json` 的 `plugin` 数组中添加包名：

```json
{
  "plugin": ["@scope/plugin-name"]
}
```

OpenCode 启动时自动用 Bun 安装到 `~/.cache/opencode/node_modules/`。

## 依赖处理

npm 插件的 `dependencies` 在 `package.json` 中声明，由 OpenCode 自动安装（`bun install --production`，**不安装 devDependencies**）。

**关键：`@opencode-ai/plugin` 必须放在 `peerDependencies`**，它是 OpenCode 宿主提供的 SDK，不应作为外部依赖安装：

```json
{
  "peerDependencies": {
    "@opencode-ai/plugin": "*"
  },
  "dependencies": {
    "my-other-dep": "^1.0.0"
  }
}
```

运行时 `import { tool } from "@opencode-ai/plugin"` 会从 OpenCode 宿主环境解析。如果把 `@opencode-ai/plugin` 放在 `dependencies` 或 `devDependencies`，**bun 不会安装它**，插件加载会静默失败。

## 发布

```bash
# scoped 包必须加 --access public
npm publish --access public

# 更新后需要 bump version 再发布
npm version patch
npm publish --access public
```

## 本地测试与热覆盖

发布前本地测试已发布的 npm 插件，无需反复 `npm version patch && npm publish`。核心思路：构建本地代码后覆盖 OpenCode 的包缓存。

```powershell
# 1. 本地构建
npm run build

# 2. 覆盖 OpenCode 包缓存（路径格式：~/.cache/opencode/packages/<包名>@latest/node_modules/<包名>/）
$cacheDir = "$env:USERPROFILE\.cache\opencode\packages\@scope\plugin-name@latest\node_modules\@scope\plugin-name"

if (Test-Path $cacheDir) {
    Copy-Item -Recurse -Force "dist" "$cacheDir\"
    Write-Host "覆盖完成: $cacheDir\dist" -ForegroundColor Green
}

# 3. 重启 OpenCode 使插件生效
```

**注意：** 缓存路径有两种可能（取决于版本）：
- `~/.cache/opencode/packages/@scope/plugin-name@latest/node_modules/@scope/plugin-name/`
- `~/.cache/opencode/packages/@scope/plugin-name/node_modules/@scope/plugin-name/`

如果路径不存在说明该插件还没被 OpenCode 安装过，需要先配置 `opencode.json` 并启动一次。

## 注意事项

- 插件包必须是 ESM（`"type": "module"`）
- 入口文件需要同时提供 `export const` 命名导出和 `export { Xxx as default }` 默认导出（两者缺一不可）
- 确保 `files` 字段只包含需要发布的文件
- 建议在发布前本地测试：先在 `opencode.json` 中用 `"plugin": ["file:./local-path"]` 测试
- `@opencode-ai/plugin` 必须放在 `peerDependencies`，详见"依赖处理"章节
