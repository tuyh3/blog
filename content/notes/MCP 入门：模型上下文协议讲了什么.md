---
title: "MCP 入门：模型上下文协议讲了什么"
date: "2026-05-13T11:46:06+05:00"
draft: false
slug: "mcp-introduction"
summary: "先用一次 GitHub issue 查询链路理解 MCP 的工作方式，再说明 Claude Agent SDK 中如何配置 MCP server、授权 tools、处理 transport、认证、tool search 和排障。"
topics: ["ai"]
---

## MCP 是什么

**MCP = Model Context Protocol，模型上下文协议。**

你可以先把它理解成一句话：

> MCP 是 AI 应用连接外部系统的统一接口。

外部系统可以是本地文件、GitHub、Slack、Postgres、浏览器、内部业务 API、搜索引擎或工作流平台。官方文档常用一个比喻：**MCP 像 AI 应用的 USB-C 接口**。不同 AI 应用不必为每个外部系统各写一套适配，只要双方都按 MCP 协议说话，就能连接。

但只看定义还是抽象。更容易理解的方式，是先看一次真实调用链。

## 先补一个概念：SDK 和 SDK 应用

SDK 是 **Software Development Kit**，中文通常叫软件开发工具包。你可以把它理解成平台官方给开发者准备的代码工具箱，用来更方便地调用平台能力。

不用 SDK 时，你可能要自己写 HTTP 请求、鉴权、流式返回解析、错误处理。用 SDK 时，这些细节会被封装成函数、类型和配置项。

在这篇文章里，**Agent SDK 应用** 指的是你自己用 Claude Agent SDK 写出来的 AI 程序。这个程序调用 `query()`，把 prompt、MCP server、`allowedTools` 等配置交给 Claude 执行。

说白了，就是把你原本会在 Claude Code 里手动输入的任务，写进代码里自动发给 Claude。

比如你本来会在 Claude Code 里问：

```text
请检查当前项目有没有潜在 bug，并给我一份 review 报告
```

换成 Agent SDK 应用，就是在程序里写：

```ts
for await (const message of query({
  prompt: "请检查当前项目有没有潜在 bug，并给我一份 review 报告",
  options: {
    allowedTools: ["Read", "Grep", "Bash"]
  }
})) {
  console.log(message);
}
```

关系可以简化成：

```text
Claude API / 平台能力
  ↓
Claude Agent SDK
  ↓
你写的程序 = Agent SDK 应用
```

Claude Code 是官方现成工具；Agent SDK 应用是你自己用 SDK 写出来的工具、自动化程序或后台服务。

## 先看一次 MCP 调用链

假设你在一个 Agent SDK 应用里问：

```text
帮我查 anthropics/claude-code 最近 3 个 issue
```

如果接入了 GitHub MCP Server，整个流程大致是：

```text
用户提问：查最近 3 个 GitHub issue
  ↓
Agent SDK 根据 mcpServers 启动 / 连接 GitHub MCP Server
  ↓
GitHub MCP Server 暴露工具：mcp__github__list_issues
  ↓
allowedTools 允许 Claude 调用这个工具
  ↓
Claude 判断需要查 GitHub，于是调用 mcp__github__list_issues
  ↓
MCP Server 使用 GITHUB_TOKEN 调 GitHub API
  ↓
GitHub API 返回 issue 数据
  ↓
MCP Server 把结果返回给 Claude
  ↓
Claude 整理成自然语言回复给用户
```

这条链路里，MCP 不是“让模型更聪明”的魔法。它解决的是：**模型怎么安全、规范、可复用地连接外部系统。**

## MCP 解决什么问题

没有 MCP 时，每个 AI 工具都要为每个外部系统写一套集成：

```text
Claude 接 GitHub：写一套
Cursor 接 GitHub：再写一套
自研 Agent 接 GitHub：再写一套
Claude 接 Postgres：写一套
自研 Agent 接 Postgres：再写一套
```

