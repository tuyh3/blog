---
title: "Hooks 详细讲解：AI Coding 工具里的“自动化钩子”到底是什么"
date: "2026-05-14T09:35:00+05:00"
draft: false
slug: "claude-code-hooks-explained"
summary: "用 Claude Code Hooks 的官方 reference 串起 hook 的工作机制：事件、matcher、handler、输入输出、异步、安全边界和调试方式。"
topics: ["ai"]
---

你可以先把 hooks 记成一句话：

> **Hooks 是 AI Coding 工具在固定生命周期节点上预留的自动化插槽。产品定义“什么时候触发”，用户定义“触发后做什么”。**

所以 hooks 不是普通 prompt，也不是 `CLAUDE.md`，更不是 skill。它更像软件工程里的自动化守门员：当 AI 准备读文件、写文件、执行命令、请求权限、结束任务、压缩上下文、启动子代理或调用 MCP 工具时，系统自动运行你预先配置好的脚本、HTTP 请求、MCP 工具、prompt 或 agent 检查。

Claude Code 官方把 hooks 定义为会在 Claude Code 生命周期中特定点自动执行的用户自定义 shell 命令、HTTP endpoint 或 LLM prompt。事件触发后，Claude Code 会把事件上下文传给 hook；hook 再通过 exit code、stdout、stderr 或结构化 JSON 影响 Claude Code 的下一步行为。

## Hooks 解决的不是“AI 更聪明”，而是“流程更可控”

Hooks 的核心价值是 **确定性自动化**。

AI 生成代码是概率性的。它可能忘记跑测试，可能改错文件，可能把 `.env` 写进仓库，也可能每次都需要你手动提醒“先看 spec”。Hooks 的作用，是把重复、固定、可判断的动作变成系统自动执行。

常见场景可以分三类：

- **减少重复手工步骤**：文件写完后自动格式化、自动记录日志、自动通知你。
- **强制项目规则**：阻止 `rm -rf`、阻止改 `.env`、阻止 production 配置被随手修改。
- **动态注入上下文**：会话开始时把 git status、当前 spec、TODO 或 CI 状态提供给 Claude。

一句话：

```text
Skill / CLAUDE.md 让 AI “知道应该怎么做”
Hook 让系统 “到点自动执行”
```

## 先跑一个最小例子：桌面通知

先看一个能跑的 hook。

下面这个 hook 的意思是：当 Claude Code 需要你注意时，比如等待输入或等待权限确认，弹一个桌面通知。

```json
{
  "hooks": {
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude Code needs your attention\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

可以先贴进用户级配置：

```text
~/.claude/settings.json
```

然后在 Claude Code 里输入：

```text
/hooks
```

检查 `Notification` 事件下面有没有这条 command hook。接着让 Claude 做一个需要权限确认的动作，切到别的窗口，如果系统弹出通知，就说明 hook 已经生效。

Linux 可以把 `osascript` 换成 `notify-send`；Windows 可以用 PowerShell 通知方案。这个例子不复杂，但它已经包含了 hook 的核心结构：事件触发，然后执行一个 handler。

## Hook 配置写在哪里

Claude Code hooks 最常见的配置位置是 settings JSON：

| 位置 | 作用范围 | 是否适合提交 |
|---|---|---|
| `~/.claude/settings.json` | 当前用户的所有项目 | 不提交 |
| `.claude/settings.json` | 当前项目，团队共享 | 可以提交 |
| `.claude/settings.local.json` | 当前项目，本地个人配置 | 不提交，应该 gitignore |

这三个位置是日常最常用的。

官方 reference 还列了其他位置：

| 位置 | 作用 |
|---|---|
| Managed policy settings | 组织级配置，由管理员控制 |
| plugin 的 `hooks/hooks.json` | 插件启用时生效，随插件分发 |
| skill 或 agent frontmatter | 当对应 skill / agent 激活时生效 |

所以更准确地说：settings JSON 是最常见入口，但不是唯一入口。

多个来源的 hooks 会合并生效。你可以用 `/hooks` 查看某条 hook 来自哪里。临时禁用所有 hooks 可以设置：

```json
{
  "disableAllHooks": true
}
```

注意：官方目前只提供“禁用全部 hooks”的配置，没有“保留配置但禁用某一个 hook”的单独开关。要移除某个 hook，就删掉对应 JSON 条目。

## Hook 的三层结构

Claude Code 的 hooks 配置有三层：

```text
hook event
  -> matcher group
    -> hook handler
