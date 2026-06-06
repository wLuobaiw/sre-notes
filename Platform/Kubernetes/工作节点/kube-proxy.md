
# 是什么

kube-proxy 是运行在每个节点上的**网络代理**，负责实现 [Service](../服务与网络/Service.md) 的网络转发规则。它不中转流量，而是通过操作 Linux 内核的网络机制（iptables 或 ipvs），让发往 Service ClusterIP 的包被自动重定向到后端 Pod。

# 核心问题：为什么需要 kube-proxy

Pod IP 会变，Service 提供稳定的 ClusterIP。但 ClusterIP 是**虚拟 IP** —— 没有真实的网络接口持有它。包的目的地址是 ClusterIP 时，内核无法直接路由。

kube-proxy 做的事：在内核中写入规则，让"发往虚拟 ClusterIP"的包被自动改写目标地址（DNAT）为真实 Pod IP。

涉及 NAT 的详细原理见 [NAT基础](../../Linux/NAT基础.md)。

# 三种代理模式

## userspace 模式 —— 历史方案

kube-proxy 进程自身作为代理，在用户态中转流量：

```
客户端 → ClusterIP → iptables 规则 → kube-proxy 用户态进程 → Pod
```

问题：用户态与内核态频繁切换，性能极差。已淘汰。

## iptables 模式 —— 完全内核态

kube-proxy 不中转流量，只向 iptables 写入 NAT 规则。规则告诉内核：

> 目标地址是 ClusterIP:Port 的包，把目标地址随机 DNAT 为后端某个 Pod IP

数据流在内核态完成，kube-proxy 不碰数据包。

优点：流量在内核态转发，不经过用户态。性能远优于 userspace。

局限：
- 规则以链表组织，匹配是 O(n) 遍历。规则数量多时性能线性下降
- 更新规则时全量替换，大规模场景中极为缓慢
- 后端选择只支持随机，不支持加权或最少连接等调度算法
- 依赖 conntrack 做连接追踪，大规模短连接场景下 conntrack 表容易满

关于 iptables 的基础知识见 [iptables基础](../../Linux/iptables基础.md)。

## ipvs 模式 —— 内核级四层负载均衡

IPVS 是 Linux 内核内置的四层负载均衡器，即 LVS（Linux Virtual Server）项目的核心。

kube-proxy 通过 IPVS 创建虚拟服务（ClusterIP）和真实服务器（Pod IP），由 IPVS 直接做转发。

| 对比 | iptables | ipvs |
|------|----------|------|
| 规则数据结构 | 链表 O(n) | 哈希表 O(1) |
| 更新模式 | 全量替换 | 增量更新 |
| 调度算法 | 仅随机 | rr / wrr / lc / wlc / sh / dh 共 8 种 |
| 大规模性能 | 规则数万级时严重退化 | 几乎不受规模影响 |
| 连接追踪 | 依赖 conntrack | IPVS 自身管理 |

**生产规模超过 1000 个 Service 或 10000 个 Pod 时，ipvs 是必然选择。**

# 总结

kube-proxy 在所有模式下做同一件事：把虚拟 ClusterIP 翻译成真实 Pod IP。区别只在用谁来做（用户态进程 / iptables 规则 / IPVS），以及效率和可扩展性的不同。
