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

### Example 1: Vertical search (compare React and Vue state-management docs)

```text
Compare how the official React and Vue documentation describes component state management, and give me citable documentation sources.
```

This query has a clear vertical domain (code documentation), so it takes 3 steps: `get_sub_domains` → `batch_search` → `extract`.

**1. `get_sub_domains` — fetch the valid code sub-domains and parameters**

```json
{ "domains": ["code"] }
```

The call returns 2 sub-domains with their parameter schemas:

| sub_domain     | Purpose                                                                 | Parameters |
| -------------- | ----------------------------------------------------------------------- | ---------- |
| `code.doc`     | Query developer documentation and code examples by library name and topic across npm, PyPI, Cargo ecosystems | `library` (required) |
| `code.snippet` | Search real code implementations across 1M+ public GitHub repositories with regex and language filtering | `lang`, `path`, `repo` (optional) |

Actual `sub_domain` and `params` keys are whatever this call returns.

**2. `batch_search` (vertical) — fetch both docs in parallel with `code.doc`**

```json
{
  "queries": [
    {
      "query": "React official documentation component state management useState",
      "domain": "code",
      "sub_domain": "code.doc",
      "sub_domain_params": { "library": "react" },
      "max_results": 2
    },
    {
      "query": "Vue official documentation component state management ref reactive",
      "domain": "code",
      "sub_domain": "code.doc",
      "sub_domain_params": { "library": "vue" },
      "max_results": 2
    }
  ]
}
```

````text
## Query 1: React official documentation component state management useState

## Search Results 

