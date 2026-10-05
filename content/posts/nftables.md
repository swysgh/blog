---
title: nftables使用教程
subtitle:
date: 2026-10-06T04:30:00+08:00
slug: nftables
draft: false
description: nftables 的核心概念、规则语法、集合、NAT，以及配置文件化部署与排错
keywords: nftables linux firewall nat 防火墙
weight: 0
categories:
  - 教程
collections:
  - network
tags:
  - nftables
  - linux
  - network
  - 防火墙

---

Debian 10（Buster）开始，nftables 就是系统里**默认且推荐**的防火墙框架，iptables 退化成兼容层。这篇按「概念 → 命令行 → 规则语法 → NAT → 配置文件化 → 排错」的顺序把 nftables 过一遍，重点放在**为什么这么写**和**写错了会怎样**上。

## 一、nftables 与 iptables 的区别

nftables 不是 iptables 的语法糖，而是换了一套内核子系统（`nf_tables`，旧的叫 `x_tables`）。这不是"新版 iptables"，两者的对象互相看不见。

| | iptables | nftables |
| --- | --- | --- |
| 内核子系统 | x_tables | nf_tables |
| 命令 | iptables / ip6tables / arptables / ebtables 各一个 | 一个 `nft` 管所有地址族 |
| 预置表与链 | 内置 filter/nat/mangle 表和 INPUT/OUTPUT/FORWARD 链 | 什么都不预置，表和链全部自己建 |
| 一条规则的动作用数 | 只能有一个 `-j 目标` | 可以一串：`counter log accept` |
| 集合 | 要额外装 ipset | 内核自带 set / map / vmap |
| 加载规则集 | `iptables-restore` 逐条提交 | `nft -f` 整个规则集**一个事务**替换 |

最后一条是实际运维里差别最大的地方：`nft -f` 会把文件全部读进内存、构建好新规则集，然后**一次性**和旧规则集交换。所以在加载过程中不存在"防火墙处于半配置状态"的窗口，也不会出现某几条规则生效、某几条还没生效的瞬间。

## 二、四个基本概念

nftables 的对象层次是：**family（地址族）→ table（表）→ chain（链）→ rule（规则）**。

### 2.1 family：地址族

每个表必须属于且只属于一个 family：

| family | 处理什么 | 可用 hook |
| --- | --- | --- |
| `ip` | 仅 IPv4 | prerouting / input / forward / output / postrouting |
| `ip6` | 仅 IPv6 | 同上 |
| `inet` | IPv4 + IPv6 混合 | 同上，另有 ingress（内核 5.10+） |
| `arp` | ARP 报文 | input / output |
| `bridge` | 过桥的二层帧 | prerouting / input / forward / output / postrouting |
| `netdev` | 网卡驱动刚收上来的包 | ingress / egress（必须指定 `device`） |

不写 family 时默认是 `ip`。**新写配置直接用 `inet`**：man 手册里说 inet 是"用来建 IPv4/IPv6 混合表的虚拟族"，一份规则同时管两栈，不用像 iptables 那样 ip/ip6 各写一遍。要区分当前包是 v4 还是 v6，用 `meta nfproto`。

### 2.2 table：表

链、集合、map、流表的容器，只起归类作用，没有语义。名字随便取，但**建议一个用途一张表**（比如 `nat`、`firewall`、`tproxy`），因为 `flush table xxx` 是按表整体清的。

### 2.3 chain：链，分两种

**base chain（基链）**：注册到 Netfilter hook 上，能看到流经的包。定义时必须带上这段花括号：

```shell
add chain inet filter input { type filter hook input priority 0; policy accept; }
```

写成声明式就是：

```shell
chain input {
    type filter hook input priority filter; policy accept;
}
```

**regular chain（普通链）**：不带 hook，自己看不到任何包，只能被别的链 `jump` / `goto` 调用。它相当于 iptables 里的自定义链，用来把规则拆成树。

> 官方文档专门警告过这个坑：`nft add chain inet filter foo` 不带花括号，建出来的是普通链，**不挂 hook、收不到任何包**。配置里链看起来没问题、就是没效果，多半是这里漏了。

#### hook：包在什么阶段被看到

一个包在内核里的路径大致是：

```text
入包 ──> [ingress] ──> [prerouting] ──> 路由决策 ──┬──> [input] ──> 本地进程 ──> [output] ──┐
                                                  │                                      │
                                                  └──> [forward] ────────────────────────┤
                                                                                         v
出包 <──────────────────────────────────────────── [postrouting] <───────────────────────┘
```

