
# 是什么

SSH（Secure Shell）是一种加密的网络协议，用于在不安全的网络中安全地远程登录和执行命令。Linux 上最常用的实现是 OpenSSH。

# 安装与启动

```shell
# Debian/Ubuntu
apt install openssh-server

# RHEL/Rocky
dnf install openssh-server

systemctl enable --now sshd
```

# SSH 客户端——远程连接

## 基本用法

```shell
ssh user@host
ssh user@host -p 2222              # 指定端口（默认 22）
ssh -i ~/.ssh/id_rsa user@host     # 使用指定私钥
```

## 执行远程命令

```shell
ssh user@host 'ls -la /var/log'
ssh user@host 'df -h'
ssh user@host 'bash -s' < local_script.sh   # 在远程执行本地脚本
```

## 文件传输

```shell
# SCP——基于 SSH 的文件拷贝
scp local_file user@host:/remote/path/
scp -r local_dir/ user@host:/remote/path/
scp user@host:/remote/file ./local/

# SFTP——交互式文件传输
sftp user@host
sftp> get remote_file
sftp> put local_file
sftp> ls / ls -l
```

# SSH 密钥认证

密码登录有被暴力破解的风险，密钥认证更安全。

## 生成密钥对

```shell
ssh-keygen -t ed25519 -C "comment"          # 推荐——Ed25519 算法
ssh-keygen -t rsa -b 4096 -C "comment"      # RSA 4096 位
```

生成的文件：
- `~/.ssh/id_ed25519`——私钥（**绝不可泄露**）
- `~/.ssh/id_ed25519.pub`——公钥（放到服务器上）

## 配置免密登录

```shell
# 将公钥复制到远程服务器
ssh-copy-id user@host

# 或手动复制
cat ~/.ssh/id_ed25519.pub | ssh user@host 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'
```

## 密钥认证流程

```
客户端持有私钥 → 连接时发送公钥 → 服务器检查公钥是否在 authorized_keys 中
→ 是 → 用公钥加密随机挑战 → 客户端用私钥解密 → 返回结果 → 验证通过
```

# SSH 服务端配置

配置文件：`/etc/ssh/sshd_config`

```ini
# === 安全加固配置 ===

# 禁止 root 直接登录（用普通用户登录后 sudo）
PermitRootLogin no

# 禁用密码登录——只用密钥
PasswordAuthentication no

# 修改默认端口（减少扫描噪声）
Port 2222

# 允许的用户白名单
AllowUsers alice bob

# 禁止空密码
PermitEmptyPasswords no

# 限制登录尝试次数
MaxAuthTries 3

# 客户端心跳（防止僵死连接）
ClientAliveInterval 60
ClientAliveCountMax 3
```

修改后重载服务：

```shell
systemctl reload sshd
```

> **重要**：修改 SSH 配置前保留一个已连接的会话，用另一个会话测试新配置。配错可能导致无法登录。

# 客户端配置

`~/.ssh/config`——简化连接：

```ini
Host myserver
    HostName 192.168.1.100
    Port 2222
    User alice
    IdentityFile ~/.ssh/my_server_key

Host jump
    HostName jump.example.com
    User admin

# 使用跳板机
Host internal
    HostName 10.0.0.50
    User alice
    ProxyJump jump
```

配置后直接 `ssh myserver` 即可，无需每次指定端口和用户。

# SSH 隧道（端口转发）

SSH 可以在不暴露端口的情况下安全访问内部服务。

```shell
# 本地转发——将本地 8080 转发到远程服务器的 80
ssh -L 8080:localhost:80 user@host
# 本地访问 http://localhost:8080 即访问远程的 80 端口

# 远程转发——将远程 8080 转发到本地的 80
ssh -R 8080:localhost:80 user@host
# 远程访问 :8080 即访问本地的 80 端口

# 动态转发——SOCKS 代理
ssh -D 1080 user@host
# 浏览器配置 SOCKS5 代理为 localhost:1080
```

# 排障

```shell
# 测试是否能连接到 SSH 端口
telnet host 22
nc -zv host 22

# 查看 SSH 服务端日志
journalctl -u sshd -f

# 客户端详细调试
ssh -vvv user@host

# 检查密钥权限（必须严格）
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
```

# 注意

- 私钥权限必须是 600，否则 SSH 拒绝使用（安全设计）
- `/etc/ssh/sshd_config` 修改后要用 `systemctl reload sshd`（非 restart——不中断现有连接）
- 生产环境建议禁用密码登录和 root 登录
- Ed25519 密钥比 RSA 更安全且更短，是现代默认选择

> 返回 [系统服务基础](./系统服务基础.md)
