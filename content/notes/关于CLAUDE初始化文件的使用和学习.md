---
title: "CLAUDE.md 使用笔记:从 /init 到日常工作流"
date: "2026-05-12T11:46:44+05:00"
draft: true
slug: "claude-md-usage-notes"
summary: "整理 CLAUDE.md 的定位、/init 的正确用法、应该写什么、几个进阶用法,以及最容易踩的坑。"
topics: ["ai"]
---

## CLAUDE.md / AGENTS.md 到底是什么

`CLAUDE.md` 是 Claude Code 会自动加载的项目级上下文文件。它给 Claude 提供项目结构、编码标准、常用命令和工作流等持久信息,会话开始时会被注入到上下文里。

`AGENTS.md` 是社区正在收敛的跨厂商约定,目标是替代各家自定义的 AI 项目配置文件(`.cursorrules`、`.windsurfrules` 等)。OpenAI Codex、Cursor、Aider、Sourcegraph 等都已支持。

**重要更正**:Claude Code 默认只读 `CLAUDE.md`,**不会自动识别 `AGENTS.md`**。如果想让 Claude Code 也用上 `AGENTS.md`,只有两种官方做法:

- 在 `CLAUDE.md` 里加 `@AGENTS.md` 导入(同时还能在下面追加 Claude 专属内容)。
- 直接建软链:`ln -s AGENTS.md CLAUDE.md`(Windows 上软链需要管理员权限,优先用 `@` 导入)。

所以这两个文件本质上是同一件事:

> 给 AI 的项目级长期上下文,告诉它"进了这个仓库应该怎么干活"。

本文重点放在 CLAUDE.md 的实战使用。AGENTS.md 作为跨工具背景出现,不展开;如果主要用 Codex / Cursor 等其他 agent,具体写法以它们各自的官方文档为准。

它适合写这些:

```
项目简介
关键目录
常用命令
测试命令
代码规范
架构约定
工作流要求
危险操作限制
应该优先阅读哪些文件
```

它不适合写这些:

```
某个功能的详细需求
当天的开发计划
完整架构长文
历史所有决策
临时 debug 记录
通用名词解释 / 词汇表
```

更准确的说法是:`CLAUDE.md` 会在会话开始时被加载,并和系统提示一起占据前置 token 预算,持续消耗模型的注意力。所以不是"重复发送贵",而是"挤占注意力预算"——写太长会让真正重要的指令被稀释。

## `/init` 是干什么的

在项目目录里运行 Claude Code,输入 `/init`,Claude 会分析代码库(package 文件、已有文档、配置、代码结构),然后生成一个初版 `CLAUDE.md`。

但重点是:**`/init` 生成的只是起点,不是最终答案**。

之后要做几件事:

```
检查生成内容是否准确
补充 Claude 无法自动推断的工作流
删除不适用于项目的泛泛规则
把文件提交到版本控制
```

不要把 `/init` 生成的 `CLAUDE.md` 当成圣旨。它只是 Claude 根据项目表面结构猜出来的版本,真正有价值的是后面逐步把真实工作习惯写进去。

## 建议 CLAUDE.md 里写什么

### 第一类:给 Claude 一张地图

告诉 Claude 项目结构、关键目录、主要依赖、架构模式和非标准组织方式。这样 Claude 进入代码库后能快速判断代码在哪里、应该改哪里。

比如一个 RAG 项目:

```
## Project Map

- `frontend/` - Next.js frontend
- `backend/` - FastAPI backend
- `backend/app/parser/` - document parsing
- `backend/app/chunker/` - chunking logic
- `backend/app/retrieval/` - dense/BM25/RRF/rerank pipeline
- `.ai/current-spec.md` - active feature spec
- `.ai/current-plan.md` - active implementation plan
```

这不是完整架构文档,只是一张"去哪找东西"的地图。

### 第二类:连接 Claude 到你的工具

