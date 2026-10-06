---
title: Debian 13 GNOME 桌面快捷方式：.desktop 文件怎么放、怎么生效
subtitle: 从 XDG 规范到 GNOME 48 的信任机制，讲清应用菜单、桌面图标与开机自启
date: 2026-10-06T21:30:00+08:00
slug: desktop-entry
draft: false
description: 在 Debian 13 GNOME 上创建桌面快捷方式：.desktop 文件语法、应用菜单条目、桌面图标信任机制与开机自启
keywords: desktop entry gnome 快捷方式 debian linux
weight: 0
categories:
  - 教程
collections:
  - linux
tags:
  - gnome
  - linux
  - 快捷方式
  - desktop
---

## 快捷方式的本质是一份 .desktop 文件

Windows 双击的是 `.lnk`，macOS 是 `.app` 包，而 Linux 桌面（GNOME、KDE、XFCE 都一样）用的是 **`.desktop` 文件**：一个遵循 XDG Desktop Entry 规范的纯文本 ini 文件，当前规范版本 1.5（2020-04-27 发布）。

有意思的地方在于，**同一份文件放在不同目录，就变成三种不同的东西**：

| 放哪里 | 用户看到的效果 | 由谁读取 |
|--------|----------------|----------|
| `~/.local/share/applications/` | 出现在应用列表和 Activities 搜索里 | GIO（GLib 的应用数据库），所有遵循 XDG 的程序都认 |
| `~/Desktop/` | 桌面上的图标，双击启动 | GNOME 48 需要额外装扩展，见下文 |
| `~/.config/autostart/` | 登录后自动运行 | gnome-session |

所以「配置桌面快捷方式」真正要学的只有两件事：**文件格式**，以及**放哪里、满足什么条件才生效**。

本文行为以 Debian 13（trixie）为准：gnome-shell 48.7、gnome-session 48.0、gnome-shell-extension-desktop-icons-ng 48.1.0、desktop-file-utils 0.28。文中所有 `.desktop` 例子都过了 `desktop-file-validate`，Exec 的实际行为在 `gio launch` 下真跑过。

## 文件格式

### 骨架

```ini
[Desktop Entry]
Type=Application
Name=Hello Editor
Exec=hello-editor %U
Icon=accessories-text-editor
```

规范原文的几条硬规则：

- 文件必须是 UTF-8，按行解析，**大小写敏感**（`Name` 和 `NAME` 是两个不同的键）。
- `#` 开头是注释行，空行忽略，但读写时应保留。
- 组头写作 `[组名]`；必须有 `[Desktop Entry]` 组，其它组（比如动作组 `[Desktop Action xxx]`）可选。
- 每行是 `Key=Value`，等号两侧的空格忽略；**键名只允许 `A-Za-z0-9-`**。
- 文件后缀 `.desktop`；只有 `Type=Directory` 的目录条目用 `.directory`。

### 值类型与转义

| 类型 | 取值 | 例子 |
|------|------|------|
| `string` | 除控制字符外的 ASCII | `Exec=foo` |
| `localestring` | UTF-8，给人看的 | `Name=编辑器` |
| `iconstring` | 图标名或绝对路径 | `Icon=/opt/foo/icon.png` |
| `boolean` | **只能是 `true` / `false`** | `Terminal=false` |
| `numeric` | C locale 下的浮点数 | |

两个最容易踩的坑：

1. **布尔值不是 yes/no/1/0**。写 `Terminal=yes`，`desktop-file-validate` 会直接判错：

   ```text
   error: value "yes" for boolean key "Terminal" in group "Desktop Entry" contains
   invalid characters, boolean values must be "false" or "true"
   ```

2. 多值（`string(s)`）用分号分隔，**结尾也建议带分号**；值里要写字面分号得转义成 `\;`。字符串支持的转义只有 `\s`（空格）、`\n`、`\t`、`\r`、`\\`——注意这里没有 `\u` 之类的写法，中文直接写。

