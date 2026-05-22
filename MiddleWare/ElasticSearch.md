# ElasticSearch 是什么

ElasticSearch（简称 ES） 是一款开源的分布式全文搜索引擎，基于 Lucene 构建，**主要用于**：

- 实时存储、搜索和分析大量数据（如日志、电商商品、用户行为等）
- 支持复杂的全文检索、过滤、聚合分析（如统计某类日志的出现次数、按时间聚合数据）
- 分布式架构，可横向扩展，处理 PB 级数据

**典型场景**：日志集中分析（配合 Logstash、Kibana 组成 ELK 栈）、电商商品搜索、企业内部检索系统等

---

# 核心概念

## 集群（Cluster）

多个节点组成的集合，通过相同的 `cluster.name` 标识（默认 `elasticsearch`），对外提供统一服务。

## 节点（Node）

组成集群的单个服务器，每个节点有唯一名称，默认随机生成。节点按角色可分为：

- **Master 节点**：负责集群管理（如创建/删除索引、分配分片），保证集群稳定性
- **Data 节点**：存储数据，执行索引、搜索、聚合等操作（性能敏感，需较多 CPU 和内存）
- **Ingest 节点**：预处理数据（如日志格式化），可独立部署也可由其他节点兼任
- **Coordinating 节点**：接受客户端请求，分发任务给其他节点，汇总结果（默认所有节点都具备此功能）

## 索引（Index）

类似数据库的"表"，是多个文档的集合（如日志索引、商品索引）。索引名称必须小写。

## 文档（Document）

索引中的单条数据，类似数据库中的"行"，以 JSON 格式存储（如一条日志 `{"time":"2024-05-01", "level":"ERROR", "message":"xxx"}`）。

## 分片（Shard）与副本（Replica）

### 分片 —— 解决「数据太大，一台机器装不下」

把一份完整数据**横向切分**成多块，每块是一个分片，分散存到不同节点上。每个分片本身就是一个完整的 Lucene 索引。

```
        没有分片：                       有分片（3个）：
 ┌──────────────┐              ┌──────┐ ┌──────┐ ┌──────┐
 │   Index: 1TB │  ❌          │ 0号  │ │ 1号  │ │ 2号  │
 │   塞不进!    │              │333GB │ │333GB │ │333GB │
 └──────────────┘              │节点A │ │节点B │ │节点C │
                               └──────┘ └──────┘ └──────┘
```

分片数在创建索引时就定死（因为 ES 通过 `hash(doc_id) % 分片数` 决定文档去哪个分片 —— 分片数一变，公式全乱）。

### 副本 —— 解决「节点挂了，数据没了」

分片解决了容量问题，但引入新风险：节点 B 宕机 → 1号分片数据永久丢失。

```
    只有主分片：                        加了副本之后：
节点A    节点B    节点C          节点A        节点B        节点C
┌──────┐ ┌──────┐ ┌──────┐    ┌──────────┐ ┌──────────┐ ┌──────────┐
│ 0号   │ │ 1号  │ │ 2号  │    │ 0号(主)   │ │ 1号(主)   │ │ 2号(主)   │
│      │ │  ❌  │ │      │    │ 1号(副)  │ │ 2号(副)   │ │ 0号(副)   │
└──────┘ └──────┘ └──────┘    └──────────┘ └──────────┘ └──────────┘
节点B宕机 → 数据永久丢失         节点B宕机 → 1号副本在节点A自动顶上
```

两个硬规则：

1. **主分片和它的副本绝不在同一节点**（在同一节点就失去了冗余意义）
2. **写入只能走主分片**，副本从主分片同步数据；读取可以从主分片或副本读

### 分片 vs 副本 对比

| | 分片（Shard） | 副本（Replica） |
|:---|:---|:---|
| 解决什么问题 | 数据量超过单机容量 | 节点故障导致数据丢失 |
| 类比技术概念 | 数据库分库分表 | 数据库主从复制 |
| 能否动态改 | ❌ 创建索引时定死 | ✅ 可随时调整 |

---

# 核心原理：倒排索引

普通数据库找数据是「正向」扫描 —— 扫每一行，看哪个字段匹配。数据一多就慢。

ES 用**倒排索引** —— 写入时先把文本拆成词，给每个词建一个「这个词出现在哪些文档」的清单：