Claude 继承你的开发环境,但它需要知道应该用哪些自定义工具、脚本、命令。测试命令、部署脚本、代码生成脚本、MCP 工具等,都应该写明什么时候用、怎么用。

例如:

```
## Common Commands

- `pnpm dev` - start frontend dev server
- `uvicorn app.main:app --reload` - start FastAPI backend
- `pytest tests/ -v` - run backend tests
- `pnpm lint` - run frontend lint
- `pnpm build` - production build check
```

减少 Claude 每次乱猜命令的概率。

### 第三类:定义标准工作流

让 Claude 直接跳进代码很容易造成返工,所以应该定义不同任务的标准流程。核心问题包括:是否需要先调查当前状态、是否需要详细计划、缺少什么信息、怎么测试有效性。

可以写:

```
## Default Workflow

Before coding:
1. Read `.ai/current-spec.md` if it exists.
2. If no spec exists, create a minimal spec first.
3. Explain the main flow in plain language.
4. Separate MVP scope from optional enhancements.

During coding:
1. Keep changes small.
2. Do not introduce new frameworks unless the spec requires it.
3. Preserve existing behavior unless the task explicitly changes it.

Before completion:
1. Run relevant tests/build commands.
2. Report exactly what was verified.
3. Update `.ai/debrief.md` if the design changed.
```

这是 `CLAUDE.md` 最有价值的部分:**把 AI 拉回你的工作节奏**。

## 三个额外技巧

### `/clear`:清理上下文

不同任务之间使用 `/clear`,因为长对话会积累很多无关上下文,降低当前任务的信噪比。`/clear` 清空当前对话,但 `CLAUDE.md` 会在新会话开始时重新加载,不会丢。

日常用法:

```
修完一个 bug → /clear
开始一个新功能 → /clear
从实现切换到安全 review → /clear 或使用 subagent
```

### subagents:不同阶段用不同上下文

实现完支付处理器后,用子代理做安全审查,而不是继续在同一个上下文里审。subagent 的真正价值不是"换一个 AI"——底层还是同一个模型——而是:

- **上下文隔离**:review 子代理看不到你"为什么这么写"的辩护性上下文,只看 diff,更接近黑盒视角。
- **角色和工具约束**:可以把 subagent 限定为只读、只用特定工具(例如禁用 Write/Edit),物理上无法"边审边改"。
- **token 预算独立**:不污染主会话。

日常分工:

```
主 Claude:写 specs / plan / 实现
security-auditor subagent:只看安全问题
test-reviewer subagent:只看测试覆盖
codex:做独立 diff review
```

### custom commands:把重复 prompt 变成命令

如果经常重复某些 prompt,比如"做性能分析""做安全审查",可以把它们写成 `.claude/commands/` 下的 Markdown 文件,文件名即命令名(例如 `security-review.md` → `/security-review`)。

常用候选命令:

```
/spec-review
/security-review
/rag-quality-check
/explain-diff
/feature-debrief
```

比每次手打一大段 prompt 稳定。

## 几个进阶用法

### CLAUDE.md 是分层加载的

Claude Code 实际会合并几个位置的 `CLAUDE.md`:

```
~/.claude/CLAUDE.md           # 用户全局(跨所有项目)
<repo>/CLAUDE.md              # 项目级
<repo>/<subdir>/CLAUDE.md     # 子目录级(进入该目录工作时叠加)
```

合理分工:个人编码偏好写进用户级,项目事实写进仓库级,子模块特殊约定写进对应子目录。

### 用 `@` 引用外部文件,而不是"告诉 Claude 去读"

`CLAUDE.md` 支持 `@path/to/file.md` 语法,把外部文件按引用加载进上下文:

```markdown
# CLAUDE.md
@docs/architecture.md
@.ai/current-spec.md
```

`@` 是结构化的强制加载;"For details, read X" 是软提示,Claude 不一定会读。需要稳定加载的内容用 `@`,可选参考用文字描述。

