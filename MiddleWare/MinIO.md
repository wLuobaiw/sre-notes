# MinIO 是什么

MinIO 是一款开源、高性能、兼容 Amazon S3 API 的分布式对象存储系统

适合存储大量非结构化数据（如图片、视频、日志等），常用于搭建私有对象存储服务



## 背景和起源

在 MinIO 出现前，做 "海量非结构化数据存储" 主要有两个选择：

- **传统文件系统/NAS**：扩展性差，单点瓶颈，不适合存几百TB甚至PB级的图片、视频、日志
- **Hadoop HDFS（分布式文件系统）**：虽然能存海量，但部署和维护极重，对小文件的IO效率低，且强依赖 Java 生态

同时，云计算时代 **Amazon S3** 成为对象存储的 "事实标准" API，但很多企业因合规、成本、数据主权等原因不行把数据放在共有云，需要在**自有服务器上搭建一个像 S3 一样好用、但又轻量高速的存储系统**



**MinIO 定位：**

- **替代私有化 S3**：完全兼容 S3 API
- **轻量、高性能**：单个二进制文件，无外部依赖，专门针对 NVMe 固态硬盘和高速网络优化，吞吐可跑满带宽
- **易于部署**：一个命令行就能在 Linux 上启动一个集群，适合云原生和容器化环境



---

# 核心概念

以下是 MinIO 体系中的几个核心概念，类似于你使用云盘时接触到的 "文件"、"文件夹"、"账号密码"。

## 服务端与客户端

**服务端（MinIO Server）**：核心服务，负责对象的存储、管理和响应 S3 API 请求

**客户端（MinIO Client, mc）**：命令行工具，用于管理 MinIO 服务

> **服务端**的核心是作为 "存储引擎" ，分布式集群的部署就是部署多个服务端
>
> **客户端**是命令行工具，用于管理 MinIO 服务，相当于遥控器

## 对象（Object）

**对象**是 MinIO 中存储的**基本单元**。一个对象 = 数据本身 + 元数据（metadata）+ 唯一标识（key）

每个对象通过一个唯一的 **对象键（Key）** 来标识，例如 `photos/2024/vacation.jpg`

> **注意**：虽然 key 看起来像文件路径（有 `/` 分隔符），但 MinIO 内部是**扁平结构**，没有真正的"文件夹"层级。控制台和工具只是用 `/` 模拟了目录树的视觉效果。

## 桶（Bucket）

**桶**是用于组织对象的容器。你可以把它理解为一个"顶级文件夹"。

- 桶名必须全局唯一（在整个 MinIO 集群中）
- 桶名必须符合 DNS 命名规范（小写字母、数字、连字符）
- 桶内的对象数量没有限制
- 每个桶可以独立设置访问策略、版本控制、生命周期规则等

## 访问密钥（Access Key & Secret Key）

- **Access Key**：相当于"用户名"，用于标识访问者身份
- **Secret Key**：相当于"密码"，用于验证访问者身份

这两个密钥成对使用，MinIO 通过它们来认证每一次 API 请求。

## 区域（Region）

MinIO 支持多区域部署。区域用于标识桶的物理位置，默认是 `us-east-1`（与 S3 默认值一致）。

在分布式部署中，你可以用区域来区分不同机房或数据中心的 MinIO 集群。



---

# 原理及架构



## 存储原理

MinIO 将硬盘上的目录和文件视作 "对象"（Buckets+Objects），通过 HTTP RESTful API 对外暴露，底层采用**纠删码（Erasure Coding）**来保证数据高可用和持久性，而不是传统 RAID 或三副本

### 纠删码（Erasure Coding）详解

纠删码是 MinIO 最核心的数据保护机制。它将一个对象拆分为 **N 份数据块 + M 份校验块**，分散存储在不同的磁盘/节点上。

例如：一个 100MB 的文件，在 4 节点各 2 盘的集群中：
1. 文件被切成多个数据分片
2. 同时计算出校验分片（用于故障恢复）
3. 所有分片分散写入不同的磁盘

**纠删码 vs 传统多副本**：

| 对比维度     | 纠删码（MinIO）           | 三副本（传统方案）   |
| :----------- | :------------------------ | :------------------- |
| **存储效率** | ~50%~83%（取决于配置）    | 33%（存1份实际占3份） |
| **容错能力** | 可容忍 ≤ M 块磁盘损坏    | 可容忍 2 块磁盘损坏  |
| **恢复速度** | 快（并行从多盘重建）      | 较慢（从单副本拷贝）  |
| **CPU 开销** | 较高（需要编解码计算）    | 极低                  |

### MinIO 的高可用

- 只要你满足 `N/2 + 1` 的磁盘在线数（4 节点，每节点 2 盘 = 8 盘，需至少 5 盘在线），集群就可以正常读写
- 节点宕机后会自动将流量导到剩余健康节点
- **注意**：如果集群规模太小（比如 2 节点），故障恢复窗口极窄，生产**建议至少 4 节点 4 盘**以上。同时，要监控磁盘和节点健康，及时更换故障硬件

