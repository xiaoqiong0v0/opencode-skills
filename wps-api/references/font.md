# Font 对象

通过 `Range.Font` 访问，控制文字的字号、字体、样式等。

## 访问方式

```python
f = para.Range.Font          # 获取段落的字体对象
f = doc.Range(0, 100).Font   # 获取指定区域的字体对象
f = wps.Selection.Font       # 获取当前选区的字体对象
```

## 属性

| 属性 | 类型 | 说明 | 示例值 |
|------|------|------|--------|
| `Font.Name` | str | 字体名称（西文） | `'Times New Roman'` |
| `Font.NameFarEast` | str | 东亚字体名称 | `'仿宋_GB2312'`、`'黑体'` |
| `Font.NameAscii` | str | ASCII 字符字体 |
| `Font.Size` | float | 字号（磅值） | `22.0`（二号）、`16.0`（三号） |
| `Font.Bold` | int | 加粗 | `-1`=加粗(wdToggle)，`0`=不加粗；也可能返回 `True`/`False` |
| `Font.Italic` | int | 斜体 | `-1`=斜体，`0`=不斜体 |
| `Font.Color` | int | 颜色（WdColor 枚举/RGB） | `0xFF0000`（蓝色） |
| `Font.ColorIndex` | int | 颜色索引 | `1`=黑，`2`=蓝，`6`=红 |
| `Font.Underline` | int | 下划线类型 | `0`=无，`1`=单线，`3`=双线，`11`=波浪线 |
| `Font.Superscript` | int | 上标 | `-1`/`True` 或 `0`/`False` |
| `Font.Subscript` | int | 下标 | `-1`/`True` 或 `0`/`False` |
| `Font.StrikeThrough` | int | 删除线 | `-1`/`True` 或 `0`/`False` |
| `Font.DoubleStrikeThrough` | int | 双删除线 |
| `Font.Shadow` | int | 阴影 |
| `Font.Outline` | int | 空心 |
| `Font.Emboss` | int | 阳文 |
| `Font.Engrave` | int | 阴文 |
| `Font.AllCaps` | int | 全部大写字母 |
| `Font.SmallCaps` | int | 小型大写字母 |
| `Font.Hidden` | int | 隐藏文字 |
| `Font.Position` | int | 字符位置（提升/降低，磅值） |
| `Font.Spacing` | float | 字符间距（磅值） |
| `Font.Scaling` | int | 字符缩放百分比 |
| `Font.Kerning` | int | 最小字距调整磅值 |
| `Font.DisableCharacterSpaceGrid` | int | 禁用字符网格 |
| `Font.EmphasisMark` | int | 着重号 |

## 常用字体名

| 字体名 | 用途 |
|--------|------|
| `'方正小标宋简体'` | 公文标题 |
| `'黑体'` | 一级标题 |
| `'楷体_GB2312'` | 二级标题 |
| `'仿宋_GB2312'` | 正文 |
| `'宋体'` | 表格文字 |
| `'Times New Roman'` | 英文/数字 |

## 中文字号 ↔ 磅值对照

| 字号 | 磅值 | Size 值 |
|------|------|---------|
| 初号 | 42 | 42.0 |
| 小初 | 36 | 36.0 |
| 一号 | 26 | 26.0 |
| 小一 | 24 | 24.0 |
| **二号** | **22** | **22.0** |
| 小二 | 18 | 18.0 |
| **三号** | **16** | **16.0** |
| 小三 | 15 | 15.0 |
| 四号 | 14 | 14.0 |
| 小四 | 12 | 12.0 |
| 五号 | 10.5 | 10.5 |
| 小五 | 9 | 9.0 |

## 实战示例

```python
# 公文标题
f = para.Range.Font
f.NameFarEast = '方正小标宋简体'
f.Name = '方正小标宋简体'
f.Size = 22.0           # 二号
f.Bold = False

# 正文
f.NameFarEast = '仿宋_GB2312'
f.Size = 16.0           # 三号
f.Bold = False

# 一级标题
f.NameFarEast = '黑体'
f.Size = 16.0           # 三号

# 二级标题
f.NameFarEast = '楷体_GB2312'
f.Size = 16.0           # 三号
f.Bold = True
```

## 注意事项

- `Font.Bold` 在 WPS COM 中可能返回 `-1`（加粗）或 `0`（不加粗），也可能返回 `True`/`False`
- 判断时用 `if f.Bold:` 而非 `if f.Bold == True`
- `Font.Name` 如果为空字符串 `''`，表示使用样式默认字体
- 中文字体建议同时设置 `Name` 和 `NameFarEast`，确保中西文分别生效
