# Table 对象 & Tables 集合

## Tables 集合

```python
tables = doc.Tables
count = tables.Count                    # 文档中表格总数

# 获取指定表格（索引从 1 开始）
table = tables.Item(1)
```

## Table 对象属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `table.Rows.Count` | int | 行数 |
| `table.Columns.Count` | int | 列数 |
| `table.Range` | Range | 表格对应的 Range |
| `table.Title` | str | 表格标题 |
| `table.Descr` | str | 表格说明（无障碍） |

## Table 方法

| 方法 | 说明 |
|------|------|
| `table.Cell(Row, Column)` | 获取指定单元格（索引从1开始） |
| `table.Select()` | 选中整个表格 |
| `table.Delete()` | 删除表格 |
| `table.AutoFormat(Format)` | 自动套用格式（传入格式编号） |
| `table.AutoFitBehavior(Behavior)` | 自动调整列宽：0=固定，1=根据内容，2=根据窗口 |
| `table.Sort(...)` | 排序 |

## 单元格操作

```python
cell = table.Cell(row=1, col=2)       # 第1行第2列（索引从1开始）
text = cell.Range.Text                # 获取单元格文本
cell.Range.Font.Name = '宋体'          # 设置字体
cell.Range.Font.Size = 12             # 设置字号

# 设置单元格宽度
cell.Width = 100                       # 磅值
```

## Cell 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `cell.Range` | Range | 单元格内容 Range |
| `cell.Range.Text` | str | 单元格文本（末尾含 `\r\x07`） |
| `cell.Width` | float | 单元格宽度（磅） |
| `cell.Height` | float | 单元格高度（磅） |
| `cell.RowIndex` | int | 所在行索引 |
| `cell.ColumnIndex` | int | 所在列索引 |
| `cell.Tables` | Tables | 单元格内嵌套表格 |

## Row / Column 操作

```python
# 获取行
row = table.Rows(1)                   # 第1行
row.Select()
row.Delete()                          # 删除行

# 获取列
col = table.Columns(1)                # 第1列
col.Width = 150                       # 设置列宽
col.Select()
```

## 合并拆分单元格

```python
# 合并单元格
cell1 = table.Cell(1, 1)
cell2 = table.Cell(1, 2)
cell1.Merge(cell2)

# 拆分单元格
cell.Split(NumRows=1, NumColumns=2)
```

## 判断段落是否在表格中

格式化操作中通常需要跳过表格段落：

```python
def is_table_cell(para):
    try:
        return '表格' in (para.Style.NameLocal or '')
    except:
        return False

# 或用 Range.Information
is_in_table = para.Range.Information(13)  # 13 = wdWithinTable
```

## AutoFitBehavior 常量

| 常量名 | 数值 | 说明 |
|--------|------|------|
| wdAutoFitFixed | 0 | 固定列宽 |
| wdAutoFitContent | 1 | 根据内容调整 |
| wdAutoFitWindow | 2 | 根据窗口调整 |
