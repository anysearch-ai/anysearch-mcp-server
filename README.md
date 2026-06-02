# AnySearch MCP Server <img alt="Version" src="https://img.shields.io/badge/version-v2.1.0-blue">

Unified real-time search MCP server supporting general web search, vertical domain search, parallel batch search, and full-page URL content extraction.

## Features

- **General Web Search** — open-ended natural language queries
- **Vertical Domain Search** — structured queries across domains (finance, academic, security, legal, code, etc.)
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
      "type": "remote",
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
| `query` | string | Yes | Natural language search query. ONE intent per call |
| `domain` | string | No | Vertical domain (e.g. `finance`, `academic`, `security`). Must come from `get_sub_domains` enum |
| `sub_domain` | string | No | Sub-domain routing key (e.g. `finance.us_stock`). Must come from `get_sub_domains` output |
| `sub_domain_params` | object | No | Structured params from `get_sub_domains` params column. NEVER invent values |
| `max_results` | integer | No | 1–10, default 10 |

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

Fetch full page content from a URL and return as Markdown. Truncated at 50,000 characters. HTML pages only.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `url` | string | Yes | Target URL (`http://` or `https://`) |