### 常用键

| 键 | 类型 | 必需 | 说明 |
|---|---|---|---|
| `Type` | string | 是 | `Application` / `Link` / `Directory`；未知类型应被实现忽略 |
| `Version` | string | 否 | 规范版本，符合 1.5 就写 `1.5` |
| `Name` | localestring | 是 | 显示名 |
| `Name[zh_CN]` | localestring | 否 | 本地化名；**同一个键不带后缀的版本必须同时存在** |
| `GenericName` | localestring | 否 | 通用名，如「网页浏览器」 |
| `Comment` | localestring | 否 | 提示文本，别和 `Name` 重复 |
| `Icon` | iconstring | 否 | 图标名（按图标主题查找）或绝对路径 |
| `Exec` | string | 否\* | 命令行；`DBusActivatable=true` 时可不写，但仍建议写以兼容 |
| `Terminal` | boolean | 否 | 是否在终端窗口里运行 |
| `Path` | string | 否 | 工作目录（仅 `Type=Application`） |
| `TryExec` | string | 否 | 探测程序是否装了；找不到就把这条当不存在 |
| `Categories` | string(s) | 否 | 菜单分类，取值见 Desktop Menu 规范的注册表 |
| `MimeType` | string(s) | 否 | 能打开的 MIME 类型 |
| `Keywords` | localestring(s) | 否 | 供搜索用的关键词，不用于显示 |
| `NoDisplay` | boolean | 否 | 「存在，但别在菜单里显示」 |
| `Hidden` | boolean | 否 | 「删除」，用于覆盖上层同名文件 |
| `OnlyShowIn` / `NotShowIn` | string(s) | 否 | 只在 / 不在某些桌面环境显示，取值如 `GNOME` |
| `StartupNotify` | boolean | 否 | 是否发送启动完成通知 |
| `StartupWMClass` | string | 否 | 窗口的 WM class，用来把任务栏窗口和启动器对应起来 |
| `Actions` | string(s) | 否 | 右键快捷动作的标识列表 |
| `DBusActivatable` | boolean | 否 | 走 D-Bus 激活（GNOME 自家应用常见） |
| `SingleMainWindow` / `PrefersNonDefaultGPU` | boolean | 否 | 只是提示，实现可以不支持 |

`Hidden` 和 `NoDisplay` 的区别值得单独记：`Hidden=true` 语义上是「被用户删掉了」，等价于文件不存在；`NoDisplay=true` 是「别在菜单里列出来」，但它仍然可以被当成默认程序、被 `gio launch` 或 MIME 关联调用。

## 落点一：应用菜单与 Activities 搜索

把文件丢进用户级的 applications 目录就行：

```bash
install -Dm644 org.example.HelloEditor.desktop \
  ~/.local/share/applications/org.example.HelloEditor.desktop
```

搜索路径由 XDG Base Directory 规范定义：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `$XDG_DATA_HOME` | `~/.local/share` | 用户级，优先级最高 |
| `$XDG_DATA_DIRS` | `/usr/local/share:/usr/share` | 系统级，从左到右 |

实际扫描的是「每个 data dir 下的 `applications/`」。

### 桌面文件 ID

规范对 ID 的定义是：

> 把文件的完整路径相对于它所在的 `$XDG_DATA_DIRS` 组件取相对路径，去掉 `applications/` 前缀，把 `/` 换成 `-`。

例如 `/usr/share/applications/foo/bar.desktop` 的 ID 是 `foo-bar.desktop`；`/usr/local/share/applications/org.foo.bar.desktop` 和 `/usr/share/applications/org.foo.bar.desktop` 的 ID 都是 `org.foo.bar.desktop`，**只有路径顺序靠前的那个会被使用**。

ID 不是学术概念：GNOME 的 Dock 收藏、默认程序设置里填的都是 ID，不是路径。

### 覆盖与隐藏系统条目

