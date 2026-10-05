---
title: systemd 服务管理入门
subtitle: 从写一个 unit 文件到接管你自己的服务
date: 2026-10-05T20:45:00+08:00
slug: systemd
draft: false
description: 用 systemd 管理自己的进程：unit 文件、依赖顺序、重启策略与 timer 定时任务
keywords: systemd linux 服务管理
weight: 0
categories:
  - 教程
collections:
  - linux
tags:
  - systemd
  - linux
  - 服务管理

---

## 为什么要让 systemd 管理你的服务

很多人跑自己的程序习惯用 `nohup ./app &` 或者写个 `screen`，图省事。时间一长问题就来了：进程崩了没人管、开机要手动起、日志散落在各处、要按顺序启动好几个依赖。systemd 作为 Debian 13 等主流发行版的默认 init，把这些事全包了——只要写一个几十行的 unit 文件，就能得到**开机自启、崩溃自动拉起、统一日志、按依赖顺序启动、资源限制**一整套能力。

这篇文章只讲自己管理服务时最常用的那部分，重点是**每个选项为什么这么设、设错了会怎样**。文中行为以 systemd 257（Debian 13 自带版本）为准。

## 单元（unit）与它的类型

systemd 里一切皆「单元」。每个单元是一个 ini 风格的纯文本文件，后缀决定类型：

| 类型 | 后缀 | 作用 |
|------|------|------|
| service | `.service` | 一个长期运行的进程，或一次性执行的命令 |
| timer | `.timer` | 定时触发一个 service，systemd 版的 cron |
| target | `.target` | 一组单元的「集合点」，用于编排出启动层级 |
| socket | `.socket` | 套接字激活（socket activation） |
| mount / automount | `.mount` / `.automount` | 挂载点，替代 fstab 的动态部分 |
| path / slice / scope | `.path` 等 | 监听文件变化、cgroup 分组等，日常自用较少碰到 |

日常自用 90% 的场景就是 **service + timer**。

### unit 文件放哪里

systemd 按固定顺序搜索，排在前面的覆盖后面的：

| 路径 | 谁来放 |
|------|--------|
| `/etc/systemd/system/` | **管理员手写的放这里**，优先级最高 |
| `/run/systemd/system/` | 运行时生成，重启即失 |
| `/usr/local/lib/systemd/system/` | 本地软件包安装的 |
| `/usr/lib/systemd/system/` | 发行版软件包带来的，**不要手改** |

软件包升级会用新版本覆盖 `/usr/lib` 下的文件，所以自己的 unit 一律放 `/etc/systemd/system/`。想改发行版自带的 unit 也别直接改原文件，用 drop-in 覆盖：

```bash
systemctl edit nginx.service   # 自动创建 /etc/systemd/system/nginx.service.d/*.conf
```

 drop-in 只写你要改的那几行，原文件升级后你的修改也不会丢。

## 写一个最小的 service

假设有个 Python 写的 API 服务要常驻，启动脚本是 `/opt/myapp/run.py`：

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=MyApp API Server
After=network-online.target
Wants=network-online.target

[Service]
Type=exec
ExecStart=/usr/bin/python3 /opt/myapp/run.py
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

三段的分工很清晰：

- `[Unit]` — 这个单元**是什么、依赖谁、排在谁后面**（所有类型单元通用）
- `[Service]` — **怎么启动、怎么停、崩了怎么办**（service 专属）
- `[Install]` — **开机自启挂在哪个 target 下**。注意这一段运行时**不被 systemd 解释**，只给 `systemctl enable / disable` 用

启动并设为开机自启：

```bash
systemctl daemon-reload      # 改完 unit 文件必须先做这一步
systemctl enable --now myapp.service   # enable + start 二合一
systemctl status myapp.service
```

`daemon-reload` 是最容易漏的一步。systemd 把已加载的 unit 配置缓存在内存里，直接改文件不重载的话，`systemctl start` 用的还是旧配置。

## [Service] 段：Type 决定「什么算启动成功」

