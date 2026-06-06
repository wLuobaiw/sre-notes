
# 是什么

kube-controller-manager 是所有 Kubernetes 内置控制器的运行载体。它的核心职责是：**不断将集群当前状态调整为期望状态**。

它不是一个控制器，而是一个**控制器集合**。所有控制器编译在同一个二进制文件中，以独立线程并发运行。

# 控制器的工作原理

所有控制器共享同一套工作模式：**Informer + WorkQueue + Worker**。

## Informer（监听器）

每个控制器通过 apiserver 的 Watch 机制监听自己关心的资源变化。

例如 Deployment 控制器同时监听三种资源：
- Deployment 对象（知道用户期望多少副本）
- ReplicaSet 对象（知道当前 ReplicaSet 的状态）
- Pod 对象（知道最终有多少 Pod 在运行）

变化事件（创建、更新、删除）触发后续处理。

## WorkQueue（工作队列）

Informer 收到变化后，不直接处理，而是把对象塞入工作队列。

为什么需要队列：
- 防止大量变化同时到达导致过载，队列做缓冲
- 同一个对象短时间多次变化，队列可以去重，最终只处理一次
- 控制器重启后只需重新 Watch，队列自动重建

## Worker（工作者）

从队列取出对象，读取其当前状态和期望状态，对比后执行操作使状态趋近。

例如 ReplicaSet 控制器：
1. 取出一个 ReplicaSet 对象
2. 数一下当前有几个 Pod 在运行
3. 对比期望的 `replicas` 字段
4. 少了 → 创建 Pod；多了 → 删除 Pod

循环往复，永不停止。

# 为什么放在一个进程里

将几十个控制器放在一个进程里运行有明确的工程优势：

- **共享 Watch 连接**：所有控制器复用同一个到 apiserver 的 Watch 通道，否则每个独立服务都要发起 Watch，apiserver 负载暴涨
- **共享内存缓存**：多个控制器关心同一类资源（如 Pod），可以读取同一份内存缓存
- **部署简单**：启动一个进程即启动全部控制器

# 内置控制器一览

| 控制器 | 职责 |
|--------|------|
| [Deployment](../工作负载/Deployment与ReplicaSet.md) Controller | 管理 Deployment → ReplicaSet → Pod 的版本变更 |
| ReplicaSet Controller | 维持指定数量的 Pod 副本 |
| [StatefulSet](../工作负载/StatefulSet.md) Controller | 管理有状态应用的有序部署和稳定标识 |
| [DaemonSet](../工作负载/DaemonSet.md) Controller | 确保每个节点运行一个 Pod |
| [Job](../工作负载/Job与CronJob.md) Controller | 管理一次性任务的 Pod 直到成功完成 |
| Node Controller | 监控节点健康状态，失联后驱逐 Pod |
| Service Controller | 为 Service 分配 ClusterIP，管理负载均衡 |
| EndpointSlice Controller | 维护 Service 到 Pod 的映射关系 |
| ServiceAccount Controller | 为新命名空间自动创建默认 ServiceAccount |
| Namespace Controller | 管理命名空间生命周期 |

每个控制器的具体逻辑不同，但底层全部使用 Informer + WorkQueue + Worker 的统一模式。

# 高可用

多个 controller-manager 实例通过选举机制竞争 Leader。只有 Leader 实例执行控制循环，其余处于待命状态。Leader 宕机后自动重新选举。
