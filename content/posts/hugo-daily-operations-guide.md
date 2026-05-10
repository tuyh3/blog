---
title: "Hugo 个人技术站日常操作手册"
date: "2026-05-10T00:00:00+05:00"
draft: false
slug: "hugo-daily-operations-guide"
summary: "给第一次使用 Hugo 的自己：从目录结构、本地启停、写文章、搜索、Git 管理到 GitHub Pages 部署的日常操作说明。"
---

这篇文章是写给第一次使用 Hugo 维护个人技术站的自己。目标不是把 Hugo 所有功能讲完，而是覆盖这个站点最常用、最容易忘、最容易出错的日常操作。

当前站点的技术选择：

- 静态站点生成器：Hugo
- 主题：PaperMod
- 内容格式：Markdown
- 部署方式：GitHub Pages
- 仓库名：`blog`
- 线上地址：`https://tuyh3.github.io/blog/`

因为仓库名是 `blog`，所以 GitHub Pages 的默认公共地址带 `/blog/` 路径。如果以后想用 `https://tuyh3.github.io/` 根路径，需要把仓库改成 `tuyh3.github.io`，或者绑定自定义域名。

## 1. 先理解 Hugo 的工作方式

Hugo 的核心思想很简单：

1. 你写 Markdown。
2. Hugo 读取配置、内容和主题。
3. Hugo 生成纯静态 HTML/CSS/JS。
4. GitHub Pages 把生成后的网站发布出去。

你平时主要维护的是源码目录，不直接维护最终网页。

最重要的区别：

- `content/` 是你写的内容，应该提交到 Git。
- `hugo.yaml` 是站点配置，应该提交到 Git。
- `archetypes/` 是新文章模板，应该提交到 Git。
- `public/` 是 Hugo 生成的最终网页，不需要提交。
- `resources/_gen/` 是 Hugo 生成的缓存，不需要提交。
- `themes/PaperMod/` 是本地拉取的主题目录，当前也不提交。

## 2. 当前目录结构

在本机进入项目：

```bash
cd /Users/tuyh3/Documents/blog
```

常见目录：

```text
.
├── archetypes/                 # 新建内容时使用的模板
├── assets/css/extended/         # PaperMod 允许追加的自定义样式
├── content/                     # 文章、笔记、日志、项目页、固定页面
│   ├── posts/                   # 正式文章
│   ├── notes/                   # 短笔记、灵感、问题清单
│   ├── logs/                    # 开发日志、学习日志、交易学习日志
│   ├── projects/                # 项目作品页
│   └── pages/                   # About、Now、Roadmap、Search 等固定页
├── public/                      # Hugo 构建产物，忽略，不提交
├── themes/PaperMod/             # PaperMod 主题，本地运行需要，忽略，不提交
├── .github/workflows/deploy.yml # GitHub Pages 自动部署
├── .gitignore                   # Git 忽略规则
├── hugo.yaml                    # Hugo 主配置
├── hugo.local.yaml              # 本地预览配置，不影响 GitHub Pages 部署
└── README.md                    # 仓库使用说明
```

日常最常打开的是：

- `content/`：写内容
- `hugo.yaml`：改菜单、站点名、地址、主题参数
- `hugo.local.yaml`：覆盖本地预览地址，避免本地链接跳到线上站点
- `README.md`：给自己和未来维护者看的操作说明
- `.github/workflows/deploy.yml`：自动部署配置

## 3. 本地启动 Hugo 预览

进入项目目录：

```bash
cd /Users/tuyh3/Documents/blog
```

启动本地预览：

```bash
hugo server -D --config hugo.yaml,hugo.local.yaml --bind 127.0.0.1 --port 1313
```

然后打开：

```text
http://localhost:1313/blog/
```

这里几个参数的作用：

- `-D`：包含 draft 草稿内容，方便本地预览未发布文章。
- `--config hugo.yaml,hugo.local.yaml`：先加载生产配置，再用本地配置覆盖 `baseURL`。
- `--bind 127.0.0.1`：只在本机访问。
- `--port 1313`：使用 Hugo 默认常见端口。
- 本地配置里的 `baseURL` 是 `http://localhost/blog/`，Hugo server 会自动补上端口，最终访问地址是 `http://localhost:1313/blog/`。

Hugo server 启动后会监听文件变化。你修改文章并保存，浏览器通常会自动刷新。

如果本地页面里的菜单、上一篇/下一篇等链接自动跳到 `https://tuyh3.github.io/blog/...`，说明当前 Hugo server 没有加载 `hugo.local.yaml`，仍在使用生产 `baseURL`。停止服务后用上面的完整命令重新启动。

