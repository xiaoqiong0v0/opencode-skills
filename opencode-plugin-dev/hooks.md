# 钩子参考

## 通用事件钩子

```js
event: async ({ event }) => {
  if (event.type === "session.created") { /* 新会话 */ }
  if (event.type === "session.deleted") { /* 会话删除 */ }
  if (event.type === "session.idle") { /* 会话空闲 */ }
  if (event.type === "message.part.updated") { /* 消息部分更新 */ }
}
```

常用事件：`session.created`、`session.deleted`、`session.idle`、`session.error`、`message.part.updated`、`message.updated`。

## 工具拦截

```js
"tool.execute.before": async (input, output) => {
  if (input.tool === "read") { /* 拦截 read 工具 */ }
},
"tool.execute.after": async (input, output) => {
  output.result = "modified result"
},
```

## 其他钩子

- `"chat.message"` — 新消息到达时
- `"chat.params"` — 修改 LLM 参数
- `"permission.ask"` — 权限询问时
- `"command.execute.before"` — TUI 命令执行前
- `"shell.env"` — 修改 shell 环境变量

## SessionStack 与会话管理

插件中多个钩子需要跟踪当前会话：

```js
const SessionStack = {
  _stack: ["default"],
  push(id) { this._stack.push(id) },
  remove(id) { /* ... */ },
  get current() { return this._stack[this._stack.length - 1] },
}
```

子会话通过 `session.created` 事件入栈，`session.deleted` 出栈。

参考文档：https://opencode.ai/docs/zh-cn/plugins/
