---
title: Dovecot 2.4 部署教程：快速跑起来 + 外接 LMTP（Debian 13）
subtitle:
date: 2026-10-10T23:40:00+08:00
slug: dovecot
draft: true
description: 五步跑起 Dovecot，建虚拟用户，再把外部 MTA 用 LMTP 接进来
keywords: dovecot IMAP POP3 LMTP 虚拟用户 passwd-file Debian 2.4
weight: 0
categories:
  - 教程
collections:
  - 搭建
tags:
  - 邮件
  - dovecot
  - Debian
---

Dovecot 只管三件事：**认证**（谁能登录）、**存储**（邮件放哪）、**协议**（IMAP/POP3/LMTP 听哪个端口）。**它不收信也不发信** —— 收信发信是 MTA（Postfix / Exim / maddy）的事，MTA 收到信后用 **LMTP** 把邮件投给 Dovecot。

下文示例统一用：域名 `example.com`、主机名 `mail.example.com`、本机地址 `10.0.0.9`。

## 一、快速开始

### 1. 装包

```bash
apt install dovecot-core dovecot-imapd dovecot-pop3d dovecot-lmtpd
```

**装了协议包就等于启用了协议**，不用手写 `protocols`。机制是：这些包的 `postinst` 会往 `/usr/share/dovecot/protocols.d/` 写 `<包名>.protocol`（内容形如 `protocols { imap = yes }`），而 `dovecot.conf` 里有 `!include_try` 收它。

### 2. 建邮件用户

```bash
groupadd -r vmail
useradd -r -g vmail -d /srv/mail -s /usr/sbin/nologin vmail
install -d -o vmail -g vmail -m 0700 /srv/mail
id vmail                     # 记下 uid/gid，第 4 步要用
```

### 3. 写配置

包自带的配置文件都不用动，自己的覆盖全写 **`/etc/dovecot/local.conf`**（`dovecot.conf` 最后 include 它，优先级最高）：

```ini
hostname = mail.example.com
postmaster_address = postmaster@example.com

mail_home = /srv/mail/%{user | domain}/%{user | username}
mail_driver = sdbox
mail_path = ~/mail
mail_uid = vmail
mail_gid = vmail

namespace inbox {
  inbox = yes
  separator = /

  mailbox Drafts {
    auto = subscribe
    special_use = \Drafts
  }

  mailbox Sent {
    auto = subscribe
    special_use = \Sent
  }

  mailbox Junk {
    auto = subscribe
    special_use = \Junk
  }

  mailbox Trash {
    auto = subscribe
    special_use = \Trash
  }
}

passdb passwd-file {
  passwd_file_path = /etc/dovecot/users
}

userdb passwd-file {
  passwd_file_path = /etc/dovecot/users
}

service lmtp {
  inet_listener lmtp {
    port = 24
  }
}

protocol lmtp {
  auth_username_format = %{user | lower}
}

ssl = required
ssl_server_cert_file = /etc/dovecot/ssl/cert.pem
ssl_server_key_file = /etc/dovecot/ssl/key.pem
```

### 4. 建账号

```bash
# 生成密码哈希（交互输入两遍，输出 {SHA512-CRYPT}$6$...）
doveadm pw -s SHA512-CRYPT

# 写进账号文件：user:密码:uid:gid:gecos:home:shell
echo 'alice@example.com:{SHA512-CRYPT}$6$上面那串:999:989::/srv/mail/example.com/alice::' >> /etc/dovecot/users
chown root:dovecot /etc/dovecot/users
chmod 0640 /etc/dovecot/users
```

`999` / `989` 换成 `id vmail` 看到的 uid/gid。**`uid:gid:home` 必须写全**（原因见第四节）。

### 5. 起服务并验证

```bash
systemctl enable --now dovecot

doveconf -n                                    # 语法错会直接报行号
ss -tlnp | grep -E ':(143|993|110|995|24)\b'
doveadm auth test alice@example.com '密码'      # → passdb: ... auth succeeded
doveadm user alice@example.com                  # → uid/gid/home/mail_path 都要有值
doveadm mailbox list -u alice@example.com       # → INBOX / Drafts / Sent / Junk / Trash
```

