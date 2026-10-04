---
title: git使用教程
subtitle:
date: 2026-10-04T12:22:30+08:00
slug: git
draft: false
description: git使用教程
keywords: git
weight: 0
categories:
  - 教程
collections:
  - git
tags:
  - git
  - code

---

## 安装git

```shell
sudo apt update
sudo apt install git
```

检查版本：

```shell
git --version
```

## 配置git

Git 提交时需要记录作者信息，安装后第一件事就是配置用户名和邮箱：

```shell
# 全局配置（对所有仓库生效）
git config --global user.name "swysgh"
git config --global user.email admin@swysgh.top

# 仅当前仓库配置（去掉 --global）
git config user.name "swysgh"
git config user.email admin@swysgh.top
```

常用配置命令：

```shell
git config --list           # 显示当前生效的配置
git config --global -e      # 编辑全局配置文件
git config --global --unset user.name  # 删除某项配置
```

一些推荐的默认配置：

```shell
# 设置默认分支名为 main
git config --global init.defaultBranch main

# 让 git 正确显示中文文件名
git config --global core.quotepath false

# 保存凭据，避免每次推送都输入密码
git config --global credential.helper store

# 输出带颜色
git config --global color.ui auto
```

配置文件位置：全局配置在 `~/.gitconfig`，仓库配置在 `.git/config`。

## 创建仓库

```shell
git init [dir]          # 初始化本地仓库
git clone <repo> [dir]  # 克隆远程仓库到本地
```

示例：

```shell
git init myproject
git clone https://github.com/swysgh/blog.git
git clone git@github.com:swysgh/blog.git   # 使用 SSH 协议
```

## 基本操作

### 文件状态与追踪

Git 中的文件有三种状态：工作区（已修改）、暂存区（已 `add`）、本地仓库（已 `commit`）。

```shell
git status          # 查看仓库当前状态
git status -s       # 精简输出

git add [file]      # 追踪指定文件（添加到暂存区）
git add .           # 添加当前目录所有改动
git add -A          # 添加所有改动（含删除）

git diff            # 比较工作区和暂存区差异
git diff --staged   # 比较暂存区和最后一次提交的差异
git diff HEAD       # 比较工作区和最后一次提交的差异

git rm [file]        # 删除文件并暂存该删除
git rm --cached [file]  # 仅从暂存区移除，保留工作区文件
git mv <old> <new>   # 移动或重命名文件
```

### 提交

```shell
git commit -m '提交说明'      # 提交暂存区到本地仓库
git commit -am '提交说明'     # 跳过 add，直接提交已追踪文件的修改
git commit --amend            # 修改上一次提交（说明或内容）
```

提交说明建议遵循约定式提交（Conventional Commits）：

```text
feat: 新功能
fix: 修复 bug
docs: 文档变更
style: 格式调整（不影响代码逻辑）
refactor: 重构
chore: 构建/工具变更
```

### 撤销与回退

```shell
# 撤销工作区的修改（危险，丢弃未暂存的改动）
git restore [file]
git checkout -- [file]   # 旧写法

# 取消暂存（保留工作区改动）
git restore --staged [file]
git reset HEAD [file]    # 旧写法

# 回退提交
git reset --soft HEAD~1   # 撤销提交，改动保留在暂存区
git reset --mixed HEAD~1  # 撤销提交，改动保留在工作区（默认）
git reset --hard HEAD~1   # 撤销提交并丢弃所有改动（危险）

# 回退到指定提交
git reset --hard <commit-id>

# 生成一个反向提交来撤销某次提交（安全，推荐用于已推送的提交）
git revert <commit-id>
```

### 查看信息

```shell
git log                 # 查看历史提交记录
git log --oneline       # 每个提交一行
git log --oneline --graph --all  # 图形化显示全部分支
git log -p [file]       # 查看指定文件的修改历史及内容
git blame [file]        # 以列表形式查看指定文件每一行的最后修改者
git shortlog            # 生成简洁的提交日志摘要
git show <commit-id>    # 显示某次提交的详细信息
git describe            # 基于标签描述当前提交
```

## 分支管理