| hook | 看到什么包 |
| --- | --- |
| `ingress` | 网卡驱动刚收上来、还没进协议栈的包（netdev 族；inet 族内核 5.10 起也支持） |
| `prerouting` | 所有入包，路由决策**之前**，包含要转发的 |
| `input` | 已经路由到本机、交给本地进程的包 |
| `forward` | 不是给本机、要转出去的包 |
| `output` | 本机进程产生的包 |
| `postrouting` | 所有出包，路由之后、发出去之前 |

#### chain type：链的用途

| type | 干什么 | 支持的 family | 可用 hook |
| --- | --- | --- | --- |
| `filter` | 过滤 | ip / ip6 / inet / arp / bridge（netdev 用 filter + ingress/egress） | 全部 |
| `route` | 改包后重新查路由（策略路由） | ip / ip6 / inet | 只有 output |
| `nat` | 地址转换 | ip / ip6 / inet | prerouting / input / output / postrouting |

**`nat` 类型的链只有一条连接的第一个包会走**，后续包直接跳过（因为转换结果已经记在 conntrack 里了）。所以别拿 nat 链做过滤——它统计不到真实流量。

#### priority：同 hook 上多条链的顺序

priority 是个整数，也可以写名字，名字只是常用值的别名：

| 名字 | 数值 | 大致对应 |
| --- | --- | --- |
| `raw` | -300 | 连接跟踪之前 |
| `mangle` | -150 | 改包 |
| `dstnat` | -100 | 目的地址转换 |
| `filter` | 0 | 常规过滤（默认） |
| `security` | 50 | SELinux 之类 |
| `srcnat` | 100 | 源地址转换 |

同一个 hook 上的多条链，按 priority **从小到大**依次遍历。这里有个必须记住的行为：

> **`drop` 是终局，`accept` 不是。** 一个包被 accept 之后，如果同 hook 上还有 priority 更大的链，它**还会继续走那条链**。所以下面这种配置会把所有入包全丢光：

```shell
table inet filter {
    chain services {
        type filter hook input priority 0; policy accept;
        tcp dport 22 accept          # accept 只是"这条链放行"
    }
    chain input {
        type filter hook input priority 1; policy drop;   # 包还会到这里，然后被丢
    }
}
```

#### policy：链的默认判决

`policy accept`（默认）或 `policy drop`。drop 表示"规则都不匹配、走到链尾的包一律丢弃"，是"白名单"式防火墙的基础。注意 drop policy 的链里，至少要放行 `ct state established,related`、`lo` 接口、必要的 ICMP，否则很容易把自己（尤其是 SSH 会话）锁在外面。

### 2.4 rule：规则

一条规则 = 零个或多个**匹配表达式** + 一个或多个**语句**。

- 匹配从左到右依次求值，前面的不匹配后面的就不看了（和 `&&` 一样）。
- 全部匹配后执行语句，语句也是从左到右。
- 判决语句（`accept` / `drop` / `return` / `jump` …）本身会结束这条规则。

```shell
tcp dport 22 ct state new log prefix "new ssh: " counter accept
```

这一条就干了四件事：匹配新 SSH 连接 → 打日志 → 计数 → 放行。iptables 里这需要两条规则。

## 三、命令行操作

### 3.1 表和链

```shell
nft add table inet firewall              # 建表
nft list tables                          # 列出所有表
nft list table inet firewall             # 看某张表
nft flush table inet firewall            # 清空表里所有规则（不清集合）
nft delete table inet firewall           # 删表连同内容（内核 3.18+）

nft add chain inet firewall input { type filter hook input priority filter\; policy accept\; }
nft flush chain inet firewall input      # 只清某条链
nft delete chain inet firewall input     # 删链（链必须是空的）
```

命令行里 `;` 是 shell 的元字符，所以整条语句要用单引号包住，或者把 `;` 转义成 `\;`。写进文件时不存在这个问题。

### 3.2 规则

```shell
nft add rule inet firewall input tcp dport 22 accept          # 追加到链尾
nft insert rule inet firewall input tcp dport 22 accept       # 插到链首
nft add rule inet firewall input position 8 tcp dport 22 drop # 插到 handle 8 之后
nft replace rule inet firewall input handle 8 counter         # 替换 handle 8 的规则
nft delete rule inet firewall input handle 8                  # 删除 handle 8
nft flush chain inet firewall input                           # 清空整条链
```

