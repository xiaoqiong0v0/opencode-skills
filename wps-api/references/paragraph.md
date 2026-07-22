# Paragraph 对象 & Paragraphs 集合

## Paragraphs 集合

### 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `Paragraphs.Count` | int | 段落总数（只读） |
| `Paragraphs.First` | Paragraph | 第一个段落 |
| `Paragraphs.Last` | Paragraph | 最后一个段落 |

### 方法

| 方法 | 说明 |
|------|------|
| `Paragraphs.Item(Index)` | 获取指定段落，**索引从 1 开始** |
| `Paragraphs.Add()` | 添加新段落并返回 |
| `Paragraphs.Add(Range)` | 在指定 Range 前添加段落 |

### 遍历段落

```python
total = doc.Paragraphs.Count
for i in range(1, total + 1):     # 索引从 1 开始！
    p = doc.Paragraphs(i)
    text = p.Range.Text.strip()
    if not text:                   # 跳过空段落
        continue
    # 处理 p ...

# 或者
for i in range(1, total + 1):
    p = doc.Paragraphs.Item(i)
```

## Paragraph 对象

### 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `p.Range` | Range | 段落对应的 Range 对象 |
| `p.Range.Text` | str | 段落文本内容（含末尾 `\r`） |
| `p.Style` | Style | 段落样式对象 |
| `p.Style.NameLocal` | str | 样式名称（本地语言），如 `"正文"`、`"表格内文字标题样式"` |
| `p.Alignment` | int | 对齐方式枚举值 |
| `p.LeftIndent` | float | 左缩进量（磅） |
| `p.RightIndent` | float | 右缩进量（磅） |
| `p.FirstLineIndent` | float | 首行缩进量（磅，正数=缩进，负数=悬挂） |
| `p.SpaceBefore` | float | 段前间距（磅） |
| `p.SpaceAfter` | float | 段后间距（磅） |
| `p.LineSpacing` | float | 行距值（具体含义取决于 LineSpacingRule） |
| `p.LineSpacingRule` | int | 行距规则枚举值 |
| `p.OutlineLevel` | int | 大纲级别（1-9=标题级别，10=正文） |
| `p.KeepTogether` | int | 段中不分页 |
| `p.KeepWithNext` | int | 与下段同页 |
| `p.WidowControl` | int | 孤行控制 |
| `p.PageBreakBefore` | int | 段前分页 |
| `p.Hyphenation` | int | 自动断字 |
| `p.Range.Information(Type)` | — | 获取段落位置信息 |

### 方法

| 方法 | 说明 |
|------|------|
| `p.Reset()` | 删除手动段落格式，恢复为样式默认值 |
| `p.Indent()` | 增加一个制表位的缩进 |
| `p.Outdent()` | 减少一个制表位的缩进 |
| `p.Space1()` | 设为单倍行距 |
| `p.Space15()` | 设为 1.5 倍行距 |
| `p.Space2()` | 设为 2 倍行距 |
| `p.Select()` | 选中该段落 |
| `p.Next()` | 返回下一个段落 |
| `p.Previous()` | 返回上一个段落 |

### 判断段落类型

```python
text = p.Range.Text.strip()

# 表格内段落
is_table_cell = '表格' in (p.Style.NameLocal or '')

# 一级标题（中文数字开头）
is_level1 = bool(text) and text[0] in '一二三四五六七八九十' and '、' in text[:3]

# 二级标题（数字序号 + 加粗）
is_level2 = p.Range.Font.Bold and text and text[0].isdigit()

# 标题（第一段大字号）
is_title = idx == 1 and p.Range.Font.Size >= 20.0
```

### Paragraph.Range.Information 常用参数

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdWithinTable | 13 | 段落是否在表格内 |
| wdFirstCharacterLineNumber | 4 | 首字符行号 |
| wdActiveEndPageNumber | 3 | 所在页码 |
| wdHorizontalPositionRelativeToPage | 5 | 相对页面的水平位置 |
| wdVerticalPositionRelativeToPage | 6 | 相对页面的垂直位置 |
