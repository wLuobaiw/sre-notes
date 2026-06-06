
# 是什么

NetworkPolicy 是 Pod 之间的**网络防火墙**。它控制哪些流量可以进出指定的 Pod，在 L3（IP）和 L4（端口/协议）层面做访问控制。

Kubernetes 默认的网络模型是"所有 Pod 之间可以自由通信"。NetworkPolicy 在此基础上建立白名单机制。

# 核心概念：白名单模式

没有 NetworkPolicy 时，所有 Pod 间的流量都放行。一旦创建了针对某组 Pod 的 NetworkPolicy，该组 Pod 的流量变为白名单模式：**只有被 NetworkPolicy 明确允许的流量才放行，其余一律拒绝。**

常见的防御策略：
1. 先创建一条对所有 Pod 的"拒绝所有入站"规则
2. 再为需要对外通信的 Pod 逐条创建"允许特定流量"的规则

# 规则的三要素

| 要素 | 对应字段 | 回答的问题 |
|------|---------|-----------|
| 保护对象 | `podSelector`（标签选择器的用法详见 [标签与选择器](../工作负载/标签与选择器.md)） | 这条规则作用在**哪些 Pod**上？ |
| 访问来源/目标 | `from` / `to` | **谁**可以访问我 / 我可以访问**谁**？ |
| 端口与协议 | `ports` | 允许**哪些端口和协议**？ |

来源可以用三种方式指定：
- **podSelector**：按标签选择特定 Pod
- **namespaceSelector**：选择整个命名空间下的所有 Pod
- **ipBlock**：指定 CIDR 网段（如 `10.0.0.0/8`，用于限制集群外部访问）

三种方式可以组合（AND 逻辑），如"来自 monitoring 命名空间且标签为 app=prometheus 的 Pod"。

# 两个方向

- **Ingress（入站）**：谁可以访问我？—— 控制进入 Pod 的流量
- **Egress（出站）**：我可以访问谁？—— 控制从 Pod 发出的流量，较少使用，典型场景如禁止 Pod 访问外网

# 底层实现

NetworkPolicy 的执行者不是 Kubernetes 自己，而是 **CNI 网络插件**。Kubernetes 只负责将 NetworkPolicy 对象存入 etcd，CNI 插件读取规则并翻译为 iptables 规则（filter 表）或 eBPF 程序，在内核层面直接过滤数据包。

> **关键注意**：如果集群使用的 CNI 插件不支持 NetworkPolicy（如 Flannel 默认配置），NetworkPolicy 资源会被 API Server 接受，但实际不生效。 支持 NetworkPolicy 的 CNI 包括 Calico、Cilium、Weave Net。

# 与 Ingress 的区别

两者名字中的"Ingress"容易混淆，但完全不同：

| | NetworkPolicy | Ingress |
|---|---|---|
| 层级 | L3/L4（IP + 端口） | L7（HTTP 域名 + 路径） |
| 功能 | 防火墙：允许/拒绝流量 | HTTP 路由：按规则转发请求 |
| 执行者 | CNI 插件 | Ingress Controller |
| 是否内置 | 是（资源定义内置） | 仅定义资源规范，Controller 需独立安装 |
