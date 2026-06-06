
# 控制器模型

Kubernetes 的控制器遵循统一的**控制循环**模式：

```
观察当前状态 → 对比期望状态 → 执行操作使其趋近期望
```

Deployment 和 ReplicaSet 是最基础的工作负载控制器，它们的分工如下：

- **ReplicaSet**：确保指定数量的 Pod 副本同时运行。多了删除，少了创建。
- **Deployment**：在 ReplicaSet 之上管理版本变更 —— 滚动更新、回滚、暂停。

实际使用中**不直接创建 ReplicaSet**，而是通过 Deployment 间接管理。Deployment → ReplicaSet → Pod 是一条三层链路。

# ReplicaSet 的职责

ReplicaSet 只做一件事：**维持 Pod 副本数**。它通过标签选择器（详见 [标签与选择器](标签与选择器.md)）匹配 Pod，若当前 Pod 数少于期望数，按模板创建新的；若多于期望数，删除多余的。

ReplicaSet 不关心 Pod 里跑的是什么 —— 它只管数量对不对。

## RS vs RC

ReplicationController（RC）是旧版控制器，ReplicaSet（RS）是新版替代。区别仅在标签选择器的表达能力：

- RC：只支持等值匹配（`env = prod`）
- RS：支持集合匹配（`env in (prod, staging)`、`tier notin (frontend)`）

现在没有理由使用 RC，所有场景都用 RS（通过 Deployment）。

# Deployment 的职责

Deployment 解决的核心问题：**如何安全地把应用从版本 A 切换到版本 B**。

每次修改 Deployment 的 Pod 模板（如更换镜像版本），Deployment 会创建新的 ReplicaSet，然后按指定策略逐步替换。

## 滚动更新

默认策略。流程如下：

1. 创建新 ReplicaSet，replicas 从 0 开始
2. 新 RS 逐步增加 Pod，旧 RS 逐步减少 Pod
3. 新旧 RS 的 Pod 数之和 = 期望副本数 + `maxSurge`
4. 新 RS 达到期望副本数，旧 RS 缩至 0

两个关键参数控制更新节奏：
- **maxSurge**：更新期间最多多出几个 Pod（默认 25%）。值越大更新越快，但峰值资源消耗越多。
- **maxUnavailable**：更新期间最多有几个 Pod 不可用（默认 25%）。设为 0 可保证更新过程中服务不中断。

## 回滚

旧 ReplicaSet 缩到 0 后**不会立刻删除**（默认保留 10 个历史版本，由 `revisionHistoryLimit` 控制）。

所以 `kubectl rollout undo` 几乎瞬间完成 —— 它只是把目标 ReplicaSet 的 replicas 加回去，不用重新拉镜像、重建 Pod。本质就是"调转新旧 ReplicaSet 的副本数"。

# Deployment 与 [DaemonSet](DaemonSet.md) 的场景区分

Deployment 的 replicas 是**绝对值**（如 `replicas: 5`），与集群节点数无关。10 个节点的集群扩容到 20 个节点，Deployment 还是 5 个 Pod。

Deployment 用于业务应用（Web 服务、API），副本数由流量需求决定，不由节点数决定。
