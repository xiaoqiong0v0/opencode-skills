# Find & Replace

## 基础查找替换

```python
find = doc.Range().Find

# 简单替换
find.Text = '旧文本'
find.Replacement.Text = '新文本'
found = find.Execute(Replace=2)   # 2 = wdReplaceAll 全部替换
```

## 使用 Selection.Find（更简单）

```python
# 先全选文档
doc.Select()
find = wps.Selection.Find

find.Text = '查找内容'
find.Replacement.Text = '替换内容'
find.Forward = True
find.Wrap = 1           # wdFindContinue
find.Execute(Replace=2) # wdReplaceAll
```

## Find.Execute 参数

```python
found = find.Execute(
    FindText='要查找的文本',       # 查找文本
    MatchCase=False,               # 区分大小写
    MatchWholeWord=False,          # 全字匹配
    MatchWildcards=False,          # 使用通配符
    MatchSoundsLike=False,         # 同音匹配
    MatchAllWordForms=False,       # 查找单词所有形式
    Forward=True,                  # 向前搜索
    Wrap=1,                        # 搜索范围：0=停止，1=循环，2=询问
    Format=False,                  # 按格式查找
    ReplaceWith='替换为文本',       # 替换文本
    Replace=2,                     # 0=不替换，1=替换一个，2=全部替换
    MatchPrefix=False,             # 匹配前缀
    MatchSuffix=False,             # 匹配后缀
    MatchPhrase=False,             # 忽略空格差异
    IgnoreSpace=False,             # 忽略空格
    IgnorePunct=False              # 忽略标点
)
```

## Find 对象属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `find.Text` | str | 要查找的文本 |
| `find.Replacement.Text` | str | 替换文本 |
| `find.Forward` | bool | 向前搜索 |
| `find.Wrap` | int | 搜索结束后的行为（见枚举） |
| `find.MatchCase` | bool | 区分大小写 |
| `find.MatchWholeWord` | bool | 全字匹配 |
| `find.MatchWildcards` | bool | 使用通配符 |
| `find.MatchSoundsLike` | bool | 同音匹配 |
| `find.MatchAllWordForms` | bool | 词形匹配 |
| `find.Found` | bool | 上一次查找是否成功（只读） |
| `find.Format` | bool | 按格式查找 |
| `find.Font` | Font | 按字体格式查找 |

## 替换枚举值

| 常量 | 数值 | 说明 |
|------|------|------|
| wdReplaceNone | 0 | 不替换 |
| wdReplaceOne | 1 | 替换一个 |
| wdReplaceAll | 2 | 全部替换 |

## Wrap 枚举值

| 常量 | 数值 | 说明 |
|------|------|------|
| wdFindStop | 0 | 搜索到文档末尾停止 |
| wdFindContinue | 1 | 到末尾后从头继续循环 |
| wdFindAsk | 2 | 询问用户是否继续 |

## 带格式的查找

```python
find = doc.Range().Find
find.Text = ''
find.Font.Bold = True          # 查找加粗文字
find.Font.Name = '黑体'         # 查找黑体文字
found = find.Execute()
```

## 通配符示例

```python
find = doc.Range().Find
find.MatchWildcards = True

# 查找中文数字序号（如"一、"到"二十、"）
find.Text = '[一二三四五六七八九十]{1,2}、'
found = find.Execute()

# 查找任意数字 + "." 开头
find.Text = '[0-9]{1,2}[.]'
found = find.Execute()
```