```
文档1: "系统磁盘故障"       →  分词: [系统, 磁盘, 故障]
文档2: "网络故障排查"       →  分词: [网络, 故障, 排查]
文档3: "磁盘扩容操作"       →  分词: [磁盘, 扩容, 操作]

倒排索引（类似书的索引页）:
  故障 → [文档1, 文档2]       ← 搜"故障"，直接定位，不扫全表
  磁盘 → [文档1, 文档3]
  系统 → [文档1]
  网络 → [文档2]
```

这就是 ES 搜索快的原因 —— 不是扫全文，而是查索引。

---

# 集群部署

ES 属于集群/分布式优先型中间件。单机模式仅供本地快速验证功能，不具备高可用和数据冗余能力。生产环境必须使用集群。

## 为何必须集群

- **宕机即不可用**：唯一节点一挂，所有索引不可读写
- **数据无冗余**：磁盘损坏 = 永久丢数据（副本分片没有其他节点可放）
- **无法横向扩展**：单节点 CPU/内存/磁盘到上限后性能到头

## 最小集群规划（3 节点）

```
         Node-1              Node-2              Node-3
    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
    │ master: true │   │ master: true │   │ master: true │
    │ data: true   │   │ data: true   │   │ data: true   │
    │              │   │              │   │              │
    │ P0  R1  P2   │   │ R0  P1  R2   │   │ R0  P1  P2   │
    └──────────────┘   └──────────────┘   └──────────────┘
```

- **节点数 = 3**：容忍 1 个节点宕机（奇数防止脑裂）
- **主分片 = 3**：与节点数相等，数据均匀分布
- **副本 = 1**：每个分片有 1 个备份，总数据量 x2

## 选主机制（Master Election）

ES 集群里多个节点可以被选为 master，但同一时刻只有一个在工作。

```
Node-1 (当前 master)    Node-2                Node-3
    ┌──────┐              ┌──────┐              ┌──────┐
    │ 班长  │──────────────│ 组员  │──────────────│ 组员  │
    └──────┘              └──────┘              └──────┘
        ↓ 宕机
    Node-1 ❌             Node-2                Node-3
    ┌──────┐              ┌──────┐              ┌──────┐
    │ 💀   │              │ 班长  │──────────────│ 组员  │
    └──────┘              └──────┘              └──────┘
```

**过半选举**：只有得票 > 1/2 节点数的候选者才能当选。3 节点 → 至少 2 票 → Node-2 当选，集群正常。如果只有 2 个节点 → 1 票不够过半 → 集群瘫痪。这就是为什么至少 3 个且为奇数。

## Docker Compose 部署文件

```yaml
version: '3.8'
services:
  es01:
    image: elasticsearch:8.12.0
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-demo
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false     # 学习环境关安全
    ports:
      - "9200:9200"
    volumes:
      - es01-data:/usr/share/elasticsearch/data

  es02:
    image: elasticsearch:8.12.0
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-demo
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - "9201:9200"
    volumes:
      - es02-data:/usr/share/elasticsearch/data

  es03:
    image: elasticsearch:8.12.0
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-demo
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - "9202:9200"
    volumes:
      - es03-data:/usr/share/elasticsearch/data

volumes:
  es01-data:
  es02-data:
  es03-data:
```

## 集群互通最关键的 3 个配置

| 配置项 | 作用 | 各节点填的值 |
|:---|:---|:---|
| `cluster.name` | 集群标识 —— 同一集群所有节点必须一致 | `es-demo`（三个节点全相同） |
| `discovery.seed_hosts` | 节点启动后找谁组队 | 每个节点列出除自己外的两个 |
| `cluster.initial_master_nodes` | 首次启动从哪些节点选班长 | 三个节点全列出 `es01,es02,es03` |

> 最容易踩的坑：`discovery.seed_hosts` 漏写或写错节点名 → 每个节点都以为自己孤零零的 → 形成三个独立集群，数据各存各的。

## 启动与验证

```bash
# 1. 检查集群健康 —— green/yellow/red
curl "http://localhost:9200/_cluster/health?pretty"

# 2. 查看所有节点列表
curl "http://localhost:9200/_cat/nodes?v"

# 3. 停掉 es02，验证容错能力
docker stop es02
curl "http://localhost:9200/_cluster/health?pretty"
# 应看到 status 从 green → yellow（副本未分配，但主分片正常）

# 4. 恢复 es02
docker start es02
```

---

# 常用命令与最小实践

> `text` 和 `keyword` 是 ES 最核心的类型区分，必须先理解再操作。

## text vs keyword

