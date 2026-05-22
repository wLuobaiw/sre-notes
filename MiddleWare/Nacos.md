# Nacos 是什么

Nacos 是一个动态服务发现、配置管理和服务管理平台
它让微服务之间能自动找到彼此，并能让所有服务的配置实时统一更新

## 背景和起源

在 Nacos 出现之前，微服务架构面临两大核心痛点

- **服务发现靠硬编码**：A 服务要调用 B 服务，早期只能把 B 的 IP 和端口卸载配置文件里。一旦 B 服务扩缩容或者宕机，A 就得手动改配置重启，极易出错且完全无法跟上云原生的动态变化
- **配置分散且无法热更新**：每个服务各自维护一堆配置文件（如数据库连接串、开关参数）。如果要改一个参数，需要登陆到每台机器修改文件，然后滚动重启所有实例——耗时、易遗漏、且容易引发崩溃。

同时，业界以及出现了像 Netflix Eureka（服务发现）和 Spring Cloud Config（配置管理）这样的组件，但他们各自为战，数据无法互通，运维需要同时维护两套集群，而且 Eureka 在高可用存在局限性（如自我保护机制容易引发混乱）

**Nacos 解决了什么**

Nacos 将 “服务发现” 和 “动态配置” 合二为一，提供一站式的解决方案。它在技术栈中的核心定位是：**微服务世界的 “通讯录+管理中心”**

---

# 核心概念

Nacos 的核心概念围绕两条主线展开：**服务发现**和**配置管理**。

## 服务发现侧

- **服务（Service）**：一个逻辑上的业务单元，比如"用户服务"。相当于 K8s 中 Service 的名字，不指具体机器，只代表一个服务身份。
- **实例（Instance）**：服务的一个具体运行副本。比如用户服务部署了 3 台机器，就有 3 个实例，每个实例 = IP + 端口。对标 K8s 中的 Endpoint/Pod IP。实例列表动态变化——扩容时新实例上线、宕机时旧实例被自动摘除。
- **健康检查（Health Check）**：Nacos 定期探测每个实例是否存活。分两种模式：
  - 客户端主动上报心跳（临时实例，类似 lease 机制）
  - 服务端主动探测（持久化实例）

## 配置管理侧

- **配置（Configuration）**：一个键值对，key 是 Data ID（全局唯一标识），value 是配置内容，格式可以是 properties、yaml、json 等。
- **Data ID**：命名格式通常为 `{应用名}-{环境}.{后缀}`，如 `order-service-dev.yaml`。

## 隔离机制

- **Namespace（命名空间）**：最顶层的硬隔离单元。不同 namespace 之间的服务和配置完全不可见，通常用于隔离环境（dev / test / prod）。不同 namespace 下可以注册同名服务互不冲突。
- **Group（分组）**：namespace 内部的软分组，默认 group 为 `DEFAULT_GROUP`。同 namespace 内不同 group 的服务默认仍可互相发现，group 更大作用是方便在控制台里分类查阅。

```
Namespace: prod
├── Group: trade
│   ├── Service: order-service
│   │   ├── Instance: 192.168.1.10:8080
│   │   └── Instance: 192.168.1.11:8080
│   └── Service: pay-service
│       └── Instance: 192.168.1.20:8080
├── Group: DEFAULT_GROUP
│   └── Config: order-service-dev.yaml → {db.url: ...}
└── ...
Namespace: dev
└── ...
```

## 与 K8s Service 的区别

- **K8s Service**：基础设施层（网络层）服务发现，通过 kube-proxy + iptables 转发请求到 Pod IP。只在 K8s 集群内部有效，不关心应用层是否真正健康（端口通 ≠ 应用就绪）。
- **Nacos**：应用层服务发现 + 配置管理。服务实例主动注册、持续健康检查、跨环境（K8s/虚拟机/云函数）统一注册发现。Nacos 不中转业务流量，只维护"通讯录"。

类比：K8s Service 像 TCP 层的路由表，Nacos 像应用层的通讯录 + 公告板。

---

# 原理及架构

## 一句话原理

```
服务实例 → [向 Nacos Server 注册 + 定期续约] → Nacos Server 维护注册表 → 调用方拉取/订阅注册表 → 直接调用目标实例
```

