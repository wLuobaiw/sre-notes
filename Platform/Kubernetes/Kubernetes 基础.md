# 一、Kubernetes 概述

Kubernetes 是一个可移植的、可扩展的开源平台，用于管理容器化的工作负载和服务，可促进声明式配置和自动化。Kubernetes 拥有一个庞大且快速增长的生态系统。

Kubernetes 这个名字源于希腊语，意为舵手或飞行员。k8s 这个缩写是因为 k 和 s 之间有八个字符。Google 在 2014 年开源了 Kubernetes 项目，建立在 Google 在大规模运行生产工作负载方面十几年的经验基础上。

## 1.1 优势

- 服务发现和负载均衡
- 存储编排（添加任何本地或云服务器）
- 自动部署和回滚
- 自动分配 CPU/内存资源
- 弹性伸缩
- 自我修复（需要时启动新容器）
- Secret（安全相关信息）和配置管理
- 大型规模的支持
  - 每个节点的 Pod 数量不超过 110
  - 节点数不超过 5000
  - Pod 总数不超过 150000
  - 容器总数不超过 300000
- 开源

## 1.2 架构全景

一个 Kubernetes 集群由控制平面（Control Plane）和若干工作节点（Worker Node）组成。控制平面管理集群状态，工作节点运行实际的应用负载。

**核心数据流**：所有操作通过 `kube-apiserver` 进行，集群的期望状态存储在 `etcd` 中，控制器不断将当前状态调整为期望状态。

---

# 二、组件索引

## 2.1 控制平面

控制平面组件为集群做出全局决策，负责调度、状态管理和事件响应。

- [kube-apiserver](控制平面/kube-apiserver.md) —— API 网关，控制平面的前端
- [etcd](控制平面/etcd.md) —— 分布式键值存储，集群数据的唯一真相源
- [kube-scheduler](控制平面/kube-scheduler.md) —— Pod 调度器，决定 Pod 在哪个节点运行
- [kube-controller-manager](控制平面/kube-controller-manager.md) —— 运行各类控制器，将当前状态调至期望状态

## 2.2 工作节点

工作节点组件在每个节点上运行，维护 Pod 并提供运行时环境。

- [kubelet](工作节点/kubelet.md) —— 节点代理，确保 Pod 中的容器正常运行
- [kube-proxy](工作节点/kube-proxy.md) —— 网络代理，维护节点网络规则
- [容器运行时](工作节点/容器运行时.md) —— 负责容器的执行和生命周期管理

## 2.3 工作负载资源

工作负载资源定义应用如何在集群中运行。

- [Pod](工作负载/Pod.md) —— 最小调度单元，包含一组紧密耦合的容器
- [Deployment 与 ReplicaSet](工作负载/Deployment与ReplicaSet.md) —— 无状态应用的声明式部署与副本管理
- [StatefulSet](工作负载/StatefulSet.md) —— 有状态应用的部署管理
- [DaemonSet](工作负载/DaemonSet.md) —— 确保每个节点运行一个 Pod 副本
- [Job 与 CronJob](工作负载/Job与CronJob.md) —— 一次性或定时任务
- [标签与选择器](工作负载/标签与选择器.md) —— 资源的组织与筛选机制

## 2.4 服务与网络

网络资源负责服务发现、流量路由与网络隔离。

- [Service](服务与网络/Service.md) —— Pod 的稳定访问入口，实现负载均衡
- [Ingress](服务与网络/Ingress.md) —— HTTP/HTTPS 路由规则，七层负载均衡
- [NetworkPolicy](服务与网络/NetworkPolicy.md) —— Pod 之间的网络访问控制（防火墙）

## 2.5 配置与存储

配置和存储资源管理应用所需的配置数据和持久化数据。

- [ConfigMap 与 Secret](配置与存储/ConfigMap与Secret.md) —— 配置与敏感信息的注入
- [Volume](配置与存储/Volume.md) —— Pod 级别的存储卷
- [PV / PVC / StorageClass](配置与存储/PV-PVC-StorageClass.md) —— 集群级持久化存储与动态供给

## 2.6 调度

调度机制决定 Pod 在哪个节点上运行。

- [亲和与反亲和](调度/亲和与反亲和.md) —— Pod 与节点的亲和/反亲和规则
- [污点与容忍](调度/污点与容忍.md) —— 节点排斥 Pod 的机制
- [资源与 QoS](调度/资源与QoS.md) —— 资源请求/限制与服务质量等级

## 2.7 安全与权限

安全资源控制集群访问和 Pod 运行安全。

- [RBAC](安全与权限/RBAC.md) —— 基于角色的权限控制
- [ServiceAccount](安全与权限/ServiceAccount.md) —— Pod 与 API Server 交互的身份凭证
- [SecurityContext 与准入控制](安全与权限/SecurityContext与准入控制.md) —— Pod 安全上下文与准入机制
