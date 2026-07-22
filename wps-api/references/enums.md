# 枚举常量

Python 晚期绑定（`Dispatch`）无类型库，无法使用 `win32.constants.wd*` 常量，直接使用以下数值。

## 对齐方式（Alignment / ParagraphFormat.Alignment）

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdAlignParagraphLeft | **0** | 左对齐 |
| wdAlignParagraphCenter | **1** | 居中 |
| wdAlignParagraphRight | **2** | 右对齐 |
| wdAlignParagraphJustify | **3** | 两端对齐 |
| wdAlignParagraphDistribute | **4** | 分散对齐 |
| wdAlignParagraphThaiJustify | **5** | 泰文分散对齐 |

## 行距规则（LineSpacingRule）

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdLineSpaceSingle | **0** | 单倍行距 |
| wdLineSpace1pt5 | **1** | 1.5 倍行距 |
| wdLineSpaceDouble | **2** | 双倍行距 |
| wdLineSpaceAtLeast | **3** | 最小值 |
| wdLineSpaceExactly | **4** | 固定值 |
| wdLineSpaceMultiple | **5** | 多倍行距 |

## 大纲级别（OutlineLevel）

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdOutlineLevel1 | 1 | 1 级标题 |
| wdOutlineLevel2 | 2 | 2 级标题 |
| wdOutlineLevel3 | 3 | 3 级标题 |
| wdOutlineLevel4 | 4 | 4 级标题 |
| wdOutlineLevel5 | 5 | 5 级标题 |
| wdOutlineLevel6 | 6 | 6 级标题 |
| wdOutlineLevel7 | 7 | 7 级标题 |
| wdOutlineLevel8 | 8 | 8 级标题 |
| wdOutlineLevel9 | 9 | 9 级标题 |
| wdOutlineLevelBodyText | **10** | 正文文本 |

## 下划线类型（Underline）

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdUnderlineNone | **0** | 无下划线 |
| wdUnderlineSingle | **1** | 单线 |
| wdUnderlineWords | **2** | 只文字下划线 |
| wdUnderlineDouble | **3** | 双线 |
| wdUnderlineDotted | **4** | 点线 |
| wdUnderlineThick | **6** | 粗线 |
| wdUnderlineDash | **7** | 虚线 |
| wdUnderlineDotDash | **9** | 点划线 |
| wdUnderlineDotDotDash | **10** | 点点划线 |
| wdUnderlineWavy | **11** | 波浪线 |
| wdUnderlineWavyHeavy | **27** | 粗波浪线 |
| wdUnderlineDottedHeavy | **20** | 粗点线 |
| wdUnderlineDashHeavy | **23** | 粗虚线 |

## 颜色索引（ColorIndex）

| 常量名 | 数值 |
|--------|------|
| wdAuto | **0** |
| wdBlack | **1** |
| wdBlue | **2** |
| wdTurquoise | **3** |
| wdBrightGreen | **4** |
| wdPink | **5** |
| wdRed | **6** |
| wdYellow | **7** |
| wdWhite | **8** |
| wdDarkBlue | **9** |
| wdTeal | **10** |
| wdGreen | **11** |
| wdViolet | **12** |
| wdDarkRed | **13** |
| wdDarkYellow | **14** |
| wdGray50 | **15** |
| wdGray25 | **16** |

## 窗口状态（WindowState）

| 常量名 | 数值 |
|--------|------|
| wdWindowStateNormal | **0** |
| wdWindowStateMaximize | **1** |
| wdWindowStateMinimize | **2** |

## 文档视图（View.Type）

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdNormalView | **1** | 普通视图 |
| wdOutlineView | **2** | 大纲视图 |
| wdPrintView | **3** | 页面视图（默认） |
| wdWebView | **6** | Web 版式视图 |
| wdReadingView | **7** | 阅读版式视图 |
| wdMasterView | **5** | 主控文档视图 |

## 页面方向（PageSetup.Orientation）

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdOrientPortrait | **0** | 纵向 |
| wdOrientLandscape | **1** | 横向 |

## 查找替换 — Replace 参数

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdReplaceNone | **0** | 不替换 |
| wdReplaceOne | **1** | 替换一个 |
| wdReplaceAll | **2** | 全部替换 |

## 查找替换 — Wrap 参数

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdFindStop | **0** | 搜索结束停止 |
| wdFindContinue | **1** | 循环搜索 |
| wdFindAsk | **2** | 询问是否继续 |

## 折叠方向（Range.Collapse）

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdCollapseStart | **0** | 折叠到起始位置 |
| wdCollapseEnd | **1** | 折叠到结束位置 |

## 分隔符类型（Range.InsertBreak）

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdPageBreak | **7** | 分页符 |
| wdColumnBreak | **8** | 分栏符 |
| wdSectionBreakNextPage | **2** | 下一页分节符 |
| wdSectionBreakContinuous | **3** | 连续分节符 |
| wdLineBreak | **6** | 换行符（Shift+Enter） |

## 表格自动调整（Table.AutoFitBehavior）

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdAutoFitFixed | **0** | 固定列宽 |
| wdAutoFitContent | **1** | 根据内容调整 |
| wdAutoFitWindow | **2** | 根据窗口调整 |

## 导出格式（ExportAsFixedFormat）

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdExportFormatPDF | **17** | PDF |
| wdExportFormatXPS | **18** | XPS |
