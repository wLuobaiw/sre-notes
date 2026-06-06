
# 是什么

FTP（File Transfer Protocol）用于在客户端和服务器之间传输文件。Linux 上常用 vsftpd（Very Secure FTP Daemon）作为服务端。

FTP 使用两个端口：
- **21**：控制连接（发送命令）
- **20**：数据连接（传输文件数据）

# 主动模式 vs 被动模式

| 模式 | 数据连接发起方 | 防火墙友好度 | 说明 |
|------|--------------|------------|------|
| 主动模式（PORT） | 服务器 → 客户端 | 差——客户端防火墙可能阻挡 | 服务器用 20 端口连接客户端随机端口 |
| 被动模式（PASV） | 客户端 → 服务器 | 好——推荐 | 客户端连接服务器的随机端口（需开放端口范围） |

> 现代网络环境默认使用**被动模式**。

# 安装与配置 vsftpd

```shell
# 安装
dnf install vsftpd        # RHEL/Rocky
apt install vsftpd        # Debian/Ubuntu

systemctl enable --now vsftpd
```

## 配置示例 `/etc/vsftpd/vsftpd.conf`

```ini
# === 基本配置 ===
anonymous_enable=NO           # 禁止匿名登录
local_enable=YES              # 允许本地用户登录
write_enable=YES              # 允许上传

# === 安全加固 ===
chroot_local_user=YES         # 将用户限制在其家目录中
allow_writeable_chroot=NO     # chroot 目录不可写（安全）

# === 被动模式端口范围 ===
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=31000

# === 限制 ===
max_clients=50
max_per_ip=5

# === 日志 ===
xferlog_enable=YES
xferlog_file=/var/log/vsftpd.log
```

修改配置后重载：

```shell
systemctl restart vsftpd
```

# 防火墙配置

FTP 的防火墙策略比其他服务复杂——需要开放控制端口 + 数据端口范围。

```shell
# firewalld 直接使用内置服务
firewall-cmd --add-service=ftp --permanent
firewall-cmd --reload

# iptables：需要显式开放被动模式端口范围
iptables -A INPUT -p tcp --dport 21 -j ACCEPT
iptables -A INPUT -p tcp --dport 30000:31000 -j ACCEPT
```

# 客户端使用

## 命令行客户端

```shell
# 交互式连接
ftp host

# 脚本化下载
ftp -n host << EOF
user username password
get remote_file local_file
quit
EOF
```

## curl / wget

```shell
curl -u username:password ftp://host/file
curl -T local_file ftp://host/upload/     # 上传
wget ftp://user:password@host/file
```

## lftp（推荐——功能更强）

```shell
lftp -u username host
lftp> ls
lftp> get file
lftp> mirror dir/       # 递归下载整个目录
lftp> put file
lftp> mirror -R dir/    # 递归上传
```

# FTP vs SFTP

| 特性 | FTP | SFTP（基于 SSH） |
|------|-----|-----------------|
| 加密 | 无——明文传输（含密码） | SSH 加密 |
| 端口 | 21+20/随机 | 22（复用 SSH 端口） |
| 防火墙 | 复杂（需被动模式端口范围） | 简单（只需 22） |
| 推荐度 | 遗留场景 | **首选**——更安全、更简单 |

> **结论**：如果只是需要安全的文件传输，优先用 SFTP（SSH 自带）而非 FTP。FTP 主要用于兼容遗留系统。

# 注意

- FTP **明文传输**用户名和密码——生产环境应使用 FTPS（FTP over TLS）或 SFTP
- vsftpd 启动前确保被动模式端口范围在防火墙中已开放
- `chroot_local_user=YES` 需要用户家目录不可写——这是安全特性，不是 bug
- SELinux 需要放行 FTP 相关策略：`setsebool -P ftpd_full_access on`（Enforcing 模式下）

> 返回 [系统服务基础](./系统服务基础.md)