### 数据一致性

MinIO 采用**强一致性**模型，所有写操作在确认成功前，数据已被写入 `N/2 + 1` 个磁盘。读取时也从 `N/2 + 1` 个磁盘读取并验证，确保不会读到过期或损坏的数据。

## 架构设计

**MinIO 要求最小集群：**至少 2 个节点，每个节点至少 2 块盘（可以目录模拟）
这样可以做到即使 1 个节点宕机也不丢数据。但生产环境推荐 >= 4 节点，每节点 4 块盘以上

### Erasure Set（纠删集）

每个节点上的多块磁盘组成一个 Erasure Set。这个集合的大小在**启动时确定**，之后不可更改。

- Erasure Set 大小 = 节点数 x 每节点磁盘数
- 决定了容错能力：8 块盘的 Erasure Set（如 4 节点各 2 盘），能容忍 1 个节点宕机或 2 块盘损坏
- 如果要扩容，MinIO 通过**添加新的 Erasure Set** 来实现，而非在原有集合中增加磁盘

### 节点自动发现

只需把所有节点的地址作为参数传给每个 MinIO Server，它们就能自动组成集群，无需额外的服务发现组件（如 ZooKeeper、Etcd）。

### 无状态设计

MinIO 节点本身不存储集群元数据。集群拓扑信息通过启动参数确定，对象元数据直接存储在每个对象中。这意味着：
- 没有单点故障
- 没有需要维护的元数据数据库
- 集群扩缩容简单



---

# 搭建部署



## 单机部署

单机模式主要用于开发测试，**不能做到高可用**

### 二进制（Linux）

以 `Linux x86_64` 为例

```shell
# 下载
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio

# 启动单机单盘模式（数据存储在 /data 目录）
./minio server /data
```

启动后会输出 `Access Key` 和 `Secret Key`，以及访问地址 `http://IP:9000`

**指定自定义凭证**：

```shell
MINIO_ROOT_USER=admin MINIO_ROOT_PASSWORD=admin123 ./minio server /data
```

### Docker

```shell
docker run -d \
    -p 9000:9000 \
    -p 9001:9001 \
    -v /data:/data \
    -e MINIO_ROOT_USER=minioadmin \
    -e MINIO_ROOT_PASSWORD=minioadmin \
    minio/minio server /data --console-address ":9001"
```

- `9000`：S3 API 端口
- `9001`：Web 控制台端口
- `/data`：数据存储目录

### 单机多盘模式

单机也可以用多块盘开启纠删码模式（用于测试纠删码特性）：

```shell
# 创建模拟目录
mkdir -p /data/minio{1,2,3,4}

# 启动（4 块盘，纠删码模式）
./minio server /data/minio{1,2,3,4}
```



## 集群/分布式部署

多节点 + 多磁盘联合工作，形成一个**统一的资源池**。
数据通过纠删码跨节点分片，容忍节点和磁盘故障。
新增节点后存储池扩大（通过新增 Erasure Set）。

**关键点**：集群模式下，**必须保持所有节点的磁盘配置一致**（数量、顺序），不能某个节点少一块盘。

### 部署前准备

以下部署以 **4 节点，每节点 2 块盘** 为例

假设四个节点 IP：`192.168.1.11 ~ 192.168.1.14`

1. 在每个节点上创建数据目录：

```shell
# 每个节点执行
mkdir -p /data/minio{1,2}
```

2. 确保所有节点时间同步（NTP），这对于 S3 签名验证至关重要。

3. 如果是生产环境，建议使用独立的磁盘分区或裸盘，而非系统盘上的目录。

### 方式一：二进制启动

在每个节点上执行相同的命令：

```shell
./minio server \
    http://192.168.1.11/data/minio{1,2} \
    http://192.168.1.12/data/minio{1,2} \
    http://192.168.1.13/data/minio{1,2} \
    http://192.168.1.14/data/minio{1,2} \
    --console-address ":9001"
```

> **关键**：每个节点上的启动命令**完全相同**，MinIO 会自动识别本机 IP，仅管理本机的磁盘。

### 方式二：Docker Compose

在管理节点上编写 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  minio1:
    image: minio/minio
    container_name: minio1
    hostname: minio1
    restart: always
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server http://minio{1...4}/data{1...2} --console-address ":9001"
    volumes:
      - /data/minio1-1:/data1
      - /data/minio1-2:/data2
    networks:
      - minio-net

  minio2:
    image: minio/minio
    container_name: minio2
    hostname: minio2
    restart: always
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server http://minio{1...4}/data{1...2} --console-address ":9001"
    volumes:
      - /data/minio2-1:/data1
      - /data/minio2-2:/data2
    networks:
      - minio-net

  minio3:
    image: minio/minio
    container_name: minio3
    hostname: minio3
    restart: always
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server http://minio{1...4}/data{1...2} --console-address ":9001"
    volumes:
      - /data/minio3-1:/data1
      - /data/minio3-2:/data2
    networks:
      - minio-net

  minio4:
    image: minio/minio
    container_name: minio4
    hostname: minio4
    restart: always
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server http://minio{1...4}/data{1...2} --console-address ":9001"
    volumes:
      - /data/minio4-1:/data1
      - /data/minio4-2:/data2
    networks:
      - minio-net