**一个常见误解**:`@` 不省 token。被引用的文件会在**会话启动时全量展开**进入上下文,效果等于直接把内容写在 `CLAUDE.md` 里。`@` 的价值是组织和复用,不是减小上下文。Anthropic 官方文档原话:

> Splitting into `@path` imports helps organization but does not reduce context, since imported files load at launch.

如果真的需要"只在改某类文件时才加载"的规则,要用 `.claude/rules/` 目录加 YAML `paths:` 前缀:

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API 规则
- 所有 endpoint 必须做输入校验
- 错误响应用统一格式
```

这种规则只在 Claude 读到匹配 `paths:` 的文件时才会加进上下文,适合 monorepo 里按模块/语言分区的局部规则。对小项目用 `@` 就够了。

### 用 `#` 快速沉淀规则

会话中以 `#` 开头输入的内容,Claude Code 会提示是否把它追加到 `CLAUDE.md` 或用户级 memory。发现某条规则反复要解释时,用 `#` 一行就固化下来,比手动开编辑器快。

### CLAUDE.md 和 `.claude/settings.json` 的分工

两个文件经常被混淆:

- `CLAUDE.md` 是给模型看的自然语言,属于**软约束**——模型可能不遵守。
- `.claude/settings.json` 是给运行时看的结构化配置(权限、hooks、MCP server),属于**硬约束**——运行时直接拦截。

例子:"不要运行 `rm -rf`" 写在 `CLAUDE.md` 里只是建议,模型可能违反;写进 `settings.json` 的 deny 规则是物理上禁止。一个完整的项目级 AI 规则通常两者都要配。

## CLAUDE.md 最容易写错的地方

### 错误 1:写太长

`CLAUDE.md` 会和系统提示一起占据前置注意力预算,巨大的综合文档会稀释关键指令的权重。更好的做法是把详细信息拆到其他 Markdown 文件,在 `CLAUDE.md` 里用 `@` 引用或文字提示:

```
@docs/architecture.md
For active work, read `.ai/current-spec.md` and `.ai/current-plan.md`.
```

### 错误 2:把临时任务写进 CLAUDE.md

不要写:

```
今天实现上传 PDF 功能。
```

这属于 spec。`CLAUDE.md` 应该写:

```
Before implementing any feature, read `.ai/current-spec.md`.
```

### 错误 3:塞敏感信息

不要把 API key、凭证、数据库连接串、安全漏洞细节写进 `CLAUDE.md`,尤其是它会被提交到版本控制时。

### 错误 4:写理论最佳实践,而不是真实流程

有效的 `CLAUDE.md` 应该解决真实问题:记录反复输入的命令、需要十分钟才能解释清楚的架构上下文、能防止返工的工作流。不要堆理论上好看但和你工作方式无关的"最佳实践"。

### 怎么知道写得有没有用

一个实用判据:在新会话里问 Claude——"如果我让你给这个项目加一个 X 功能,你会先做什么?" 看它的回答是否复述了 `CLAUDE.md` 里定义的工作流。如果它绕开了,说明那段指令要么写得不够强,要么位置太靠后。

## 日常 SOP

```
开始新功能:
1. /clear
2. 创建或更新 .ai/current-spec.md
3. 创建 .ai/current-plan.md
4. 再开始写代码

实现过程中:
1. 只按 current-spec 做
2. 新术语必须解释
3. 复杂方案必须分 MVP / later
4. 不允许偷偷扩大范围

完成前:
1. 跑测试 / build / lint
2. 让 Claude 写 debrief
3. 用 Codex 根据 AGENTS.md 做独立 review
4. 根据 review 修复
```

`CLAUDE.md` 会在每次会话开始和 `/clear` 之后由 Claude Code 自动加载,不需要手动让它读。

## 参考

[使用 CLAUDE.md 文件： 定制 克劳德 代码 为了 你的 代码库](https://claude.com/blog/using-claude-md-files)
