
# 是什么

Volume 不是独立的 Kubernetes 资源（没有 `kind: Volume`），而是 Pod spec 中的一个字段。它解决容器文件系统的两个根本问题：

1. 容器重启后文件丢失（容器文件系统是临时的）
2. Pod 内多个容器需要共享文件

对应 Docker 用户理解的场景：Volume 就是 `docker run -v` 的声明式表达。

# Volume 与 VolumeMount

```yaml
kind: Pod
spec:
  volumes:                        # Pod 级别声明：这个 Pod 有哪些存储
  - name: data
    emptyDir: {}
  containers:
  - volumeMounts:                 # 容器级别挂载：这块存储挂到容器的哪个路径
    - name: data
      mountPath: /app/data        # 容器内路径
```

Volume 在 Pod 级别声明（可以被多个容器共享），VolumeMount 在容器级别挂载（指定挂载路径）。同一个 Volume 可以挂给 Pod 中多个容器。

# 常用 Volume 类型

## emptyDir

Pod 创建时自动分配节点上的临时目录。**Pod 删除后数据永久丢失。** 容器重启数据不丢（因为 Pod 还在）。

用途：Pod 内容器间共享临时数据（如 Sidecar 写日志、主容器读日志）。

## hostPath

**挂载宿主机上的目录或文件**到容器中。对应 Docker `-v /host/path:/container/path`。

```yaml
volumes:
- name: host-data
  hostPath:
    path: /data/app
    type: DirectoryOrCreate
```

业务 Pod 极少使用 hostPath。原因：Pod 重新调度到其他节点后，新节点的 hostPath 没有原来的数据。主要用于 DaemonSet 部署的节点级服务（日志采集、监控 agent）。

## ConfigMap / Secret

将 ConfigMap 或 Secret 中的键值对挂载为容器中的文件。详见 [ConfigMap与Secret](ConfigMap与Secret.md)。

## PersistentVolumeClaim（PVC）

将持久化存储挂载到 Pod。这是生产中最常用的方式，详见 [PV-PVC-StorageClass](PV-PVC-StorageClass.md)。

# Volume 的生命周期

Volume 的生命周期与 **Pod 绑定**。Pod 存在期间 Volume 存在，Pod 删除后：
- emptyDir 被清理
- hostPath 数据保留（数据在宿主机上）
- PVC 引用的数据由 PV 的回收策略决定（保留或删除）
