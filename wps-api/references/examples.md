# 实战示例

## 完整示例：公文格式调整

以下脚本按照标准公文格式要求，批量调整 Word 文档格式：

- 标题：方正小标宋简体，二号(22pt)，不加粗，居中，单倍行距
- 一级标题：黑体，三号(16pt)，不加粗，1.5 倍行距
- 二级标题：楷体_GB2312，三号(16pt)，加粗，1.5 倍行距
- 三级/四级标题：仿宋_GB2312，三号(16pt)，加粗，1.5 倍行距
- 正文：仿宋_GB2312，三号(16pt)，不加粗，1.5 倍行距，首行缩进2字符
- 附件标签：三号黑体，不加粗
- 附件标题：方正小标宋简体，二号(22pt)，不加粗，居中，单倍行距

```python
import win32com.client
import os, glob, re

# ===== 配置 =====
INPUT_DIR = r'D:\input'
OUTPUT_DIR = r'D:\output'
os.makedirs(OUTPUT_DIR, exist_ok=True)

# 字号（磅值）
SIZE_TITLE = 22.0    # 二号
SIZE_BODY = 16.0     # 三号

# 枚举常量
ALIGN_CENTER = 1
ALIGN_JUSTIFY = 3
LINE_SINGLE = 0      # 单倍行距
LINE_1PT5 = 1        # 1.5倍行距

# 已知的一级标题关键词（非中文数字开头的情况）
L1_KEYWORDS = ['不足和问题', '优化和改进建议', '特色和亮点', '基本结果', '评价结论']


def classify(p, idx):
    """识别段落类型，返回: title|level1|level2|level3|attachment|body|empty|skip"""
    text = p.Range.Text.strip()
    if not text:
        return 'empty'

    # 表格内段落：跳过
    if '表格' in (p.Style.NameLocal or ''):
        return 'skip'

    f = p.Range.Font
    bold = f.Bold  # -1/True = 加粗, 0/False = 不加粗

    # 标题
    if idx == 1 and f.Size >= 20:
        return 'title'

    # 附件标签
    m = re.match(r'^附件[：:]\s*$', text)
    if m:
        return 'attachment'
    # 附件标题
    m = re.match(r'^附件[：:]\s*(.+)', text)
    if m:
        return 'attachment_title'

    # 一级标题
    if f.Name == '黑体' and f.Size == SIZE_BODY:
        return 'level1'
    if text[0] in '一二三四五六七八九十' and '、' in text[:3]:
        return 'level1'
    for kw in L1_KEYWORDS:
        if text.startswith(kw):
            return 'level1'

    # 二级标题（加粗 + 数字序号 + 较短）
    if bold and text[0].isdigit() and len(text) <= 40:
        return 'level2'

    # 三级标题（加粗 + 其它序号模式）
    if bold and len(text) <= 30:
        if re.match(r'^[（(]\d+[）)]', text):
            return 'level3'

    return 'body'


def format_doc(input_path, output_path):
    wps = win32com.client.Dispatch('Kwps.Application')
    wps.Visible = False
    try:
        doc = wps.Documents.Open(input_path)
        total = doc.Paragraphs.Count

        for i in range(1, total + 1):
            p = doc.Paragraphs(i)
            t = classify(p, i)
            f = p.Range.Font
            pf = p.Range.ParagraphFormat

            if t == 'empty':
                pass
            elif t == 'skip':
                pass
            elif t == 'title':
                f.Name = '方正小标宋简体'
                f.Size = SIZE_TITLE
                f.Bold = False
                pf.Alignment = ALIGN_CENTER
                pf.LineSpacingRule = LINE_SINGLE
            elif t == 'level1':
                f.Name = '黑体'
                f.Size = SIZE_BODY
                f.Bold = False
                pf.LineSpacingRule = LINE_1PT5
            elif t == 'level2':
                f.Name = '楷体_GB2312'
                f.Size = SIZE_BODY
                f.Bold = True
                pf.LineSpacingRule = LINE_1PT5
            elif t == 'level3':
                f.Name = '仿宋_GB2312'
                f.Size = SIZE_BODY
                f.Bold = True
                pf.LineSpacingRule = LINE_1PT5
            elif t == 'attachment':
                f.Name = '黑体'
                f.Size = SIZE_BODY
                f.Bold = False
            elif t == 'attachment_title':
                f.Name = '方正小标宋简体'
                f.Size = SIZE_TITLE
                f.Bold = False
                pf.Alignment = ALIGN_CENTER
                pf.LineSpacingRule = LINE_SINGLE
            elif t == 'body':
                f.Name = '仿宋_GB2312'
                f.Size = SIZE_BODY
                f.Bold = False
                pf.LineSpacingRule = LINE_1PT5

        doc.SaveAs(output_path)
        doc.Close()
        return True
    finally:
        wps.Quit()


# 批量处理
for f in sorted(glob.glob(os.path.join(INPUT_DIR, '*.docx'))):
    out = os.path.join(OUTPUT_DIR, os.path.basename(f))
    ok = format_doc(f, out)
    print(f'{"✅" if ok else "❌"} {os.path.basename(f)}')
```

## 最小模板

```python
import win32com.client

wps = win32com.client.Dispatch('Kwps.Application')
wps.Visible = False

doc = wps.Documents.Open(r'input.docx')

# 遍历所有段落
for i in range(1, doc.Paragraphs.Count + 1):
    p = doc.Paragraphs(i)
    text = p.Range.Text.strip()
    if not text:
        continue

    f = p.Range.Font
    pf = p.Range.ParagraphFormat

    # TODO: 你的格式逻辑

doc.SaveAs(r'output.docx')
doc.Close()
wps.Quit()
```

## Word 兼容模式

如果 WPS 不可用，回退到 Microsoft Word：

```python
import win32com.client

try:
    wps = win32com.client.Dispatch('Kwps.Application')
except:
    try:
        wps = win32com.client.Dispatch('Word.Application')
    except:
        raise RuntimeError('WPS 和 Word 均不可用')
```
