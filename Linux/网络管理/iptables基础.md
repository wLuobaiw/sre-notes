
# 是什么

iptables 是 Linux 内核提供的**网络包过滤和 NAT 框架**。它在网络协议栈中设置"规则"，每个经过的网络包都要逐一匹配规则，决定其命运（放行、丢弃、改地址、转发到别处）。

在 Kubernetes 中，kube-proxy 利用 iptables 规则实现 Service 的负载均衡 —— 将发往 Service ClusterIP 的流量随机转发到后端 Pod。

# 五链四表

iptables 的规则组织在两个维度上：**表（table）** 定义"做什么"，**链（chain）** 定义"在哪个时机做"。

## 五条链（Chains）—— 数据包的五个"检查点"

数据包在经过 Linux 协议栈时会经过五个检查点（链）：

```
本机发出的包：OUTPUT → POSTROUTING → 网络
本机收到的包：网络 → PREROUTING → INPUT
转发的包：   网络 → PREROUTING → FORWARD → POSTROUTING → 网络
```

| 链 | 触发时机 |
|----|---------|
| PREROUTING | 包进入网卡后，路由决策**之前** |
| INPUT | 路由判定目标是**本机**后 |
| FORWARD | 路由判定目标是**其他机器**后 |
| OUTPUT | 本机进程发出包，路由决策**之前** |
| POSTROUTING | 包即将离开网卡，路由决策**之后** |

## 四张表（Tables）—— 四种"处理动作"

| 表 | 功能 | 常见用途 |
|----|------|---------|
| raw | 标记包**不进行连接追踪** | 性能优化，跳过追踪 |
| mangle | 修改包的 TTL/TOS 等字段 | 特殊网络策略 |
| nat | **网络地址转换** | SNAT（改源地址）、DNAT（改目标地址） |
| filter | **过滤**（放行或丢弃） | 防火墙规则 |

两张最常用的表：**nat**（Kubernetes 做 Service 负载均衡用这个）和 **filter**（NetworkPolicy 用这个）。

## 表和链的对应关系

不是每张表的规则在每条链上都会执行。实际生效的组合：

| 表 ↓ / 链 → | PREROUTING | INPUT | FORWARD | OUTPUT | POSTROUTING |
|-------------|-----------|-------|---------|--------|-------------|
| raw | ✅ | | | ✅ | |
| mangle | ✅ | ✅ | ✅ | ✅ | ✅ |
| nat | ✅ | | | ✅ | ✅ |
| filter | | ✅ | ✅ | ✅ | |

# 一条规则的三个要素

每条 iptables 规则包含：

```
匹配条件 + 处理动作
```

**匹配条件**：什么样的包？来源 IP、目标 IP、端口、协议（TCP/UDP）、进入的网卡……

**处理动作（target）**：
- **ACCEPT**：放行
- **DROP**：丢弃（不发任何通知，对端超时）
- **REJECT**：拒绝（回复 ICMP 告知"端口不可达"）
- **DNAT**：改目标地址（目标地址转换）
- **SNAT**：改来源地址（源地址转换）
- **MASQUERADE**：动态 SNAT（来源 IP 设为出口网卡的 IP）

# kube-proxy 为什么需要 NAT

kube-proxy 用 iptables 的 NAT 表来实现 Service：

1. 用户访问 Service 的 ClusterIP:Port
2. iptables 在 PREROUTING 链用 **DNAT** 把目标地址从 ClusterIP 改为某个后端 Pod 的 IP
3. 包被路由到 Pod 所在节点
4. 如果 Pod 在其他节点，还需要在 POSTROUTING 链做 SNAT/MASQUERADE，确保回包能正确路由回来

整个过程对内核对网络包的处理就是"改一下地址然后继续转发"，效率很高。

# 连接追踪（conntrack）

iptables 基于连接追踪机制来判断包的"来源"。每个 TCP/UDP 连接在 conntrack 表中有一条记录。

当回包到达时，iptables 自动识别它属于某个已建立的连接，执行反向 NAT 转换 —— 这不需要写额外的规则。

conntrack 表有大小限制。大规模集群中可能因 conntrack 表满导致丢包。ipvs 模式解决的就是这个问题。

> 返回 [网络管理基础](./网络管理基础.md)