### 1. Usage > Adding state to a component
- **URL**: https://github.com/reactjs/react.dev/blob/main/src/content/reference/react/useState.md
- Call `useState` at the top level of your component to declare one or more [state variables.](/learn/state-a-components-memory) The convention is to name state variables like `[something, setSomething]` using [array destructuring.](https://javascript.info/destructuring-assignment) `useState` returns an array with exactly two items: 1. The current state of this state variable, initially set to the initial state you provided. 2. The `set` function that lets you change it to any other value in response to interaction. To update what’s on the screen, call the `set` function with some next state: React will store the next state, render your component again with the new values, and update the UI. Calling the `set` function [**does not** change the current state in the already executing code](#ive-updated-the-state-but-logging-gives-me-the-old-value): It only affects what `useState` will return starting from the *next* render.

### 2. Implement a counter using useState
- **URL**: https://github.com/reactjs/react.dev/blob/main/src/content/learn/referencing-values-with-refs.md
- Use useState when a value must be displayed on screen and changes need to trigger a component re-render. Updating count with setCount causes React to re-render the component with the new value. ```javascript import { useState } from 'react'; export default function Counter() { const [count, setCount] = useState(0); function handleClick() { setCount(count + 1); } return ( You clicked {count} times ); } ```

---

## Query 2: Vue official documentation component state management ref reactive

## Search Results 

### 1. State Management | Vue.js
- **URL**: https://vuejs.org/guide/scaling-up/state-management
- # State Management [](#state-management) ## What is State Management? [](#what-is-state-management) Technically, every Vue component instance already "manages" its own reactive state. Take a simple counter component as an example: ``` import { ref } from 'vue' // state const count = ref(0) // actions function increment() { count.value++ } ## Simple State Management with Reactivity API [](#simple-state-management-with-reactivity-api) If you have a piece of state that should be shared by multiple instances, you can use `reactive()` to create a reactive object, and then import it into multiple components: ``` import { reactive } from 'vue' export const store = reactive({ count: 0 }) ``` ``` import { store } from './store.js' From A: {{ store.count }} ``` ``` import { store } from './store.js' From B: {{ store.count }} ``` Now whenever the `store` object is mutated, both `` and `` will update their views automatically - we have a single source of truth now. However, this also means any component importing `store` can mutate it however they want: ``` From B: {{ store.count }} ``` While this works in simple cases, global state that can be arbitrarily mutated by any component is not going to be very maintainable in the long run. To ensure the state-mutating logic is centralized like the state itself, it is recommended to define methods on the store with names that express the intention of the actions: ``` import { reactive } from 'vue' export const store = reactive({ count: 0, increment() { this.count++ } }) ``` ``` From B: {{ store.count }} ``` TIP Note the click handler uses `store.increment()` with parentheses - this is necessary to call the method with the proper `this` context since it's not a component method. Although here we are using a single reactive object as a store, you can also share reactive state created using other [Reactivity APIs](https://vuejs.org/api/reactivity-core) such as `ref()` or `computed()`, or even return global state from a [Composable](https://vuejs.org/guide/reusability/composables): ``` import { ref } from 'vue' // global state, created in module scope const globalCount = ref(1) export function useCount() { // local state, created per-component const localCount = ref(1) return { globalCount, localCount } } ``` The fact that Vue's reactivity system is decoupled from the component model makes it extremely flexible.
````

The two queries are independent, so one failure does not block the other.

**3. `extract` — fetch the body of a citable documentation source**

```json
{ "url": "https://vuejs.org/guide/scaling-up/state-management.md" }
```

````text
{"url":"https://vuejs.org/guide/scaling-up/state-management.md","title":"","content":"---\nurl: /guide/scaling-up/state-management.md\n---\n# State Management {#state-management}\n\n## What is State Management? {#what-is-state-management}\n\nTechnically, every Vue component instance already \"manages\" its own reactive state. ... (truncated here for readability)"}
````

`content` holds the page's Markdown body (truncated here for readability) and can be cited directly as a documentation source.

### Example 2: General search (MCP official docs)

```text
Find the official Model Context Protocol documentation and specification entry points, and give me citable body text.
```

This query has no vertical-domain intent, so it uses general `search`; use `batch_search` to compare several sources in parallel, then `extract` for the body text.

**1. `search` (general)**

```json
{ "query": "Find the official Model Context Protocol documentation and specification entry points, and give me citable body text.", "max_results": 10 }
```

````text
## Search Results (10 results, 1752ms)

### 1. Specification - Model Context Protocol
- **URL**: https://modelcontextprotocol.io/specification/draft
- [Model Context Protocol](https://modelcontextprotocol.io)(MCP) is an open protocol that enables seamless integration between LLM applications and external data sources and tools. Whether you’re building an AI-powered IDE, enhancing a chat interface, or creating custom AI workflows, MCP provides a standardized way to connect LLMs with the context they need. This specification defines the authoritative protocol requirements, based on the TypeScript schema in [schema.ts](https://github.com/modelcontextprotocol/specification/blob/main/schema/2024-11-05/schema.ts). For implementation guides and examples, visit [modelcontextprotocol.io](https://modelcontextprotocol.io). The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [BCP 14](https://datatracker.ietf.org/doc/html/bcp14)[ [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119)] [ [RFC8174](https://datatracker.ietf.org/doc/html/rfc8174)] when, and only when, they appear in all capitals, as shown here. ## Overview - Share contextual information with language models - Expose tools and capabilities to AI systems - Build composable integrations and workflows [JSON-RPC](https://www.jsonrpc.org/)2.0 messages to establish communication between: - **Hosts**: LLM applications that initiate connections - **Clients**: Connectors within the host application - **Servers**: Services that provide context and capabilities [Language Server Protocol](https://microsoft.github.io/language-server-protocol/), which standardizes how to add support for programming languages across a whole ecosystem of development tools. In a similar way, MCP standardizes how to integrate additional context and tools into the ecosystem of AI applications. ### Features - **Resources**: Context and data, for the user or the AI model to use - **Prompts**: Templated messages and workflows for users - **Tools**: Functions for the AI model to execute - **Sampling**: Server-initiated agentic behaviors and recursive LLM interactions

### 2. Specification (Draft) – Model Context Protocol （MCP）
- **URL**: https://modelcontextprotocol.info/specification/draft/
- # Specification [Model Context Protocol](https://modelcontextprotocol.io) (MCP) is an open protocol that enables seamless integration between LLM applications and external data sources and tools. Whether you’re building an AI-powered IDE, enhancing a chat interface, or creating custom AI workflows, MCP provides a standardized way to connect LLMs with the context they need. This specification defines the authoritative protocol requirements, based on the TypeScript schema in [schema.ts](https://github.com/modelcontextprotocol/specification/blob/main/schema/2025-03-26/schema.ts). For implementation guides and examples, visit [modelcontextprotocol.io](https://modelcontextprotocol.io). The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [BCP 14](https://datatracker.ietf.org/doc/html/bcp14) \[[RFC2119](https://datatracker.ietf.org/doc/html/rfc2119)] \[[RFC8174](https://datatracker.ietf.org/doc/html/rfc8174)] when, and only when, they appear in all capitals, as shown here. ## Overview MCP provides a standardized way for applications to: - Share contextual information with language models - Expose tools and capabilities to AI systems - Build composable integrations and workflows The protocol uses [JSON-RPC](https://www.jsonrpc.org/) 2.0 messages to establish communication between: - **Hosts**: LLM applications that initiate connections - **Clients**: Connectors within the host application - **Servers**: Services that provide context and capabilities MCP takes some inspiration from the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/), which standardizes how to add support for programming languages across a whole ecosystem of development tools. In a similar way, MCP standardizes how to integrate additional context and tools into the ecosystem of AI applications. ## Security and Trust & Safety The Model Context Protocol enables powerful capabilities through arbitrary data access and code execution paths. With this power comes important security and trust considerations that all implementors must carefully address.

### 3. Model Context Protocol Documentation: Spec, Schema, Changelog - DEV Community
- **URL**: https://dev.to:443/smartgate/model-context-protocol-documentation-spec-schema-changelog-2enb
- - **Do this next:**open your client's negotiated revision, find the matching changelog page, and write the revision date into your integration notes next to the client's version — that pair is what you will need the next time behaviour changes after an upgrade. Documentation is the part of a protocol ecosystem that ages fastest, because the ecosystem moves while the text stays dated. The Model Context Protocol's documentation is unusually explicit about this: the site keeps every revision online, each one carries its own schema, and a changelog records the distance between them. What follows is how to read that set — which page answers which question, and which copy is authoritative when two of them disagree. ## Where the protocol documentation actually lives The documentation set is four surfaces rather than one. The specification is the normative text, published per revision under `modelcontextprotocol.io/specification//`, with chapters for the lifecycle, the transports, authorization and each server and client feature. The schema is the TypeScript source of the wire types, published beside each revision and rendered on the site as a schema reference. The changelog is a per-revision page that lists what changed since the previous revision, with the proposal identifiers behind each change. And the registry is a separate metadata service for discovering servers, which answers a question the specification deliberately leaves out. When an argument starts, the order to settle it is short: specification for requirements, schema for shapes, changelog for history, registry only for discovery. If the actors themselves are still unclear — which side is a client, what a capability negotiation is for — [the protocol overview page](https://smartgate.network/industry/model-context-protocol-explained) is the one page worth reading before any of the four, because every citation below assumes it. ## Sources - Model Context Protocol — specification for the current revision (2026-07-28) and the previous one (2025-11-25): [https://modelcontextprotocol.io/specification/2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28)·[https://modelcontextprotocol.io/specification/2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) - Model Context Protocol — versioning and compatibility, and the per-revision changelog: [https://modelcontextprotocol.io/specification/draft/basic/versioning](https://modelcontextprotocol.io/specification/draft/basic/versioning)·[https://modelcontextprotocol.io/specification/2026-07-28/changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog) - Model Context Protocol — schema reference, published beside each revision: [https://modelcontextprotocol.io/specification/2025-11-25/schema](https://modelcontextprotocol.io/specification/2025-11-25/schema) - Model Context Protocol — the official registry, a metadata service for discovery: [https://modelcontextprotocol.io/registry](https://modelcontextprotocol.io/registry) - Databricks — AI gateway documentation: [https://docs.databricks.com/aws/en/ai-gateway/](https://docs.databricks.com/aws/en/ai-gateway/) - Cloudflare — AI Gateway: [https://developers.cloudflare.com/ai-gateway/](https://developers.cloudflare.com/ai-gateway/)

### 4. Model Context Protocol Documentation: Spec, Schema, Changelog - DEV Community
- **URL**: https://mailto:support@dev.to/smartgate/model-context-protocol-documentation-spec-schema-changelog-2enb
- - **Do this next:**open your client's negotiated revision, find the matching changelog page, and write the revision date into your integration notes next to the client's version — that pair is what you will need the next time behaviour changes after an upgrade. Documentation is the part of a protocol ecosystem that ages fastest, because the ecosystem moves while the text stays dated. The Model Context Protocol's documentation is unusually explicit about this: the site keeps every revision online, each one carries its own schema, and a changelog records the distance between them. What follows is how to read that set — which page answers which question, and which copy is authoritative when two of them disagree. ## Where the protocol documentation actually lives The documentation set is four surfaces rather than one. The specification is the normative text, published per revision under `modelcontextprotocol.io/specification//`, with chapters for the lifecycle, the transports, authorization and each server and client feature. The schema is the TypeScript source of the wire types, published beside each revision and rendered on the site as a schema reference. The changelog is a per-revision page that lists what changed since the previous revision, with the proposal identifiers behind each change. And the registry is a separate metadata service for discovering servers, which answers a question the specification deliberately leaves out. When an argument starts, the order to settle it is short: specification for requirements, schema for shapes, changelog for history, registry only for discovery. If the actors themselves are still unclear — which side is a client, what a capability negotiation is for — [the protocol overview page](https://smartgate.network/industry/model-context-protocol-explained) is the one page worth reading before any of the four, because every citation below assumes it. ## Sources - Model Context Protocol — specification for the current revision (2026-07-28) and the previous one (2025-11-25): [https://modelcontextprotocol.io/specification/2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28)·[https://modelcontextprotocol.io/specification/2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) - Model Context Protocol — versioning and compatibility, and the per-revision changelog: [https://modelcontextprotocol.io/specification/draft/basic/versioning](https://modelcontextprotocol.io/specification/draft/basic/versioning)·[https://modelcontextprotocol.io/specification/2026-07-28/changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog) - Model Context Protocol — schema reference, published beside each revision: [https://modelcontextprotocol.io/specification/2025-11-25/schema](https://modelcontextprotocol.io/specification/2025-11-25/schema) - Model Context Protocol — the official registry, a metadata service for discovery: [https://modelcontextprotocol.io/registry](https://modelcontextprotocol.io/registry) - Databricks — AI gateway documentation: [https://docs.databricks.com/aws/en/ai-gateway/](https://docs.databricks.com/aws/en/ai-gateway/) - Cloudflare — AI Gateway: [https://developers.cloudflare.com/ai-gateway/](https://developers.cloudflare.com/ai-gateway/)

### 5. Model Context Protocol specification – Model Context Protocol Specification
- **URL**: https://spec.modelcontextprotocol.io/specification/2024-11-05/
- ℹ️ Protocol Revision: 2024-11-05 Model Context Protocol (MCP) is an open protocol that enables seamless integration between LLM applications and external data sources and tools. Whether you’re building an AI-powered IDE, enhancing a chat interface, or creating custom AI workflows, MCP provides a standardized way to connect LLMs with the context they need.

### 6. Model Context Protocol (MCP)
- **URL**: https://patrickstox.com/ai-search/optimization/model-context-protocol/
- ## What MCP is, in one sentence Model Context Protocol is an open protocol for connecting AI applications to external data sources and tools. **Evidence for this claim** Model Context Protocol is an open protocol for connecting AI applications to external systems. **Scope:** The MCP specification and official documentation; individual host and server implementations vary. **Confidence:** high · **Verified:** [MCP: Introduction](https://modelcontextprotocol.io/docs/getting-started/intro) **Evidence for this claim** MCP defines host, client, and server roles and server primitives including resources, prompts, and tools. **Scope:** Current MCP architecture; negotiated capabilities determine which features a connection supports. **Confidence:** high · **Verified:** [MCP: Architecture](https://modelcontextprotocol.io/docs/learn/architecture) The **Model Context Protocol** is an open standard that lets AI applications connect to external tools and data sources through one common plug, instead of a custom integration for every combination. Anthropic — the company behind Claude — created it and released it as open source in November 2024. **Evidence for this claim**Model Context Protocol is an open protocol for connecting AI applications to external systems. **Scope:**The MCP specification and official documentation; individual host and server implementations vary. **Confidence:**high · **Verified:** ## Official documentation Primary-source documentation on MCP. **Anthropic / MCP project** - [Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol)— the original November 2024 announcement: the N×M problem, launch partners, and the USB-C framing. - [Donating the Model Context Protocol and establishing the Agentic AI Foundation](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation)— the December 2025 governance move to the Linux Foundation. - [What is the Model Context Protocol (MCP)?](https://modelcontextprotocol.io/docs/getting-started/intro)— the official getting-started intro. - [Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture)— host/client/server, primitives, data and transport layers, JSON-RPC mechanics. - [MCP 2026-07-28 release overview](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)— the stateless core, extensions, authorization hardening, caching, and deprecation policy. - [Discovery (](https://modelcontextprotocol.io/specification/draft/server/discover)— modern capability/version discovery and its cacheable response contract.`server/discover`) - [Streamable HTTP transport](https://modelcontextprotocol.io/specification/draft/basic/transports)— per-request protocol and routing headers plus transport security requirements. - [MCP Specification (2025-11-25)](https://modelcontextprotocol.io/specification/2025-11-25)— the preceding handshake/session-era spec, still relevant for compatibility. **Google** - [What is Model Context Protocol (MCP)? A guide](https://cloud.google.com/discover/what-is-model-context-protocol)— Google Cloud’s general developer explainer (not a Search-ranking document). **Microsoft / Bing** - [Model Context Protocol (MCP) on Windows overview](https://learn.microsoft.com/en-us/windows/ai/mcp/overview)— the Windows On-Device Agent Registry. - [microsoft/mcp](https://github.com/microsoft/mcp)— Microsoft’s catalog of MCP server implementations. - [Microsoft partners with Anthropic to create the official C# SDK](https://developer.microsoft.com/blog/microsoft-partners-with-anthropic-to-create-official-c-sdk-for-model-context-protocol)— the official C# SDK.

### 7. Specification - Model Context Protocol
- **URL**: https://mcp.mintlify.app/specification/2024-11-05
- [Model Context Protocol](https://modelcontextprotocol.io)(MCP) is an open protocol that enables seamless integration between LLM applications and external data sources and tools. Whether you’re building an AI-powered IDE, enhancing a chat interface, or creating custom AI workflows, MCP provides a standardized way to connect LLMs with the context they need. This specification defines the authoritative protocol requirements, based on the TypeScript schema in [schema.ts](https://github.com/modelcontextprotocol/specification/blob/main/schema/2025-03-26/schema.ts). For implementation guides and examples, visit [modelcontextprotocol.io](https://modelcontextprotocol.io). The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [BCP 14](https://datatracker.ietf.org/doc/html/bcp14)[ [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119)] [ [RFC8174](https://datatracker.ietf.org/doc/html/rfc8174)] when, and only when, they appear in all capitals, as shown here. ## Overview - Share contextual information with language models - Expose tools and capabilities to AI systems - Build composable integrations and workflows [JSON-RPC](https://www.jsonrpc.org/)2.0 messages to establish communication between: - **Hosts**: LLM applications that initiate connections - **Clients**: Connectors within the host application - **Servers**: Services that provide context and capabilities [Language Server Protocol](https://microsoft.github.io/language-server-protocol/), which standardizes how to add support for programming languages across a whole ecosystem of development tools. In a similar way, MCP standardizes how to integrate additional context and tools into the ecosystem of AI applications. ### Features - **Resources**: Context and data, for the user or the AI model to use - **Prompts**: Templated messages and workflows for users - **Tools**: Functions for the AI model to execute - **Sampling**: Server-initiated agentic behaviors and recursive LLM interactions

### 8. GitHub - modelcontextprotocol/modelcontextprotocol: Specification and documentation for the Model Context Protocol · GitHub
- **URL**: https://github.com/modelcontextprotocol/modelcontextprotocol
- *Just heard of MCP and not sure where to start? Check out our [documentation website](https://modelcontextprotocol.io).* This repo contains the: - MCP specification - MCP protocol schema - Official MCP documentation The schema is [defined in TypeScript](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/schema/2026-07-28/schema.ts) first, but [made available as JSON Schema](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/schema/2026-07-28/schema.json) as well, for wider compatibility. The official MCP documentation is built using Mintlify and available at [modelcontextprotocol.io](https://modelcontextprotocol.io). The Model Context Protocol was created by David Soria Parra ([@dsp](https://github.com/dsp)) and Justin Spahr-Summers ([@jspahrsummers](https://github.com/jspahrsummers)). Contributions are made through [working groups](https://modelcontextprotocol.io/community/working-interest-groups) and issues: - **Changes to the specification**are discussed and approved in the relevant working group before a pull request or[SEP](https://modelcontextprotocol.io/community/sep-guidelines)is opened. - **Pull requests**may only be opened by members of the`modelcontextprotocol`GitHub organization. - **Everyone else**can report bugs and problems by[opening an issue](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/new/choose)or get involved by joining a working group.

### 9. MCP Docs – Model Context Protocol （MCP）
- **URL**: https://modelcontextprotocol.info/docs/
- Model Context Protocol (MCP) documentation organised by reader task: get started, develop, connect and run, track protocol versions, and look up reference material for the 2026-07-28 revision. Adopting remote agent skills? Read FastMCP 4.2 and Skills over MCP for the API migration, file checks and host-support boundaries, then the Skills specification.

### 10. Specification – Model Context Protocol （MCP）
- **URL**: https://modelcontextprotocol.info/specification/2025-06-18/
- # Specification [Model Context Protocol](https://modelcontextprotocol.io) (MCP) is an open protocol that enables seamless integration between LLM applications and external data sources and tools. Whether you’re building an AI-powered IDE, enhancing a chat interface, or creating custom AI workflows, MCP provides a standardized way to connect LLMs with the context they need. This specification defines the authoritative protocol requirements, based on the TypeScript schema in [schema.ts](https://github.com/modelcontextprotocol/specification/blob/main/schema/2025-11-25/schema.ts). For implementation guides and examples, visit [modelcontextprotocol.io](https://modelcontextprotocol.io). The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [BCP 14](https://datatracker.ietf.org/doc/html/bcp14) \[[RFC2119](https://datatracker.ietf.org/doc/html/rfc2119)] \[[RFC8174](https://datatracker.ietf.org/doc/html/rfc8174)] when, and only when, they appear in all capitals, as shown here. ## Overview MCP provides a standardized way for applications to: - Share contextual information with language models - Expose tools and capabilities to AI systems - Build composable integrations and workflows The protocol uses [JSON-RPC](https://www.jsonrpc.org/) 2.0 messages to establish communication between: - **Hosts**: LLM applications that initiate connections - **Clients**: Connectors within the host application - **Servers**: Services that provide context and capabilities MCP takes some inspiration from the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/), which standardizes how to add support for programming languages across a whole ecosystem of development tools. In a similar way, MCP standardizes how to integrate additional context and tools into the ecosystem of AI applications. ## Security and Trust & Safety The Model Context Protocol enables powerful capabilities through arbitrary data access and code execution paths. With this power comes important security and trust considerations that all implementors must carefully address.
````

**2. `batch_search` (general) — compare the docs and specification sources in parallel**

```json
{
  "queries": [
    { "query": "Model Context Protocol official documentation", "max_results": 1 },
    { "query": "Model Context Protocol specification 2026-07-28", "max_results": 1 }
  ]
}
```

> `max_results` is set to `1` here only to keep the example compact; it does not change the flow.

````text
## Query 1: Model Context Protocol official documentation

## Search Results (1 results, 1102ms)

### 1. Specification - Model Context Protocol
- **URL**: https://modelcontextprotocol.io/specification/draft
- [Model Context Protocol](https://modelcontextprotocol.io)(MCP) is an open protocol that enables seamless integration between LLM applications and external data sources and tools. Whether you’re building an AI-powered IDE, enhancing a chat interface, or creating custom AI workflows, MCP provides a standardized way to connect LLMs with the context they need. This specification defines the authoritative protocol requirements, based on the TypeScript schema in [schema.ts](https://github.com/modelcontextprotocol/specification/blob/main/schema/2024-11-05/schema.ts). For implementation guides and examples, visit [modelcontextprotocol.io](https://modelcontextprotocol.io). The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [BCP 14](https://datatracker.ietf.org/doc/html/bcp14)[ [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119)] [ [RFC8174](https://datatracker.ietf.org/doc/html/rfc8174)] when, and only when, they appear in all capitals, as shown here. ## Overview - Share contextual information with language models - Expose tools and capabilities to AI systems - Build composable integrations and workflows [JSON-RPC](https://www.jsonrpc.org/)2.0 messages to establish communication between: - **Hosts**: LLM applications that initiate connections - **Clients**: Connectors within the host application - **Servers**: Services that provide context and capabilities [Language Server Protocol](https://microsoft.github.io/language-server-protocol/), which standardizes how to add support for programming languages across a whole ecosystem of development tools. In a similar way, MCP standardizes how to integrate additional context and tools into the ecosystem of AI applications. ### Features - **Resources**: Context and data, for the user or the AI model to use - **Prompts**: Templated messages and workflows for users - **Tools**: Functions for the AI model to execute - **Sampling**: Server-initiated agentic behaviors and recursive LLM interactions

---

## Query 2: Model Context Protocol specification 2026-07-28

## Search Results (1 results, 1141ms)

### 1. for consumption.
- **URL**: https://modelcontextprotocol.io/docs/2026-07-28/learn/versioning.md
- [Key passage] * **Draft**: in-progress specifications, not yet ready for consumption. * **Current**: the current protocol version, which is ready for use and may continue to receive backwards compatible changes. * **Final**: past, complete specifications that will not be changed. The **current** protocol version is [**2026-07-28**](https://modelcontextprotocol.io/specification/2026-07-28.md). [End of key passage] > ## Documentation Index > Fetch the complete documentation index at: https://modelcontextprotocol.io/llms.txt > Use this file to discover all available pages before exploring further. Versioning The Model Context Protocol uses string-based version identifiers following the format `YYYY-MM-DD`, to indicate the last date backwards incompatible changes were made. > [!NOTE] > > The protocol version will *not* be incremented when the > protocol is updated, as long as the changes maintain backwards compatibility. This allows > for incremental improvements while preserving interoperability. Revisions Revisions may be marked as: Feature States Individual features of the specification may additionally be marked as **Deprecated** under the [feature lifecycle and deprecation policy](https://modelcontextprotocol.io/community/feature-lifecycle.md): the feature remains part of the specification, but is scheduled for removal. Deprecated features document a migration path (or state that none is required) and remain in the specification for at least twelve months, or at least ninety days under the policy's before they become eligible for removal, after which they may be **Removed** in a future revision. Features that are currently Deprecated are listed in the [deprecated features registry](https://modelcontextprotocol.io/specification/2026-07-28/deprecated.md). Negotiation Every request declares the protocol version it is using via the `io.modelcontextprotocol/protocolVersion` key in its [`_meta`](https://modelcontextprotocol.io/specification/2026-07-28/basic/index#_meta) field, and the server accepts or rejects each request independently. On Streamable HTTP, the same value is also carried in the [`MCP-Protocol-Version` header](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http#protocol-version-header). Clients and servers **MAY** support multiple protocol versions simultaneously. If the server does not support the requested version, it responds with an listing the versions it does support. The client can then retry the request with a mutually supported version, or surface an error to the user if none exists. Clients that want to select a version up front can call [`server/discover`](https://modelcontextprotocol.io/specification/2026-07-28/server/discover), a mandatory RPC that returns the server's supported protocol versions, capabilities, and identity in a single request. Calling it is optional: a client is free to send any request directly and handle a version error if one comes back. For interoperability with servers and clients that implement the handshake-based protocol revisions (`2025-11-25` and earlier), see [Backward Compatibility](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning.md#backward-compatibility-with-initialization-based-versions). This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.
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
