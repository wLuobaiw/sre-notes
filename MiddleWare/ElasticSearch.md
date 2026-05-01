# ElasticSearch 是什么

ElasticSearch（简称 ES） 是一款开源的分布式全文搜索引擎，基于 Lucene 构建，**主要用于**：

- 实时存储、搜索和分析大量数据（如日志、电商商品、用户行为等）
- 支持复杂的全文检索、过滤、聚合分析（如统计某类日志的出现次数、按时间聚合数据）
- 分布式架构，可横向扩展，处理 PB 级数据

**典型场景**：日志集中分析（配合 Logstash、Kibana 组成 ELK 栈）、电商商品搜索、企业内部检索系统等

## 核心组件

**集群（Cluster）**：多个节点组成的集合，通过相同的 `cluster.name` 标识（默认 `elasticsearch`），对外提供统一服务

**节点（Node）**：组成集群的单个服务器，每个节点有唯一名称，默认随机生成。节点按角色可分为：

- **Master 节点**：负责集群管理（如拆功能键/删除索引、分配分片），保证集群稳定性
- **Data 节点**：存储数据，执行索引、搜索、聚合等操作（性能铭感，需较多CPU和内存
- **Ingest 节点**：预处理数据（如日志格式化），可独立部署也可由其他节点兼任
- **Coordinating 节点**：接受客户端请求，分发任务给其他节点，汇总结果（默认所有节点都具备此功能）

**索引（Index）**：类似数据库的“表”，是多个文档的集合（如日志索引、商品索引）。索引名称必须小写

**文档（Document）**：索引中的单条数据，类似数据库中的“行”，以 JSON 格式存储（如一条日志 `{"time":"2024-05-01", "level":"ERROR", "message":"xxx"}`）

**分片（Shard）**：索引的分片，用于数据拆分（横向扩展）。一个索引默认分为 1 个主分片和 1 个副本分片：

- **主分片（Primary Shard）**：数据原始存储位置，不可修改数量（创建索引时指定）
- **副本分片（Replica Shard）**：主分片的副本，用于冗余备份和负载均衡（可动态调整数量）



---



# 基本操作

ES 的操作方式不是命令行的命令形式，而是通过调用 API 请求来处理。

API 请求调用格式（Linux 命令行为例）

```shell
curl -X 请求类型 “请求地址” -H “请求头” -d “请求体”
```



## 请求工具

**作用**：发送 HTTP 请求的工具，如示例中的 `curl`



## 请求地址（URL）

**作用**：指定要操作的资源路径（类似“文件路径”）

**格式**：`协议://服务器地址:端口/资源路径?参数`

- **协议**：`http` 或 `https`（ES 默认 http）
- **服务器地址:端口**：如 `localhost:9200`（ES 默认端口 9200）
- **资源路径**：ES 中是 `索引/文档/操作` 的路径，例如：
  - `products`：操作名为 `products` 的索引
  - `products/_doc/1001`：操作 `products` 索引中 ID 为 1001 的文档
  - `_cluster/health`：操作集群健康状态
- **参数**：可选，如 `?pretty` 让返回结果格式化（方便阅读）



## 请求头（Header）

**作用**：告诉服务器 “请求的格式是什么” “需要什么处理” 等元信息

**常用请求头**：`Content-Type: application/json`

- 含义：声明请求体是 JSON 格式（ES 中所有操作都用 JSON）

**用法**：在 `curl` 中用 `-H` 指定，例如：

```SHELL
-H "Content-Type: application/json"
```



## 请求体（Body）

**作用**：传递具体的操作数据（如创建索引的配置、文档内容、查询条件等）

**格式**：必须是 JSON 字符串（因为请求头声明了 `application/json`）

**用法**：在 `curl` 中用 `-d` 指定，例如：

```shell
-d '{"name":"机械键盘", "price":299.9}'
```

**注意**：

- `GET`/`DELETE` 类型的请求通常不需要请求体（除非有复杂参数）
- `PUT`/`POST` 类型的请求几乎都要请求体（因为要传递数据）

 

## 请求类型（HTTP 方法）

ES 的 API 严格遵循 HTTP 方法的语义，不同的”请求类型“对应不同的操作目的，核心有 4 种

### GET（读取资源）

**作用**：获取已存在的资源

**特点**：无请求体，不会修改数据

**示例**：

```shell
# 查询集群健康状态
curl -X GET "http://localhost:9200/_cluster/health?pretty"

# 查询 products 索引中 ID=1001 的文档
curl -X GET "http://localhost:9200/products/_doc/1001?pretty"
```

### PUT（创建或全量更新资源）

**作用**：

- 当资源不存在时：创建新资源
- 当资源已存在时：全量覆盖更新

**特点**：需要请求体

**示例**：

```shell
# 创建 products 索引
curl -X PUT "http://localhost:9200/products?pretty"\
-H "Content-Type:application/json"\
-d '{"settings":{"number_of_shards":1}}'

# 添加 ID=1001 的文档
curl -X PUT "http://localhost:9200/products/_doc/1001?pretty"\
-H "Content-Type:application/json"\
-d '{"name":"机械键盘", "price":299.9}'
```

### POST（灵活操作）

**作用**：

- 创建无需指定 ID 的文档（ID 由 ES 自动生成）
- 局部更新已有资源
- 执行复杂操作（如搜索、批量处理）

**特点**：需要请求体，语义更灵活

**示例**：

```shell
# 局部更新 ID=1001 的文档（只改price字段）
curl -X POST "http://localhost:9200/products/_update/1001?pretty"\
-H "Content-Type:application/json"\
-d '{"doc":{"price":279.9}}'

# 执行搜索（查询price<300的文档）
curl -X POST "http://localhost:9200/products/_search?pretty"\
-H "Content-Type:application/json"\
-d '{"query":{"range":{"price":{"It":300}}}}'
```

### DELETE（删除资源）

**作用**：删除指定资源

**特点**：无请求体，直接通过资源路径定位要删除的内容

**示例**：

```shell
# 删除 products 索引
curl -X DELETE "http://localhost:9200/products?pretty"

# 删除 ID=1001 的文档
curl -X DELETE "http://localhost:9200/_doc/1001?pretty"
```