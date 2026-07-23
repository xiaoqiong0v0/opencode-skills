# 日志

## 推荐方案：文件日志

推荐使用 `@xiaoqiong0v0/opencode-plugin-logger` 进行日志记录，模块顶层和钩子内部均可使用，API 统一且简单：

```ts
import createLogger from "@xiaoqiong0v0/opencode-plugin-logger"

const log = createLogger("my-plugin")

// 模块顶层可用
try { loadCfg() } catch (e) { log.error("初始化失败", e) }

export const MyPlugin = async () => {
  log.loaded()
  return {
    event: async ({ event }) => {
      log.info(`收到事件: ${event.type}`)
    },
  }
}
```

在 `~/.config/opencode/plugin-logger.jsonc` 中启用并配置：

```jsonc
{
  "enabled": true,
  "dir": "~/.opencode/plugins-log"
}
```

## 备选方案：OpenCode 服务端日志

如果希望日志出现在 OpenCode 的日志系统中（而非本地文件），可用 `client.app.log()`：

### 类型签名

```ts
client.app.log(options: {
  body: {
    service: string          // 插件名称或服务标识
    level: "debug" | "info" | "warn" | "error"
    message: string          // 日志消息
    extra?: Record<string, unknown>  // 附加数据
  }
})
```

## 基本用法

```ts
export const MyPlugin = async ({ client }) => {
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

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `body.service` | `string` | 是 | 插件名称或服务标识 |
| `body.level` | `string` | 是 | 日志级别 |
| `body.message` | `string` | 是 | 日志消息 |
| `body.extra` | `object` | 否 | 附加数据（任意 JSON 结构） |

## 注意事项

- 文件日志 `log.info()` 是同步操作，不会阻塞模块加载
- `client.app.log()` 是异步操作，需要 `await`
- 避免在频繁触发的事件处理器中大量写日志（如 `message.part.updated`）