networks:
  minio-net:
    driver: bridge
```

### 方式三：Kubernetes 部署

MinIO 官方提供了 Helm Chart 和 Operator 两种方式在 Kubernetes 上部署：

```shell
# 使用 Helm 部署
helm repo add minio https://charts.min.io/
helm install minio minio/minio \
    --set mode=distributed \
    --set replicas=4 \
    --set drivesPerNode=2 \
    --set rootUser=minioadmin \
    --set rootPassword=minioadmin
```



---

# 常用命令（mc 客户端）

MinIO 主要使用其客户端（MinIO Client）也就是 `mc` 命令来进行操作

## 安装 mc 客户端

```shell
# Linux
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# macOS
brew install minio/stable/mc

# Windows
# 下载 exe 文件并放入 PATH 目录
```

## 启动配置

mc 首次使用前，需要先配置。mc 会将配置信息保存在 `~/.mc/config.json`：

```shell
mc alias set myminio http://192.168.1.11:9000 minioadmin minioadmin
```

这条命令创建了一个名为 `myminio` 的集群别名，之后所有操作都可以通过这个别名进行。

> 此处的集群 IP 可以是集群的任意一个节点。

## 桶管理

### 创建桶

```shell
mc mb myminio/mybucket
```

### 查看桶列表

```shell
mc ls myminio
```

### 查看桶信息

```shell
mc stat myminio/mybucket
```

### 删除桶（桶必须为空）

```shell
mc rb myminio/mybucket

# 强制删除（即使桶中有对象）
mc rb --force myminio/mybucket
```

## 对象操作

### 上传文件

```shell
# 上传单个文件
mc cp localfile myminio/mybucket

# 上传整个目录（递归）
mc cp --recursive ./local-dir/ myminio/mybucket/

# 上传并设置对象元数据
mc cp --attr "Content-Type=image/png;x-amz-meta-author=luobai" photo.png myminio/mybucket
```

### 下载文件

```shell
# 下载单个文件
mc cp myminio/mybucket/localfile .

# 下载整个目录
mc cp --recursive myminio/mybucket/ ./download-dir/
```

### 列出对象

```shell
# 列出桶中的所有对象
mc ls myminio/mybucket

# 递归列出（包括所有"子目录"）
mc ls --recursive myminio/mybucket

# 列出某个前缀下的对象
mc ls myminio/mybucket/photos/
```

### 查看对象信息

```shell
mc stat myminio/mybucket/localfile
```

### 删除对象

```shell
# 删除单个对象
mc rm myminio/mybucket/localfile

# 递归删除某个前缀下的所有对象
mc rm --recursive --force myminio/mybucket/temp/

# 删除超过 7 天的对象
mc rm --recursive --older-than 7d myminio/mybucket/logs/
```

## 数据同步（mirror）

`mc mirror` 是本地的目录和 MinIO 的桶之间做双向同步：

```shell
# 将本地目录同步到 MinIO（本地 → MinIO）
mc mirror ./local-folder/ myminio/mybucket/

# 将 MinIO 同步到本地（MinIO → 本地）
mc mirror myminio/mybucket/ ./local-folder/

# 持续监控并同步（类似 rsync 的 watch 模式）
mc mirror --watch ./local-folder/ myminio/mybucket/

# 覆盖同步（强制覆盖已存在的文件）
mc mirror --overwrite ./local-folder/ myminio/mybucket/
```

## 查找对象（find）

```shell
# 查找桶中所有 .jpg 文件
mc find myminio/mybucket --name "*.jpg"

# 查找大于 100MB 的文件
mc find myminio/mybucket --larger 100M

# 查找超过 30 天未修改的文件
mc find myminio/mybucket --older-than 30d
```

## 管理类命令

### 查看集群信息

```shell
mc admin info myminio
```

### 查看磁盘使用情况

```shell
mc admin info myminio --json
```

### 列出所有用户

```shell
mc admin user list myminio
```

### 添加用户

```shell
mc admin user add myminio newuser newpassword
```

### 列出所有策略

```shell
mc admin policy list myminio
```

### 给用户绑定策略

```shell
mc admin policy attach myminio readwrite --user newuser
```

## 匿名访问（预签名 URL）

```shell
# 生成一个 7 天有效的下载链接
mc share download myminio/mybucket/localfile --expire 168h

