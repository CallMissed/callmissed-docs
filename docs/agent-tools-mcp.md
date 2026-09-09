---
title: "Account MCP Server"
description: "Give an AI agent 41 tools that act on your CallMissed account: place and review calls, work the inbox, search and update the CRM, send WhatsApp and email, generate images, and check credits, over the Model Context Protocol with your API key."
slug: "agent-tools-mcp"
breadcrumb: "Getting Started"
---

# Account MCP Server

Give an AI agent 41 tools that act on your CallMissed account: place and review calls, work the inbox, search and update the CRM, send WhatsApp and email, generate images, and check credits, over the Model Context Protocol with your API key.

:::cards
/docs/mcp-server | Docs MCP Server | book | Searchable CallMissed docs inside your coding agent
/docs/quickstart | Quickstart | rocket | Make your first API call in under a minute
:::

## Overview

The **Account MCP server** lets an AI agent take actions in your CallMissed account through the [Model Context Protocol](https://modelcontextprotocol.io). Point any MCP client at one URL, authenticate with an API key, and the agent gets 41 tools: place and inspect phone calls, read transcripts, work the shared inbox, search and update the CRM, summarise and score calls, send WhatsApp messages and email, generate images, and check what it all cost.

It is hosted, so there is nothing to install and nothing to run locally.

<Callout type="info">
  This is a **different server** from the [Docs MCP Server](/docs/mcp-server). That one is a local npm package that makes this documentation searchable inside your editor and needs no credentials. This one is hosted, needs an API key, and acts on your real account and credits.
</Callout>

## Endpoint

```
https://api.callmissed.com/api/v1/mcp
```

The transport is **Streamable HTTP**: every call is a single `POST` that returns one JSON response. There is no session to open or close, so the server works behind any load balancer and needs no sticky routing. `GET` and `DELETE` return `405` by design, because this server does not offer a server-initiated event stream.

The server answers `initialize`, `tools/list`, `tools/call`, `resources/list` and `resources/read`, and declares the `tools` and `resources` capabilities.

## Authentication

Authenticate with an API key from your dashboard, exactly as you would for any other CallMissed endpoint:

```
Authorization: Bearer cm_your_api_key
```

**API keys only.** A dashboard login token is refused with `401`, even though it works elsewhere in the API. Scope checks are what keep an agent inside its lane, and those only apply to API keys, so this endpoint accepts nothing else.

Everything attached to the key still applies: its scopes, its spend budget, its rate limit, and its domain allowlist. A key that runs out of budget gets `402`; one over its rate limit gets `429`.

## What a key needs

Two different gates decide whether a tool works:

* **Resource scopes** such as `contacts:read` or `whatsapp:send`. Most tools use these, and the table below names the one each tool needs. A tool whose scope is missing returns a readable refusal naming the scope to add, so the agent can tell you what to fix.
* **Service permissions** on the key. The three voice-session tools need a key with the `stt`, `tts` and `llm` permissions; the two image tools need the `image` permission. `get_credit_balance` needs nothing beyond a valid key.

Give each key only what its agent needs. A read-only key is a perfectly good way to let an agent look at the account without letting it spend anything or reach a real person.

<Callout type="warn">
  Eight tools spend credits and several of them reach real people. They are marked in the tables below. Consider a separate key holding only the read scopes for agents that should look but not act.
</Callout>

## Tools

41 tools in eight groups. `Access` is the tool's own `readOnlyHint`. `Credits` marks the tools that draw down your balance. Every tool's full JSON Schema, including argument names, types and bounds, comes back from `tools/list`.

### Voice and calls

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `place_call` | Place an outbound call from one of your numbers, answered by one of your voice agents | Write | `telephony:write` | Yes |
| `list_calls` | List calls, newest first, with status, duration and cost | Read | `telephony:read` | No |
| `get_call` | Fetch one call by id, with hangup cause and cost | Read | `telephony:read` | No |
| `end_call` | Hang up a call that is still in progress. Immediate and not undoable | Write, destructive | `telephony:write` | No |
| `get_call_recording` | Get a short-lived download link for a call's recording | Read | `telephony:read` | No |
| `click_to_call` | Ring one of your people, dial the contact, and bridge the two. No AI agent involved | Write | `telephony:write` | Yes |
| `list_phone_numbers` | List your numbers and their status, to find the `from_number_id` `place_call` takes | Read | `telephony:read` | No |
| `list_voice_sessions` | List voice agent sessions, newest first | Read | `stt` + `tts` + `llm` permissions | No |
| `get_voice_session_transcript` | Get a session's full turn-by-turn transcript | Read | `stt` + `tts` + `llm` permissions | No |
| `get_voice_session_cost` | Break down what one voice session cost in credits | Read | `stt` + `tts` + `llm` permissions | No |

### Conversations

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_conversations` | List conversations across channels with a preview and unread count | Read | `conversations:read` | No |
| `get_conversation_messages` | Read the messages in one conversation, oldest first | Read | `conversations:read` | No |
| `set_conversation_status` | Change a conversation's status, for example to close or escalate it | Write | `conversations:write` | No |
| `list_handoffs` | List conversations escalated to a person and still waiting | Read | `conversations:read` | No |
| `resolve_handoff` | Mark an escalated conversation handled, optionally keeping the AI quiet | Write | `conversations:write` | No |

### CRM

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `crm_search` | Search contacts, companies, deals, notes and tasks in one call | Read | `crm_search:read` | No |
| `crm_timeline` | One contact's or company's full history: conversations, calls, notes, tasks, deals | Read | `crm_timeline:read` | No |
| `list_contacts` | List contacts with their per-channel opt-in state | Read | `contacts:read` | No |
| `create_contact` | Add a person to the CRM. Needs at least a phone or an email | Write | `contacts:write` | No |
| `update_contact` | Change fields on an existing contact | Write | `contacts:write` | No |
| `get_contact_memory` | Get the facts your agents have learned about one contact | Read | `contacts:read` | No |
| `create_deal` | Open a deal in a pipeline | Write | `crm_deals:write` | No |
| `move_deal` | Move a deal to another stage of its own pipeline | Write | `crm_deals:write` | No |
| `create_task` | Add a follow-up task, optionally attached to a record | Write | `crm_tasks:write` | No |
| `complete_task` | Mark a task done | Write | `crm_tasks:write` | No |
| `create_crm_note` | Attach a note to a contact, company or deal | Write | `crm_notes:write` | No |

### Call intelligence

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `get_call_notes` | Get the notes already generated for a call: summary, action items, outcome | Read | `conversations:read` | No |
| `generate_call_notes` | Read a call's transcript and write structured notes from it | Write | `conversations:write` | Yes |
| `score_call` | Grade a call against one of your scorecards and store the result | Write | `conversations:write` | Yes |

### Messaging

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `send_whatsapp_message` | Send a free-form WhatsApp text, inside the 24-hour service window only | Write | `whatsapp:send` | Yes |
| `send_whatsapp_template` | Send an approved template, the only way to reach someone outside that window | Write | `whatsapp:send` | Yes |
| `list_whatsapp_campaigns` | List broadcast campaigns with status and recipient counts | Read | `whatsapp:read` | No |
| `send_email` | Send an email from a domain this account has verified | Write | `email` permission | Yes |

### Images

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `generate_image` | Generate an image from a prompt and get a link to it | Write | `image` permission | Yes |
| `list_generated_images` | List images made with this key, with fresh short-lived links | Read | `image` permission | No |

### Agents and knowledge

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_agents` | List the voice and chat agents on the account, to find a `bot_id` | Read | `bots:read` | No |
| `get_agent` | Fetch one agent's configuration by id | Read | `bots:read` | No |
| `knowledge_search` | Search the knowledge base and get the passages that match | Read | `knowledge:read` | No |
| `add_agent_memory` | Store a durable fact for one agent, applied on every future conversation | Write | `bots:write` | No |

### Usage and credits

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `get_credit_balance` | Check how many credits are left on the account | Read | Any valid key | No |
| `get_usage_summary` | Summarise recent spend, broken down by service | Read | `usage:read` | No |

### Telephony tools are deployment-gated

Seven of the 41 tools are the telephony ones: `place_call`, `list_calls`, `get_call`, `end_call`, `get_call_recording`, `click_to_call` and `list_phone_numbers`. They appear only where calling is switched on for the deployment. Where it is not, `tools/list` returns the other 34 and the seven are simply absent, because the REST routes behind them are not mounted there. Calling one by name anyway returns the same `Unknown tool` error as a typo.

If you do not see them and you expect to, talk to us and we will get calling turned on for you.

## Public catalog

```
GET https://api.callmissed.com/api/v1/mcp/catalog
```

Unauthenticated, no key needed. It returns the live tool list this deployment serves, which is what the tables above are built from, so you can check what is available before you wire anything up.

```bash
curl https://api.callmissed.com/api/v1/mcp/catalog
```

```json
{
  "serverUrl": "https://api.callmissed.com/api/v1/mcp",
  "protocolVersion": "2025-06-18",
  "count": 41,
  "categories": { "voice": "Voice and calls", "crm": "CRM" },
  "tools": [
    {
      "name": "list_contacts",
      "title": "List contacts",
      "description": "List contacts in your address book, newest first...",
      "category": "crm",
      "categoryLabel": "CRM",
      "readOnly": true,
      "destructive": false,
      "costsCredits": false,
      "scope": "contacts:read"
    }
  ]
}
```

It carries declarations only: no argument schemas, nothing about the caller, nothing about tools this deployment has gated off. For the JSON Schema of a tool's arguments, call `tools/list` with your key.

## Interactive views

Some results render as an interactive view instead of a wall of JSON, using the official MCP Apps extension (`io.modelcontextprotocol/ui`). Each view is one self-contained HTML document served over `resources/read`; a tool points at it through `_meta.ui.resourceUri` in its declaration.

| View | Shows | Tools that use it |
| --- | --- | --- |
| `ui://callmissed/call-log` | Calls with status, duration and cost | `list_calls` |
| `ui://callmissed/transcript` | A call's turns, caller and agent side by side | `get_voice_session_transcript` |
| `ui://callmissed/inbox` | Conversations with previews and unread counts | `list_conversations`, `list_handoffs` |
| `ui://callmissed/timeline` | A contact's history in one stream | `crm_timeline` |
| `ui://callmissed/image-gallery` | Generated images | `generate_image`, `list_generated_images` |

So five views across seven tools, or six tools where telephony is off.

**Where they render.** claude.ai, Claude Desktop, Claude mobile and Claude Cowork render MCP Apps, as do ChatGPT, VS Code Copilot and Cursor. The **Claude Code terminal does not**: it shows the text result instead. That is not a degraded mode. Every tool returns its full data as text and as `structuredContent` whether or not a view exists, so a client that ignores the extension loses nothing but the pictures.

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

Keep the key out of version control. Reference an environment variable if your client supports it.

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

A successful call returns the data twice, once as `structuredContent` and once serialized into a text block, so clients that only read one shape still work:

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

Two different shapes, matching the protocol.

**A malformed request**, meaning an unknown tool, a missing argument, or a bad method, comes back as a JSON-RPC `error`:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "error": { "code": -32602, "message": "Unknown tool: send_sms" }
}
```

**A refused action**, meaning a missing scope, no credits, or a closed messaging window, is a successful result carrying `isError`, so the agent can read the reason and adapt instead of treating it as a crash:

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

Omit the header and the server assumes `2025-03-26`, per the specification. Send a version it does not support and the call returns `400`. Batched requests are not accepted, because the `2025-06-18` revision removed them, so send one request object per POST.
