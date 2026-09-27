---
title: LXC
subtitle:
date: 2026-07-11T22:16:42+08:00
slug: LXC
draft: false
description: lxc使用教程
keywords: lxc linux
weight: 0
categories:
  - 教程
collections:
  - vm
tags:
  - lxc
  - vm
  - linux

---

## 安装lxc

```shell
sudo apt update
sudo apt install lxc
```

**检查安装** `lxc-checkconfig`

## 创建容器

```shell
sudo lxc-create -n test -t download
sudo lxc-create -n test -t download -- --dist debian --release trixie --arch amd64
```

## 容器管理

```shell
sudo lxc-start -n test
sudo lxc-ls -f
sudo lxc-attach -n test
sudo lxc-stop -n test
sudo lxc-stop -n test
```

## 配置文件

位置 `/etc/lxc/lxc.conf`  
修改lxc默认文件位置 `lxc.lxcpath = /zfspool/lxc`
