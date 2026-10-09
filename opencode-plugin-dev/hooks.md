# 事件钩子参考

## 通用事件处理器

使用 `event` 处理器接收所有事件，根据 `event.type` 分发：

```ts
export const MyPlugin = async () => {
  return {
    event: async ({ event }) => {
      switch (event.type) {
        case "session.created":
          // 新会话创建（含子 agent 会话）
          break
        case "session.updated":
          // 主会话重入时触发（不会重复触发 created）
          break
        case "session.deleted":
          // 会话删除——清理资源
          break
        case "session.idle":
          // 会话空闲——可做后台操作
          break
        case "message.updated":
          // 消息更新
          break
      }
    },
  }
}
```

## 工具拦截

在工具执行前后插入逻辑：

```ts
export const MyPlugin = async () => {
  return {
    // 工具执行前——可修改入参或阻止执行
    "tool.execute.before": async (input, output) => {
      if (input.tool === "read") {
        // 拦截 read 工具
      }
    },

    // 工具执行后——可修改结果
    "tool.execute.after": async (input, output) => {
      output.result = `拦截修改: ${output.result}`
    },
  }
}
```

## config 钩子：能力边界

`config` 钩子在 OpenCode 完成配置加载、但插件正式初始化之前调用，可以**原地修改运行时配置对象**。

**机制（源码依据，opencode `1.18.25`，并以 `v1.18.34` 交叉核对）：**

- `bootstrap` 的顺序是**先 `config.get()` 再 `plugin.init()`**——源码注释原话是「插件会改 config，必须最先初始化」。
- `hook.config(cfg)` 拿到的**就是同一个对象**（`config.ts:620` 返回 `s.config`，**不拷贝**），因此对它的原地修改会直接影响后续读取。

**判据：某字段的读取点是否在 `config` 钩子之后。** 读取点在钩子之后的字段才能被动态改；在钩子之前就已消费的字段改不动。

**能动态改 ✅**

| 字段 | 说明 |
|------|------|
| `command` | 动态注册命令（对照组，最常用） |
| `provider.*.models.*.cost` | 模型计价 |
| `provider.*.models.*.limit` | 上下文/输出上限（⚠️ 前提：该 provider **必须已存在**于 `cfg.provider`） |
| `model` / `small_model` | 默认模型 / 小模型 |
| `agent.*` | model / prompt / permission / temperature |
| `permission` / `instructions` / `mcp` / `lsp` / `formatter` | 权限、指令、MCP、LSP、格式化器 |

**不能动态改 ✗**

| 字段 | 原因 | 替代手段 |
|------|------|---------|
| `plugin` | 钩子执行前已按 `plugin_origins` 加载完外部插件 | 落配置 + 重启 |
| `agent.*.tools` | 已废弃；在**配置载入期**就折进 `permission` | 改用 `agent.*.permission` |

**修改请求参数 / 头**不走 `config`，用 `chat.params` / `chat.headers`。

**示例：按运行时条件覆盖某个模型字段**

```ts
export const MyPlugin = async () => ({
  config: async (config) => {
    // 前提：该 provider 必须已存在于 config.provider，才能改它的模型字段
    const model = config.provider?.myprovider?.models?.["my-model"]
    if (model && process.env.MY_PLUGIN_OVERRIDE === "1") {
      model.limit = { context: 200_000, output: 32_000 }
    }
  },
})
```

> **版本与可信度**：以上结论出自 opencode `1.18.25` 源码，并以 `v1.18.34` tag 交叉核对；**未做实验验证**。使用前建议在目标版本上自行验证。

## 各 hook 的 payload 速查

hook 的 `input` 携带内容各不相同——尤其**要从 hook 里取模型信息**时，不要假设每个 hook 都能拿到完整 `Model`（例如 `limit`）。

| 钩子 | `input` 关键字段 | 能否拿到模型 limit |
|------|-----------------|-------------------|
| `chat.params` | 含 `model: Model` | ✅ `model.limit.context` / `model.limit.output` |
| `chat.headers` | 含 `model` | ✅ |
| `chat.message` | `{ sessionID, agent, model: { providerID, modelID } }`（实测） | ✗（`model` 无 limit） |
| `experimental.chat.messages.transform` | `input` **恒为 `{}`**，消息在 `output.messages` | ✗ |
| `experimental.chat.system.transform` | 含 `model` | ✅ |
| `experimental.compaction.autocontinue` | 含 `model` | ✅ |

> 版本同「config 钩子」节：opencode `1.18.25`（`v1.18.34` 交叉核对），**未实验验证**。

## hook 运行时实测事实

