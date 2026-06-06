
# 是什么

StatefulSet 是用于管理**有状态应用**的工作负载控制器。与 Deployment 的核心区别在于：StatefulSet 的每个 Pod 拥有**不可替代的身份**。

# 有状态 vs 无状态

判断标准不是"有没有存储数据"，而是 **Pod 之间是否可以互换**：

| | 无状态（Deployment） | 有状态（StatefulSet） |
|---|---|---|
| Pod 可互换 | 是，任意 Pod 可处理任意请求 | 否，每个 Pod 有独特角色 |
| Pod 标识 | 随机 hash（如 `nginx-7d8f4c9b6-x2k9p`） | 固定序号（如 `mysql-0`、`mysql-1`） |
| 启动/关闭顺序 | 无顺序，同时启停 | 严格有序：从 0 到 N 启动，从 N 到 0 关闭 |
| 网络标识 | 无独立 DNS | 每个 Pod 有稳定的 DNS 名称 |
| 存储 | 共享 PVC 或无存储 | 每个 Pod 可绑定独立的 PVC |

典型例子：MySQL 主从集群。`mysql-0` 是主库，`mysql-1` 和 `mysql-2` 是从库。如果 `mysql-0` 挂了重建，新 Pod 仍然叫 `mysql-0`，仍然挂载同一个 PVC（数据不丢），仍然作为主库。

# 三个核心能力

## 稳定网络标识

StatefulSet 需要一个**无头 Service**（Headless Service，`clusterIP: None`）来为每个 Pod 生成固定的 DNS 记录：

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
# 例如: mysql-0.mysql.default.svc.cluster.local
```

Deployment 的 Pod 没有独立的 DNS，只能通过 Service 的 ClusterIP 访问，请求随机落到某个 Pod。

## 有序扩缩

- **扩容**：0 → 1 成功后 → 2 成功后 → 3。前一个没 Ready，后一个不会启动。
- **缩容**：3 → 2 → 1 → 0。从最后一个开始删除。
- **滚动更新**：按倒序更新（N-1 → N-2 → ... → 0）。

这一点对有状态应用至关重要：主库必须先启动，从库才能连接主库同步数据。乱序启动会导致集群无法正确组建。

## 独立持久化存储

StatefulSet 通过 `volumeClaimTemplates` 为每个 Pod 动态创建独立的 PVC（详见 [PV-PVC-StorageClass](../配置与存储/PV-PVC-StorageClass.md)）。Pod 被删除重建后，新的 Pod 自动重新挂载原来的 PVC，数据完整保留。

而 Deployment 的所有 Pod 共享同一个 Pod 模板，要么共享同一个 PVC（有并发冲突风险），要么不持久化。

# 使用场景

- 数据库集群（MySQL、PostgreSQL、MongoDB）
- 消息队列（Kafka、RabbitMQ）
- 分布式存储（Elasticsearch、Ceph）
- 任何需要稳定网络标识或有序部署的应用
