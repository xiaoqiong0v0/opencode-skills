# opencode-skills

个人维护的 OpenCode **公共技能仓**：收录通用、可分享给他人的技能（无个人 / 本机专属上下文）。

## 收录标准

- 与个人开发环境无关，他人可直接安装使用
- 正文不出现个人路径、账号、私有约定

## 目录结构

每个技能一个目录，含 `SKILL.md`（可带 `references/`、`evals/`）。

## 现有技能

| 技能 | 说明 |
| --- | --- |
| `opencode-plugin-dev` | OpenCode 插件开发：创建自定义工具、事件钩子、本地插件和 npm 插件发布（含 tool()、Zod 参数校验、ToolContext 等） |
| `wps-api` | WPS Office COM 自动化 API 参考——通过 Python win32com、VBA、VB.NET、JavaScript 控制 WPS 文字、表格、演示 |

## 维护约定

- 仓库内统一 LF（见 `.gitattributes`）
- 个人开发环境专属技能不放本仓（放在本机 opencode 配置仓库，`xqv-` 前缀）
