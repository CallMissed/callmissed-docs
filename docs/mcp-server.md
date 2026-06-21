---
title: "Docs MCP Server"
description: "Connect the CallMissed documentation to your AI coding agent with the official callmissed-docs MCP server — searchable docs inside Claude, Cursor, VS Code, Windsurf, and OpenCode."
slug: "mcp-server"
breadcrumb: "Getting Started"
---

# Docs MCP Server

Connect the CallMissed documentation to your AI coding agent with the official callmissed-docs MCP server — searchable docs inside Claude, Cursor, VS Code, Windsurf, and OpenCode.

:::cards
/docs/quickstart | Quickstart | rocket | Make your first API call in under a minute
/docs/sdks | SDKs & Libraries | package | Use the OpenAI & Anthropic SDKs
:::

## Overview

The **CallMissed Docs MCP server** brings this entire documentation site into your AI coding agent through the [Model Context Protocol](https://modelcontextprotocol.io). Instead of copy-pasting docs, your agent can search, grep, and read CallMissed docs directly while it writes your integration.

It is published on npm as [`callmissed-docs-mcp`](https://www.npmjs.com/package/callmissed-docs-mcp), runs locally over stdio via `npx`, and stays current by fetching the live docs export from the CallMissed API with an offline cache fallback.

- **Always up to date** — pulls the latest docs on startup, caches for 1 hour
- **Works offline** — falls back to the local cache when the API is unreachable
- **Fast search** — fuzzy search plus regex grep over every doc page
- **Zero install** — `npx` runs the latest version on demand

> **Tip:** Use the **Copy page** dropdown at the top of any docs page to copy the MCP config or one-click connect to Cursor and VS Code.

## Install

The server runs through `npx`, so no global install is required. To install it globally anyway:

```bash
npm install -g callmissed-docs-mcp
```

## Connect Your Client

Add the server to your client's MCP configuration.

:::tabs
```json [Claude Desktop]
{
  "mcpServers": {
    "callmissed-docs": {
      "command": "npx",
      "args": ["-y", "callmissed-docs-mcp"]
    }
  }
}
```
```json [Cursor]
{
  "mcpServers": {
    "callmissed-docs": {
      "command": "npx",
      "args": ["-y", "callmissed-docs-mcp"],
      "env": {}
    }
  }
}
```
```json [VS Code]
{
  "servers": {
    "callmissed-docs": {
      "command": "npx",
      "args": ["-y", "callmissed-docs-mcp"],
      "type": "stdio"
    }
  }
}
```
```json [Windsurf]
{
  "mcpServers": {
    "callmissed-docs": {
      "command": "npx",
      "args": ["-y", "callmissed-docs-mcp"]
    }
  }
}
```
```json [OpenCode]
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "callmissed-docs": {
      "type": "local",
      "command": ["npx", "-y", "callmissed-docs-mcp"],
      "enabled": true
    }
  }
}
```
:::

Config file locations:

| Client | Path |
|--------|------|
| Claude Desktop | `claude_desktop_config.json` |
| Cursor | `~/.cursor/mcp.json` |
| VS Code | `.vscode/mcp.json` (workspace) or User Configuration |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` |
| OpenCode | `opencode.json` (project root) |

After adding the config, restart your client. The CallMissed docs tools then appear automatically alongside your other tools.

## Use via Context7

[Context7](https://context7.com) aggregates library documentation for AI agents. CallMissed docs are indexed there under the library ID `/callmissed/callmissed-docs` — so if you already run the Context7 MCP server, you can pull CallMissed docs through it without installing a separate server.

Reference the library directly in your prompt:

```text
Use the CallMissed docs (/callmissed/callmissed-docs) from Context7 to wire up streaming chat completions.
```

> **Tip:** The dedicated `callmissed-docs-mcp` server always serves the live docs export. Context7 is a convenient option when it's already part of your toolchain.

## Available Tools

Once connected, your agent gains these tools:

| Tool | Description |
|------|-------------|
| `callmissed_search` | Fuzzy search across all documentation |
| `callmissed_grep` | Regex search over doc content |
| `callmissed_cat` | Read a documentation chunk by ID |
| `callmissed_ls` | List available documentation pages |
| `callmissed_find` | Substring search across pages |

## Configuration

The server reads one optional environment variable:

| Variable | Default | Description |
|----------|---------|-------------|
| `CALLMISSED_DOCS_API` | `https://api.callmissed.com/api/v1/docs/export` | Docs export endpoint the server fetches on startup |

To point the server at a self-hosted or staging docs export, set the variable in your MCP client's `env` block:

```json
{
  "mcpServers": {
    "callmissed-docs": {
      "command": "npx",
      "args": ["-y", "callmissed-docs-mcp"],
      "env": {
        "CALLMISSED_DOCS_API": "https://staging-api.callmissed.com/api/v1/docs/export"
      }
    }
  }
}
```

**Ready to connect?** Use the **Copy page** menu at the top of any docs page to grab the config or one-click install into Cursor or VS Code.