同名（严格说是同 ID）时用户目录优先，所以想改发行版自带的条目，别去动 `/usr/share`：

```bash
cp /usr/share/applications/foo.desktop ~/.local/share/applications/
# 之后只改自己这一份，升级不会覆盖
```

想彻底隐藏某条：

```ini
[Desktop Entry]
Type=Application
Name=Foo
Exec=/usr/bin/foo
Hidden=true
```

（注意 `Hidden` 只在「用户级覆盖系统级」时有意义；写在自己独占的条目里，等于把它删掉。）

### 生效时机

GIO 会监听 applications 目录的变化（`GAppInfoMonitor`），所以新建或改名 `.desktop` 后一般**不用重启 shell**，Activities 搜索立刻能找到；实在不放心就重新登录一次。

但有两件事需要额外跑命令：

- 想让条目参与 **MIME 关联 / 默认程序**，要刷新缓存：

  ```bash
  update-desktop-database ~/.local/share/applications
  ```

  它构建的是 `mimeinfo.cache`，GNOME 的「默认应用程序」和 `xdg-open` 都读它。
- 用 `xdg-mime default foo.desktop text/plain` 设默认程序时，同样依赖这个缓存。

## 落点二：桌面图标（GNOME 48 要装扩展）

GNOME 从 3.28 起就不在桌面上画图标了，Debian 13 的 GNOME 48 默认同样没有。想要桌面图标，先装 DING 扩展：

```bash
sudo apt install gnome-shell-extension-desktop-icons-ng   # trixie: 48.1.0-1
gnome-extensions enable ding@rastersoft.com
gnome-extensions list --enabled
```

（`gnome-extensions` 命令来自 `gnome-shell` 包；扩展 UUID 是 `ding@rastersoft.com`。）

> 刚装完扩展时，`gnome-extensions list` 能看到它（这个命令直接读文件系统），但 `enable` 是通过 D-Bus 让**正在运行的** GNOME Shell 去启用，而 Shell 只在启动时扫描扩展目录。所以如果 `enable` 报「扩展不存在」，注销重新登录一次再执行即可。

然后把 `.desktop` 拷到 `~/Desktop/`。**拷过去还不能双击运行**——GNOME 从 3.x 起就要求 `.desktop` 必须「被信任」，否则双击只会用文本编辑器把它打开。这是安全设计：别人塞给你一个 `.desktop`，不应该立刻变成可执行程序。

DING 48.1.0 的判定在 `app/fileItem.js` 里，`trustedDesktopFile` 是四个条件的与：

```js
get trustedDesktopFile() {
    return this._isValidDesktopFile &&
        this._attributeCanExecute &&
        this.metadataTrusted &&
        !this._desktopManager.writableByOthers &&
        !this._writableByOthers;
}
```

翻译成人话：

| 条件 | 含义 | 怎么满足 |
|---|---|---|
| `_isValidDesktopFile` | 能被解析成合法条目 | 先跑 `desktop-file-validate` |
| `access::can-execute` | 文件带可执行位 | `chmod u+x` |
| `metadata::trusted == "true"` | GIO 元数据里的信任标记 | 右键「允许运行」，或 `gio set` |
| 目录和文件都不是「其他人可写」 | 防止别人替换内容 | 别把 `~/Desktop` 设成 777 |

图形界面的做法是右键图标 → **允许运行**（英文 Allow Launching）。DING 的代码里，这个动作就是写 `metadata::trusted=true`，顺手把可执行位补上：

```js
onAllowDisallowLaunchingClicked() {
    this.metadataTrusted = !this.trustedDesktopFile;
    /* we're marking as trusted, make the file executable too. Note that we
     * do not ever remove the executable bit, since we don't know who set it. */
```

命令行等价写法（要脚本化就用这个）：

