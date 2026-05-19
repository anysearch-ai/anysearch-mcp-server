# AnySearch MCP Server

Unified real-time search MCP server supporting general web search, vertical domain search, parallel batch search, and full-page URL content extraction.

## Features

- **General Web Search** — open-ended natural language queries
- **Vertical Domain Search** — structured queries across 23 domains (finance, academic, security, legal, code, etc.)
- **Parallel Batch Search** — up to 5 independent queries in one call
- **URL Content Extraction** — fetch and extract full page content as Markdown
- **Anonymous Access** — works without an API key (with lower rate limits)

## API Key Configuration

An API key is **optional but recommended**. Without a key, all features still work via anonymous access with lower rate limits.

### Get an API Key

Visit https://anysearch.com/console/api-keys to create a free API key.

### Key Priority

| Priority | Source |
|----------|--------|
| 1 (highest) | `--api_key` CLI flag / `Authorization` header |
| 2 | Environment variable `ANYSEARCH_API_KEY` |
| 3 | `.env` file (`ANYSEARCH_API_KEY=<key>`) |
| 4 | Anonymous access (lower rate limits) |

### Key Behavior

| Scenario | Behavior |
|----------|----------|
| No key | Proceed with anonymous access (lower rate limits) |
| Has key | Sent via `Authorization: Bearer <key>` header, higher rate limits |
| Key exhausted, auto-registered key returned | Agent should ask user for confirmation, then persist the new key |
| Key exhausted, no new key | Inform user and suggest configuring a new API key |

## MCP Transport

AnySearch MCP server **natively supports Streamable HTTP** transport (MCP spec 2025-03-26). SSE and stdio clients can connect via proxy.

| Transport | Native? | Best for |
|-----------|---------|----------|
| **Streamable HTTP** | Yes | OpenCode, Claude Desktop (2025.6+), web-based clients |
| **SSE** | Via proxy | Cursor, Windsurf |
| **stdio** | Via proxy | Claude Desktop (legacy), VS Code Copilot, Cline |

## Installation

### Streamable HTTP (Recommended — No Proxy Needed)

For agents that support the Streamable HTTP transport (MCP spec 2025-03-26+):

**OpenCode** (`~/.opencode/config.json` or project `opencode.json`):

```json
{
  "mcp": {
    "anysearch": {
      "type": "streamable-http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}"
      }
    }
  }
}
```