```shell
git branch                 # 查看本地分支
git branch -r              # 查看远程分支
git branch -a              # 查看本地和远程所有分支
git branch <name>          # 创建分支
git branch -d <name>       # 删除已合并的分支
git branch -D <name>       # 强制删除分支

git switch <name>          # 切换分支
git switch -c <name>       # 创建并切换分支
git checkout <name>        # 旧写法
git checkout -b <name>     # 旧写法：创建并切换

git merge <branch>         # 将指定分支合并到当前分支
git merge --no-ff <branch> # 禁用快进合并，保留分支历史
```

### 解决合并冲突

合并出现冲突时，Git 会在冲突文件中标记：

```text
<<<<<<< HEAD
当前分支的内容
=======
被合并分支的内容
>>>>>>> feature
```

手动编辑文件，保留正确内容并删除标记，然后：

```shell
git add <冲突文件>
git commit          # 完成合并
```

放弃本次合并：

```shell
git merge --abort
```

### 变基（rebase）

把当前分支的提交“移动”到另一分支的最新提交之后，形成线性历史：

```shell
git switch feature
git rebase main

# 交互式变基，可合并、编辑、删除提交
git rebase -i HEAD~3
```

> 注意：不要对已经推送到远程并被他人使用的分支做 rebase。

## 远程仓库

```shell
git remote -v                              # 查看远程仓库地址
git remote add origin <url>                # 添加远程仓库
git remote remove origin                   # 删除远程仓库
git remote set-url origin <new-url>        # 修改远程地址

git fetch origin            # 从远程拉取，不合并
git pull                    # 拉取并合并（fetch + merge）
git pull --rebase           # 拉取并变基，保持线性历史
git push                    # 推送当前分支到远程
git push -u origin main     # 首次推送并建立追踪关系
git push origin --delete <branch>  # 删除远程分支

git submodule add <url> <path>   # 添加子模块
git submodule update --init --recursive  # 初始化并更新子模块
```

## 标签管理

```shell
git tag                     # 列出所有标签
git tag v1.0.0              # 创建轻量标签
git tag -a v1.0.0 -m '版本说明'  # 创建附注标签
git show v1.0.0             # 查看标签信息
git push origin v1.0.0      # 推送单个标签
git push origin --tags      # 推送所有标签
git tag -d v1.0.0           # 删除本地标签
git push origin --delete v1.0.0  # 删除远程标签
```

## 储藏（stash）

临时保存未提交的改动，用于切换分支或拉取代码：

```shell
git stash                   # 储藏当前改动
git stash -u                # 连同未追踪文件一起储藏
git stash list              # 查看储藏列表
git stash pop               # 恢复并删除最新储藏
git stash apply             # 恢复但保留储藏
git stash drop              # 删除最新储藏
git stash clear             # 清空所有储藏
```

## .gitignore

在项目根目录创建 `.gitignore`，忽略不需要提交的文件：

```text
# 忽略所有 .log 文件
*.log

# 忽略 build 目录
/build/

# 忽略指定文件
secret.txt

# 不忽略 build 下的 keep 文件
!build/keep

# 忽略所有 .env 文件
.env
```

如果文件已被追踪，需先取消追踪：

```shell
git rm --cached <file>
```

## SSH 免密登录

生成密钥对：

```shell
ssh-keygen -t ed25519 -C "swysgh@gmail.com"
```

将 `~/.ssh/id_ed25519.pub` 内容添加到 GitHub 的 `Settings → SSH and GPG keys`。测试：

```shell
ssh -T git@github.com
```

使用 SSH 地址克隆：

```shell
git clone git@github.com:swysgh/blog.git
```

## 常见问题

**提交后发现漏加文件**

```shell
git add <遗漏文件>
git commit --amend --no-edit
```

**本地与远程历史不一致，推送被拒绝**

```shell
git pull --rebase origin main
git push
```

**误删本地分支恢复**

```shell
git reflog                # 查看操作记录找到提交
git branch <name> <commit-id>
```

**查看某文件在某次提交时的内容**

```shell
git show <commit-id>:<file>
```

## 常用工作流

从远程拉取最新代码，开发完成后推送：

```shell
# 1. 拉取最新代码
git pull --rebase

# 2. 修改文件后查看状态和差异
git status
git diff

# 3. 添加并提交
git add .
git commit -m 'feat: 新增功能'

# 4. 推送到远程
git push
```