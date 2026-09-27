---
title: tcpdump教程
subtitle:
date: 2026-07-11T22:16:42+08:00
slug: tcpdump
draft: false
description: tcpdump使用教程
keywords: tcpdump linux network
weight: 0
categories:
  - 教程
collections:
  - network
tags:
  - tcpdump
  - linux

---

## 安装 tcpdump

```shell
sudo apt update
sudo apt install tcpdump -y
```

## 使用

基本格式：

```shell
sudo tcpdump -e -n -i [interface] [filter]
```

常用参数：

| 参数               | 作用                   |
| ---------------- | -------------------- |
| `-i [interface]` | 指定监听的网卡              |
| `-e`             | 显示二层 Ethernet MAC 地址 |
| `-n`             | 不解析 IP 地址为域名         |
| `-nn`            | 不解析 IP 地址和端口号        |
| `-v`             | 显示更详细的信息             |
| `-vv` / `-vvv`   | 显示更详细的信息             |
| `-c [数量]`        | 抓取指定数量的数据包后退出        |
| `-X`             | 同时显示十六进制和 ASCII 内容   |
| `-w [文件]`        | 保存为 pcap 文件          |
| `-r [文件]`        | 读取 pcap 文件           |

例如：

```shell
sudo tcpdump -e -nn -i eth0
```

监听 `eth0` 上的所有流量，并显示 MAC 地址。

## 按 IP 筛选

### 指定主机

```shell
sudo tcpdump -nn -i eth0 host 192.168.1.100
```

源 IP 或目标 IP 为 `192.168.1.100`。

### 指定源 IP

```shell
sudo tcpdump -nn -i eth0 src host 192.168.1.100
```

### 指定目标 IP

```shell
sudo tcpdump -nn -i eth0 dst host 192.168.1.100
```

### 指定网段

```shell
sudo tcpdump -nn -i eth0 net 192.168.1.0/24
```

## 按端口筛选

### 指定端口

```shell
sudo tcpdump -nn -i eth0 port 443
```

源端口或目标端口为 `443`。

### 指定源端口

```shell
sudo tcpdump -nn -i eth0 src port 443
```

### 指定目标端口

```shell
sudo tcpdump -nn -i eth0 dst port 443
```

### 指定 TCP / UDP

```shell
sudo tcpdump -nn -i eth0 tcp port 443
sudo tcpdump -nn -i eth0 udp port 53
```

### 指定端口范围

```shell
sudo tcpdump -nn -i eth0 portrange 8000-9000
```

源端口或目标端口在 `8000-9000` 范围内。

也可以指定源或目标端口范围：

```shell
sudo tcpdump -nn -i eth0 src portrange 8000-9000
sudo tcpdump -nn -i eth0 dst portrange 8000-9000
```

或者组合协议：

```shell
sudo tcpdump -nn -i eth0 'tcp portrange 8000-9000'
```

## 按 MAC 地址筛选

### 指定 MAC

```shell
sudo tcpdump -e -nn -i eth0 ether host 00:11:22:33:44:55
```

源 MAC 或目标 MAC 为指定地址。

### 指定源 MAC

```shell
sudo tcpdump -e -nn -i eth0 ether src 00:11:22:33:44:55
```

### 指定目标 MAC

```shell
sudo tcpdump -e -nn -i eth0 ether dst 00:11:22:33:44:55
```

## 按协议筛选

### ICMP

```shell
sudo tcpdump -nn -i eth0 icmp
```

### ICMPv6

```shell
sudo tcpdump -nn -i eth0 icmp6
```

### TCP

```shell
sudo tcpdump -nn -i eth0 tcp
```

### UDP

```shell
sudo tcpdump -nn -i eth0 udp
```

## 多个条件组合

使用：

* `and` / `&&`：同时满足
* `or` / `||`：满足其中一个
* `not` / `!`：排除条件

### MAC + IP

```shell
sudo tcpdump -e -nn -i eth0 'ether host 00:11:22:33:44:55 and host 192.168.1.100'
```

### IP + TCP 443

```shell
sudo tcpdump -nn -i eth0 'host 192.168.1.100 and tcp port 443'
```

### MAC + ICMP

```shell
sudo tcpdump -e -nn -i eth0 'ether host 00:11:22:33:44:55 and icmp'
```

### 排除某个主机

```shell
sudo tcpdump -nn -i eth0 'not host 192.168.1.100'
```

### 多个端口

```shell
sudo tcpdump -nn -i eth0 'port 53 or port 443'
```

## 查看 TCP 建立连接

只看 TCP SYN：

```shell
sudo tcpdump -nn -i eth0 'tcp[tcpflags] & tcp-syn != 0'
```

只看 SYN，不看 SYN-ACK：

```shell
sudo tcpdump -nn -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'
```

## 保存和读取抓包

保存：

```shell
sudo tcpdump -i eth0 -w capture.pcap
```

读取：

```shell
tcpdump -nn -r capture.pcap
```

也可以在保存时直接进行筛选：

```shell
sudo tcpdump -i eth0 -w capture.pcap 'host 192.168.1.100'
```

## 常用示例

### 查看某个设备的全部流量

```shell
sudo tcpdump -e -nn -i eth0 'ether host 00:11:22:33:44:55'
```

### 查看某设备访问 HTTPS

```shell
sudo tcpdump -e -nn -i eth0 'ether host 00:11:22:33:44:55 and tcp port 443'
```

### 查看 DNS 请求

```shell
sudo tcpdump -nn -i eth0 'udp port 53 or tcp port 53'
```

### 查看 ICMP 请求

```shell
sudo tcpdump -nn -i eth0 icmp
```

### 抓取 100 个数据包后退出

```shell
sudo tcpdump -nn -i eth0 -c 100
```

### 查看所有网卡

```shell
sudo tcpdump -D
```

然后选择对应的网卡：

```shell
sudo tcpdump -nn -i eth0
```

> 注意：`tcpdump` 的筛选条件属于 **BPF（Berkeley Packet Filter）语法**。复杂条件建议使用单引号包起来，避免 `&&`、`||`、`!` 等字符被 Shell 自己解析。