# 生成一个 1 小时有效的上传链接
mc share upload myminio/mybucket/ --expire 1h
```



---

# SDK 使用

在实际项目中，我们一般通过代码来操作 MinIO，而非命令行。MinIO 官方提供了多种语言的 SDK。

## Java SDK

### 引入依赖

Maven：

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.7</version>
</dependency>
```

### 初始化客户端

```java
import io.minio.MinioClient;

MinioClient minioClient = MinioClient.builder()
    .endpoint("http://192.168.1.11:9000")
    .credentials("minioadmin", "minioadmin")
    .build();
```

### 桶操作

```java
// 检查桶是否存在
boolean exists = minioClient.bucketExists(
    BucketExistsArgs.builder().bucket("mybucket").build()
);

// 创建桶
if (!exists) {
    minioClient.makeBucket(
        MakeBucketArgs.builder().bucket("mybucket").build()
    );
}

// 列出所有桶
List<Bucket> buckets = minioClient.listBuckets();
for (Bucket bucket : buckets) {
    System.out.println(bucket.name());
}

// 删除桶
minioClient.removeBucket(
    RemoveBucketArgs.builder().bucket("mybucket").build()
);
```

### 上传文件

```java
// 上传本地文件
minioClient.uploadObject(
    UploadObjectArgs.builder()
        .bucket("mybucket")
        .object("photos/sunset.jpg")
        .filename("/local/path/sunset.jpg")
        .contentType("image/jpeg")
        .build()
);

// 上传输入流（适合前端传来的文件）
InputStream inputStream = new FileInputStream("/local/path/file.pdf");
minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("mybucket")
        .object("documents/report.pdf")
        .stream(inputStream, inputStream.available(), -1)
        .contentType("application/pdf")
        .build()
);
```

### 下载文件

```java
// 下载到本地文件
minioClient.downloadObject(
    DownloadObjectArgs.builder()
        .bucket("mybucket")
        .object("photos/sunset.jpg")
        .filename("/download/sunset.jpg")
        .build()
);

// 读取为输入流（用于返回给前端）
InputStream stream = minioClient.getObject(
    GetObjectArgs.builder()
        .bucket("mybucket")
        .object("photos/sunset.jpg")
        .build()
);
```

### 删除对象

```java
minioClient.removeObject(
    RemoveObjectArgs.builder()
        .bucket("mybucket")
        .object("photos/sunset.jpg")
        .build()
);
```

### 生成预签名 URL

```java
// 生成一个有效期 7 天的下载链接（可用于分享给用户）
String presignedUrl = minioClient.getPresignedObjectUrl(
    GetPresignedObjectUrlArgs.builder()
        .bucket("mybucket")
        .object("photos/sunset.jpg")
        .expiry(7, TimeUnit.DAYS)
        .method(Method.GET)
        .build()
);
```

## Python SDK

### 安装

```shell
pip install minio
```

### 初始化客户端

```python
from minio import Minio

client = Minio(
    "192.168.1.11:9000",
    access_key="minioadmin",
    secret_key="minioadmin",
    secure=False  # 是否使用 HTTPS
)
```

### 桶操作

```python
# 检查桶是否存在
if not client.bucket_exists("mybucket"):
    client.make_bucket("mybucket")

# 列出所有桶
buckets = client.list_buckets()
for bucket in buckets:
    print(bucket.name)

# 删除桶
client.remove_bucket("mybucket")
```

### 上传文件

```python
# 上传本地文件
client.fput_object(
    "mybucket",
    "photos/sunset.jpg",
    "/local/path/sunset.jpg",
    content_type="image/jpeg"
)

# 上传字节数据（适合处理前端传来的文件）
from io import BytesIO
data = BytesIO(b"Hello, MinIO!")
client.put_object(
    "mybucket",
    "text/hello.txt",
    data,
    length=data.getbuffer().nbytes,
    content_type="text/plain"
)
```

### 下载文件

```python
# 下载到本地文件
client.fget_object(
    "mybucket",
    "photos/sunset.jpg",
    "/download/sunset.jpg"
)

# 读取为字节数据（用于返回给前端）
response = client.get_object("mybucket", "photos/sunset.jpg")
data = response.read()
response.close()
```

### 生成预签名 URL

```python
# 生成一个有效期 7 天的下载链接
url = client.presigned_get_object("mybucket", "photos/sunset.jpg", expires=timedelta(days=7))
```



---

# 桶策略与访问控制

MinIO 使用 **IAM（Identity and Access Management）** 风格的策略进行权限管理。策略使用 JSON 格式编写，遵循 AWS IAM Policy 语法。

## 内置策略

MinIO 自带三个内置策略：

| 策略名       | 权限            | 说明                       |
| :----------- | :-------------- | :------------------------- |
| `consoleAdmin` | 管理员全部权限  | Web 控制台的默认管理员策略 |
| `readwrite`  | 读写权限        | 可以对桶进行读写操作       |
| `readonly`   | 只读权限        | 只能读取对象，不能修改     |

