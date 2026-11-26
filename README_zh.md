# AnySearch MCP Server

> [English](./README.md) | 简体中文

统一实时搜索 MCP 服务器，支持通用网页搜索、垂直领域搜索、并行批量搜索和页面正文提取。

官方 MCP 安装文档：[https://www.anysearch.com/docs/mcp-install](https://www.anysearch.com/docs/mcp-install)

如果想使用AnySearch的skill服务，详情可访问[AnySearch skill仓库](https://github.com/anysearch-ai/anysearch-skill)

如果想使用AnySearch的Rest API服务，详情可访问[AnySearch Rest API官网文档](https://www.anysearch.com/docs/api-endpoints)

## 功能

- **通用网页搜索** —— 开放式自然语言查询
- **垂直领域搜索** —— 支持金融、学术、安全、法律、代码等领域的结构化查询
- **并行批量搜索** —— 在一次调用中执行多个独立查询
- **URL 内容提取** —— 获取页面正文并提取为 Markdown
- **匿名访问** —— 无需 API key 即可使用（按 IP 限流，计入每日免费额度）

## API Key 配置

API key **是可选项，但建议配置**。即使没有 key，所有功能仍可通过匿名访问使用，但匿名访问按客户端 IP 限流，并计入每日免费额度。

当前公开额度为：Free 1,000 次/天、最高 10 QPS；Pro 最高 20 QPS；学生/开发者计划 2,000 次/天、20 QPS （https://www.anysearch.com/pricing）。实际额度和并发限制以账户及服务端响应为准。

### 注册获取 API Key（推荐）

智能体可以在**一次调用**中完成用户注册并获取 API key —— 无需验证码，无需手动注册。向用户索取一个**真实邮箱地址**：它将作为账户用户名。注册前必须获得用户明确同意；不要要求用户把 key 粘贴到聊天中，也不要回显、打印或提交 key。

```bash
curl -s -X POST "https://api.anysearch.com/v1/auth/email/register" \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com"}'
```

成功响应（`code: 0`）会返回账户信息和仅显示一次的明文 API key：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "username": "you@example.com",
    "email": "you@example.com",
    "login_url": "https://www.anysearch.com/login",
    "api_key": {
      "id": "key_xxxxxxxx",
      "key": "as_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      "key_prefix": "as_sk_xxxxxx...",
      "name": "default",
      "rate_limit": 20,
      "quota_limit": 0,
      "expires_at": null,
      "created_at": "2026-06-23T10:23:00Z"
    }
  }
}
```

`rate_limit` 等额度字段以实际响应为准，不要把示例值视为固定套餐。

成功后，智能体应当：

1. 告知用户其用户名（= 邮箱）和 `login_url`。
2. 在获得用户确认后，将 `data.api_key.key` 写入私有 MCP 配置或环境变量 `ANYSEARCH_API_KEY`，作为 `Authorization: Bearer <key>` 的值。它只显示一次（之后也可从控制台重新获取）。
3. 不要回显、打印、提交 key，也不要将 key 写入项目级共享配置或聊天记录。

错误响应可能使用 HTTP 状态码、应用级 `code` 或 `message`，也可能在 HTTP 200 下返回非零 `code`；不要假设所有错误的 `code` 都是 `-1`，也不要只看 HTTP 状态：

| message 特征                 | 处理方式                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| `"invalid_email"`            | 请用户重新输入邮箱                                           |
| `"email_already_registered"` | 邮箱已被注册 —— 告知用户在 `login_url` 登录；**不要**重试    |
| `"Rate limited"`             | 优先读取 `Retry-After`、`data.retry_after_seconds` 或 message 中的重试时间；未提供时间时停止连续请求，稍后再试或先使用匿名访问 |
| `"Key creation failed."`     | 账户已创建，但 key 创建失败 —— 从 message 中提取邮箱和 URL（`"Key creation failed. Your account <email> was created; sign in at <url>."`），告知用户在该地址登录并手动创建 key |
| `"Internal server error."`   | 稍后重试，或回退到匿名访问                                   |

> 该邮箱**必须真实且可接收邮件**。

### 获取 API Key

也可访问 https://anysearch.com/console/api-keys 创建免费 API key。

### Key 优先级

远程 MCP 端点只支持以下两种方式：

| 优先级    | 来源                                                         |
| --------- | ------------------------------------------------------------ |
| 1（最高） | `Authorization: Bearer <key>` 请求头（key 可通过环境变量 `ANYSEARCH_API_KEY` 由客户端注入） |
| 2         | 匿名访问（按 IP 限流，计入每日免费额度）                     |

> `--api_key` CLI 标志和 `.env` 文件读取仅适用于本地 CLI/SDK，不适用于托管 MCP 端点。

### Key 行为

| 场景                 | 行为                                                         |
| -------------------- | ------------------------------------------------------------ |
| 无 key               | 匿名访问；按客户端 IP 限流，计入每日免费额度                 |
| 有 key               | 通过 `Authorization: Bearer <key>` 请求头发送，使用 key 的付费额度并享有更高并发 |
| key 无效、禁用或过期 | 返回 HTTP `401` 或 `403`，**不会**静默回退到匿名访问         |
| 额度耗尽             | 返回 HTTP `402`                                              |
| 触发限流             | 返回 HTTP `429`，按 `Retry-After` 或响应体中的重试时间等待   |

## MCP 传输

线上服务端点为：

```text
https://api.anysearch.com/mcp
```

该端点原生使用 **Streamable HTTP**。当前版本的 OpenCode、Claude Code、Cursor、VS Code、Cline、OpenAI Codex、Google Antigravity、DeepSeek Harness、Hermes Agent 和 OMP 均可直接连接，无需 SSE 或 stdio 代理。官方 MCP 文档见 [AnySearch MCP Server Setup](https://www.anysearch.com/docs/mcp-install)。

以下配置依据各客户端当前官方文档整理。请使用支持远程 HTTP 的客户端版本；配置位置和环境变量语法可能随客户端版本变化，以客户端官方文档为准。

## 安装

API key 是可选项。若客户端官方文档说明了自定义 Header 的环境变量或密钥存储方式，下面默认给出推荐的鉴权配置；若未说明，则示例保持匿名，并说明如何只在私有用户配置中添加鉴权。其他示例如需匿名访问，只删除 `Authorization` 项，并保留 `X-Anysearch-Client`。

### OpenCode

官方文档：[MCP servers](https://opencode.ai/docs/mcp-servers/) 和 [配置文件位置](https://opencode.ai/docs/config/)。

全局配置使用 `~/.config/opencode/opencode.json`，项目配置使用项目根目录下的 `opencode.json`。Windows 全局路径为 `%USERPROFILE%\.config\opencode\opencode.json`。

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "anysearch": {
      "type": "remote",
      "url": "https://api.anysearch.com/mcp",
      "enabled": true,
      "oauth": false,
      "headers": {
        "Authorization": "Bearer {env:ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

OpenCode 使用 `{env:NAME}` 语法引用环境变量。`"oauth": false` 可避免 API key 鉴权服务触发不必要的 OAuth 探测。

保存后重启 OpenCode，运行 `opencode mcp list`，确认 `anysearch` 显示为 `connected`。

### Claude Code

官方文档：[通过 MCP 将 Claude Code 连接到工具](https://code.claude.com/docs/en/mcp)。

如需仅供当前用户使用、并在本机所有项目中生效，运行：

```bash
claude mcp add --transport http anysearch https://api.anysearch.com/mcp --scope user --header "Authorization: Bearer <your_api_key>" --header "X-Anysearch-Client: mcp/1.0.0"
```

`user` scope 会将服务写入 `~/.claude.json`，并在本机所有项目中提供。上面的 `--header` 会在执行时把 key 以明文写入配置；如果不想写入明文 key，请改用下面的项目级 `.mcp.json`，并在启动 Claude Code 前设置 `ANYSEARCH_API_KEY`。匿名访问时，省略 `Authorization` 对应的 `--header` 参数。

> Windows PowerShell 中环境变量写法为 `$env:ANYSEARCH_API_KEY`，引号和变量展开行为与 bash 不同。CLI 命令中的 `${VAR}` 写法没有官方文档保证，建议用 `.mcp.json` 做变量展开。

如需共享项目配置，在项目根目录创建 `.mcp.json`：

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

Claude Code 会在 `.mcp.json` 中展开 `${VAR}` 和 `${VAR:-default}`，包括 `url` 与 `headers`。启动 Claude Code 前需设置 `ANYSEARCH_API_KEY`。首次以交互方式打开项目级 MCP 服务时需要审批。

使用以下命令验证连接：

```bash
claude mcp get anysearch
claude mcp list
```

在 Claude Code 内运行 `/mcp` 可查看服务状态和可用工具。

### Cursor

官方文档：[Model Context Protocol](https://cursor.com/docs/mcp)。

项目配置使用 `.cursor/mcp.json`，全局配置使用 `~/.cursor/mcp.json`：

```json
{
  "mcpServers": {
    "anysearch": {
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

Cursor 会自动识别远程 HTTP 端点，并支持在 `url` 和 `headers` 中使用 `${env:NAME}` 环境变量插值。

可在 **Customize > MCPs** 或 **Output > MCP Logs** 查看连接；已安装 Cursor CLI 时也可运行：

```bash
agent mcp list
agent mcp list-tools anysearch
```

### VS Code

官方文档：[添加和管理 MCP 服务器](https://code.visualstudio.com/docs/agent-customization/mcp-servers) 和 [MCP 配置参考](https://code.visualstudio.com/docs/agents/reference/mcp-configuration)。

运行 **MCP: Open User Configuration** 可配置全局服务，也可在工作区创建 `.vscode/mcp.json`。下面使用密码输入变量，首次启动时提示输入 key 并安全保存，避免将其提交到仓库：

```json
{
  "inputs": [
    {
      "type": "promptString",
      "id": "anysearch-api-key",
      "description": "AnySearch API key",
      "password": true
    }
  ],
  "servers": {
    "anysearch": {
      "type": "http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${input:anysearch-api-key}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

如需匿名访问，删除 `Authorization` 项和整个 `inputs` 数组。

运行 **MCP: List Servers**，选择 `anysearch` 后执行 **Start** 或 **Show Output** 验证连接。注意：`${input:...}` 适用于本机 VS Code；需要跨 Agent Host/Copilot 复用时，请改用工作区 `.mcp.json` 等非交互式配置，具体位置以 VS Code/Copilot 官方文档为准。

### Cline

官方文档：[MCP](https://docs.cline.bot/mcp/mcp-overview)。

在 Cline 面板中打开 **MCP Servers > Configure > Configure MCP Servers**，也可从 **Remote Servers** 标签页添加托管端点。必须显式使用 `streamableHttp` 类型；省略时会为兼容旧配置而回退到 SSE。

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "streamableHttp",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

Cline IDE 支持在 `url`、`headers` 和 `env` 中使用 `${env:VAR}`。运行 `cline mcp` 可打开 CLI 交互式向导，`cline config mcp --json` 可列出配置。Cline 的配置文件位置可能随版本变化，建议通过 IDE 的 **Configure MCP Servers** 或 CLI 向导打开实际配置，不要依赖硬编码路径。

### OpenAI Codex

官方文档：[Model Context Protocol](https://developers.openai.com/codex/mcp/)。

全局配置使用 `~/.codex/config.toml`，受信任项目可使用 `.codex/config.toml`。Codex CLI、IDE 扩展和桌面应用共享该配置：

```toml
[mcp_servers.anysearch]
url = "https://api.anysearch.com/mcp"
bearer_token_env_var = "ANYSEARCH_API_KEY"
http_headers = { "X-Anysearch-Client" = "mcp/1.0.0" }
```

启动 Codex 前需设置 `ANYSEARCH_API_KEY`。匿名访问时删除 `bearer_token_env_var`。运行 `codex mcp list`，或在 Codex TUI 中运行 `/mcp`，确认服务及工具可用。

### Google Antigravity

官方文档：[Antigravity 中的 MCP 服务](https://antigravity.google/docs/ide/mcp)。

全局配置使用 `~/.gemini/config/mcp_config.json`，工作区配置使用 `.agents/mcp_config.json`。远程服务必须使用 `serverUrl`；不支持旧的 `url` 和 `httpUrl` 键：

```json
{
  "mcpServers": {
    "anysearch": {
      "serverUrl": "https://api.anysearch.com/mcp",
      "headers": {
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

Antigravity 官方文档目前只展示 Header 明文值，未说明远程 Header 的环境变量插值方式。如需鉴权，仅在私有全局配置中添加 `"Authorization": "Bearer YOUR_API_KEY"`，绝不要提交到 `.agents/mcp_config.json`。在 IDE 中打开 **MCP Servers > Manage MCP Servers**，或在 CLI 中运行 `/mcp`，可查看实时状态、日志和工具。

### DeepSeek Harness

官方文档：[DeepSeek Harness 集成](https://anysearch.com/docs/open-source-integrations/deepseek)。

推荐使用 AnySearch 专用插件（匿名额度可直接使用）：

```bash
npx -y @deepseek-ai/dsh plugin --profile web add @anysearch/anysearch-dsh
```

安装前确认 Node.js 22.19 或 24+、pnpm 11.7 可从 `PATH` 直接运行。安装后验证 profile：

```bash
npx -y @deepseek-ai/dsh --profile web --dump-config
npx -y @deepseek-ai/dsh web
```

`--dump-config` 输出应包含 `searchProvider: anysearch`、`fetchProvider: anysearch` 和 `web-search-anysearch` 插件条目。插件会接管原生 `web_search`/`web_fetch`，并额外提供：

- `anysearch_capabilities`
- `anysearch_search`
- `anysearch_batch_search`

API key 可选；如需账户额度，将 `ANYSEARCH_API_KEY` 写入 `~/.dsh/.credentials.yaml`，不要把 key 放进会话、命令或仓库。旧的通用 cordis MCP 配置（`@deepseek-ai/dsh-mcp-client`）仍可用，但建议优先使用上述专用插件。DeepSeek Harness 仍处于 developer preview，升级时应重新核对官方插件文档。

### Hermes Agent

官方文档：[MCP 集成](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/mcp.md)。

将服务添加到 `$HERMES_HOME/config.yaml`，并把 `ANYSEARCH_API_KEY` 放入 `$HERMES_HOME/.env` 或进程环境。macOS 和 Linux 的 `$HERMES_HOME` 默认为 `~/.hermes`；原生 Windows 安装器当前默认为 `%LOCALAPPDATA%\hermes`：

```yaml
mcp_servers:
  anysearch:
    url: "https://api.anysearch.com/mcp"
    headers:
      Authorization: "Bearer ${ANYSEARCH_API_KEY}"
      X-Anysearch-Client: "mcp/1.0.0"
```

运行 `hermes mcp test anysearch` 和 `hermes mcp list` 验证连接。如在会话中修改了配置，运行 `/reload-mcp` 或新建会话。

### OMP (Oh My Pi)

官方文档：[MCP 配置](https://github.com/can1357/oh-my-pi/blob/main/docs/mcp-config.md)。

项目配置使用 `.omp/mcp.json`，全局配置使用 `~/.omp/agent/mcp.json`：

```json
{
  "$schema": "https://raw.githubusercontent.com/can1357/oh-my-pi/main/packages/coding-agent/src/config/mcp-schema.json",
  "mcpServers": {
    "anysearch": {
      "type": "http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

OMP 会在配置值中展开 `${VAR}` 和 `${VAR:-default}`。设置 `ANYSEARCH_API_KEY` 后，在 OMP 中依次运行 `/mcp reload`、`/mcp test anysearch` 和 `/mcp list`。

### `search`

执行通用或垂直领域搜索查询，普通查询可直接调用。

| 参数                | 类型    | 必填 | 说明                                                         |
| ------------------- | ------- | ---- | ------------------------------------------------------------ |
| `query`             | string  | 是   | 自然语言搜索查询。每次调用只包含一个意图                     |
| `domain`            | string  | 否   | 垂直领域（例如 `finance`、`academic`、`security`）。必须来自 `get_sub_domains` 枚举 |
| `sub_domain`        | string  | 否   | 子领域路由 key（例如 `finance.us_stock`）。必须来自 `get_sub_domains` 输出 |
| `sub_domain_params` | object  | 否   | 来自 `get_sub_domains` params 列的结构化参数。**绝不要**臆造参数值 |
| `max_results`       | integer | 否   | 1–10，默认为 10                                              |

不传 `domain` 的通用搜索可以直接调用，无需先调用 `get_sub_domains`；一旦要传 `domain`，必须先调用 `get_sub_domains` 获取合法的 `sub_domain` 和参数定义。

### `get_sub_domains`

查询垂直领域目录。**使用 domain 进行任何搜索前都必须先调用** —— 返回有效的 sub_domains 及其参数 schema。

| 参数      | 类型     | 必填   | 说明                                          |
| --------- | -------- | ------ | --------------------------------------------- |
| `domain`  | string   | 二选一 | 要查询的单个领域                              |
| `domains` | string[] | 二选一 | 批量查询最多 5 个领域（推荐 —— 覆盖范围更广） |

返回 Markdown 表格：`sub_domain | description | params`

### `batch_search`

并行执行 1–5 个独立搜索查询。单个查询失败不会阻塞其他查询。

| 参数      | 类型     | 必填 | 说明                                           |
| --------- | -------- | ---- | ---------------------------------------------- |
| `queries` | object[] | 是   | 1–5 个查询对象，每个对象的字段与 `search` 相同 |

### `extract`

从 URL 获取页面正文并以 Markdown 返回。支持 HTML/XHTML、纯文本、JSON 和 Markdown；不支持 PDF、DOC/DOCX、图片、音视频、压缩包等二进制格式。HTML/纯文本内容超过 50,000 个字符时会被截断；超大 JSON/Markdown 会返回错误。

| 参数  | 类型   | 必填 | 说明                                |
| ----- | ------ | ---- | ----------------------------------- |
| `url` | string | 是   | 目标 URL（`http://` 或 `https://`） |

## 端到端示例（所有客户端通用）

下面两个示例分别对应**垂类搜索**和**通用搜索**，两者都会用到 `batch_search` 和 `extract`；真实使用时按 query 的信息类型选一个即可。

### 示例 1：垂类搜索（对比 React 与 Vue 的状态管理文档）

```text
帮我对比 React 和 Vue 官方文档中关于组件状态管理的写法，并给出可引用的文档来源。
```

这条 query 有明确的垂直领域（代码文档），流程是 3 步：`get_sub_domains` → `batch_search` → `extract`。

**1. `get_sub_domains` —— 取 code 的合法子领域与参数**

```json
{ "domains": ["code"] }
```

接口返回 2 个子领域及各自参数 schema：

| sub_domain     | 用途                                                         | 参数                           |
| -------------- | ------------------------------------------------------------ | ------------------------------ |
| `code.doc`     | 按库名与主题查询 npm/PyPI/Cargo 等生态的开发者文档与代码示例 | `library`（必填）              |
| `code.snippet` | 在 1M+ 公开 GitHub 仓库中按正则与语言搜索真实代码实现        | `lang`、`path`、`repo`（可选） |

`sub_domain` 和 `params` 的实际键名以本次返回为准。

**2. `batch_search`（垂类）—— 两个 `code.doc` 子查询并行取文档**

```json
{
  "queries": [
    {
      "query": "React 官方文档 组件状态管理 useState",
      "domain": "code",
      "sub_domain": "code.doc",
      "sub_domain_params": { "library": "react" },
      "max_results": 2
    },
    {
      "query": "Vue 官方文档 组件状态管理 ref reactive",
      "domain": "code",
      "sub_domain": "code.doc",
      "sub_domain_params": { "library": "vue" },
      "max_results": 2
    }
  ]
}
```

````text
## Query 1: React 官方文档 组件状态管理 useState

## Search Results (2 results, 2858ms)

### 1. useState – React 中文文档
- **URL**: https://zh-hans.react.dev/reference/react/useState
- ## 参考 ### `useState(initialState)` 在组件的顶层调用 `useState` 来声明一个 [状态变量](https://zh-hans.react.dev/learn/state-a-components-memory)。 ``` import { useState } from 'react';function MyComponent() { const [age, setAge] = useState(28); const [name, setName] = useState('Taylor'); const [todos, setTodos] = useState(() => createTodos()); // ... ``` 按照惯例使用 [数组解构](https://javascript.info/destructuring-assignment) 来命名状态变量，例如 `[something, setSomething]`。 ## 用法 ### 为组件添加状态 在组件的顶层调用 `useState` 来声明一个或多个 [状态变量](https://zh-hans.react.dev/learn/state-a-components-memory)。 ``` import { useState } from 'react';function MyComponent() { const [age, setAge] = useState(42); const [name, setName] = useState('Taylor'); // ... ``` 按照惯例使用 [数组解构](https://javascript.info/destructuring-assignment) 来命名状态变量，例如 `[something, setSomething]`。 `useState` 返回一个只包含两个项的数组： 1. 该状态变量 当前的 state，最初设置为你提供的 初始化 state。 2. `set`函数，它允许你在响应交互时将 state 更改为任何其他值。 要更新屏幕上的内容，请使用新状态调用 `set` 函数： ``` function handleClick() { setName('Robin');} ``` React 会存储新状态，使用新值重新渲染组件，并更新 UI。 ### 避免重复创建初始状态 React 只在初次渲染时保存初始状态，后续渲染时将其忽略。 ``` function TodoList() { const [todos, setTodos] = useState(createInitialTodos()); // ... ``` 尽管 `createInitialTodos()` 的结果仅用于初始渲染，但你仍然在每次渲染时调用此函数。如果它创建大数组或执行昂贵的计算，这可能会浪费资源。 为了解决这个问题，你可以将它 **作为初始化函数传递给** `useState`： ``` function TodoList() { const [todos, setTodos] = useState(createInitialTodos); // ... ``` 请注意，你传递的是 `createInitialTodos` **函数本身**，而不是 `createInitialTodos()` 调用该函数的结果。如果将函数传递给 `useState`，React 仅在初始化期间调用它。

### 2. useState – React
- **URL**: https://react.dev/reference/react/useState
- `useState` is a React Hook that lets you add a [state variable](https://react.dev/learn/state-a-components-memory) to your component. ``` const [state, setState] = useState(initialState) ``` ## Reference ### `useState(initialState)` Call `useState` at the top level of your component to declare a [state variable.](https://react.dev/learn/state-a-components-memory) ``` import { useState } from 'react';function MyComponent() { const [age, setAge] = useState(28); const [name, setName] = useState('Taylor'); const [todos, setTodos] = useState(() => createTodos()); // ... ``` The convention is to name state variables like `[something, setSomething]` using [array destructuring.](https://javascript.info/destructuring-assignment) ## Usage ### Adding state to a component Call `useState` at the top level of your component to declare one or more [state variables.](https://react.dev/learn/state-a-components-memory) ``` import { useState } from 'react';function MyComponent() { const [age, setAge] = useState(42); const [name, setName] = useState('Taylor'); // ... ``` The convention is to name state variables like `[something, setSomething]` using [array destructuring.](https://javascript.info/destructuring-assignment) `useState` returns an array with exactly two items: 1. The current state of this state variable, initially set to the initial state you provided. 2. The `set`function that lets you change it to any other value in response to interaction. To update what’s on the screen, call the `set` function with some next state: ``` function handleClick() { setName('Robin');} ``` React will store the next state, render your component again with the new values, and update the UI.

---

## Query 2: Vue 官方文档 组件状态管理 ref reactive

## Search Results (2 results, 4886ms)

### 1. Ref Unwrapping as Reactive Object Property
- **URL**: https://vuejs.org/guide/essentials/reactivity-fundamentals
- A ref is automatically unwrapped when accessed or mutated as a property of a reactive object. Use this to treat refs like normal properties inside reactive objects. ```js const count = ref(0) const state = reactive({ count }) console.log(state.count) // 0 state.count = 1 console.log(count.value) // 1 ```

### 2. Create reactive state with ref in script setup
- **URL**: https://vuejs.org/api/sfc-script-setup
- Reactive state must be explicitly created using Reactivity APIs. Refs are automatically unwrapped when referenced in templates. ```vue import { ref } from 'vue' const count = ref(0) {{ count }} ```
````

两条子查询互相独立，单条失败不影响另一条。

**3. `extract` —— 取可引用的官方文档正文**

```json
{ "url": "https://vuejs.org/guide/essentials/reactivity-fundamentals.md" }
```

````text
{"url":"https://vuejs.org/guide/essentials/reactivity-fundamentals.md","title":"","content":"---\nurl: /guide/essentials/reactivity-fundamentals.md\n---\n\n# Reactivity Fundamentals {#reactivity-fundamentals}\n\n## Declaring Reactive State \\* {#declaring-reactive-state}\n\nWith the Options API, we use the `data` option to declare reactive state of a component. ...（正文较长，此处省略）"}
````

`content` 里就是页面 Markdown 正文（此处为便于阅读已截断），可直接作为可引用的文档来源。

### 示例 2：通用搜索（MCP 官方文档）

```text
帮我查一下 Model Context Protocol 的官方文档和规范入口，并给出可引用的正文。
```

这条 query 没有垂直领域意图，直接走通用 `search`；需要并行比较多个来源时再用 `batch_search`，最后用 `extract` 取正文。

**1. `search`（通用）**

```json
{ "query": "Model Context Protocol 官方文档", "max_results": 10 }
```

````text
## Search Results (10 results, 237ms)

### 1. Model Context Protocol Specification – Model Context Protocol （MCP）
- **URL**: https://modelcontextprotocol.info/specification/
- The authoritative sources are the official specification pages at modelcontextprotocol.io/specification/2026-07-28 and the release announcement The 2026-07-28 Specification. Understand tool definitions, results, and errors: follow a tool call and compare protocol revisions. The Model Context Protocol (MCP) is an open protocol that enables seamless integration between LLM applications and external data sources and tools. Move a client across protocol eras: see the 2025-11-25 handshake and the 2026-07-28 stateless flow side by side, with runnable examples. Design tools a model can use: review tool descriptions, parameters, and error handling.

### 2. Model Context Protocol 官方文档｜MCP 协议规范与服务器开发官网 - Model Context Protocol 官网是 MCP 协议的官方文档站，说明客户端与服务器架构、SDK 用法、调试工具与示例，适合为 AI 应用接入数据库、文件与业务系统的开发者。|一方导航网
- **URL**: https://www.1fdh.com/sites/4810.html
- [极速社区是专业的 IDC 与站长综合交流社区，提供服务器IDC评测、域名主机、SEO 优化、网站搭建、源码资源及运维技术分享，助力广大站长高效建站、稳定运营。](https://www.1fdh.com/sites/3546.html) Model Context Protocol（简称 MCP）是一套开放协议，用来把 AI 应用与外部系统连起来。modelcontextprotocol.io 是该协议的官方文档站，规格、SDK 与示例都从这里查。 ## 数据评估 本站一方导航网提供的Model Context Protocol 官方文档｜MCP 协议规范与服务器开发都来源于网络，不保证外部链接的准确性和完整性，同时，对于该外部链接的指向，不由一方导航网实际控制，在2026年11月3日 下午8:33收录时，该网页上的内容，都属于合规合法，后期网页的内容如出现违规，可以直接联系网站管理员进行删除，一方导航网不承担任何责任。

### 3. 规范 – Model Context Protocol Specification
- **URL**: https://model-context-protocol.github.io/specification/
- ℹ️ 协议修订: 2024-11-05 Model Context Protocol (MCP) 是一个开放协议，能够在 LLM 应用程序和外部数据源及工具之间实现无缝集成。无论您是在构建 AI 驱动的 IDE、增强聊天界面，还是创建自定义 AI 工作流，MCP 都提供了一种标准... ℹ️ 协议修订: 2024-11-05 Model Context Protocol (MCP) 是一个开放协议，能够在 LLM 应用程序和外部数据源及工具之间实现无缝集成。无论您是在构建 AI 驱动的 IDE、增强聊天界面，还是创建自定义 AI 工作流，MCP 都提供了一种标准化的方式来连接 LLM 与所需的上下文。 本规范定义了权威的协议要求，基于 schema.ts 中的 TypeScript 模式。 有关实施指南和示例，请访问 model-context-protocol.mintlify.app。 本文件中的关键字 “MUST”、“MUST NOT”、“REQUIRED”、“SHALL”、“SH MCP 从 Language Server Protocol 中汲取了一些灵感，该协议标准化了如何在整个开发工具生态系统中添加编程语言支持。类似地，MCP 标准化了如何将额外的上下文和工具集成到 AI 应用程序的生态系统中。 ... Model Context Protocol 通过任意数据访问和代码执行路径提供强大的功能。随着这种能力的增强，所有实施者必须仔细解决重要的安全和信任问题。

### 4. 规范 – MCP 中文站（Model Context Protocol 中文）
- **URL**: https://mcpcn.com/specification/2026-07-28/
- # 规范 [模型上下文协议](https://modelcontextprotocol.io)（Model Context Protocol，简称 MCP）是一项开放协议，可将 LLM 应用与外部数据源和工具无缝集成。无论你是在构建 AI 驱动的 IDE、增强聊天界面，还是创建自定义 AI 工作流，MCP 都提供了一种标准化方式，将 LLM 连接到它们所需的上下文。 本规范基于 [schema.ts](https://github.com/modelcontextprotocol/specification/blob/main/schema/2026-07-28/schema.ts) 中的 TypeScript schema，定义了权威的协议要求。 有关实现指南和示例，请访问 [modelcontextprotocol.io](https://modelcontextprotocol.io)。 本文档中的关键词 “MUST”、“MUST NOT”、“REQUIRED”、“SHALL”、“SHALL NOT”、“SHOULD”、“SHOULD NOT”、“RECOMMENDED”、“NOT RECOMMENDED”、“MAY” 和 “OPTIONAL”，仅当它们以这里所示的全大写形式出现时，才应按照 [BCP 14](https://datatracker.ietf.org/doc/html/bcp14) \[[RFC2119](https://datatracker.ietf.org/doc/html/rfc2119)] \[[RFC8174](https://datatracker.ietf.org/doc/html/rfc8174)] 中的定义解释。 ## 概述 MCP 为应用提供了一种标准化方式，用于： - 与语言模型共享上下文信息 - 向 AI 系统公开工具和能力 - 构建可组合的集成和工作流 该协议使用 [JSON-RPC](https://www.jsonrpc.org/) 2.0 消息，在以下各方之间建立通信： - **主机（Hosts）**：发起连接的 LLM 应用 - **客户端（Clients）**：主机应用中的连接器 - **服务器（Servers）**：提供上下文和能力的服务 MCP 在一定程度上借鉴了[语言服务器协议](https://microsoft.github.io/language-server-protocol/)。后者标准化了如何在整个开发工具生态中增加对编程语言的支持；类似地，MCP 标准化了如何将额外的上下文和工具集成到 AI 应用生态中。

### 5. Model Context Protocol (MCP) specification
- **URL**: https://ozeki-ai-gateway.com/p_9224-model-context-protocol-mcp-specification.html
- # Model Context Protocol (MCP) specification Model Context Protocol (MCP) is an open protocol that enables seamless integration between LLM applications and external data sources and tools. Whether you're building an AI-powered IDE, enhancing a chat interface, or creating custom AI workflows, MCP provides a standardized way to connect LLMs with the context they need. This specification defines the authoritative protocol requirements, based on the TypeScript schema in [schema.ts](https://github.com/modelcontextprotocol/specification/blob/main/schema/2025-11-25/schema.ts). For implementation guides and examples, visit [modelcontextprotocol.io](https://modelcontextprotocol.io). The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14](https://datatracker.ietf.org/doc/html/bcp14) [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119) [RFC8174](https://datatracker.ietf.org/doc/html/rfc8174) when, and only when, they appear in all capitals, as shown here. ## Overview MCP provides a standardized way for applications to: - Share contextual information with language models - Expose tools and capabilities to AI systems - Build composable integrations and workflows The protocol uses [JSON-RPC](https://www.jsonrpc.org/) 2.0 messages to establish communication between: - **Hosts**: LLM applications that initiate connections - **Clients**: Connectors within the host application - **Servers**: Services that provide context and capabilities MCP takes some inspiration from the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/), which standardizes how to add support for programming languages across a whole ecosystem of development tools. In a similar way, MCP standardizes how to integrate additional context and tools into the ecosystem of AI applications. ## Security and Trust & Safety The Model Context Protocol enables powerful capabilities through arbitrary data access and code execution paths. With this power comes important security and trust considerations that all implementors must carefully address.

### 6. MCP Docs – Model Context Protocol （MCP）
- **URL**: https://modelcontextprotocol.info/docs/
- Model Context Protocol (MCP) documentation organised by reader task: get started, develop, connect and run, track protocol versions, and look up reference material for the 2026-07-28 revision. Start from what you are trying to do: meet MCP for the first time, build something, connect and run it, or track versions and look things up. Each chapter below opens with the page to read first and lists the pages that follow. All of them describe the current protocol revision, 2026-07-28. Tracking revisions, or moving a client or server between them? See the Protocol and Versions chapter for the version rules and the revision matrix, and Migrating Between Protocol Eras for the step-by-step migration.

### 7. 规范 – MCP 中文站（Model Context Protocol 中文）
- **URL**: https://mcpcn.com/specification/2025-06-18/
- 规范 # 规范 [模型上下文协议](https://modelcontextprotocol.io)（Model Context Protocol，简称 MCP）是一个开放协议，用于实现 LLM 应用与外部数据源和工具之间的无缝集成。无论您正在构建 AI 驱动的 IDE、增强聊天界面，还是创建自定义 AI 工作流，MCP 都提供了一种标准化的方式来连接 LLM 及其所需的上下文。 本规范基于 [schema.ts](https://github.com/modelcontextprotocol/specification/blob/main/schema/2025-06-18/schema.ts) 中的 TypeScript schema 定义了权威的协议要求。 有关实施指南和示例，请访问 [modelcontextprotocol.io](https://modelcontextprotocol.io)。 本文档中的关键词 “MUST”、“MUST NOT”、“REQUIRED”、“SHALL”、“SHALL NOT”、“SHOULD”、“SHOULD NOT”、“RECOMMENDED”、“NOT RECOMMENDED”、“MAY” 和 “OPTIONAL” 应按照 [BCP 14](https://datatracker.ietf.org/doc/html/bcp14) \[[RFC2119](https://datatracker.ietf.org/doc/html/rfc2119)] \[[RFC8174](https://datatracker.ietf.org/doc/html/rfc8174)] 中的描述进行解释，且仅当它们以全部大写形式出现时才适用，如此处所示。 ## 概述 MCP 为应用程序提供了一种标准化的方式来： - 与语言模型共享上下文信息 - 向 AI 系统暴露工具和能力 - 构建可组合的集成和工作流 该协议使用 [JSON-RPC](https://www.jsonrpc.org/) 2.0 消息在以下各方之间建立通信： - **主机（Hosts）**：发起连接的 LLM 应用程序 - **客户端（Clients）**：主机应用程序中的连接器 - **服务器（Servers）**：提供上下文和能力的服务 MCP 从[语言服务器协议](https://microsoft.github.io/language-server-protocol/)中获得了一些灵感，后者标准化了如何在整个开发生态系统的工具中添加对编程语言的支持。类似地，MCP 标准化了如何将额外的上下文和工具集成到 AI 应用程序的生态系统中。

### 8. 基础协议 – Model Context Protocol Specification
- **URL**: https://model-context-protocol.github.io/specification/basic/
- ℹ️ 协议修订: 2024-11-05 ... 错误。结果可以遵循任何 JSON 对象结构，而错误必须至少包含错误代码和消息。 协议层 Model Context Protocol 由多个关键组件组成，这些组件协同工作：... ℹ️ 协议修订: 2024-11-05 所有 MCP 客户端和服务器之间的消息 必须 遵循 JSON-RPC 2.0 规范。该协议定义了三种基本类型的消息： 类型 描述 要求 Requests 用于启动操作的消息 必须包含唯一 ID 和方法名称 Responses 回复请求的消息 必须包含与请求相同的 ID Notifications 无需回复的单向消息 不得包含 ID Responses 进一步分为 成功结果 或 错误。结果可以遵循任何 JSON 对象结构，而错误必须至少包含错误代码和消息。 协议层 Model Context Protocol 由多个关键组件组成，这些组件协同工作：

### 9. 规范 – MCP 中文站（Model Context Protocol 中文）
- **URL**: https://mcpcn.com/specification/2025-11-25/
- # 规范 [模型上下文协议](https://modelcontextprotocol.io)（Model Context Protocol，简称 MCP）是一个开放协议，用于实现 LLM 应用与外部数据源和工具之间的无缝集成。无论您正在构建 AI 驱动的 IDE、增强聊天界面，还是创建自定义 AI 工作流，MCP 都提供了一种标准化的方式来连接 LLM 及其所需的上下文。 本规范基于 [schema.ts](https://github.com/modelcontextprotocol/specification/blob/main/schema/2025-11-25/schema.ts) 中的 TypeScript schema 定义了权威的协议要求。 有关实施指南和示例，请访问 [modelcontextprotocol.io](https://modelcontextprotocol.io)。 本文档中的关键词 “MUST”、“MUST NOT”、“REQUIRED”、“SHALL”、“SHALL NOT”、“SHOULD”、“SHOULD NOT”、“RECOMMENDED”、“NOT RECOMMENDED”、“MAY” 和 “OPTIONAL” 应按照 [BCP 14](https://datatracker.ietf.org/doc/html/bcp14) \[[RFC2119](https://datatracker.ietf.org/doc/html/rfc2119)] \[[RFC8174](https://datatracker.ietf.org/doc/html/rfc8174)] 中的描述进行解释，且仅当它们以全部大写形式出现时才适用，如此处所示。 ## 概述 MCP 为应用程序提供了一种标准化的方式来： - 与语言模型共享上下文信息 - 向 AI 系统暴露工具和能力 - 构建可组合的集成和工作流 该协议使用 [JSON-RPC](https://www.jsonrpc.org/) 2.0 消息在以下各方之间建立通信： - **主机（Hosts）**：发起连接的 LLM 应用程序 - **客户端（Clients）**：主机应用程序中的连接器 - **服务器（Servers）**：提供上下文和能力的服务 MCP 从[语言服务器协议](https://microsoft.github.io/language-server-protocol/)中获得了一些灵感，后者标准化了如何在整个开发生态系统的工具中添加对编程语言的支持。类似地，MCP 标准化了如何将额外的上下文和工具集成到 AI 应用程序的生态系统中。

### 10. 规范（草案） – MCP 中文站（Model Context Protocol 中文）
- **URL**: https://mcpcn.com/specification/draft/
- # 规范（草案） **协议版本**: 草案 [模型上下文协议](https://mcpcn.com)（MCP）是一个开放协议， 使 LLM 应用程序与外部数据源和工具之间能够无缝集成。无论您是在构建 AI 驱动的IDE、 增强聊天界面，还是创建自定义 AI 工作流，MCP 都提供了一种标准化的方式将 LLM 与所需的上下文连接起来。 本规范定义了权威的协议要求，基于 [schema.ts](https://github.com/modelcontextprotocol/specification/blob/main/schema/draft/schema.ts) 中的 TypeScript 架构。 有关实现指南和示例，请访问 [mcpcn.com](https://mcpcn.com)。 本文档中的关键词"必须"、“禁止”、“要求”、“应当”、“不应当”、“应该”、“不应该”、 “推荐”、“不推荐”、“可以"和"可选"应按照 [BCP 14](https://datatracker.ietf.org/doc/html/bcp14) \[[RFC2119](https://datatracker.ietf.org/doc/html/rfc2119)] \[[RFC8174](https://datatracker.ietf.org/doc/html/rfc8174)] 中的描述进行解释， 当且仅当它们以全大写形式出现时，如此处所示。 ## 概述 MCP 为应用程序提供了一种标准化的方式来： - 与语言模型共享上下文信息 - 向 AI 系统暴露工具和功能 - 构建可组合的集成和工作流 该协议使用 [JSON-RPC](https://www.jsonrpc.org/) 2.0 消息在以下组件之间建立通信： - **主机**：启动连接的 LLM 应用程序 - **客户端**：主机应用程序内的连接器 - **服务器**：提供上下文和功能的服务 MCP 从[语言服务器协议](https://microsoft.github.io/language-server-protocol/)获得了一些灵感， 该协议标准化了如何在整个开发工具生态系统中添加对编程语言的支持。类似地， MCP 标准化了如何将额外的上下文和工具集成到 AI 应用程序的生态系统中。

```yaml
request_id: 8ac5d90993534c42a7cf112f55f4a027
```
````

**2. `batch_search`（通用）—— 并行比较文档与规范两个来源**

```json
{
  "queries": [
    { "query": "Model Context Protocol 官方文档", "max_results": 1 },
    { "query": "Model Context Protocol specification 2026-07-28", "max_results": 1 }
  ]
}
```

> 这里把 `max_results` 设为 `1` 只是为了示例更紧凑，不影响整个流程。

````text
## Query 1: Model Context Protocol 官方文档

## Search Results (1 results, 545ms)

### 1. Specification - Model Context Protocol
- **URL**: https://modelcontextprotocol.io/specification/2026-07-28/index
- Model Context Protocol (MCP) is an open protocol that enables seamless integration between LLM applications and external data sources and tools. Whether you’re building an AI-powered IDE, enhancing a chat interface, or creating custom AI workflows, MCP provides a standardized way to connect LLMs with the context they need.This specification defines the authoritative protocol requirements, based on the TypeScript schema in schema.ts.For implementation guides and examples, visit modelcontextprotocol.io.The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here. Overview MCP provides a standardized way for applications to: Share contextual information with language models Expose tools and capabilities to AI systems Build composable integrations and workflows The protocol uses JSON-RPC 2.0 messages to establish communication between: Hosts: LLM applications that initiate connections Clients: Connectors within the host application Servers: Services that provide context and capabilities MCP takes some inspiration from the Language Server Protocol, which standardizes how to add support for programming languages across a whole ecosystem of development tools. In a similar way, MCP standardizes how to integrate additional context and tools into the ecosystem of AI applications. Servers offer any of the following features to clients: Resources: Context and data, for the user or the AI model to use Prompts: Templated messages and workflows for users Tools: Functions for the AI model to execute Clients may offer the following features to servers: Elicitation: Server requests for additional information from users Additional Utilities Configuration Progress tracking Cancellation Error reporting Extensions Beyond the core protocol, MCP defines optional extensions that add modular, specialized, or experimental functionality. Extensions are always opt-in and require explicit support from both client and server, negotiated through capabilities that clients declare on each request and servers declare in their server/discover result. Notable extensions include: Tasks: Asynchronous execution of long-running operations, with polling, mid-flight input, and durable handles Skills over MCP: Rich, structured instructions for agent workflows, discovered and consumed through MCP The Model Context Protocol enables powerful capabilities through arbitrary data access and code execution paths. With this power comes important security and trust considerations that all implementors must carefully address. Key Principles User Consent and Control Users must explicitly consent to and understand all data access and operations Users must retain control over what data is shared and what actions are taken Implementors should provide clear UIs for reviewing and authorizing activities Data Privacy Hosts must obtain explicit user consent before exposing user data to servers Hosts must not transmit resource data elsewhere without user consent User data should be protected with appropriate access controls Tool Safety Tools represent arbitrary code execution and must be treated with appropriate caution. In particular, descriptions of tool behavior such as annotations should be considered untrusted, unless obtained from a trusted server. Hosts must obtain explicit user consent before invoking any tool Users should understand what each tool does before authorizing its use Implementation Guidelines While MCP itself cannot enforce these security principles at the protocol level, implementors SHOULD: Build robust consent and authorization flows into their applications Provide clear documentation of security implications Implement appropriate access controls and data protections Follow security best practices in their integrations Consider privacy implications in their feature designs Learn More Explore the detailed specification for each protocol component:

```yaml
request_id: 3c9b78643657466194f804bdc1df6444
```

---

## Query 2: Model Context Protocol specification 2026-07-28

## Search Results (1 results, 594ms)

### 1. MCP 2026-07-28 spec: stateless core, coming to Claude - Skip to main content
- **URL**: https://claude.com/resources/articles/bringing-mcp-2026-07-28-to-claude?utm_source=www.therundown.ai&utm_medium=newsletter&utm_campaign=google-s-viral-model-changes-ai-image-editing&_bhlid=2d078c877b5bb38f383c5e4b1b6921da95cbfc5a#1
- Skip to main content # Bringing MCP 2026-07-28 to Claude The fifth spec release of the Model Context Protocol, MCP 2026-07-28, is live today. The latest spec moves MCP to a stateless core, while hardening authorization and graduating official extensions. Support is being rolled out across Claude products. ## What's new in MCP MCP recently surpassed 400M monthly SDK downloads, a 4x increase this year, and has become the industry standard for connecting AI agents to applications. MCP 2026-07-28 is one of the most significant spec releases to date: Stateless core. MCP moves from a bidirectional stateful protocol to a request/response model. Servers can now deploy on serverless and edge infrastructure. This simplifies the experience of building MCP servers for Claude and scaling their usage as they grow in adoption. Standardized extensions. MCP Apps and Tasks now ship under a versioned extensions framework, giving developers a formal path to add capabilities like interactive UIs and long-running work without changing the core protocol. Auth hardening. Authorization now aligns with production OAuth 2.0 and OIDC deployments, so MCP servers connect to enterprise identity systems like Entra or Okta without workarounds. Companies across the ecosystem have been building on the new spec alongside the MCP community since beta: “More builders are using our MCP server to bring generated outputs into Figma's canvas, where they can explore, riff and refine them with their team into products that stand out. As that usage grows, our stateless architecture can scale with it, and with MCP Apps, Tasks, and Enterprise-Managed Auth, we can do even more to keep design and code together in one, connected flow.” “MCP is the industry standard for connecting AI agents to tools and data, and Intuit is proud to support the new MCP 2026-07-28 spec. The stateless protocol core and extensions framework, including MCP Apps and Tasks, let our technologists and customers build and connect agentic experiences at enterprise scale, and allow Intuit to continue delivering trusted financial intelligence experiences to its 100 million consumers and businesses, wherever they choose to work.” “The stateless core in the 2026-07-28 spec makes MCP a first-class HTTP workload with no session management to work around. Our customers wanted MCPs on Netlify to be as simple as the rest of the platform and this new spec unlocks this at its core. Building MCP Apps into the new extensions framework is a huge step forward for scalability, accessibility, and capability across the whole ecosystem.” “Moving MCP to a stateless protocol makes it easier to scale our own service and makes it easier for us to add analytics for our customers' MCP servers. This helps us show people how their MCP tools are being used and what tools are missing that their users would want to use. It's great to see this protocol growing in this direction.” “Anthropic pairs frontier models with a developer experience that keeps raising the bar. The stateless core in the open MCP 2026-07-28 spec reduces the complexity we manage, so we can ship more features to our customers, faster and at scale.” “At Zoom, we believe organizational context is what enables AI to deliver meaningful work, which is why we've built MCP servers that securely bring Zoom meeting intelligence into AI platforms like Claude. The new MCP spec makes it far easier to deploy and scale MCP servers on standard HTTP infrastructure — so users get Zoom's meeting intelligence faster and more reliably, right inside the AI workflows they depend on every day.” 1/6 See the MCP 2026-07-28 release announcement for full details on the new spec. ## Advancing MCP in Claude Claude now lists over 950 MCP servers in the connectors directory, used by millions of people every day. This year we shipped support for new protocol extensions alongside features that make MCP easier to build on and deploy: MCP Apps let servers render interactive UI directly in the conversation. Users can see what a connector is doing and work with it inline, without switching tabs. Enterprise-managed auth lets admins provision MCP connectors for their whole organization through their identity provider. Admins authorize a connector once, users inherit access through their existing IdP groups, and it's connected on first login: zero-touch setup for the end user. Observability for developers building connectors gives published connectors in our directory a dashboard showing how they perform across Claude product surfaces. Developers can use it to track adoption, diagnose errors and latency, and break down usage by product. MCP tunnels (research preview) connect Claude to MCP servers inside a private network without exposing them to the public internet. Teams can bring internal tools to Claude with no inbound firewall rules, no public endpoints, and no IP allowlisting on the origin. The stateless core, standardized extensions, and hardened auth in 2026-07-28 will help developers bring more applications to Claude, with a lower-friction, more consistent end-user experience. We'll continue investing in MCP as an open standard alongside the community, and in the Claude features that make MCP more accessible and effective in production. ## Getting started Explore the spec and SDKs to get started. Support is rolling out across Claude products soon. If you’re planning to submit your MCP server to Claude’s connectors directory, you can learn more here.

```yaml
request_id: a7e3702658e246c5a1ba720ed9c0c435
```
````

**3. `extract` —— 取可引用的规范仓库正文**

```json
{ "url": "https://raw.githubusercontent.com/modelcontextprotocol/modelcontextprotocol/main/README.md" }
```

````text
{"url":"https://raw.githubusercontent.com/modelcontextprotocol/modelcontextprotocol/main/README.md","title":"","content":"# Model Context Protocol (MCP)\n\n_Just heard of MCP and not sure where to start? Check out our [documentation website](https://modelcontextprotocol.io)._\n\nThis repo contains the:\n\n- MCP specification\n- MCP protocol schema\n- Official MCP documentation\n\nThe schema is [defined in TypeScript](schema/2026-07-28/schema.ts) first, but\n[made available as JSON Schema](schema/2026-07-28/schema.json) as well, for wider\ncompatibility.\n\nThe official MCP documentation is built using Mintlify and available at\n[modelcontextprotocol.io](https://modelcontextprotocol.io).\n\n## Authors\n\nThe Model Context Protocol was created by David Soria Parra ([@dsp](https://github.com/dsp)) and Justin Spahr-Summers ([@jspahrsummers](https://github.com/jspahrsummers)).\n\n## Contributing\n\nContributions are made through [working groups](https://modelcontextprotocol.io/community/working-interest-groups)\nand issues:\n\n- **Changes to the specification** are discussed and approved in the relevant working group\n  before a pull request or [SEP](https://modelcontextprotocol.io/community/sep-guidelines) is opened.\n- **Pull requests** may only be opened by members of the `modelcontextprotocol` GitHub organization.\n- **Everyone else** can report bugs and problems by\n  [opening an issue](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/new/choose)\n  or get involved by joining a working group.\n\nSee [CONTRIBUTING.md](./CONTRIBUTING.md) for details.\n\n## License\n\nThis project is licensed under the [MIT License](LICENSE).\n"}
````

`extract` 会把 Markdown 正文放进 `content` 字段，可直接引用。

> 示例 1 覆盖 `get_sub_domains`、`batch_search`、`extract`；示例 2 覆盖 `search`、`batch_search`、`extract`。
