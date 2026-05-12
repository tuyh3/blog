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

当前建议只使用少量稳定主题：`ai`、`oracle`、`sre`、`site`、`finance`、`hugo-blog`。判断不出来就不填，后续整理时再补。

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

## 日常学习笔记流程

建议把学习内容分成两个层级：

- 每天先写 `notes/`：只记录事实、问题、收获和后续线索，不追求成文。
- 每周再写 `posts/`：从本周笔记里提炼一个完整主题，形成可长期复用的文章。

每日学习总结：

```bash
hugo new content -k learning-note notes/2026-05-12-learning-summary.md
```

写的时候优先补齐四块内容：

- 今天学了什么。
- 关键收获是什么。
- 还有哪些卡点或问题。
- 哪些内容值得周末整理成文章。

如果当天笔记想公开展示，把 front matter 里的 `draft: true` 改成 `draft: false`。如果只是给周末整理用，可以先保持草稿状态。

每周整理正式文章：

```bash
hugo new content -k weekly-learning-article posts/2026-05-17-weekly-learning-summary.md
```

整理时不要把每日笔记简单拼接到文章里。建议按这个顺序处理：

1. 先扫一遍本周所有学习笔记，标出重复出现的问题和概念。
2. 选一个最值得沉淀的主题，其他碎片先留在笔记里。
3. 把每日记录改写成“问题 -> 结论 -> 方法 -> 案例 -> 未解决问题”的结构。
4. 给文章补 `summary` 和合适的 `topics`。
5. 预览确认后，把文章的 `draft: true` 改成 `draft: false`。

这个流程的目标是降低每天记录的压力，同时保证每周至少沉淀一篇更完整的技术资产。

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
