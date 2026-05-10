# blog

个人技术博客 / 作品集网站，基于 Hugo + PaperMod + GitHub Pages。

第一阶段目标是低维护成本：静态站点、Markdown 写作、GitHub Actions 自动部署，不引入 Node.js / npm。

## 访问地址

仓库名计划使用 `blog`，因此 GitHub Pages 默认地址是：

```text
https://tuyh3.github.io/blog/
```

如果以后想使用 `https://tuyh3.github.io/` 根地址，需要把 GitHub 仓库改为用户站点仓库名 `tuyh3.github.io`，或绑定自定义域名后调整 `hugo.yaml` 里的 `baseURL`。

## 内容结构

```text
content/
  posts/      正式文章
  notes/      短笔记和灵感
  logs/       开发日志、学习日志、交易学习日志
  projects/   项目作品页
  pages/      About、Now、Roadmap、Search 等固定页面
```

## 本地预览

先安装 Hugo Extended 和 Git。

首次预览前拉取 PaperMod 主题：

```bash
git clone --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

启动本地预览：

```bash
hugo server -D --config hugo.yaml,hugo.local.yaml --bind 127.0.0.1 --port 1313
```

默认访问：

```text
http://localhost:1313/blog/
```

## 新建内容

日常写作不用先想分类和标签。只需要先选一个粗目录：

- `posts/`：准备整理成正式文章时使用。
- `notes/`：大多数想法、摘记、问题清单先放这里。
- `logs/`：过程记录、学习记录、交易学习日志。
- `projects/`：项目作品页。

如果主题非常明确，可以额外加一个粗分类：

```yaml
topics:
  - "ai"
```

当前建议只使用少量稳定主题：`ai`、`oracle`、`sre`、`site`、`finance`。判断不出来就不填，后续整理时再补。

正式文章：

```bash
hugo new posts/my-article.md
```

短笔记：

```bash
hugo new notes/my-note.md
```

日志：

```bash
hugo new logs/2026-05-10-my-log.md
```

项目页：

```bash
hugo new projects/my-project.md
```

新建后把 front matter 里的 `draft: true` 改成 `draft: false` 才会在生产构建中发布。

默认模板不要求填写 `tags` 或 `categories`。日常检索主要依赖粗主题、目录、标题、摘要和站内搜索。

## 部署到 GitHub Pages

1. 在 GitHub 创建仓库 `tuyh3/blog`。
2. 把本地代码推送到 `main` 分支。
3. 在仓库 Settings -> Pages 中，把 Source 设置为 GitHub Actions。
4. 推送后 `.github/workflows/deploy.yml` 会自动构建并发布。

workflow 会在 GitHub Actions 里安装 Hugo Extended，并在构建时拉取 PaperMod 主题。

## 维护原则

- 内容优先，不做复杂前端。
- 生产配置集中在 `hugo.yaml`，本地预览覆盖配置放在 `hugo.local.yaml`。
- 主题不直接提交到仓库，按需拉取。
- VPS 后续只用于项目 Demo、API 或实验服务，不用于第一版博客部署。