## 自定义策略

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::mybucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::mybucket"
      ]
    }
  ]
}
```

### 创建和使用自定义策略

```shell
# 将上面的 JSON 保存为 custom-policy.json，然后执行
mc admin policy create myminio mypolicy custom-policy.json

# 绑定策略到用户
mc admin policy attach myminio mypolicy --user newuser

# 查看用户当前的策略
mc admin user info myminio newuser

# 解绑策略
mc admin policy detach myminio mypolicy --user newuser
```

## 公开桶（匿名访问）

如果需要让桶里的文件可以被匿名访问（比如做图床或静态资源托管），设置桶的匿名访问策略：

```shell
# 允许匿名下载（所有人可读）
mc anonymous set download myminio/mybucket

# 允许匿名上传和下载（不推荐，安全风险高）
mc anonymous set public myminio/mybucket

# 关闭匿名访问（恢复为默认私有）
mc anonymous set private myminio/mybucket
```

## 服务账户（Service Account）

服务账户是一种特殊的访问凭证，可以绑定到具体用户下，适合给不同应用分配不同的密钥：

```shell
# 创建一个服务账户（绑定到某个用户）
mc admin user svcacct add myminio newuser --access-key "myappkey" --secret-key "myappsecret"

# 列出用户的所有服务账户
mc admin user svcacct list myminio newuser

# 删除服务账户
mc admin user svcacct rm myminio myappkey
```



---

# 数据保护

## 版本控制（Versioning）

开启版本控制后，每次覆盖或删除对象时，MinIO 会保留历史版本，可以随时恢复。

### 开启版本控制

```shell
mc version enable myminio/mybucket
```

### 查看对象的所有版本

```shell
mc ls --versions myminio/mybucket/localfile
```

### 恢复指定版本

```shell
mc cp --version-id "版本ID" myminio/mybucket/localfile ./restored-file
```

**版本控制 + 删除**：

- 普通删除：对象被标记为 "Delete Marker"，数据仍在，可以恢复
- 永久删除：需要指定版本 ID 删除，这样该版本数据才会真正清除

> **注意**：开启版本控制后，所有历史版本都会占用存储空间，需要配合生命周期规则清理旧版本。

## 对象锁定（Object Lock）

对象锁定用于防止对象被删除或修改，常用于合规场景（如金融、医疗行业的数据留存）。

需要先在创建桶时开启：

```shell
mc mb --with-lock myminio/locked-bucket
```

锁定模式：

- **监管模式（GOVERNANCE）**：普通用户不能删除/修改，但拥有特殊权限的角色可以
- **合规模式（COMPLIANCE）**：在保留期内，**任何人都不能**删除/修改（包括 root 用户）

## 站点复制（Site Replication）

站点复制用于在不同 MinIO 集群之间做**全量数据同步**，适合多机房灾备或多活架构：

```shell
# 将 myminio 集群的数据同步到 backup 集群
mc admin replicate add myminio backup --sync-id "site-replication-1"
```

> 站点复制同步的内容包括：桶、对象、IAM 策略、服务账户、通知配置等几乎所有集群配置。

## 桶复制（Bucket Replication）

桶复制是比站点复制更细粒度的同步方式，可以只同步特定桶，且支持过滤规则：

```shell
# 配置桶复制规则（需要先创建 replication rule JSON）
mc replicate add myminio/mybucket --remote-bucket backup/mybucket --priority 1
```

## 备份与恢复

```shell
# 全量备份一个桶到本地目录
mc mirror myminio/mybucket /backup/mybucket/

# 恢复（反向 mirror）
mc mirror /backup/mybucket/ myminio/restored-bucket/

# 增量备份（结合 --watch 和 cron）
mc mirror --watch --overwrite myminio/mybucket /backup/mybucket/
```

### 使用 Restic 做加密备份

```shell
# 初始化 restic 仓库（在 MinIO 桶上）
restic -r s3:http://192.168.1.11:9000/mybucket init

# 备份本地目录到 MinIO
restic -r s3:http://192.168.1.11:9000/backup-bucket backup /data
```



---

# 事件通知

MinIO 可以在对象发生变化时（上传、删除、访问等）发送通知到外部系统，常见场景：上传图片后自动生成缩略图、上传文件后触发病毒扫描、删除日志等。

## 支持的通知目标

| 类型             | 说明                | 典型场景          |
| :--------------- | :------------------ | :---------------- |
| **Webhook**      | 发送 HTTP POST 请求 | 触发自定义服务    |
| **Redis**        | 发布到 Redis 频道   | 实时消息通知      |
| **NATS**         | 发布到 NATS 主题    | 云原生事件总线    |
| **Kafka**        | 发送到 Kafka Topic  | 大数据 pipeline   |
| **AMQP**         | 发送到 RabbitMQ     | 异步任务队列      |
| **MySQL**        | 写入数据库表        | 审计日志持久化    |
| **Elasticsearch** | 写入 ES 索引       | 日志分析/Kibana   |

## 配置事件通知

以 Webhook 为例，当有对象上传到 `uploads` 桶时，自动 POST 到我们的服务：

```shell
# 添加 Webhook 通知配置
mc event add myminio/uploads arn:minio:sqs::WEBHOOK:webhook \
    --event put,delete \
    --suffix .jpg,.png
