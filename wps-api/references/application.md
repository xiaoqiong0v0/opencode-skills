# Application 对象

## ProgID

| ProgID | 说明 |
|--------|------|
| `Kwps.Application` | WPS Office 12.x 个人版/专业版（推荐） |
| `WPS.Application` | 旧版/兼容模式 |
| `Et.Application` | WPS 表格 |
| `Wpp.Application` | WPS 演示 |

## 连接与断开

```python
import win32com.client

# 晚期绑定（推荐，无需类型库）
wps = win32com.client.Dispatch('Kwps.Application')

# 或早期绑定（需提前引用类型库，可能有缓存问题）
# wps = win32com.client.Dispatch('WPS.Application')
```

### 32位 / 64位兼容性

这是最常见的连接失败原因：

| Python 位数 | WPS 位数 | 能否连接 |
|-------------|----------|---------|
| 32-bit | 32-bit（WPS个人版默认） | ✅ |
| 64-bit | 64-bit（WPS专业版可选） | ✅ |
| **64-bit** | **32-bit** | **❌ 失败** |

如果连接 `Kwps.Application` 失败：
1. 检查 Python 位数（`python -c "import struct; print(struct.calcsize('P') * 8)"`）
2. 如果 Python 是 64-bit，WPS 是 32-bit → 改用 32-bit Python 运行
3. 或者安装 64-bit 版本 WPS（专业版支持）

### 连接到已有 WPS 实例

```python
try:
    wps = win32com.client.GetActiveObject('Kwps.Application')
except:
    wps = win32com.client.Dispatch('Kwps.Application')
```

## 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `wps.Version` | str | WPS 版本号，如 `"12.0"` |
| `wps.Name` | str | 应用程序名称 |
| `wps.Visible` | bool | 窗口可见性：`True`（前台）/ `False`（后台） |
| `wps.Documents` | Documents | 文档集合 |
| `wps.ActiveDocument` | Document | 当前活动文档 |
| `wps.ActiveWindow` | Window | 当前活动窗口 |
| `wps.Options` | Options | 应用程序选项 |
| `wps.Path` | str | WPS 安装路径 |
| `wps.Caption` | str | 窗口标题栏文字 |
| `wps.WindowState` | int | 窗口状态：0=正常，1=最大化，2=最小化 |
| `wps.DisplayAlerts` | int | 是否显示警告对话框（0=不显示） |
| `wps.ScreenUpdating` | bool | 是否刷新屏幕（关闭可提速） |

## 方法

| 方法 | 说明 |
|------|------|
| `wps.Quit()` | 退出 WPS，必须显式调用以释放 COM |
| `wps.Activate()` | 激活 WPS 窗口 |

## 性能优化

```python
wps.ScreenUpdating = False   # 关闭屏幕刷新，大幅提速
wps.DisplayAlerts = 0        # 关闭弹窗提示
try:
    # ... 批量操作 ...
finally:
    wps.ScreenUpdating = True
    wps.DisplayAlerts = -1
```

## 常见问题

**连接失败（无效类字符串）**：
- 确保 WPS 已正确安装
- 尝试 `Kwps.Application` 和 `WPS.Application` 两个 ProgID
- WPS 专业版/政务版可能使用不同的 ProgID