```bash
chmod u+x ~/Desktop/org.example.HelloEditor.desktop
gio set ~/Desktop/org.example.HelloEditor.desktop metadata::trusted true
gio info -a metadata::trusted ~/Desktop/org.example.HelloEditor.desktop
```

`metadata::trusted` **不是文件系统 xattr**，而是 GIO/GVfs 的元数据存储，由 `gvfsd-metadata` 守护进程维护（Debian 里由 `gvfs-daemons` 提供，装 GNOME 桌面时会一起装上）。在没装 GVfs 的环境（比如容器里）执行会直接报错，这不是你的语法写错了：

```text
$ gio set foo.desktop metadata::trusted true
gio: Setting attribute metadata::trusted not supported
```

## 落点三：开机自启

自启目录也是 XDG 那一套：

| 路径 | 说明 |
|---|---|
| `~/.config/autostart/` | 用户级（`$XDG_CONFIG_HOME`） |
| `/etc/xdg/autostart/` | 系统级（`$XDG_CONFIG_DIRS`），软件包安装的 |

同名时用户级覆盖系统级。文件格式和菜单条目完全一样，只是**不会被显示在菜单里**。

最小例子：

```ini
[Desktop Entry]
Type=Application
Name=Example Sync
Exec=/home/alice/bin/sync-notes.sh
Icon=folder-documents
Terminal=false
```

### 怎么禁用一条自启

- 规范给的标准做法：在 `~/.config/autostart/` 放一个**同名**文件，内容里写 `Hidden=true`。
- GNOME 的老做法（gnome-session 48 的源码里仍然识别，注释直接写着 "used by old gnome-session"）：`X-GNOME-Autostart-enabled=false`。

gnome-session 判定「这条要不要跳过」的顺序（`gsm-autostart-app.c` 的 `is_disabled()`）就是：`X-GNOME-Autostart-enabled` 为假 → `Hidden` → `OnlyShowIn` / `NotShowIn` / `TryExec` 检查。

顺带说一句 **GNOME Tweaks 的「开机启动程序」页**：它并不写什么特殊配置，只是往 `~/.config/autostart/` 复制和删除 `.desktop` 文件（`gtweak/utils.py` 里就是 glob 这个目录，再把选中的条目拷进来）。所以你自己往这个目录丢文件，Tweaks 里也能看到、也能开关。

### GNOME 私有键

除规范键之外，gnome-session 还认一批 `X-GNOME-*` 键（定义在 `gnome-session/gsm-autostart-app.h`）：

| 键 | 作用 |
|---|---|
| `X-GNOME-Autostart-enabled` | 兼容旧 gnome-session 的开关，`false` 即禁用 |
| `X-GNOME-Autostart-Phase` | 启动阶段：`EarlyInitialization` / `PreDisplayServer` / `DisplayServer` / `Initialization` / `WindowManager` / `Panel` / `Desktop` |
| `X-GNOME-AutoRestart` | 崩溃后自动重启 |
| `X-GNOME-HiddenUnderSystemd` | 被 systemd 托管时不再由会话启动 |
| `X-GNOME-Autostart-discard-exec` | 不把 Exec 写进会话保存的配置 |
| `X-GNOME-Autostart-startup-id` | 指定启动通知 ID |
| `X-GNOME-DBus-Name` / `-Path` / `-Start-Arguments` | 通过 D-Bus 启动，而不是 fork 进程 |
| `AutostartCondition` | 条件自启 |

关于 **`X-GNOME-Autostart-Delay`**：网上（包括不少看起来挺新的教程）还在推荐用它给自启加延迟，但在 Debian 13 的 gnome-session 48.0 源码里**已经搜不到这个键**，写了不会有任何效果。要延迟启动，用 `sh -c "sleep 5; exec ..."` 或者干脆写个 systemd user 服务更可靠。

`AutostartCondition` 支持的语法（`parse_condition_string()`）：