别只看 `systemctl is-active`。想确认 TLS 真的对（证书链被验证，不是跳过校验）：

```bash
openssl s_client -connect 10.0.0.9:993 -servername mail.example.com -brief
# → Verification: OK，然后是 "* OK ... Dovecot ready."
```

## 二、创建用户

账号文件格式和 `/etc/passwd` 一样：

```
user:password:uid:gid:gecos:home:shell:额外字段
```

| 字段 | 怎么填 |
|---|---|
| `user` | 写全 `user@domain`。多域名就加多行，`auth_username_format` 默认保留域名 |
| `password` | `{SHA512-CRYPT}$6$...`，用 `doveadm pw -s SHA512-CRYPT` 生成，别写明文 |
| `uid:gid` | 都填 vmail 的；Dovecot 读邮件时会切到这个 uid |
| `home` | 用户状态目录（索引、Sieve 脚本、崩溃 core），一般 `/srv/mail/<域名>/<用户名>` |

写个脚本省事，放 `/usr/local/sbin/dovecot-adduser`：

```bash
#!/bin/bash
# 用法: dovecot-adduser alice@example.com
set -e
user=$1
hash=$(doveadm pw -s SHA512-CRYPT)          # 交互输入密码
uid=$(id -u vmail); gid=$(id -g vmail)
echo "$user:$hash:$uid:$gid::/srv/mail/${user#*@}/${user%@*}::" >> /etc/dovecot/users
doveadm user "$user"                        # 立即回显 uid/gid/home，确认写对了
```

加账号**不需要重启 Dovecot**（认证是每次读文件的）。验证：

```bash
doveadm auth test alice@example.com '密码'
doveadm user alice@example.com
```

## 三、外接 LMTP 使用

MTA 收信后通过 LMTP 把邮件交给 Dovecot 存。**LMTP 不做认证**（它信任网络），所以只在内网/隧道上开。

### 1. Dovecot 侧

上面第 3 步的配置里已经有了：

```ini
service lmtp {
  inet_listener lmtp {
    port = 24
  }
}
```

两个注意点：

- **2.4.1 的 `inet_listener` 没有 `address` 设置**（官方文档里 `address = ...` 的示例在这个版本上会报 `Unknown setting: address`）。要限定监听地址，用全局的 `listen = 10.0.0.9`，`inet_listener` 里只写端口。
- 不加 `listen` 的话是**所有接口**都听。放到有公网地址的机器上之前，务必用防火墙限制来源。

### 2. MTA 侧

Postfix：

```ini
# main.cf
virtual_transport = lmtp:inet:10.0.0.9:24
```

maddy（`maddy.conf`）：

```ini
target.lmtp local_mailboxes {
  targets tcp://10.0.0.9:24
}
```

同机部署就换成 unix socket（`unix:///var/run/dovecot/lmtp`），少开一个端口。

### 3. 实测投递

```bash
swaks --to alice@example.com --server 10.0.0.9:24 --protocol LMTP
```

或者 Python（`smtplib` 自带 LMTP）：

```python
import smtplib

msg = "From: outside@example.net\r\nTo: alice@example.com\r\nSubject: lmtp test\r\n\r\nhi\r\n"
c = smtplib.LMTP("10.0.0.9", 24, timeout=10)
print(c.docmd("LHLO", "tester.local"))                                  # 250 + 能力列表
print(c.sendmail("outside@example.net", ["alice@example.com"], msg))    # {} = 收件人全部被接受
c.quit()
```

投递日志：

```bash
journalctl -u dovecot -f | grep -i lmtp
```

### 4. 两个必踩的坑

