
# 是什么

systemd 是 Linux 系统的初始化系统——内核加载完成后，第一个启动的用户空间进程（PID=1）。它管理系统中所有服务的启动、停止、监控和资源控制。

`systemctl` 是 systemd 的主要命令行工具，所有对服务的管理都通过它完成。

# 核心概念

## Unit——systemd 管理的资源单位

| Unit 类型 | 文件后缀 | 用途 |
|-----------|---------|------|
| service | `.service` | 系统服务（最常用） |
| target | `.target` | 一组 unit 的集合——类似旧的运行级别 |
| timer | `.timer` | 定时任务（替代 cron） |
| socket | `.socket` | 套接字激活 |
| mount | `.mount` | 挂载点管理 |
| path | `.path` | 路径监控——文件变化触发操作 |
| slice | `.slice` | 进程组资源控制（关联 cgroup） |
| device | `.device` | 设备文件 |
| swap | `.swap` | 交换分区 |

## Unit 文件位置（优先级从低到高）

| 目录 | 来源 |
|------|------|
| `/usr/lib/systemd/system/` | 软件包自带的 unit 文件 |
| `/run/systemd/system/` | 运行时动态创建的 unit |
| `/etc/systemd/system/` | 用户自定义 unit（覆盖前两者） |

# systemctl 常用命令

## 服务生命周期

```shell
systemctl start nginx.service       # 启动
systemctl stop nginx.service        # 停止
systemctl restart nginx.service     # 重启
systemctl reload nginx.service      # 重载配置（不中断服务）
systemctl status nginx.service      # 查看状态
```

## 开机自启

```shell
systemctl enable nginx.service      # 启用开机自启
systemctl disable nginx.service     # 禁用开机自启
systemctl is-enabled nginx.service  # 检查是否开机自启
```

## 查看服务

```shell
systemctl list-units --type service --all   # 列出所有服务
systemctl list-unit-files --type service    # 列出所有 unit 文件
systemctl list-dependencies nginx.service   # 查看服务依赖关系
```

## 查看状态

```shell
systemctl is-active nginx.service    # 是否运行中
systemctl is-failed nginx.service    # 是否启动失败

# 查看失败的 unit
systemctl --failed
```

# journalctl——日志查看

systemd 使用 journald 管理日志，通过 journalctl 查看：

```shell
journalctl -u nginx.service          # 查看某服务日志
journalctl -u nginx.service -f       # 实时跟踪（类似 tail -f）
journalctl --since "10 min ago"      # 查看最近 10 分钟日志
journalctl --since "2026-06-03 08:00" --until "2026-06-03 10:00"
journalctl -p err                     # 只看错误级别及以上
journalctl -n 50                     # 最近 50 条
journalctl -k                         # 内核日志
```

# 自定义 service 示例

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Custom Application
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp --config /etc/myapp.conf
Restart=on-failure
RestartSec=5s
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
StandardOutput=journal
StandardError=journal

# 资源限制（通过 cgroup）
MemoryMax=512M
CPUQuota=50%

[Install]
WantedBy=multi-user.target
```

## 加载自定义 service

```shell
systemctl daemon-reload              # 重新加载 unit 文件
systemctl enable --now myapp.service # 启用并立即启动
```

## [Install] 段说明

`WantedBy=multi-user.target` 意味着系统达到多用户模式时启动此服务。常见的 target：
- `multi-user.target`——多用户命令行模式（服务器默认）
- `graphical.target`——图形界面模式
- `network-online.target`——网络完全就绪后

# 与 cgroup 的集成

每个 service 自动放入独立的 cgroup。通过 `systemctl status` 可以看到 cgroup 路径。资源限制在 service 文件中直接配置（`MemoryMax`、`CPUQuota` 等），底层由 cgroup 执行。详见 [cgroup](../资源管理/cgroup.md)。

# 注意

- 修改 unit 文件后必须执行 `systemctl daemon-reload`，否则不生效
- `systemctl restart` 会中断服务——建议用 `reload` 热重载配置
- 对于需要长期运行的脚本或服务，用 systemd 替代 nohup/screen/tmux
- 日志默认存储在 journal 中，不会自动写入 `/var/log/` 下的文本文件——需要配置 `ForwardToSyslog=yes` 或 rsyslog 转发

> 返回 [进程管理基础](./进程管理基础.md)
