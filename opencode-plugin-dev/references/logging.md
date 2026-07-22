# 结构化日志

推荐使用 `client.app.log()` 代替 `console.log` 进行日志记录，日志会显示在 OpenCode 的日志系统中。

## 基本用法

```ts
export const MyPlugin = async ({ client }) => {
  // 初始化时记录
  await client.app.log({
    body: {
      service: "my-plugin",
      level: "info",
      message: "插件初始化完成",
      extra: { version: "1.0.0" },
    },
  })

  return {
    // ... hooks
  }
}
```

## 日志级别

| 级别 | 说明 | 使用场景 |
|------|------|---------|
| `debug` | 调试信息 | 开发时跟踪流程 |
| `info` | 常规信息 | 插件启动、正常操作 |
| `warn` | 警告 | 潜在问题、性能降级 |
| `error` | 错误 | 操作失败、异常情况 |

## 参数说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `service` | `string` | 插件名称或服务标识 |
| `level` | `string` | 日志级别 |
| `message` | `string` | 日志消息 |
| `extra` | `object?` | 附加数据（任意 JSON 结构） |

## 注意事项

- `console.log` 的输出也会显示在日志中，但缺少结构化的 `service` 和 `level` 字段
- 日志记录是异步操作，不会阻塞插件主流程
- 避免在频繁触发的事件处理器中大量写日志（如 `message.part.updated`）
