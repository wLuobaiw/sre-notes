# Docker in Docker

## 两种"在容器里跑 Docker"

在 CI/CD 流水线或开发环境中，经常需要在容器内执行 `docker build` 或 `docker run`。实现这一点有两种截然不同的方式：

| | 真正的 DinD | 挂载 socket（DooD） |
|---|---|---|
| 原理 | 容器内运行独立的 Docker Daemon | 容器内的 Docker CLI 操控宿主机的 Docker Daemon |
| Docker Daemon | 容器内自己有一个 | 共享宿主机的，只有一个 |
| 隔离性 | 高（独立进程、独立存储） | 无隔离（共享所有镜像和容器） |
| 性能 | 有额外开销 | 几乎无开销 |
| 安全性 | 较高 | **极低**（等于宿主机 root） |

## 真正的 DinD（Docker in Docker）

### 原理

在容器内启动一个完整的 Docker Daemon 进程，`docker` 命令连接本地的 Daemon。

```
┌─────────────────────────────────┐
│ 容器                             │
│  ┌──────────┐    ┌────────────┐ │
│  │Docker CLI│───→│Docker Daemon│ │
│  └──────────┘    └─────┬──────┘ │
│                        │         │
│                  ┌─────▼──────┐ │
│                  │ 业务容器     │ │ ← 这些容器跑在外层容器内部
│                  └────────────┘ │
└─────────────────────────────────┘
```

外层的 DinD 容器需要 `--privileged` 权限。原因是 Docker Daemon 需要操作 cgroup、创建命名空间、管理网络和存储，这些操作需要完整的 Linux capabilities 和宿主机内核的访问权。

### 运行方式

```bash
docker run --privileged --name dind -d docker:dind
```

`docker:dind` 镜像内置了 Docker Daemon，启动后容器内可以独立跑 Docker。

### 典型场景

需要**完全隔离**的构建环境。例如：
- CI/CD 平台（如 GitLab CI），多个构建任务需要独立 Docker 环境，互不污染
- 本地测试 Docker 新版本，不影响宿主机 Docker

### 核心问题：存储驱动冲突

Docker 默认使用 overlay2 存储驱动。DinD 嵌套时，内层的 Docker 也会尝试使用 overlay2，但 overlay2 不支持多层嵌套。解决方案：
- 外层 DinD 容器挂载宿主机目录作为内层 Docker 的数据目录
- 内层 Docker 使用 vfs 或 fuse-overlayfs 驱动（性能较差）

## DooD（Docker outside of Docker，挂载 socket）

### 原理

不启动新的 Docker Daemon，而是把宿主机的 Docker Unix Socket（`/var/run/docker.sock`）挂进容器：

```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock docker
```

容器内的 Docker CLI 通过这个 socket 直接操控**宿主机的 Docker Daemon**：

```
┌─────────┐
│ 容器     │
│ ┌──────┐│
│ │Docker││──socket──→ 宿主机的 Docker Daemon → 容器实际在宿主机上创建
│ │ CLI  ││
│ └──────┘│
└─────────┘
```

你在容器里执行 `docker run nginx`，这个 Nginx 容器实际上是**在宿主机上创建的**，和 DinD 容器平级，而不是嵌套在里面。

### 典型场景

极轻量的 Docker 操作场景：
- 简单的 CI 镜像构建（如 Jenkins 需要在 slave 容器里 `docker build`）
- 不想维护特权容器的场景

### 致命缺陷：安全问题

`docker.sock` 是 Docker Daemon 的 API 入口，拥有对它的一切权限等于拥有宿主机 root。在容器里：

```bash
docker run -v /:/host alpine cat /host/etc/shadow   # 读取宿主机任意文件
docker run -v /:/host busybox rm -rf /host/*        # 删除宿主机文件
```

**任何能访问 docker.sock 的人 = 宿主机的 root。**

此外，DooD 还有两个工程问题：
- **并发构建同名镜像冲突**：多个容器同时 `docker build -t app .`，共享同一个 Daemon 的镜像缓存，可能构建出错误的内容
- **容器残留**：容器崩溃后，它在宿主机上创建的容器不会被自动清理

## 选择指南

| 需求场景 | 推荐方案 | 原因 |
|---------|---------|------|
| CI/CD 构建并对环境有严格隔离要求 | DinD（`--privileged`） | 独立 Daemon，不污染其他任务 |
| 临时使用 docker CLI，只在容器里跑简单命令 | 挂载 socket（DooD） | 简单轻量 |
| 生产环境 | **都不推荐** | 用 Kaniko、BuildKit 等无 Daemon 构建工具 |
| 需要确保最小权限，安全要求高 | **都不用，改用无 Daemon 方案** | 避免 `--privileged` 和 socket 暴露 |

## 无 Daemon 构建工具（安全替代方案）

Daemon-less 构建工具不需要 Docker Daemon，以普通用户权限运行，适合安全敏感场景：

- **Kaniko**：Google 开源，在容器内构建镜像。不需要 Docker Daemon，不需要特权模式。适合 Kubernetes 集群中的 CI/CD
- **BuildKit**：Docker 官方的新一代构建引擎，可以以非 root 用户运行
- **img**：由 Genuine Tools 开发，无 Daemon 的镜像构建工具

这些工具的核心思路：不需要 Daemon、不需要特权，直接读写文件系统来完成镜像构建。
