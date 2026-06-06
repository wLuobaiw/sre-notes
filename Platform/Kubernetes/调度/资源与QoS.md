
# 资源请求与限制

每个容器可以声明资源需求，Kubernetes 基于此做调度决策和运行时管控：

```yaml
resources:
  requests:        # 最低保障，调度器用
    cpu: 100m      # 0.1 核
    memory: 128Mi
  limits:          # 硬上限，运行时强制执行
    cpu: 500m      # 0.5 核
    memory: 256Mi
```

## requests 与 limits 的区别

| | requests | limits |
|---|---|---|
| 作用 | 调度依据（节点必须满足） | 运行时限制（不能超过） |
| CPU 超出 | 可以被调度到满足 requests 的节点 | 超限时被内核 throttling（限速），不杀容器 |
| 内存超出 | 同上 | **超限时容器被 OOMKilled（杀掉）** |

关键区别：CPU 是"可压缩资源"，超限只是变慢。内存是"不可压缩资源"，超限直接杀容器。

## 在调度中的作用

- **预选阶段**：节点剩余资源必须满足 Pod 所有容器的 requests 之和。不满足 → 淘汰。调度流程详见 [kube-scheduler](../控制平面/kube-scheduler.md)。
- **优选阶段**：剩余资源较多的节点获得更高分数

# QoS 等级

根据 requests 和 limits 的配置方式，Pod 被自动划分到三个 QoS 等级。QoS 决定了节点资源紧张时 Pod 的**驱逐优先级**。

## Guaranteed（最后被驱逐）

条件：每个容器都同时设置了 CPU 和内存的 requests 和 limits，且 **request == limit**。

这是最稳定的等级，适合核心数据库等关键服务。

## Burstable（中等优先级）

条件：至少一个容器设了 request（CPU 或内存），但不满足 Guaranteed 条件。即：不是 BestEffort 也不是 Guaranteed 的 Pod。

大多数普通业务 Pod 属于此等级。

## BestEffort（最先被驱逐）

条件：没有任何容器设置 requests 或 limits。

节点内存不足时，BestEffort 的 Pod **最先被杀**。适合非关键、可随时重建的临时任务。

# 驱逐顺序

当节点内存不足时，[kubelet](../工作节点/kubelet.md) 按以下优先级驱逐 Pod：

```
BestEffort（最先被杀）→ Burstable（使用超出 request 的先杀）→ Guaranteed（最后被杀）
```

同一 QoS 等级内，**内存使用超过 request 越多的 Pod 越早被驱逐**。

# 实践建议

- 关键服务必须设 request=limit 达到 Guaranteed 等级
- 所有业务 Pod 至少设 request，避免落入 BestEffort
- limits 可以不等于 requests（Burstable），利用集群空闲资源的同时保证最低保障
- 内存 limits 不宜设过高过松 —— 内存泄漏的 Pod 在 limits 很高的情况下会耗尽节点内存、触发连锁驱逐