`Type=` 是 [Service] 段里最关键、也最容易设错的一项。它告诉 systemd「用什么机制判断服务启动完成了」。

| Type | 判定启动完成的时机 | 适用场景 |
|------|--------------------|----------|
| `simple` | `fork()` 出子进程**立刻**算成功 | 传统默认，但太乐观 |
| `exec` | 子进程 `execve()` 成功才算成功 | **长期服务推荐** |
| `forking` | 父进程退出才算成功（传统 daemon 行为） | 老式 daemon，官方已不推荐 |
| `oneshot` | 主进程**退出**才算完成（且成功） | 一次性脚本、开机配置 |
| `notify` | 服务通过 `sd_notify()` 发 `READY=1` 才算成功 | 服务代码里集成了 sd_notify |
| `notify-reload` | 同上，额外支持 SIGHUP 热重载协议 | 支持热重载的现代服务 |
| `dbus` | 拿到指定 D-Bus 名称才算成功 | 提供 D-Bus 接口的服务 |
| `idle` | 类似 simple，但延迟到其它任务都派发完 | 只为控制台输出整洁 |

默认值容易踩坑：**指定了 `ExecStart=` 但没写 `Type=`（也没写 `BusName=`）时默认是 `simple`；`Type=` 和 `ExecStart=` 都没指定时默认是 `oneshot`。**

`simple` 和 `exec` 的区别值得记住：`simple` 在 `fork()` 返回后就认为启动成功——此时子进程还没换上真正的可执行文件。如果二进制路径不存在、或者 `User=` 指定的用户不存在，`systemctl start` **照样报成功**，`status` 显示 active，但服务其实根本没跑起来。`exec` 会等到 `execve()` 成功，这种低级错误就能被正确报成失败。官方对长期服务的推荐就是 `exec`。

`oneshot` 有两个反直觉的行为：

1. 不设 `RemainAfterExit=yes` 时，脚本跑完后单元直接从 `activating` 变成 `dead`——`status` 看起来像「没启动过」，这是**正常的**。要让跑完之后仍显示 active（方便被别的单元依赖），加 `RemainAfterExit=yes`。
2. **永远不会在干净退出时被重启**——`Restart=always` 和 `Restart=on-success` 对 oneshot 是被拒绝的。

`forking` 官方已明确不推荐（"discouraged"），新服务用 `notify`/`notify-reload`/`dbus` 代替；如果你的程序是传统 fork 型 daemon 非用它不可，记得同时写 `PIDFile=`，否则 systemd 猜不准主进程，崩溃检测和自动重启都会不可靠。

### Exec* 家族

| 选项 | 执行时机 |
|------|----------|
| `ExecCondition=` | 最先跑，退出码非 0 则**跳过**（不算失败）整个单元 |
| `ExecStartPre=` | `ExecStart` 之前，常用来建目录、做前置检查 |
| `ExecStart=` | 主启动命令 |
| `ExecStartPost=` | 启动成功之后（如写入运行时状态） |
| `ExecReload=` | `systemctl reload` 时执行 |
| `ExecStop=` | 停止时，应写成**同步**操作 |
| `ExecStopPost=` | 停止之后无条件执行（清理） |

`ExecStop=` 一个常见错误是写一条异步命令（比如只发个信号就返回）：systemd 在 `ExecStop` 命令退出后会立即按 `KillMode=`/`KillSignal=` 清理剩余进程，异步命令往往来不及完成。要么让停止命令自己等待进程退出，要么干脆不写、让 systemd 直接发信号。

### Exec* 命令行的前缀

每条 `Exec*=` 的可执行路径前可以加特殊前缀，最常用的是 `-`：

| 前缀 | 效果 |
|------|------|
| `-` | 命令失败也视为成功（常用于非关键的 `ExecStartPre`） |
| `@` | 第二个参数作为 `argv[0]` 传给进程 |
| `:` | 不做环境变量替换 |
| `+` | 以完整权限执行，绕过本单元的 `User=`/沙箱限制 |
| `!` | 只绕过 `User=`/`Group=` 等凭据设置 |
| `!!` | 同 `!`，但仅在系统不支持 ambient capabilities 时生效 |