```

官方 reference 用这三个词区分不同层级：

- **hook event**：生命周期节点，比如 `PreToolUse`、`PostToolUse`、`Stop`。
- **matcher group**：过滤条件，比如只匹配 `Bash` 或 `Edit|Write`。
- **hook handler**：真正执行的动作，比如 shell command、HTTP 请求、MCP tool、prompt 或 agent。

看一个完整例子：在 Claude 准备执行 Bash 命令前，阻止危险的 `rm -rf`。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

可以按这个流程理解：

```text
1. Claude 决定调用 Bash
2. Claude Code 触发 PreToolUse
3. matcher 检查工具名是不是 Bash
4. if 再检查 Bash 参数是否匹配 rm *
5. 命中后执行 block-rm.sh
6. 脚本从 stdin 读取事件 JSON
7. 脚本返回 allow / deny / block / context 等结果
8. Claude Code 根据结果决定是否继续
```

这里有两个关键点：

- `PreToolUse` 是事前 hook，所以能阻止工具调用。
- `PostToolUse` 是事后 hook，所以只能提醒或补救，不能撤销已经发生的副作用。

## 事件 event 怎么选

写 hook 的第一步不是写脚本，而是选对事件。

日常最常用的是这些：

| 事件 | 触发时机 | 适合做什么 |
|---|---|---|
| `SessionStart` | 新会话开始或恢复时 | 注入 git status、当前任务、TODO、环境提示 |
| `UserPromptSubmit` | 用户 prompt 提交后、Claude 处理前 | 检查 prompt 是否包含 secret，追加 sprint/spec 上下文 |
| `PreToolUse` | 工具真正执行前 | 阻止危险命令、阻止改敏感文件 |
| `PermissionRequest` | 出现权限确认请求时 | 自动允许安全命令，自动拒绝危险命令 |
| `Notification` | Claude Code 发送通知时 | 桌面通知、Slack/IM 通知 |
| `PostToolUse` | 工具成功执行后 | 自动格式化、自动 lint、记录日志 |
| `Stop` | Claude 准备结束本轮响应时 | 检查测试是否跑过，未完成则让 Claude 继续 |
| `SubagentStop` | 子代理结束时 | 检查报告质量、风险等级、文件位置 |
| `PreCompact` | 上下文压缩前 | 保存计划、决策、未完成事项 |

如果只是构建入门工作流，先掌握这些就够。

官方 reference 实际列出了完整事件地图：

| 分组 | 事件 |
|---|---|
| 会话级 | `SessionStart`、`SessionEnd`、`Setup` |
| 用户交互 | `UserPromptSubmit`、`UserPromptExpansion`、`Notification` |
| 工具调用 | `PreToolUse`、`PostToolUse`、`PostToolUseFailure`、`PostToolBatch` |
| 权限 | `PermissionRequest`、`PermissionDenied` |
| 响应结束 | `Stop`、`StopFailure` |
| 子代理 | `SubagentStart`、`SubagentStop` |
| 任务管理 | `TaskCreated`、`TaskCompleted` |
| Agent Team | `TeammateIdle` |
| 配置和上下文 | `InstructionsLoaded`、`ConfigChange`、`CwdChanged`、`FileChanged` |
| Worktree | `WorktreeCreate`、`WorktreeRemove` |
| 上下文压缩 | `PreCompact`、`PostCompact` |
| MCP elicitation | `Elicitation`、`ElicitationResult` |

这些事件不是都要背。更实用的判断是：

- 想在工具执行前阻止：用 `PreToolUse`。
- 想在工具执行后检查：用 `PostToolUse` 或 `PostToolBatch`。
- 想影响用户 prompt：用 `UserPromptSubmit`。
- 想影响 Claude 是否停下：用 `Stop`。
- 想处理长上下文：用 `PreCompact` / `PostCompact`。
- 想处理文件或配置变化：用 `FileChanged` / `ConfigChange` / `CwdChanged`。

## matcher 和 if 怎么过滤

`matcher` 是第一层过滤。

它不是永远匹配工具名。更准确地说：**不同事件的 matcher 匹配不同字段**。

| 事件类型 | matcher 匹配什么 |
|---|---|
| `PreToolUse` / `PostToolUse` / `PostToolUseFailure` / `PermissionRequest` / `PermissionDenied` | 工具名 |
| `SessionStart` | 启动来源，比如 `startup`、`resume`、`clear`、`compact` |
| `Notification` | 通知类型，比如 `permission_prompt`、`idle_prompt` |
| `SubagentStart` / `SubagentStop` | agent 类型 |
| `PreCompact` / `PostCompact` | 压缩原因，比如 `manual`、`auto` |
| `ConfigChange` | 配置来源，比如 user/project/local/policy/settings |
| `FileChanged` | 要监听的文件名 |
| `StopFailure` | 错误类型 |
| `InstructionsLoaded` | 规则加载原因 |
| `Elicitation` / `ElicitationResult` | MCP server 名称 |

有些事件不支持 matcher，比如 `UserPromptSubmit`、`PostToolBatch`、`Stop`、`TaskCreated`、`TaskCompleted`、`WorktreeCreate`、`WorktreeRemove`、`CwdChanged`。这些事件会在每次发生时触发；即使你写了 `matcher`，也会被忽略。

### matcher 的三种写法

官方规则可以简化成三类：

| 写法 | 含义 |
|---|---|
| `""`、`"*"` 或省略 | 匹配全部 |
| 只包含字母、数字、`_`、`|` | 精确匹配，`|` 表示多个精确值 |
| 包含其他字符 | 按 JavaScript regular expression 处理 |

例子：

```json
{ "matcher": "Bash" }
```

只匹配 Bash。

```json
{ "matcher": "Edit|Write" }
```

匹配 `Edit` 或 `Write`。

```json
{ "matcher": "^Notebook" }
```

因为包含 `^`，按正则处理，匹配以 `Notebook` 开头的工具名。

### if 比 matcher 更细

对工具相关事件，`matcher` 通常只过滤工具名，`if` 可以按工具参数继续过滤。

```json
{
  "matcher": "Bash",
  "hooks": [
    {
      "type": "command",
      "if": "Bash(git push *)",
      "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/check-push.sh",
      "args": []
    }
  ]
}
```

`if` 使用和 permission rules 相同的语法。注意几个坑：

- `if` 只在工具相关事件上生效：`PreToolUse`、`PostToolUse`、`PostToolUseFailure`、`PermissionRequest`、`PermissionDenied`。
- 在非工具事件上写 `if`，这个 handler 不会运行。
- 一个 `if` 里只能写一条 permission rule，没有 `&&`、`||` 或列表语法。
- 如果要多个条件，写多个 handler。
- Bash 命令会按 subcommand 匹配，例如 `npm test && git push` 也可能命中 `Bash(git push *)`。

### MCP 工具怎么匹配

MCP server 暴露的工具也会作为工具进入 hook 匹配体系。命名模式是：

```text
mcp__<server>__<tool>
```

例如：

```text
mcp__filesystem__read_file
mcp__github__search_repositories
```

匹配某个 server 下所有工具：

```json
{ "matcher": "mcp__github__.*" }
```

注意 `.*` 很重要。`mcp__github` 这种只含字母和下划线的 matcher 会被当作精确字符串，不会匹配任何具体工具。

## handler 类型怎么选

内层 `hooks` 数组里的每个对象都是 hook handler。官方支持五类：

| type | 做什么 | 适合场景 |
|---|---|---|
| `command` | 执行本地命令 | 最常用，适合 lint、test、grep、写日志、调用脚本 |
| `http` | 把事件 JSON POST 到 URL | 通知外部服务、调用内部平台 |
| `mcp_tool` | 调用已连接 MCP server 上的工具 | 复用已有 MCP 能力 |
| `prompt` | 调用一次 Claude 模型做判断 | 轻量规则判断、语义判断 |
| `agent` | 启动子代理做检查 | 需要读文件/搜索代码的复杂验证，experimental |

所有 handler 都支持这些常见字段：

| 字段 | 作用 |
|---|---|
| `type` | 必填，handler 类型 |
| `if` | 可选，只适用于工具事件 |
| `timeout` | 超时时间，command 默认 600 秒，prompt 默认 30 秒，agent 默认 60 秒 |
| `statusMessage` | hook 运行时显示的 spinner 文案 |
| `once` | 仅 skill frontmatter 中有效，一次 session 只运行一次 |

### command：最常用的 handler

`command` 会运行本地命令，并从 stdin 收到事件 JSON。

```json
{
  "type": "command",
  "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/check-style.sh",
  "args": []
}
```

官方建议：只要命令里引用 `${CLAUDE_PROJECT_DIR}`、`${CLAUDE_PLUGIN_ROOT}` 这类路径占位符，优先使用 `args` 形式。这样 Claude Code 会按 exec form 启动进程，不经过 shell tokenization，路径里有空格或特殊字符也更稳。

没有 `args` 时是 shell form，适合需要管道、重定向、`&&` 这类 shell 功能的场景。

### http：调用外部服务

`http` 会把 hook 事件 JSON 作为 POST body 发出去。它适合接企业内部平台、通知网关、审计系统。

注意：HTTP 的非 2xx 响应只是非阻塞错误。要真正阻止工具调用或拒绝权限，HTTP endpoint 需要返回 2xx，并在 body 里返回符合 hook JSON 输出格式的决策。

### mcp_tool：调用 MCP 工具

`mcp_tool` 会调用已经连接的 MCP server。

```json
{
  "type": "mcp_tool",
  "server": "my_server",
  "tool": "security_scan",
  "input": {
    "file_path": "${tool_input.file_path}"
  }
}
```

它不会触发 OAuth 或连接流程；server 必须已经连上。`SessionStart` 和 `Setup` 这类早期事件可能发生在 MCP server 完成连接之前，所以要能容忍“未连接”的非阻塞错误。

### prompt 和 agent：让模型做判断

`prompt` hook 会把 hook input 和你的 prompt 发给一个 Claude 模型做单轮判断，返回类似：

```json
{
  "ok": true,
  "reason": "..."
}
```

`agent` hook 类似 prompt，但会启动一个有工具访问能力的子代理，可以读文件、搜索代码、检查项目状态。官方标注 agent hook 是 experimental，生产规则建议优先用 `command` 或 `prompt`。

并不是每个事件都支持每种 handler。官方 reference 里把事件分成三类：

- 支持全部五类 handler：常见工具事件、`Stop`、`SubagentStop`、任务事件、用户 prompt 事件等。
- 只支持 `command`、`http`、`mcp_tool`：如 `Notification`、`ConfigChange`、`FileChanged`、`PreCompact` 等。
- `SessionStart` 和 `Setup`：支持 `command` 和 `mcp_tool`，不支持 `http`、`prompt`、`agent`。

## 输入输出与决策控制

这是 hooks 最容易写错的地方。

### 输入：hook 会收到什么

command hook 从 stdin 收到 JSON。HTTP hook 收到同样的 JSON，只是作为 POST body。

通用字段包括：

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/path/to/project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse"
}
```