```

Webhook 服务端会收到如下 JSON：

```json
{
  "EventName": "s3:ObjectCreated:Put",
  "Key": "uploads/photo.jpg",
  "Records": [...]
}
```

### 查看和删除通知配置

```shell
# 查看桶的通知配置
mc event list myminio/uploads

# 删除通知配置
mc event remove myminio/uploads arn:minio:sqs::WEBHOOK:webhook
```

### 配置 Redis 通知

```shell
mc admin config set myminio notify_redis:1 \
    address="redis-host:6379" \
    key="minio-events" \
    format="namespace"

# 重启或重载配置
mc admin service restart myminio
```

### 配置 Kafka 通知

```shell
mc admin config set myminio notify_kafka:1 \
    brokers="kafka:9092" \
    topic="minio-events"

mc admin service restart myminio
```



---

# 生命周期管理

随着时间的推移，桶中的数据会不断增长。生命周期规则可以**自动清理或转移**旧数据，节省存储成本。

## 配置生命周期规则

将如下 JSON 保存为 `lifecycle.json`：

```json
{
  "Rules": [
    {
      "ID": "delete-old-logs",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Expiration": {
        "Days": 30
      }
    },
    {
      "ID": "cleanup-old-versions",
      "Status": "Enabled",
      "Filter": {
        "Prefix": ""
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 7
      }
    }
  ]
}
```

第一条规则：30 天后自动删除 `logs/` 前缀下的对象
第二条规则：对于非当前版本的对象（即被覆盖的旧版本），7 天后自动删除

### 导入规则

```shell
mc ilm import myminio/mybucket < lifecycle.json
```

### 查看规则

```shell
mc ilm rule list myminio/mybucket
```



---

# 安全配置

## TLS/HTTPS 配置

生产环境中**必须启用 TLS**，防止数据在传输过程中被窃取或篡改。

### 自动获取证书（Let's Encrypt）

```shell
export MINIO_SERVER_URL="https://minio.example.com:9000"

./minio server /data \
    --certs-dir /etc/minio/certs \
    --console-address ":9001"
```

MinIO 会自动检测 `.certs` 目录下的证书文件。

### 手动配置证书

将证书和私钥放到指定目录：

```shell
# 证书文件路径
~/.minio/certs/public.crt
~/.minio/certs/private.key

# 或者每个域名单独目录
~/.minio/certs/minio.example.com/public.crt
~/.minio/certs/minio.example.com/private.key
```

### 生成自签名证书（仅用于测试）

```shell
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout private.key -out public.crt \
    -subj "/CN=minio.example.com"
```

## 服务端加密（SSE）

MinIO 支持服务端加密，数据在落盘前自动加密，读取时自动解密。

| 加密方式    | 说明                                             |
| :---------- | :----------------------------------------------- |
| **SSE-S3**  | 使用 MinIO 管理的密钥（简单，一键开启）         |
| **SSE-C**   | 使用客户提供的密钥（每次请求都需带密钥）         |
| **SSE-KMS** | 使用外部 KMS 系统管理密钥（如 HashiCorp Vault） |

SSE-S3 最简单，只需在创建桶时开启：

```shell
mc mb --with-lock myminio/sensitive-data

# 在桶级别开启默认加密
mc encrypt set sse-s3 myminio/sensitive-data
```

## 网络隔离

生产环境建议将 MinIO 的 API 端口和控制台端口分开管理：

```shell
# API 端口仅内网暴露
./minio server /data --address ":9000" --console-address ":9001"

# 通过防火墙规则限制
# 9000 端口：仅允许应用服务器 IP 访问
# 9001 端口：仅允许运维管理 IP 访问
```



---

# 监控与日志



## Prometheus + Grafana 监控

MinIO 内置了 Prometheus 指标端点，可以实时采集集群的运行状态。

### 配置 Prometheus 采集

1. 生成 Prometheus 配置所需的 bearer token：

```shell
mc admin prometheus generate myminio
```

2. 在 `prometheus.yml` 中添加：

```yaml
scrape_configs:
  - job_name: minio
    metrics_path: /minio/v2/metrics/cluster
    scheme: http
    static_configs:
      - targets: ['192.168.1.11:9000']
    bearer_token: "生成的token"