**handle** 是内核给每条规则分配的编号，用 `-a` 才显示：

```shell
nft -a list ruleset
```

```text
table inet firewall {
    chain input {
        type filter hook input priority filter; policy accept;
        tcp dport 22 accept # handle 4
        ip saddr 10.0.0.5 drop # handle 5
    }
}
```

删规则、按位置插规则都要用 handle。目前**还不支持按内容删规则**（`nft delete rule ... ip saddr 1.2.3.4` 这种写法官方标着"未实现"），所以要么用 handle，要么改配置文件整体重载——后者在配置文件化部署里更常见。

### 3.3 输出修饰符

`nft list` 的输出可以按需加工，调规则的时候很有用：

| 参数 | 作用 |
| --- | --- |
| `-a` | 显示规则 handle |
| `-n` | 全数字输出（不解析服务名/协议名） |
| `-y` | priority 显示成数字而不是名字 |
| `-p` | 协议显示成数字 |
| `-S` | 端口显示成 /etc/services 里的服务名 |
| `-N` | 反查 IP 的域名（会发 DNS 请求，可能变慢） |
| `-s` | 不显示计数等状态信息 |
| `-t` | 不显示集合内容 |

### 3.4 交互模式

```shell
nft -i
```

进入后逐条敲命令，用 `quit` 或 Ctrl-D 退出。想试探语法又不想反复写引号时很方便。

## 四、规则怎么写

### 4.1 常用匹配

| 表达式 | 匹配什么 |
| --- | --- |
| `iifname "eth0"` / `oifname "ppp0"` | 入/出接口名（字符串） |
| `iif` / `oif` | 入/出接口索引（数字） |
| `ip saddr` / `ip daddr` | IPv4 源/目的地址，支持 CIDR、区间、集合 |
| `ip6 saddr` / `ip6 daddr` | IPv6 源/目的地址 |
| `ether saddr` / `ether daddr` | 二层 MAC 地址 |
| `tcp dport` / `tcp sport` | TCP 目的/源端口 |
| `udp dport` / `udp sport` | UDP 目的/源端口 |
| `icmp type` / `icmpv6 type` | ICMP / ICMPv6 类型 |
| `meta l4proto` | 四层协议（跨 v4/v6 都能用） |
| `th dport` / `th sport` | 传输层头的端口 |
| `ct state` | 连接状态：`new` / `established` / `related` / `invalid` |
| `meta mark` | netfilter mark |
| `meta skuid` / `meta skgid` | 本机发包进程的 UID/GID |
| `meta nfproto` | 当前包是 IPv4 还是 IPv6 |
| `fib daddr type local` | 目的地址属于本机 |

几个容易写错的点：

**端口必须跟协议。** nftables 里**没有** `ip dport` 这种东西，端口是 TCP/UDP 头里的字段：

```shell
tcp dport 22 accept                    # 对
udp dport 53 accept                    # 对
ip dport 22 accept                     # 报错：语法都不成立
meta l4proto { tcp, udp } th dport 53  # 想同时匹配 TCP/UDP 时这么写
```

**跨 v4/v6 匹配四层协议用 `meta l4proto`。** 在 `inet` 表里 `ip protocol` 只对 IPv4 报文有意义，`ip6 nexthdr` 只对 IPv6 有意义。man 手册里明确写了 `meta l4proto` 的用途就是"匹配属于 IPv4 或 IPv6 报文的传输层协议"，而且它**会跳过 IPv6 扩展头**（`ip6 nexthdr` 在带扩展头的包上匹配到的是扩展头，不是真正的传输层协议）。

**`iifname` 比 `iif` 稳。** `iif` 匹配的是接口索引，网卡删了重建索引就变了；配置里写接口名更不容易出问题。

### 4.2 常用语句

判决：

| 语句 | 含义 |
| --- | --- |
| `accept` | 放行（不等于"这条链后面的链不看了"） |
| `drop` | 丢弃，立即生效，后面的链和规则都不再执行 |
| `reject` | 回一个错误包再丢；默认回 ICMP port-unreachable |
| `return` | 结束当前链，回到调用它的链继续 |
| `jump` / `goto` | 跳到普通链；`goto` 不返回 |
| `queue` | 交给用户态程序处理 |

`reject` 在 `inet` 表里要用 `icmpx`（专门为混合族设计的）：

```shell
reject with icmpx type admin-prohibited
reject with tcp reset
```

非判决语句，可以和判决写在同一条规则里：