MCP 的思路是把外部系统包装成 MCP Server：

```text
GitHub MCP Server
Postgres MCP Server
Filesystem MCP Server
Slack MCP Server
      ↓
统一 MCP 协议
      ↓
Claude / Cursor / 自研 Agent / 其他 AI 应用
```

所以 MCP 的价值是：**外部系统只要暴露一次 MCP Server，多个 AI 应用就可以用统一方式接入。**

## 三个角色：Host / Client / Server

理解 MCP 先抓住三个角色。

| 角色 | 是什么 | 在 GitHub issue 例子里是谁 |
|---|---|---|
| Host | 运行 AI / agent 的应用 | Agent SDK 应用、Claude Code、Claude Desktop |
| Client | Host 里负责连接 MCP Server 的组件 | SDK 为 GitHub server 建立的连接 |
| Server | 暴露外部系统能力的程序 | GitHub MCP Server |

更直观一点：

```text
Host：你的 AI 应用
  ↓
Client：Host 里的 MCP 连接器
  ↓
Server：外部能力的提供方
  ↓
外部系统：GitHub / Postgres / 文件 / Slack / API
```

一个 Host 可以同时连接多个 MCP Server。每连接一个 Server，Host 里面通常会有一个对应的 Client 连接。所以常见关系是：

```text
1 个 Host : N 个 MCP Client : N 个 MCP Server
```

这里的重点不是背名词，而是记住责任边界：

- Host 负责运行 agent。
- Client 负责按 MCP 协议连接 server。
- Server 负责把外部系统包装成模型可用的能力。

## Server 暴露什么：Tools 为主，Resources / Prompts 为辅

通用 MCP 规范里，Server 可以暴露三类能力：

| 能力 | 谁使用 | 作用 | 例子 |
|---|---|---|---|
| Tools | 模型根据任务调用 | 执行动作或查询 | `list_issues`、`query`、`search_docs` |
| Resources | 用户或客户端附加 | 提供上下文数据 | README、数据库 schema、日志片段 |
| Prompts | 用户从菜单选择 | 复用任务模板 | `review_pr`、`summarize_doc` |

### Tools

**Tool 是模型可以调用的动作 / 函数。**

例如 GitHub MCP Server 可以暴露：

```text
list_issues(owner, repo)
get_pull_request(owner, repo, number)
search_code(query)
```

Postgres MCP Server 可以暴露：

```text
query(sql)
describe_table(table_name)
```

Agent SDK 这篇官方文档的主线就是 MCP tools：怎么配置 server、怎么允许 Claude 调用 tool、怎么认证、怎么排障。

### Resources

**Resource 是可读取的上下文数据。**

比如：

```text
file:///project/README.md
file:///project/docs/architecture.md
postgres://local/app/schema/users
```

Resource 更像“给 AI 看一份资料”。它不一定代表一个动作。

### Prompts

**Prompt 是 MCP Server 提供的可复用任务模板。**

比如一个 GitHub server 可能提供：

```text
review_pull_request(owner, repo, number)
```

Prompt 更像菜单里的任务入口；Tool 更像模型可主动调用的函数。新手最容易混的是 Prompt 和 Tool：**Prompt 通常由用户选，Tool 通常由模型根据任务决定是否调用。**

## Agent SDK 里怎么接 MCP Server

在 Claude Agent SDK 里，你不需要手动创建 MCP Client。你通过 `query()` 的 options 配置 `mcpServers`，SDK 会负责连接。

最小示例：

```ts
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "列出项目里的文件",
  options: {
    mcpServers: {
      filesystem: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
      }
    },
    allowedTools: ["mcp__filesystem__*"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

这里的几个关键点：

- `mcpServers.filesystem` 定义了一个名为 `filesystem` 的 MCP Server。
- `command + args` 表示这是一个本地 stdio server。
- `allowedTools` 明确允许 Claude 调用这个 server 暴露的工具。

也可以把 server 配到项目根目录的 `.mcp.json`：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    }
  }
}
```

