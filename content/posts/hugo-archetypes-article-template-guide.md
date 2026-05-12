---
title: "Hugo 文章模板构建教程"
date: "2026-05-12T00:00:00+05:00"
draft: false
slug: "hugo-archetypes-article-template-guide"
summary: "用本站的 Hugo 配置说明 archetypes 文章模板怎么设计、怎么新建、怎么验证，以及日常写作时怎么区分通用模板和专用模板。"
topics:
  - "hugo-blog"
---

这篇记录专门说明 Hugo 文章模板怎么构建。

这里的“模板”不是网页样式模板，而是 `hugo new` 新建 Markdown 内容时使用的初始文件模板。Hugo 把这类模板放在 `archetypes/` 目录下。

## 1. 先理解 archetypes 的作用

Hugo 写作流程通常是：

```bash
hugo new content posts/my-article.md
```

执行后，Hugo 会根据路径和模板生成一个 Markdown 文件，里面自动带上 front matter 和正文骨架。

例如正式文章模板可以提前放好：

- 标题
- 日期
- 草稿状态
- 摘要
- 主题
- 正文结构

这样每次写文章时，不需要从空文件开始，也不容易漏掉 `summary`、`draft`、`topics` 这些字段。

## 2. 当前站点的模板目录

本站模板都放在：

```text
archetypes/
```

常用模板包括：

```text
archetypes/default.md
archetypes/posts.md
archetypes/notes.md
archetypes/logs.md
archetypes/projects.md
archetypes/learning-note.md
archetypes/weekly-learning-article.md
```

可以把它理解成两类：

- 通用目录模板：`posts.md`、`notes.md`、`logs.md`、`projects.md`
- 专用写作模板：`learning-note.md`、`weekly-learning-article.md`

通用目录模板解决“这个内容属于哪个栏目”；专用写作模板解决“这次写作具体要按什么结构写”。

## 3. 文章模板最小结构

正式文章模板一般从这几项开始：

```yaml
---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: "{{ .Date }}"
draft: true
summary: ""
# topics: ["hugo-blog"]
---
```

这些字段的作用：

- `title`：文章标题。模板里会先根据文件名自动生成，后面手动改成中文标题。
- `date`：创建时间，由 Hugo 自动写入。
- `draft`：是否草稿。生产发布前要改成 `false`。
- `summary`：列表页、首页、搜索结果里常用的摘要。
- `topics`：本站使用的粗主题分类，不是精细标签。

当前建议把 Hugo、PaperMod、GitHub Pages、写作流程相关内容放到：

```yaml
topics:
  - "hugo-blog"
```

## 4. 构建一个正式文章模板

如果要调整正式文章默认结构，编辑：

```text
archetypes/posts.md
```

一个适合技术文章的基础结构可以是：

```markdown
---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: "{{ .Date }}"
draft: true
summary: ""
# topics: ["hugo-blog"]
---

## 问题

## 结论

## 过程

## 参考
```

这个结构适合大多数技术文章，因为它先逼自己回答“为什么写”，再给结论和过程。

如果是操作手册类文章，可以把正文结构改成：

```markdown
## 适用场景

## 前置条件

## 操作步骤

## 验证方式

## 常见问题
```

模板不需要一次设计完美。先让它帮助你开始写，后续再根据真实写作习惯调整。

## 5. 构建一个专用模板

当某类内容会重复出现，就适合单独建模板。

例如每日学习总结可以建：

```text
archetypes/learning-note.md
```

每周学习文章可以建：

```text
archetypes/weekly-learning-article.md
```

使用专用模板时，用 `-k` 指定 kind：

```bash
hugo new content -k learning-note notes/2026-05-12-learning-summary.md
```

```bash
hugo new content -k weekly-learning-article posts/2026-05-17-weekly-learning-summary.md
```

这里的 `-k learning-note` 对应 `archetypes/learning-note.md`。路径仍然决定最终内容放在哪里，比如 `notes/` 还是 `posts/`。

## 6. 新建文章的推荐流程

正式文章：

```bash
hugo new content posts/hugo-archetypes-article-template-guide.md
```

如果要用专用模板：

```bash
hugo new content -k weekly-learning-article posts/2026-05-17-weekly-learning-summary.md
```

新建后做四件事：

1. 把 `title` 改成真实中文标题。
2. 写好 `summary`。
3. 填合适的 `topics`。
4. 发布前把 `draft: true` 改成 `draft: false`。

不要直接编辑 `public/` 里的 HTML。`public/` 是 Hugo 构建产物，应该由源文件自动生成。

## 7. 模板常见坑

第一个坑：模板只影响新文件。

你修改了 `archetypes/posts.md`，不会自动改变已经存在的文章。旧文章要手动调整。

第二个坑：路径和 kind 是两件事。

```bash
hugo new content posts/example.md
```

这通常会使用 `posts` 对应模板。

```bash
hugo new content -k learning-note notes/example.md
```

这会强制使用 `learning-note` 模板，但生成位置仍然是 `notes/example.md`。

第三个坑：`topics` 的显示位置容易误解。

本站的 `topics` 用于主题归档页，比如：

```text
/topics/hugo-blog/
```

当前文章详情页底部不显示 `topics`，因为 PaperMod 默认显示的是 `tags`。所以看到文章页底部没有主题，不代表 `topics` 没生效。

## 8. 修改模板后的验证

修改模板或新增文章后，先跑构建：

```bash
hugo --gc --minify
```

如果要本地预览：

```bash
hugo server -D --config hugo.yaml,hugo.local.yaml --bind 127.0.0.1 --port 1313
```

然后打开：

```text
http://localhost:1313/blog/
```

重点检查：

- 新文章是否出现在文章列表。
- `draft: false` 的文章是否能在生产构建中出现。
- `/topics/hugo-blog/` 是否收录了 Hugo 相关文章。
- 搜索页能否搜到标题和摘要。

## 9. 我的使用建议

不要一开始设计很多模板。这个站点最实用的组合是：

- `posts.md`：正式文章通用模板。
- `notes.md`：普通笔记模板。
- `learning-note.md`：每日学习总结模板。
- `weekly-learning-article.md`：每周从笔记整理成文章的模板。

每天低成本写 `notes/`，每周挑一条主线整理到 `posts/`。模板的作用不是限制写作，而是减少每次开始时的阻力。