Nacos **不代理流量**，只存"通讯录"，服务间通信是点对点的。

## 核心组件

| 组件 | 职责 |
|:---|:---|
| **Nacos Server** | 接收注册、维护注册表、健康检查、配置存储与下发 |
| **注册表（Service Registry）** | 内存中的 Map，key=服务名，value=实例列表，变化时通知订阅者 |
| **Provider（服务提供者）** | 启动时向 Server 注册，定期发心跳续约 |
| **Consumer（服务消费者）** | 从 Server 拉取服务列表，本地缓存，订阅变更通知 |

## 配置管理原理

```
开发者修改配置 → Nacos Server 持久化到内置 Derby/外置 MySQL → 通知订阅该配置的客户端 → 客户端更新本地 Bean/变量（无需重启）
```

客户端通过长轮询（long polling）监听配置变更，实现秒级推送。

## 架构简图

```
                    ┌──────────────┐
                    │  Nacos Server │  ← 注册表 + 配置存储
                    └──┬───┬───┬──┘
         注册+心跳     │   │   │  拉取注册表+订阅变更
         ┌────────────┘   │   └────────────┐
         ▼                │                ▼
  ┌──────────┐            │         ┌──────────┐
  │ Provider │            │         │ Consumer │
  │ 用户服务  │            │         │ 订单服务  │
  └──────────┘            │         └──────────┘
                          │
              ┌───────────┴───────────┐
              │                      │
              ▼                      ▼
      ┌─────────────┐       ┌─────────────┐
      │ 实例A:8080  │       │  配置变更通知  │
      │ 实例B:8081  │       │  (长轮询)     │
      └─────────────┘       └─────────────┘
```

---

# 部署搭建

## 单机部署（Docker，快速验证）

```bash
# 拉取镜像（必须指定版本号，不用 latest）
docker pull nacos/nacos-server:v2.5.1

# 启动单机 Nacos
docker run -d \
  --name nacos-standalone \
  -p 8848:8848 \
  -p 9848:9848 \
  -e MODE=standalone \
  nacos/nacos-server:v2.5.1
```

关键参数：
- `-p 8848:8848`：HTTP API 和控制台主端口
- `-p 9848:9848`：gRPC 通信端口（Nacos 2.x 新增，**忘了映射会导致服务注册失败**）
- `-e MODE=standalone`：单机模式，不加默认尝试集群模式会报错

**验证**：浏览器访问 `http://<IP>:8848/nacos`，默认账号密码 `nacos/nacos`。

## 集群部署

### 单机的三个致命问题

1. **单点故障**：Nacos Server 一挂，全系统服务发现和配置管理瘫痪
2. **注册表无冗余**：内置 Derby 数据库不可靠，集群不能共用同一 Derby
3. **无法横向扩展**：所有服务都来一台 Nacos 查地址，请求量上来撑不住

### 集群架构

```
                    ┌─────────────┐
                    │  Nginx/CLB  │  ← 对外统一入口
                    └──┬───┬───┬──┘
                       │   │   │
          ┌────────────┼───┼───┼────────────┐
          ▼            ▼   │   ▼            ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │ Nacos-1 │  │ Nacos-2 │  │ Nacos-3 │
    └────┬─────┘  └────┬─────┘  └────┬─────┘
         │              │              │
         └──────────────┼──────────────┘
                        ▼
              ┌─────────────────┐
              │   MySQL 集群     │  ← 存储配置 & 服务注册数据
              └─────────────────┘
```

关键点：
- Nacos 节点通过 Raft 协议选主，Leader 宕机自动切换
- 所有 Nacos 节点共享同一个外部 MySQL，不再用内置 Derby
- 前端用 Nginx/CLB 做负载均衡

### Docker Compose 集群配置

