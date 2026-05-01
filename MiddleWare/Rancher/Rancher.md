# Rancher 是什么

Rancher 是一个容器管理平台，可以帮助：

- 轻松部署和管理 Kubernetes 集群
- 统一管理多个 Kubernetes 集群
- 提供用户权限控制、应用商店、监控告警等附加功能

## 核心组件

**Rancher Server**：管理中心，负责集群管理、用户认证、权限控制等

**Kubernetes 集群**：Rancher 管理的对象，用于运行容器化应用

**项目（Project）**：用于在集群内隔离资源和应用，类似命名空间的扩展

**应用商店（Catalog）**：提供预置的应用模板（如MySQL、Redis等），一键部署