```text
AutostartCondition=GSettings org.gnome.desktop.background show-desktop-icons
AutostartCondition=if-exists /path/to/some/file
AutostartCondition=unless-exists /path/to/some/file
AutostartCondition=if-session gnome-classic
AutostartCondition=unless-session gnome
```

含义分别是：看某个 GSettings 键是否为真、看文件在不在、比对当前会话名。

## Exec 键：坑最集中的地方

### 字段码

| 码 | 含义 |
|---|---|
| `%f` | 单个文件（含路径） |
| `%F` | 一组文件 |
| `%u` | 单个 URL |
| `%U` | 一组 URL |
| `%i` | 展开成 `--icon <Icon>`，仅当 `Icon` 存在 |
| `%c` | 应用名（翻译后的 `Name`） |
| `%k` | `.desktop` 文件自身的位置 |
| `%%` | 字面量 `%` |
| `%d` `%D` `%n` `%N` `%v` `%m` | 已废弃，实现应删掉 |

两条硬规则：**命令行里最多出现一个 `%f`/`%u`/`%F`/`%U`**；`%F`、`%U`、`%i` 只能单独作为一个参数出现。写错了会被校验工具拦下：

```text
error: value "example-viewer %F %U" for key "Exec" in group "Desktop Entry" may
contain at most one "%f", "%u", "%F" or "%U" field code
```

实测（用 `gio launch` 传两个文件进去）：

```text
Exec=/tmp/argdump.sh %U   →  收到两个路径，各自是一个参数
Exec=/tmp/argdump.sh %f   →  只收到一个路径
```

所以「要一次打开选中的全部文件」必须用 `%U` / `%F`。

### 引号与转义

- **只有双引号是引号**。单引号是保留字符，只能出现在双引号包住的参数里：

  ```text
  Exec=/bin/sh -c 'echo hi'              # error: contains a reserved character "'" outside of a quote
  Exec=/bin/sh -c "echo 'hi' > /tmp/a"   # 合法
  ```

- 参数含保留字符就必须用双引号包住。保留字符是：空格、Tab、换行、`"`、`'`、`\`、`>`、`<`、`~`、`|`、`&`、`;`、`$`、`*`、`?`、`#`、`(`、`)`、`` ` ``。
- 双引号内部的 `"`、`` ` ``、`$`、`\` 要用反斜杠转义。因为字符串本身的转义规则先应用，**要在文件里表示一个字面反斜杠得写四个**。
- **`%` 必须写成 `%%`**。实测：`Exec=/bin/sh -c "echo 100% > out.txt"` 校验直接报 `invalid field code "%>"`，而且真跑起来输出变成 `100`——`%>` 被当成字段码吃掉了。写成 `100%%` 才得到 `100%`。
- **不要指望 shell 展开**：`~`、`$VAR`、管道、重定向在 `Exec` 里都不成立（`~` 本身就是保留字符）。要 shell 语义就显式调 `sh -c`：

```ini
[Desktop Entry]
Type=Application
Name=Terminal Here
Comment=Open a shell in a fixed directory
Exec=/bin/sh -c "cd /srv/www && exec gnome-terminal"
Icon=utilities-terminal
Categories=System;TerminalEmulator;
```

（实测这条 `gio launch` 能正常在 `/srv/www` 下起终端；`&&`、`>`、`;` 只要在双引号里都没问题。）

## 图标

`Icon=` 有两种写法：

- 绝对路径：`Icon=/opt/foo/icon.png`，最省事，不怕主题缺失。
- 图标名：`Icon=utilities-terminal`，按 Icon Theme 规范的算法在主题目录里找。

自己做的图标按 hicolor 主题的目录约定放就行，不用动 `/usr/share`：

```text
~/.local/share/icons/hicolor/scalable/apps/my-tool.svg
```

图标主题有缓存机制，通常放进去就能用；如果之前生成过缓存，可以刷一下（可选，`-t` 表示跳过 `index.theme` 检查，自建的 hicolor 目录一般没有这个文件）：

```bash
gtk4-update-icon-cache -t ~/.local/share/icons/hicolor
```

## 收藏到 Dock

GNOME 的 Dock（Dash）收藏项是一串 desktop file ID：

```bash
gsettings get org.gnome.shell favorite-apps
gsettings set org.gnome.shell favorite-apps \
  "['org.gnome.Nautilus.desktop', 'org.example.HelloEditor.desktop']"