默认 `query()` 会加载 project setting source，所以项目根目录的 `.mcp.json` 会被读取。如果你显式设置了 `settingSources`，要确保包含 `"project"`，否则这个文件不会生效。

## allowedTools 是关键权限开关

接上 MCP Server，不等于 Claude 能调用它的工具。

官方 Agent SDK 文档里很重要的一点是：**MCP tools 需要显式授权。**没有授权时，Claude 能看到工具存在，但不能调用。

MCP tool 的命名格式是：

```text
mcp__<server-name>__<tool-name>
```

例如：

```text
mcp__github__list_issues
mcp__postgres__query
mcp__filesystem__read_file
```

授权方式：

```ts
allowedTools: [
  "mcp__github__*",       // 允许 github server 的全部工具
  "mcp__postgres__query", // 只允许 postgres 的 query 工具
  "mcp__slack__send_message"
]
```

建议优先用 `allowedTools` 精确授权，而不是为了省事开启过宽的 permission mode。

原因很简单：

- `acceptEdits` 主要自动允许文件编辑和部分文件系统 Bash 命令，不会自动允许 MCP tools。
- `bypassPermissions` 会自动允许 MCP tools，但也绕过了其他安全确认，范围太大。
- `allowedTools` 可以只开放你要的 MCP server 或某几个 tool。

所以日常规则是：

```text
能写 allowedTools，就不要用 bypassPermissions 解决 MCP 授权。
```

如果 Claude 没有调用 MCP tool，第一件事就是检查 `allowedTools`。

## Transport 怎么选

MCP Server 可以用不同 transport 和 Agent SDK 通信。

| transport | 适合场景 | 配置形态 |
|---|---|---|
| stdio | 本地 server，本机命令启动 | `command` + `args` |
| HTTP | 远程 server，普通 HTTP endpoint | `type: "http"` + `url` |
| SSE | 远程 server，SSE endpoint | `type: "sse"` + `url` |
| SDK MCP server | 在 SDK 应用内直接定义工具 | 应用内代码 |

### stdio

如果 server 文档给你的是一条命令，比如：

```bash
npx -y @modelcontextprotocol/server-github
```

那通常就是 stdio。

```ts
mcpServers: {
  github: {
    command: "npx",
    args: ["-y", "@modelcontextprotocol/server-github"],
    env: {
      GITHUB_TOKEN: process.env.GITHUB_TOKEN
    }
  }
}
```

### HTTP / SSE

如果 server 文档给你的是 URL，就用 HTTP 或 SSE。

```ts
mcpServers: {
  "remote-api": {
    type: "http",
    url: "https://api.example.com/mcp",
    headers: {
      Authorization: `Bearer ${process.env.API_TOKEN}`
    }
  }
}
```

SSE 版本类似：

```ts
mcpServers: {
  "remote-api": {
    type: "sse",
    url: "https://api.example.com/mcp/sse"
  }
}
```

补一个细节：JSON 配置里 `"streamable-http"` 可以作为 `"http"` 的 alias；但在程序化 `mcpServers` options 里，应该写 `"http"`。

### SDK MCP server

如果你不是接已有外部 server，而是想在 SDK 应用内部直接定义工具，可以用 SDK MCP server。它适合把应用自己的函数包装成 MCP tool，不需要再单独启动一个 server 进程。

## 认证怎么处理

多数 MCP Server 都要凭据。GitHub 要 token，Slack 要 token，Postgres 要连接串，内部 API 要 bearer token。

先把认证问题拆开：

```text
认证：你是谁？token / key 是否有效？
授权：你能不能调用这个 tool？
订阅：你有没有付费？套餐和额度是否允许？
```

MCP 负责把认证信息带给 server，并提供授权承载方式；但“用户是否订阅”“额度是否用完”通常是服务商自己的后台系统判断，不是 MCP 协议本身负责。

