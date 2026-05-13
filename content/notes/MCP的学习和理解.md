---
title: "MCP 入门：模型上下文协议讲了什么"
date: "2026-05-13T11:46:06+05:00"
draft: true
slug: "mcp-introduction"
summary: "MCP（模型上下文协议）入门笔记：Host / Client / Server 三个角色，Resources / Tools / Prompts 三类能力，以及它们各自解决什么问题。"
topics: ["ai"]
---

## MCP 是什么

**MCP = Model Context Protocol，模型上下文协议。**

官方定义是：MCP 是一个开源标准，用来把 AI 应用连接到外部系统，比如本地文件、数据库、搜索引擎、业务 API、工作流等。官方文档也用了一个比喻：**MCP 像 AI 应用的 USB-C 接口**，让不同 AI 应用可以用统一方式连接外部数据和工具。

## MCP 解决什么问题

没有 MCP 时，每个 AI 工具要接外部系统，都要单独开发适配：

```
Claude 接 GitLab：写一套
Cursor 接 GitLab：再写一套
Codex 接 GitLab：再写一套
Claude 接数据库：写一套
Cursor 接数据库：再写一套
```

这会变成很多重复集成。

MCP 的思路是：

```
GitLab MCP Server
Database MCP Server
File MCP Server
Browser MCP Server
      ↓
统一 MCP 协议
      ↓
Claude / Cursor / Codex / 其他 AI 应用
```

也就是说，外部系统只要通过 MCP Server 暴露能力，AI 应用就可以用统一协议接入。Anthropic 在发布 MCP 时也明确说，它是为了让开发者在数据源和 AI 工具之间建立安全的双向连接；开发者可以构建 MCP servers 暴露数据，也可以构建 MCP clients 去连接这些 server。

## MCP 里面有哪些角色

先掌握三个词：

```
MCP Host
MCP Client
MCP Server
```

### MCP Host

Host 是你正在使用的 AI 应用，比如：

```
Claude Desktop
Claude Code
Cursor
某个 AI IDE
你自己写的 AI 应用
```

它是“运行 AI 的外壳”。

### MCP Client

Client 是 Host 里面负责连接 MCP Server 的组件。

你可以先把它理解成：

> **Host 里面的 MCP 连接器。**

### MCP Server

Server 是真正暴露外部能力的程序。

比如：

```
GitLab MCP Server：暴露 issue、MR、仓库信息
Database MCP Server：暴露表结构、查询能力
File MCP Server：暴露本地文件读取能力
Browser MCP Server：暴露网页操作能力
```

Server 在部署上有两种主流形态：

- **stdio**：本地子进程。Host 启动一个 server 进程，通过标准输入/输出通信。文件、git、本地数据库这类场景几乎都用 stdio，是目前最常见的形态。
- **HTTP / SSE**：远程服务。适合云端或团队内网共享的 server。

所以整体结构是：

```
AI 应用 / Host
  ↓
MCP Client
  ↓
MCP Server
  ↓
外部系统：文件、数据库、API、业务系统
```

一个 Host 里可以同时连接多个 MCP Server——每连一个 Server，Host 内部就创建一个对应的 Client 实例。所以三者的数量关系是：**1 个 Host : N 个 Client : N 个 Server**，其中 Client 与 Server 是 1:1 对应。

### MCP Server 暴露什么能力

官方规范里，Server 主要向 Client 提供三类能力：

```
Resources
Prompts
Tools
```

MCP 规范明确说，Servers 可以向 Clients 提供：Resources、Prompts、Tools；其中 Resources 是上下文和数据，Prompts 是模板化消息和工作流，Tools 是给 AI 模型执行的函数。

#### Resources 是什么

**Resources = 可读取的上下文数据。**

比如：

```
文件内容
数据库 schema
项目配置
业务文档
日志片段
接口说明
```

官方文档说，Resources 让 server 向 client 共享可作为模型上下文的数据，比如文件、数据库 schema 或应用特定信息；每个 resource 都有 URI。

你可以理解成：

> **Resource 是“给 AI 看”的东西。**

比如文件 MCP Server 可以暴露：

```
file:///project/README.md
file:///project/docs/architecture.md
```

数据库 MCP Server 则会用类似 `postgres://localhost/mydb/schema/users` 这种 scheme 暴露表结构。

AI 读取这些资源后，可以把它们当上下文使用。

#### Tools 是什么

**Tools = AI 可以调用的动作 / 函数。**

官方规范说，Tools 允许 server 暴露可由语言模型调用的工具，用于查询数据库、调用 API 或执行计算；每个 tool 有名称和描述其输入输出的 schema。

你可以理解成：

> **Tool 是“让 AI 去做”的东西。**

比如：

```
search_gitlab_issues(query)
get_merge_request(project_id, mr_id)
query_database(sql)
search_documents(query)
create_ticket(title, description)
```

Resource 偏“读取上下文”，Tool 偏“执行动作”。

区别：

```
Resource：这里有一份数据，你可以读。
Tool：这里有一个能力，你可以调用。
```

#### Prompts 是什么

**Prompts = MCP Server 提供的可复用提示模板 / 工作流模板。**

官方规范说，Prompts 让 server 给 client 提供结构化消息和指令模板；client 可以发现可用 prompts，获取其内容，并传入参数定制。

比如一个 GitLab MCP Server 可以提供 prompt：

```
review_merge_request(project_id, mr_id)
```

它内部可能规定：

```
1. 读取 MR diff
2. 读取关联 issue
3. 检查测试结果
4. 输出 review 报告
```

你可以理解成：

> **Prompt 是“怎么让 AI 开始某类任务”的模板。**

#### Resources / Tools / Prompts 怎么区分

三者最大的区别不在“做什么”，在“谁发起”：

| | 谁发起 | 类比 |
|---|---|---|
| **Resource** | 用户或 Client 主动 attach | 给 AI 看一份文档 |
| **Tool** | 模型自己根据任务判断要不要调用 | function calling |
| **Prompt** | 用户从菜单里点选 | `/slash` 命令 |

新手最容易混的是 Prompt 和 Tool——它们的功能列表看起来很像，但 **Prompt 是用户触发，Tool 是模型触发**，这是两套完全不同的交互模型。

## 怎么实际用一个 MCP Server

以 Claude Code 为例，给它加一个 filesystem server：

```bash
# 添加一个 server（用 npx 直接拉官方包，免本地安装）
claude mcp add filesystem npx -y @modelcontextprotocol/server-filesystem ~/Documents

# 查看已连接的 server
claude mcp list

# 在会话里，Claude 会自动发现 server 暴露的 Tools / Resources / Prompts
```

社区里几个开箱即用的常用 server（都在 `@modelcontextprotocol/` 命名空间下）：

- `server-filesystem` — 读写本地文件
- `server-git` — 查 git log / diff / blame
- `server-github` — GitHub issue、PR、仓库
- `server-postgres` — 查 Postgres 数据库
- `server-puppeteer` — 浏览器自动化
- `server-slack` — Slack 工作区

第三方 server 还有很多，列表见 [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers)。

## 参考

- [Model Context Protocol 官网](https://modelcontextprotocol.io/)
- [MCP GitHub 组织](https://github.com/modelcontextprotocol)
- [Anthropic 发布 MCP 的博客（2024-11）](https://www.anthropic.com/news/model-context-protocol)