| 语句 | 含义 |
| --- | --- |
| `counter` | 计数，`nft list` 时能看到包数/字节数 |
| `log prefix "xxx: "` | 打日志，`level` 默认 `warn` |
| `limit rate 10/second burst 5 packets` | 限速，常配 `log` 防日志刷屏 |
| `meta mark set 0x1` | 打 mark |
| `masquerade` / `snat to` / `dnat to` | NAT |
| `tproxy to :12345` | 透明代理 |

组合示例：

```shell
# 丢弃并记录（限速，避免刷爆日志）
tcp dport 23 log prefix "nft-telnet: " limit rate 5/minute drop

# 新连接记日志并计数
tcp dport 22 ct state new log prefix "new ssh: " counter accept
```

### 4.3 集合与变量

**匿名集合**：直接写在规则里，适合固定不变的清单。

```shell
tcp dport { 22, 80, 443 } accept
ip saddr { 192.168.1.0/24, 10.0.0.0/8 } accept
```

匿名集合和规则绑定，规则删了集合就没了，也不能单独往里加元素。

**define 变量**：给一组值起名字，文件里用 `$名字` 引用，可以嵌套。

```shell
define LAN = { 192.168.1.0/24, 192.168.2.0/24 }
define DNS_SERVERS = { 1.1.1.1, 8.8.8.8 }
define TRUSTED = { $LAN, $DNS_SERVERS }

ip saddr $TRUSTED accept
```

**命名集合**：可以运行时增删元素，功能上取代了 iptables 时代的 ipset。

```shell
nft add set inet firewall blackhole { type ipv4_addr\; flags interval\; }
nft add element inet firewall blackhole { 203.0.113.0/24 }
nft delete element inet firewall blackhole { 203.0.113.0/24 }
nft list set inet firewall blackhole
nft get element inet firewall blackhole { 203.0.113.1 }
```

规则里用 `@名字` 引用：

```shell
ip saddr @blackhole drop
```

配置文件里的写法：

```shell
table inet firewall {
    set blackhole {
        type ipv4_addr
        flags interval
        auto-merge
        elements = { 203.0.113.0/24, 198.51.100.7 }
    }

    chain input {
        type filter hook input priority filter; policy accept;
        ip saddr @blackhole drop
    }
}
```

命名集合的属性：

| 属性 | 说明 |
| --- | --- |
| `type` | 元素类型：`ipv4_addr` / `ipv6_addr` / `ether_addr` / `inet_proto` / `inet_service` / `mark` / `ifname` |
| `typeof` | 用表达式让 nft 自己推断类型（0.9.4+），如 `typeof ip saddr` |
| `flags interval` | 集合里存的是区间/CIDR |
| `auto-merge` | 自动合并重叠/相邻元素，只对 interval 集合有效 |
| `timeout 3h45s` | 元素超时自动删除 |
| `size 1000` | 元素数量上限 |
| `policy memory` | 选择省内存（默认 `performance`） |
| `counter` | 每个元素单独计数 |

两个细节：**集合名字不能超过 16 个字符**；interval 集合里元素重叠（比如同时写了 `10.0.0.1` 和 `10.0.0.0/8`）会直接报 `conflicting intervals specified` 加载失败，加 `auto-merge` 就会自动收敛成 `10.0.0.0/8`。

**map / vmap**：把匹配到的值映射成别的值（map）或者判决（vmap），相当于内置的查表跳转。注意 **map 得到的是一个值，要交给别的语句使用；要直接映射判决只能用 vmap**。

```shell
# vmap：不同端口给不同判决
tcp dport vmap { 22 : accept, 23 : drop, 3306 : drop }

# map：按源地址决定 SNAT 到哪个出口地址
snat ip to ip saddr map { 192.168.1.0/24 : 203.0.113.2, 192.168.2.0/24 : 203.0.113.3 }
```

写成 `tcp dport map { 22 : accept }` 是错的（语法报错），判决用 `vmap`、取值用 `map`。

## 五、NAT 与转发

做 NAT 之前先确认内核开了转发：

```shell
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding
```

要转发（`forward` hook 上有包）才需要 NAT；`filter` 类型的 forward 链默认是 accept，所以只要不设 `policy drop` 就不拦。

```shell
table inet nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        # 把公网进来的 8080 转到内网主机（inet 表里必须写 dnat ip to，否则地址族有歧义）
        iifname "ppp0" tcp dport 8080 dnat ip to 192.168.1.10:80
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        # 内网出网伪装成出口地址
        ip saddr 192.168.1.0/24 oifname "ppp0" masquerade
    }
}
```