| 类型 | 做了什么 | 搜"磁盘故障"能匹配吗 | 适用场景 |
|:---|:---|:---|:---|
| `text` | 写入时**分词**，建倒排索引 | ✅ 能匹配「磁盘故障」「磁盘」「故障」 | 标题、正文、日志内容 —— 需要全文搜索 |
| `keyword` | **原样保留**，不分词，只精确匹配 | ❌ 只能匹配"磁盘故障"四个字完整出现 | 状态码、标签、服务器名 —— 需要精确过滤 |

## 最小 Demo：故障日志搜索

场景：工单系统的故障日志，从写入到搜索到聚合。

### 1. 创建索引并指定字段类型

```bash
curl -X PUT "http://localhost:9200/tickets?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "mappings": {
      "properties": {
        "title":        { "type": "text" },
        "level":        { "type": "keyword" },
        "server":       { "type": "keyword" },
        "description":  { "type": "text" },
        "created_at":   { "type": "date" }
      }
    }
  }'
```

### 2. 批量插入文档（_bulk）

```bash
curl -X POST "http://localhost:9200/_bulk?pretty" \
  -H "Content-Type: application/json" \
  -d '
{ "index": { "_index": "tickets", "_id": "1" } }
{ "title": "核心交换机故障", "level": "P0", "server": "switch-core-01", "description": "核心交换机持续告警，出现大量丢包，影响全机房服务", "created_at": "2025-05-01T10:00:00" }
{ "index": { "_index": "tickets", "_id": "2" } }
{ "title": "数据库磁盘故障", "level": "P0", "server": "db-master-03", "description": "数据库服务器磁盘出现坏道，IO延迟飙升到5秒，影响交易服务", "created_at": "2025-05-01T10:15:00" }
{ "index": { "_index": "tickets", "_id": "3" } }
{ "title": "Nginx内存告警", "level": "P1", "server": "web-nginx-12", "description": "Nginx实例内存使用率超过95%，疑似内存泄漏", "created_at": "2025-05-01T10:30:00" }
{ "index": { "_index": "tickets", "_id": "4" } }
{ "title": "网络延迟异常", "level": "P1", "server": "switch-access-07", "description": "接入交换机到核心交换机之间延迟从2ms飙升到200ms", "created_at": "2025-05-01T11:00:00" }
{ "index": { "_index": "tickets", "_id": "5" } }
{ "title": "日志磁盘空间不足", "level": "P2", "server": "log-server-02", "description": "日志服务器磁盘使用率达到92%，需立即清理过期日志", "created_at": "2025-05-01T11:30:00" }
'
```

`_bulk` 是 ES 的高性能批量写入接口。每个文档两行：第一行声明目标索引和 ID，第二行是文档内容。比逐条 POST 快得多。

### 3. 全文搜索（match 查询）

```bash
curl -X POST "http://localhost:9200/tickets/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "match": {
        "description": "磁盘故障"
      }
    }
  }'
```

ES 自动把搜索条件「磁盘故障」分词为「磁盘」「故障」，然后查倒排索引：

```json
"hits": {
  "total": { "value": 2, "relation": "eq" },
  "hits": [
    { "_score": 1.2, "_source": { "title": "数据库磁盘故障", ... } },
    { "_score": 0.8, "_source": { "title": "日志磁盘空间不足", ... } }
  ]
}
```

`_score` 是相关性评分 —— 两个词都命中的排前面，只命中一个词的排后面。

### 4. 聚合分析（terms 聚合）

```bash
curl -X POST "http://localhost:9200/tickets/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "by_level": {
        "terms": { "field": "level" }
      }
    }
  }'
```

`"size": 0` 表示不返回文档，只返回统计结果：

```json
"aggregations": {
  "by_level": {
    "buckets": [
      { "key": "P0", "doc_count": 2 },
      { "key": "P1", "doc_count": 2 },
      { "key": "P2", "doc_count": 1 }
    ]
  }
}
```

这就是 Kibana 柱状图、饼图的底层能力 —— ES 聚合引擎，类似 SQL 的 `GROUP BY`。

### 5. 组合搜索 + 聚合（bool 查询）

SRE 最常用模式：先筛选再统计。

```bash
curl -X POST "http://localhost:9200/tickets/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must": [
          { "match": { "description": "磁盘" } }
        ],
        "filter": [
          { "range": { "created_at": { "gte": "2025-05-01T10:00:00", "lte": "2025-05-01T11:00:00" } } }
        ]
      }
    },
    "aggs": {
      "by_server": {
        "terms": { "field": "server" }
      }
    }
  }'
```

