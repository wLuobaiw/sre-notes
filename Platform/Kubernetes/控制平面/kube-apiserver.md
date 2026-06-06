
# 是什么

kube-apiserver 是 Kubernetes 控制平面的**唯一入口**。集群中所有组件（kubelet、scheduler、controller-manager）以及外部用户（kubectl）都通过 apiserver 与集群交互。它是集群的 API 网关。

apiserver 提供 RESTful HTTP API。执行 `kubectl apply -f pod.yaml` 时，kubectl 做的事就是：
1. 读取 YAML 文件
2. 序列化成 JSON
3. 发送 HTTP POST 请求给 apiserver

kubectl 只是一个命令行客户端，apiserver 才是真正处理和持久化请求的组件。

# 为什么能水平扩展

apiserver 是控制平面中唯一的**无状态组件**。它不存储任何数据 —— 所有数据都在 etcd 中。因此可以部署多个 apiserver 实例，前面挂一个负载均衡器，任意实例都能处理请求。

控制平面其他组件都有状态或单实例约束：
- etcd：有状态（Raft 共识）
- scheduler：单实例（同一时刻只有一个在做调度决策）
- controller-manager：单实例（通过选举机制保证）

# 请求处理三阶段

每个请求进入 apiserver 后，按固定顺序经过三道关卡：

## 1. 认证 —— "你是谁？"

确认请求是否来自合法身份。支持多种认证方式：
- **客户端证书**（双向 TLS）：kubelet、kubectl 等持有证书证明身份
- **Bearer Token**：ServiceAccount 的认证方式（详见 [ServiceAccount](../安全与权限/ServiceAccount.md)）
- **OIDC**：对接外部身份系统

认证失败 → 直接返回 401 Unauthorized。认证通过 → 进入鉴权。

## 2. 鉴权 —— "你能干这个吗？"

确认身份后，检查是否有权限执行当前操作。Kubernetes 中默认使用 RBAC（详见 [RBAC](../安全与权限/RBAC.md)）。

鉴权失败 → 返回 403 Forbidden。鉴权通过 → 进入准入控制。

## 3. 准入控制 —— "这个请求合法吗？"

最后一关，做合理性校验和自动填充。准入控制器是一串插件链，分两类：
- **变更准入（Mutating）**：可以**修改**请求内容。典型例子是 Istio 自动注入 Sidecar 容器
- **验证准入（Validating）**：只做**校验**，不能修改。典型例子是限制镜像标签不能为 `latest`

常见准入插件：
- **NamespaceLifecycle**：禁止在正在删除的命名空间里创建资源
- **ResourceQuota**：命名空间资源配额检查

三关的通俗类比：
- 认证 = 门卫查工牌（你是谁）
- 鉴权 = 门禁系统（你能进这个房间吗）
- 准入 = 进房间后的合规检查（你带的东西没问题吧）

# 请求处理是并行还是串行？

apiserver 内部**并行**处理请求，可以同时处理数百个请求。

但最终持久化到 etcd 时，etcd 的 Raft 共识在写入层面是**串行化**的 —— Leader 一条一条按序写入日志。

# 读取优化：Watch 缓存

apiserver 并不是每个读请求都访问 etcd。启动时从 etcd 拉全量数据到内存，然后通过 etcd 的 Watch 机制持续接收增量更新。绝大多数 `kubectl get` 请求直接命中内存缓存，无需访问 etcd。

这使得 apiserver 可以承担极高的读取吞吐量，同时保护 etcd 不被大量读请求压垮。