`masquerade` 和 `snat to` 的区别：`masquerade` 每次按出口接口当前的地址动态取源地址，适合 PPPoE 这种地址会变的场景，代价是每个新连接都要查一次出口地址；出口地址固定时用 `snat to <地址>` 更明确也更快。

再提醒一次：**nat 链只处理每条连接的第一个包**，后续包不会经过这里。所以"统计某个端口的流量"、"封禁某个来源"这类事，都应该放在 `filter` 类型的链里做。

## 六、落地成配置文件

命令行一条条敲的东西重启就没了，生产上都是写成一个规则集文件。文件格式就是 nft 自己的语法，首行加上 shebang 之后可以直接当脚本执行：

```shell
#!/usr/sbin/nft -f

flush ruleset
```

然后用 `nft -f 文件` 加载，或者 `chmod +x` 之后直接跑。

**开头为什么要 `flush ruleset`**：`nft -f` 把文件里所有命令作为一个事务提交，如果不清空，第二次加载就变成"在已有规则上再追加一遍"，规则会翻倍。加上 `flush ruleset` 之后，清空和加载在同一个事务里完成，仍然没有中间状态。注意 `flush table xxx` 不会清掉表里的集合，要彻底清就用 `flush ruleset`。

**加载前先校验**：

```shell
nft -c -f /etc/nftables.conf
```

`-c` / `--check` 只做语法和语义检查，不会下发任何改动。改完配置先跑一遍，比加载失败后从 journal 里翻报错省事得多。

### 6.1 Debian 上的持久化

```shell
apt install nftables
systemctl enable --now nftables
```

Debian 的包会装一个 `/etc/nftables.conf`，内容是空壳（三条没有任何规则的链）：

```shell
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority filter;
    }
    chain forward {
        type filter hook forward priority filter;
    }
    chain output {
        type filter hook output priority filter;
    }
}
```

`nftables.service` 是 oneshot 类型，开机时执行一次，关键就三行：

```shell
ExecStart=/usr/sbin/nft -f /etc/nftables.conf
ExecReload=/usr/sbin/nft -f /etc/nftables.conf
ExecStop=/usr/sbin/nft flush ruleset
```

也就是说改完 `/etc/nftables.conf` 之后：

```shell
nft -c -f /etc/nftables.conf && systemctl reload nftables
```

就完成了一次原子替换，不用重启。

### 6.2 例子：服务器最小防火墙