工具事件还会有：

```json
{
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  }
}
```

不同事件有不同的额外字段。比如 `Notification` 有 `message`、`title`、`notification_type`；`PermissionDenied` 有 `reason`；`FileChanged` 有文件路径相关字段。写复杂 hook 前应该先把 stdin 打出来看真实结构。

### exit code：最简单的控制方式

command hook 可以用 exit code 表达结果：

| exit code | 含义 |
|---|---|
| `0` | 成功，Claude Code 会继续，并尝试解析 stdout 里的 JSON |
| `2` | 阻塞错误，stderr 会反馈给 Claude 或用户，具体效果取决于事件 |
| 其他非零 | 多数事件下是非阻塞错误，动作通常继续 |

最关键的规则：

```text
只有 exit 0 时，stdout JSON 才会被处理。
exit 2 时，stdout 和其中的 JSON 会被忽略，stderr 才是阻塞信息。
```

也就是说，不能一边 `exit 2`，一边期待 stdout 里的 JSON 决策生效。

如果 hook 是强制策略，比如阻止危险命令，要用 `exit 2` 或者 exit 0 + JSON decision。不要用 `exit 1`，因为在大多数事件里 `exit 1` 只是非阻塞错误。

### 哪些事件能被 exit 2 阻止

不是所有事件都能阻止。