| 现象 | 真因 | 修法 |
|---|---|---|
| LMTP 投给一个**存在的**用户，却回 `550 5.1.1 <user@domain> User doesn't exist` | Debian 的 `conf.d/20-lmtp.conf` 给 LMTP 单独设了 `auth_username_format = %{user \| username \| lower}`，**会把域名去掉**，于是拿 `alice` 去查账号文件 | 在 `local.conf` 里覆盖：`protocol lmtp { auth_username_format = %{user \| lower} }` |
| 不存在的收件人也被接受 | 收件人写成了裸用户名，或账号文件缺 `uid:gid:home`，userdb 查不到就按"接受后丢弃"处理 | 收件人必须完整 `user@domain`；账号文件写全字段 |

### 5. 可选：地址扩展

```ini
recipient_delimiter = +
lmtp_save_to_detail_mailbox = yes
```

开启后 `alice+github@example.com` 会投进 `alice` 名下的 `github` 文件夹（没有就自动建），适合按来源分类，不用注册新账号。

## 四、出问题先看这张表

全是实测踩出来的，**报错指向的方向基本都是错的**。

| 现象 | 真因 | 修法 |
|---|---|---|
| `SELECT INBOX` 报 `[SERVERBUG] Internal error`，日志 `mkdir_parents(/var/mail/<user>) failed: Permission denied` | `conf.d/10-mail.conf` 里 Debian 给系统用户准备的 **mbox 默认值是激活的**（`mail_driver = mbox`、`mail_home = /home/%{user \| username}`、`mail_path = %{home}/mail`、`mail_inbox_path = /var/mail/%{user}`）。前三个被 `local.conf` 盖住，但 `mail_inbox_path` 是**另一个设置**，INBOX 就被指到 `/var/mail/` | 把这四行注释掉 |
| `doveadm user` 报 `user doesn't exist`，但 `doveadm auth test` 成功 | `conf.d/10-auth.conf` 末尾的 `!include auth-system.conf.ext` **是激活的**，多了 `passdb pam` + `userdb passwd`，系统 userdb 先答"没这人" | 把那行 include 注释掉 |
| 日志 `User <u> is missing userdb info` | 2.4 里 `userdb passwd-file { fields { uid:default = ... } }` **不能**给只有两段的记录补字段 | 把 `uid:gid:home` 写进账号文件，删掉 `fields` 块 |
| passwd-file `Permission denied (euid=100(dovecot))` | 认证进程不是 root | `chown root:dovecot` + `chmod 0640` |
| `Couldn't parse private SSL key ... No supported data to decode` | 同样是权限（login 进程读不到私钥），**不是密钥坏了** | 私钥也要 `root:dovecot 0640` |
| `Fatal: ... Garbage after '{'` | 2.4 解析器要求 `{` 后必须换行 | 设置拆到下一行 |
| 客户端里出现两个"已发送" | `conf.d/15-mailboxes.conf` 里额外定义了 `Sent Messages` | 删掉那一块，只留一个 `\Sent` |

**改配置前先干跑**，不用重启服务就能验证语法：

```bash
cat > /tmp/t.conf <<'EOF'
dovecot_config_version = 2.4.0
dovecot_storage_version = 2.4.0
!include /etc/dovecot/dovecot.conf
# 这里写要试的设置
EOF
doveconf -c /tmp/t.conf -n >/dev/null && echo "语法 OK" || echo "语法错"
```

（`!include` 不算"设置"，所以文件开头仍必须是那两个版本号。）

## 五、为什么要这么配

**passdb 和 userdb 是分开的。** passdb 只回答"密码对不对"，userdb 回答"这个账号的 uid/gid/home/配额"。所以认证后端（密码文件 / SQL / LDAP）和邮件存放位置可以各自替换。流程是：`imap-login`（以 dovenull 跑，只做 TLS 和收密码）→ `auth` 进程查 passdb/userdb → 把连接交给 `imap` 进程，**切成 userdb 返回的 uid** 去读写。所以别把 dovecot 用户加进邮件目录的组，也别用它拥有邮件。

