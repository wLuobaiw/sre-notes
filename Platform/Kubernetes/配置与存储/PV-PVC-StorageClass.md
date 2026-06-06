
# 核心概念

Kubernetes 将存储管理拆分为三个独立的抽象，对应三种角色：

| 资源 | 角色 | 做什么 |
|------|------|--------|
| PersistentVolume（PV） | 管理员 | 提供实际的存储资源 |
| PersistentVolumeClaim（PVC） | 开发者 | 声明对存储的需求 |
| StorageClass | 管理员 | 定义存储类型，实现自动创建 PV |

类比：PV 是停车场的车位，PVC 是停车申请单，StorageClass 是"有人申请车位时自动划一个出来"的自动分配系统。

# 为什么需要 PV/PVC

[Pod](../工作负载/Pod.md) 会重建、会漂移到其他节点。如果把存储信息写在 Pod 里，每个 Pod 都绑死了存储实现，无法统一管理和复用。

PV/PVC 将**存储的提供**和**存储的消费**解耦：管理员关心物理存储，开发者只声明"我要多大、什么访问模式"。

# 静态供给 vs 动态供给

## 静态供给

管理员先在存储系统上准备好实际空间，再手动创建 PV：

```yaml
kind: PersistentVolume
spec:
  capacity:
    storage: 100Gi            # 实际可用容量
  accessModes:
  - ReadWriteOnce
  nfs:                        # 指向 NFS 服务器
    server: 10.0.0.5
    path: /exports/data
```

同一个物理存储设备（如一台 NFS 服务器）上可以创建多个 PV，只要 `path` 不冲突。1TB 的 NFS 可以拆成 10 个 100G 的 PV 分别指向不同子目录。

问题：每次有新的存储需求，管理员就要手动操作，无法自动化。

## 动态供给（StorageClass）

管理员只配置一次 StorageClass（声明"用这个 NFS 服务器"），之后开发者创建 PVC 时，PV 自动生成：

```yaml
kind: StorageClass
metadata:
  name: nfs-sc
provisioner: nfs.csi.k8s.io   # 使用 NFS CSI 驱动
parameters:
  server: 10.0.0.5
  path: /exports
```

之后任何人创建的 PVC 引用 `storageClassName: nfs-sc`，系统自动创建 PV 并绑定。这是生产环境的标准做法。

# PVC 的请求机制

```yaml
kind: PersistentVolumeClaim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi           # 请求 10G
    # 没有 limits —— PV 的容量就是硬上限
  storageClassName: nfs-sc    # 动态供给时指定 StorageClass
```

Kubernetes 找到满足条件的 PV 后，PVC 与之**一对一绑定**。规则：
- 一个 PV 只能绑定一个 PVC
- PVC 不能跨多个 PV 拼碎片
- 绑定后整个 PV 属于该 PVC（100G PV 被 10G PVC 绑了，剩余 90G 浪费 —— 这正是动态供给要解决的问题）
- PVC 请求的容量必须 ≤ PV 容量

如果没有满足条件的 PV 且配置了 StorageClass，动态供给会创建刚好够用的 PV。如果没有 StorageClass，PVC 一直 Pending。

# 访问模式（Access Modes）

| 模式 | 缩写 | 含义 |
|------|------|------|
| ReadWriteOnce | RWO | 单个节点读写 |
| ReadOnlyMany | ROX | 多节点只读 |
| ReadWriteMany | RWX | 多节点读写 |
| ReadWriteOncePod | RWOP | 单个 Pod 读写（K8s 1.22+） |

底层存储的物理限制无法被 Kubernetes 完全抽象掉：

| 存储类型 | RWO | ROX | RWX |
|---------|-----|-----|-----|
| hostPath | ✅ | ✅ | ❌ |
| NFS | ✅ | ✅ | ✅ |
| AWS EBS（云盘） | ✅ | ❌ | ❌ |
| CephFS | ✅ | ✅ | ✅ |

**云盘本质是块存储，只能挂载到单节点，天然不支持 RWX。** 如果 PVC 申请了 RWX 但底层 StorageClass 对应的是云盘，Pod 永远起不来。声明式抽象消除了操作差异，但消除不了物理限制。

# 回收策略

PV 绑定的 PVC 被删除后，PV 的处理方式：

| 策略 | 行为 |
|------|------|
| Retain | 保留数据，需手动清理后 PV 才能再次使用 |
| Delete | 同时删除底层存储（动态供给默认此策略） |
| Recycle | 执行 `rm -rf` 清空数据（已废弃） |