### stdio 用 env 或 args

本地 stdio server 常用 `env` 传 token 或 API key：

```ts
mcpServers: {
  github: {
    command: "npx",
    args: ["-y", "@modelcontextprotocol/server-github"],
    env: {
      GITHUB_TOKEN: process.env.GITHUB_TOKEN
    }
  }
}
```

`.mcp.json` 里可以写：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

`${GITHUB_TOKEN}` 会在运行时从环境变量展开。

这种模式常见于本地启动的 server：

```text
Claude / SDK 应用
  ↓ stdio
本地 MCP Server
  ↓ 使用 env 里的 token
GitHub / Slack / 远程业务 API
```

注意：stdio server 是本地进程，但它要访问 GitHub、Slack、搜索服务或付费 API 时，仍然需要联网。只有 filesystem、git、local sqlite、local docs search 这类纯本地 server 才可能完全离线。

有些 server 把连接串放在 args 里，例如 Postgres：

```ts
const connectionString = process.env.DATABASE_URL;

mcpServers: {
  postgres: {
    command: "npx",
    args: ["-y", "@modelcontextprotocol/server-postgres", connectionString]
  }
}
```

这里的 Postgres 如果是本机数据库，可以不联网；如果是云数据库，就需要网络。

### HTTP / SSE 用 headers

远程 server 常用 header：

```ts
mcpServers: {
  "secure-api": {
    type: "http",
    url: "https://api.example.com/mcp",
    headers: {
      Authorization: `Bearer ${process.env.API_TOKEN}`
    }
  }
}
```

`Authorization: Bearer ...` 是最常见写法。这里的 token 可能是 API key，也可能是 OAuth access token。

### OAuth 2.1

MCP 规范支持 OAuth 2.1，但 Agent SDK 不会自动帮你跑完整 OAuth flow。你的应用要先完成 OAuth 授权，拿到 access token 后，再把 token 放进 MCP server 的 headers。

```ts
const accessToken = await getAccessTokenFromOAuthFlow();

mcpServers: {
  "oauth-api": {
    type: "http",
    url: "https://api.example.com/mcp",
    headers: {
      Authorization: `Bearer ${accessToken}`
    }
  }
}
```

说白了，OAuth 解决的是：

```text
第三方应用想访问你的账号资源
但你不想把账号密码交给它
```

典型流程是：

```text
1. 应用把你跳到服务商登录页
2. 你在服务商官方页面登录
3. 服务商问你是否授权
4. 你同意后，应用拿到 access token
5. 应用用这个 token 调 MCP Server 或背后的业务 API
```

首次 OAuth 授权基本需要联网。拿到 token 后，应用可以缓存一段时间；但只要要确认 token 是否撤销、是否过期、订阅是否有效，通常还是要联系授权服务器或服务商后台。

### API Key / License Key

很多付费 MCP 服务不会一开始就做完整 OAuth，而是先用 API key 或 license key。

用户订阅后，服务商后台生成：

```text
MCP_API_KEY=sk_xxx
```

本地 stdio server 可以这样配置：

```json
{
  "mcpServers": {
    "paid-search": {
      "command": "npx",
      "args": ["-y", "@vendor/paid-search-mcp"],
      "env": {
        "MCP_API_KEY": "${MCP_API_KEY}"
      }
    }
  }
}
```

远程 HTTP server 可以这样配置：

```ts
mcpServers: {
  "paid-search": {
    type: "http",
    url: "https://api.vendor.com/mcp",
    headers: {
      Authorization: `Bearer ${process.env.MCP_API_KEY}`
    }
  }
}
```

如果 server 只是检查“有没有 key”，不一定联网。但如果要判断 key 是否有效、订阅是否过期、额度是否用完，通常要联网请求服务商后台。

### Bearer token 和 JWT

HTTP / SSE 场景里经常看到：

```http
Authorization: Bearer <token>
```

