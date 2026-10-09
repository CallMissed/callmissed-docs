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
| CRM | `update_contact`, `set_contact_optin`, `add_call_note`, `create_follow_up_task`, `update_contact_fields`, `create_or_update_deal` |
| Scheduling | `calcom_list_slots`, `calcom_book`, `google_calendar_find_free_slots`, `google_calendar_create_event` |
| Spreadsheets | `google_sheets_find_rows`, `google_sheets_append_row`, `google_sheets_update_row`, `google_sheets_list_spreadsheets` |
| Commerce | `shopify_order_status`, `shopify_product_lookup`, `woocommerce_order_status` |
| WhatsApp messaging | `send_text_message`, `list_message_templates`, `send_template_message`, `send_quick_reply_buttons`, `send_list_menu`, `send_cta_url_button`, `send_location`, `send_document`, `request_contact_info` |
| Email | `send_email`, `gmail_send_email` |
| HTTP | `http_request` |

### Sending a WhatsApp template during a call

On phone and WhatsApp calls the WhatsApp messaging tools are on automatically
when your workspace has a connected WhatsApp number. A template is sent from a
number on the WhatsApp Business Account that owns it, so a workspace with several
accounts works too; only templates whose account has a connected number are
offered. To send an approved template, the agent:

1. calls `list_message_templates`, which returns each approved template with
   its `inputs` (every value the template needs: body placeholders, a text
   header placeholder as `header_<n>`, a link ending as `button_<n>`), each
   tagged with the `field` it is sent in (`input_1`, `input_2`, …);
2. asks the caller whether to send it to the number they are calling from or
   to a different number, and reads a different number back to confirm. A
   number given without a country code takes the caller's own (India only);
3. asks for any input it does not already know, and reads the details back;
4. calls `send_template_message` with `send_to` and each input as its field,
   for example `"input_1": "Rahul", "input_2": "12 Oct, 4 pm"` (up to
   `input_8`). If a value is missing the tool names the field, so the agent
   asks for it instead of the send failing.

