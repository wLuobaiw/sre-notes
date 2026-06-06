
# 是什么

namespace（命名空间）是 Linux 内核提供的**资源隔离**机制。它让一组进程看到独立的系统资源视图——比如进程 A 看到的文件系统、网络接口、进程列表可以和进程 B 完全不同，即使它们在同一台物理机上运行。

namespace 是容器技术的核心基础——容器本质就是 namespace + cgroup 的组合。

# 8 种 namespace

| namespace | 引入版本 | 隔离的资源 | 容器中的作用 |
|-----------|---------|-----------|------------|
| Mount (mnt) | 2.4.19 | 文件系统挂载点 | 容器有自己独立的根文件系统 |
| UTS | 2.6.19 | 主机名和域名 | 每个容器有自己的 hostname |
| IPC | 2.6.19 | System V IPC、POSIX 消息队列 | 容器间不能通过 IPC 通信 |
| PID | 2.6.24 | 进程 ID 编号空间 | 容器内进程 PID=1，看不到宿主机进程 |
| Network (net) | 2.6.29 | 网络设备、IP、端口、路由表 | 容器有独立网卡、IP、端口空间 |
| User | 3.8 | 用户和组 ID | 容器内 root 映射到宿主机普通用户 |
| Cgroup | 4.6 | cgroup 文件系统视图 | 容器只能看到自己的 cgroup |
| Time | 5.6 | 系统时钟 | 容器可有独立时间（如仿真过去时间） |

# 核心 namespace 详解

## Mount namespace

**隔离文件系统挂载点**。这是容器有自己根文件系统的原因。结合 `chroot` / `pivot_root`，容器内的 `/` 可以是一个镜像层的挂载点，与宿主机的 `/` 完全隔离。

## PID namespace

**隔离进程编号**。容器内的进程从 PID=1 开始编号，看不到宿主机或其他容器的进程。这有两个关键效果：
- 容器内的 PID=1 进程（通常是容器的主进程）是容器内所有进程的父进程
- 容器内无法通过 PID 影响其他容器的进程

## Network namespace

**隔离网络栈**。每个容器有自己的：
- 网卡（虚拟的 veth pair）
- IP 地址
- 端口空间（不同容器可以同时监听 80 端口）
- 路由表、iptables 规则

Docker/K8s 用 veth pair 将容器的 net namespace 连接到宿主机网桥，实现网络互通。

## User namespace

**隔离用户和组 ID**。可以将容器内的 root（UID=0）映射到宿主机的普通用户（如 UID=1000）。这样即使容器被攻破，攻击者拿到的也是宿主机上的普通权限，不是真 root。

```shell
# 查看进程所属 namespace
ls -l /proc/self/ns/
```

输出示例：
```
lrwxrwxrwx 1 ... mnt -> mnt:[4026531840]
lrwxrwxrwx 1 ... net -> net:[4026531840]
lrwxrwxrwx 1 ... pid -> pid:[4026531836]
...
```

括号中的数字是 namespace ID——两个进程如果某个 namespace ID 相同，说明它们共享该 namespace。

# 与容器的关系

```
容器 = namespace（资源隔离） + cgroup（资源限制） + rootfs（文件系统镜像）
```

- **namespace** 解决"能看到什么"——让容器以为自己独占了操作系统
- **cgroup** 解决"能用多少"——限制容器能消耗的 CPU、内存、IO

# 查看与操作

```shell
# 查看所有 namespace 类型的列表
lsns

# 查看特定进程的 namespace
ls -l /proc/<PID>/ns/

# 进入运行中容器的 namespace 执行命令
nsenter -t <PID> --mount --uts --ipc --net --pid /bin/bash

# 创建新的 network namespace（手动模拟容器网络）
ip netns add myns
ip netns exec myns ip addr
ip netns exec myns ping 127.0.0.1
```

# 注意

- 默认情况下，所有系统进程共享宿主机同一套 namespace
- PID namespace 是树状结构——父 namespace 能看到子 namespace 中的进程（通过不同的 PID），子看不到父
- Network namespace 是容器网络的基础——K8s 中同一 Pod 的容器共享同一个 net namespace
- User namespace 在 Docker 中默认**不启用**（"rootless container" 才启用），这是安全方面需要注意的点

> 返回 [资源管理基础](./资源管理基础.md)
