
# 是什么

NAT（Network Address Translation，网络地址转换）是在网络包经过路由器或防火墙时，**修改包的源地址或目标地址**的技术。它不是一种"通信方式"，而是对 IP 包头的改写操作。

在 Kubernetes 中，kube-proxy 利用 NAT 将发往 Service ClusterIP 的流量重定向到真实 Pod IP。

# 为什么需要 NAT

核心矛盾：**私有 IP 地址在公网上不可路由。**

但 NAT 的应用场景远不止"上网"。任何需要"用一个 IP 代表另一组 IP"的场景都会用到它。Kubernetes Service 就是典型 —— ClusterIP 是虚拟 IP，必须用 DNAT 翻译成真实 Pod IP。

# 四种 NAT 类型

## SNAT（Source NAT）—— 改源地址

**把包的来源地址改掉。**

最经典的场景：家庭路由器。你家里 3 台设备都是 192.168.1.x（私网 IP），访问百度时：

```
原始包：源 192.168.1.5 → 目标 百度公网IP
SNAT后：源 你的公网IP → 目标 百度公网IP
```

百度回复时发回你的公网 IP，路由器根据连接追踪记录再翻回 192.168.1.5。

Kubernetes 中用在哪：Pod 访问集群外服务时，如果 Pod IP 在外部网络不可路由，需要在节点出口做 SNAT（MASQUERADE）。

## DNAT（Destination NAT）—— 改目标地址

**把包的目的地址改掉。**

负载均衡器的基本原理：客户端以为自己连的是 `10.96.0.1:80`（Service ClusterIP），实际上被 DNAT 改为 `10.244.2.8:8080`（后端 Pod）：

```
原始包：源 PodA → 目标 10.96.0.1:80（ClusterIP）
DNAT后：源 PodA → 目标 10.244.2.8:8080（Pod IP）
```

这是 Kubernetes Service 运作的核心机制。

## NAPT/PAT（端口转换）

纯 IP 地址一对一转换太浪费。NAPT 利用**端口号**做多对一映射：

```
192.168.1.5:34567 → 公网IP:34567 → 百度
192.168.1.6:45678 → 公网IP:45678 → 百度
```

两设备共用同一个公网 IP，通过端口区分。绝大多数家用路由器用的都是 NAPT，而非纯 NAT。

Kubernetes 中 NodePort 也是 NAPT 的变体 —— 每个 Service 在端口范围（30000-32767）里占一个唯一端口。

## 双向 NAT（Full Cone / Symmetric 等）

包出去时改源地址，进来时改目标地址，两个方向都做转换。复杂场景（如两个私网通过公网互联）中用到。Kubernetes 场景中较少直接涉及，不展开。

# NAT 的关键依赖：连接追踪

NAT 靠连接追踪表（conntrack）来记录"这个包对应的是哪个原始连接"。

只有第一个包需要查规则表、决定怎么做 NAT。后续包属于同一连接，直接查 conntrack 表（O(1) 哈希查找），跳过规则匹配。回包也靠 conntrack 自动做反向转换。

代价：conntrack 表有大小上限。大规模集群中高频短连接可能撑爆表，导致新连接丢包。这就是 ipvs 模式要缓解的问题之一。

# 三种网络模式对比

## Host 模式

容器与宿主机共享网络命名空间。`localhost` 就是宿主机的 `localhost`。没有 NAT、没有隔离、性能最佳。

## Bridge 模式

Docker 默认模式。宿主机上创建一个虚拟网桥（docker0），每个容器通过 veth pair 接入网桥。容器出站（访问外网）需要 SNAT —— 把容器私网 IP 改成宿主机 IP。

## NAT 模式

泛指通过 NAT 实现跨网络通信的模式，Bridge 模式就是 NAT 模式的一种实现。Kubernetes 中 Service 的 ClusterIP DNAT 也是 NAT 的一种应用。

> 返回 [网络管理基础](./网络管理基础.md)
