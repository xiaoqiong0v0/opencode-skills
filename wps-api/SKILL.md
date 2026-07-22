---
name: wps-api
description: WPS Office COM 自动化 API 参考——通过 Python win32com、VBA、VB.NET、JavaScript 控制 WPS 文字、表格、演示。当用户提到 WPS 办公自动化、WPS 批量处理文档/排版/格式调整、Python 操作 WPS、win32com + WPS、WPS COM 开发、VBA 宏迁移到 Python、公文格式（标题/正文/字体/行距/页边距）、WPS 文字段落遍历、表格单元格读写、查找替换、WPS 导出 PDF、WPS 页面设置、公文自动排版、政府公文/企业报告/标书格式处理、字体设置仿宋/黑体/楷体/方正小标宋简体时，**必须立即使用此技能**。用户可能不会直接说"COM 自动化"，而是说"帮我把这些文档排下版""批量处理文件夹里的公文"或类似需求——这种情况下也应该触发并使用本技能提供正确的 WPS COM 操作方式。不含通用 Office 文档问题排查（模板/样式损坏等不属于 COM 编程的问题）。
---

# WPS COM 自动化

通过 COM 控制 WPS Office，兼容 Python（`win32com`）、VBA、VB.NET、JavaScript。WPS COM 对象模型与 Microsoft Word VBA **高度兼容**。

## 通用模式

所有 WPS COM 脚本都必须遵循以下模式：

```python
import win32com.client

wps = win32com.client.Dispatch('Kwps.Application')
wps.Visible = False     # 后台运行
wps.DisplayAlerts = 0   # 关闭弹窗
try:
    doc = wps.Documents.Open(r'input.docx')
    # ... 业务操作 ...
    doc.SaveAs(r'output.docx')
finally:
    doc.Close()
    wps.Quit()           # 必须调用，否则 WPS 进程残留
```

**错误的 Bold 判断：** WPS COM 中 `Font.Bold` 返回 `0`/`-1`/`True`/`False` 都可能出现，**始终用 `if f.Bold:` 而非 `== True`**。

## 核心对象层次

```
Application
├── Documents → Document
│   ├── Paragraphs → Paragraph → Range
│   │   ├── Font (Name, Size, Bold, Color...)
│   │   └── ParagraphFormat (Alignment, LineSpacing, Indent...)
│   ├── Tables → Table → Cell → Range
│   ├── Sections → Section → PageSetup
│   ├── Styles → Style
│   └── Content → Range (全文范围)
└── Selection → Range (当前光标选区)
```

## 参考文件

| 文件 | 内容 | 何时读取 |
|------|------|----------|
| [application.md](references/application.md) | Application 对象 — 连接、ProgID、性能优化、32/64位兼容 | 连接 WPS 或遇到连接失败时 |
| [document.md](references/document.md) | Document 对象 — 打开/保存/关闭/导出PDF/PageSetup | 操作文档级别时 |
| [sections.md](references/sections.md) | Section / Sections — 节管理、页面设置、页眉页脚 | 分节、页面方向/页边距差异时 |
| [range.md](references/range.md) | Range 对象 — 文本读写、选区、插入/删除、Move/Collapse | 操作文档内容时 |
| [paragraph.md](references/paragraph.md) | Paragraphs / Paragraph — 遍历段落、获取文本 | 处理段落时 |
| [font.md](references/font.md) | Font 对象 — 字体名、字号、加粗、颜色、字号对照表 | 设置字体格式时 |
| [paragraph-format.md](references/paragraph-format.md) | ParagraphFormat — 对齐、行距、缩进、孤行控制 | 设置段落格式时 |
| [find-replace.md](references/find-replace.md) | Find/Replacement — 文本查找替换 | 批量替换文本时 |
| [table.md](references/table.md) | Table/Tables — 表格遍历、单元格读写 | 处理表格时 |
| [enums.md](references/enums.md) | 枚举常量数值表 — 对齐、行距、颜色、下划线、大纲级别 | 需要枚举常量数值时 |
| [examples.md](references/examples.md) | 实战示例 — 完整格式化脚本、公文模板 | 需要参考完整代码时 |

## 常见问题

### 32位 vs 64位 COM 不匹配
WPS 个人版是 32 位，如果 Python 是 64 位会导致 `Dispatch` 失败。解决方案：
- 用 32 位 Python 运行脚本（推荐）
- 或在 64 位 Python 中使用 64 位 WPS（WPS 专业版可选）

### WPS 进程残留
脚本异常退出后 WPS 进程仍驻留，导致下次连接失败。症状：`Documents.Open` 报错或卡死。解决方案：
- 任务管理器杀掉 `wps.exe` 和 `et.exe`/`wpp.exe`
- 务必在代码中使用 `try/finally` + `doc.Close()` + `wps.Quit()`

### ProgID 连接失败
- 首选 `Kwps.Application`（WPS 12.x+）
- 备选 `WPS.Application`（旧版兼容）
- 表格用 `Et.Application`，演示用 `Wpp.Application`
- 安装 WPS 专业版/政务版时 ProgID 可能不同

### 枚举常量
晚期绑定（`Dispatch`）无类型库，无法使用 `win32com.constants.wd*`，所有枚举用数值代替（见 [enums.md](references/enums.md)）。
