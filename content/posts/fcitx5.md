---
title: Fcitx5使用教程
subtitle:
date: 2026-09-26T19:37:18+08:00
slug: fcitx5
draft: true
description:
keywords:
weight: 0
categories:
  - 教程
collections:
  - fcitx
tags:
  - linux
  - fcitx
---

## fcitx5添加词典

创建txt文件，内容格式如下

```text
救救 jiu'jiu 0
```

然后使用 `libime_pinyindict dict.txt dict.dict` 将txt转换为dict格式，放到 `~/.local/share/fcitx5/pinyin/dictionaries/` 下

确认转换完成之后使用 `fcitx5-remote -r` 重载配置
