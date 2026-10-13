# TUI 插件

> ⚠️ 这套 API **官方文档未记载**，跨版本可能变。落地前按**目标版本**复核 `@opencode-ai/plugin` 的 `dist/tui.d.ts` 与 opencode 源码。
>
> ⚠️ **易错点**：`tui` 必须是**函数** `(api) => ...`，不是 `{ id, setup }` 对象（那是 V2 server 插件的风格）。opencode 源码 `plugin/shared.ts` 的 `readV1Plugin()` 对 `tui` 做 `typeof !== "function"` 校验，已装类型 `TuiPluginModule`（`tui.d.ts`）也只接受函数。（**已实测**：对象形式会被加载器拒收、槽位不渲染；函数形式正常渲染。）

## 速览

**能做**：给 TUI 加**常驻 UI**（槽位）、**热键**、**路由**、读会话、**写入输入框**、**toast/通知**。

**不能做**：改 web UI / `opencode serve`（固定 OpenAPI，无插件 UI 钩子）。

TUI 插件与 **server 插件**是两套完全不同的子系统：

| 维度 | server 插件 | TUI 插件 |
|------|-------------|----------|
| 配置文件 | `opencode.json` 的 `plugin` 数组 / 插件目录自动发现 | 独立的 **`tui.json`** |
| 导出形态 | `export const X = async () => Hooks` | `export default { id, tui }`，`tui` 是函数 |
| 运行位置 | server 进程（无 UI） | TUI 进程，**SolidJS** 渲染 |
| 生效时机 | 启动加载 | 启动加载，**改完需重启 opencode** |

## 最小可运行骨架

```tsx
/** @jsxImportSource @opentui/solid */
import type { TuiPlugin } from "@opencode-ai/plugin/tui"

const tui: TuiPlugin = async (api) => {
  // 1) 渲染进槽位（示例：会话输入框右侧）
  api.slots.register({
    order: 100,
    slots: {
      session_prompt_right: () => <text>🎙</text>,
    },
  })

  // 2) 注册热键（老的 api.command.* 已 deprecated）
  api.keymap.registerLayer({
    priority: 0,
    commands: [
      {
        name: "my.toggle",
        title: "Toggle",
        desc: "示例命令",
        run: () => api.ui.toast({ message: "hello" }),
      },
    ],
    bindings: [{ key: "ctrl+shift+v", cmd: "my.toggle" }],
  })
}

// default 导出必须是对象：file 来源必须有 id
export default {
  id: "my-tui-plugin",
  tui,
}
```

### 激活（项目级 `.opencode/tui.json`）

```json
{ "plugin": ["./.opencode/tui/my-plugin.tsx"] }
```

> 路径**相对声明它的配置文件**解析；重启 opencode 生效。

## 装载（取证）

- TUI 插件有**自己的配置文件与加载流程**，与 server 插件（`opencode.json` 的 `plugin`）不是一回事（`packages/opencode/src/config/tui.ts`）。配置文件名为 **`tui.json`**（或 `tui.jsonc`），两处都会加载：**项目根目录**（沿目录树向上查找）与 **`.opencode/` 目录**；此外全局配置目录与 `OPENCODE_TUI_CONFIG` 覆盖也会生效，就近优先。
- **file 来源**模块的 default 导出须是**对象**，且**必须有 `id`**：`packages/opencode/src/plugin/shared.ts` 的 `readV1Plugin()` 与 `resolvePluginId()`。
- **npm 包**方式：需包内 `exports["./tui"]` 指向入口：`packages/opencode/src/plugin/shared.ts` 的 `resolvePackageEntrypoint()`。
- **相对路径解析**：相对声明插件的配置文件所在目录：`packages/opencode/src/config/plugin.ts` 的 `resolvePluginSpec()`。
- `OPENCODE_PURE` 为真时**忽略外部插件**：`packages/opencode/src/plugin/tui/runtime.ts`。
- ⚠️ `.opencode/plugin/*`（含 `plugins/`）**不会**被 TUI 自动发现——那套只服务 server 插件；TUI 插件当前靠上面的 `tui.json`。

## 渲染与槽位（SolidJS）

整套渲染是 **SolidJS**（`SolidPlugin<TuiSlotMap, TuiSlotContext>`），组件里用 `@opentui/solid` 的 JSX（文件顶部加 `/** @jsxImportSource @opentui/solid */`）。

注册接口：

```ts
api.slots.register({ order, slots })
```

- `order`：同槽位多个插件的排序权重。
- `slots`：`{ [槽位名]: () => JSX }`。

**可用槽位（`TuiHostSlotMap`，`@opencode-ai/plugin` 的 `dist/tui.d.ts`）**：

| 槽位 | 说明 |
|------|------|
| `app` | 应用根层，覆盖整个界面 |
| `app_bottom` | 应用底部 |
| `home_logo` / `home_prompt` / `home_prompt_right` / `home_bottom` / `home_footer` | 首页各位置 |
| `session_prompt` / `session_prompt_right` | 会话输入框（左/右） |
| `sidebar_title` / `sidebar_content` / `sidebar_footer` | 会话侧边栏标题/内容/页脚 |

