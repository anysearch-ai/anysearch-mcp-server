# AnySearch MCP Server

> English | [简体中文](./README_zh.md)

Unified real-time search MCP server supporting general web search, vertical domain search, parallel batch search, and page body extraction.

Official MCP setup docs: [https://www.anysearch.com/docs/mcp-install](https://www.anysearch.com/docs/mcp-install)

To use AnySearch's skill service, see the [AnySearch skill repository](https://github.com/anysearch-ai/anysearch-skill).

To use the AnySearch API, see the [official AnySearch API documentation](https://www.anysearch.com/docs/api-endpoints).

## Features

- **General Web Search** — open-ended natural language queries
- **Vertical Domain Search** — structured queries across finance, academic, security, legal, code, and more
- **Parallel Batch Search** — execute multiple independent queries in one call
- **URL Content Extraction** — fetch the page body and extract it as Markdown
- **Anonymous Access** — works without an API key (rate-limited per client IP and counted against the daily free quota)

## API Key Configuration

An API key is **optional but recommended**. Without a key, all features still work via anonymous access, but anonymous access is rate-limited per client IP and counted against the daily free quota.

Current public quotas: Free 1,000 calls/day, up to 10 QPS; Pro up to 20 QPS; Student/Developer plans 2,000 calls/day, 20 QPS (https://www.anysearch.com/pricing). Actual quotas and concurrency limits depend on your account and the server response.

### Register for an API Key (Recommended)

The agent can register the user and obtain an API key in a **single call** — no verification code, no manual signup. Ask the user for a **real email address**: it becomes the account username. Obtain the user's explicit consent before registering; do not ask the user to paste the key into chat, and do not echo, print, or commit the key.

```bash
curl -s -X POST "https://api.anysearch.com/v1/auth/email/register" \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com"}'
```

Success response (`code: 0`) returns the account info and a one-time plaintext API key:

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

Quota fields such as `rate_limit` are account-specific response values; do not treat the example as a fixed plan.

On success the agent should:

1. Tell the user their username (= email) and the `login_url`.
2. After user confirmation, write `data.api_key.key` into a private MCP config or the `ANYSEARCH_API_KEY` environment variable as the `Authorization: Bearer <key>` value. It is shown only once (it can also be retrieved later from the dashboard).
3. Do not echo, print, or commit the key, and do not write it into project-level shared configuration or chat logs.

Errors can be expressed through the HTTP status, application-level `code`, or `message`, and may return a non-zero `code` with HTTP 200; do not assume every error uses `code: -1`, and do not rely on HTTP status alone:

| message pattern              | what to do                                                   |
| ---------------------------- | ------------------------------------------------------------ |
| `"invalid_email"`            | ask the user to re-enter the email                           |
| `"email_already_registered"` | email is taken — tell the user to sign in at `login_url`; do **not** retry |
| `"Rate limited"`             | prefer a retry delay from `Retry-After`, `data.retry_after_seconds`, or the message; if none is provided, stop repeated requests and retry later or use anonymous access |
| `"Key creation failed."`     | account created but key failed — extract the email and URL from the message (`"Key creation failed. Your account <email> was created; sign in at <url>."`) and tell the user to sign in there to create a key manually |
| `"Internal server error."`   | retry later or fall back to anonymous                        |

> The email **must be real and reachable**.

### Get an API Key

Alternatively, visit https://anysearch.com/console/api-keys to create a free API key.

### Key Priority

The remote MCP endpoint supports only these two modes:

| Priority    | Source                                                       |
| ----------- | ------------------------------------------------------------ |
| 1 (highest) | `Authorization: Bearer <key>` request header (the key can be injected by the client from `ANYSEARCH_API_KEY`) |
| 2           | Anonymous access (rate-limited per IP and counted against the daily free quota) |

> The `--api_key` CLI flag and `.env` file reading apply only to local CLI/SDK clients, not to the hosted MCP endpoint.

### Key Behavior

| Scenario                          | Behavior                                                     |
| --------------------------------- | ------------------------------------------------------------ |
| No key                            | Anonymous access; rate-limited per client IP and counted against the daily free quota |
| Has key                           | Sent via the `Authorization: Bearer <key>` header; uses the key's paid quota and higher concurrency |
| Key invalid, disabled, or expired | Returns HTTP `401` or `403`; does **not** silently fall back to anonymous access |
| Quota exhausted                   | Returns HTTP `402`                                           |
| Rate limited                      | Returns HTTP `429`; wait according to `Retry-After` or the retry time in the response body |

## MCP Transport

The production endpoint is:

```text
https://api.anysearch.com/mcp
```

It natively uses **Streamable HTTP**. Current OpenCode, Claude Code, Cursor, VS Code, Cline, OpenAI Codex, Google Antigravity, DeepSeek Harness, Hermes Agent, and OMP releases can connect to it directly; no SSE or stdio proxy is needed. Official MCP docs: [AnySearch MCP Server Setup](https://www.anysearch.com/docs/mcp-install).

The configurations below follow each client's current official documentation. Use a client version that supports remote HTTP; configuration locations and environment-variable syntax may vary by client version, so check the client's official documentation.

## Installation

The API key is optional. Where a client documents environment-variable or secret storage for custom headers, the examples use the recommended authenticated setup. Where it does not, the example stays anonymous and explains how to add authentication only in private user configuration. To use anonymous access elsewhere, remove only the `Authorization` entry and keep `X-Anysearch-Client`.

### OpenCode

Official docs: [MCP servers](https://opencode.ai/docs/mcp-servers/) and [configuration locations](https://opencode.ai/docs/config/).

Use `~/.config/opencode/opencode.json` for a global configuration or `opencode.json` in the project root. On Windows, the global path is `%USERPROFILE%\.config\opencode\opencode.json`.

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

OpenCode uses `{env:NAME}` for environment-variable substitution. `"oauth": false` prevents an unnecessary OAuth discovery flow for this API-key-authenticated server.

After saving, restart OpenCode, then run `opencode mcp list` and confirm that `anysearch` is shown as `connected`.

### Claude Code

Official docs: [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp).

For a private, user-wide installation, run:

```bash
claude mcp add --transport http anysearch https://api.anysearch.com/mcp --scope user --header "Authorization: Bearer <your_api_key>" --header "X-Anysearch-Client: mcp/1.0.0"
```

The `user` scope stores the server in `~/.claude.json` and makes it available across all projects on the machine. The `--header` above writes the key in plaintext to the configuration; if you do not want a plaintext key, use the project-level `.mcp.json` below and set `ANYSEARCH_API_KEY` before starting Claude Code. For anonymous access, omit the `Authorization` `--header` option.

> On Windows PowerShell, the environment variable syntax is `$env:ANYSEARCH_API_KEY`, and quoting and variable expansion differ from bash. The `${VAR}` form in the CLI command is not guaranteed by official documentation; use `.mcp.json` for variable expansion.

For a shareable project configuration, create `.mcp.json` in the project root:

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

Claude Code expands `${VAR}` and `${VAR:-default}` in `.mcp.json`, including inside `url` and `headers`. Set `ANYSEARCH_API_KEY` before starting Claude Code. Project-scoped servers require approval when first opened interactively.

Verify the connection with:

```bash
claude mcp get anysearch
claude mcp list
```

Inside Claude Code, `/mcp` shows the server status and available tools.

### Cursor

Official docs: [Model Context Protocol](https://cursor.com/docs/mcp).

Use `.cursor/mcp.json` for a project or `~/.cursor/mcp.json` globally:

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

Cursor automatically recognizes the remote HTTP endpoint and supports `${env:NAME}` interpolation in `url` and `headers`.

Check the connection under **Customize > MCPs** or **Output > MCP Logs**. If Cursor CLI is installed, you can also run:

```bash
agent mcp list
agent mcp list-tools anysearch
```

### VS Code

Official docs: [Add and manage MCP servers](https://code.visualstudio.com/docs/agent-customization/mcp-servers) and [MCP configuration reference](https://code.visualstudio.com/docs/agents/reference/mcp-configuration).

Run **MCP: Open User Configuration** for a user-wide server, or create `.vscode/mcp.json` in a workspace. This example uses a password input so the key is prompted for and stored securely instead of being committed:

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

For anonymous access, remove the `Authorization` entry and the entire `inputs` array.

Run **MCP: List Servers**, select `anysearch`, then choose **Start** or **Show Output** to verify the connection. Note that `${input:...}` works in local VS Code; to reuse the server across Agent Host/Copilot, use a non-interactive configuration such as a workspace `.mcp.json`, and check VS Code/Copilot official documentation for the exact location.

### Cline

Official docs: [MCP](https://docs.cline.bot/mcp/mcp-overview).

In the Cline panel, open **MCP Servers > Configure > Configure MCP Servers**, or add a hosted endpoint from the **Remote Servers** tab. Use the explicit `streamableHttp` type; omitting it falls back to legacy SSE behavior.

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

Cline IDE supports `${env:VAR}` in `url`, `headers`, and `env`. Run `cline mcp` to open the CLI wizard, or `cline config mcp --json` to list the configuration. Cline's configuration file location may vary by version; open the effective configuration through **Configure MCP Servers** in the IDE or the CLI wizard instead of relying on a hard-coded path.

### OpenAI Codex

Official docs: [Model Context Protocol](https://developers.openai.com/codex/mcp/).

Use `~/.codex/config.toml` globally or `.codex/config.toml` in a trusted project. The Codex CLI, IDE extension, and desktop app share this configuration:

```toml
[mcp_servers.anysearch]
url = "https://api.anysearch.com/mcp"
bearer_token_env_var = "ANYSEARCH_API_KEY"
http_headers = { "X-Anysearch-Client" = "mcp/1.0.0" }
```

Set `ANYSEARCH_API_KEY` before starting Codex. For anonymous access, remove `bearer_token_env_var`. Run `codex mcp list`, or use `/mcp` inside the Codex TUI, to confirm the server and its tools are available.

### Google Antigravity

Official docs: [MCP servers in Antigravity](https://antigravity.google/docs/ide/mcp).

Use `~/.gemini/config/mcp_config.json` globally or `.agents/mcp_config.json` in a workspace. Remote servers must use `serverUrl`; the legacy `url` and `httpUrl` keys are not supported:

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

Antigravity's official documentation currently shows literal custom-header values but does not document environment-variable interpolation for remote headers. If authentication is required, add `"Authorization": "Bearer YOUR_API_KEY"` only to the private global file and never commit it to `.agents/mcp_config.json`. Open **MCP Servers > Manage MCP Servers** in the IDE, or run `/mcp` in the CLI, to inspect live status, logs, and tools.

### DeepSeek Harness

Official docs: [DeepSeek Harness integration](https://anysearch.com/docs/open-source-integrations/deepseek).

The recommended setup uses the AnySearch plugin (anonymous quota works out of the box):

```bash
npx -y @deepseek-ai/dsh plugin --profile web add @anysearch/anysearch-dsh
```

Before installing, confirm that Node.js 22.19 or 24+ and pnpm 11.7 are available directly from `PATH`. After installation, verify the profile:

```bash
npx -y @deepseek-ai/dsh --profile web --dump-config
npx -y @deepseek-ai/dsh web
```

The `--dump-config` output should include `searchProvider: anysearch`, `fetchProvider: anysearch`, and the `web-search-anysearch` plugin entry. The plugin takes over the native `web_search`/`web_fetch` tools and adds:

- `anysearch_capabilities`
- `anysearch_search`
- `anysearch_batch_search`

The API key is optional; for account quota, store `ANYSEARCH_API_KEY` in `~/.dsh/.credentials.yaml`. Do not put the key in chat, commands, or the repository. The older generic cordis MCP configuration (`@deepseek-ai/dsh-mcp-client`) still works, but prefer the dedicated plugin above. DeepSeek Harness is still a developer preview; recheck the official plugin documentation when upgrading.

### Hermes Agent

Official docs: [MCP integration](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/mcp.md).

Add the server to `$HERMES_HOME/config.yaml` and store `ANYSEARCH_API_KEY` in `$HERMES_HOME/.env` or the process environment. On macOS and Linux, `$HERMES_HOME` defaults to `~/.hermes`; the native Windows installer currently defaults to `%LOCALAPPDATA%\hermes`:

```yaml
mcp_servers:
  anysearch:
    url: "https://api.anysearch.com/mcp"
    headers:
      Authorization: "Bearer ${ANYSEARCH_API_KEY}"
      X-Anysearch-Client: "mcp/1.0.0"
```

Verify it with `hermes mcp test anysearch` and `hermes mcp list`. After editing the configuration during a session, run `/reload-mcp` or start a new session.

### OMP (Oh My Pi)

Official docs: [MCP configuration](https://github.com/can1357/oh-my-pi/blob/main/docs/mcp-config.md).

Use `.omp/mcp.json` in a project or `~/.omp/agent/mcp.json` globally:

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

OMP expands `${VAR}` and `${VAR:-default}` in configuration values. Set `ANYSEARCH_API_KEY`, then run `/mcp reload`, `/mcp test anysearch`, and `/mcp list` inside OMP.

### `search`

Execute a search query — general or vertical domain.

| Parameter           | Type    | Required | Description                                                  |
| ------------------- | ------- | -------- | ------------------------------------------------------------ |
| `query`             | string  | Yes      | Natural language search query. ONE intent per call           |
| `domain`            | string  | No       | Vertical domain (e.g. `finance`, `academic`, `security`). Must come from `get_sub_domains` enum |
| `sub_domain`        | string  | No       | Sub-domain routing key (e.g. `finance.us_stock`). Must come from `get_sub_domains` output |
| `sub_domain_params` | object  | No       | Structured params from `get_sub_domains` params column. NEVER invent values |
| `max_results`       | integer | No       | 1–10, default 10                                             |

General search without `domain` can be called directly without `get_sub_domains`; once you pass `domain`, you must call `get_sub_domains` first to obtain a valid `sub_domain` and parameter definitions.

### `get_sub_domains`

Query the vertical domain directory. **Required before any search that uses a domain** — returns valid sub_domains and their parameter schemas.

| Parameter | Type     | Required | Description                                            |
| --------- | -------- | -------- | ------------------------------------------------------ |
| `domain`  | string   | One of   | Single domain to query                                 |
| `domains` | string[] | One of   | Batch up to 5 domains (preferred — covers more ground) |

Returns a Markdown table: `sub_domain | description | params`

### `batch_search`

Execute 1–5 independent search queries in parallel. Single failure does not block others.

| Parameter | Type     | Required | Description                                          |
| --------- | -------- | -------- | ---------------------------------------------------- |
| `queries` | object[] | Yes      | 1–5 query objects, each with same fields as `search` |

### `extract`

Fetch the page body from a URL and return it as Markdown. Supports HTML/XHTML, plain text, JSON, and Markdown; does not support PDF, DOC/DOCX, images, audio/video, archives, or other binary formats. HTML/plain-text content longer than 50,000 characters is truncated; oversized JSON/Markdown returns an error.

| Parameter | Type   | Required | Description                          |
| --------- | ------ | -------- | ------------------------------------ |
| `url`     | string | Yes      | Target URL (`http://` or `https://`) |

## End-to-End Example (all clients)

The two examples below cover **vertical search** and **general search**. Both use `batch_search` and `extract`; in real use, pick one based on the query type.

### Example 1: Vertical search (compare two currency pairs)

```text
Compare the current USD/CNY and EUR/USD exchange rates and cite sources for the quotes.
```

This query has a clear vertical domain (forex quotes), so it takes 3 steps: `get_sub_domains` → `batch_search` → `extract`.

**1. `get_sub_domains` — fetch the valid finance sub-domains and parameters**

```json
{ "domains": ["finance"] }
```

The call returns 6 sub-domains with their parameter schemas (the full docs are long, so only names and purposes are listed here):

| sub_domain            | Purpose                                                      |
| --------------------- | ------------------------------------------------------------ |
| `finance.calendar`    | Earnings, dividends, IPO, and economic data calendars        |
| `finance.fundamental` | Statements, valuation, analyst ratings, shareholders, SEC filings |
| `finance.macro`       | Macro indicators such as GDP, CPI, rates, money supply       |
| `finance.news`        | Financial news, company announcements, broker summaries      |
| `finance.quote`       | Quotes for stocks, forex, crypto, commodities, indices, ETFs |
| `finance.screen`      | Screen stocks by sector and country                          |

Actual `sub_domain` and `params` keys are whatever this call returns.

**2. `batch_search` (vertical) — fetch both quotes in parallel with `finance.quote`**

```json
{
  "queries": [
    {
      "query": "USD CNY spot rate",
      "domain": "finance",
      "sub_domain": "finance.quote",
      "sub_domain_params": { "type": "forex", "symbol": "USDCNY" },
      "max_results": 10
    },
    {
      "query": "EUR USD spot rate",
      "domain": "finance",
      "sub_domain": "finance.quote",
      "sub_domain_params": { "type": "forex", "symbol": "EURUSD" },
      "max_results": 10
    }
  ]
}
```

````text
## Query 1: USD CNY spot rate

## Search Results (1 results, 453ms)

### 1. USD/CNY Exchange Rate
- **URL**: https://site.financialmodelingprep.com/?ref=quote-USDCNY
- Pair: USD/CNY | Rate: 6.6772 | Change: +0.0107 | Change%: +0.16% | Open: 6.6660 | PrevClose: 6.6665 | DayHigh: 6.6794 | DayLow: 6.6623 | YearHigh: 7.1155 | YearLow: 6.6369 | Avg50: 6.6800 | Avg200: 6.7500

```yaml
request_id: 6d36ab15bf624488b336a4f1fc951571
```

---

## Query 2: EUR USD spot rate

## Search Results (1 results, 456ms)

### 1. EUR/USD Exchange Rate
- **URL**: https://site.financialmodelingprep.com/?ref=quote-EURUSD
- Pair: EUR/USD | Rate: 1.0969 | Change: +0.0002 | Change%: +0.02% | Open: 1.0969 | PrevClose: 1.0967 | DayHigh: 1.1000 | DayLow: 1.0957 | YearHigh: 1.2083 | YearLow: 1.0953 | Avg50: 1.1200 | Avg200: 1.1500

```yaml
request_id: 6866d24cdfa84d56bd0605c2c3456cd8
```
````

The two pairs are independent, so one failure does not block the other.

**3. `extract` — fetch the body of a citable rate source**

```json
{ "url": "https://api.frankfurter.app/latest?from=USD&to=CNY" }
```

````text
{"url":"https://api.frankfurter.app/latest?from=USD\u0026to=CNY","title":"","content":"{\"amount\":1.0,\"base\":\"USD\",\"date\":\"2026-11-20\",\"rates\":{\"CNY\":6.6777}}"}
````

`content` is the raw JSON string, usable directly as the cited exchange-rate source.

### Example 2: General search (MCP official docs)

```text
Find the official Model Context Protocol documentation and specification entry points, and give me citable body text.
```

This query has no vertical-domain intent, so it uses general `search`; use `batch_search` to compare several sources in parallel, then `extract` for the body text.

**1. `search` (general)**

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

**2. `batch_search` (general) — compare the docs and specification sources in parallel**

```json
{
  "queries": [
    { "query": "Model Context Protocol 官方文档", "max_results": 1 },
    { "query": "Model Context Protocol specification 2026-07-28", "max_results": 1 }
  ]
}
```

> `max_results` is set to `1` here only to keep the example compact; it does not change the flow.

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

**3. `extract` — fetch the body of a citable specification repository**

```json
{ "url": "https://raw.githubusercontent.com/modelcontextprotocol/modelcontextprotocol/main/README.md" }
```

````text
{"url":"https://raw.githubusercontent.com/modelcontextprotocol/modelcontextprotocol/main/README.md","title":"","content":"# Model Context Protocol (MCP)\n\n_Just heard of MCP and not sure where to start? Check out our [documentation website](https://modelcontextprotocol.io)._\n\nThis repo contains the:\n\n- MCP specification\n- MCP protocol schema\n- Official MCP documentation\n\nThe schema is [defined in TypeScript](schema/2026-07-28/schema.ts) first, but\n[made available as JSON Schema](schema/2026-07-28/schema.json) as well, for wider\ncompatibility.\n\nThe official MCP documentation is built using Mintlify and available at\n[modelcontextprotocol.io](https://modelcontextprotocol.io).\n\n## Authors\n\nThe Model Context Protocol was created by David Soria Parra ([@dsp](https://github.com/dsp)) and Justin Spahr-Summers ([@jspahrsummers](https://github.com/jspahrsummers)).\n\n## Contributing\n\nContributions are made through [working groups](https://modelcontextprotocol.io/community/working-interest-groups)\nand issues:\n\n- **Changes to the specification** are discussed and approved in the relevant working group\n  before a pull request or [SEP](https://modelcontextprotocol.io/community/sep-guidelines) is opened.\n- **Pull requests** may only be opened by members of the `modelcontextprotocol` GitHub organization.\n- **Everyone else** can report bugs and problems by\n  [opening an issue](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/new/choose)\n  or get involved by joining a working group.\n\nSee [CONTRIBUTING.md](./CONTRIBUTING.md) for details.\n\n## License\n\nThis project is licensed under the [MIT License](LICENSE).\n"}
````

`extract` returns the Markdown body in the `content` field, ready to cite.

> Example 1 covers `get_sub_domains`, `batch_search`, and `extract`; example 2 covers `search`, `batch_search`, and `extract`.