`bool` 查询中：
- `must`：必须满足，影响相关性评分
- `filter`：必须满足，但不影响评分，且可被缓存（性能更好）

---

# 关键配置

ES 配置文件在容器内路径 `config/elasticsearch.yml`。以下是最重要的 6 个配置项：

| 配置项 | 作用 | 默认值 | 推荐值 | 改错会导致 |
|:---|:---|:---|:---|:---|
| `cluster.name` | 集群标识，同名节点才会组队 | `elasticsearch` | 自定义，如 `prod-logs` | 节点进错集群或多集群混淆 |
| `node.roles` | 节点角色（master / data / ingest） | 全部角色 | 生产环境分离：master 专做管理，data 专做存储 | 单节点负载过重，master 被数据操作拖慢 → 选主超时 → 脑裂 |
| `network.host` | ES 监听的 IP | `127.0.0.1`（仅本地） | 内网 IP，如 `0.0.0.0` 或具体网卡 IP | 配错导致外部连不上或暴露到公网 |
| `discovery.seed_hosts` | 新节点启动后去联系谁加入集群 | 无 | 列 3 个稳定 master 候选节点 | 节点孤立、各自运行形成多集群 |
| `-Xms/-Xmx` (JVM 堆) | ES 运行内存上限 | 自动取物理内存 50% | 不超过 31GB，且 Xms=Xmx | 过大触发 GC 暂停，过小频繁 OOM |
| `path.data` | 数据存储目录 | `$ES_HOME/data` | 挂载到独立磁盘/卷 | 和数据一起被删除或磁盘写满 |

## node.roles —— 角色分离

生产环境最重要的一项。默认所有节点什么活都干，需要分离：

```
❌ 默认（角色不分离）：             ✅ 角色分离后：
    3 个节点全一样                       es01        es02        es03
    ┌──────┐ ┌──────┐ ┌──────┐        ┌──────┐    ┌──────┐    ┌──────┐
    │m,d,i │ │m,d,i │ │m,d,i │        │master│    │master│    │master│
    └──────┘ └──────┘ └──────┘        │ 只管  │    │ data │    │ data │
                                      └──────┘    └──────┘    └──────┘

配置：
  es01: node.roles: [master]           ← 纯管理，不存数据
  es02: node.roles: [master, data]     ← 干活 + 参与选主
  es03: node.roles: [master, data]     ← 干活 + 参与选主

  master-eligible 仍为 3（奇数，过半 = 2 票）
  data 节点 = 2（至少 2 个才能存副本）
```

> ⚠️ master-eligible 节点必须 ≥3 且为奇数。如果只设 1 个 master-eligible，它一挂集群管理直接停摆。

## -Xms / -Xmx —— JVM 堆内存

这是最容易搞错的配置。ES 是 Java 写的，内存分两块：

```
物理内存 16GB
├── JVM 堆（Xmx）：≤ 50%，存放索引元数据、聚合结果等 Java 对象
└── 操作系统页缓存：剩余内存，Lucene 用它操作磁盘上的段文件
                            ↑
                    这部分才是搜索快的关键！
```

**公式：Xmx ≤ min(31GB, 物理内存 × 50%)**

为什么不超过 31GB？Java 的对象指针在 ≤32GB 时用 4 字节压缩，超过后自动切 8 字节，同样大小的堆实际存的东西反而变少。

## 常见配置故障排查

| 现象 | 可能原因 | 检查项 |
|:---|:---|:---|
| 启动即退出 | `network.host` 配的 IP 本机没有 | `ip addr` 核对 |
| 节点各起各的，互不相识 | `cluster.name` 不一致或 `seed_hosts` 没配 | 检查每个节点的这两个值 |
| 频繁 Full GC，搜索卡顿 | `-Xmx` 设太大（>31GB）或太小 | 确认 ≤31GB 且 ≥4GB |
| `_cluster/health` 返回 red | 某主分片丢失且无有效副本 | `GET _cat/indices?health=red` |
| 磁盘占满后写入全失败 | ES 内置保护：磁盘到 95% 锁住该节点索引 | `GET _nodes/stats/fs` |

---

# 常用监控指标（SRE 视角）

## 指标 1：集群健康状态（Cluster Health）

**一句话定位**：ES 集群的「心率」—— 告诉你集群能不能正常工作。

```bash
curl "http://localhost:9200/_cluster/health?pretty"
```

```json
{
  "cluster_name": "es-demo",
  "status": "green",
  "number_of_nodes": 3,
  "active_primary_shards": 5,
  "active_shards": 10,
  "unassigned_shards": 0
}
```

