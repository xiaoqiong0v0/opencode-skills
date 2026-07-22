# Range 对象

Range 是 WPS 文字中最核心的内容操作对象，代表文档中的一个连续文本区域。文本的读写、格式设置、查找替换几乎都通过 Range 完成。

## 获取 Range

```python
# 文档全部内容
rng = doc.Content
rng = doc.Range()

# 指定字符偏移范围 (Start, End)，0-based
rng = doc.Range(0, 100)       # 前100个字符

# 段落的 Range
rng = doc.Paragraphs(1).Range

# 当前选区的 Range
rng = wps.Selection.Range
```

## 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `rng.Text` | str | 文本内容（读写） |
| `rng.Start` | int | 起始位置（字符偏移，0-based） |
| `rng.End` | int | 结束位置 |
| `rng.Font` | Font | 字体对象 |
| `rng.ParagraphFormat` | ParagraphFormat | 段落格式对象 |
| `rng.Paragraphs` | Paragraphs | 该区域内的段落集合 |
| `rng.Tables` | Tables | 该区域内的表格集合 |
| `rng.Style` | Style | 该区域的样式 |
| `rng.StoryType` | int | 区域类型（正文/页眉页脚等） |
| `rng.Characters` | Characters | 该区域内的字符集合 |
| `rng.Words` | Words | 该区域内的单词集合 |
| `rng.Sentences` | Sentences | 该区域内的句子集合 |
| `rng.HighlightColorIndex` | int | 高亮颜色 |
| `rng.Bold` | int | 是否全部加粗（只读）；-1=全是，0=全不是，9999999=部分 |
| `rng.Italic` | int | 是否全部斜体（只读） |

## 方法

| 方法 | 说明 |
|------|------|
| `rng.Select()` | 选中该区域 |
| `rng.Copy()` | 复制 |
| `rng.Cut()` | 剪切 |
| `rng.Paste()` | 粘贴 |
| `rng.Delete()` | 删除内容 |
| `rng.InsertBefore(Text)` | 在区域前插入文本，区域会后移 |
| `rng.InsertAfter(Text)` | 在区域末尾插入文本 |
| `rng.InsertParagraphAfter()` | 在区域后插入新段落 |
| `rng.InsertParagraphBefore()` | 在区域前插入新段落 |
| `rng.InsertBreak(Type)` | 插入分隔符 |
| `rng.SetRange(Start, End)` | 重新定义区域范围 |
| `rng.Collapse(Direction)` | 折叠区域：`0`=到起始位置(wdCollapseStart)，`1`=到结束位置(wdCollapseEnd) |
| `rng.Move(Unit, Count)` | 移动区域（不扩展） |
| `rng.MoveEnd(Unit, Count)` | 移动结束位置（可扩展） |
| `rng.MoveStart(Unit, Count)` | 移动起始位置 |
| `rng.Expand(Unit)` | 扩展区域到指定单元 |
| `rng.Next(Unit, Count)` | 返回下一个指定单元的 Range |
| `rng.Previous(Unit, Count)` | 返回上一个指定单元的 Range |
| `rng.ComputeStatistics(Statistic)` | 统计字数/段落数等 |
| `rng.WholeStory()` | 扩展到整个 Story |

## Move/MoveEnd/MoveStart 的 Unit 参数

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdCharacter | 1 | 字符 |
| wdWord | 2 | 单词 |
| wdSentence | 3 | 句子 |
| wdParagraph | 4 | 段落 |
| wdSection | 5 | 节 |
| wdStory | 6 | Story |
| wdCell | 12 | 单元格 |
| wdColumn | 9 | 列 |
| wdRow | 10 | 行 |
| wdLine | 5 | 行 |

## StoryType 枚举

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdMainTextStory | 1 | 正文 |
| wdFootnotesStory | 2 | 脚注 |
| wdEndnotesStory | 3 | 尾注 |
| wdCommentsStory | 4 | 批注 |
| wdTextFrameStory | 5 | 文本框 |
| wdPrimaryHeaderStory | 7 | 首页页眉 |
| wdEvenPagesHeaderStory | 8 | 偶数页页眉 |
| wdPrimaryFooterStory | 9 | 首页页脚 |
| wdEvenPagesFooterStory | 10 | 偶数页页脚 |
| wdFirstPageHeaderStory | 11 | 第一页页眉 |
| wdFirstPageFooterStory | 12 | 第一页页脚 |

## InsertBreak 的 Type 参数

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdPageBreak | 7 | 分页符 |
| wdColumnBreak | 8 | 分栏符 |
| wdSectionBreakNextPage | 2 | 下一页分节符 |
| wdSectionBreakContinuous | 3 | 连续分节符 |
| wdLineBreak | 6 | 换行符（Shift+Enter） |