可以阻止或回滚的典型事件：

- `PreToolUse`：阻止工具调用。
- `PermissionRequest`：拒绝权限。
- `UserPromptSubmit`：阻止 prompt 被处理。
- `UserPromptExpansion`：阻止 slash command 展开。
- `Stop`：阻止 Claude 停下，让它继续。
- `SubagentStop`：阻止子代理停下。
- `TaskCreated`：回滚任务创建。
- `TaskCompleted`：阻止任务标记完成。
- `PostToolBatch`：在下一次模型调用前停止 agentic loop。
- `PreCompact`：阻止上下文压缩。
- `WorktreeCreate`：任何非零 exit code 都会导致 worktree 创建失败。

不能撤销已经发生的典型事件：

- `PostToolUse`：工具已经成功执行，只能把 stderr 展示给 Claude。
- `PostToolUseFailure`：工具已经失败。
- `Notification`：只是通知。
- `SessionStart` / `SessionEnd`：会话生命周期事件。
- `InstructionsLoaded`：exit code 会被忽略。

### JSON 输出：更细的控制

除了 exit code，hook 还可以 exit 0 并输出结构化 JSON。

常见字段：

```json
{
  "continue": false,
  "stopReason": "Build failed, fix errors before continuing",
  "suppressOutput": false,
  "systemMessage": "Tests failed"
}
```