**namespace / mailbox 是客户端看到的那棵树。** `namespace inbox` 定义根和分隔符；`mailbox` 里 `auto = subscribe` 让客户端自动订阅，`special_use = \Sent` 告诉客户端"这就是已发送"。没有 `special_use`，客户端只能靠名字猜。

**`mail_location` 在 2.4 被拆成三个：** `mail_home`（状态目录）、`mail_driver`（`sdbox` / `maildir` / `mbox`）、`mail_path`（邮件本体，`~` 指 mail_home）。

存储格式：`maildir` 一消息一文件、兼容性最好；`sdbox` 一 mailbox 一文件、inode 少，新部署首选；`mbox` 别用于新系统。**邮件和索引都不要放 NFS**。如果存储机在 MTU 1400 的隧道后面而内网是 1500，又没做 MSS clamp，症状会是"大附件投递卡住"，不是任何 MTU 报错。

### 2.4 和 2.3 语法不兼容

网上大多数教程是 2.3 语法，照抄会起不来。会咬人的几处：

| 2.3 | 2.4 |
|---|---|
| （无） | `dovecot_config_version` 和 `dovecot_storage_version` 必须是文件里最前面的两个设置 |
| `protocols = imap pop3 lmtp` | `protocols { imap = yes }` 块；**默认不启用任何协议** |
| `passdb { }` / `userdb { }` | 必须起名：`passdb some_name { }` |
| `%u` `%d` `%n` `%h` | `%{user}` `%{user \| domain}` `%{user \| username}` `%{home}` |
| `mail_location = ...` | `mail_home` + `mail_driver` + `mail_path` |
| `disable_plaintext_auth` | `auth_allow_cleartext` |
| `ssl_cert` / `ssl_key` | `ssl_server_cert_file` / `ssl_server_key_file` |
| `auth_debug = yes` | `log_debug = category=auth` |

另外：无效的 `%{变量}` 现在**直接报错**（以前静默忽略），表现为"认证莫名失败"；弱密码方案默认禁用。

### 顺带：让 Dovecot 自己当 submission 代理

2.4 新增了 `submission` 服务（默认 587 STARTTLS，465 要显式加 `service submission-login { inet_listener submissions { port = 465  ssl = yes } }`）：**终结客户端 TLS + 自己做认证**，再把信转发给真正的 MTA（`submission_relay_host`）。好处是 MTA 完全不用配 SASL；代价是官方说它"still pretty basic"，**不是完整 SMTP 服务器，必须有后端 MTA 兜着**。

同理，MTA 的 submission 也可以把认证委托给 Dovecot（Dovecot auth 协议，Postfix 侧 `smtpd_sasl_type = dovecot` + `smtpd_sasl_path = inet:10.0.0.9:12345`）。**这个 auth 协议是明文的**，端口只能绑内网/隧道。

## 六、端口一览

| 端口 | 用途 |
|---|---|
| 143 / 993 | IMAP / IMAPS |
| 110 / 995 | POP3 / POP3S |
| 24 | LMTP（MTA 投递给 Dovecot） |
| 12345（示例） | Dovecot auth 协议（给外部 MTA 做 submission 认证，只开内网/隧道） |
| 4190 | ManageSieve |
| 25 / 465 / 587 | **不属于 Dovecot**，是 MTA 的 |

## 参考

- Dovecot 官方文档（2.4）：<https://doc.dovecot.org/2.4.0/>
- 2.3 → 2.4 升级说明（语法差异唯一权威）：<https://doc.dovecot.org/2.4.0/installation/upgrade/2.3-to-2.4.html>
- 快速配置：<https://doc.dovecot.org/2.4.0/core/config/guides/quick.html>
- 虚拟用户：<https://doc.dovecot.org/2.4.0/core/config/users/virtual.html>
- passwd-file 认证：<https://doc.dovecot.org/2.4.0/core/config/auth/databases/passwd_file.html>
- LMTP：<https://doc.dovecot.org/2.4.0/core/config/delivery/lmtp.html>
- 给外部 SMTP 做 SASL：<https://doc.dovecot.org/2.4.0/howto/sasl/postfix.html>