## 4. 停止 Hugo 服务

如果 Hugo 是在当前终端前台运行的，按：

```text
Ctrl + C
```

如果忘了 Hugo 跑在哪个终端，可以查 1313 端口：

```bash
lsof -nP -iTCP:1313 -sTCP:LISTEN
```

输出里会看到类似：

```text
COMMAND   PID  USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
hugo    55737 tuyh3    4u  IPv4  ...    0t0  TCP 127.0.0.1:1313 (LISTEN)
```

确认是自己的 Hugo 进程后停止它：

```bash
kill 55737
```

把 `55737` 换成你本机实际看到的 PID。

## 5. 构建静态网站

本地正式构建：

```bash
hugo --gc --minify
```

构建成功后会生成 `public/` 目录。这个目录是最终网站文件，但当前仓库不提交它，因为 GitHub Actions 会在云端重新构建。

常见用途：

- 本地确认配置没有错误。
- 发布前检查是否能正常生成。
- 排查 GitHub Actions 部署失败。

如果只是写文章预览，通常用上面的完整 `hugo server` 命令；如果要确认生产构建，用 `hugo --gc --minify`。

## 6. 新建正式文章

正式文章放在 `content/posts/`。

新建文章：

```bash
hugo new posts/my-first-real-article.md
```

Hugo 会使用 `archetypes/posts.md` 模板生成 front matter。

示例：

```yaml
---
title: "My First Real Article"
date: "2026-05-10T00:00:00+05:00"
draft: true
summary: ""
---
```

发布前注意三件事：

1. 把 `title` 改成中文标题。
2. 写好 `summary`，它会出现在列表页和首页。
3. 把 `draft: true` 改成 `draft: false`。

如果 `draft: true`，生产构建默认不会发布这篇文章；本地加 `-D` 时才能看到。

不需要先填 `tags`、`categories` 或 `series`。这个站点默认采用“先写后整理”的方式：目录只负责大方向，检索交给站内搜索，专题和标签以后内容多了再补。

当前站点暂时关闭了标签和分类页生成。这样写作时不用维护标签体系，也不会因为标签命名不一致产生额外负担。

## 7. 新建短笔记、日志和项目页

短笔记：

```bash
hugo new notes/rag-retrieval-questions.md
```

开发日志或学习日志：

```bash
hugo new logs/2026-05-10-rag-learning-log.md
```

项目作品页：

```bash
hugo new projects/codebase-reading-agent.md
```

极简分工：

- `posts/`：已经整理成体系、适合公开传播的文章。
- `notes/`：默认入口。想法、问题、阅读摘记、未成文的技术线索都可以先放这里。
- `logs/`：过程记录，例如开发日志、学习日志、交易学习日志。
- `projects/`：项目作品页，记录项目定位、技术栈、当前状态、仓库和 Demo。
- `pages/`：长期固定页面，例如 About、Now、Roadmap。

不要一开始追求分类完美。先写，再根据真实内容增长调整结构。判断不出来放哪里时，优先放 `notes/`。

## 8. 修改菜单和站点信息

菜单、站点标题、首页说明都在 `hugo.yaml`。

常改位置：

```yaml
baseURL: "https://tuyh3.github.io/blog/"
title: "tuyh3 技术站"
theme: "PaperMod"
```

首页介绍：

```yaml
params:
  homeInfoParams:
    Title: "把技术积累公开化、结构化、可复用。"
    Content: "这里记录 AI 工程化、代码库理解、RAG、DBA/SRE、技术文档、个人学习系统，以及少量金融学习和交易复盘。"
```

顶部菜单：

```yaml
languages:
  zh:
    menu:
      main:
        - name: "文章"
          url: "/posts/"
          weight: 10
```

`weight` 越小越靠前。

## 9. 搜索怎么用

这个站点已经创建了搜索页：

```text
content/pages/search.md
```

本地地址：

```text
http://localhost:1313/blog/search/
```

线上地址：

```text
https://tuyh3.github.io/blog/search/
```

PaperMod 的搜索依赖首页 JSON 输出。`hugo.yaml` 中已经配置：

```yaml
outputs:
  home:
    - HTML
    - RSS
    - JSON
```

如果搜索页异常，优先检查：

1. `content/pages/search.md` 是否存在。
2. front matter 里是否有 `layout: "search"`。
3. `hugo.yaml` 的 `outputs.home` 是否包含 `JSON`。
4. 浏览器访问 `/blog/index.json` 是否能看到搜索索引。

搜索适合找文章、笔记、日志和项目页。标题、摘要和正文都会参与检索。