```

### 核心监控指标

| 指标                     | 含义               | 告警参考值          |
| :----------------------- | :----------------- | :------------------ |
| `disk_storage_available` | 磁盘可用空间       | < 20% 告警          |
| `disk_storage_used`      | 磁盘已用空间       | > 80% 告警          |
| `cluster_disk_online`    | 在线磁盘数         | < N/2+1 严重告警    |
| `cluster_disk_offline`   | 离线磁盘数         | > 0 告警            |
| `s3_requests_total`      | S3 请求总数        | 观察趋势，做容量规划 |
| `s3_requests_errors`     | S3 请求错误数      | 突然升高需排查      |
| `heal_objects_total`     | 修复对象数         | 持续高值表示磁盘异常  |

## 日志配置

MinIO 默认将日志输出到标准输出。生产环境建议将日志集中收集。

```shell
# 查看实时日志（Docker 环境）
docker logs -f minio1

# 配置日志输出到文件
export MINIO_LOG_FILE=/var/log/minio/minio.log
./minio server /data
```

## 审计日志

MinIO 支持将每次 API 请求记录到审计日志：

```shell
# 配置审计日志输出到 Webhook
mc admin config set myminio audit_webhook:1 \
    endpoint="http://log-server:8080/audit"
```

每条审计日志包含：请求时间、API 名称、请求者、对象路径、响应状态码、耗时等。



---

# 性能调优

## 硬件建议

| 组件   | 建议                                                         |
| :----- | :----------------------------------------------------------- |
| **CPU** | 纠删码需要 CPU 编解码，建议至少 8 核，生产环境 16 核以上     |
| **内存** | 最小 8GB，生产建议 32GB+（MinIO 会使用内存做读写缓存）       |
| **磁盘** | NVMe SSD 最佳，至少 SATA SSD。**不要用 HDD 做生产**          |
| **网络** | 万兆网络起步（纠删码模式下节点间有大量数据传输）             |

## 内核参数调优

```shell
# /etc/sysctl.conf

# 提高网络连接数
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096

# 优化 TCP
net.ipv4.tcp_tw_reuse = 1
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728

# 提高文件描述符限制
fs.file-max = 1000000
```

应用配置：

```shell
sysctl -p
```

## MinIO 环境变量

| 变量名                      | 说明             | 建议值                  |
| :-------------------------- | :--------------- | :---------------------- |
| `MINIO_STORAGE_CLASS_STANDARD` | 标准存储类 EC 配比 | `EC:4`（4 数据 + 4 校验） |
| `MINIO_API_REQUESTS_MAX`    | 最大并发 API 请求 | 根据 CPU 核数调整       |
| `MINIO_IDLE_CONNS_PER_HOST` | 每个主机的空闲连接 | `256`                   |
| `MINIO_COMPRESSION_ENABLE`  | 启用数据压缩     | `on`                    |

## 常见性能问题

1. **上传速度慢**：
   - 检查网络带宽是否跑满
   - 确认是否使用 SSD（HDD 的 IOPS 极低）
   - 减小分片大小（MinIO 默认 16MB 分片，小文件多时可调小）

2. **下载速度慢**：
   - 检查是否开启了 TLS（TLS 握手有开销，但影响不大）
   - 确认客户端是否与 MinIO 节点在同一网络（跨地域延迟高）

3. **大量小文件（KB 级）性能差**：
   - 对象存储本身不适合海量小文件场景
   - 考虑在上层做文件合并（如 Parquet 格式）
   - 确保使用 NVMe SSD



---

# Spring Boot 集成 MinIO

企业中 MinIO 最常用的场景之一就是与 Spring Boot 应用集成，用于文件上传下载。

## 依赖与配置

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.7</version>
</dependency>
```

```yaml
# application.yml
minio:
  endpoint: http://192.168.1.11:9000
  access-key: minioadmin
  secret-key: minioadmin
  bucket-name: myapp-files
```

## 配置类

```java
@Configuration
public class MinioConfig {

    @Value("${minio.endpoint}")
    private String endpoint;

    @Value("${minio.access-key}")
    private String accessKey;

    @Value("${minio.secret-key}")
    private String secretKey;

    @Bean
    public MinioClient minioClient() {
        return MinioClient.builder()
            .endpoint(endpoint)
            .credentials(accessKey, secretKey)
            .build();
    }
}
```

## 上传下载工具类

```java
@Component
public class MinioUtil {

    private final MinioClient minioClient;

    @Value("${minio.bucket-name}")
    private String bucketName;

    public MinioUtil(MinioClient minioClient) {
        this.minioClient = minioClient;
    }

    // 上传文件并返回访问 URL
    public String uploadFile(MultipartFile file, String objectName) {
        minioClient.putObject(
            PutObjectArgs.builder()
                .bucket(bucketName)
                .object(objectName)
                .stream(file.getInputStream(), file.getSize(), -1)
                .contentType(file.getContentType())
                .build()
        );
        return "/file/" + objectName; // 返回相对路径，由 Controller 代理下载
    }

    // 下载文件
    public InputStream downloadFile(String objectName) {
        return minioClient.getObject(
            GetObjectArgs.builder()
                .bucket(bucketName)
                .object(objectName)
                .build()
        );
    }

    // 生成预签名 URL（适用于大文件直传）
    public String presignedUrl(String objectName, int expireMinutes) {
        return minioClient.getPresignedObjectUrl(
            GetPresignedObjectUrlArgs.builder()
                .bucket(bucketName)
                .object(objectName)
                .expiry(expireMinutes, TimeUnit.MINUTES)
                .method(Method.GET)
                .build()
        );
    }
}
```

