
# 是什么

grep（Global Regular Expression Print）在文本中按行搜索匹配正则表达式的模式，输出匹配的行。它是 Linux 日常运维中使用频率最高的文本搜索工具。

# 语法

```shell
grep [选项] 模式 [文件...]
```

# 常用选项

| 选项 | 作用 |
|------|------|
| `-i` | 忽略大小写 |
| `-v` | 反向匹配——输出不匹配的行 |
| `-n` | 显示行号 |
| `-c` | 只输出匹配行数 |
| `-l` | 只输出包含匹配的文件名 |
| `-r` / `-R` | 递归搜索目录 |
| `-w` | 匹配整个单词 |
| `-x` | 匹配整行 |
| `-A N` | 同时输出匹配行**之后** N 行 |
| `-B N` | 同时输出匹配行**之前** N 行 |
| `-C N` | 同时输出匹配行**前后**各 N 行 |
| `-E` | 扩展正则表达式（等同于 `egrep`） |
| `-F` | 固定字符串匹配（等同于 `fgrep`） |
| `-o` | 只输出匹配的文本片段，而非整行 |
| `-P` | 使用 Perl 兼容正则（PCRE） |
| `-q` | 安静模式——不输出，只通过退出码表示是否匹配 |
| `--color=auto` | 高亮匹配的文本 |

# 正则表达式基础

| 元字符 | 含义 | 示例 |
|--------|------|------|
| `.` | 匹配任意单个字符 | `h.t` 匹配 hat, hit, h3t |
| `^` | 行首 | `^ERROR` 匹配以 ERROR 开头的行 |
| `$` | 行尾 | `:$` 匹配以冒号结尾的行 |
| `*` | 前一个字符重复 0 次或多次 | `ab*c` 匹配 ac, abc, abbc |
| `+` | 前一个字符重复 1 次或多次（需 `-E`） | `ab+c` 匹配 abc, abbc |
| `?` | 前一个字符可选（需 `-E`） | `colou?r` 匹配 color, colour |
| `[]` | 字符集合 | `[Ff]atal` 匹配 Fatal, fatal |
| `[^]` | 字符集合取反 | `[^0-9]` 匹配非数字字符 |
| `\|` | 或（需 `-E`） | `error\|fatal` 匹配 error 或 fatal |
| `()` | 分组（需 `-E`） | `(error\|fatal).*occurred` |
| `{}` | 重复次数（需 `-E`） | `[0-9]{3}` 匹配 3 位数字 |
| `\b` | 单词边界 | `\bcat\b` 只匹配单词 cat |

## 基本正则 vs 扩展正则

- **基本正则（BRE）**：`?` `+` `{}` `()` `|` 需要反斜杠转义：`\?` `\+` `\{` `\(` `\|`
- **扩展正则（ERE）**：上述字符直接使用，加 `-E` 选项或直接使用 `egrep`

**推荐统一使用 `grep -E`**，语法更直观。

# 实用示例

```shell
# 搜索错误日志
grep -i 'error' /var/log/messages

# 递归搜索代码中的 TODO，只搜索 .py 文件
grep -r 'TODO' --include='*.py' ./src

# 查看 nginx 进程（过滤掉 grep 自身）
ps aux | grep '[n]ginx'

# 统计匹配行数
grep -c '404' /var/log/nginx/access.log

# 搜索不含注释的有效配置行
grep -vE '^\s*#|^\s*$' /etc/ssh/sshd_config

# 提取 IP 地址（-o 只输出匹配部分）
grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' access.log | sort | uniq -c | sort -rn

# 安静模式——脚本中判断是否存在
if grep -q 'pattern' file; then
    echo "found"
fi
```

# 注意

- 不要 `cat file | grep pattern`，直接 `grep pattern file`——少一个进程，且 grep 能显示文件名
- `grep -E` 已替代 `egrep`，后者逐步废弃
- 递归搜索大目录时建议用 `--include` 限制文件类型，或用 `rg`（ripgrep）替代
- 搜索含特殊字符的字符串时用 `grep -F`（固定字符串，无正则转义问题）

> 返回 [文件操作](./文件操作.md)
