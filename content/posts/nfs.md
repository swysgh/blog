---
title: NFS搭建和维护
subtitle:
date: 2026-07-09T21:28:33+08:00
slug: 0ab03bb
draft: false
description: nfs搭建和后期维护记录
keywords: nfs 存储
weight: 0
categories:
  - draft
collections:
  - draft
tags:
  - draft

---

## NFS安装

```shell
sudo apt update
sudo apt install nfs-kernel-server -y
```

## 创建目录

```shell
mkdir -p /zfspool/share
chmod -R 0777 /zfspool/share
```

## 配置nfs共享

`sudo nano /etc/exports`

### 在文件里写入挂载的目录

`/zfspool  10.0.0.0/8(rw,sync,crossmnt,all_squash,anonuid=0,anongid=0,no_subtree_check,insecure)`

`/zfspool  10.0.0.0/8(rw,sync,crossmnt,no_subtree_check,insecure)`

生效配置 `exportfs -arv`

- /zfspool 分享出去的绝对路径

- 10.0.0.0/8 允许访问的客户端ip范围，如果为全部则*

- rw 允许读写

- sync 同步写入

- crossmnt 允许跨越挂载点：比如zfs根目录下的其他数据集

- no_subtree_check 禁用子树检查，提升性能，大幅提高文件频繁变动是的系统稳定性

- all_squash,anonuid=0,anonguid=0 强制映射用户为root

- insecure 允许客户端从大于 1024 的非特权端口连接服务端（方便过nat）