```yaml
version: "3.8"
services:
  mysql:
    image: mysql:8.0
    container_name: nacos-mysql
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: nacos_config
    volumes:
      - ./mysql:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3306:3306"

  nacos1:
    image: nacos/nacos-server:v2.5.1
    container_name: nacos1
    depends_on:
      - mysql
    environment:
      - MODE=cluster
      - NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=mysql
      - MYSQL_SERVICE_PORT=3306
      - MYSQL_SERVICE_DB_NAME=nacos_config
      - MYSQL_SERVICE_USER=root
      - MYSQL_SERVICE_PASSWORD=root123
    ports:
      - "8848:8848"
      - "9848:9848"
    volumes:
      - ./nacos1:/home/nacos/data

  nacos2:
    image: nacos/nacos-server:v2.5.1
    container_name: nacos2
    depends_on:
      - mysql
    environment:
      - MODE=cluster
      - NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=mysql
      - MYSQL_SERVICE_PORT=3306
      - MYSQL_SERVICE_DB_NAME=nacos_config
      - MYSQL_SERVICE_USER=root
      - MYSQL_SERVICE_PASSWORD=root123
    ports:
      - "8849:8848"
      - "9849:9848"
    volumes:
      - ./nacos2:/home/nacos/data

  nacos3:
    image: nacos/nacos-server:v2.5.1
    container_name: nacos3
    depends_on:
      - mysql
    environment:
      - MODE=cluster
      - NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=mysql
      - MYSQL_SERVICE_PORT=3306
      - MYSQL_SERVICE_DB_NAME=nacos_config
      - MYSQL_SERVICE_USER=root
      - MYSQL_SERVICE_PASSWORD=root123
    ports:
      - "8850:8848"
      - "9850:9848"
    volumes:
      - ./nacos3:/home/nacos/data
```

初始化 MySQL 表：

```bash
wget https://raw.githubusercontent.com/alibaba/nacos/2.5.1/distribution/conf/mysql-schema.sql -O init.sql
```

启动：

```bash
docker compose up -d
```

验证集群：控制台 → 集群管理 → 节点列表，应看到 3 个节点都是 UP。

### 部署最容易踩的三个坑

1. **忘了映射 9848 端口**：Nacos 2.x gRPC 端口。只映射 8848 会导致客户端向 Server 拉到的 gRPC 地址不正确，注册失败。每个节点两个端口都要映射，且宿主机端口不能冲突（9848/9849/9850）。
2. **`NACOS_SERVERS` 里填宿主机端口**：三个容器在同一 Docker 网络内互相通信，应写容器内端口 8848，而不是宿主机的 8848/8849/8850。
3. **MySQL 未就绪**：`depends_on` 只保证启动顺序，不保证 MySQL health check 通过。线上建议加 healthcheck。

---

# 常用命令与最小实践

Nacos 通过 HTTP REST API 交互，所有操作可用 `curl` 完成。官方没有独立 CLI 工具。

## 服务注册与发现

```bash
# 注册服务实例
curl -X POST 'http://localhost:8848/nacos/v1/ns/instance' \
  -d 'serviceName=order-service' \
  -d 'ip=192.168.1.10' \
  -d 'port=8080'

# 查询服务实例列表
curl -X GET 'http://localhost:8848/nacos/v1/ns/instance/list' \
  -d 'serviceName=order-service'

# 发送心跳（续约）
curl -X PUT 'http://localhost:8848/nacos/v1/ns/instance/beat' \
  -d 'serviceName=order-service' \
  -d 'ip=192.168.1.10' \
  -d 'port=8080' \
  -d 'beat={"ip":"192.168.1.10","port":8080}'
```

## 配置管理

```bash
# 发布配置
curl -X POST 'http://localhost:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service-dev.yaml' \
  -d 'group=DEFAULT_GROUP' \
  -d 'content=db.url: jdbc:mysql://localhost:3306/orders
db.pool.size: 20'

# 获取配置
curl -X GET 'http://localhost:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service-dev.yaml' \
  -d 'group=DEFAULT_GROUP'
```

## 最小化 Demo 流程

```bash
# 1. 确认 Nacos 存活
curl http://localhost:8848/nacos/v1/console/health/readiness
# → ok

# 2. 注册实例
curl -X POST 'http://localhost:8848/nacos/v1/ns/instance' \
  -d 'serviceName=demo-service' \
  -d 'ip=127.0.0.1' \
  -d 'port=8080'
# → ok

# 3. 查实例
curl -X GET 'http://localhost:8848/nacos/v1/ns/instance/list' \
  -d 'serviceName=demo-service'

# 4. 发布配置
curl -X POST 'http://localhost:8848/nacos/v1/cs/configs' \
  -d 'dataId=hello-config' \
  -d 'group=DEFAULT_GROUP' \
  -d 'content=hello: world'
# → true

# 5. 读配置
curl -X GET 'http://localhost:8848/nacos/v1/cs/configs' \
  -d 'dataId=hello-config' \
  -d 'group=DEFAULT_GROUP'
# → hello: world

# 6. 清理
curl -X DELETE 'http://localhost:8848/nacos/v1/ns/instance' \
  -d 'serviceName=demo-service' \
  -d 'ip=127.0.0.1' \
  -d 'port=8080'
curl -X DELETE 'http://localhost:8848/nacos/v1/cs/configs' \
  -d 'dataId=hello-config' \
  -d 'group=DEFAULT_GROUP'
```

