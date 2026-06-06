
# 是什么

Service 是 Kubernetes 中将一组 Pod 暴露为**单一、稳定的网络入口**的资源。它解决的核心问题：**Pod 的 IP 是临时的（Pod 重启后 IP 变化），但调用方需要一个不变的地址。**

Service 通过[标签选择器](../工作负载/标签与选择器.md)匹配一组 Pod，自动维护选择器与 Pod 的映射关系。调用方只关心 Service 的 ClusterIP 和端口，不需要知道后端 Pod 是否变化。

# 为什么不能直接用 Pod IP

Pod 被删除重建后 IP 会变。如果用 Pod IP 直接通信，每次 Pod 重启都需要更新所有调用方。Service 提供一个不变的门牌号 —— ClusterIP 在 Service 生命周期内不变，无论后端 Pod 如何变化。

# Service 如何工作

1. 用户创建 Service，指定标签选择器
2. EndpointSlice Controller 持续监控，将匹配的 Pod IP 列表写入 EndpointSlice 对象
3. **kube-proxy**（详见 [kube-proxy](../工作节点/kube-proxy.md)）在每个节点上订阅 Service 和 EndpointSlice 的变化
4. kube-proxy 将 ClusterIP 到 Pod IP 的映射写入 iptables/ipvs 规则
5. 发往 ClusterIP 的流量被内核自动 DNAT 到后端 Pod

# 四种类型

## ClusterIP（默认）

仅在集群内部可访问，分配一个集群内部 IP（如 `10.96.x.x`）。是最常用的类型，用于微服务间的内部调用。

```
Service ClusterIP: 10.96.0.1:80 → Pod1 (10.244.1.5:8080) / Pod2 (10.244.2.8:8080)
```

## NodePort

在每个节点的 IP 上开放一个静态端口（默认范围 30000-32767），集群外部可通过 `<任意节点IP>:<NodePort>` 访问。

```
外部请求 → Node1:30080 → iptables DNAT → 后端 Pod
```

NodePort 建立在 ClusterIP 之上 —— 创建 NodePort Service 时会自动创建一个 ClusterIP 和一个 NodePort。

## LoadBalancer

在 NodePort 基础上，云厂商自动创建一个外部负载均衡器（如 AWS ELB、GCP LB），分配一个公网或私网 IP。

```
外部请求 → 负载均衡器IP:80 → 各节点 NodePort → 后端 Pod
```

是生产环境对外暴露服务的标准方式，但需要云厂商支持。

## ExternalName

不创建代理或转发，只是返回一个 DNS CNAME 记录，将集群内部的服务名映射到外部 DNS 名称。用于将集群外部服务（如托管数据库）映射为集群内的 Service 名。

# 会话亲和（Session Affinity）

默认情况下，每次请求随机分发到不同 Pod。设置 `sessionAffinity: ClientIP` 后，同一客户端 IP 的请求始终发往同一个 Pod。适用于需要粘性会话的场景。
