# Repository Guidelines

## Project Shape

This repository is a personal technical blog and portfolio site built with Hugo,
PaperMod, Markdown, and GitHub Pages. Keep the first-stage goal in mind: low
maintenance, static output, and no Node.js/npm dependency chain.

The site content is primarily Chinese. Preserve the existing direct, practical
writing style when editing posts, notes, logs, project pages, or fixed pages.

## Important Paths

- `content/posts/` contains polished long-form articles.
- `content/notes/` contains shorter notes, ideas, and question lists.
- `content/logs/` contains process records, learning logs, and trading-study logs.
- `content/projects/` contains portfolio/project pages.
- `content/pages/` contains fixed pages such as About, Now, Roadmap, and Search.
- `archetypes/` contains Hugo templates for new content.
- `assets/css/extended/custom.css` contains PaperMod extension CSS.
- `hugo.yaml` is the production site configuration.
- `hugo.local.yaml` is only for local preview overrides.
- `.github/workflows/deploy.yml` builds and deploys to GitHub Pages.

Generated or local-only paths such as `public/`, `resources/_gen/`,
`.hugo_build.lock`, `.DS_Store`, and `themes/PaperMod/` should stay untracked.

## Content Conventions

- Prefer content and maintainability over frontend complexity.
- Use the existing front matter style: `title`, `date`, `draft`, `slug` when
  needed, `summary`, and optional `topics`.
- Keep `topics` coarse and stable. Current intended topics are `ai`, `oracle`,
  `sre`, `site`, and `finance`. If a topic is unclear, omit it.
- New publishable content should set `draft: false`; local drafts can remain
  `draft: true`.
- Do not manually edit generated files in `public/`; change source content,
  config, layout, or assets instead.

## Local Commands

Before local preview, ensure the PaperMod theme exists:

```bash
git clone --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

Run the local preview with drafts enabled:

```bash
hugo server -D --config hugo.yaml,hugo.local.yaml --bind 127.0.0.1 --port 1313
```

The expected local URL is:

```text
http://localhost:1313/blog/
```

Run a production-style local build with:

```bash
hugo --gc --minify
```

## Change Guidelines

- Do not add new dependencies unless explicitly requested.
- Prefer Hugo/PaperMod configuration, archetypes, Markdown, and small CSS changes
  over custom JavaScript or complex layout overrides.
- Keep changes small and reversible.
- If changing deployment behavior, verify `.github/workflows/deploy.yml` still
  installs Hugo Extended and builds for GitHub Pages.
- For content-only changes, a Hugo build is the main verification.
- For config, theme, or CSS changes, run the Hugo build and inspect local preview
  when practical.
