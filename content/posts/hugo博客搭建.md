---
title: hugo博客搭建
subtitle:
date: 2026-07-09T18:28:13+08:00
slug: 5c6225e
draft: false
description:
keywords:
weight: 0
categories:
  - draft
collections:
  - draft
tags:
  - 博客

---

## hugo的特点

🌟 Hugo
语言：Go

特点：快到飞起。几百篇文章生成只需几毫秒。单文件二进制部署，不需要在服务器上装复杂的运行环境。

为什么适合你：有很多极简且极具黑客风格、极客感的技术主题（如 PaperMod、Blowfish）。它对代码高亮、短代码（Shortcodes）的支持非常完善，非常适合写长篇的运维架构笔记。

## 第一步：安装

在github的releases里选择安装自己系统的hugo <https://github.com/gohugoio/hugo>

`hugo version` 验证安装

## 第二步：新建博客站点

```shell
# 创建一个名为 my_blog 的新站点目录
hugo new site my_blog

# 进入该目录
cd my_blog

# 初始化 git 仓库（后续管理主题和发布必用）
git init
```

执行完后，你会看到如下目录结构：

- archetypes/：存放新建文章时的 Markdown 模板。

- content/：核心目录，你以后写的 Markdown 文章都放在这里。

- public/：执行编译命令后，生成的静态 HTML 网页文件会在这里（默认初始没有，编译后才有）。

- themes/：存放下载的博客主题。

- hugo.toml (或 hugo.yaml/hugo.json)：全局配置文件。

## 第三步：添加博客主题

由于hugo默认不带主题，所以必须手动下载一个

我使用的是fixIt，如果想自己找可以到 <https://themes.gohugo.io/>

fixIt的安装可以参考官网，这里我简单写一下 <https://fixit.lruihao.cn/zh-cn/documentation/installation/>

```shell
# 安装
git submodule add https://github.com/hugo-fixit/FixIt.git themes/FixIt
# 升级
git submodule update --remote --merge themes/FixIt
```

## 第四步：写文章

### 创建新文章

hugo new posts/nginx-config.md

### 本地实时预览（在浏览器看效果）

hugo server -D

## 第五步：推送到github并配置action

可以参考hugo官方的部署方式，也可以看我总结的方式来部署 <https://gohugo.io/host-and-deploy/host-on-github-pages/#article>

建议写一个gitignore确保本地编译生成的public目录和临时缓存不会被推送到 Git

```text
/public/
/resources/_gen/
.hugo_build.lock
```

1. 创建github仓库，记得一定要公开！

2. 在github仓库里选择 **Settings** > **Pages**，然后在**Build and deployment**的**source**里选择github action

3. 在博客的根目录下创建`.github/workflows`文件夹，然后创建hugo.yaml文件，输入如下actions

```yaml
name: Build and deploy
on:
  push:
    branches:
      - main
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: false
defaults:
  run:
    shell: bash
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      # Define tool versions
      DART_SASS_VERSION: 1.101.0
      GO_VERSION: 1.26.4
      HUGO_VERSION: 0.164.0
      NODE_VERSION: 24.18.0

      # Set the build time zone
      TZ: Europe/Oslo
    steps:
      - name: Checkout
        uses: actions/checkout@v6
        with:
          submodules: recursive
          fetch-depth: 0
          lfs: false

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v6

      - name: Create a local tools directory
        run: |
          mkdir -p "${HOME}/.local"

      - name: Install Go
        if: hashFiles('go.mod') != ''
        uses: actions/setup-go@v6
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: false

      - name: Install Node.js
        if: hashFiles('package-lock.json') != ''
        uses: actions/setup-node@v6
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Install Dart Sass
        run: |
          echo "Installing Dart Sass ${DART_SASS_VERSION}..."
          curl -sfL --output-dir "${{ runner.temp }}" -O "https://github.com/sass/dart-sass/releases/download/${DART_SASS_VERSION}/dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          tar -C "${HOME}/.local" -xf "${{ runner.temp }}/dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          echo "${HOME}/.local/dart-sass" >> "${GITHUB_PATH}"

      - name: Install Hugo
        run: |
          echo "Installing Hugo ${HUGO_VERSION}..."
          curl -sfL --output-dir "${{ runner.temp }}" -O "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir "${HOME}/.local/hugo"
          tar -C "${HOME}/.local/hugo" -xf "${{ runner.temp }}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "${HOME}/.local/hugo" >> "${GITHUB_PATH}"

      - name: Log tool versions
        run: |
          echo "Logging tool versions..."
          command -v sass &> /dev/null && echo "Dart Sass: $(sass --version)" || echo "Dart Sass: not installed"
          command -v go &> /dev/null && echo "Go: $(go version)" || echo "Go: not installed"
          command -v hugo &> /dev/null && echo "Hugo: $(hugo version)" || echo "Hugo: not installed"
          command -v node &> /dev/null && echo "Node.js: $(node --version)" || echo "Node.js: not installed"

      - name: Configure Git
        run: |
          echo "Configuring Git..."
          git config --global core.quotepath false

      - name: Fetch full Git history
        run: |
          if [[ $(git rev-parse --is-shallow-repository) == true ]]; then
            echo "Fetching full Git history..."
            git fetch --unshallow
          fi

      - name: Initialize Git submodules
        run: |
          if [[ -f .gitmodules ]]; then
            echo "Initializing Git submodules..."
            git submodule update --init --recursive
          fi

      - name: Install Node.js dependencies
        run: |
          if [[ -f package-lock.json ]]; then
            echo "Installing Node.js dependencies..."
            npm ci
          fi

      - name: Cache restore
        id: cache-restore
        uses: actions/cache/restore@v5
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: hugo-${{ github.run_id }}
          restore-keys: hugo-

      - name: Build
        run: |
          echo "Building the project..."
          hugo build \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --cacheDir "${{ runner.temp }}/hugo_cache"

      - name: Cache save
        uses: actions/cache/save@v5
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: ${{ steps.cache-restore.outputs.cache-primary-key }}

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v5
        with:
          include-hidden-files: false
          path: ./public
  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v5
```

然后在hugo的配置文件里加上这个以配置缓存

```toml
[caches]
  [caches.images]
    dir = ':cacheDir/images'
```

4. 把本地的代码推送到github（我这里使用的是sshkey的方式验证，需要把自己的ssh公钥上传到github里，然后在本地配置好密钥）

```shell
# 1. 确保本地已经初始化了 git（如果之前做过可以跳过）
git init

# 2. 将所有文件（除去 .gitignore 里的）添加到暂存区
git add .

# 3. 提交到本地仓库
git commit -m "feat: 首次初始化 Hugo 博客并配置 FixIt 主题"

# 4. 强制切换主分支名为 main（符合现代 GitHub 规范）
git branch -M main

# 5. 关联你在 GitHub 上刚刚创建的远程仓库（把下面的地址换成你复制的那个）
git remote add origin git@github.com:你的用户名/my-hugo-blog.git

# 6. 首次推送代码到 GitHub
git push -u origin main
```

5. 如果不出意外，在仓库的actions界面里应该就能看到一个工作流了，之后，每次从本地 Git 存储库推送更改时，GitHub Pages 都会重新构建并部署网站
