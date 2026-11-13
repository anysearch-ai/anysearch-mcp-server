# AnySearch MCP Server

> English | [简体中文](./README_zh.md)

Unified real-time search MCP server supporting general web search, vertical domain search, parallel batch search, and page body extraction (truncated when too long).

Official MCP setup docs: [https://www.anysearch.com/docs/mcp-install](https://www.anysearch.com/docs/mcp-install)

## Features

- **General Web Search** — open-ended natural language queries
- **Vertical Domain Search** — structured queries across finance, academic, security, legal, code, and more
- **Parallel Batch Search** — execute multiple independent queries in one call
- **URL Content Extraction** — fetch the page body and extract it as Markdown (truncated when too long)
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

Quota fields such as `rate_limit` are account-specific response values (the observed default is 20); do not treat the example as a fixed plan.

On success the agent should:

1. Tell the user their username (= email) and the `login_url`.
2. After user confirmation, write `data.api_key.key` into a private MCP config or the `ANYSEARCH_API_KEY` environment variable as the `Authorization: Bearer <key>` value. It is shown only once (it can also be retrieved later from the dashboard).
3. Do not echo, print, or commit the key, and do not write it into project-level shared configuration or chat logs.

Errors can be expressed through the HTTP status, application-level `code`, or `message`, and may return a non-zero `code` with HTTP 200; do not assume every error uses `code: -1`, and do not rely on HTTP status alone:

| status / code / message | what to do |
| ----------------------- | ---------- |
| `invalid_email` (HTTP `200`, `code: 40001`) | ask the user to re-enter the email |
| HTTP `409` (observed empty body) or `email_already_registered` | email is taken — tell the user to sign in at `login_url`; do **not** retry |
| `code: 42901`, `rate_limited`, or contains `Rate limited` | prefer a retry delay from `Retry-After`, `data.retry_after_seconds`, or the message; if none is provided, stop repeated requests and retry later or use anonymous access |
| starts with `Key creation failed.` | account created but key failed — extract the email and URL from the message (`"Key creation failed. Your account <email> was created; sign in at <url>."`) and tell the user to sign in there to create a key manually |
| `Internal server error.` | retry later or fall back to anonymous |

> Observed behavior: an invalid email returns HTTP 200 + `code: 40001` + `message: "invalid_email"`; rapid repeated registration returns HTTP 200 + `code: 42901` + `data.retry_after_seconds`; duplicate registration after the cooldown returns HTTP 409 with an empty body.

> The email **must be real and reachable**.

### Get an API Key

Alternatively, visit https://anysearch.com/console/api-keys to create a free API key.

### Key Priority

The remote MCP endpoint supports only these two modes:

| Priority | Source |
|----------|--------|
| 1 (highest) | `Authorization: Bearer <key>` request header (the key can be injected by the client from `ANYSEARCH_API_KEY`) |
| 2 | Anonymous access (rate-limited per IP and counted against the daily free quota) |

> The `--api_key` CLI flag and `.env` file reading apply only to local CLI/SDK clients, not to the hosted MCP endpoint.

### Key Behavior

| Scenario | Behavior |
|----------|----------|
| No key | Anonymous access; rate-limited per client IP and counted against the daily free quota |
| Has key | Sent via the `Authorization: Bearer <key>` header; uses the key's paid quota and higher concurrency |
| Key invalid, disabled, or expired | Returns HTTP `401` or `403`; does **not** silently fall back to anonymous access |
| Quota exhausted | Returns HTTP `402` |
| Rate limited | Returns HTTP `429`; wait according to `Retry-After` or the retry time in the response body |

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

## Verify the Installation

After saving the configuration, confirm these three points:

1. The client shows `anysearch` as connected.
2. The tool list includes `search`, `get_sub_domains`, `batch_search`, and `extract`.
3. Send the test prompt below and confirm that the client actually calls AnySearch:

```text
Use AnySearch to search for "Model Context Protocol official documentation". Return 3 results and keep the title and source URL for each result.
```

If no tool is called, check the MCP connection status and tool permissions first.

## Client Quick Reference

| Client | Official connection | Configuration location | Direct Streamable HTTP? |
|--------|---------------------|------------------------|-------------------------|
| OpenCode | Remote MCP config | `~/.config/opencode/opencode.json` or project `opencode.json` | Yes |
| Claude Code | Remote HTTP MCP | User `~/.claude.json` or project `.mcp.json` | Yes |
| Cursor | Remote MCP config | `.cursor/mcp.json` or `~/.cursor/mcp.json` | Yes |
| VS Code | HTTP MCP config | Local user config or `.vscode/mcp.json`; Agent Host configuration follows VS Code/Copilot official docs | Yes |
| Cline | Remote Streamable HTTP | Open the effective config through Cline IDE or CLI wizard | Yes |
| OpenAI Codex | Remote MCP config | `~/.codex/config.toml` or trusted project `.codex/config.toml` | Yes |
| Google Antigravity | Remote MCP config using `serverUrl` | `~/.gemini/config/mcp_config.json` or workspace `.agents/mcp_config.json` | Yes |
| DeepSeek Harness | Dedicated `@anysearch/anysearch-dsh` plugin | Profile under `$DSH_HOME`; credentials in `~/.dsh/.credentials.yaml` | Yes |
| Hermes Agent | Remote HTTP MCP | `$HERMES_HOME/config.yaml` | Yes |
| OMP (Oh My Pi) | HTTP MCP config | `.omp/mcp.json` or `~/.omp/agent/mcp.json` | Yes |

## Available Tools

The server currently provides 4 tools: `search`, `get_sub_domains`, `batch_search`, and `extract`.

### `search`

Execute a search query — general or vertical domain.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Natural language search query. ONE intent per call |
| `domain` | string | No | Vertical domain (e.g. `finance`, `academic`, `security`). Must come from `get_sub_domains` enum |
| `sub_domain` | string | No | Sub-domain routing key (e.g. `finance.us_stock`). Must come from `get_sub_domains` output |
| `sub_domain_params` | object | No | Structured params from `get_sub_domains` params column. NEVER invent values |
| `max_results` | integer | No | 1–10, default 10 |

General search without `domain` can be called directly without `get_sub_domains`; once you pass `domain`, you must call `get_sub_domains` first to obtain a valid `sub_domain` and parameter definitions.

### `get_sub_domains`

Query the vertical domain directory. **Required before any search that uses a domain** — returns valid sub_domains and their parameter schemas.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | One of | Single domain to query |
| `domains` | string[] | One of | Batch up to 5 domains (preferred — covers more ground) |

Returns a Markdown table: `sub_domain | description | params`

### `batch_search`

Execute 1–5 independent search queries in parallel. Single failure does not block others.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `queries` | object[] | Yes | 1–5 query objects, each with same fields as `search` |

### `extract`

Fetch the page body from a URL and return it as Markdown. Supports HTML/XHTML, plain text, JSON, and Markdown; does not support PDF, DOC/DOCX, images, audio/video, archives, or other binary formats. HTML/plain-text content longer than 50,000 characters is truncated; oversized JSON/Markdown returns an error.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `url` | string | Yes | Target URL (`http://` or `https://`) |

## Tool Call Examples

`search`:

```json
{
  "query": "Model Context Protocol official documentation",
  "max_results": 3
}
```

`get_sub_domains`:

```json
{
  "domains": ["finance", "academic"]
}
```

`batch_search`:

```json
{
  "queries": [
    { "query": "MCP official getting started guide", "max_results": 3 },
    { "query": "Cursor MCP configuration documentation", "max_results": 3 }
  ]
}
```

`extract`:

```json
{
  "url": "https://example.com"
}
```