The tool then waits a few seconds for WhatsApp's delivery report and tells the
agent whether the message was delivered. If the number cannot receive WhatsApp,
the agent offers to send it to another number. A temporary WhatsApp failure is
retried automatically (up to 3 attempts); a send that may already have gone out
is never sent twice, and the same message is not sent twice on one call unless
the caller asks. Every send, its recipient and its outcome appear on the call
in the console and through
[`GET /v1/voice/sessions/{id}/whatsapp_sends`](/docs/voice-sessions-api#whatsapp-messages).
A delivered paid template is billed like any other WhatsApp template.

Templates with an image, video or document header cannot be sent this way.

A tool whose integration is not connected — a Shopify store, a Cal.com key, a
Google account — does not break the call. It returns an error the model can
speak its way out of ("I can't reach the calendar right now"), and the
conversation continues.

The Google Sheets tools only reach the spreadsheets you choose with the Google
file picker in the console (Tools → Connected apps → Google Sheets → Choose
spreadsheets). Row 1 of each tab holds
the column names, and the tools address columns by those names.

### Order automation from Google Sheets

A spreadsheet chosen on the Google Sheets connection can also run on its own:
when a new order row appears, CallMissed sends an approved WhatsApp template
and/or has an agent call the customer to confirm, with separate settings for
cash-on-delivery and other orders, and writes the result into the sheet's
`CallMissed Status` and `CallMissed Note` columns (added if missing). Orders
already in the sheet when you switch it on are left alone, calls stay inside
the calling window you set and respect your do-not-call list, and unanswered
calls are retried.

The calling agent receives the order as call variables — `{{order_id}}`,
`{{name}}`, `{{payment_type}}`, `{{order_details}}` and one variable per column
(e.g. `{{total}}`) — so its prompt should mention `{{order_details}}`.
Set it up in the console (Tools → Connected apps → Google Sheets → Automate
orders) or with `POST /api/v1/sheet-automations`; `GET
/api/v1/sheet-automations/{id}/orders` lists what happened to each order.

An unknown name in `config.tools` is skipped and logged rather than taking the
agent offline, so a stale entry left behind by a renamed tool is not an outage.

### Messaging tools on calls

On **WhatsApp calls and phone calls**, the WhatsApp messaging tools above are
enabled automatically — you do not have to list them. The most common request on
a voice call is "send me that in the chat": a tracking link, an address, a
price. If the call is not linked to a connected WhatsApp sender, the tool
returns an error the model relays instead of sending anything.

### CRM writes during a call

Four CRM tools are for phone and WhatsApp calls. Add the ones you want to
`config.tools`; none is on by default.

| Tool | What it does |
| --- | --- |
| `add_call_note` | Saves a short note on the caller's contact, tagged with the call. Retrying the same note does not duplicate it. |
| `create_follow_up_task` | Creates a follow-up task for the caller. Give the due moment as an ISO 8601 date or date-time in `due`, or whole days from today in `due_in_days` (up to 366; a bare date means noon UTC). With no time, the task is created without a due date. |
| `update_contact_fields` | Saves `name`, `email`, `company_name`, `lifecycle_stage` (`subscriber`, `lead`, `qualified`, `opportunity`) and `lead_source` (only when none is recorded). An email already on another contact is refused. |
| `create_or_update_deal` | Creates a deal in your default (or a named) pipeline, or updates the caller's open deal with the same title. Amount is capped at 100,000,000. Stage and pipeline must already exist. |

What they will not do, whatever the caller says:

- They only touch the contact matched to the number on the call. If no contact
  matches, they return an error and nothing is created. Nothing takes an id,
  owner or assignee, and passing one is refused.
- `lifecycle_stage` cannot be set to `customer` or `churned`, and a contact
  already at one of those stages is left alone. A deal cannot be moved to a
  won or lost stage from a call.
- `create_or_update_deal` requires `caller_confirmed: true`. The tool
  description tells the model to read the deal back and wait for a yes first.

Set `crm_task_assignee_user_id` on the agent to a user in your workspace to
assign the tasks it creates; otherwise they are unassigned. The id is checked
on every call.

**Retention and privacy.** When the call runs under zero data retention, the
note tool refuses, the task keeps only a generic title with no details, the deal
keeps a generic title, and `update_contact_fields` accepts only
`lifecycle_stage` and `lead_source`. With `redact_pii` on, card, Aadhaar, PAN
and OTP digits are masked in the text these tools store.

### Not available on voice

Four categories are excluded from voice calls:

| Category | Why |
| --- | --- |
| `conversation` | Inbox-thread actions — notes, tags, status, escalation — belong to the chat channels |
| `personal_whatsapp` | Needs a linked personal-WhatsApp session, which a call does not have |
| `calling` | `request_call` and `request_phone_call` ask for a phone call from a chat; on a call, use `transfer_to_human` below for a callback |
| `callback` | `request_callback` arranges a call back from a WhatsApp chat. It is switched on by the agent's [`callback_calls`](/docs/bots#whatsapp-callback-calls) setting rather than `config.tools`, so it is not in the tool catalog |

Every tool in the first three categories comes back from `GET /api/v1/bots/tool-catalog`
with `"unavailable_on": ["voice"]`, and listing one in a calling agent's
`config.tools` returns an `incompatible` warning from
`POST /api/v1/bots/validate-config`.

For escalation on a call, use `transfer_to_human` below.

### Skills are not available on voice either

`config.skills` — the named bundles from `GET /api/v1/bots/skill-catalog` — is
resolved on chat channels only. A calling agent never reads the key, so a skill
set there adds neither its tools nor its instructions to the call. Put the tools
a call needs directly in `config.tools`.

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

It takes a short `reason` for the team and a 1–2 sentence `summary` written for
the colleague picking it up: what the caller wants, what has been covered, key
facts like an order number, and whether identity was checked. The result tells
the model what to say next, including what to say when the team could not be
reached.

Use it when the caller explicitly asks for a person, is upset and wants
escalation, or has a request the agent genuinely cannot handle.

**Live transfer on phone calls.** List up to 10 people in the agent's config as
`transfer_targets`, each `{name, phone, description}` with the phone number in
international format (`+919812345678`). On a phone call the tool then takes a
`target` — one of those names — and puts the caller through while the call is
live. The model only ever sees the names and descriptions, and can only reach a
number you listed.

- The caller hears hold music while the person's phone rings, for up to 30
  seconds. The music stops as soon as they answer, or when the attempt fails.
- `transfer_mode: "warm"` (the default) has the agent introduce the caller and
  the reason for the call, both listening, then leave. `"cold"` connects them
  straight away. `"cold_refer"` hands the call to your phone carrier instead
  (see below).
- A person can carry their own `mode` (`warm`, `cold` or `cold_refer`), which
  overrides `transfer_mode` for them only:
  `{"name": "Billing", "phone": "+919812345679", "description": "", "mode": "cold_refer"}`.
- If nobody answers, the caller is returned to the agent and a callback request
  is recorded for your team instead.
- The transferred leg is billed like any outbound call minute from your
  workspace's number. The agent's own session ends with `end_reason`
  `transferred`.
- On a call the agent placed, the transferred call still ends at the agent's
  maximum call duration.

**Handing the call to your carrier (`cold_refer`).** Instead of connecting the
person through the call, the call is handed to your phone carrier with a SIP
REFER: the carrier connects the caller to the person and the agent leaves the
call at once. It applies only on numbers you bring from your own carrier, and
only if that carrier accepts call transfers (some need it switched on for the
trunk). The onward call is then placed and billed by your carrier. On a number
you rent from us, or if the carrier refuses the transfer or the person does not
pick up, the person is connected through the call as with `"cold"` instead, so
choosing `cold_refer` never loses a transfer.

**Fallback when the agent fails.** Set `fallback_transfer_target` to the name of
one of your `transfer_targets`, and a phone call whose agent fails is put
through to that person instead of being hung up on. `fallback_on` picks which
failures count:

| Value | When |
| --- | --- |
| `agent_error` | The agent fails mid-call: a speech or language service stops working and cannot recover, or the agent stops unexpectedly. The default when `fallback_on` is unset |
| `stack_unavailable` | The agent's voice setup cannot be started for the call at all |

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/{bot_id}/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"values": {"fallback_transfer_target": "Sales", "fallback_on": ["agent_error", "stack_unavailable"]}}'
```

- The name must be one of the agent's `transfer_targets` when you save it, and
  the number is always the one saved on that person.
- The caller hears hold music while the phone rings. If nobody answers, a
  callback request is recorded for your team and the call ends.
- A person with `mode: "cold_refer"` is reached through your carrier as
  described above.
- A phone number can override both keys for calls to that number through its
  per-number config (`fallback_transfer_target`, `fallback_on`), for example to
  send failures on a night line to a different person.
- The transferred call is billed like any other live transfer, and the session
  ends with `end_reason` `transferred`.

<Callout type="warn">
Without `transfer_targets`, and on WhatsApp and browser calls, `transfer_to_human`
does **not** connect a person to the live call. It notifies your team, who call
the caller back, and the agent tells the caller someone will ring them back.
</Callout>

## Errors

A tool failure is not a call failure. Whatever the tool raises is returned to
the model as a readable message, and the model apologises, retries, or takes
another route while the caller stays on the line. You see the failure in your
[usage and logs](/docs/usage-api), not as a dropped call.
