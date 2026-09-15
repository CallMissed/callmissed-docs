---
title: "Account MCP Server"
description: "Give an AI agent 41 tools that act on your CallMissed account: place and review calls, work the inbox, search and update the CRM, send WhatsApp and email, generate images, and check credits, over the Model Context Protocol. Connect by signing in with your CallMissed account, or with an API key."
slug: "agent-tools-mcp"
breadcrumb: "Getting Started"
---

# Account MCP Server

Give an AI agent 41 tools that act on your CallMissed account: place and review calls, work the inbox, search and update the CRM, send WhatsApp and email, generate images, and check credits, over the Model Context Protocol. Connect by signing in with your CallMissed account, or with an API key.

:::cards
/docs/mcp-server | Docs MCP Server | book | Searchable CallMissed docs inside your coding agent
/docs/quickstart | Quickstart | rocket | Make your first API call in under a minute
:::

## Overview

The **Account MCP server** lets an AI agent take actions in your CallMissed account through the [Model Context Protocol](https://modelcontextprotocol.io). Point any MCP client at one URL, sign in with your CallMissed account or pass an API key, and the agent gets 41 tools: place and inspect phone calls, read transcripts, work the shared inbox, search and update the CRM, summarise and score calls, send WhatsApp messages and email, generate images, and check what it all cost.

It is hosted, so there is nothing to install and nothing to run locally.

<Callout type="info">
  This is a **different server** from the [Docs MCP Server](/docs/mcp-server). That one is a local npm package that makes this documentation searchable inside your editor and needs no credentials. This one is hosted, needs your account, and acts on your real data and credits.
</Callout>

## Endpoint

```
https://api.callmissed.com/api/v1/mcp
```

The transport is **Streamable HTTP**: every call is a single `POST` that returns one JSON response. There is no session to open or close, so the server works behind any load balancer and needs no sticky routing. `GET` and `DELETE` return `405` by design, because this server does not offer a server-initiated event stream.

The server answers `initialize`, `tools/list`, `tools/call`, `resources/list` and `resources/read`, and declares the `tools` and `resources` capabilities.

## Connect

Two ways in. Both reach the same tools; they differ in how the connection is authorised and how much you have to set up.

### Option 1: sign in with your CallMissed account

The easy path, and the recommended one. Paste the server URL into your client, click **Connect**, sign in to CallMissed, and tick what the connection is allowed to do. There is no key to create and nothing to paste into a header.

```
https://api.callmissed.com/api/v1/mcp
```

Paste that URL exactly as written, path included. Discovery is bound to the URL you type, so a shortened or altered one is not recognised as this server and sign-in never starts.

If your client cannot complete the sign-in, use an API key instead (option 2 below).

#### Choose what to allow

The consent screen offers three permission groups.

| Group | What it allows | On by default | Can spend credits |
| --- | --- | --- | --- |
| **Read your data** | See your calls, contacts, companies, deals, tasks, notes, conversations, agents and usage. Cannot change anything. | Yes | No |
| **Read call transcripts** | Read what was said on your voice calls, and what each one cost. This also lets the connection use AI models on your account, which spends credits. | No | Yes |
| **Take actions** | Create and update records, send WhatsApp messages and email, place and end calls, and generate images. Spends credits. | No | Yes |

**Read your data** is the only group ticked when the screen opens. The other two start off and are granted only if you tick them yourself, so a single click cannot hand an agent the ability to message a customer or spend credits.

At least one group has to be ticked. Approving nothing would create a connection that fails every call, so the consent screen asks you to choose something or press **Deny**.

Against the tool tables further down: **Read your data** covers every `:read` scope, **Read call transcripts** grants the `stt`, `tts` and `llm` permissions the three voice-session tools need, and **Take actions** covers every `:write` scope plus `whatsapp:send` and the `image` and `email` permissions.

#### The connection appears as an API key

Approving creates an API key on your account named `MCP connection: {host}`, where the host is the one shown on the consent screen as the client that asked. You will find it at **Developer → API keys** in the [console](https://console.callmissed.com/developer/keys) next to your other keys, carrying exactly the scopes you ticked, with the same budget, rate limit and credit accounting as any key you make by hand. Request logging is off for it.

Its value is never shown and cannot be revealed or copied out, so it can only ever be used by the connection it was made for.

**To disconnect, deactivate that key.** That is the whole revocation story: once the key is inactive, every token issued to that connection stops working immediately.

#### Staying connected

The access token your client receives is short-lived and your client renews it in the background using the refresh token issued alongside it. You are not asked to sign in again each time you use the tools. You will be asked again if you deny, if the key is deactivated, if you want to change what is allowed, or after the connection has gone unused for a long stretch.

#### Discovery endpoints

A client finds the flow from the URL you paste, by fetching these:

| Endpoint | What it is for |
| --- | --- |
| `GET /.well-known/oauth-protected-resource` | Protected-resource metadata (RFC 9728): the MCP resource URL, the authorization server, and the scopes this server understands. |
| `GET /.well-known/oauth-protected-resource/api/v1/mcp` | The same document at the path-suffixed address, which is what a client derives from the full server URL. Both forms are published so discovery works either way. |
| `GET /.well-known/oauth-authorization-server` | Authorization-server metadata (RFC 8414): the authorization and token endpoints, the supported grant types, and the PKCE methods. |

The flow itself is a standard OAuth 2.1 authorization code exchange:

| Endpoint | Method |
| --- | --- |
| `https://api.callmissed.com/api/v1/mcp/oauth/authorize` | `GET` |
| `https://api.callmissed.com/api/v1/mcp/oauth/token` | `POST`, form-encoded |

Two things to know if you are writing the client yourself:

* **PKCE with `S256` is required.** There is no client secret, so a request without `code_challenge_method=S256` is refused.
* **`client_credentials` is not supported.** The only grant types are `authorization_code` and `refresh_token`, because every connection needs a person to approve it on the consent screen.

There is no registration step. `client_id` is an `https://` URL that identifies your client, and the consent screen shows its host as plain text so the person approving can see who is asking.

Redirect addresses are limited to ones CallMissed has approved, plus loopback addresses for native apps. Any other `redirect_uri` is refused with a `400` before the consent screen is ever shown, so nothing is redirected to an address we have not vetted.

### Option 2: connect with an API key

For clients that do not do the sign-in flow, and for calling the endpoint yourself, pass an API key from your dashboard exactly as you would for any other CallMissed endpoint:

```
Authorization: Bearer cm_your_api_key
```

**A dashboard login token is refused with `401`**, even though it works elsewhere in the API. Scope checks are what keep an agent inside its lane, and those apply to keys, so this endpoint takes an API key or a token from the sign-in flow above and nothing else.

Everything attached to the key still applies: its scopes, its spend budget, its rate limit, and its domain allowlist. A key that runs out of budget gets `402`; one over its rate limit gets `429`.

## What a connection needs

Two different gates decide whether a tool works, whether the connection came from signing in or from a key you made by hand:

* **Resource scopes** such as `contacts:read` or `whatsapp:send`. Most tools use these, and the table below names the one each tool needs. A tool whose scope is missing returns a readable refusal naming the scope to add, so the agent can tell you what to fix.
* **Service permissions** on the key. The three voice-session tools need a key with the `stt`, `tts` and `llm` permissions; the two image tools need the `image` permission. `get_credit_balance` needs nothing beyond a valid key.

Give each key only what its agent needs. A read-only key is a perfectly good way to let an agent look at the account without letting it spend anything or reach a real person.

<Callout type="warn">
  Eight tools spend credits and several of them reach real people. They are marked in the tables below. For an agent that should look but not act, tick only **Read your data** when you sign in, or use a key holding only the read scopes.
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
| `generate_image` | Generate an image from a prompt. Returns the image itself, so the model can see it, plus a link to the full-size version | Write | `image` permission | Yes |
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

If your client supports signing in, add the server by URL alone and click **Connect**: no header, no key. See [option 1](#option-1-sign-in-with-your-callmissed-account) above.

Otherwise, most MCP clients accept a remote server as a URL plus a header. In Claude Code:

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

Authentication failures happen before any tool runs and use normal HTTP status codes: `401` for a missing or malformed credential, a dashboard login token, or a connection that has been revoked; `402` when the key is out of budget; `429` when it is over its rate limit.

Every `401` from this endpoint carries a `WWW-Authenticate` header pointing at the protected-resource document, including the very first call a client makes with no credential at all. That is what lets a client offer "Connect and sign in" instead of asking you to paste a key.

## Protocol version

The server implements MCP revision `2025-06-18` and also accepts `2025-03-26`. Send the version you speak on every call after initializing:

```
MCP-Protocol-Version: 2025-06-18
```

Omit the header and the server assumes `2025-03-26`, per the specification. Send a version it does not support and the call returns `400`. Batched requests are not accepted, because the `2025-06-18` revision removed them, so send one request object per POST.