这个 `<token>` 有两种常见形态：

- **opaque token**：随机字符串，server 通常要联网问授权服务器“这个 token 是否有效”。
- **JWT**：自包含 token，server 可以用公钥本地验证签名和过期时间。

JWT 可以减少每次联网校验的需求，但它不等于完全离线。订阅状态、额度、封禁、撤销授权这些动态信息，通常还是要查服务商后台，或者依赖短期缓存。

### 付费 MCP 怎么做订阅校验

付费 MCP 的本质通常是两层：

```text
认证层：API key / OAuth token 证明你是谁
订阅层：服务商后台判断你有没有资格用这个 tool
```

每次工具调用时，MCP Server 可能会检查：

```text
这个 token 属于哪个用户？
用户属于哪个 organization？
organization 买了哪个 plan？
这个 plan 是否允许当前 tool？
本月额度是否用完？
请求是否超出 rate limit？
```

所以付费 MCP 通常不是“MCP 协议自己收费”，而是：

```text
MCP Server 负责接入和暴露 tool
认证系统负责识别用户
订阅 / 计费系统负责判断能不能用
```

Stripe、Lemon Squeezy、自建 billing 系统都可以在这里出现。

### 哪些认证能力要求联网

不一定所有认证都要求联网。

| 场景 | 是否需要联网 |
|---|---|
| 读取本地文件、git、本地 sqlite | 通常不需要 |
| 检查本地 env 里有没有 API key | 不一定需要 |
| 用 API key 调 GitHub、Slack、搜索服务 | 需要 |
| OAuth 首次登录授权 | 需要 |
| opaque bearer token 在线校验 | 需要 |
| JWT 签名和过期时间校验 | 不一定需要 |
| 订阅状态、额度、封禁、撤销授权校验 | 通常需要 |

可以把规则记成一句话：

```text
认证格式本身不一定联网；
访问远程资源、OAuth 首次授权、订阅和额度校验通常需要联网。
```

## 工具太多、或者 Claude 忘了调用 tool 怎么办

MCP tool 通常不是传统代码里的：

```text
if condition:
  call_tool()
```

而是模型根据当前任务、系统提示、tool name、tool description、input schema、上下文和权限，判断是否需要调用。

所以会出现两类问题：

```text
工具太多：上下文里塞不下，模型也更难选。
工具没调用：明明该查外部系统，模型却凭已有上下文直接回答。
```

### Tool metadata 很重要

一个 MCP tool 通常不只是一个函数名，而是会暴露：

```text
name
description
input schema
```

模型判断是否调用 tool 时，会参考这些信息。

如果你用 FastMCP 写 server，可能没有显式写过 `description=`，但 description 仍然可能存在。FastMCP 通常会根据 Python 函数自动生成 tool metadata：

- 函数名变成 tool name
- docstring 变成 tool description
- 参数和类型注解变成 input schema

例如：

```python
from fastmcp import FastMCP

mcp = FastMCP(name="GitHubServer")

@mcp.tool
def search_github_issues(query: str, state: str = "open") -> list[dict]:
    """Search GitHub issues by keyword and state. Use when the user asks about bugs, feature requests, issue status, backlog, or repository planning."""
    ...
```

你没有单独写 `description=`，但 docstring 会成为模型看到的 tool description。

如果写成这样：

```python
@mcp.tool
def search(q: str):
    ...
```

模型只看到一个很模糊的 `search`，就更容易不用、误用或选错。

### 先把工具描述写清楚

不好的 tool metadata：

```text
name: search
description: Search things
```

更好的写法：

```text
name: search_github_issues
description: Search GitHub issues by keyword, state, label, assignee, or date range. Use when the user asks about bugs, feature requests, issue status, backlog, roadmap, or repository planning.
```

description 要尽量说明：

```text
这个 tool 能做什么
什么时候应该用
什么时候不该用
需要哪些输入
返回什么结果
```

### 把调用规则写进系统提示 / CLAUDE.md / agent instruction

