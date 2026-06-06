
# SecurityContext

## 是什么

SecurityContext 是 Pod 和容器级别的**运行安全配置**，定义容器以什么身份、什么权限运行。将 Docker `docker run --user=1000 --cap-drop=ALL` 等安全参数声明式地写入 Pod spec。

## 两个层级

```yaml
spec:
  securityContext:                      # Pod 级别，对所有容器生效
    runAsUser: 1000                     # 以 UID=1000 运行（非 root）
    fsGroup: 2000                       # 挂载卷的文件所属组
  containers:
  - name: app
    securityContext:                    # 容器级别，仅对此容器生效
      runAsNonRoot: true                # 强制禁止 root
      readOnlyRootFilesystem: true      # 根文件系统只读
      allowPrivilegeEscalation: false   # 禁止 setuid/setgid 提权
      capabilities:
        drop: ["ALL"]                   # 去掉所有 Linux capabilities
        add: ["NET_BIND_SERVICE"]       # 只加必要的

# 容器级别会覆盖 Pod 级别的同名字段
```

## 关键配置项

| 配置 | 作用 | 安全意义 |
|------|------|---------|
| `runAsNonRoot: true` | 禁止以 root 运行 | 容器逃逸后攻击者只是普通用户 |
| `readOnlyRootFilesystem: true` | 根文件系统只读 | 防止攻击者写入恶意二进制 |
| `allowPrivilegeEscalation: false` | 禁止 setuid/setgid | 防止普通用户通过 setuid 二进制提权成 root |
| `capabilities.drop: ["ALL"]` | 去掉所有 Linux 能力 | 需要的能力按需逐项加 |
| `privileged: false` | 非特权模式 | privileged=true 时容器几乎拥有宿主机所有权限 |

## 安全原则

不是"全禁用最安全"，而是"只给必要的"。例如 Nginx 绑定 80 端口需要 `NET_BIND_SERVICE` capability，全 drop 会导致启动失败 —— 正确做法是 drop ALL，只 add 需要的。

# Pod Security Standards

## 是什么

Kubernetes 内置的三个**安全合规等级**，在准入阶段自动检查 Pod 配置是否合规。这是 [kube-apiserver](../控制平面/kube-apiserver.md) 请求处理链中**准入控制**的最后一道防线。

## 三个等级

| 等级 | 限制程度 | 典型场景 |
|------|---------|---------|
| **Privileged** | 无限制 | CI/CD 构建、需要特权的系统组件 |
| **Baseline** | 禁止已知危险配置 | 一般业务应用，默认推荐 |
| **Restricted** | 严格遵循最佳实践 | 安全敏感应用（金融、医疗） |

Restricted 级别的强制要求包括：
- `runAsNonRoot: true`（禁止 root）
- `allowPrivilegeEscalation: false`（禁止提权）
- 禁止 hostPath、hostNetwork、hostPID
- capabilities 只能加 `NET_BIND_SERVICE`

违规 → Pod 创建被拒。

## 启用方式

通过命名空间标签指定等级：

```yaml
kind: Namespace
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

# 三层安全体系的关系

```
认证（TLS 证书 / Token）→ 鉴权（[RBAC](RBAC.md)）→ 准入控制（Pod Security + SecurityContext）
   你是谁？             你有权限吗？          你做的事合规且安全吗？
```

SecurityContext 是主动声明的安全配置，Pod Security Standards 是集群级的强制基线，两者配合构成完整的运行时安全。