```shell
#!/usr/sbin/nft -f

flush ruleset

table inet firewall {
    chain input {
        type filter hook input priority filter; policy drop;

        # 已建立的连接和关联连接（必须放行，否则自己把自己锁外面）
        ct state established,related accept
        ct state invalid drop

        # 回环
        iifname "lo" accept

        # ICMP / ICMPv6：ping 和路径 MTU 发现
        ip protocol icmp icmp type { echo-request, destination-unreachable, time-exceeded } accept
        ip6 nexthdr icmpv6 icmpv6 type { echo-request, nd-neighbor-solicit, nd-neighbor-advert, nd-router-advert, packet-too-big, destination-unreachable } accept

        # 对外开放的端口
        tcp dport { 22, 80, 443 } accept

        # 其它一律丢弃并记录（限速防刷日志）
        log prefix "nft-input-drop: " limit rate 10/minute
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

### 6.3 例子：默认放行 + 单点封禁

不是所有场景都适合白名单。路由器、内网机器更常见的是"默认放行，只挡几个已知不需要的口子"，这时 policy 保持 accept，规则里只写要挡的：

```shell
table inet firewall {
    chain input {
        type filter hook input priority filter; policy accept;

        # 默认全放行，只挡从 WAN 口进来的管理端口
        iifname "ppp0" tcp dport 22 drop
    }
}
```

这种写法的好处是改错了也不会把自己关在门外，代价是"放行"是隐式的，需要你自己清楚哪些口不该开。

## 七、排错

**规则到底有没有命中**——给规则加 `counter`，然后看计数：

```shell
nft -a list ruleset
```

```text
tcp dport 22 ct state new counter packets 137 bytes 9856 accept # handle 12
```

计数不动说明前面的规则先匹配走了，或者这条链根本没看到包（比如挂错 hook）。

**看被丢掉的包**——加 `log` 语句：

```shell
log prefix "nft-drop: " level warn
```

然后：

```shell
journalctl -k -g "nft-drop"        # 或 dmesg
```

注意 `log` 也是一条普通语句，位置决定它记录的是什么。下面这条规则会记录**所有从 lo 进来的包**，而不是只有 22 端口的：

```shell
iif lo log tcp dport 22 accept     # 陷阱：log 在端口匹配之前
```

想只记 22 端口，把 `log` 放到匹配后面：`iif lo tcp dport 22 log accept`。

**看规则集变更**：

```shell
nft monitor
```

实时输出规则、链、集合的增删事件，调试脚本或者排查"谁在改我的防火墙"时很有用。

**让 nft 帮你优化**：

```shell
nft -c -o -f /etc/nftables.conf    # 只看优化建议，不生效
nft -o -f /etc/nftables.conf       # 实际加载优化后的规则集
```

它会把一堆同字段的线性匹配合并成集合查找，规则多的时候效果明显。

**导出与备份**：

```shell
nft list ruleset > ruleset.nft     # 可以直接被 nft -f 读回
nft -j list ruleset                # JSON 格式，给程序读
```

**和 tcpdump 对照**：包没到就查路由/接口，包到了但被丢就查 counter 和 log，能快速区分"防火墙问题"和"根本不是防火墙的问题"。

## 八、常见坑

1. **没有 `ip dport`**，端口必须跟协议写：`tcp dport` / `udp dport`，或者 `meta l4proto { tcp, udp } th dport`。
2. **`inet` 表里别用 `ip protocol` 匹配四层协议**，它只对 IPv4 有意义，跨栈要用 `meta l4proto`。
3. **`nft add chain` 不带 `{ type ... hook ... }`** 建的是普通链，永远看不到包。
4. **`accept` 不是终局，`drop` 才是**。同 hook 上 priority 更大的链还会看到被 accept 的包。
5. **`flush table` 不清集合**，要彻底清用 `flush ruleset`。
6. **命令行里的分号要引号包住**：`nft 'add rule ...'`，否则被 shell 吃掉。
7. **`INPUT` / `OUTPUT` 只是普通名字**，nftables 没有内置链，大写只是从 iptables 带过来的习惯，不带 hook 一样收不到包。
8. **不要混用 iptables-legacy 和 nft**：两套内核子系统互相看不见。iptables-nft 写出来的对象其实就是 nf_tables 对象，`nft list ruleset` 能看到；而 legacy 那套是另一回事。
9. **改完 `/etc/nftables.conf` 要 reload**，只 `nft -f` 加载一次重启就没了；反过来只改文件不 reload 也不生效。
10. **`policy drop` 之前先想好放行什么**：`ct state established,related`、`lo`、ICMP（IPv6 还要 nd-* 邻居发现，否则 IPv6 直接不通）漏一个就够难受。
11. **删规则、按位置插规则都要 handle**，目前不支持按内容删规则。
12. **同一个 hook 上 priority 相同的两条 base chain，执行顺序没有保证**，别依赖它们之间的先后关系。
13. **`inet` 表里做 NAT 要指明地址族**：写 `dnat ip to 192.168.1.10:80`，只写 `dnat to ...` 会报 `specify 'dnat ip' or 'dnat ip6' in inet table to disambiguate`。
14. **判决映射用 `vmap`，取值映射用 `map`**：`tcp dport vmap { 22 : accept }` 对，`tcp dport map { 22 : accept }` 语法报错；`map` 取出来的值要交给别的语句用，比如 `snat ip to ip saddr map { ... }`。

## 参考

- [nftables wiki：Quick reference-nftables in 10 minutes](https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes)
- [nftables wiki：Configuring chains](https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains)
- [nftables wiki：Simple rule management](https://wiki.nftables.org/wiki-nftables/index.php/Simple_rule_management)
- [nftables wiki：Sets](https://wiki.nftables.org/wiki-nftables/index.php/Sets)
- [nftables wiki：Atomic rule replacement](https://wiki.nftables.org/wiki-nftables/index.php/Atomic_rule_replacement)
- [nftables wiki：Logging traffic](https://wiki.nftables.org/wiki-nftables/index.php/Logging_traffic)
- [nftables wiki：Moving from iptables to nftables](https://wiki.nftables.org/wiki-nftables/index.php/Moving_from_iptables_to_nftables)
- [nft(8) man page（Debian trixie）](https://manpages.debian.org/trixie/nftables/nft.8.en.html)
- [Debian wiki：nftables](https://wiki.debian.org/nftables)