三个状态：
- **Green**：所有主分片 + 副本分片正常分配 —— 放心睡觉
- **Yellow**：主分片全正常，但部分副本未分配 —— 数据没丢但冗余不够，持续观察
- **Red**：有主分片丢失 —— 数据不可读写，需立即介入

> ⚠️ 一定要验证 `number_of_nodes` 等于预期值，不能只看 status 颜色。discovery 配错时每个节点各自为政，status 都是 green 但 nodes=1，谎报正常。

## 指标 2：未分配的分片数（Unassigned Shards）

**一句话定位**：Yellow 状态的根因指标 —— 告诉你到底有几个分片没地方放。

```bash
curl "http://localhost:9200/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason"
```

常见原因：
| unassigned.reason | 含义 | 应对 |
|:---|:---|:---|
| `NODE_LEFT` | 持有该分片的节点离开了集群 | 等 ES 自动重新分配，或手动 reroute |
| `CLUSTER_RECOVERED` | 集群恢复后分片无处分配 | 检查节点数是否 ≥ 副本数 |
| `ALLOCATION_FAILED` | 尝试分配但失败 | 检查目标节点磁盘空间 |

## 指标 3：JVM 堆使用率 + GC 频率

**一句话定位**：ES 性能稳压器 —— JVM 堆满 = 全群停摆。

```bash
curl "http://localhost:9200/_nodes/stats/jvm?pretty"
```

关注两个值：
```json
"jvm": {
  "heap_used_percent": 72,
  "gc": {
    "collectors": {
      "old": {
        "collection_count": 15,
        "collection_time_in_millis": 8500
      }
    }
  }
}
```

判断逻辑：
- `heap_used_percent < 60%` → 正常
- `60-80%` → 观察趋势，GC 频率是否上升
- `80-90%` → 堆不足信号，查是否有大聚合/大查询
- `> 90% + 频繁 Full GC` → 堆即将耗尽，搜索卡顿

关键在于**两个值一起看**：堆用 75% 但 GC 稳定 → 可能只是正常加载索引；堆用 65% 但 Full GC 频繁 → 内存泄漏或大对象问题。

## 指标与配置的关联

| 配置失误 | 监控表现 | 告警信号 |
|:---|:---|:---|
| `-Xmx` 设太大（>31GB） | Old GC 频率飙升 | 搜索延迟 P99 暴增 |
| `node.roles` 未分离 | master 节点堆吃紧 | 集群管理响应慢 |
| 副本数 = 0 | unassigned 挂零但挂节点直接 RED | 无冗余，一挂即死 |
| `discovery` 配错 | 多个独立集群，各自 nodes=1 | 数据各存各的，不可见 |

---

# 故障排查：节点失联

当集群出现 Yellow + unassigned_shards + number_of_nodes 减少时，按顺序排查：

```
第一步：节点还活着吗？
  docker ps | grep es0X          → 容器在不在？
  docker logs es0X --tail 50     → 日志有什么错误？

第二步：确认是哪种「挂」：
  ├── 进程还在但集群失联（网络分区 / GC 长暂停）
  │   → docker exec es02 curl localhost:9200
  │
  ├── 进程直接没了（OOM kill / 崩溃）
  │   → docker ps -a | grep es02 看退出码
  │
  └── 整台机器挂了（硬件 / OS 崩溃）
      → ssh 连不上主机本身

第三步：查具体原因（取决于上面是哪条分支）
  └── JVM 堆只是「进程活着但有问题」分支上的一个子项，
      不是第一手排查目标
```

> ⚠️ 不要第一时间查 JVM 堆 —— 先确认节点是死是活，活的死的排查路径完全不同。

---

# ES vs MinIO 对比

两者在架构上（分布式、分片、副本）相似，但核心定位完全不同：

| | MinIO | ES |
|:---|:---|:---|
| 定位 | 对象存储（分布式文件系统） | 搜索引擎 |
| 找数据方式 | 按文件名/路径精确查找 | 按内容模糊搜索 |
| 底层核心 | 文件分块存储 + 纠删码 | 倒排索引 + 分词器 |
| 类比 | 网盘 | 百度/Google 搜索 |

ES 无法复用 MinIO 存储的原因：ES 底层是 Lucene，需要本地文件系统维护自己的索引结构（倒排索引、段文件、事务日志）—— MinIO 只提供文件存取能力，没有索引/分词/近实时搜索能力。
