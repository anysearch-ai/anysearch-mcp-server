# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in this project, please report it responsibly.

**Do NOT open a public GitHub issue for security vulnerabilities.**

### How to Report

Send an email to **security@anysearch.com** with:

- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

### Response Timeline

| Action | Timeframe |
|--------|-----------|
| Acknowledgment | Within 48 hours |
| Initial assessment | Within 5 business days |
| Fix release | Depends on severity |

### Scope

This policy covers:
- The AnySearch MCP server endpoint (`api.anysearch.com`)
- This repository's documentation and configuration examples
- Official client integrations

### Out of Scope

- Third-party proxy tools (`mcp-remote`, `supergateway`)
- User misconfiguration of API keys

## Supported Versions

| Version | Supported |
|---------|-----------|
| Latest  | Yes       |

## Security Best Practices for Users

- Store API keys in environment variables, never in code
- Use `.env` files locally (add to `.gitignore`)
- Rotate API keys periodically
- Use the minimum required permissions