## 10. Git 日常管理

先看状态：

```bash
git status
```

查看改了什么：

```bash
git diff
```

暂存所有源码变更：

```bash
git add .
```

提交：

```bash
git commit -m "Add Hugo daily operations guide"
```

建议每次提交只做一类事情：

- 新增一篇文章
- 修改站点配置
- 调整菜单
- 修复部署 workflow

不要把完全无关的修改混在一个提交里。

当前 `.gitignore` 已经忽略：

- `public/`
- `resources/_gen/`
- `.hugo_build.lock`
- `.DS_Store`
- `themes/PaperMod/`

所以正常情况下，生成物和主题源码不会进入 Git。

## 11. 第一次推送到 GitHub

在 GitHub 上创建仓库：

```text
tuyh3/blog
```

仓库创建好后，在本地绑定远端。

SSH 方式：

```bash
git remote add origin git@github.com:tuyh3/blog.git
```

HTTPS 方式：

```bash
git remote add origin https://github.com/tuyh3/blog.git
```

检查远端：

```bash
git remote -v
```

推送：

```bash
git push -u origin main
```

以后日常推送：

```bash
git push
```

## 12. 部署到 GitHub Pages 公共域名

这个仓库已经有 workflow：

```text
.github/workflows/deploy.yml
```

它会做这些事：

1. 拉取仓库代码。
2. 安装 Hugo Extended。
3. 拉取 PaperMod 主题。
4. 执行 Hugo 构建。
5. 上传 `public/`。
6. 发布到 GitHub Pages。

第一次需要在 GitHub 仓库里设置 Pages：

1. 打开 GitHub 仓库 `tuyh3/blog`。
2. 进入 `Settings`。
3. 进入 `Pages`。
4. Source 选择 `GitHub Actions`。
5. 回到 `Actions` 查看部署是否成功。

部署成功后访问：

```text
https://tuyh3.github.io/blog/
```

由于这是项目站点，不是用户根站点，所以 URL 带 `/blog/`。

## 13. 日常发布流程

最常见的一次发布流程：

```bash
cd /Users/tuyh3/Documents/blog

hugo server -D --config hugo.yaml,hugo.local.yaml --bind 127.0.0.1 --port 1313
```

打开本地预览：

```text
http://localhost:1313/blog/
```

写完文章后，另开一个终端：

```bash
hugo --gc --minify
git status
git add .
git commit -m "Add article about ..."
git push
```

然后去 GitHub Actions 看部署状态。通过后访问线上地址确认。

## 14. 更新 PaperMod 主题

当前主题目录是本地忽略目录：

```text
themes/PaperMod/
```

首次本地运行时，如果主题不存在，拉取：

```bash
git clone --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

如果以后想更新主题，最简单的方式是删除后重新拉取：

```bash
rm -rf themes/PaperMod
git clone --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

更新后一定要运行：

```bash
hugo --gc --minify
```

如果构建有警告或错误，先不要急着推送。

## 15. 常见问题

### 本地打开是 404

确认访问的是：

```text
http://localhost:1313/blog/
```

不是：

```text
http://localhost:1313/
```

因为当前站点按 GitHub Pages 项目路径 `/blog/` 配置。

### 页面改了但浏览器没变化

先确认 Hugo server 是否还在运行：

```bash
lsof -nP -iTCP:1313 -sTCP:LISTEN
```

再看终端里是否有构建错误。必要时停止服务后重新启动。

### 新文章本地能看到，线上看不到

优先检查 front matter：

```yaml
draft: false
```

生产构建不会发布 `draft: true` 的文章。

### GitHub Actions 构建失败

先在本地跑：

```bash
hugo --gc --minify
```

如果本地也失败，先修本地错误。如果本地成功但 Actions 失败，再看 GitHub Actions 日志。

### 线上样式丢失

优先检查 `baseURL` 是否是：

```yaml
baseURL: "https://tuyh3.github.io/blog/"
```

项目站点必须带 `/blog/`。如果未来换自定义域名，再同步改这里。

## 16. 未来可能调整的方向

第一阶段先保持简单：

- 不做复杂前端。
- 不引入 Node.js / npm。
- 不把博客部署到 VPS。
- 不急着绑定自定义域名。

后续内容稳定后，可以考虑：

- 给重点文章增加英文摘要。
- 增加项目 Demo 链接。
- 绑定自定义域名。
- 把 VPS 用于项目实验、API 服务或交互式 Demo。

这个站点真正重要的不是主题，而是持续积累高质量内容。工具链只要稳定、简单、可维护，就已经足够。
