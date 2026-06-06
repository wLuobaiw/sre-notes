
# 是什么

ServiceAccount 是 Pod 在集群中的**身份凭证**。Pod 通过它与 [kube-apiserver](../控制平面/kube-apiserver.md) 通信，类似于"Pod 的身份证"。

每个命名空间有一个默认 ServiceAccount（名为 `default`），若不指定，Pod 自动使用它。

# 为什么需要

集群中有大量 Pod 需要跟 apiserver 交互（查自己的 IP、监控工具拉节点列表等）。Pod 不是"人"，无法持有 TLS 客户端证书。ServiceAccount 为 Pod 提供一种替代身份证明。

# 工作原理

1. 创建 ServiceAccount 时，Kubernetes 自动生成一个 JWT Token
2. Pod 引用此 SA 时，kubelet 自动将 Token 以文件形式挂载到容器内
3. 容器内应用读取 Token 文件，通过 HTTP 请求头 `Authorization: Bearer <token>` 与 apiserver 通信

挂载是**自动的**，每个 Pod 内默认存在：

```
/var/run/secrets/kubernetes.io/serviceaccount/token      ← JWT Token
/var/run/secrets/kubernetes.io/serviceaccount/ca.crt      ← apiserver CA 证书
/var/run/secrets/kubernetes.io/serviceaccount/namespace   ← 所在命名空间
```

# SA + RBAC

ServiceAccount 本身只证明身份，不携带任何权限。权限由 RBAC 授予：

```
1. 创建 ServiceAccount "monitoring-sa"
2. 创建 Role：允许对所有命名空间的 Pod 执行 get、list
3. 创建 RoleBinding：将 Role 绑给 ServiceAccount "monitoring-sa"
4. Pod 引用 SA "monitoring-sa" → Pod 拥有读 Pod 列表的权限
（RBAC 的完整说明详见 [RBAC](RBAC.md)）
```

# 最佳实践

永远不要让业务 Pod 使用默认 SA。为每种权限需求创建专用 SA，通过 RBAC 授予最小权限。默认 SA 不应绑定额外权限。
