# ParagraphFormat 对象

控制段落级别的格式：对齐方式、行距、缩进、段前段后间距等。
通过 `Range.ParagraphFormat` 或 `Paragraph.Format` 访问。

## 访问方式

```python
pf = para.Range.ParagraphFormat    # 段落的段落格式
pf = doc.Paragraphs(1).Range.ParagraphFormat
```

## 属性

| 属性 | 类型 | 说明 | 典型值 |
|------|------|------|--------|
| `pf.Alignment` | int | 对齐方式 | `0`=左，`1`=居中，`3`=两端对齐，`4`=分散对齐 |
| `pf.LineSpacingRule` | int | 行距规则 | `0`=单倍，`1`=1.5倍，`2`=双倍，`3`=最小值，`4`=固定值，`5`=多倍 |
| `pf.LineSpacing` | float | 行距值（含义取决于 LineSpacingRule） |
| `pf.FirstLineIndent` | float | 首行缩进（磅，正数值） | `32.0`（约2字符缩进） |
| `pf.LeftIndent` | float | 左缩进（磅） |
| `pf.RightIndent` | float | 右缩进（磅） |
| `pf.CharacterUnitFirstLineIndent` | float | 首行缩进（字符数） | `2.0` |
| `pf.CharacterUnitLeftIndent` | float | 左缩进（字符数） |
| `pf.CharacterUnitRightIndent` | float | 右缩进（字符数） |
| `pf.SpaceBefore` | float | 段前间距（磅） | `6.0` |
| `pf.SpaceAfter` | float | 段后间距（磅） | `6.0` |
| `pf.SpaceBeforeAuto` | int | 段前间距自动 | `0`=手动，`-1`=自动 |
| `pf.SpaceAfterAuto` | int | 段后间距自动 | `0`=手动，`-1`=自动 |
| `pf.OutlineLevel` | int | 大纲级别 | `1`-`9`=标题级别，`10`=正文 |
| `pf.WidowControl` | int | 孤行控制 | `-1`=开，`0`=关 |
| `pf.KeepWithNext` | int | 与下段同页 | `-1`=开，`0`=关 |
| `pf.KeepTogether` | int | 段中不分页 | `-1`=开，`0`=关 |
| `pf.PageBreakBefore` | int | 段前分页 | `-1`=开，`0`=关 |
| `pf.Hyphenation` | int | 自动断字 | `-1`=开，`0`=关 |
| `pf.FirstLineIndent` | float | 首行缩进 | 正数=缩进，负数=悬挂 |
| `pf.TabStops` | TabStops | 制表位集合 |
| `pf.Borders` | Borders | 段落边框 |
| `pf.Shading` | Shading | 段落底纹 |

### LineSpacingRule 取值时 LineSpacing 的含义

| LineSpacingRule 值 | LineSpacing 含义 |
|--------------------|------------------|
| 0 (单倍行距) | 自动计算，忽略此值 |
| 1 (1.5倍行距) | 自动计算，忽略此值 |
| 2 (双倍行距) | 自动计算，忽略此值 |
| 3 (最小值) | 最小行距磅值 |
| 4 (固定值) | 精确行距磅值 |
| 5 (多倍行距) | 行数为倍数（如 1.15 = 1.15倍） |

## 方法

| 方法 | 说明 |
|------|------|
| `pf.Reset()` | 删除手动设置的段落格式，恢复为样式默认值 |
| `pf.Space1()` | 设置为单倍行距 |
| `pf.Space15()` | 设置为 1.5 倍行距 |
| `pf.Space2()` | 设置为双倍行距 |
| `pf.OpenUp()` | 设置段前间距为 12 磅 |
| `pf.CloseUp()` | 清除段前间距（设为 0） |
| `pf.TabHangingIndent(Count)` | 悬挂缩进（指定制表位数） |
| `pf.TabIndent(Count)` | 首行缩进（指定制表位数） |
| `pf.IndentCharWidth(Count)` | 增加缩进量（以字符数为单位） |

## 实战示例

```python
pf = para.Range.ParagraphFormat

# 正文：两端对齐，1.5倍行距，首行缩进2字符
pf.Alignment = 3                        # wdAlignParagraphJustify
pf.LineSpacingRule = 1                  # wdLineSpace1pt5
pf.CharacterUnitFirstLineIndent = 2.0   # 首行缩进2字符

# 标题：居中，单倍行距，无首行缩进
pf.Alignment = 1                        # wdAlignParagraphCenter
pf.LineSpacingRule = 0                  # wdLineSpaceSingle
pf.FirstLineIndent = 0                  # 取消首行缩进
```
