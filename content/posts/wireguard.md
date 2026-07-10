---
title: Wireguard配置
subtitle:
date: 2026-07-10T23:09:57+08:00
slug: 58dfa69
draft: false
description:
keywords:
weight: 0
categories:
  - 教程
collections:
  - VPN
tags:
  - DN42
  - Linux
  - wireguard
---

## 安装wireguard

```shell
sudo apt update && sudo apt install wireguard
```

## 计算最大MTU

```shell
ping <address> -M do -s <MTU>
```

底层网络可用MTU = MTU + 28

**WIREGUARD头部占用MTU** IPv4: 60 IPv6: 80

## 编写wireguard配置

生成privatekey: `wg genkey`

生成publickey: `echo '<privatekey>' | wg pubkey`

(可选)生成presharedkey: `wg genpsk`

### 示例配置

```shell
[Interface]
PrivateKey = yDkd/5geb29zHEOt4E8L1C5HGq7orVJzAF38rtAF6kU=
Address = fe80::306/64
ListenPort = 20298
MTU = 1420
Table = off

[Peer]
PublicKey = T6lJB0jlRViYxzWED/i0zxUW/4M8zgtvhRnBcLY1SC0=
#PresharedKey = wJPKdgWhbEDlKGuQba6SBaA8xE5HcNdEFl21HlupbAU=
Endpoint = hkg.server.net:51820
AllowedIPs = 0.0.0.0/0, ::/0

```