```ini
ExecStartPre=-/usr/bin/mkdir -p /var/lib/myapp   # 目录已存在也不要紧
```

### 记住：这不是 shell

`ExecStart=` 的语法只是**看起来**像 shell，实际上：

- 不支持管道 `|`、重定向 `<` `>` `>>`、后台 `&`
- 第一个参数必须是**绝对路径**或**不含斜杠的文件名**（后者在编译期固定的搜索路径里查找，可用 `systemd-path search-binaries-default` 查看，一般是 `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin`）
- 要用 shell 语法，显式包一层：`ExecStart=/bin/sh -c 'dmesg | tac'`
- 环境变量替换有两种：`${FOO}` 替换成一个完整参数（含空格），`$FOO` 按空白拆成零或多个参数；要传字面 `$` 写 `$$`
- 支持 `%` 占位符，常用的有 `%H`（主机名）、`%i`（模板实例名）、`%n`（单元名）

```ini
Environment="ONE=one" "TWO=two two"
ExecStart=/bin/echo $ONE $TWO ${TWO}   # 得到 4 个参数：one  two  two  "two two"
```

## [Unit] 段：依赖与顺序

这是新手最容易混淆的地方：**「依赖」和「顺序」是两回事，要分开设。**

依赖类选项回答「要不要把对方也启动」：

| 选项 | 强度 |
|------|------|
| `Wants=` | 弱依赖：对方启动失败也不影响我。**日常首选** |
| `Requires=` | 强依赖：对方被显式停止时我也会被停止；但**不保证**对方一直在线 |
| `BindsTo=` | 更强：对方意外消失（崩溃、设备拔出）我也会被停掉 |
| `Requisite=` | 对方没在跑就立刻失败，不会去启动它 |
| `PartOf=` | 只在「停止/重启」时联动，单向 |
| `Upholds=` | 我在线时就持续把对方拉起来（类似外部 `Restart=`） |

顺序类选项回答「谁先谁后」：

| 选项 | 效果 |
|------|------|
| `After=` | 我在列出的单元**之后**启动 |
| `Before=` | 我在列出的单元**之前**启动 |

**关键点：`Requires=` 本身不含排序语义。** 只写 `Requires=network-online.target` 而不加 `After=`，两者会同时启动，你的服务可能在网络还没好时就跑起来了。常见且稳妥的写法是两者都写：

```ini
Wants=network-online.target
After=network-online.target
```

官方也确实更推荐 `Wants=` 而不是 `Requires=`——服务越是能在依赖缺失时降级运行，系统整体越健壮。

反过来，`[Install]` 段的 `WantedBy=` / `RequiredBy=` 是给 `systemctl enable` 用的：enable 时会在 `multi-user.target.wants/myapp.service` 建一个软链接，等价于在 target 上写了一条 `Wants=myapp.service`。所以 `enable` 之后的自启依赖是**存在磁盘上的软链接**，不是每次启动现算的。

## 崩溃自愈与防频繁重启

```ini
[Service]
Restart=on-failure      # 退出码非 0 / 被信号杀死 / 超时 才重启
RestartSec=10s
RestartSteps=4
RestartMaxDelaySec=160s
```

`Restart=` 各取值的区别：

| 取值 | 何时重启 |
|------|----------|
| `no`（默认） | 从不 |
| `on-success` | 仅干净退出时 |
| `on-failure` | 非零退出码、被信号终止（含 core dump）、操作超时、watchdog 超时 |
| `on-abnormal` | 被信号杀死、超时（不含非零退出码） |
| `on-abort` | 仅未捕获信号导致的退出 |
| `on-watchdog` | 仅 watchdog 超时 |
| `always` | 无条件（oneshot 除外） |