> Java 项目中用 `nacos-client` SDK（Maven 依赖）对 HTTP API 做了封装：`namingService.registerInstance(...)` / `configService.getConfig(...)`。其他语言同理。

---

# 关键配置

## 服务发现侧

| 配置项 | 作用 | 默认值 | 推荐值 | 改错会导致 |
|:---|:---|:---|:---|:---|
| `nacos.naming.clean.empty-service.interval` | 清理空服务间隔（ms） | 60000 | 不变 | 空服务残留 |
| `nacos.naming.heartbeat.interval` | 心跳间隔（秒） | 5 | 不变 | 网络抖动时实例被误踢 |
| `nacos.naming.instance.expire.timeout` | 实例过期超时（秒） | 15 | 网络差可调至 30 | 太小：实例频繁被误踢；太大：宕机后仍显示健康 |
| `protect.threshold` | 保护阈值（0-1） | 0 | 0.85 | 设为 1 后不踢实例，网络抖动时注册表错乱 |

**`protect.threshold` 详解**：当健康实例占比低于该阈值时，Nacos 停止踢出任何实例，宁可返回不健康的实例也不让调用方看到空列表。这是牺牲准确性换取可用性（雪崩保护）。

## 配置管理侧

| 配置项 | 作用 | 默认值 | 推荐值 |
|:---|:---|:---|:---|
| `db.num` | 数据库连接池大小 | 10 | 默认够用 |
| `db.url.0` | MySQL 连接地址 | 无 | 集群模式必填 |

## 常见误区

1. **`protect.threshold` 设为 1 以为更安全**：实际效果是宕机实例永远不会被摘除，调用方持续失败。
2. **忘记显式指定 `nacos.inetutils.ip-address`**：多网卡场景下 Nacos 可能自动选了错误的 IP，必须手动指定。

---

# 常用监控指标

Nacos 不是流量中转件，不要看吞吐量/延迟。核心关注这两个：

## 1. 服务实例变化率

反应服务拓扑是否稳定。短时间内大量实例上下线 → 批量发布或故障。正常运行时接近 0。

查看：控制台 → 服务管理 → 服务列表 → 实例上下线时间。

## 2. CPU 使用率 & GC 频率

Nacos 注册表全部在**内存**中，实例越多，堆内存占用越大。GC 频繁会导致 Nacos 响应变慢甚至短暂不可用。

GC（Garbage Collection，垃圾回收）：JVM 自动清理不再使用的对象。运行时会"Stop-The-World"暂停响应。实例从 100 涨到 10000 时：
- 注册表中存活对象暴增，挤占老年代空间
- 心跳更新持续产生新对象（不是原地修改，而是创建新对象替换旧对象），加快年轻代填满
- Full GC 扫描的对象变多，暂停时间从毫秒级拉长到秒级

Nacos 官方建议：实例超 10 万级别时需调大 JVM 堆内存。

---

# 常见问题

**Q：Nacos 和 K8s Service 有什么区别？**
K8s Service 是网络层服务发现，只在 K8s 集群内有效，不检查应用健康状态。Nacos 是应用层服务发现 + 配置管理，跨环境，主动健康检查，不中转业务流量。

**Q：Nacos 流量是经过 Server 中转的吗？**
不是。Nacos 只存通讯录，Provider 和 Consumer 之间直连通信。Nacos Server 宕机不影响已有连接的通信，只影响新实例的发现。

**Q：开发中用 curl 还是 SDK？**
生产代码用 SDK（nacos-client），由框架（如 Spring Cloud Alibaba）在应用启动时自动注册/订阅。curl 用于运维排查和理解底层原理。