以下为在 opencode `v1.18.33` 上**实测确认**的行为（非文档推断）：

- `experimental.chat.messages.transform`：`input` 恒为 `{}`；消息在 `output.messages`，其中 `messages[i].info` 带 `sessionID` / `agent`；修改要**原地 push**，不要替换 `output.messages` 引用；`info.synthetic === true` 的消息**不落库**。
- `experimental.session.compacting`：`input.sessionID` **有值**。
- `chat.message`：`input` 带 `sessionID`。
- 消息级去重可用消息 id（`info.id`）；压缩产生的摘要消息带 `info.summary === true`，可作为「这是压缩产物」的标识。
- 原生**自动压缩在回合循环内**按容量触发 ⇒ 触发那一刻**没有模型回合**，无法在压缩前做插件侧处理。

## 用户输入：行首半角 `!` = shell 模式（不可禁用）

opencode TUI 把**半角 `!` 键**绑定为 shell 模式：光标在**首字符（offset 0）**时按 `!` → 进入 shell 模式，**回车直接 spawn 执行**，且**不走权限确认**（源码 `packages/tui/src/component/prompt/index.tsx:829-841`；执行走 `packages/opencode/src/session/prompt.ts` 的 `shellImpl`；记成 tool `bash`）。

- 该绑定**不可通过 `keybinds` 配置禁用**（配置项里没有）。
- 后果：任何依赖「用户消息以 `!` 开头」的插件/功能都**收不到**这类输入（`!` 被 shell 吃掉，消息到不了 hook）。
- 规避：换其它前缀字符；或行首改用**全角 `！`**（全角不触发该绑定，实测可正常进入模型与 `chat.message`）。
- 实测版本：opencode `1.18.x`。

## 完整事件列表

### 会话事件
| 事件 | 触发时机 |
|------|---------|
| `session.created` | 新会话创建 |
| `session.deleted` | 会话删除 |
| `session.updated` | 会话更新 |
| `session.idle` | 会话空闲 |
| `session.error` | 会话错误 |
| `session.compacted` | 会话压缩完成 |
| `session.diff` | 会话差异 |
| `session.status` | 会话状态变更 |

### 消息事件
| 事件 | 触发时机 |
|------|---------|
| `message.updated` | 消息更新 |
| `message.removed` | 消息删除 |
| `message.part.updated` | 消息部分内容更新 |
| `message.part.removed` | 消息部分内容删除 |

### 工具事件
| 事件 | 触发时机 |
|------|---------|
| `tool.execute.before` | 工具执行前（可修改入参） |
| `tool.execute.after` | 工具执行后（可修改结果） |

### 文件 & LSP
| 事件 | 触发时机 |
|------|---------|
| `file.edited` | 文件被编辑 |
| `file.watcher.updated` | 文件监听变更 |
| `lsp.client.diagnostics` | LSP 诊断结果 |
| `lsp.updated` | LSP 状态更新 |

### 权限 & 命令
| 事件 | 触发时机 |
|------|---------|
| `permission.asked` | 权限询问时 |
| `permission.replied` | 用户回复权限时 |
| `command.executed` | TUI 命令执行后 |

### 消息钩子（V1 插件）
| 钩子 | 触发时机 | input/output |
|------|---------|-------------|
| `chat.message` | 新消息到达时 | 可修改 `output.message` 和 `output.parts` |
| `chat.params` | 修改 LLM 参数 | 可设置 `temperature`、`topP`、`topK`、`maxOutputTokens` |
| `chat.headers` | 修改 LLM 请求头 | 可注入自定义 `headers` |

### 工具定义钩子
| 钩子 | 触发时机 | 说明 |
|------|---------|------|
| `tool.definition` | 修改工具定义发送给 LLM 前 | 可修改 `description` 和 `parameters` |

### 实验性钩子
| 钩子 | 触发时机 | 说明 |
|------|---------|------|
| `experimental.chat.messages.transform` | 修改发送给 LLM 的消息列表 | 可增删改消息 |
| `experimental.chat.system.transform` | 修改系统提示词 | 可追加 system prompt |
| `experimental.compaction.autocontinue` | 压缩后是否自动继续 | 设置 `output.enabled = false` 跳过自动继续 |
| `experimental.provider.small_model` | 为 provider 选择小模型 | 返回轻量模型用于简单任务 |
| `experimental.text.complete` | 文本补全 | 注入自定义补全文本 |

