---
title: "API、SDK、SDK 应用到底是什么"
date: "2026-05-16T00:00:00+05:00"
draft: false
slug: "api-sdk-sdk-application-explained"
summary: "用 Claude Agent SDK 和常见平台调用场景解释 API、SDK、SDK 应用三者的区别，以及什么时候直接调 API，什么时候用 SDK。"
topics:
  - "ai"
---

写 MCP、Agent、OpenAI SDK、Claude Agent SDK、LangChain 或企业内部平台时，经常会遇到三个词：

```text
API
SDK
SDK 应用
```

这三个词很容易混。可以先用一句话区分：

```text
API 是平台提供的能力入口。
SDK 是平台给开发者准备的代码工具包。
SDK 应用是你用这个工具包写出来的程序。
```

## API 是什么

API 是 **Application Programming Interface**，应用程序编程接口。

你可以把它理解成：平台对外暴露的一组规则，告诉你“怎么请求我、传什么参数、会返回什么结果”。

比如一个 AI 平台可能提供这样的 API：

```text
POST https://api.example.com/v1/messages
```

你要自己处理：

- 请求地址
- HTTP method
- headers
- API key
- JSON body
- 流式返回
- 错误码
- 重试

直接调用 API 的好处是控制力强，坏处是细节多。

## SDK 是什么

SDK 是 **Software Development Kit**，软件开发工具包。

SDK 通常是平台官方或社区封装好的一组代码，让你不用每次从 HTTP 请求开始写。

一个 SDK 里通常会包含：

- 客户端对象
- 函数和类
- 类型定义
- 鉴权封装
- 请求和响应格式封装
- 流式返回处理
- 错误处理
- 示例代码

比如 Claude Agent SDK 提供了 `query()`。你不用自己拼 HTTP 请求，而是直接写：

```ts
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "帮我总结这个项目"
})) {
  console.log(message);
}
```

SDK 的价值是：**把平台 API 的细节包装成更容易使用的代码接口。**

## SDK 应用是什么

SDK 应用不是一种特殊产品。

它就是：

> 你自己写的、调用某个 SDK 来使用平台能力的程序。

比如你写了一个 TypeScript 脚本，用 Claude Agent SDK 调 `query()`，并让它总结项目、连接 MCP server、调用 GitHub 工具，那么这个脚本就是一个 Agent SDK 应用。

关系是：

```text
Claude API / 平台能力
  ↓
Claude Agent SDK
  ↓
你写的程序 = SDK 应用
```

类比一下：

```text
你直接使用 Claude Code = 使用现成应用
你用 Claude Agent SDK 写一个自动 review PR 的程序 = SDK 应用
```

## 不用 SDK 怎么调用 API

不用 SDK 时，你通常要自己写请求。

伪代码大概是：

```ts
const response = await fetch("https://api.example.com/v1/messages", {
  method: "POST",
  headers: {
    "content-type": "application/json",
    "authorization": `Bearer ${process.env.API_KEY}`
  },
  body: JSON.stringify({
    model: "some-model",
    messages: [
      { role: "user", content: "帮我总结这个项目" }
    ]
  })
});

const data = await response.json();
console.log(data);
```

这当然能用，但真实项目里还要处理更多问题：

- 流式输出怎么读
- 请求失败怎么重试
- token 用量怎么解析
- 类型怎么约束
- 工具调用怎么表示
- 长连接和超时怎么处理

SDK 的作用就是把这些重复细节封装起来。

## 用 SDK 后省掉什么

用 SDK 后，你通常不用关心底层 HTTP 细节，而是直接使用更贴近业务的函数。

比如：

```ts
for await (const message of query({
  prompt: "帮我总结这个项目",
  options: {
    allowedTools: ["Read", "Grep"]
  }
})) {
  console.log(message);
}
```

这里你关注的是：

- prompt 是什么
- 允许哪些工具
- 如何处理返回消息

而不是：

- HTTP endpoint 是哪个
- SSE 怎么解析
- tool use message 怎么组装
- 每种错误码怎么处理

这就是 SDK 的意义：**让开发者把注意力放在应用逻辑上，而不是底层协议细节上。**

## Claude Agent SDK 举例

假设你想写一个程序：读取项目文件，并让 Claude 总结当前项目。

用 Claude Agent SDK 可以这样写：

```ts
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "阅读这个项目并总结它的用途、目录结构和运行方式",
  options: {
    allowedTools: ["Read", "Grep", "Glob"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

这个程序就是 SDK 应用。

如果再接 MCP server：

```ts
for await (const message of query({
  prompt: "列出 GitHub 仓库最近 3 个 issue",
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
  console.log(message);
}
```

这里：

- Claude Agent SDK 负责运行 agent。
- `mcpServers` 负责连接 GitHub MCP Server。
- `allowedTools` 负责授权 Claude 调用 GitHub 工具。
- 你写的这段程序就是 Agent SDK 应用。

## OpenAI SDK 举例

OpenAI SDK 也是同样道理。

你可以直接调用 HTTP API，也可以用 SDK：

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY
});

const response = await client.responses.create({
  model: "gpt-4.1",
  input: "用一句话解释 SDK 是什么"
});

console.log(response.output_text);
```

这里：

- OpenAI API 是平台能力入口。
- `openai` npm 包是 SDK。
- 你写的脚本是 SDK 应用。

不同平台的 SDK 长得不一样，但关系是一样的。

## 什么时候直接调 API，什么时候用 SDK

优先用 SDK 的场景：

- 官方 SDK 成熟稳定。
- 你需要流式输出、工具调用、结构化消息等复杂能力。
- 你希望少写协议细节。
- 你想获得类型提示和更好的错误封装。

可以直接调 API 的场景：

- SDK 还没支持某个新功能。
- 你只需要一个极小的 HTTP 请求。
- 你在不方便安装 SDK 的环境里。
- 你需要完全控制请求细节。

实际工作里常见做法是：

```text
默认用 SDK
SDK 不支持或太重时，再直接调 API
```

## 常见误区

误区一：SDK 等于 API。

不是。API 是平台能力入口，SDK 是对 API 的代码封装。

误区二：SDK 应用是官方产品。

不一定。SDK 应用通常是你自己写出来的程序。

误区三：用了 SDK 就不用懂 API。

也不完全对。SDK 能省掉很多细节，但遇到认证、权限、错误码、流式返回、工具调用问题时，理解 API 背后的模型仍然重要。

误区四：直接调 API 更高级。

不是。直接调 API 控制力更强，但维护成本也更高。工程上应该选更稳定、更清晰、更少出错的方式。

## 总结

可以用这张关系图记住：

```text
平台能力 / 模型服务
  ↓
API：平台对外暴露的接口
  ↓
SDK：把 API 封装成好用的代码工具包
  ↓
SDK 应用：你用 SDK 写出来的程序
```

如果你只是使用 Claude Code、ChatGPT、Cursor，那你是在使用现成应用。

如果你用 Claude Agent SDK、OpenAI SDK 或其他平台 SDK 写自己的自动化程序、网站后端、命令行工具、内部平台，那你就在写 SDK 应用。