```

这里必须填 **ID**（`org.example.HelloEditor.desktop`），不能填 `/home/alice/.local/share/applications/...` 这种路径——前面讲的 ID 规则不是白讲的。整机统一的默认值可以用 dconf keyfile 下发（`/etc/dconf/db/local.d/`），细节见 GNOME 的 System Admin Guide。

## 验证与排错

### 三件套

```bash
# 1. 语法与必需键
desktop-file-validate ~/.local/share/applications/org.example.HelloEditor.desktop
# 2. 直接运行（比双击快，错误会打在终端里）
gio launch ~/.local/share/applications/org.example.HelloEditor.desktop
# 3. 检查桌面图标的信任状态
gio info -a metadata::trusted,access::can-execute ~/Desktop/org.example.HelloEditor.desktop
```

`desktop-file-validate` 退出码非 0 就代表有 error；`hint:` 开头的只是建议（比如分类加得不完整），可以用 `--no-hints` 屏蔽，`--no-warn-deprecated` 屏蔽废弃键警告，`--warn-kde` 打开 KDE 扩展键检查。

一个故意写坏的例子（缺 `Name`、布尔值写成 `yes`、分类没注册）：

```text
$ desktop-file-validate bad.desktop
error: value "yes" for boolean key "Terminal" in group "Desktop Entry" contains
invalid characters, boolean values must be "false" or "true"
error: value "webbrowser" for key "Categories" in group "Desktop Entry" contains
an unregistered value "webbrowser"; values extending the format should start with "X-"
hint:  value "webbrowser" for key "Categories" in group "Desktop Entry" does not
contain a registered main category; application might only show up in a "catch-all" section
error: required key "Name" in group "Desktop Entry" is not present
```

### 症状对照表

| 症状 | 多半是 |
|---|---|
| 菜单 / 搜索里找不到 | 放错目录；`NoDisplay=true`；`Hidden=true`；`OnlyShowIn` 不匹配；`TryExec` 指向的程序不存在 |
| 桌面图标双击打开的是文本编辑器 | 没信任：缺可执行位或缺 `metadata::trusted` |
| 弹窗说权限不对 | `~/Desktop` 或文件被设成「其他人可写」（`o+w`） |
| 图标显示成通用图标 | `Icon=` 的名字在主题里不存在；改用绝对路径 |
| 桌面图标名字显示成 `xxx.desktop` | 同上，未信任时 DING 不用 `Name=`，直接显示文件名 |
| 右键菜单里没有「允许运行」 | 扩展没启用，或条目本身解析失败（`_isValidDesktopFile` 为假） |
| 自启没跑 | 文件不在 `~/.config/autostart/`；被 `Hidden` 或 `X-GNOME-Autostart-enabled=false` 关了；`TryExec` 找不到；程序需要图形会话的环境变量 |
| 改了文件没变化 | 改的是 `/usr/share/applications/` 下的包文件（升级会被覆盖），或没重新登录 |

### 装到系统目录

要给整机或别人用，别手工 `cp`，用 desktop-file-utils 自带的安装器（它会顺手校验）：

```bash
sudo desktop-file-install --dir=/usr/local/share/applications \
     --mode=644 --rebuild-mime-info-cache org.example.HelloEditor.desktop
