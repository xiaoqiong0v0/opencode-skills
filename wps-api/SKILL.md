---
name: wps-api
description: WPS Office COM automation API reference — control WPS Writer (Word), Spreadsheets (Excel), Presentation (PowerPoint) via Python win32com, VBA, VB.NET, or JavaScript. Covers Application, Document, Range, Paragraph, Font, ParagraphFormat, Table, Find/Replace, PageSetup, and all enumeration constants. Use this skill whenever the user asks about WPS automation, WPS COM interfaces, formatting Word documents programmatically with WPS, pywin32 + WPS, or WPS VBA macro development.
---

# WPS COM Automation API

通过 COM 自动化控制 WPS Office，兼容 Python（`win32com`）、VBA、VB.NET、JavaScript 等语言。WPS COM 对象模型与 Microsoft Word VBA **高度兼容**，大部分 Word VBA 代码可直接在 WPS 中运行。

## 快速开始

```python
import win32com.client
wps = win32com.client.Dispatch('Kwps.Application')  # 或 'WPS.Application'
wps.Visible = False
doc = wps.Documents.Open(r'C:\path\to\file.docx')
# ... 操作文档 ...
doc.SaveAs(r'C:\output.docx')
doc.Close()
wps.Quit()
```

## 参考文件索引

根据需求读取对应文件：

| 文件 | 内容 | 何时读取 |
|------|------|----------|
| [application.md](references/application.md) | Application 对象 — 连接、版本、ProgID、退出 | 需要连接/断开 WPS 时 |
| [document.md](references/document.md) | Document 对象 — 打开、保存、关闭、属性、PageSetup | 操作文档级别时 |
| [range.md](references/range.md) | Range 对象 — 文本读写、选区、插入/删除 | 操作文档内容时 |
| [paragraph.md](references/paragraph.md) | Paragraphs / Paragraph — 遍历段落、获取文本 | 处理段落时 |
| [font.md](references/font.md) | Font 对象 — 字体名、字号、加粗、颜色、字号对照表 | 设置字体格式时 |
| [paragraph-format.md](references/paragraph-format.md) | ParagraphFormat 对象 — 对齐、行距、缩进、孤行控制 | 设置段落格式时 |
| [enums.md](references/enums.md) | 枚举常量 — 对齐、行距规则、下划线、颜色、大纲级别 | 需要用数值代替常量名时 |
| [find-replace.md](references/find-replace.md) | Find / Replacement — 文本查找替换 | 批量替换文本时 |
| [table.md](references/table.md) | Table / Tables — 表格遍历、单元格读写 | 处理表格时 |
| [examples.md](references/examples.md) | 实战示例 — 完整格式化脚本模板 | 需要参考完整代码时 |

## 核心对象层次

```
Application
├── Documents → Document
│   ├── Paragraphs → Paragraph → Range
│   │   ├── Font (Name, Size, Bold, Color...)
│   │   └── ParagraphFormat (Alignment, LineSpacing, Indent...)
│   ├── Tables → Table → Cell
│   ├── Styles → Style
│   └── Sections → Section → PageSetup
└── Selection → Range (当前光标位置)
```

## 重要提示

- **段落索引从 1 开始**（`Paragraphs(1)` 是第一段），不是 0
- **晚期绑定无类型库** — 枚举常量直接用数值，见 [enums.md](references/enums.md)
- **字体名用中文** — `'仿宋_GB2312'`、`'黑体'`、`'楷体_GB2312'`、`'方正小标宋简体'`
- **Bold 判断** — WPS COM 中 `Font.Bold` 可能返回 `0`/`-1`/`True`/`False`，用 `if f.Bold:` 而非 `== True`
- **COM 释放** — 务必 `doc.Close()` + `wps.Quit()`，避免进程残留
- **表格段落** — 格式化时通常跳过表格内段落（`Style.NameLocal` 含"表格"）
