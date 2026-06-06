
# 是什么

NFS（Network File System）允许通过网络**挂载远程目录**——就像使用本地磁盘一样。它是 Linux/Unix 环境中最常用的网络文件共享方案。

# NFS 角色

| 角色 | 职责 | 常用命令 |
|------|------|---------|
| NFS 服务端 | 导出目录供客户端挂载 | `exportfs`、`/etc/exports` |
| NFS 客户端 | 挂载远程导出目录到本地 | `mount -t nfs` |

# 服务端配置

## 安装

```shell
# RHEL/Rocky
dnf install nfs-utils

# Debian/Ubuntu
apt install nfs-kernel-server
```

## 定义导出目录

编辑 `/etc/exports`：

```ini
# 格式：共享目录   客户端(选项) 客户端(选项)...

# 示例1：允许 192.168.1.0/24 网段读写访问
/data/shared    192.168.1.0/24(rw,sync,no_root_squash)

# 示例2：所有客户端只读
/data/public    *(ro,sync,no_subtree_check)

# 示例3：多个目录
/home           192.168.1.0/24(rw,sync,no_subtree_check)
/backup         192.168.1.10(rw,sync)
```

## 常用导出选项

| 选项 | 说明 |
|------|------|
| `rw` / `ro` | 读写 / 只读 |
| `sync` | 同步写入——数据立即写入磁盘（推荐，数据更安全） |
| `async` | 异步写入——性能更好但断电可能丢数据 |
| `no_root_squash` | 客户端的 root 保持 root 权限（默认会映射到 nobody） |
| `root_squash` | 客户端 root 映射到 `nobody`（默认，更安全） |
| `all_squash` | 所有用户都映射到 `nobody`（匿名共享） |
| `no_subtree_check` | 不检查子目录权限——提高性能 |
| `noexec` | 禁止在 NFS 卷上执行文件 |

## 应用配置

```shell
# 导出所有 /etc/exports 中定义的目录
exportfs -a

# 重新导出（修改 /etc/exports 后）
exportfs -ra

# 查看当前导出的目录
exportfs -v

# 取消导出某个目录
exportfs -u :/data/shared
```

## 启动服务

```shell
# RHEL/Rocky（需要启动 rpcbind 和 nfs-server）
systemctl enable --now rpcbind nfs-server

# Debian/Ubuntu
systemctl enable --now nfs-kernel-server
```

# 客户端配置

## 挂载

```shell
# 基本挂载
mount -t nfs server_ip:/data/shared /mnt/nfs

# 指定 NFS 版本
mount -t nfs4 server_ip:/data/shared /mnt/nfs
mount -t nfs -o vers=3 server_ip:/data/shared /mnt/nfs

# 常用挂载选项
mount -t nfs -o rw,hard,intr,noexec server_ip:/data/shared /mnt/nfs
```

## 客户端挂载选项

| 选项 | 说明 |
|------|------|
| `hard` | 连接失败时持续重试（推荐） |
| `soft` | 连接失败超时后返回错误 |
| `intr` | 允许在 hard 模式下中断等待（NFSv3） |
| `rsize=8192` | 读缓冲区大小（NFSv3 可调优） |
| `wsize=8192` | 写缓冲区大小 |
| `noexec` | 禁止在 NFS 上执行文件 |
| `nosuid` | 忽略 SUID/SGID |
| `noatime` | 不更新 atime——减少 IO |

## 开机自动挂载

在 `/etc/fstab` 中配置：

```ini
server_ip:/data/shared   /mnt/nfs   nfs   defaults,hard,noexec,_netdev   0 0
```

> `_netdev` 确保在网络就绪后才尝试挂载 NFS。

## 验证挂载

```shell
# 查看所有挂载的 NFS
mount | grep nfs
df -h /mnt/nfs

# 查看 NFS 服务器导出的目录
showmount -e server_ip
```

# 排障

```shell
# 检查客户端连接
nfsstat -c

# 检查 RPC 服务是否正常
rpcinfo -p server_ip

# NFS 挂载卡死——尝试强制卸载
umount -f /mnt/nfs     # 强制卸载
umount -l /mnt/nfs     # 延迟卸载（断开连接，等不忙时清理）
```

# 注意

- NFS 依赖 RPC（portmapper）——服务端需要 `rpcbind` 先启动
- `/etc/exports` 修改后必须执行 `exportfs -ra` 或 `systemctl reload nfs-server`
- NFS 在高延迟网络中性能较差——同一机柜内的服务器共享用 NFS 合适
- 客户端和服务器的 UID/GID 需统一——否则会出现权限错乱（root 除外）
- NFSv4 比 NFSv3 更安全（支持 Kerberos 认证）、性能更好、防火墙更友好（只需 2049 端口）
- 生产环境用 `hard` 挂载选项确保数据一致性——`soft` 可能导致静默数据丢失

> 返回 [系统服务基础](./系统服务基础.md)
