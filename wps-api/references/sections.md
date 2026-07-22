# Section / Sections 对象

一个文档有 **至少一个节**（Section）。分节符将文档分为不同的节，每节可独立设置页面方向、页边距、页眉页脚。

## 获取 Sections 集合

```python
sections = doc.Sections
count = doc.Sections.Count   # 节的个数
```

## 获取指定节

```python
sec = doc.Sections(1)        # 第一节（索引从1开始）
sec = doc.Sections(2)        # 第二节
last = doc.Sections(doc.Sections.Count)  # 最后一节
```

## Section 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `sec.Range` | Range | 该节的范围 |
| `sec.PageSetup` | PageSetup | 页面设置（可独立于其他节） |
| `sec.Headers` | Headers | 页眉集合 |
| `sec.Footers` | Footers | 页脚集合 |
| `sec.Index` | int | 索引（从1开始） |

## 各节独立页面设置

```python
# 第一节：纵向
doc.Sections(1).PageSetup.Orientation = 0   # wdOrientPortrait
# 第二节：横向
doc.Sections(2).PageSetup.Orientation = 1   # wdOrientLandscape
# 单独设置某节边距
doc.Sections(2).PageSetup.LeftMargin = 144  # 2英寸
```

## 插入分节符

```python
# 在 Range 末尾插入下一页分节符
rng.InsertBreak(2)  # 2 = wdSectionBreakNextPage

# 或使用 Paragraphs 的 Range
doc.Paragraphs(5).Range.InsertBreak(2)

# 其他分节符类型
# 2 = 下一页
# 3 = 连续
```

## Headers / Footers

```python
# 第一节页眉
header = doc.Sections(1).Headers(1)  # 1 = wdHeaderFooterPrimary（首页页眉）
header.Range.Text = "报告标题"

# 页脚
footer = doc.Sections(1).Footers(1)
footer.Range.Text = "第 1 页"

# 链接到上一节（默认 True，断开后可独立设置）
doc.Sections(2).Headers(1).LinkToPrevious = False
```

### Headers/Footers 索引

| 数值 | 常量名 | 说明 |
|------|--------|------|
| 1 | wdHeaderFooterPrimary | 页眉/页脚（非首页） |
| 2 | wdHeaderFooterFirstPage | 首页页眉/页脚 |
| 3 | wdHeaderFooterEvenPages | 偶数页页眉/页脚 |

## 注意事项

- 插入分节符后文档段落索引不变，但新节从分节符后的段落开始
- `LinkToPrevious = False` 必须在设置独立页眉/页脚内容之前调用
- WPS 的 Header/Footer 操作与 Word COM 完全兼容
