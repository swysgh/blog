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

## 提交签名（GPG / SSH Key 签名）

当使用 `git commit -S` 要求对提交进行加密签名，但 Git 尚未配置签名密钥时，会报错：

```text
致命错误：需要配置 user.signingkey 或者 gpg.ssh.defaultKeyCommand 其中之一
```

现代 Git 支持两种签名方式：
1. **SSH Key 签名（最简单、最推荐）**：直接复用你现有的 SSH 密钥，无需折腾 GPG，GitHub 完美支持并显示绿色的 `Verified` 标签。
2. **GPG Key 签名（传统方式）**：需要生成 GPG 密钥对并配置 Key ID。

### 方案一：使用 SSH Key 进行签名（推荐）

1. 查看你现有的 SSH 公钥：

```shell
ls -la ~/.ssh/
```

通常会有 `id_ed25519.pub` 或 `id_rsa.pub`（推荐使用 `ed25519`）。

2. 配置 Git 使用 SSH 格式签名：

```shell
# 告诉 Git 使用 SSH 作为签名格式（默认是 openpgp）
git config --global gpg.format ssh

# 指定用于签名的 SSH 公钥文件路径
git config --global user.signingkey ~/.ssh/id_ed25519.pub

# （可选）如果希望以后每次 commit 自动签名，不需要每次手动敲 -S：
git config --global commit.gpgsign true
```

3. 将公钥同步到 GitHub（让 GitHub 显示 Verified）：
   - 打印你的公钥内容：`cat ~/.ssh/id_ed25519.pub`
   - 打开 GitHub -> **Settings** -> **SSH and GPG keys**。
   - 点击 **New SSH Key**：
     - **Key type** 务必选择：**`Signing Key`**（注意：不是 Authentication Key）。
     - 将公钥内容粘贴进去并保存。
   - 现在执行 `git commit -S -m "feat: xxx"` 即可成功签名，推送到 GitHub 后提交记录会带有绿色的 `Verified` 徽章。

### 方案二：使用传统 GPG Key 签名

1. 查找已有的 GPG 密钥：

```shell
gpg --list-secret-keys --keyid-format=long
```

输出示例中的 `3AA5C34371567BD2` 就是你的 Key ID。

2. 如果没有 GPG 密钥，先生成一个：

```shell
gpg --full-generate-key
# 按照提示选择：(1) RSA and RSA -> 4096 -> 永不过期 -> 填写姓名和邮箱 -> 设置密码
```

3. 配置到 Git：

```shell
# 告诉 Git 默认签名格式为 openpgp
git config --global gpg.format openpgp

# 填入上面获取到的 Key ID
git config --global user.signingkey 3AA5C34371567BD2
```

### 原理与避坑

- **为什么优先选 SSH 签名？** 传统 GPG 工具链在无图形界面的服务器/LXC 环境中经常会遇到 `pinentry` 弹窗失败、`gpg-agent` 套接字未转发或权限问题，配置繁琐。Git 2.34+ 引入的 SSH 签名直接复用 `ssh-keygen` 生成的密钥，纯命令行交互，更契合服务器开发环境。
- **邮箱一致性**：提交时使用的 `git config user.email` 必须与 GitHub 账号绑定的邮箱一致，否则即便签名成功，GitHub 也会提示 `Unverified`。

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
git tag -a v1.0.0 -m '版本说明'  # 创建附注标签（推荐）
git tag -s v1.0.0 -m '版本说明'  # 创建 GPG 签名标签
git show v1.0.0             # 查看标签信息
git push origin v1.0.0      # 推送单个标签
git push origin --tags      # 推送所有标签（谨慎）
git tag -d v1.0.0           # 删除本地标签
git push origin --delete v1.0.0  # 删除远程标签
```

### 轻量标签 vs 附注标签

- **轻量标签**：仅指向 commit 的指针，无元数据，适合临时标记
- **附注标签**：包含作者、日期、说明，可被 GPG 签名，适合正式发布

**发布标准流程**（配合 GitHub Actions）：

```shell
# 1. 打附注标签（语义化版本：主版本.次版本.修订号）
git tag -a v1.0.0 -m "Release v1.0.0: 新功能说明"

# 2. 推送代码和标签（必须两步）
git push origin main
git push origin v1.0.0
```

**触发 GitHub Actions 自动编译**：推送 `v*` 格式 tag 会自动触发 Release 构建。

## GitHub Actions 自动编译

通过 `.github/workflows/` 目录下的 YAML 配置，实现 push 或 tag 时自动编译、测试、发布。

### 基础配置（Go 项目示例）

创建 `.github/workflows/build.yml`：

```yaml
name: Build and Release

on:
  push:
    branches: [ main ]    # push 到 main 触发
    tags: [ 'v*' ]        # 推送 v1.0.0 格式 tag 触发发布
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.22'
        
    - name: Cache
      uses: actions/cache@v4
      with:
        path: ~/go/pkg/mod
        key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
        
    - name: Build
      run: |
        CGO_ENABLED=0 go build -ldflags="-s -w" -o app .
        
    - name: Test
      run: go test -v ./...
      
    - name: Upload artifact
      uses: actions/upload-artifact@v4
      with:
        name: app-binary
        path: app
        
  release:
    needs: build
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
    - uses: actions/checkout@v4
    - name: Setup Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.22'
    - name: Build multi-platform
      run: |
        GOOS=linux GOARCH=amd64 go build -o app-linux-amd64 .
        GOOS=windows GOARCH=amd64 go build -o app-windows-amd64.exe .
    - name: Create Release
      uses: softprops/action-gh-release@v2
      with:
        files: |
          app-linux-amd64
          app-windows-amd64.exe
        generate_release_notes: true
```

### 常用触发条件

| 场景 | 配置 |
|------|------|
| Push 到分支 | `on: push: branches: [main]` |
| 推送 Tag | `on: push: tags: ['v*']` |
| Pull Request | `on: pull_request` |
| 定时任务 | `on: schedule: cron: '0 0 * * *'` |
| 手动触发 | `on: workflow_dispatch` |

### 多平台编译

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
runs-on: ${{ matrix.os }}
```

### 缓存依赖加速

```yaml
- uses: actions/cache@v4
  with:
    path: ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
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