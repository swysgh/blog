---
title: Bird2
subtitle:
date: 2026-07-10T00:39:40+08:00
slug: 79d1c20
draft: false
description:
keywords:
weight: 0
categories:
  - 教程
collections:
  - 路由
tags:
  - bird2
  - linux
  - dn42
---

## 安装

```shell
sudo apt update
sudo apt install apt-transport-https ca-certificates wget -y

sudo wget -O /etc/apt/keyrings/cznic-labs-pkg.gpg https://pkg.labs.nic.cz/gpg

echo "deb [signed-by=/etc/apt/keyrings/cznic-labs-pkg.gpg] https://pkg.labs.nic.cz/bird2 trixie main" | sudo tee /etc/apt/sources.list.d/cznic-labs-bird2.list 

sudo apt update && sudo apt install bird2
```

## 配置

`birdc interface` 显示bird识别到的接口

`birdc show route` 显示bird内部的路由表

`birdc show protocol` 显示协议

`birdc show ospf` 显示ospf相关