长驻服务官方推荐 `on-failure`。如果你希望服务能「自己决定退出且不被立刻拉起」，用 `on-abnormal`。注意 `systemctl stop` 触发的停止**永不触发重启**，不用担心停不下来。

`RestartSec=` 默认只有 **100ms**——服务如果在启动阶段就崩（比如配置错误），会以每秒 10 次的速率疯狂重启。配指数退避：

```ini
RestartSec=10s
RestartSteps=4
RestartMaxDelaySec=160s
```

产生 10s → 20s → 40s → 80s → 160s → 160s … 的间隔。

还有一层保护在 `[Unit]` 段：

```ini
StartLimitIntervalSec=60s
StartLimitBurst=5
```

60 秒内最多允许启动 5 次，超过即进入 `failed` 且不再尝试。**它对所有启动方式生效，包括你手动 `systemctl start`**，不只是 `Restart=` 触发的。默认值在 `/etc/systemd/system.conf` 的 `DefaultStartLimitIntervalSec=` / `DefaultStartLimitBurst=` 里。踩到限速后用 `systemctl reset-failed myapp.service` 清除状态。

## 日志：别再自己写日志文件

用了 systemd 就不用再操心日志落地。服务进程往 stdout/stderr 输出的内容会自动被 journald 收走：

```bash
journalctl -u myapp.service            # 全部日志
journalctl -u myapp.service -f         # 实时跟踪（类似 tail -f）
journalctl -u myapp.service --since today
journalctl -u myapp.service -p err     # 只看错误级别
journalctl -u myapp.service -o cat     # 只要正文，去掉前缀
```

想看「这次启动以来」的日志，`journalctl -b`。服务程序本身只需要往标准输出打印，不需要管日志切割——这是 journald 帮你省掉的又一件麻烦事。

## 用 timer 替代 cron

systemd 的定时任务由一个 `.timer` + 一个同名的 `.service` 组成：timer 到点去激活 service。

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Backup job

[Service]
Type=oneshot
ExecStart=/opt/backup/run.sh
```

```bash
systemctl enable --now backup.timer
systemctl list-timers
```

`[Timer]` 段有两类触发方式。**单调定时器**（相对单调时钟，与时区无关）：

| 选项 | 起算点 |
|------|--------|
| `OnActiveSec=` | timer 单元自身激活时 |
| `OnBootSec=` | 系统启动时 |
| `OnStartupSec=` | systemd 启动时 |
| `OnUnitActiveSec=` | 上次触发单元激活时 |
| `OnUnitInactiveSec=` | 上次触发单元结束时 |

**日历定时器**用 `OnCalendar=`，语法是 `星期 日期 时间`，任何部分可用 `*`、逗号列表 `1,5`、范围 `2..4`、步进 `1/2`：

| 表达式 | 含义 |
|--------|------|
| `*-*-* 03:00:00` | 每天 03:00（等价简写 `daily`） |
| `Mon..Fri *-*-* 09:00:00` | 工作日早上 9 点 |
| `*:0/15` | 每 15 分钟 |
| `*-2,5,8,11-01 00:00:00` | 每季度第一天（等价 `quarterly`） |
| `Mon *-05~07/1` | 五月的最后一个周一（`~` 表示从月末倒数，`/1` 表示每 1 天） |
| `hourly` / `weekly` / `monthly` / `yearly` | 预置简写 |

可用 `systemd-analyze calendar` 立刻验证表达式并算出下次触发时间，**写完先验证再部署**：

```bash
$ systemd-analyze calendar "Mon..Fri *-*-* 09:00:00"
  Normalized form: Mon..Fri *-*-* 09:00:00
    Next elapse: Tue 2026-10-06 09:00:00 UTC
