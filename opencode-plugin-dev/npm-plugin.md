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

发布前本地测试已发布的 npm 插件，无需反复 `npm version patch && npm publish`。核心逻辑：

1. `npm run build` 构建本地代码
2. 把 `dist/` + `package.json` 覆盖到 OpenCode 包缓存（**两个路径都要覆盖**，可能从任意一个加载）
3. **同步依赖**：插件 `package.json` 里声明的 `dependencies`，若外层 node_modules 缺失或版本不同，从本地拷贝

```powershell
# .tmp/publish-local.ps1
$ErrorActionPreference = "Stop"

# 按插件 package.json 的 dependencies，只拷贝本地有、外层缺失或版本不同的依赖
function Sync-Dependencies {
  param([string]$PkgJson, [string]$LocalNode, [string]$OuterNode)
  $deps = (Get-Content $PkgJson -Raw | ConvertFrom-Json).dependencies
  if (-not $deps) { return }
  foreach ($name in $deps.PSObject.Properties.Name) {
    $localPkg = Join-Path $LocalNode $name
    if (-not (Test-Path (Join-Path $localPkg "package.json"))) { continue }

    $localVer = (Get-Content (Join-Path $localPkg "package.json") -Raw | ConvertFrom-Json).version
    $outerPkg = Join-Path $OuterNode $name
    $need = $true
    if (Test-Path (Join-Path $outerPkg "package.json")) {
      $outerVer = (Get-Content (Join-Path $outerPkg "package.json") -Raw | ConvertFrom-Json).version
      $need = ($localVer -ne $outerVer)   # 版本不同才同步
    }
    if ($need) {
      $destParent = Split-Path $outerPkg -Parent
      New-Item -ItemType Directory -Force -Path $destParent | Out-Null
      Copy-Item -Recurse -Force $localPkg $outerPkg
    }
  }
}

Write-Host "Building..." -ForegroundColor Cyan
npm run build

$localNode = Join-Path (Get-Location) "node_modules"

# 覆盖 opencode 已安装的插件包缓存（按实际参考的包名替换）
$pkg = "@scope/plugin-name"
$scopes = @(
  "$env:USERPROFILE\.cache\opencode\packages\$pkg@latest\node_modules\$pkg",
  "$env:USERPROFILE\.cache\opencode\packages\$pkg\node_modules\$pkg"
)

$copied = $false
foreach ($dest in $scopes) {
  if (Test-Path $dest) {
    Copy-Item -Recurse -Force "dist" "$dest\"
    Copy-Item -Force "package.json" "$dest\package.json"

    # 外层 node_modules（依赖应放这里，保证插件 import 可解析）
    # dest 结构: <scopeRoot>\node_modules\<scope>\pkg，上溯 3 级 = scopeRoot
    $scopeRoot = Split-Path (Split-Path (Split-Path $dest -Parent) -Parent) -Parent
    $outer = Join-Path $scopeRoot "node_modules"
    New-Item -ItemType Directory -Force -Path $outer | Out-Null
    Sync-Dependencies -PkgJson (Join-Path $dest "package.json") -LocalNode $localNode -OuterNode $outer

    Write-Host "覆盖完成: $dest (dist + package.json + deps)" -ForegroundColor Green
    $copied = $true
  }
}

if (-not $copied) {
  Write-Host "未找到 opencode 插件安装目录，请先确认插件已在 opencode.json 引用并启动过一次。" -ForegroundColor Yellow
}

# 3. 重启 OpenCode 使插件生效
```

**为什么依赖要同步到外层 node_modules？**
OpenCode 安装 npm 插件时，依赖放在包缓存的**外层** `node_modules`（而非插件目录内），插件通过 `file://` URL import 时依赖从外层解析。本地覆盖只复制 `dist` 不改依赖，会导致插件 `import` 第三方包失败；若本地用了新版本依赖而缓存里是旧版，行为也会不一致。

## 注意事项

- 插件包必须是 ESM（`"type": "module"`）
- 入口文件需要同时提供 `export const` 命名导出和 `export { Xxx as default }` 默认导出（两者缺一不可）
- 确保 `files` 字段只包含需要发布的文件
- 建议在发布前本地测试：先在 `opencode.json` 中用 `"plugin": ["file:./local-path"]` 测试
- `@opencode-ai/plugin` 必须放在 `peerDependencies`，详见"依赖处理"章节
