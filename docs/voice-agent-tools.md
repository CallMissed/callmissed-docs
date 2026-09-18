---
title: "Voice Agent Tools"
description: "What a voice agent can call mid-conversation on a phone, WhatsApp or WebRTC call: built-in tools, your own REST tools, MCP servers, and the two tools that are always there."
slug: "voice-agent-tools"
breadcrumb: "Voice Agents"
---

# Voice Agent Tools

What a voice agent can call mid-conversation on a phone, WhatsApp or WebRTC call: built-in tools, your own REST tools, MCP servers, and the two tools that are always there.

## Overview

A voice agent on a phone, WhatsApp or WebRTC call can call tools mid-sentence —
look an order up, check a calendar, send the caller a link, hang up. This page
covers that surface.

:::cards
/docs/agent-tools | Custom REST Tools | cable | Define, test and attach your own HTTP endpoints
/docs/bots | Agents | bot | `config.tools` and the tool catalogue
/docs/managed-voice-agent | Managed Voice Agent | radio | Tool calling over the WebSocket gateway
:::

<Callout>
The [Managed Voice Agent](/docs/managed-voice-agent) WebSocket gateway has its
own tool protocol — you declare tools in `Settings` and answer
`FunctionCallRequest` over the same socket, with a 30-second response window.
That is a different mechanism from this page: there, your client runs the tool;
here, we do.
</Callout>

## Four sources, one tool list

Everything below is merged into a single list the model sees on the call.

| Source | Where it comes from | Runs where |
| --- | --- | --- |
| Built-in tools | `config.tools` on the agent, by name | Our backend |
| Custom REST tools | The [agent tools API](/docs/agent-tools) | Your endpoint, called by our backend |
| MCP servers | `config.mcp_servers` on the agent | Your MCP server |
| `end_call`, `transfer_to_human` | Always attached | Our backend |

A custom REST tool and a built-in tool cannot share a name — the custom tool
wins and the built-in one is dropped, so the model never sees two tools with
one name.

## Built-in tools

Enable them by name in the agent's `config.tools`. `GET /api/v1/bots/tool-catalog`
([Agents](/docs/bots)) lists every name you can put there.

**`web_search` is always on** — it is enabled for every agent whether or not it
appears in `config.tools`.

| Category | Examples |
| --- | --- |
| Utility | `web_search`, `calculator`, `get_current_time` |
| Knowledge | `search_knowledge_base`, `save_to_knowledge` |
| Memory | `remember`, `recall_memory` |
| CRM | `update_contact`, `set_contact_optin` |
| Scheduling | `calcom_list_slots`, `calcom_book`, `google_calendar_find_free_slots`, `google_calendar_create_event` |
| Commerce | `shopify_order_status`, `shopify_product_lookup`, `woocommerce_order_status` |
| WhatsApp messaging | `send_text_message`, `send_template_message`, `send_quick_reply_buttons`, `send_list_menu`, `send_cta_url_button`, `send_location` |
| Calling | `request_call` |
| Email | `send_email`, `gmail_send_email` |
| HTTP | `http_request` |

A tool whose integration is not connected — a Shopify store, a Cal.com key, a
Google account — does not break the call. It returns an error the model can
speak its way out of ("I can't reach the calendar right now"), and the
conversation continues.

An unknown name in `config.tools` is skipped and logged rather than taking the
agent offline, so a stale entry left behind by a renamed tool is not an outage.

### Messaging tools on calls

On **WhatsApp calls and phone calls**, the WhatsApp messaging tools above are
enabled automatically — you do not have to list them. The most common request on
a voice call is "send me that in the chat": a tracking link, an address, a
price. If the call is not linked to a connected WhatsApp sender, the tool
returns an error the model relays instead of sending anything.

### Not available on voice

Two categories are excluded from voice calls:

| Category | Why |
| --- | --- |
| `conversation` | Inbox-thread actions — notes, tags, status, escalation — belong to the chat channels |
| `personal_whatsapp` | Needs a linked personal-WhatsApp session, which a call does not have |

For escalation on a call, use `transfer_to_human` below.

## Custom REST tools

Your own read-only HTTP endpoint, described so the model can call it
mid-conversation. Create and test it through the
[agent tools API](/docs/agent-tools) — the same definitions the console's
**Tools** screen writes — and it is attached to the call automatically.

On a voice call each invocation is one round trip to our backend, which owns the
URL, your stored credentials, the outbound-request checks, the per-tool timeout
and the response-size cap. Your credentials never reach the caller's client.

A failing custom tool returns a JSON error object to the model rather than
raising, so a broken endpoint costs one turn, not the call. A definition we
cannot attach is skipped and the rest stay — one bad tool does not drop the
others.

## MCP servers

Point an agent at remote MCP servers with `config.mcp_servers`. Each server's
tools are listed at call setup and offered to the model alongside everything
else.

```json
{
  "mcp_servers": [
    {
      "id": "inventory",
      "url": "https://mcp.yourcompany.com/mcp",
      "header_value": "Bearer your_server_token",
      "allowed_tools": ["check_stock", "reserve_item"]
    }
  ]
}
```

| Field | Required | Notes |
| --- | --- | --- |
| `id` | Yes | Your label for the server, up to 64 characters |
| `url` | Yes | **HTTPS only**, and must resolve to a public address |
| `header_value` | No | Sent as the `Authorization` header to your server. Never logged |
| `allowed_tools` | No | Allow-list of tool names. Omit it and every tool the server lists is offered |

| Limit | Value |
| --- | --- |
| Servers per agent | 10 |
| Tools taken per server | 40 |
| Tools taken across all servers | 100 |

A server that is unreachable or slow to list is skipped for that call and logged
— the agent still answers with its remaining tools. Tool listings are cached
briefly per server, so a burst of calls does not re-list on every one.

<Callout type="warn">
Tool descriptions and tool results from an MCP server are untrusted text that
reaches the model. A hostile or compromised server can attempt to steer the
agent through either one. Set `allowed_tools` to the names you actually expect,
and point agents only at servers you control or trust.
</Callout>

## Always attached

Two tools are added to every call and cannot be removed.

### `end_call`

Lets the agent hang up when the conversation is genuinely finished — the caller
says goodbye, the request is resolved, or the caller has gone silent despite
check-ins. It takes an optional `reason` for your logs and an `unresponsive`
flag for the silence case.

A server-side check refuses `end_call` on a call that is only seconds old or
where the caller has not spoken yet, so a model that reaches for it too early is
told to keep going. That check is not something the system prompt can talk its
way past.

### `transfer_to_human`

<Callout type="warn">
`transfer_to_human` does **not** connect a person to the live call. It notifies
your team, who call the caller back. The agent tells the caller someone will
ring them back, then wraps up. Write your prompt around a callback, not a warm
transfer.
</Callout>

It takes a short `reason` for the team and a 1–2 sentence `summary` written for
the colleague picking it up: what the caller wants, what has been covered, key
facts like an order number, and whether identity was checked. The result tells
the model what to say next, including what to say when the team could not be
reached.

Use it when the caller explicitly asks for a person, is upset and wants
escalation, or has a request the agent genuinely cannot handle.

## Errors

A tool failure is not a call failure. Whatever the tool raises is returned to
the model as a readable message, and the model apologises, retries, or takes
another route while the caller stays on the line. You see the failure in your
[usage and logs](/docs/usage-api), not as a dropped call.