```

几个影响行为的开关：

| 选项 | 默认 | 说明 |
|------|------|------|
| `AccuracySec=` | `1min` | 允许的触发时间窗。timer 会在窗口内随机偏移一点以合并 CPU 唤醒；要精确触发设 `1us` |
| `RandomizedDelaySec=` | `0` | 额外随机延迟，**作用与 AccuracySec 相反**——把相似任务错开，避免雪崩 |
| `Persistent=` | `false` | 记录上次触发时间，**关机期间错过的次数在开机后补跑一次**。只对 `OnCalendar=` 有效 |
| `WakeSystem=` | `false` | 从挂起状态唤醒系统（需硬件支持） |

`Persistent=true` 是替代 cron 时最容易漏的一项——cron 在机器关机时就直接跳过本次任务，timer 加上它才能补跑。

一个要注意的机制细节：**被触发的 service 若已经 active，timer 到点不会重启它**。所以被 timer 触发的 service 不要设 `RemainAfterExit=yes`（那样它跑完一次就永远 active，后面再也不触发了）。

## 调试工具箱

```bash
systemd-analyze verify /etc/systemd/system/myapp.service   # 静态检查 unit 文件
systemd-analyze calendar "..."        # 验证日历表达式
systemd-analyze timespan "1h 30min"   # 验证时间跨度写法
systemd-analyze blame                 # 看启动各阶段耗时
systemd-analyze critical-chain        # 看启动关键路径

systemctl list-units --type=service --state=running
systemctl list-timers                 # 所有定时器的下次触发时间
systemctl list-dependencies myapp.service   # 依赖树
systemctl status myapp.service        # 状态 + 最近日志 + 主 PID
systemctl cat myapp.service           # 看生效的完整配置（含 drop-in）
systemctl reset-failed myapp.service  # 清除 failed 状态与启动计数
systemctl daemon-reload               # 改完 unit 文件之后必做
```

`systemd-analyze verify` 能在部署前就抓出一类问题，比如 `ExecStart` 指向的二进制不存在会直接报错：

```
g.service: Command /bin/nonexistent-binary is not executable: No such file or directory
```

常用状态对照：

| 状态 | 含义 |
|------|------|
| `active (running)` | 正在运行 |
| `active (exited)` | oneshot 已成功执行完（通常配了 `RemainAfterExit`） |
| `activating` | 启动中 |
| `deactivating` | 停止中 |
| `failed` | 失败（或触发启动限速） |
| `inactive (dead)` | 未运行 |
| `waiting`（timer） | 已启用，等下一次触发 |

## 常见坑速查

- **改了 unit 文件不 `daemon-reload`** → 生效的还是旧配置
- **只写 `After=` 不写 `Wants=`** → 只排序不建立依赖，手动 start 时对方不会被拉起
- **只写 `Requires=` 不写 `After=`** → 有依赖没顺序，仍可能同时启动
- **改了 `/usr/lib/systemd/system/` 下的包文件** → 升级时被覆盖，用 `systemctl edit` 做 drop-in
- **`Type=oneshot` + `Restart=always`** → 无效，oneshot 不会在干净退出时重启
- **`Type=oneshot` 不设 `RemainAfterExit=yes`** → 跑完显示 dead，看起来像没跑过
- **`RestartSec` 用默认值** → 崩溃循环时每秒重启 10 次，记得加 `RestartSteps` 退避
- **被 timer 触发的 service 设了 `RemainAfterExit=yes`** → 只会触发一次
- **`Persistent=` 配在单调定时器上** → 无效，它只对 `OnCalendar=` 生效
- **在 `ExecStart` 里用管道或重定向** → 不支持，要 shell 语法就 `/bin/sh -c '...'`

## 参考

- [systemd.unit(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html) — [Unit]/[Install] 段全部选项、依赖语义、单元搜索路径
- [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) — Type、Exec* 家族、重启策略、命令行解析规则
- [systemd.timer(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html) — 定时器全部选项
- [systemd.time(7)](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html) — 时间跨度与日历事件表达式语法
- [systemd.exec(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html) — 执行环境（`User=`、`WorkingDirectory=`、沙箱选项）
- [systemctl(1)](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) / [journalctl(1)](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) / [systemd-analyze(1)](https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html) — 三个最常用的命令行工具
