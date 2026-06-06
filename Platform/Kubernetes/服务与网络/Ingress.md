
# 是什么

Ingress 是 Kubernetes 中管理**七层（HTTP/HTTPS）路由规则**的资源。它根据请求的域名和路径，将流量转发到不同的后端 Service。

与 [Service](Service.md) 的区别：Service 工作在 L4（TCP/UDP），仅按 IP:端口转发，不理解 HTTP 内容。Ingress 工作在 L7，能够解析 HTTP 请求中的 Host 头和 URL 路径，实现更精细的路由。

# 为什么需要 Ingress

一个集群中可能有多个 HTTP 服务，都占用 80/443 端口：

```
api.example.com/users   → 用户服务
api.example.com/orders  → 订单服务
web.example.com/        → 前端页面
```

Service 做不到这种级别的路由 —— 它不管 HTTP 请求里写的是什么域名和路径。Ingress 解决了这个问题：**一个入口，按域名和路径分发到不同 Service。**

# 不是内置组件

Kubernetes 只定义了 Ingress 资源规范，不内置 Ingress Controller。设计思路与 CRI/CNI/CSI（详见 [kubelet](../工作节点/kubelet.md)）一致：**核心只定义接口规范，具体实现由社区和厂商提供。**

常用的 Ingress Controller：nginx-ingress、Traefik、HAProxy、AWS ALB Ingress Controller。

# 工作原理

```
外网请求 → Ingress Controller (Nginx Pod) → 读取 Ingress 规则 → 动态更新 nginx.conf → 转发到 Service → Pod
```

1. 用户创建 Ingress 资源（YAML），声明路由规则
2. Ingress Controller 持续监控 Ingress 资源的变化
3. Controller 根据 Ingress 规则动态生成和更新自己的配置（如 nginx.conf）
4. 流量到达 Controller 后，按规则转发到对应的 Service

用户不写 nginx.conf —— Ingress YAML 是声明式规范，Controller 负责翻译成实际配置。

# 核心能力

## 基于域名的路由

多个域名共用一个入口 IP，按 Host 头分发到不同后端。

## 基于路径的路由

同一域名下按 URL 路径分发。路径匹配规则：

- `Prefix`：按路径段前缀匹配。如 `/users` 匹配 `/users`、`/users/123`，**不**匹配 `/user_login`（不同的路径段）
- `Exact`：精确匹配，大小写敏感

## TLS/HTTPS 终结

Ingress 在入口处处理 HTTPS 加解密，后端 Service 使用 HTTP 通信。TLS 证书存在 Secret 中（详见 [Secret](../配置与存储/ConfigMap与Secret.md)），Ingress 通过名称引用：

```
用户 → HTTPS → Ingress Controller（解密）→ HTTP → Service → Pod
```

证书的来源：手动用 openssl 生成（开发测试）、cert-manager 自动对接 Let's Encrypt 申请续期（生产推荐）、云厂商证书服务。

## 注解（Annotations）

除了核心路由规则，不同 Ingress Controller 通过注解暴露额外的配置能力。以 nginx-ingress 为例：

- URL 重写、强制 HTTPS 跳转
- 上传大小限制、连接超时
- 限流、跨域（CORS）、IP 白名单

注解相当于把 nginx.conf 里的配置项声明式地写在 Ingress YAML 中，无需手动编辑 Controller 的配置文件。

# 多 Ingress 共享一个 Controller

集群中通常只部署一套 Ingress Controller（如一个 Nginx Pod），但可以有几十个 Ingress 资源。Controller 将所有 Ingress 规则合并为一份完整的配置。

# 与 Service 的关系

Ingress 不能直接转发到 Pod —— 它必须通过 Service。链路始终是：

```
Ingress → [Service](Service.md) → Pod
```

Service 负责 Pod 发现和负载均衡（L4），Ingress 负责在此之上做 HTTP 路由（L7），各司其职。
