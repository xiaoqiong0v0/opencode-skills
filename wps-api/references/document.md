# Document 对象

## 打开/创建/保存/关闭

```python
# 打开已有文档
doc = wps.Documents.Open(r'C:\path\to\file.docx')

# 新建空白文档
doc = wps.Documents.Add()

# 另存为
doc.SaveAs(r'C:\path\to\output.docx')

# 保存（覆盖原文件）
doc.Save()

# 另存为 PDF
doc.ExportAsFixedFormat(r'C:\output.pdf', 17)  # 17 = wdExportFormatPDF

# 关闭
doc.Close(SaveChanges=False)   # False=不保存, True=保存
```

### Open 完整参数

```python
doc = wps.Documents.Open(
    FileName=r'C:\file.docx',
    ConfirmConversions=False,       # 不询问格式转换
    ReadOnly=False,                 # 只读
    AddToRecentFiles=False,         # 不加入最近文件列表
    Visible=False                   # 不可见（后台操作）
)
```

## Documents 集合

| 属性/方法 | 说明 |
|-----------|------|
| `Documents.Count` | 已打开文档数量 |
| `Documents.Add()` | 新建空白文档 |
| `Documents.Open(Path)` | 打开文档 |
| `Documents.Close()` | 关闭所有文档 |

## Document 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `doc.Name` | str | 文件名（只读），如 `"report.docx"` |
| `doc.FullName` | str | 完整路径（只读） |
| `doc.Path` | str | 所在目录（只读） |
| `doc.Paragraphs` | Paragraphs | 段落集合 |
| `doc.Paragraphs.Count` | int | 段落总数 |
| `doc.Tables` | Tables | 表格集合 |
| `doc.Tables.Count` | int | 表格总数 |
| `doc.Styles` | Styles | 样式集合 |
| `doc.Sections` | Sections | 节集合 |
| `doc.Sections.Count` | int | 节总数 |
| `doc.PageSetup` | PageSetup | 页面设置 |
| `doc.BuiltInDocumentProperties` | — | 内置属性（作者、标题等） |
| `doc.Content` | Range | 文档全部内容 Range |
| `doc.Range(Start, End)` | Range | 按字符偏移获取 Range |
| `doc.Saved` | bool | 文档自上次保存后是否无修改 |
| `doc.Words.Count` | int | 字数统计（含标点） |
| `doc.Characters.Count` | int | 字符总数 |
| `doc.ReadOnly` | bool | 是否只读 |

## 文档方法

| 方法 | 说明 |
|------|------|
| `doc.Activate()` | 激活该文档窗口 |
| `doc.Close(SaveChanges)` | 关闭文档 |
| `doc.Save()` | 保存 |
| `doc.SaveAs(FileName)` | 另存为 |
| `doc.SaveAs2(FileName)` | 另存为（新版接口） |
| `doc.PrintOut()` | 打印 |
| `doc.ExportAsFixedFormat(Path, Format)` | 导出为固定格式（PDF/XPS） |
| `doc.Select()` | 全选文档内容 |
| `doc.Undo(N)` | 撤销 N 次操作 |
| `doc.Redo(N)` | 重做 N 次操作 |
| `doc.Repaginate()` | 重新分页 |

## PageSetup（页面设置）

```python
ps = doc.PageSetup
ps.TopMargin = 72          # 上边距（磅），1英寸=72磅
ps.BottomMargin = 72       # 下边距
ps.LeftMargin = 90         # 左边距
ps.RightMargin = 90        # 右边距
ps.PageWidth = 595.3       # A4 宽度（磅）
ps.PageHeight = 841.9      # A4 高度（磅）
ps.Orientation = 0         # 0=纵向(wdOrientPortrait), 1=横向(wdOrientLandscape)
```

## ExportAsFixedFormat 格式常量

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdExportFormatPDF | 17 | PDF |
| wdExportFormatXPS | 18 | XPS |