如果某类问题必须查外部系统，不要只靠模型自己想起来。

可以把规则写在这些地方：

| 位置 | 适合放什么 |
|---|---|
| system prompt | 当前 SDK 应用的全局行为规则 |
| `CLAUDE.md` | 当前项目长期稳定的工作规则 |
| agent instruction | 某个专用 agent 的角色和工具使用规则 |
| SDK 应用 prompt | 当前这次任务的临时要求 |

例如，在系统提示、`CLAUDE.md` 或 agent instruction 里写：

```text
当用户询问 GitHub issue、PR、CI 状态或仓库历史时，必须先调用 GitHub MCP tool 获取实时数据，不要凭记忆回答。
```

或者：

```text
回答数据库结构、线上配置、工单状态、库存、最新日志时，必须调用对应 MCP tool。
```

这类规则可以显著提高 tool 调用率。

如果只是一次性任务，也可以直接写在用户 prompt 或 SDK 应用 prompt 里：

```text
请先调用 GitHub MCP 查询最新 issue，再总结。
```

判断放在哪里，可以用这个原则：

```text
长期项目规则 -> CLAUDE.md
某个 agent 的专用规则 -> agent instruction
当前 SDK 应用的全局规则 -> system prompt
一次性任务要求 -> prompt
```

### allowedTools 要配对

工具没调用，有时不是模型忘了，而是不能调。

要检查：

```ts
allowedTools: ["mcp__github__*"]
```

或者精确授权：

```ts
allowedTools: ["mcp__github__list_issues"]
```

如果只配置了 MCP server，但没有授权 tool，Claude 可能知道工具存在，但不能调用。

### 工具要少而准

工具太多时，即使有 tool search，选择也会变难。

更好的做法是：

```text
按任务只接入相关 MCP Server
allowedTools 收窄到当前任务需要的工具
tool name 和 description 写具体
避免多个工具功能重复
```

比如只想查 GitHub issue，就不要同时开放一堆浏览器、数据库、Slack、文件系统、部署平台工具。

### MCP tool search 解决的是上下文压力

MCP tool search 的思路是：

```text
不要一开始把所有工具定义都塞进上下文
先让 Claude 搜索可用工具
只在需要时加载相关工具定义
```

官方 Agent SDK 文档说 tool search 默认启用。对入门用户来说，先记住两点：

- 工具多时，不是所有 tool schema 都必须常驻上下文。
- tool search 缓解的是“工具太多占上下文”的问题，不保证模型一定选择正确工具。

所以如果 Claude 没调用本该调用的 MCP tool，排查顺序是：

```text
1. allowedTools 是否授权
2. tool name / description / input schema 是否清楚
3. prompt 或 instruction 是否明确要求先查外部系统
4. 是否暴露了太多无关工具
5. 是否需要在 SDK 应用里做 tool_use 后验检查
```

### 关键流程可以做后验检查

如果你写的是 SDK 应用，可以检查 Claude 是否真的调用了预期工具。

例如任务要求查 GitHub issue，但返回过程中没有出现：

```text
tool_use: mcp__github__list_issues
```

那程序可以追加一轮提示：

```text
你还没有调用 GitHub MCP tool。请先调用 mcp__github__list_issues 获取实时数据，再回答。
```

可以理解成三层兜底：

```text
第一层：用好 tool metadata 和 instruction，提高主动调用率
第二层：用 allowedTools，保证工具确实可调用
第三层：用程序检查 tool_use，没调用就纠偏
```

## 怎么排障

MCP 接不通时，不要只看最后的 result。Agent SDK 在每次 query 开始时会发一个 `system` message，`subtype` 是 `init`，里面包含 MCP servers 的连接状态。

可以这样看：

```ts
for await (const message of query({ prompt, options })) {
  if (message.type === "system" && message.subtype === "init") {
    console.log(message.mcp_servers);
  }
}
```

