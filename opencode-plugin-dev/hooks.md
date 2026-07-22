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
        const { sessionID, parentID } = event.data
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
        const { sessionID } = event.data
        if (!parentMap.has(sessionID) && !childMap.has(sessionID)) {
          parentMap.set(sessionID, new Set())
        }
      }

      if (event.type === "session.deleted") {
        const { sessionID } = event.data
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