事件特定控制放在 `hookSpecificOutput` 里，并且要写 `hookEventName`。

例如给 Claude 注入上下文：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "This file is generated. Edit src/schema.ts instead."
  }
}
```

`additionalContext` 会作为系统提醒注入 Claude 的上下文，但不会像普通聊天消息那样显示出来。适合放“当前状态”“条件性规则”“外部系统结果”。静态项目规则不要用 hook 注入，优先写进 `CLAUDE.md`。

## 异步 hooks：长任务不要卡住 Claude

默认情况下，hook 会阻塞 Claude，等 hook 跑完后 Claude 才继续。

如果你要跑长任务，比如测试、部署、外部 API，可以用 command hook 的异步模式：

```json
{
  "type": "command",
  "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/run-tests.sh",
  "args": [],
  "async": true,
  "timeout": 300
}
```

异步 hook 的特点：

- 只有 `type: "command"` 支持 `async`。
- Claude 不等它完成，会继续工作。
- 异步 hook 不能阻止工具调用，也不能返回决策，因为它完成时原动作已经过去了。
- 输出会在下一轮对话送回 Claude；如果 session 空闲，要等下一次用户交互。
- `asyncRewake: true` 可以在 hook 以 exit 2 结束时唤醒 Claude，并把 stderr 或 stdout 作为 system reminder 交给它处理。

适合异步的任务：跑测试、慢速安全扫描、部署状态查询、外部服务通知。

不适合异步的任务：阻止危险命令、拒绝权限、阻止写敏感文件。

## 安全边界：hooks 权限很大

官方 reference 明确提醒：command hooks 以当前系统用户权限运行。你能读写什么，hook 就能读写什么。

这意味着它可以：

- 修改文件。
- 删除文件。
- 读取密钥。
- 访问 `.env`、`.git/`、SSH key、云凭据等敏感路径。

所以写 hook 要按安全脚本来写，而不是按普通 demo 来写。

基本原则：

- 不要盲信 stdin JSON，先校验字段。
- shell 变量要加引号：用 `"$VAR"`，不要裸写 `$VAR`。
- 检查路径穿越：拒绝包含 `..` 的路径。
- 用绝对路径或 `${CLAUDE_PROJECT_DIR}`。
- 跳过 `.env`、`.git/`、key、token、credential 文件。
- policy 型 hook 要用 `exit 2` 或结构化 JSON 决策，不要用普通 `exit 1`。
- 复杂规则先在本地项目里试，稳定后再考虑放到团队共享配置或 plugin。

## 调试 hooks 的方法

调试第一步：看 hook 是否注册成功。

```text
/hooks
```

`/hooks` 是只读浏览器。它会显示：

- 每个 event 下有多少 hooks。
- matcher 是什么。
- handler 类型是什么。
- 这条 hook 来自 user、project、local、plugin、session 还是 built-in。
- command、prompt 或 URL 具体是什么。

第二步：把 stdin 保存下来。

```bash
#!/usr/bin/env bash
set -euo pipefail

