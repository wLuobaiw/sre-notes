
# 是什么

LVM（Logical Volume Manager）在物理磁盘和文件系统之间加了一层抽象，把僵硬的物理分区变成灵活的逻辑卷。有了 LVM，你可以随时**动态调整**文件系统大小、**跨多块物理盘**创建卷、做**快照备份**——而不需要停机或移动数据。

# 为什么需要 LVM

- **动态扩容**：普通分区扩容需要备份→删分区→重建→恢复，LVM 在线执行
- **跨盘组合**：多块物理盘组成一个逻辑卷，突破单盘容量限制
- **快照**：瞬间创建文件系统的一致性快照，用于备份或测试
- **灵活缩容**：ext4 文件系统支持在线缩容（XFS 不支持）

# 核心概念

```
物理磁盘 → 分区 → PV（物理卷）→ VG（卷组）→ LV（逻辑卷）→ 文件系统 → 挂载
```

| 概念 | 缩写 | 说明 |
|------|------|------|
| 物理卷 | PV（Physical Volume） | 被 LVM 管理的磁盘或分区——初始化的第一步 |
| 卷组 | VG（Volume Group） | 由一个或多个 PV 组成的存储池 |
| 逻辑卷 | LV（Logical Volume） | 从 VG 中划分出的可用空间，最终在此之上创建文件系统 |

物理盘（PE）是 VG 的最小分配单位，默认 4MB。

# 创建流程

```shell
# 1. 创建物理卷（初始化磁盘/分区为 PV）
pvcreate /dev/sdb1 /dev/sdc1

# 2. 创建卷组（把 PV 加入 VG）
vgcreate vg_data /dev/sdb1 /dev/sdc1

# 3. 创建逻辑卷（从 VG 中划分空间）
lvcreate -n lv_mysql -L 50G vg_data

# 4. 格式化逻辑卷
mkfs.ext4 /dev/vg_data/lv_mysql

# 5. 挂载使用
mkdir /data/mysql
mount /dev/vg_data/lv_mysql /data/mysql
```

# 扩容

```shell
# 场景：lv_mysql 空间不足，需要扩容

# 方法一：VG 有剩余空间——直接扩 LV
lvextend -L +20G /dev/vg_data/lv_mysql
# 然后扩展文件系统（ext4）
resize2fs /dev/vg_data/lv_mysql
# 或 XFS
xfs_growfs /data/mysql

# 方法二：VG 空间也不够——先加新盘扩 VG，再扩 LV
pvcreate /dev/sdd1
vgextend vg_data /dev/sdd1
lvextend -L +20G /dev/vg_data/lv_mysql
resize2fs /dev/vg_data/lv_mysql

# 一步到位（-r 自动同步扩展文件系统）
lvextend -r -L +20G /dev/vg_data/lv_mysql
```

# 缩容

```shell
# XFS **不支持**缩容！只有 ext4 支持

# 1. 先卸载
umount /data/mysql

# 2. 检查文件系统
e2fsck -f /dev/vg_data/lv_mysql

# 3. 先缩文件系统
resize2fs /dev/vg_data/lv_mysql 30G

# 4. 再缩逻辑卷
lvreduce -L 30G /dev/vg_data/lv_mysql

# 5. 重新挂载
mount /dev/vg_data/lv_mysql /data/mysql
```

# 快照

```shell
# 创建快照（瞬间完成，只记录差异数据）
lvcreate -s -n lv_mysql_snap -L 5G /dev/vg_data/lv_mysql

# 挂载快照查看/备份
mount /dev/vg_data/lv_mysql_snap /mnt/snap

# 快照用完后删除
umount /mnt/snap
lvremove /dev/vg_data/lv_mysql_snap
```

> 快照只存储自创建后变化的块——因此快照大小通常远小于原始卷，5%-20% 即可。

# 常用命令速查

| 操作 | PV | VG | LV |
|------|----|----|-----|
| 创建 | `pvcreate` | `vgcreate` | `lvcreate` |
| 查看 | `pvdisplay` / `pvs` | `vgdisplay` / `vgs` | `lvdisplay` / `lvs` |
| 扩容 | - | `vgextend` | `lvextend` |
| 缩容 | - | `vgreduce` | `lvreduce` |
| 移除 | `pvremove` | `vgremove` | `lvremove` |
| 扫描 | `pvscan` | `vgscan` | `lvscan` |

> 提示：`pvs` / `vgs` / `lvs` 是紧凑格式，适合快速查看；`pvdisplay` / `vgdisplay` / `lvdisplay` 是详细格式。

# 注意

- XFS 文件系统**不支持缩容**——创建 LV 时如果未来可能缩容，选 ext4
- 删除 LV/VG/PV 前确保已卸载、数据已备份
- 快照不宜长期保留——累计差异数据会填满快照空间，快照满了会自动失效
- `/etc/fstab` 中使用 `/dev/mapper/vg_data-lv_mysql` 或 `/dev/vg_data/lv_mysql` 均可

> 返回 [存储管理基础](./存储管理基础.md)
