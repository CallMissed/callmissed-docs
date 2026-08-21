---
title: "Account MCP Server"
description: "Give an AI agent tools that act on your CallMissed account — send WhatsApp messages, review campaigns, conversations and contacts, and place calls — over the Model Context Protocol with your API key."
slug: "agent-tools-mcp"
breadcrumb: "Getting Started"
---

# Account MCP Server

Give an AI agent tools that act on your CallMissed account — send WhatsApp messages, review campaigns, conversations and contacts, and place calls — over the Model Context Protocol with your API key.

:::cards
/docs/mcp-server | Docs MCP Server | book | Searchable CallMissed docs inside your coding agent
/docs/quickstart | Quickstart | rocket | Make your first API call in under a minute
:::

## Overview

The **Account MCP server** lets an AI agent take actions in your CallMissed account through the [Model Context Protocol](https://modelcontextprotocol.io). Point any MCP client at one URL, authenticate with an API key, and the agent can send a WhatsApp message, look through conversations and contacts, or place a call with one of your voice agents.

It is hosted, so there is nothing to install and nothing to run locally.

<Callout type="info">
  This is a **different server** from the [Docs MCP Server](/docs/mcp-server). That one is a local npm package that makes this documentation searchable inside your editor and needs no credentials. This one is hosted, needs an API key, and acts on your real account and credits.
</Callout>

## Endpoint

```
https://api.callmissed.com/api/v1/mcp
```

The transport is **Streamable HTTP**: every call is a single `POST` that returns one JSON response. There is no session to open or close, so the server works behind any load balancer and needs no sticky routing. `GET` and `DELETE` return `405` by design — this server does not offer a server-initiated event stream.

## Authentication

Authenticate with an API key from your dashboard, exactly as you would for any other CallMissed endpoint:

```
Authorization: Bearer cm_your_api_key
```

**API keys only.** A dashboard login token is refused with `401`, even though it works elsewhere in the API. Scope checks are what keep an agent inside its lane, and those only apply to API keys — so this endpoint accepts nothing else.

Everything attached to the key still applies: its scopes, its spend budget, its rate limit, and its domain allowlist. A key that runs out of budget gets `402`; one over its rate limit gets `429`.

## Scopes

Give each key only the scopes the agent needs. A tool whose scope is missing returns a readable refusal naming the scope to add, so the agent can tell you what to fix.

| Tool | Scope required |
| --- | --- |
| `send_whatsapp_message` | `whatsapp:send` |
| `list_whatsapp_campaigns` | `whatsapp:read` |
| `list_conversations` | `conversations:read` |
| `list_contacts` | `contacts:read` |
| `place_call` | `telephony:write` |

<Callout type="warn">
  `send_whatsapp_message` and `place_call` spend credits and reach real people. Consider a separate key with only the read scopes for agents that should look but not act.
</Callout>

## Tools

### `send_whatsapp_message`

Send a free-form WhatsApp text from one of your connected business numbers.

| Argument | Type | Notes |
| --- | --- | --- |
| `to` | string | **Required.** Recipient in E.164 form, e.g. `+919000000000`. |
| `text` | string | **Required.** Message body, up to 4096 characters. |
| `phone_id` | string | Which connected number to send from. Optional when you have one. |
| `phone_number_id` | string | Alternative selector for the sending number. |

Free-form text only reaches someone inside the 24-hour customer service window. Outside it, send an approved template from the [messages API](/docs/whatsapp-messages).

### `list_whatsapp_campaigns`

List your broadcast campaigns, newest first, with status and recipient counts.

| Argument | Type | Notes |
| --- | --- | --- |
| `limit` | integer | 1–100, default 50. |

### `list_conversations`

List conversations across channels, newest first, each with a preview of the latest message and an unread count.

| Argument | Type | Notes |
| --- | --- | --- |
| `channel` | string | Restrict to one channel, e.g. `whatsapp`, `voice`, `web`. |
| `status` | string | Restrict to one status, e.g. `active`, `escalated`, `closed`. |
| `bot_id` | string | Restrict to conversations handled by one agent. |
| `limit` | integer | 1–500, default 50. |
| `offset` | integer | For paging. |

### `list_contacts`

List contacts in your address book, newest first, with their per-channel opt-in state.

| Argument | Type | Notes |
| --- | --- | --- |
| `q` | string | Substring match on phone, email, or name. |
| `limit` | integer | 1–200, default 50. |
| `offset` | integer | For paging. |

### `place_call`

Place an outbound call from one of your numbers, answered by one of your voice agents.

| Argument | Type | Notes |
| --- | --- | --- |
| `from_number_id` | string | **Required.** Which of your numbers to call from. |
| `to_e164` | string | **Required.** Number to call, in E.164 form. |
| `bot_id` | string | Which voice agent handles the call. |
| `reason` | string | Plain-language purpose; the agent uses it in its opening line. |
| `variables` | object | Per-call `{{token}}` values merged into the greeting and prompt. |

<Callout type="info">
  `place_call` only appears in `tools/list` on accounts where calling is switched on. If you do not see it, calling is not enabled for your account yet — talk to us and we will turn it on.
</Callout>

## Connect a client

Most MCP clients accept a remote server as a URL plus a header. In Claude Code:

```bash
claude mcp add --transport http callmissed \
  https://api.callmissed.com/api/v1/mcp \
  --header "Authorization: Bearer cm_your_api_key"
```

For clients configured by file, the shape is usually:

```json
{
  "mcpServers": {
    "callmissed": {
      "type": "http",
      "url": "https://api.callmissed.com/api/v1/mcp",
      "headers": {
        "Authorization": "Bearer cm_your_api_key"
      }
    }
  }
}
```

Keep the key out of version control — reference an environment variable if your client supports it.

## Call it directly

The endpoint is plain JSON-RPC, so you can drive it with `curl`. List the tools your key can reach:

```bash
curl https://api.callmissed.com/api/v1/mcp \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list"
  }'
```

Call one:

```bash
curl https://api.callmissed.com/api/v1/mcp \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "list_contacts",
      "arguments": { "q": "jane", "limit": 10 }
    }
  }'
```

A successful call returns the data twice — once as `structuredContent` and once serialized into a text block — so clients that only read one shape still work:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [{ "type": "text", "text": "[{\"id\":\"...\",\"name\":\"Jane\"}]" }],
    "structuredContent": { "result": [{ "id": "...", "name": "Jane" }] },
    "isError": false
  }
}
```

## Errors

Two different shapes, matching the protocol:

**A malformed request** — an unknown tool, a missing argument, a bad method — comes back as a JSON-RPC `error`:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "error": { "code": -32602, "message": "Unknown tool: send_sms" }
}
```

**A refused action** — a missing scope, no credits, a closed messaging window — is a successful result carrying `isError`, so the agent can read the reason and adapt instead of treating it as a crash:

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "content": [{ "type": "text", "text": "API key missing required scope: whatsapp:send." }],
    "isError": true
  }
}
```

Authentication failures happen before any tool runs and use normal HTTP status codes: `401` for a missing, malformed, or non-API-key credential, `402` when the key is out of budget, `429` when it is over its rate limit.

## Protocol version

The server implements MCP revision `2025-06-18` and also accepts `2025-03-26`. Send the version you speak on every call after initializing:

```
MCP-Protocol-Version: 2025-06-18
```

Omit the header and the server assumes `2025-03-26`, per the specification. Send a version it does not support and the call returns `400`. Batched requests are not accepted — the `2025-06-18` revision removed them, so send one request object per POST.
