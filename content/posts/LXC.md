---
title: LXC
subtitle:
date: 2026-09-27T20:42:42+08:00
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

## lxc容器配置

```text
lxc.start.auto = 1 # 开机自启动

# 添加网卡直通
lxc.net.1.type = phys
lxc.net.1.link = enp4s0
lxc.net.1.flags = up
lxc.net.1.name = eth1
lxc.net.1.hwaddr = 02:72:d0:c6:ed:37 # mac=$(printf '02:%02x:%02x:%02x:%02x:%02x\n' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256))); echo "$mac"

# 添加内核模块映射
lxc.cgroup2.devices.allow = c 108:0 rwm
lxc.mount.entry = /dev/ppp dev/ppp none bind,create=file

lxc.cgroup2.devices.allow = c 10:200 rwm
lxc.mount.entry = /dev/net/tun dev/net/tun none bind,create=file

# 映射宿主机路径
lxc.mount.entry = /zfspool zfspool none rbind,create=dir

```

## 解决 Locale 警告

如果 LXC 中出现：

```text
perl: warning: Setting locale failed.
```

通常是 `zh_CN.UTF-8` locale 未生成导致的。

安装并生成 locale：

```bash
sudo apt install locales -y
sudo sed -i 's/^# *zh_CN.UTF-8 UTF-8/zh_CN.UTF-8 UTF-8/' /etc/locale.gen
sudo locale-gen
```

如果需要将中文设为默认 locale：

```bash
sudo update-locale LANG=zh_CN.UTF-8 LANGUAGE=zh_CN:zh LC_ALL=zh_CN.UTF-8
```

重新登录后检查：

```bash
locale
```
