---
title: ifupdown up to netplan
subtitle:
date: 2026-07-09T20:21:30+08:00
slug: ef9efaa
draft: false
description:
keywords:
weight: 0
categories:
  - 教程
collections:
  - 教程
tags:
  - network
  - netplan
  - ifupdown

---

# 从 ifupdown 到 Netplan：Linux 网络配置平滑迁移实战指南

随着 Ubuntu 18.04 LTS 及其后续版本的普及，传统使用 `/etc/network/interfaces` 的 **ifupdown** 工具已被全新的 **Netplan** 正式取代。

对于习惯了传统 Linux 网络配置的系统管理员和 DevOps 工程师来说，语法从“键值对/脚本命令”转向“严格缩进的 YAML”难免有些不习惯。本文将带你搞懂 Netplan 的核心设计逻辑，并通过对比和实战步骤，帮助你无痛完成网络配置迁移。

---

## 一、 Netplan 与 ifupdown 的核心差异

`ifupdown` 是直接通过命令行与底层网络接口交互的；而 **Netplan 本身并不直接管理网络**，它是一个**配置生成器（Abstract Configuration Generator）**。

你只需要编写干净的 YAML 配置文件，Netplan 会将其翻译并传递给底层的渲染引擎（Renderer）：

* **`systemd-networkd`**：适用于服务器（Server）环境，轻量高效。
* **`NetworkManager`**：适用于桌面（Desktop）环境，便于 GUI 和 Wi-Fi 动态管理。

| 配置特性 | ifupdown (`/etc/network/interfaces`) | Netplan (`/etc/netplan/*.yaml`) |
| --- | --- | --- |
| **文件格式** | 自定义结构化文本（脚本风格） | YAML 格式（严格缩进） |
| **底层引擎** | 直接控制网络设备接口 | systemd-networkd / NetworkManager |
| **测试机制** | 无（配置写错重启服务直接断网） | 提供 `netplan try`（带超时自动撤销） |
| **网关设置** | `gateway 192.168.1.1` | `routes:` 列表声明（旧版 `gateway4` 已弃用） |

---

## 二、 常用配置对齐与语法转换

在编写 Netplan 配置时，最关键的是注意 YAML 的缩进（必须使用空格，**严禁使用 Tab 键**）。

### 1. 场景一：DHCP 动态获取 IP

* **旧版 ifupdown (`/etc/network/interfaces`)：**
```text
auto eth0
iface eth0 inet dhcp

```


* **新版 Netplan (`/etc/netplan/01-netcfg.yaml`)：**
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true

```



---

### 2. 场景二：静态 IP + 网关 + DNS

> 💡 **专家提示**：Netplan 在新版本中已弃用 `gateway4` / `gateway6` 关键字，推荐统一采用标准的 `routes` 结构声明默认网关。

* **旧版 ifupdown (`/etc/network/interfaces`)：**
```text
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1

```


* **新版 Netplan (`/etc/netplan/01-netcfg.yaml`)：**
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1

```



---

## 三、 零中断平滑迁移步骤

对于远程 SSH 连入的服务器，直接修改网络配置非常容易因语法错误导致失联。遵循以下操作流程可极大降低失联风险：

1. **备份现有网络配置:** 防止迁移失败时无法还原网络环境.
在进行任何更改前，请将原有的 `/etc/network/interfaces` 文件备份至安全位置：

```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.bak

```


2. **创建 Netplan 配置文件:** 注意：YAML 文件必须以 .yaml 结尾.
在 `/etc/netplan/` 目录下新建配置文件（例如 `01-netcfg.yaml`）：

```bash
sudo nano /etc/netplan/01-netcfg.yaml

```

参照上文语法填入对应的网卡配置，保存并退出。


3. **使用 netplan try 安全测试:** 非常重要：内置防失联安全机制.
执行以下命令测试配置：

```bash
sudo netplan try --timeout 120

```

> **为什么推荐 `netplan try`？**
> 系统会临时应用该配置并开启倒计时（默认 120 秒）。如果在倒计时内你没有按下 `ENTER` 键确认（例如由于配置错误导致 SSH 连接断开），Netplan 会**自动回滚**到旧配置，防止远程服务器失联。


4. **清理 ifupdown 并应用配置:** 避免新旧网络服务冲突.
确认连接正常后，清空或注释掉 `/etc/network/interfaces` 中的网卡定义（仅保留 `lo` 回环接口），然后停用旧服务并启用 Netplan：

```bash
sudo systemctl stop networking
sudo systemctl disable networking
sudo netplan apply

```


---

## 四、 常见踩坑与 Debug 指南

1. **`Invalid YAML at ...` 语法报错**
* **原因**：YAML 中使用了 Tab 缩进或冒号后缺少空格。
* **解决**：检查空格缩进，格式应为 `key: value`（冒号后必须留空格）。


2. **`netplan apply` 提示权限问题**
* **解决**：Netplan 配置文件建议将权限设置为最高安全的 `600`：
```bash
sudo chmod 600 /etc/netplan/*.yaml

```




3. **想查看翻译后的底层系统配置？**
* 如果后端使用的是 `systemd-networkd`，可以在 `/run/systemd/network/` 目录下找到 Netplan 自动生成的 `.network` 文件进行排查。



---

## 总结

从 `ifupdown` 迁移到 `Netplan` 刚开始可能会觉得 YAML 的缩进要求偏严格，但 Netplan 带来的结构化声明式设计、强大的安全测试机制（`netplan try`）以及统一的配置抽象，极大简化了复杂网络的配置难度。
