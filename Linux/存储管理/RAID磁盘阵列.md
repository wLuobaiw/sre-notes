
# 是什么

RAID（Redundant Array of Independent Disks，独立磁盘冗余阵列）将多块物理磁盘组合成一个逻辑存储单元，目的有三个：**提高性能**（并行读写）、**增加冗余**（数据容错）、**扩大容量**。

# RAID 级别对比

| 级别 | 最少盘数 | 可用容量 | 读性能 | 写性能 | 容错盘数 | 特点 |
|------|---------|---------|--------|--------|---------|------|
| RAID 0 | 2 | N × 单盘 | 高 | 高 | 0 | 条带化，**无冗余**，坏一块全丢 |
| RAID 1 | 2 | 单盘容量 | 中 | 低 | N-1 | 镜像，最安全的冗余方案 |
| RAID 5 | 3 | (N-1) × 单盘 | 高 | 中 | 1 | 分布式奇偶校验，兼顾性能与冗余 |
| RAID 6 | 4 | (N-2) × 单盘 | 高 | 低 | 2 | 双奇偶校验，比 RAID 5 更安全 |
| RAID 10 | 4 | N/2 × 单盘 | 高 | 高 | 1/每组 | 先镜像再条带，性能+冗余的最佳选择 |

## 各等级详解

**RAID 0（条带化）**：数据均匀分散写入多盘，读写可并行。性能最好，但无任何冗余——任意一块盘损坏，所有数据丢失。适合缓存、临时数据等不在乎数据丢失的场景。

**RAID 1（镜像）**：数据完整复制到每块盘。读性能提高（可从任意盘读取），写性能略低（需同时写入所有盘）。冗余度最高——只要有一块盘存活，数据就在。

**RAID 5（分布式奇偶校验）**：数据和奇偶校验信息均匀分布在所有盘上。读性能好，写性能受校验计算拖累（每次写需读→计算→写）。允许坏 1 块盘，重建期间性能大幅下降。不适合大容量盘（重建时间长，重建失败风险高）。

**RAID 6（双奇偶校验）**：比 RAID 5 多存一份校验数据。允许同时坏 2 块盘。写性能更低（双份校验计算）。推荐用于 4TB 以上大容量硬盘。

**RAID 10（镜像+条带）**：先做 RAID 1 镜像组，再对镜像组做 RAID 0 条带。同时拥有两者的优势：高性能+高冗余。代价是容量利用率只有 50%。生产环境首选。

# 软 RAID vs 硬 RAID

| 类型 | 实现方式 | 优点 | 缺点 |
|------|---------|------|------|
| 硬 RAID | 专用 RAID 卡（含处理器+缓存） | 性能好、不占 CPU、独立于 OS | 贵、卡故障可能导致数据无法恢复 |
| 软 RAID | 内核模块（md）实现 | 免费、灵活、不绑定硬件 | 占用 CPU、性能略低 |

Linux 服务器常用软 RAID，通过 `mdadm` 管理。

# mdadm 常用操作

## 创建 RAID

```shell
# 创建 RAID 1（两块盘做镜像）
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1

# 创建 RAID 5（三块盘+一块热备盘）
mdadm --create /dev/md0 --level=5 --raid-devices=3 \
      --spare-devices=1 /dev/sd{b,c,d,e}1

# 创建 RAID 10（四块盘）
mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sd{b,c,d,e}1
```

## 查看状态

```shell
# 查看所有 RAID 设备概况
cat /proc/mdstat

# 查看指定 RAID 详情
mdadm --detail /dev/md0

# 简要查看
mdadm --query /dev/md0
```

## 故障处理与重建

```shell
# 标记故障盘并移除
mdadm --manage /dev/md0 --fail /dev/sdb1
mdadm --manage /dev/md0 --remove /dev/sdb1

# 添加新盘重建
mdadm --manage /dev/md0 --add /dev/sde1

# 查看重建进度
cat /proc/mdstat
```

## 配置文件与持久化

```shell
# 生成 mdadm 配置文件（RAID 信息持久化）
mdadm --detail --scan >> /etc/mdadm.conf
```

## 停止与移除

```shell
# 停止 RAID 设备
mdadm --stop /dev/md0

# 彻底清除磁盘上的 RAID 元数据
mdadm --zero-superblock /dev/sdb1 /dev/sdc1
```

# 注意

- RAID **不是备份**——它解决的是硬件故障的可用性问题，不能防止误删除、病毒、灾难等数据丢失
- RAID 5 不适合大容量 SATA 盘（4TB+），重建时间长达数天，重建期间再坏一块盘的概率不可忽视
- 生产环境优先选择 RAID 10 或 RAID 6
- 创建 RAID 前确保磁盘没有现有数据——`--create` 会清空元数据区域
- 条带大小（chunk size）影响性能——默认 512KB，数据库场景可调大（1MB）

> 返回 [存储管理基础](./存储管理基础.md)
