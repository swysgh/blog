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