### Server 是 failed

如果某个 server 状态是 failed，常见原因有：

- 环境变量没设置，比如缺 `GITHUB_TOKEN`。
- npm 包或 server 命令不存在。
- Node.js 不在 PATH。
- 数据库连接串格式错。
- 远程 HTTP/SSE URL 不通。
- 防火墙或代理阻止连接。

### Claude 看得到工具但不调用

优先检查：

```ts
allowedTools: ["mcp__servername__*"]
```

没有授权时，Claude 可能知道工具存在，但不能调用。

然后再检查：

- server 名字和 tool 名字是否拼错。
- wildcard 是否写成了正确格式。
- tool name、description、input schema 是否太模糊。
- prompt 或 instruction 是否明确要求先查外部系统。
- 是否开放了太多无关工具。
- prompt 是否真的需要这个工具。

### 连接超时

Agent SDK 的 MCP server 连接默认 timeout 是 60 秒。如果 server 启动很慢，连接会失败。

处理思路：

- 先单独运行 server 命令，看它能不能正常启动。
- 检查 server 日志。
- 检查凭据和网络。
- 如果 server 很重，考虑预热或换更轻的 server。

## 两个常见例子

### 例子 1：查 GitHub issue

目标：让 Claude 查询 GitHub issue。

```ts
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "列出 anthropics/claude-code 最近 3 个 issue",
  options: {
    mcpServers: {
      github: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-github"],
        env: {
          GITHUB_TOKEN: process.env.GITHUB_TOKEN
        }
      }
    },
    allowedTools: ["mcp__github__list_issues"]
  }
})) {
  if (message.type === "system" && message.subtype === "init") {
    console.log("MCP servers:", message.mcp_servers);
  }

  if (message.type === "assistant") {
    for (const block of message.message.content) {
      if (block.type === "tool_use" && block.name.startsWith("mcp__")) {
        console.log("MCP tool called:", block.name);
      }
    }
  }

  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

这里最重要的是三件事：

- `env.GITHUB_TOKEN` 负责认证。
- `allowedTools` 负责授权。
- `system/init` 和 `tool_use` 日志负责排障。

### 例子 2：查 Postgres 数据库

目标：用户用自然语言问数据问题，Claude 写 SQL，通过 Postgres MCP Server 查询。

```ts
const connectionString = process.env.DATABASE_URL;

for await (const message of query({
  prompt: "上周每天新增了多少用户？按天拆分。",
  options: {
    mcpServers: {
      postgres: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-postgres", connectionString]
      }
    },
    allowedTools: ["mcp__postgres__query"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

这里建议只开放查询工具，不要随便开放写入类工具。数据库类 MCP Server 尤其要注意权限最小化。

## 什么时候该用 MCP，什么时候不用

适合用 MCP 的场景：

- 你要接的是外部系统，比如 GitHub、Slack、Postgres、浏览器、内部 API。
- 这个能力可能被多个 AI 应用复用。
- 你希望工具描述、认证、传输协议、权限控制有统一方式。
- 你不想为每个 agent 单独手写一套工具适配。

不一定需要 MCP 的场景：

- 只是当前 SDK 应用里一个很小的内部函数。
- 这个工具不会复用，也不需要独立 server。
- 你只需要一次性脚本。
- 用 Agent SDK custom tools 更直接。

我的理解：

```text
MCP：适合可复用的外部系统连接
Agent SDK custom tools：适合当前应用内的专用工具
普通 prompt / CLAUDE.md：适合指导模型怎么做，而不是连接外部系统
```

## 参考

- [Claude Agent SDK MCP 文档](https://code.claude.com/docs/en/agent-sdk/mcp)
- [Model Context Protocol 官网](https://modelcontextprotocol.io/)
- [MCP GitHub 组织](https://github.com/modelcontextprotocol)
- [Anthropic 发布 MCP 的博客（2024-11）](https://www.anthropic.com/news/model-context-protocol)
