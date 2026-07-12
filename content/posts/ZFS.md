---
title: ZFS
subtitle:
date: 2026-07-09T21:28:33+08:00
slug: 0ab03bb
draft: false
description: zfs搭建和后期维护记录
keywords: zfs 存储
weight: 0
categories:
  - 教程
collections:
  - 存储
tags:
  - zfs
  - 存储
  - linux

---

# ZFS安装

```shell
sudo apt update
sudo nano /etc/apt/sources.list.d/debian.sources
Components: main contrib non-free-firmware
sudo apt install zfsutils -y
```

# ZFS相关命令

## 存储池相关

```shell
# 创建/销毁/初始化存储池
zpool create [-fnd] [-o property=value] ...
      [-O file-system-property=value] ...
      [-m mountpoint] [-R root] <pool> <vdev> ...
zpool destroy [-f] <pool>
zpool initialize [-c | -s] [-w] <pool> [<device> ...]

# 添加/移除/更新磁盘设备
zpool add [-fgLnP] [-o property=value] <pool> <vdev> ...
zpool remove <pool> <device> ...
zpool upgrade
zpool upgrade [-v]

# 导入/导出存储池
zpool import [-d dir] [-D]
zpool export [-af] <pool> ...

# 将所有内存中的脏数据写入主存储池
zpool sync [pool] ...
```
## 查看存储池状态

```shell
# 查看池状态
zpool status [-x]
zpool list [pool]
zpool get [all,size,health,guid,free...]
# 查看修改
zpool history [pool]
```

```shell
# 对应信息解释
$ sudo zpool status
  pool: data2   # 池的名称
 state: ONLINE  # 池的当前运行状况
  scan: scrub repaired 0 in 0h39m with 0 errors on Sun Dec  8 01:03:48 2019

config: # 发出读取请求时出现I/O错误|发出写入请求时出现I/O错误|校验和错误
  NAME        STATE     READ WRITE CKSUM
  data2       ONLINE       0     0     0
    vdd       ONLINE       0     0     0
    vde       ONLINE       0     0     0

errors: No known data errors  # 确定是否存在已知的数据错误
```

# 快照相关

```shell
# 创建快照
zfs snapshot [-r] pool[/data]@snapname

# 查看快照
# 列出系统内所有的 ZFS 快照
zfs list -t snapshot
# 仅查看某个特定数据集下的快照
zfs list -t snapshot -r zfspool/prod
# 定制输出列（查看快照名、创建时间、实际占用空间）
zfs list -t snapshot -o name,creation,used

# 销毁快照
zfs destroy [-r] pool[/data]@snapname
# 批量删除老旧快照（区间删除）
# 彻底删除 @snap_old 到 @snap_new 之间的所有中间快照（包含 old，不包含 new）
zfs destroy zfspool/data@snap_old%snap_new

# 回滚到最近的一个快照（-r连续回滚）
zfs rollback zfspool/data@today

# 将快照克隆为一个可读写的新数据集 /zfspool/test-zone
zfs clone zfspool/data@today zfspool/test-zone

# 将全量快照流导出为本地文件（可用于冷备份）
zfs send zfspool/data@today > /media/usb/data.zfs
# 递归发送（带大写 -R）：连同所有子数据集、克隆体、以及相关的其他快照结构一起完整打包
zfs send -R zfspool/data@today | ssh 远程节点 zfs recv ...
# 增量发送 仅把 @today 到 @tomorrow 之间被修改过的数据块发送过去（体积极小、传输极快）
zfs send -i zfspool/data@today zfspool/data@tomorrow | ssh 远程节点 zfs recv ...
```

## 恢复备份

### 方法一 .zfs

`zfs set snapdir=visible zfspool/data` 开启.zfs

`cd /zfspool/data/.zfs/snapshot/`

`ls -l`

### 方法二 挂载快照

`mkdir -p /mnt/my_readonly_snap`

`mount -t zfs -o ro zfspool/prod-data@backup /mnt/my_readonly_snap`

`umount /mnt/my_readonly_snap`
