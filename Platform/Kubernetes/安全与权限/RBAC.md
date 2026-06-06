
# 是什么

RBAC（Role-Based Access Control，基于角色的权限控制）是 Kubernetes 的**鉴权**机制。它定义"谁对什么资源可以做什么操作"。

注意区分：TLS 证书/Token 是**认证**（你是谁），RBAC 是**鉴权**（你有权限吗）。两者是 [kube-apiserver](../控制平面/kube-apiserver.md) 请求处理链中独立的两关。

# 四个核心概念

RBAC 的本质是一个映射：**Subject → Role → 资源 → 操作**

## Role / ClusterRole（权限规则集合）

Role 定义一组权限：

```yaml
kind: Role
metadata:
  namespace: default            # Role 受命名空间限制
rules:
- apiGroups: [""]               # 核心 API 组（Pod、Service 等）
  resources: ["pods"]
  verbs: ["get", "list", "watch"]  # 允许的操作
```

ClusterRole 与 Role 相同，但作用范围是整个集群（不受命名空间限制），用于访问集群级资源（Node、Namespace、PV）或跨命名空间访问。

## RoleBinding / ClusterRoleBinding（绑定角色到主体）

把 Role 授予具体的主体：

```yaml
kind: RoleBinding
metadata:
  namespace: default
subjects:                        # 谁获得这个角色
- kind: User
  name: dev-alice
roleRef:                         # 授予哪个角色
  kind: Role
  name: pod-reader
```

## Subject（被授权的主体）

三种类型的主体可以被绑定：

| 类型 | 说明 |
|------|------|
| User | 通过 TLS 证书或 Token 认证的普通用户 |
| Group | 用户组，一组用户共享权限 |
| ServiceAccount | Pod 的身份凭证（详见 [ServiceAccount](ServiceAccount.md)） |

# 最小权限原则

用户默认没有任何权限。只有被 RoleBinding 显式授权后才能操作。

最佳实践：
- 为每种角色创建专用 ServiceAccount
- 授予完成工作所需的最小权限集
- 不在默认 ServiceAccount 上绑定额外权限
