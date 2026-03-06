---
title: "Hugo + PaperMod 部署到 GitHub Pages"
date: 2026-03-06
draft: false
author: "wenzy"
tags:
  - Hugo
  - PaperMod
  - GitHub Pages
  - GitHub Actions
categories:
  - 博客搭建
summary: "使用 Hugo + PaperMod 搭建博客并部署到 GitHub Pages 的一套完整流程。"
showToc: true
TocOpen: false
---

## 前言

这篇笔记只记录一套能直接执行的流程：使用 **Hugo + PaperMod** 搭建个人博客，并部署到 **GitHub Pages**。

环境默认是 macOS，主题使用 PaperMod，部署方式使用 GitHub Actions。

---

## 一、安装 Hugo 和 Git

先安装 Hugo 与 Git：

```bash
brew install hugo
brew install git
```

安装完成后检查版本：

```bash
hugo version
git --version
```

只要都能正常输出版本信息，就可以继续。

---

## 二、创建 Hugo 站点

创建站点目录：

```bash
hugo new site my-blog --format yaml
cd my-blog
git init
```

这一步会生成 Hugo 站点的基础结构。

---

## 三、安装 PaperMod 主题

使用 submodule 安装 PaperMod：

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive
```

安装完成后，主题目录应为：

```text
themes/PaperMod
```

---

## 四、配置 hugo.yaml

在项目根目录创建或修改 `hugo.yaml`。

下面这份配置尽量保持通用，只保留常用项，站点标题、菜单名称、仓库地址这些内容按你自己的实际情况修改。

```yaml
baseURL: "https://yourname.github.io/"
languageCode: "zh-cn"
title: "My Blog"
theme: "PaperMod"

pagination:
  pagerSize: 5

enableRobotsTXT: true
buildDrafts: false
buildFuture: false
buildExpired: false

minify:
  disableXML: true
  minifyOutput: true

params:
  env: production
  author: "yourname"
  defaultTheme: auto
  disableThemeToggle: false

  homeInfoParams:
    Title: "My Blog"
    Content: >
      记录开发过程中的问题、排查与实现。

  ShowReadingTime: true
  ShowShareButtons: false
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
  ShowCodeCopyButtons: true
  ShowWordCount: true
  ShowRssButtonInSectionTermList: false
  UseHugoToc: true
  disableSpecial1stPost: true
  disableScrollToTop: false
  comments: false
  hidemeta: false
  hideSummary: false
  showtoc: true
  tocopen: false

menu:
  main:
    - identifier: posts
      name: 文章
      url: /posts/
      weight: 5
    - identifier: categories
      name: 分类
      url: /categories/
      weight: 10
    - identifier: tags
      name: 标签
      url: /tags/
      weight: 20
    - identifier: archives
      name: 归档
      url: /archives/
      weight: 30
    - identifier: search
      name: 搜索
      url: /search/
      weight: 40
    - identifier: github
      name: GitHub
      url: https://github.com/yourname
      weight: 50

outputs:
  home:
    - HTML
    - JSON
  section:
    - HTML
  taxonomy:
    - HTML
  term:
    - HTML

taxonomies:
  category: categories
  tag: tags

pygmentsUseClasses: true

markup:
  highlight:
    noClasses: false
```

这份配置包含这些内容：

- 首页直接显示文章列表
- 顶部菜单包含文章、分类、标签、归档、搜索、GitHub
- 保留搜索功能
- 不额外生成 RSS，减少不必要的问题

如果你不需要首页说明，可以删掉 `homeInfoParams` 整段。

---

## 五、创建归档页和搜索页

### 1. 创建归档页

创建文件 `content/archives.md`：

```md
---
title: "归档"
layout: "archives"
url: "/archives/"
summary: "archives"
---
```

### 2. 创建搜索页

创建文件 `content/search.md`：

```md
---
title: "搜索"
layout: "search"
summary: "search"
placeholder: "输入关键词..."
---
```

---

## 六、添加文章

在 `content/posts/` 目录下放入文章文件，例如：

```text
content/posts/first-post.md
content/posts/second-post.md
```

文章的 front matter 推荐使用 YAML，例如：

```md
---
title: "示例文章"
date: 2026-03-06
draft: false
author: "yourname"
tags:
  - Hugo
  - PaperMod