## Controller 示例

```java
@RestController
@RequestMapping("/file")
public class FileController {

    private final MinioUtil minioUtil;

    public FileController(MinioUtil minioUtil) {
        this.minioUtil = minioUtil;
    }

    @PostMapping("/upload")
    public String upload(@RequestParam("file") MultipartFile file) {
        String objectName = UUID.randomUUID() + "_" + file.getOriginalFilename();
        return minioUtil.uploadFile(file, objectName);
    }

    @GetMapping("/{objectName}")
    public void download(@PathVariable String objectName, HttpServletResponse response) {
        try (InputStream inputStream = minioUtil.downloadFile(objectName)) {
            response.setContentType("application/octet-stream");
            org.apache.commons.io.IOUtils.copy(inputStream, response.getOutputStream());
            response.flushBuffer();
        }
    }
}
```

> **安全提醒**：生产环境中，文件下载应该做权限校验（如用户是否登录、是否有权访问该文件），不能直接暴露裸 URL。

---

# 常见问题排查



## 启动失败："not enough disks" 错误

**现象**：`Error: Invalid number of endpoints` 或 `not enough disks`

**原因**：集群模式下，在线磁盘数不足（小于 Erasure Set 的一半 + 1）

**解决**：
1. 检查每个节点的磁盘是否正常挂载
2. 检查网络连通性（节点间能否互相通信）
3. 确保所有节点使用相同的启动参数

## 上传文件报 "Access Denied"

**原因**：访问密钥无权限或签名错误

**排查步骤**：
1. 确认 Access Key 和 Secret Key 正确
2. 检查该用户/服务账户的 IAM 策略是否包含 `s3:PutObject`
3. 如果客户端和服务器时间差超过 5 分钟，S3 签名验证也会失败，确保 NTP 同步

## 桶无法删除

```
mc: <ERROR> Unable to remove bucket. Bucket not empty.
```

**解决**：必须先清空桶中所有对象，包括所有版本（如果开启了版本控制）：
```shell
# 先清空桶
mc rm --recursive --force --versions myminio/mybucket
# 再删除桶
mc rb myminio/mybucket
```

## 磁盘空间不足

**预防措施**：
1. 配置 Prometheus 监控 `disk_storage_available` 指标
2. 设置生命周期规则自动清理旧数据
3. 定期检查：`mc admin info myminio`

**紧急处理**：扩容（添加新的 Erasure Set）或清理不必要的数据

## mc 命令报 "Unable to establish connection"

1. 确认 MinIO 服务是否在运行：`ps aux | grep minio`
2. 检查端口是否在监听：`netstat -tlnp | grep 9000`
3. 检查防火墙规则是否放行了 9000 端口
4. 确认 `mc alias set` 时使用的地址和端口正确



---

# MinIO vs 其他对象存储

| 特性           | MinIO               | Ceph RGW               | FastDFS                 |
| :------------- | :------------------ | :--------------------- | :---------------------- |
| **API 兼容**   | 完全兼容 S3         | 兼容 S3 + Swift        | 自有协议，不兼容 S3     |
| **部署复杂度** | 极简（单二进制）    | 复杂（需多个组件）     | 中等                    |
| **存储机制**   | 纠删码              | CRUSH 算法 + 副本/纠删码 | 多副本                  |
| **高性能**     | 极致优化（NVMe）    | 良好                   | 一般                    |
| **社区生态**   | 活跃，K8s 友好      | 成熟，OpenStack 生态   | 较少更新                |
| **适用场景**   | 私有云、混合云      | 大规模统一存储         | 小文件（已较过时）      |
| **运维成本**   | 低                  | 高                     | 中                      |



---

# 推荐学习路径

如果你是零基础学习 MinIO，建议按以下顺序：

1. **理解概念**：先搞明白 "对象存储" 和传统文件存储的区别，理解 Bucket / Object 模型
2. **动手部署**：用 Docker 在本地跑一个单机模式，登录 Web 控制台体验
3. **命令行操作**：安装 mc 客户端，练习桶和对象的基本增删改查
4. **SDK 集成**：用你最熟悉的语言（Java/Python），写一个"上传-下载-删除"的小 Demo
5. **进阶特性**：依次学习版本控制、生命周期、事件通知、桶策略
6. **生产实践**：学习集群部署、TLS 配置、监控告警、备份恢复
