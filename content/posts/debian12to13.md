---
title: debian12 升级 debian13 教程
subtitle:
date: 2026-10-05T00:42:42+08:00
slug: debian12update
draft: false
description: lxc使用教程
keywords: linux
weight: 0
categories:
  - 教程
collections:
  - linux
tags:
  - linux

---

## 更新debian12至最新

```shell
sudo apt update && sudo apt upgrade -y
```

## 替换debian源

```shell
sudo sed -i 's/bookworm/trixie/g' /etc/apt/sources.list /etc/apt/sources.list.d/*.list
```

## 更新并升级

```shell
sudo apt update
sudo apt upgrade -y
sleep 5
reboot
```

## 清理

```shell
sudo apt autoremove -y
```
