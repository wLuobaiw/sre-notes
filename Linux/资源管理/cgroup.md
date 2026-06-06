
# 是什么

cgroup（Control Groups，控制组）是 Linux 内核提供的**资源限制和隔离**机制。它把一组进程绑定到一起，对这组进程统一限制 CPU、内存、IO 等资源用量。

容器技术的基石——Docker 的 `--memory` / `--cpus`、Kubernetes 的 `resources.limits` 底层都是 cgroup。

# 核心功能

| 功能 | 说明 |
|------|------|
| 资源限制 | 限制进程组能使用的 CPU、内存、IO 上限 |
| 优先级控制 | 不同组分配不同的资源权重 |
| 资源审计 | 统计各组实际使用的资源量 |
| 进程控制 | 挂起/恢复/冻结进程组 |

# cgroup v1 vs v2

| 特性 | v1 | v2 |
|------|----|----|
| 层次结构 | 每个控制器独立树 | 统一一棵树 |
| 进程归属 | 进程可归属不同树的任意节点 | 进程只能位于叶节点 |
| 内核默认 | RHEL 7、旧版 Ubuntu | RHEL 8+、Ubuntu 22.04+、现代发行版 |
| 推荐 | 逐步淘汰 | **当前标准** |

RHEL 9 / Rocky 9 默认纯 cgroup v2 模式。

# 常用子系统（控制器）

| 控制器 | 作用 | 典型限制 |
|--------|------|---------|
| cpu | CPU 时间分配 | `--cpus=2` 限制最多使用 2 核 |
| cpuset | CPU 核心绑定 + NUMA | 将进程绑定到特定 CPU 核 |
| memory | 内存使用上限 | `--memory=512m` 限制最大 512MB |
| blkio/io | 块设备 IO 限制 | 限制磁盘读写速度 |
| pids | 进程数量上限 | 防止 fork 炸弹 |
| devices | 设备访问控制 | 允许/禁止访问特定设备 |
| freezer | 挂起/恢复进程组 | 暂停容器内所有进程 |

# 通过 systemd 使用 cgroup

systemd 是 cgroup 的主要管理入口，每个 service 自动分配一个 cgroup。

## 查看资源使用

```shell
# 实时查看各 service 的资源使用（类似 top）
systemd-cgtop

# 以树状显示 cgroup 层级
systemd-cgls
```

## 在 service unit 文件中限制资源

```ini
[Service]
# 内存上限：512MB
MemoryMax=512M
MemoryHigh=400M        # 软限制——超过后开始回收

# CPU 配额：最多用 1 个核的 50%
CPUQuota=50%

# IO 权重（1-10000，默认 100）
IOWeight=100

# 最大进程数
TasksMax=100

# 块设备 IO 限制
IOReadBandwidthMax=/dev/sda 10M
IOWriteBandwidthMax=/dev/sda 5M
```

## 运行时动态限制

```shell
# 不修改 unit 文件，直接限制运行中的 service
systemctl set-property nginx.service MemoryMax=512M
systemctl set-property nginx.service CPUQuota=30%
```

这些动态设置保存在 `/etc/systemd/system.control/` 下，重启后仍生效。

# 手动操作（通过 sysfs）

```shell
# 查看当前进程属于哪些 cgroup
cat /proc/self/cgroup

# 查看所有挂载的 cgroup 控制器
lssubsys -a

# cgroup v2 统一挂载点
ls /sys/fs/cgroup/
```

# 容器场景

## Docker

```shell
# 限制内存 512MB
docker run --memory=512m --memory-swap=1g nginx

# 限制 CPU 0.5 核
docker run --cpus=0.5 nginx

# 绑定 CPU 核心
docker run --cpuset-cpus=0-1 nginx
```

## Kubernetes

```yaml
resources:
  requests:          # 调度保证（不影响 cgroup 限制本身）
    memory: "256Mi"
    cpu: "250m"
  limits:            # cgroup 硬限制
    memory: "512Mi"
    cpu: "500m"
```

- `requests` 影响调度决策——节点必须有这么多可用资源才调度 Pod
- `limits` 是 cgroup 的实际限制——超过 memory limit 会被 OOMKilled

# 注意

- 超过 memory 硬限制 → 进程被 OOM Killer 杀死，容器表现为 OOMKilled
- CPU 限制不是绝对上限——未使用的 CPU 配额可能被其他 cgroup 借用（cpu 是可压缩资源）
- 调试 cgroup 问题时，`systemd-cgtop` 和 `systemctl status` 是最快的入口
- 容器中 `free` 命令显示的"可用内存"是宿主机的，不是容器的 cgroup 限制值——看容器内存用 `cat /sys/fs/cgroup/memory/memory.limit_in_bytes`（v1）或 `cat /sys/fs/cgroup/memory.max`（v2）

> 返回 [资源管理基础](./资源管理基础.md)