宿主渲染点（源码位置）：`packages/tui/src/app.tsx`（`app` / `app_bottom`）、`packages/tui/src/routes/home.tsx`（首页各槽位）、`packages/tui/src/routes/session/sidebar.tsx`（`sidebar_title` / `sidebar_content` / `sidebar_footer`）、`packages/tui/src/routes/session/index.tsx`（`session_prompt_right`）。

真实参考实现：`packages/tui/src/feature-plugins/system/which-key.tsx`（`api.slots.register` + `api.keymap.registerLayer` 的完整用法）。

整页路由另可 `api.route.register([{ name, render }])`。

## 热键

```ts
api.keymap.registerLayer({
  priority: 0,
  commands: [{ name, title, desc, run }],
  bindings: [{ key: "ctrl+shift+v", cmd: name }],
})
```

- `api.keymap.registerLayer(...)`：注册一层 keymap（命令 + 按键绑定）。
- `Layer` / `Binding` 形状见 `@opentui/keymap`。
- 老 `api.command.*`（`register` / `trigger` / `show`）**已 deprecated**，类型里仅作兼容保留，勿在新插件使用。参考实现见 `packages/tui/src/feature-plugins/system/which-key.tsx`。

## 路由与会话

- 注册路由：`api.route.register([{ name, render }])`；跳转：`api.route.navigate(name, params)`。
- 当前路由（含会话）：`api.route.current`，形如 `{ name: "session", params: { sessionID } }`。
- 会话数据：`api.session.get(id)` / `messages(id)` / `status(id)` / `permission(id)` / `question(id)`。
- **切换会话**：`api.route.navigate("session", { sessionID })`，或发事件 `tui.session.select { sessionID }`。

## 写入输入框与提示

- 写入输入框：`api.prompt.set(TuiPromptInfo)`；或发事件 `tui.prompt.append { text }`（插入并把光标移到行尾）。
- toast：`api.ui.toast({ title?, message, variant?, duration? })`。
- 系统级提醒（attention）：`api.ui.notify(...)`。

## TUI 事件

由 `POST /tui/*` 发布（`packages/opencode/src/server/routes/instance/httpapi/groups/tui.ts` 的 `root = "/tui"`），按 workspace 过滤。事件定义见 `packages/schema/src/tui-event.ts`。

| 事件 | 参数 | 效果 |
|------|------|------|
| `tui.prompt.append` | `{ text }` | 往输入框插入文本并跳行尾 |
| `tui.command.execute` | `{ command }` | 派发命令（`keymap.dispatchCommand`）；内置白名单如 `session.new`/`session.list`/`session.interrupt`/`session.compact`/`prompt.clear`/`prompt.submit`/`agent.cycle` 等，也接受任意字符串 |
| `tui.toast.show` | `{ title?, message, variant, duration }` | 弹 toast（`duration` 默认 5000ms，`variant` 为 `info`/`success`/`warning`/`error`） |
| `tui.session.select` | `{ sessionID }` | 切到该会话（内部走 `route.navigate`） |

## 不可扩展边界

- **web UI / `opencode serve`**：固定 OpenAPI，server 端插件 Hooks **没有 HTTP 路由 / UI 钩子**，无法扩展 web 界面。

## 验证方法

**手动（唯一能确认渲染的方式）**：

1. 写 `.opencode/tui.json`（`{"plugin":["<path>"]}`）→ **重启 opencode** → 看槽位是否出现、热键是否生效。
2. 失败看启动日志里的 `[tui.plugin]` 前缀（`packages/opencode/src/plugin/tui/runtime.ts` 的 `fail` / `warn`）。

**离线编译验证**（只能证明 TSX 能被宿主链路编译，**不能**证明渲染）：

- 宿主用 `@opentui/solid` + Babel（solid universal + preset-typescript + module-resolver）编译，并把 import 重写成运行时模块 id。
- 本机**未装 `@opentui/*`** 时 `tsc` 会报 `@opentui/solid/jsx-runtime` 类型缺失——**只是类型解析失败，运行由宿主重写**，不影响运行。
- `node --check` **不适用于 `.tsx`**；要用等价的 Babel 链离线跑一遍才算验证。

## 本机 spike 实测结论

在一次本机 spike 中，**本地 file 插件 + `.opencode/tui.json` 路径**方案**成功**：占位组件渲染进了 `session_prompt_right`（用户目视确认）。

结论：**自定义 TUI 插件可加载、可渲染进槽位**；`export default { id, tui }`（`tui` 为函数）+ `tui.json` 路径是可用的最小装载路径。

> ⚠️ 该实测运行版本与本文取证的源码版本可能不同；跨版本落地前按目标版本的 `tui.d.ts` 与源码复核。