categories:
  - 博客搭建
summary: "这是一篇示例文章。"
---
```

`author` 建议直接写成字符串，`tags` 和 `categories` 使用数组格式即可。

---

## 七、本地预览

启动本地服务：

```bash
hugo server
```

浏览器访问：

```text
http://localhost:1313
```

先确认以下内容是否正常：

- 首页是否显示文章列表
- 顶部菜单是否可见
- 文章页是否能打开
- 搜索页和归档页是否正常
- 分类和标签页是否正常

---

## 八、创建 GitHub 仓库

在 GitHub 创建一个仓库。

如果你想直接使用 GitHub Pages 的默认域名，仓库名应为：

```text
yourname.github.io
```

部署完成后，站点地址通常是：

```text
https://yourname.github.io/
```

---

## 九、添加 .gitignore

在项目根目录创建 `.gitignore`：

```gitignore
public/
resources/_gen/
.hugo_build.lock
```

这样可以避免把本地构建产物一并提交到仓库。

---

## 十、提交博客代码

先检查状态：

```bash
git status
```

然后提交全部站点文件：

```bash
git add .
git commit -m "init Hugo PaperMod blog"
```

添加远程仓库：

```bash
git remote add origin git@github.com:yourname/yourname.github.io.git
```

推送到 GitHub：

```bash
git branch -M main
git push -u origin main
```

---

## 十一、配置 GitHub Actions 自动部署

在项目根目录创建目录：

```bash
mkdir -p .github/workflows
```

然后创建文件 `.github/workflows/hugo.yaml`，内容如下：

```yaml
name: Deploy Hugo site to Pages

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
  cancel-in-progress: true

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.157.0
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Install Hugo
        run: |
          wget -O hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb
          sudo dpkg -i hugo.deb

      - name: Build with Hugo
        run: hugo --gc --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

这份 workflow 的关键点是：

- 使用 `actions/checkout@v4` 拉取代码
- `submodules: recursive` 用于拉取 PaperMod 主题
- 使用 GitHub Actions 自动构建并发布 Hugo 站点
- 构建结果来自 `public/` 目录

---

## 十二、再次提交部署配置

将 workflow 文件提交并推送：

```bash
git add .github/workflows/hugo.yaml
git commit -m "add GitHub Pages workflow"
git push
```

---

## 十三、开启 GitHub Pages

进入仓库页面：

```text
Settings -> Pages
```

将发布来源设置为：

```text
GitHub Actions
```

设置完成后，GitHub 会根据 workflow 自动构建并发布站点。

---

## 十四、检查 Actions 构建状态

进入仓库的：

```text
Actions
```

查看最新一次工作流执行情况。

如果显示成功，说明部署已经完成。

---

## 十五、访问站点

部署成功后，通过以下地址访问博客：

```text
https://yourname.github.io/
```

如果页面没有立刻更新，等一会儿再刷新即可。

---

## 十六、后续更新文章

后续只需要：

1. 在 `content/posts/` 下新增文章
2. 提交并推送代码

常用流程如下：

```bash
git add .
git commit -m "add new post"
git push
```

GitHub Actions 会自动重新构建并发布。

---

## 十七、小结

整个流程可以概括为：

1. 安装 Hugo 和 Git
2. 创建 Hugo 站点
3. 安装 PaperMod 主题
4. 配置 `hugo.yaml`
5. 创建 `search.md` 和 `archives.md`
6. 添加文章
7. 本地用 `hugo server` 预览
8. 创建 GitHub 仓库
9. 提交站点代码
10. 添加 GitHub Actions workflow
11. 在 Pages 中启用 GitHub Actions
12. 推送后自动部署

完成这些步骤后，一个基础可用的 Hugo 博客就已经搭好了。
