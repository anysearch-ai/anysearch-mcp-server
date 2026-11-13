# AnySearch MCP Server

> [English](./README.md) | 简体中文

统一实时搜索 MCP 服务器，支持通用网页搜索、垂直领域搜索、并行批量搜索和页面正文提取（超长截断）。

官方 MCP 安装文档：[https://www.anysearch.com/docs/mcp-install](https://www.anysearch.com/docs/mcp-install)

## 功能

- **通用网页搜索** —— 开放式自然语言查询
- **垂直领域搜索** —— 支持金融、学术、安全、法律、代码等领域的结构化查询
- **并行批量搜索** —— 在一次调用中执行多个独立查询
- **URL 内容提取** —— 获取页面正文并提取为 Markdown（超长内容截断）
- **匿名访问** —— 无需 API key 即可使用（按 IP 限流，计入每日免费额度）

## API Key 配置

API key **是可选项，但建议配置**。即使没有 key，所有功能仍可通过匿名访问使用，但匿名访问按客户端 IP 限流，并计入每日免费额度。

当前公开额度为：Free 1,000 次/天、最高 10 QPS；Pro 最高 20 QPS；学生/开发者计划 2,000 次/天、20 QPS (https://www.anysearch.com/pricing)。实际额度和并发限制以账户及服务端响应为准。

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

`rate_limit` 等额度字段以实际响应为准（实测默认值为 20），不要把示例值视为固定套餐。

成功后，智能体应当：

1. 告知用户其用户名（= 邮箱）和 `login_url`。
2. 在获得用户确认后，将 `data.api_key.key` 写入私有 MCP 配置或环境变量 `ANYSEARCH_API_KEY`，作为 `Authorization: Bearer <key>` 的值。它只显示一次（之后也可从控制台重新获取）。
3. 不要回显、打印、提交 key，也不要将 key 写入项目级共享配置或聊天记录。

错误响应可能使用 HTTP 状态码、应用级 `code` 或 `message`，也可能在 HTTP 200 下返回非零 `code`；不要假设所有错误的 `code` 都是 `-1`，也不要只看 HTTP 状态：

| 状态 / code / message | 处理方式 |
| --------------------- | -------- |
| `invalid_email`（HTTP `200`，`code: 40001`） | 请用户重新输入邮箱 |
| HTTP `409`（实测 body 为空）或 `email_already_registered` | 邮箱已被注册 —— 告知用户在 `login_url` 登录；**不要**重试 |
| `code: 42901`、`rate_limited` 或包含 `Rate limited` | 优先读取 `Retry-After`、`data.retry_after_seconds` 或 message 中的重试时间；未提供时间时停止连续请求，稍后再试或先使用匿名访问 |
| 以 `Key creation failed.` 开头 | 账户已创建，但 key 创建失败 —— 从 message 中提取邮箱和 URL（`"Key creation failed. Your account <email> was created; sign in at <url>."`），告知用户在该地址登录并手动创建 key |
| `Internal server error.` | 稍后重试，或回退到匿名访问 |

> 实测行为：非法邮箱返回 HTTP 200 + `code: 40001` + `message: "invalid_email"`；连续注册触发限流返回 HTTP 200 + `code: 42901` + `data.retry_after_seconds`；冷却后重复注册返回 HTTP 409 且 body 为空。

> 该邮箱**必须真实且可接收邮件**。

### 获取 API Key

也可访问 https://anysearch.com/console/api-keys 创建免费 API key。

### Key 优先级

远程 MCP 端点只支持以下两种方式：

| 优先级 | 来源 |
| -------- | ---- |
| 1（最高） | `Authorization: Bearer <key>` 请求头（key 可通过环境变量 `ANYSEARCH_API_KEY` 由客户端注入） |
| 2 | 匿名访问（按 IP 限流，计入每日免费额度） |

> `--api_key` CLI 标志和 `.env` 文件读取仅适用于本地 CLI/SDK，不适用于托管 MCP 端点。

### Key 行为

| 场景 | 行为 |
| ---- | ---- |
| 无 key | 匿名访问；按客户端 IP 限流，计入每日免费额度 |
| 有 key | 通过 `Authorization: Bearer <key>` 请求头发送，使用 key 的付费额度并享有更高并发 |
| key 无效、禁用或过期 | 返回 HTTP `401` 或 `403`，**不会**静默回退到匿名访问 |
| 额度耗尽 | 返回 HTTP `402` |
| 触发限流 | 返回 HTTP `429`，按 `Retry-After` 或响应体中的重试时间等待 |

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

## 验证安装

保存配置后，确认以下三点：

1. 客户端中 `anysearch` 显示为已连接。
2. 工具列表中包含 `search`、`get_sub_domains`、`batch_search` 和 `extract`。
3. 发送下面的测试提示，确认客户端实际调用了 AnySearch：

```text
Use AnySearch to search for "Model Context Protocol official documentation". Return 3 results and keep the title and source URL for each result.
```

如果没有调用工具，先检查 MCP 连接状态和工具权限。

## 客户端速查表

| 客户端 | 官方接入方式 | 配置位置 | 可直连 Streamable HTTP？ |
| ------ | ------------ | -------- | ------------------------ |
| OpenCode | 远程 MCP 配置 | `~/.config/opencode/opencode.json` 或项目 `opencode.json` | 是 |
| Claude Code | 远程 HTTP MCP | 用户级 `~/.claude.json` 或项目级 `.mcp.json` | 是 |
| Cursor | 远程 MCP 配置 | `.cursor/mcp.json` 或 `~/.cursor/mcp.json` | 是 |
| VS Code | HTTP MCP 配置 | 本机用户配置或 `.vscode/mcp.json`；Agent Host 配置以 VS Code/Copilot 官方文档为准 | 是 |
| Cline | 远程 Streamable HTTP | 通过 Cline IDE 或 CLI 向导打开实际配置 | 是 |
| OpenAI Codex | 远程 MCP 配置 | `~/.codex/config.toml` 或受信任项目 `.codex/config.toml` | 是 |
| Google Antigravity | 使用 `serverUrl` 的远程 MCP 配置 | `~/.gemini/config/mcp_config.json` 或工作区 `.agents/mcp_config.json` | 是 |
| DeepSeek Harness | 专用 `@anysearch/anysearch-dsh` 插件 | `$DSH_HOME` 下对应 profile；凭据在 `~/.dsh/.credentials.yaml` | 是 |
| Hermes Agent | 远程 HTTP MCP | `$HERMES_HOME/config.yaml` | 是 |
| OMP (Oh My Pi) | HTTP MCP 配置 | `.omp/mcp.json` 或 `~/.omp/agent/mcp.json` | 是 |

## 可用工具

服务端当前提供 4 个工具：`search`、`get_sub_domains`、`batch_search` 和 `extract`。

### `search`

执行通用或垂直领域搜索查询。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `query` | string | 是 | 自然语言搜索查询。每次调用只包含一个意图 |
| `domain` | string | 否 | 垂直领域（例如 `finance`、`academic`、`security`）。必须来自 `get_sub_domains` 枚举 |
| `sub_domain` | string | 否 | 子领域路由 key（例如 `finance.us_stock`）。必须来自 `get_sub_domains` 输出 |
| `sub_domain_params` | object | 否 | 来自 `get_sub_domains` params 列的结构化参数。**绝不要**臆造参数值 |
| `max_results` | integer | 否 | 1–10，默认为 10 |

不传 `domain` 的通用搜索可以直接调用，无需先调用 `get_sub_domains`；一旦要传 `domain`，必须先调用 `get_sub_domains` 获取合法的 `sub_domain` 和参数定义。

### `get_sub_domains`

查询垂直领域目录。**使用 domain 进行任何搜索前都必须先调用** —— 返回有效的 sub_domains 及其参数 schema。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `domain` | string | 二选一 | 要查询的单个领域 |
| `domains` | string[] | 二选一 | 批量查询最多 5 个领域（推荐 —— 覆盖范围更广） |

返回 Markdown 表格：`sub_domain | description | params`

### `batch_search`

并行执行 1–5 个独立搜索查询。单个查询失败不会阻塞其他查询。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `queries` | object[] | 是 | 1–5 个查询对象，每个对象的字段与 `search` 相同 |

### `extract`

从 URL 获取页面正文并以 Markdown 返回。支持 HTML/XHTML、纯文本、JSON 和 Markdown；不支持 PDF、DOC/DOCX、图片、音视频、压缩包等二进制格式。HTML/纯文本内容超过 50,000 个字符时会被截断；超大 JSON/Markdown 会返回错误。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `url` | string | 是 | 目标 URL（`http://` 或 `https://`） |

## 调用示例

`search`：

```json
{
  "query": "Model Context Protocol official documentation",
  "max_results": 3
}
```

`get_sub_domains`：

```json
{
  "domains": ["finance", "academic"]
}
```

`batch_search`：

```json
{
  "queries": [
    { "query": "MCP official getting started guide", "max_results": 3 },
    { "query": "Cursor MCP configuration documentation", "max_results": 3 }
  ]
}
```

`extract`：

```json
{
  "url": "https://example.com"
}
```