**Claude Desktop** (2025.6+, `claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "streamable-http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}"
      }
    }
  }
}
```

> Without an API key, omit the `headers` section. The server will use anonymous access automatically.

### stdio (Via Proxy)

For agents that only support stdio transport. Two proxy options:

#### Option A: mcp-remote (Recommended)

[`mcp-remote`](https://github.com/geelen/mcp-remote) — auto-detects Streamable HTTP, simplest config:

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "anysearch": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.anysearch.com/mcp",
        "--header",
        "Authorization: Bearer ${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

**VS Code Copilot** (`.vscode/mcp.json`):

```json
{
  "servers": {
    "anysearch": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.anysearch.com/mcp",
        "--header",
        "Authorization: Bearer ${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

**Cline** (VS Code settings):

```json
{
  "mcpServers": {
    "anysearch": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.anysearch.com/mcp",
        "--header",
        "Authorization: Bearer ${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

> Without an API key, omit the `"--header"` and `"Authorization: Bearer ..."` args.

#### Option B: supergateway

[`supergateway`](https://github.com/supercorp-ai/supergateway) — more transport options, supports SSE output:

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "anysearch": {
      "command": "npx",
      "args": [
        "-y",
        "supergateway",
        "--streamableHttp",
        "https://api.anysearch.com/mcp",
        "--oauth2Bearer",
        "${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

> Without an API key, omit the `"--oauth2Bearer"` and key args.

### SSE (Via Proxy)

For agents that only support SSE transport (Cursor, Windsurf). Requires running a local SSE proxy server:

#### Start the proxy

```bash
npx -y supergateway \
  --streamableHttp https://api.anysearch.com/mcp \
  --outputTransport sse \
  --port 8000 \
  --oauth2Bearer <your_api_key>
```

> Without an API key, omit the `--oauth2Bearer` flag.

Then configure your agent:

**Cursor** (`.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "sse",
      "url": "http://localhost:8000/sse"
    }
  }
}
```

**Windsurf** (`~/.codeium/windsurf/mcp_config.json`):

```json
{
  "mcpServers": {
    "anysearch": {
      "serverUrl": "http://localhost:8000/sse"
    }
  }
}
```

> The SSE proxy must remain running while the agent is active. Consider running it as a background service.

## Agent Quick Reference

| Agent | Transport | Config Location | Needs Proxy? | Proxy Tool |
|-------|-----------|----------------|-------------|------------|
| OpenCode | Streamable HTTP | `opencode.json` | No | — |
| Claude Desktop (2025.6+) | Streamable HTTP | `claude_desktop_config.json` | No | — |
| Claude Desktop (legacy) | stdio | `claude_desktop_config.json` | Yes | `mcp-remote` |
| Cursor | SSE | `.cursor/mcp.json` | Yes | `supergateway` |
| VS Code Copilot | stdio | `.vscode/mcp.json` | Yes | `mcp-remote` |
| Windsurf | SSE | `mcp_config.json` | Yes | `supergateway` |
| Cline | stdio | VS Code settings | Yes | `mcp-remote` |

## Available Tools

### `search`

Execute a search query — general or vertical domain.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Search query. For vertical search, follow the `query_format` from `list_domains` |
| `domain` | string | No | Vertical domain (e.g. `finance`, `academic`, `security`) |
| `sub_domain` | string | No | Sub-domain routing key (e.g. `finance.us_stock`). Required for vertical search |
| `sub_domain_params` | object | No | Extra params per sub_domain schema |
| `content_types` | string[] | No | Filter: `web`, `news`, `code`, `doc`, `academic`, `data`, `image`, `video`, `audio` |
| `zone` | string | No | `cn` or `intl`. Required when sub_domain marks `zone=CN` |
| `max_results` | integer | No | 1–100, default 10 |
| `freshness` | string | No | `day`, `week`, `month`, `year` |

### `list_domains`

Query the vertical domain directory. **Must be called before vertical search** to discover available sub_domains and their query formats.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `domain` | string | One of | Single domain to query |
| `domains` | string[] | One of | Batch up to 5 domains |

Returns a Markdown table: `sub_domain | description | query_format | params_schema | zone`

### `batch_search`

Execute 2–5 independent search queries in parallel. Single failure does not block others.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `queries` | object[] | Yes | 1–5 query objects, each with same fields as `search` |

### `extract`

Fetch full page content from a URL and return as Markdown. Truncated at 50,000 characters. HTML pages only.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `url` | string | Yes | Target URL (`http://` or `https://`) |

## Decision Flow

```
User query
  |
  +-- Has structured identifiers? (Stock:/CVE:/DOI:/IATA:/patent etc.)
  |     YES -> 1) list_domains -> discover sub_domain & query_format
  |             2) search with domain + sub_domain + zone
  |
  +-- Multiple independent intents?
  |     YES -> batch_search
  |
  +-- Need deeper content than snippets?
  |     YES -> extract the URL
  |
  +-- Otherwise -> search (general)
```

## Vertical Search Constraints

Before vertical search, you **must** call `list_domains` for the target domain and follow:

1. **`query_format`** — exact format for the query string (e.g. raw ticker, not natural language)
2. **`params_schema`** — JSON schema for optional extra parameters
3. **`zone`** — if `CN`, you must set `zone: "cn"` in the search call
4. **`sub_domain` selection** — match user intent to the best sub_domain description

## Supported Domains

`code` `tech` `fashion` `travel` `home` `ecommerce` `gaming` `film` `music` `finance` `academic` `legal` `business` `ip` `security` `education` `health` `religion` `geo` `environment` `energy` `ugc`

## Examples

### General search

```json
{ "query": "quantum computing breakthroughs 2025", "max_results": 5, "freshness": "month" }
```

### Vertical search (stock)

```json
{ "query": "AAPL", "domain": "finance", "sub_domain": "finance.us_stock", "max_results": 5 }
```

### Batch search

```json
{
  "queries": [
    { "query": "AAPL", "domain": "finance", "sub_domain": "finance.us_stock" },
    { "query": "python async http client", "domain": "code", "sub_domain": "code.general" }
  ]
}
```

### Extract URL

```json
{ "url": "https://en.wikipedia.org/wiki/Quantum_computing" }
```

## Security Notes

- Search queries, extracted URLs, and API keys are sent to `https://api.anysearch.com`
- Do not use for queries containing sensitive information (passwords, personal data, trade secrets) unless you trust the provider
- Avoid pasting API keys directly in chat — use environment variables or `.env` files
