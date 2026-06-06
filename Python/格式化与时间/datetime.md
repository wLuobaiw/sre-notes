
# datetime 模块

Python 标准库 `datetime` 模块用于处理日期和时间，提供 `date`、`time`、`datetime`、`timedelta` 四个核心类。

## 获取当前时间

```python
from datetime import datetime, date, time, timedelta

now = datetime.now()        # 当前日期时间
today = date.today()        # 当前日期
```

## datetime 对象

```python
dt = datetime(2026, 6, 5, 14, 30, 0)   # 指定日期时间
dt.year        # 2026
dt.month       # 6
dt.day         # 5
dt.hour        # 14
dt.minute      # 30
dt.second      # 0
dt.weekday()   # 4 —— 0=周一，所以周五=4
```

## timedelta —— 时间差

```python
dt1 = datetime(2026, 6, 5)
dt2 = datetime(2026, 6, 10)
delta = dt2 - dt1           # timedelta(days=5)
delta.days                  # 5
```

## 字符串与 datetime 互转

```python
# str → datetime（解析）
dt = datetime.strptime("2026-06-05", "%Y-%m-%d")

# datetime → str（格式化）
s = dt.strftime("%Y-%m-%d %H:%M:%S")   # "2026-06-05 00:00:00"
```

## 常用格式化符号

| 符号 | 含义 | 示例 |
|------|------|------|
| `%Y` | 四位年份 | 2026 |
| `%m` | 月份（补零） | 06 |
| `%d` | 日期（补零） | 05 |
| `%H` | 24小时制小时 | 14 |
| `%M` | 分钟 | 30 |
| `%S` | 秒 | 00 |
| `%A` | 星期全名 | Friday |

# 本目录笔记索引

- [格式化基础](./格式化基础.md) —— 字符串格式化与 datetime 处理
- [格式化字符串](./格式化字符串.md) —— % / str.format() / f-string

# 相关笔记

- [Python基础](../Python基础.md)