```

## 一个完整例子

`~/.local/share/applications/org.example.Toolkit.desktop`：带两个右键动作、固定工作目录、图标走自定义 SVG。

```ini
[Desktop Entry]
Type=Application
Version=1.5
Name=Example Toolkit
Name[zh_CN]=示例工具箱
GenericName=Developer Tool
Comment=Open the example toolkit
Keywords=toolkit;example;dev;
Exec=/opt/example/bin/toolkit %U
Icon=/opt/example/share/icons/toolkit.svg
Terminal=false
Path=/opt/example
Categories=Development;
MimeType=text/plain;application/json;
StartupNotify=true
StartupWMClass=example-toolkit
Actions=NewWindow;Logs;

[Desktop Action NewWindow]
Name=New Window
Exec=/opt/example/bin/toolkit --new-window

[Desktop Action Logs]
Name=Open Log Directory
Exec=/bin/sh -c "exec nautilus /var/log/example"
```

装好后自检：

```bash
desktop-file-validate ~/.local/share/applications/org.example.Toolkit.desktop
update-desktop-database ~/.local/share/applications
gio launch ~/.local/share/applications/org.example.Toolkit.desktop
```

## 常见坑速查

1. **布尔值只有 `true`/`false`**，`yes` 会被判错。
2. **`%` 在 Exec 里必须写 `%%`**，否则被当字段码吃掉（`100%` 变 `100`）。
3. **Exec 里只有双引号是引号**，单引号是保留字符，必须包在双引号里。
4. **一个 Exec 里最多一个 `%f/%u/%F/%U`**，且 `%F`/`%U`/`%i` 要单独成参数。
5. **别在 Exec 里写 `~`、`$VAR`、管道、重定向**；要 shell 就 `sh -c "..."`。
6. **桌面图标要同时满足**：可执行位 + `metadata::trusted=true` + 目录和文件都不被他人可写。
7. **`Hidden=true` 是「删掉」，`NoDisplay=true` 是「别显示但仍能用」**。
8. **改 `/usr/share/applications/` 里的文件会被升级覆盖**，要在用户目录放同名文件覆盖。
9. **Dock 收藏、默认程序里填的是 desktop file ID**，不是文件路径。
10. **自启延迟别用 `X-GNOME-Autostart-Delay`**（gnome-session 48 已无此键），用 `sh -c "sleep N; exec ..."` 或 systemd user 服务。

## 参考

- [Desktop Entry Specification 1.5](https://specifications.freedesktop.org/desktop-entry/latest/) — 键表、Exec 字段码与引号规则、桌面文件 ID、本地化键
- [Desktop Application Autostart Specification](https://specifications.freedesktop.org/autostart-spec/latest/) — 自启目录、`Hidden`、`TryExec`
- [Desktop Menu Specification](https://specifications.freedesktop.org/menu-spec/latest/) — applications 目录、`Categories` 注册表、`OnlyShowIn` 取值
- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/) — `$XDG_DATA_*` / `$XDG_CONFIG_*` 的定义
- [desktop-file-utils](https://www.freedesktop.org/wiki/Software/desktop-file-utils/) — `desktop-file-validate` / `desktop-file-install` / `update-desktop-database`
- [Desktop Icons NG (DING)](https://gitlab.com/rastersoft/desktop-icons-ng) — 信任机制说明；判定逻辑见源码 `app/fileItem.js` 的 `trustedDesktopFile`
- [GNOME System Admin Guide：Set default favorite applications](https://help.gnome.org/system-admin-guide/desktop-favorite-applications.html) — `org.gnome.shell favorite-apps`
- [gnome-session 源码（Debian trixie 48.0）](https://sources.debian.org/src/gnome-session/) — `gnome-session/gsm-autostart-app.c` 与 `.h`：GNOME 私有自启键、`AutostartCondition` 解析
- [gnome-tweaks 源码](https://sources.debian.org/src/gnome-tweaks/) — `gtweak/utils.py` 的 `AutostartManager`：Tweaks 的启动项页到底做了什么
- [Debian 包：gnome-shell-extension-desktop-icons-ng](https://packages.debian.org/trixie/gnome-shell-extension-desktop-icons-ng)
