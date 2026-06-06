
# 是什么

kubelet 是运行在每个节点上的**节点代理**，负责将 Pod 的声明式配置转化为实际运行的容器。它通过 Watch 机制从 apiserver 获取分配给自己节点的 Pod，然后调用容器运行时启动容器，并持续监控其健康状态。

kubelet 不管理不是由 Kubernetes 创建的容器。

# Pod 生命周期管理

kubelet 对每个 Pod 执行一个严格的、有序的初始化流程：

## 阶段一：创建 Pod 沙箱（Sandbox）

kubelet 通过 CRI 调用 `RunPodSandbox`，让容器运行时启动 Pause 容器。此时 CNI 插件介入，执行网络配置：
1. 为 Pod 分配 IP 地址
2. 创建虚拟网络接口（veth pair）
3. 将网络接口接入 Pod 的网络命名空间

沙箱创建完成后，Pod 拥有了 IP 和可用的网络栈。

## 阶段二：Init 容器

如果有 Init 容器，kubelet 按声明顺序逐个启动。规则：
- 前一个 Init 容器成功退出（exit 0）后，才启动下一个
- 任一 Init 容器失败，kubelet 重启整个 Init 流程（从头开始）
- 没有 Init 容器则跳过此阶段

## 阶段三：启动业务容器

所有 Init 容器成功后，kubelet 同时启动 Pod 中所有的业务容器。

如果有 Startup Probe（详见 [Pod 探针](../工作负载/Pod.md)），kubelet 在 Startup Probe 成功之前**阻塞 Liveness 和 Readiness 检查** —— 这对启动慢的应用至关重要，避免被 Liveness 误杀导致无限重启。

## 阶段四：健康检查循环

容器启动后，kubelet 进入持续监控模式：

- **Liveness Probe 失败**：kubelet 杀掉容器并重新启动（按 `restartPolicy` 处理）
- **Readiness Probe 失败**：kubelet 通知 apiserver 将 Pod 从 Service 的 Endpoint（详见 [Service](../服务与网络/Service.md)）中移除。不杀容器，等恢复后重新加入
- **Startup Probe 失败**：同 Liveness，杀容器重启

所有这些探针检查都由 kubelet 本地执行，不经过 apiserver。

# CRI 交互

kubelet 本身不直接操作容器 —— 它通过 CRI（容器运行时接口）的 gRPC 协议与容器运行时通信。

核心 gRPC 方法：
- `RunPodSandbox` —— 创建沙箱（Pause 容器）
- `CreateContainer` —— 创建容器（不启动）
- `StartContainer` —— 启动容器
- `StopContainer` —— 停止容器
- `RemoveContainer` —— 删除容器

kubelet 不关心 CRI 接口后面是 containerd 还是 CRI-O，只要实现标准 gRPC 协议即可。

# 对接的四种接口插件

kubelet 通过四种标准接口接入外部能力，均在节点级别运行：

| 接口 | 全称 | 职责 | 常见实现 |
|------|------|------|---------|
| CRI | 容器运行时接口 | 容器生命周期管理 | containerd（详见 [容器运行时](容器运行时.md)）、CRI-O |
| CNI | 容器网络接口 | Pod 网络配置（IP 分配、网络连接） | Calico、Flannel、Cilium |
| CSI | 容器存储接口 | 存储卷挂载 | 云厂商 CSI 驱动、NFS CSI |
| Device Plugin | 设备插件 | GPU、FPGA 等硬件管理 | NVIDIA GPU Plugin |

四种接口都遵循"Kubernetes 定义规范，社区实现插件"的设计模式。