cat > /tmp/claude-hook-input.json
echo "hook received input at /tmp/claude-hook-input.json" >&2
exit 0
```

然后触发一次 hook，查看真实 JSON：

```bash
jq . /tmp/claude-hook-input.json
```

第三步：打开 debug log。

官方 reference 给了两种方式：

```bash
claude --debug
```

或者：

```bash
claude --debug-file /tmp/claude-debug.log
```

`--debug` 不会直接把日志打到终端，而是写到 `~/.claude/debug/<session-id>.txt`。如果要看更细的 matcher 命中细节，可以设置：

```bash
export CLAUDE_CODE_DEBUG_LOG_LEVEL=verbose
```

debug log 里会看到哪些 hooks 被匹配、执行了什么命令、exit code 是多少、stdout/stderr 是什么。

## 实战配方

### 1. 改文件后自动检查格式

适合 `PostToolUse + Edit|Write`：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/check-style.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

这类 hook 是事后检查。它可以提醒 Claude “刚才改坏了格式”，但不能阻止写文件本身。

### 2. 执行 Bash 前阻止危险命令

适合 `PreToolUse + Bash + if`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

这种规则应该尽量放在 `PreToolUse`，不要放在 `PostToolUse`。

### 3. Claude 准备停止前检查任务是否完成

适合 `Stop`：

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Evaluate if Claude should stop: $ARGUMENTS. Check whether requested tests and verification are complete."
          }
        ]
      }
    ]
  }
}
```

这适合轻量判断。如果需要真的读文件、跑测试、查日志，应该用 `command` 或 experimental 的 `agent`。

### 4. 会话开始时注入项目状态

适合 `SessionStart`：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/session-context.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

这个脚本可以输出 git branch、git status、当前 spec、TODO 摘要等。注意：长期不变的规范还是应该写进 `CLAUDE.md`，不要每次靠 hook 动态注入。

### 5. 文件变化后刷新环境

适合 `FileChanged`：

```json
{
  "hooks": {
    "FileChanged": [
      {
        "matcher": ".envrc|package.json",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/on-file-change.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

`FileChanged` 的 matcher 是要监听的文件名，不按普通工具 matcher 规则理解。

## 设计 hooks 的判断标准

不是所有规则都应该写成 hook。

适合 hook 的：

- 必须自动执行。
- 可以用脚本或结构化规则判断。
- 忘记执行会造成明显风险或浪费。
- 依赖当前动态状态，比如当前分支、文件路径、命令参数、CI 状态。

不适合 hook 的：

- 长期静态项目规范，优先写 `CLAUDE.md`。
- 专项复杂工作流程，优先写 skill。
- 需要人类判断的产品取舍，不要用 hook 假装自动化。
- 只偶尔用一次的个人提醒，不值得放进项目共享配置。

我的经验判断：

```text
CLAUDE.md：放长期稳定规则
Skill：放可复用专项能力
Hook：放固定时机必须自动运行的检查或动作
```

## 参考

- [Hooks Guide（官方教学，含桌面通知 demo 和 `/hooks` 命令）](https://code.claude.com/docs/en/hooks-guide)
- [Hooks Reference（完整事件表、handler 字段、`if` 过滤、JSON 输入输出 schema）](https://code.claude.com/docs/en/hooks)
- [Tools Reference（所有内置工具名，写 matcher 时的权威清单）](https://code.claude.com/docs/en/tools)
- [How to configure hooks（Anthropic 官方博客，事件名速查）](https://claude.com/blog/how-to-configure-hooks)