### 其他
| 事件 | 触发时机 |
|------|---------|
| `server.connected` | 服务器连接 |
| `installation.updated` | 安装更新 |
| `shell.env` | 修改 shell 环境变量 |
| `todo.updated` | 待办事项更新 |
| `tui.prompt.append` | TUI 提示追加 |
| `tui.command.execute` | TUI 命令执行 |
| `tui.toast.show` | TUI Toast 通知 |

### 实验性事件
| 事件 | 触发时机 |
|------|---------|
| `experimental.session.compacting` | 会话压缩中，用于注入自定义上下文或替换压缩提示词 |

## experimental.session.compacting

在 LLM 生成续接摘要之前触发。可以向 `output.context` 注入领域特定上下文（默认压缩可能遗漏的信息），或通过设置 `output.prompt` 完全替换压缩提示词。

```ts
export const MyPlugin = async () => {
  return {
    "experimental.session.compacting": async (input, output) => {
      // 注入额外上下文
      output.context.push(`
## 自定义上下文

压缩时应保留的状态：
- 当前任务状态
- 重要决策记录
`)

      // 可选：完全替换压缩提示词
      // output.prompt = "你的自定义提示词..."
    },
  }
}
```

实战示例见 [examples.md](references/examples.md)。

## session.created 属性

| 字段 | 类型 | 说明 |
|------|------|------|
| `sessionID` | `string` | 当前会话 ID |
| `parentID` | `string?` | 父会话 ID（子 agent 时存在，主子会话为 `null`） |
| `agent` | `string` | 代理类型（`build`/`general`/`pro`/`explore`） |
| `title` | `string` | 会话标题 |
| `slug` | `string` | 会话别名 |
| `directory` | `string` | 工作目录 |

## message.updated 的 parentID

`message.updated` 事件中的 `info.parentID` 是**消息回复的父消息 ID**（即这条消息回复的是哪条消息），与会话的 `parentID`（父子会话关系）含义不同，注意区分。

## 会话管理

子 agent 会创建独立会话（不同 `sessionID`），通过 `parentID` 关联。父子会话间不共享缓存/数据。插件如需跨会话跟踪数据，需通过 `parentID` 链回退查找。

**正确做法：用 Map 记录父子关系，而非栈。**

```ts
// parentID → Set<childSessionID>，支持多子会话
const parentMap = new Map<string, Set<string>>()
// sessionID → parentID，快速向上查找
const childMap = new Map<string, string>()

export const MyPlugin = async () => {
  return {
    event: async ({ event }) => {
      if (event.type === "session.created") {
        const { sessionID, parentID } = event.properties
        if (parentID) {
          // 子会话：记录父子关系
          childMap.set(sessionID, parentID)
          if (!parentMap.has(parentID)) {
            parentMap.set(parentID, new Set())
          }
          parentMap.get(parentID)!.add(sessionID)
        } else {
          // 主子会话：确保有映射占位
          parentMap.set(sessionID, new Set())
        }
      }

      if (event.type === "session.updated") {
        // updated 频繁触发（消息更新、token 变化、摘要刷新等），
        // 仅在 Map 中不存在该 sessionID 时注册（首次见到的会话）
        const { sessionID } = event.properties
        if (!parentMap.has(sessionID) && !childMap.has(sessionID)) {
          parentMap.set(sessionID, new Set())
        }
      }

      if (event.type === "session.deleted") {
        const { sessionID } = event.properties
        // 清理子会话记录
        if (childMap.has(sessionID)) {
          const pid = childMap.get(sessionID)!
          parentMap.get(pid)?.delete(sessionID)
          childMap.delete(sessionID)
        }
        parentMap.delete(sessionID)
      }
    },
  }
}
```

## 会话树与数据共享

子 agent 创建独立会话，形成会话树结构：

```
主会话 (sessionID: A)
├── 子 agent 会话 (sessionID: B, parentID: A)
│   └── 孙 agent 会话 (sessionID: C, parentID: B)
└── 子 agent 会话 (sessionID: D, parentID: A)
```

**数据共享规则：**

- 父子会话间**不共享**缓存、变量、上下文
- 插件如需访问父会话数据，需通过 `parentID` 链向上回退查找
- 获取会话祖先链：
  ```ts
  function getAncestors(sessionID: string): string[] {
    const chain = [sessionID]
    let current = sessionID
    while (childMap.has(current)) {
      current = childMap.get(current)!
      chain.unshift(current)
    }
    return chain  // [root, ..., sessionID]
  }
  ```

## 注意事项

- 事件处理器中不要做耗时操作（如网络请求），否则影响 OpenCode 性能
- `tool.execute.before` 中修改 `input` 会影响实际执行；不调用则不影响
- `tool.execute.after` 中修改 `output.result` 会改变返回给用户的内容
