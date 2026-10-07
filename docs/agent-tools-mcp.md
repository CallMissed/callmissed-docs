---
title: "Account MCP Server"
description: "Give an AI agent 333 tools that act on your CallMissed account: draft, build and configure voice agents, set up call flows, menus and queues, place and review calls, run outbound calling campaigns end to end, work the inbox and the support desk, search and update the CRM with its pipelines, custom fields and lead scoring, send WhatsApp and email, sell from a WhatsApp catalog, generate images, connect and disconnect WhatsApp, Facebook, Instagram, integrations and knowledge sources, manage webhooks and managed prompts, version and roll back agents, run evals and A/B experiments, and check credits, over the Model Context Protocol. Connect by signing in with your CallMissed account, or with an API key."
slug: "agent-tools-mcp"
breadcrumb: "Getting Started"
---

# Account MCP Server

Give an AI agent 333 tools that act on your CallMissed account: draft, build and configure voice agents, set up call flows, menus and queues, place and review calls, run outbound calling campaigns end to end, work the inbox and the support desk, search and update the CRM with its pipelines, custom fields and lead scoring, send WhatsApp and email, sell from a WhatsApp catalog, generate images, connect and disconnect WhatsApp, Facebook, Instagram, integrations and knowledge sources, manage webhooks and managed prompts, version and roll back agents, run evals and A/B experiments, and check credits, over the Model Context Protocol. Connect by signing in with your CallMissed account, or with an API key.

:::cards
/docs/mcp-server | Docs MCP Server | book | Searchable CallMissed docs inside your coding agent
/docs/quickstart | Quickstart | rocket | Make your first API call in under a minute
:::

## Overview

The **Account MCP server** lets an AI agent take actions in your CallMissed account through the [Model Context Protocol](https://modelcontextprotocol.io). Point any MCP client at one URL, sign in with your CallMissed account or pass an API key, and the agent gets 333 tools: draft, build and configure voice agents end to end, set up how inbound calls are answered with flows, menus and queues, place and inspect phone calls, read transcripts, build and run outbound calling campaigns, work the shared inbox and the support desk, search and update the CRM and set up its pipelines, custom fields and lead scoring, summarise and score calls, send WhatsApp messages, catalog products and Flow forms, send email, generate images, connect and disconnect WhatsApp, Facebook, Instagram, integrations and knowledge sources, wire up webhooks and managed prompts, version and roll back an agent, run evaluation suites and A/B experiments, and check what it all cost.

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

The consent screen offers five permission groups.

| Group | What it allows | On by default | Can spend credits |
| --- | --- | --- | --- |
| **Read your data** | See your calls, contacts, companies, deals, leads, quotes, invoices, products, tasks, notes, conversations, agents and usage, plus how your CRM and support desk are set up: pipelines, custom fields, saved views, lead scores, tickets, SLA policies, macros, routing rules and satisfaction scores. Cannot change anything. Searching your knowledge base costs a small amount to read the question. | Yes | Yes |
| **Read call transcripts** | Read what was said on your voice calls, and what each one cost. This also lets the connection use AI models on your account, which spends credits. | No | Yes |
| **Take actions** | Create and update records — including leads, products, quotes and invoices — send WhatsApp messages and email — including catalog products, order updates and Flow forms — place and end calls, rent phone numbers, and generate images. It can also build outbound calling campaigns and start them, which dials everyone on the list, change how your CRM and support desk are set up — pipelines, custom fields, lead scoring, tickets, SLA policies, macros and routing — and open satisfaction surveys. Spends credits, and a rented number is a recurring charge. | No | Yes |
| **Set up and change your agents** | Create agents and change their instructions, voice, tools and knowledge, draft a new agent from a description, give them custom tools that call your own URLs, change how a team of agents hands callers between them, and run test suites and A/B experiments against them. Cannot send messages or place calls. Indexing a knowledge source, drafting an agent and running a test suite spend credits. | No | Yes |
| **Configure your workspace** | Set up where your events are delivered and when you are alerted, connect an external service or a WhatsApp, Facebook or Instagram account by handing you a link to approve, choose which agent answers each connected number or account, disconnect them, create, edit or delete WhatsApp message templates, and manage the prompts the API resolves by name. Cannot read or write the credentials of a connected service. Publishing a prompt or pinning a model to one changes what your own API calls do and what they cost. | No | Yes |

**Read your data** is the only group ticked when the screen opens. The other four start off and are granted only if you tick them yourself, so a single click cannot hand an agent the ability to message a customer or spend credits.

Building an agent is its own tick rather than part of **Take actions**: letting an assistant send a WhatsApp message should not also let it rewrite the instructions, voice and tools of the agent answering your phone. Running an eval suite or an A/B experiment sits on that same tick, because it is work done on the agents it already governs.

At least one group has to be ticked. Approving nothing would create a connection that fails every call, so the consent screen asks you to choose something or press **Deny**.

Against the tool tables further down: **Read your data** covers every `:read` scope, including the campaign, CRM-setup, support-desk, commerce, prompt and eval reads; **Read call transcripts** grants the `stt`, `tts` and `llm` permissions the three voice-session tools need; **Take actions** covers the record, CRM-setup, support-desk, messaging, commerce, telephony and campaign `:write` scopes plus `whatsapp:send` and the `image` and `email` permissions; **Set up and change your agents** covers `bots:write`, `knowledge:write`, `squads:write`, `evals:write` and `experiments:write`; and **Configure your workspace** covers `webhooks:write`, `integrations:write`, `prompts:write` and `whatsapp:write`.

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
  Twenty-five tools spend credits and several of them reach real people. `start_campaign` reaches the most: it dials every person on the campaign's list. They are marked in the tables below. For an agent that should look but not act, tick only **Read your data** when you sign in, or use a key holding only the read scopes.
</Callout>

## Tools

333 tools in sixteen groups. `Access` is the tool's own `readOnlyHint`. `Credits` marks the tools that draw down your balance. Every tool's full JSON Schema, including argument names, types and bounds, comes back from `tools/list`.

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
| `search_available_numbers` | Search the numbers you could rent, with monthly price and whether each takes calls. Buys nothing | Read | `telephony:read` | No |
| `provision_phone_number` | Rent a number. It is bought immediately and renews monthly until released; needs an approved compliance application | Write | `telephony:write` | Yes |
| `attach_number_to_agent` | Point one of your numbers at an agent, so calls to it are answered by that agent. Pass `bot_id: null` to detach | Write | `telephony:write` | No |
| `list_voice_sessions` | List voice agent sessions, newest first | Read | `stt` + `tts` + `llm` permissions | No |
| `get_voice_session_transcript` | Get a session's full turn-by-turn transcript | Read | `stt` + `tts` + `llm` permissions | No |
| `get_voice_session_cost` | Break down what one voice session cost in credits | Read | `stt` + `tts` + `llm` permissions | No |

#### Call handling

How an inbound call is answered: the flow it walks, the menu it hears, the queue it waits in, the message left on a machine, and whether the number has been flagged as spam. This is the whole console setup journey, available to an agent.

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_call_flows` | List call flows with each one's shape and whether it is live. Returns no graphs | Read | `telephony:read` | No |
| `get_call_flow` | One flow with its full draft graph; the live snapshot is summarised, not repeated | Read | `telephony:read` | No |
| `create_call_flow` | Create a flow. It answers nothing until published and attached to a number | Write | `telephony:write` | No |
| `update_call_flow` | Change the draft. Sending `graph` replaces the whole graph | Write | `telephony:write` | No |
| `publish_call_flow` | Make the draft the version that answers **real** calls, from the next call on | Write, destructive | `telephony:write` | No |
| `attach_number_to_flow` | Answer one of your numbers with this flow. `detach: true` undoes it. A flow answers once it is published | Write | `telephony:write` | No |
| `simulate_call_flow` | Dry-run one step and see the instruction a real call would get. Webhook steps are not called | Read | `telephony:read` | No |
| `list_call_menus` | List press-1-for-sales menus with their prompt and key map | Read | `telephony:read` | No |
| `get_call_menu` | One menu with its prompt, key map and retry settings | Read | `telephony:read` | No |
| `create_call_menu` | Create a menu: what the caller hears and where each key sends them | Write | `telephony:write` | No |
| `update_call_menu` | Change a menu. Sending `options` replaces the whole key map | Write | `telephony:write` | No |
| `attach_number_to_menu` | Answer one of your numbers with this menu. `detach: true` undoes it. | Write | `telephony:write` | No |
| `list_call_queues` | List queues: how callers are held and in what order they are served | Read | `telephony:read` | No |
| `get_call_queue` | One queue with its serving order, hold settings and overflow behaviour | Read | `telephony:read` | No |
| `create_call_queue` | Create a queue that holds callers until a roster member is free | Write | `telephony:write` | No |
| `update_call_queue` | Change serving order, hold audio, waiting limit or overflow behaviour | Write | `telephony:write` | No |
| `list_queue_members` | Who answers for a queue, with priority, capacity and how busy each is | Read | `telephony:read` | No |
| `add_queue_member` | Put an agent (`bot_id`) or a person (`user_id`) on a queue's roster | Write | `telephony:write` | No |
| `remove_queue_member` | Take an answerer off the roster. They stop receiving calls immediately | Write, destructive | `telephony:write` | No |
| `attach_number_to_queue` | Send one of your numbers through this queue. `detach: true` undoes it | Write | `telephony:write` | No |
| `get_queue_live` | Who is holding right now, how long, and how much of the roster is free. Callers show as last four digits only | Read | `telephony:read` | No |
| `list_voicemail_templates` | The messages your calls leave on a machine, and whether each is ready to play | Read | `telephony:read` | No |
| `get_voicemail_template` | One voicemail message with its words, voice and state | Read | `telephony:read` | No |
| `create_voicemail_template` | Write the message a call should leave when a machine answers | Write | `telephony:write` | No |
| `update_voicemail_template` | Change a message. New words throw away the recording made from the old ones | Write | `telephony:write` | No |
| `get_number_reputation` | Whether one of your numbers is flagged as spam, and the complaints still open against it | Read | `telephony:read` | No |

Deleting a flow, menu, queue or voicemail message is not an MCP tool: an assistant should not be able to remove a routing object that live numbers still point at. Take one out of service reversibly instead — `enabled: false` on a menu or queue, or detach the number. Clearing a spam complaint needs a proof document and stays in the dashboard.

#### More voice and calls tools

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `get_voice_session` | Get one voice agent session's status, timing and post-call analysis (summary, outcome, sentiment) once it exists | Read | `stt` + `tts` + `llm` permissions | No |
| `update_number_call_settings` | Set a number's call settings | Write | `telephony:write` | No |
| `declare_number_for_ai_calls` | Record that a number has been declared to your telecom provider for automated/AI calls (India TRAI). Records a declaration already made; does not file one | Write | `telephony:write` | No |
| `list_telephony_compliance` | List this account's phone-number KYC applications and whether each is approved, pending or rejected (with the reason) | Read | `telephony:read` | No |
| `unpublish_call_flow` | Stop answering live calls with this flow; numbers on it answer normally again | Write, destructive | `telephony:write` | No |
| `update_queue_member` | Change a queue member's priority, concurrent-call limit, or enabled flag | Write | `telephony:write` | No |

### Calling campaigns

An outbound campaign is a list of people, a voice agent and the rules for when and how fast to dial. Build it as a draft, load the list, then start it. The account-wide do-not-call list lives here too.

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_campaigns` | List campaigns, newest first, with status and contact counts | Read | `campaigns:read` | No |
| `get_campaign` | Fetch one campaign: status, schedule, calling hours, dial mode | Read | `campaigns:read` | No |
| `create_campaign` | Create a campaign. It starts as a draft and calls nobody | Write | `campaigns:write` | No |
| `update_campaign` | Change name, schedule, calling number, calling hours, dial mode or DLT registration fields | Write | `campaigns:write` | No |
| `add_campaign_contacts` | Add up to 1000 people to the calling list. Adding calls nobody | Write | `campaigns:write` | No |
| `list_campaign_contacts` | The per-person result: waiting, calling, done or failed, with attempts and last error | Read | `campaigns:read` | No |
| `get_campaign_live_state` | What the campaign is doing right now, and in preview mode who is awaiting a go-ahead | Read | `campaigns:read` | No |
| `get_campaign_trai_readiness` | What stops the campaign meeting India's TRAI rules for AI calls, and whether the account enforces them | Read | `campaigns:read` | No |
| `start_campaign` | **Starts calling.** Real calls to every person on the list | Write, destructive | `campaigns:write` | Yes |
| `pause_campaign` | Stop placing new calls. Can be started again later | Write | `campaigns:write` | No |
| `stop_campaign` | Cancel the campaign for good. Cannot be started again | Write, destructive | `campaigns:write` | No |
| `list_do_not_call` | List the numbers suppressed from all calling on this account | Read | `campaigns:read` | No |
| `add_do_not_call` | Suppress numbers so no call is ever placed to them again | Write | `campaigns:write` | No |

Taking a number back **off** the do-not-call list is deliberately not a tool. Adding one is safe in the direction that matters — the worst case is a call that does not happen — but removing one un-does somebody's opt-out, and the next thing that happens is a call to a person who asked not to be called. That stays with a person in the [console](https://console.callmissed.com). The `DELETE /dnc/{entry_id}` endpoint is still there for your own code; see the [Campaigns API](/docs/voice-campaigns).

Uploading a CSV to a campaign is not a tool either: it is a file upload, and MCP arguments are JSON. Use `add_campaign_contacts` for a list the agent already holds, or the [Campaigns API](/docs/voice-campaigns) for a file.

### Conversations

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_conversations` | List conversations across channels with a preview and unread count | Read | `conversations:read` | No |
| `get_conversation_messages` | Read the messages in one conversation, oldest first | Read | `conversations:read` | No |
| `set_conversation_status` | Change a conversation's status, for example to close or escalate it | Write | `conversations:write` | No |
| `assign_conversation` | Give a conversation to a teammate by id or email, or unassign it | Write | `conversations:write` | No |
| `set_conversation_labels` | Replace a conversation's labels (short tags such as refund or vip) | Write | `conversations:write` | No |
| `add_conversation_note` | Write an internal note on the contact behind a conversation. Never sent to the customer | Write | `crm_notes:write` | No |
| `list_handoffs` | List conversations escalated to a person and still waiting | Read | `conversations:read` | No |
| `resolve_handoff` | Mark an escalated conversation handled, optionally keeping the AI quiet | Write | `conversations:write` | No |

#### More conversations tools

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `get_conversation` | Get one conversation's details: channel, status, contact, agent and unread count | Read | `conversations:read` | No |
| `mark_conversation_read` | Clear a conversation's unread count, as opening it in the inbox does | Write | `conversations:write` | No |
| `assign_handoff` | Assign a waiting handoff to a teammate, or pass assignee_id null to unassign it | Write | `conversations:write` | No |
| `callback_handoff` | For a voice handoff, phone your operator number and then the customer and bridge them (two real, billed calls) | Write | `conversations:write` | Yes |

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
| `list_companies` | List companies, optionally matching a name or domain | Read | `companies:read` | No |
| `create_company` | Add a company. A domain has to be unique on the account | Write | `companies:write` | No |
| `update_company` | Change fields on an existing company | Write | `companies:write` | No |

#### Pipelines and stages

The board a deal moves across. Pipelines and stages share the deal scopes, because a stage is only meaningful as part of the pipeline it belongs to.

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_pipelines` | List the sales pipelines, the default one first | Read | `crm_deals:read` | No |
| `create_pipeline` | Add a pipeline. It starts with no stages | Write | `crm_deals:write` | No |
| `update_pipeline` | Rename a pipeline, or make it the default | Write | `crm_deals:write` | No |
| `list_pipeline_stages` | List one pipeline's stages in board order | Read | `crm_deals:read` | No |
| `create_pipeline_stage` | Add a column, with its win probability and won/lost meaning | Write | `crm_deals:write` | No |
| `update_pipeline_stage` | Rename a stage, move it, or change what it means | Write | `crm_deals:write` | No |
| `reorder_pipeline_stages` | Rewrite the whole column order. Must name every stage exactly once | Write | `crm_deals:write` | No |

#### Custom fields

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_custom_fields` | List the custom fields defined on contacts, companies and deals, with each one's type | Read | `crm_custom_fields:read` | No |
| `create_custom_field` | Define a typed field: text, number, date, boolean or select | Write | `crm_custom_fields:write` | No |
| `update_custom_field` | Rename, reorder, re-option or (un)require a field. Its key and type never change | Write | `crm_custom_fields:write` | No |
| `list_custom_field_values` | Read every custom value on one record | Read | `crm_custom_fields:read` | No |
| `set_custom_field_value` | Set one field on one record. The value has to match the field's type | Write | `crm_custom_fields:write` | No |
| `clear_custom_field_value` | Unset one field on one record. The stored value is gone | Write, destructive | `crm_custom_fields:write` | No |

Deleting a custom field **definition** is not on this surface: the delete cascades to every value stored against it, on every record. Do that in the console.

#### Saved views

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_saved_views` | List the saved filtered lists you can see: shared ones plus your own | Read | `crm_views:read` | No |
| `create_saved_view` | Save a named filter, sort and column preset over a CRM list | Write | `crm_views:write` | No |

#### Lead scoring

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_lead_score_rules` | List the rules that award points, oldest first | Read | `crm_scores:read` | No |
| `create_lead_score_rule` | Add a rule over one of the allowed signals | Write | `crm_scores:write` | No |
| `update_lead_score_rule` | Change a rule's signal, comparison, points or enabled state | Write | `crm_scores:write` | No |
| `recompute_lead_scores` | Rescore up to 200 contacts or companies. All or nothing | Write | `crm_scores:write` | No |
| `list_lead_scores` | The leaderboard: scored records, highest first | Read | `crm_scores:read` | No |
| `get_lead_score` | One record's score with the breakdown of which rules fired | Read | `crm_scores:read` | No |

A new or changed rule does not move any score until `recompute_lead_scores` runs. The signals a rule may test are a closed list, documented on [Lead scoring](/docs/crm-lead-scores).

#### More cRM tools

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `get_contact` | Get one contact's full record by id | Read | `contacts:read` | No |
| `list_deals` | List deals, newest first, filtered by pipeline, stage, status, owner, contact or company | Read | `crm_deals:read` | No |
| `get_deal` | Get one deal by id | Read | `crm_deals:read` | No |
| `update_deal` | Change a deal's title, value, links, owner, close date, stage or open/won/lost status | Write | `crm_deals:write` | No |
| `list_tasks` | List CRM tasks, filtered by status, assignee, attached record, overdue or due window | Read | `crm_tasks:read` | No |
| `update_task` | Edit a task's title, description, due date, assignee, attached record or status | Write | `crm_tasks:write` | No |
| `list_crm_notes` | List the notes on one contact, company or deal, newest first | Read | `crm_notes:read` | No |
| `update_crm_note` | Replace the text of a note | Write | `crm_notes:write` | No |
| `list_crm_duplicates` | Find groups of contacts or companies that look like the same record | Read | `crm_search:read` | No |
| `merge_crm_duplicates` | Irreversibly fold duplicates into a primary record: their linked records move to it and the duplicates are deleted | Write, destructive | `crm_search:write` | No |
| `update_saved_view` | Rename a saved view or change its filters, sort, columns, layout or sharing; the list it belongs to cannot change | Write | `crm_views:write` | No |
| `get_company` | Get one company's record by id | Read | `companies:read` | No |

### Support desk

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_tickets` | List tickets by status, priority, assignee, contact, conversation or subject | Read | `support_tickets:read` | No |
| `get_ticket` | Fetch one ticket with its lifecycle timestamps | Read | `support_tickets:read` | No |
| `create_ticket` | Open a ticket, optionally linked to a conversation or contact | Write | `support_tickets:write` | No |
| `update_ticket` | Change subject, description, priority, links or tags | Write | `support_tickets:write` | No |
| `set_ticket_status` | Move a ticket through its lifecycle, keeping the SLA clocks honest | Write | `support_tickets:write` | No |
| `assign_ticket` | Hand a ticket to a teammate, or send it back to the queue | Write | `support_tickets:write` | No |

Deleting a ticket is not on this surface — it would destroy the record of a customer interaction and every SLA number computed from it. Close it instead.

#### SLA

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_sla_policies` | List the first-response and resolution targets the desk promises | Read | `sla:read` | No |
| `create_sla_policy` | Promise a first-response time, a resolution time, or both, with business hours | Write | `sla:write` | No |
| `update_sla_policy` | Change a policy's targets, priority, hours or enabled state | Write | `sla:write` | No |
| `get_ticket_sla_status` | Where one ticket stands: due times, what is missed, minutes left | Read | `sla:read` | No |
| `list_sla_breaches` | Tickets already past a deadline, oldest first | Read | `sla:read` | No |

#### Macros, tags and routing

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_support_macros` | List canned replies, most-used first | Read | `support_ops:read` | No |
| `create_support_macro` | Save a canned reply, with `{{placeholders}}` and optional actions | Write | `support_ops:write` | No |
| `update_support_macro` | Change a macro's text, actions, category or enabled state | Write | `support_ops:write` | No |
| `use_support_macro` | Fill a macro's placeholders and get the finished text. Sends nothing | Write | `support_ops:write` | No |
| `list_support_tags` | List the tag vocabulary with colours and descriptions | Read | `support_ops:read` | No |
| `create_support_tag` | Add a tag to the vocabulary | Write | `support_ops:write` | No |
| `list_ticket_routing_rules` | List the triage rules in evaluation order | Read | `support_ops:read` | No |
| `create_ticket_routing_rule` | Add a triage rule that assigns, prioritises and tags a match | Write | `support_ops:write` | No |
| `update_ticket_routing_rule` | Change a rule's conditions, actions, position or enabled state | Write | `support_ops:write` | No |
| `reorder_ticket_routing_rules` | Rewrite the evaluation order. Must name every rule exactly once | Write | `support_ops:write` | No |
| `preview_ticket_routing` | Dry run: which rule would match a ticket, and what it would do. Writes nothing | Read | `support_ops:read` | No |

Macros and rules are retired with `is_active: false` rather than deleted, so nothing that a past ticket refers to disappears.

#### Satisfaction

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `create_csat_survey` | Mint a survey and its public response link. **Does not send it** | Write | `csat:write` | No |
| `list_csat_surveys` | List surveys, optionally only answered or only still-open ones | Read | `csat:read` | No |
| `get_csat_stats` | Response rate, average rating and NPS over a window of up to a year | Read | `csat:read` | No |

`create_csat_survey` returns a `token`; the link you send is that token on the public response page. Delivering it over WhatsApp or email is a separate call, and that send is what costs credits.

#### More support desk tools

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `update_support_tag` | Rename a support tag or change its color or description | Write | `support_ops:write` | No |

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
| `send_whatsapp_buttons` | Send a message with up to three tap-to-reply buttons, inside the 24-hour window | Write | `whatsapp:send` | Yes |
| `send_whatsapp_media` | Send an image, video, audio clip, document or sticker from a public https link or an uploaded media id, inside the 24-hour window | Write | `whatsapp:send` | Yes |
| `list_whatsapp_templates` | Your templates with review status in plain words, rejection reason and quality | Read | `whatsapp:read` | No |
| `review_whatsapp_template` | WhatsApp's template rules, the practices that keep your account from being restricted (opt-in, opt-outs, the 24-hour window, quality rating), and a scored review of a draft: errors, warnings and tips. Call it before creating or editing a template | Read | `whatsapp:read` | No |
| `create_whatsapp_template` | Create a template (or copy a library template by name) and submit it for review. A draft that breaks WhatsApp's template rules is refused with the list of problems; remaining warnings come back as `policy_warnings` | Write | `whatsapp:write` | No |
| `edit_whatsapp_template` | Replace an approved, rejected or paused template's content and resubmit it, with the same rules check as create | Write | `whatsapp:write` | No |
| `delete_whatsapp_template` | **Permanently** delete a template in every language | Write, destructive | `whatsapp:write` | No |
| `get_whatsapp_business_verification` | Whether the business behind your WhatsApp account is verified | Read | `whatsapp:read` | No |
| `list_whatsapp_campaigns` | List broadcast campaigns with status and recipient counts | Read | `whatsapp:read` | No |
| `send_email` | Send an email from a domain this account has verified | Write | `email` permission | Yes |
| `send_whatsapp_product` | Send one product from a WhatsApp catalog so the recipient can add it to a cart in the chat | Write | `wa_commerce:write` | Yes |
| `send_whatsapp_product_list` | Send up to 30 products grouped into titled sections, as one browsable message | Write | `wa_commerce:write` | Yes |
| `send_whatsapp_catalog` | Send the whole catalog attached to a business number, fronted by one product as the cover | Write | `wa_commerce:write` | Yes |
| `list_whatsapp_orders` | List the carts customers sent from your catalog, with status, currency and subtotal | Read | `wa_commerce:read` | No |
| `get_whatsapp_order` | One cart order with every line item: SKU, quantity and unit price | Read | `wa_commerce:read` | No |
| `update_whatsapp_order_status` | Advance a stored order and tell the customer in the same step. `completed` and `canceled` are final | Write, destructive | `wa_commerce:write` | Yes |
| `list_whatsapp_flows` | List your WhatsApp Flows with status and categories. The design document is left out | Read | `wa_flows:read` | No |
| `send_whatsapp_flow` | Send a Flow — a form filled in without leaving the chat — and get the token that identifies the reply | Write | `wa_flows:write` | Yes |
| `list_whatsapp_flow_responses` | The Flow forms customers submitted, with the answers they gave | Read | `wa_flows:read` | No |
| `publish_whatsapp_flow` | Publish a draft Flow. One-way: the design is then frozen and changing it means a new Flow | Write, destructive | `wa_flows:write` | No |

#### More messaging tools

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_whatsapp_numbers` | List your connected WhatsApp Business numbers with their status, quality rating and linked agent | Read | `whatsapp:read` | No |
| `refresh_whatsapp_number` | Re-read a number's status, name and quality rating from WhatsApp | Write | `whatsapp:write` | No |
| `link_whatsapp_number_to_agent` | Choose which agent auto-replies on a WhatsApp number; null unlinks it | Write | `whatsapp:write` | No |
| `set_whatsapp_autoreply` | Turn the linked agent's automatic replies on or off for one number | Write | `whatsapp:write` | No |
| `get_whatsapp_business_profile` | Read the public business profile customers see on a number | Read | `whatsapp:read` | No |
| `update_whatsapp_business_profile` | Change the text fields of a number's public business profile | Write | `whatsapp:write` | No |
| `search_whatsapp_template_library` | Search WhatsApp's pre-written utility templates, which can be created by name with create_whatsapp_template | Read | `whatsapp:read` | No |
| `sync_whatsapp_templates` | Pull the latest templates and review statuses from WhatsApp | Write | `whatsapp:write` | No |
| `draft_whatsapp_template` | Write a template draft from a plain-language intent, with notes on review-rejection risks | Write | `whatsapp:write` | Yes |
| `create_whatsapp_campaign` | Create a broadcast campaign draft for an approved template | Write | `whatsapp:write` | No |
| `get_whatsapp_campaign` | Read one campaign with its status and delivery counts | Read | `whatsapp:read` | No |
| `list_whatsapp_campaign_recipients` | Page through every recipient of a campaign with its delivery status and WhatsApp's timestamp for each status, for auditing who got it | Read | `whatsapp:read` | No |
| `add_whatsapp_campaign_recipients` | Add recipients, each with optional template variables, to a campaign that has not launched | Write | `whatsapp:write` | No |
| `launch_whatsapp_campaign` | Start sending a campaign to all its recipients | Write | `whatsapp:write` | Yes |
| `cancel_whatsapp_campaign` | Stop a campaign; messages already sent are not recalled | Write, destructive | `whatsapp:write` | No |
| `get_whatsapp_analytics` | Read WhatsApp messaging analytics: the delivery funnel, a daily time series, or credits spent | Read | `whatsapp:read` | No |
| `list_whatsapp_webhook_events` | List the most recent events WhatsApp delivered to your numbers, for debugging | Read | `whatsapp:read` | No |
| `get_whatsapp_message_status` | Look up one message by its `wamid` to prove whether it was delivered: its furthest status, WhatsApp's timestamp for each status, the failure error and every status recorded | Read | `whatsapp:read` | No |
| `list_whatsapp_calls` | List WhatsApp voice calls, newest first | Read | `whatsapp:read` | No |
| `get_whatsapp_call_settings` | Read whether calling is on for a number, its call hours and callback settings | Read | `whatsapp:read` | No |
| `update_whatsapp_call_settings` | Turn calling on or off for a number and set its call hours, call icon and callback permission | Write | `whatsapp:write` | No |
| `request_whatsapp_call_permission` | Send a customer WhatsApp's call-permission prompt; you can only call them after they accept | Write | `whatsapp:send` | No |
| `place_whatsapp_call` | Call a customer on WhatsApp; the number's linked agent does the talking | Write | `whatsapp:send` | Yes |
| `end_whatsapp_call` | Hang up a live WhatsApp call | Write, destructive | `whatsapp:send` | No |

### Images

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `generate_image` | Generate an image from a prompt. `model` picks which image model draws it — its enum lists each model's strength. Returns the image itself, so the model can see it, plus a link to the full-size version | Write | `image` permission | Yes |
| `list_generated_images` | List images made with this key, with fresh short-lived links | Read | `image` permission | No |

### Agents and knowledge

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_agents` | List the voice and chat agents on the account, to find a `bot_id` | Read | `bots:read` | No |
| `get_agent` | Fetch one agent's configuration by id | Read | `bots:read` | No |
| `craft_agent_prompt` | Everything needed to write a strong voice-agent prompt — the four-section structure and limits, the rules that make a prompt work on a phone call, the agent's current prompt, enabled tools and declared `{{variables}}` — plus a scored review of your draft. Free, writes nothing | Read | `bots:read` | No |
| `get_agent_prompt` | An agent's instructions split into the four boxes its owner edits: objective, response guidelines, conversation script and first message | Read | `bots:read` | No |
| `list_voice_tiers` | The voice tiers a calling agent can run on: the `voice_tier` id, what it sounds like, what a minute costs in credits, and the voices you may pick from. Read before creating or reconfiguring a calling agent | Read | `bots:read` | No |
| `get_agent_config_schema` | Every configuration key an agent accepts, with allowed values, defaults, what happens when a key is omitted, and a minimal working example | Read | `bots:read` | No |
| `validate_agent_config` | Check a config you assembled and get one finding per problem. Writes nothing | Read | `bots:read` | No |
| `list_agent_tool_catalog` | The agent-tool names that go in the config's `tools` list, with what each does and the channels it cannot run on | Read | `bots:read` | No |
| `list_agent_skills` | The skill names that go in the config's `skills` list, with the tools each bundles and a preview of the instructions it injects. Chat channels only | Read | `bots:read` | No |
| `create_agent` | Create a voice or WhatsApp agent. Active immediately | Write | `bots:write` | No |
| `update_agent` | Rename an agent, or replace its instructions with one flat block | Write | `bots:write` | No |
| `update_agent_prompt` | Change the instructions section by section, so the objective / guidelines / script structure survives the write | Write | `bots:write` | No |
| `update_agent_config` | Change configuration keys in place — set some, remove others, leave the rest alone | Write | `bots:write` | No |
| `toggle_agent` | Flip an agent between active and paused | Write | `bots:write` | No |
| `add_agent_knowledge` | Add a document to one agent's knowledge base | Write | `knowledge:write` | No |
| `knowledge_search` | Search the knowledge base and get the passages that match | Read | `knowledge:read` | No |
| `add_agent_memory` | Store a durable fact for one agent, applied on every future conversation | Write | `bots:write` | No |
| `list_agent_custom_tools` | List the custom HTTP tools this account has defined. Stored credentials are never returned | Read | `bots:read` | No |
| `create_agent_custom_tool` | Define an HTTP endpoint an agent may call mid-conversation | Write | `bots:write` | No |
| `update_agent_custom_tool` | Change a custom HTTP tool. Omitted fields, including secrets, are kept | Write | `bots:write` | No |
| `test_agent_custom_tool` | Run a saved custom tool once. This **really calls** the endpoint with its stored credentials | Write | `bots:write` | No |
| `delete_agent_custom_tool` | **Permanently** delete a custom tool and its stored credentials, for every agent that uses it | Write, destructive | `bots:write` | No |
| `set_agent_voicemail_drop` | Make a calling agent leave a saved voicemail message when a machine answers | Write | `bots:write` | No |
| `draft_agent_from_description` | Describe the agent you want and get a complete, reviewable draft back. Creates nothing | Write | `squads:write` | Yes |

#### Building an agent with these tools

`create_agent` is not the first call. An agent's `config` is a free-form object whose keys are mostly not checked when written — a wrong value is stored and then fails on the call, usually without an error. So the order is:

1. `get_agent_config_schema` — the keys this agent type accepts, and a `minimal_example` that works. `draft_agent_from_description` is the shortcut past the blank page: describe the agent and review what comes back. It creates nothing, so whatever it proposes still goes through the steps below.
2. `validate_agent_config` — fix every `error` finding before writing anything.
3. `create_agent`, then `add_agent_knowledge`.
4. For a voice agent: `search_available_numbers` → `provision_phone_number` → `attach_number_to_agent`, then `place_call` and `get_voice_session_transcript` to hear the result.
5. `update_agent_config` to change one key later; it merges, so it cannot wipe the rest of the configuration.
6. `craft_agent_prompt` → `update_agent_prompt` to write or improve the instructions. `craft_agent_prompt` returns the structure, the phone-call rules and the agent's real tools and variables; write the four sections, pass them back as `draft` until the review has no `error` findings, then save. An agent's prompt is edited in four boxes — objective, response guidelines, conversation script and first message — and `update_agent`'s flat `system_prompt` collapses them into one, silently. The pair above is what keeps them.

The same flow over plain HTTP, with curl for every step and the five ways a config silently misbehaves, is on [Build an Agent](/docs/build-an-agent).

#### More agents and knowledge tools

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_knowledge_sources` | List the documents, URLs and facts in this account's knowledge base, optionally for one agent, with their ingest status | Read | `knowledge:read` | No |
| `add_agent_knowledge_url` | Fetch a public web page and add its text to one agent's knowledge base | Write | `knowledge:write` | Yes |
| `list_agent_memories` | List the durable facts one agent has been taught, newest first | Read | `bots:read` | No |
| `list_agent_squads` | List squads: groups of agents that hand a caller to one another by role | Read | `squads:read` | No |
| `get_agent_squad` | Fetch a squad with its handoff policy and full member roster | Read | `squads:read` | No |
| `create_agent_squad` | Create a squad whose entry agent answers the call; the entry agent is enrolled as its first member | Write | `squads:write` | No |
| `update_agent_squad` | Change a squad's name, description, entry agent (must already be a member), handoff policy, or active flag | Write | `squads:write` | No |
| `add_squad_member` | Put an agent on a squad's roster under a role; its description is what routing matches callers against | Write | `squads:write` | No |
| `update_squad_member` | Change a squad member's role, description or position | Write | `squads:write` | No |
| `remove_squad_member` | Take a member off a squad's roster; the entry agent cannot be removed | Write, destructive | `squads:write` | No |
| `simulate_squad_handoff` | Dry-run a squad's routing: given what a caller said, which member would take the call and why | Read | `squads:read` | No |

### Usage and credits

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `get_credit_balance` | Check how many credits are left on the account | Read | Any valid key | No |
| `get_usage_summary` | Summarise recent spend in US$, broken down by service, model and day, with prompt-cache reads and writes counted separately | Read | `usage:read` | No |

### Platform configuration

Webhooks, integrations, managed prompts and agent versions. Every webhook tool
— including the two that only read — needs `webhooks:write`, because a delivery
row carries customer conversation data and there is no `webhooks:read` scope to
grant.

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_webhooks` | List webhook subscriptions with the events each receives. Signing secrets come back masked | Read | `webhooks:write` | No |
| `create_webhook` | Subscribe an HTTPS endpoint to account events. Returns the signing secret in full, once | Write | `webhooks:write` | No |
| `update_webhook` | Change a subscription's URL, events, agent scope or active flag. `is_active: false` pauses it | Write | `webhooks:write` | No |
| `test_webhook` | Send a real signed POST to the configured endpoint and report its status code and latency | Write | `webhooks:write` | No |
| `list_webhook_deliveries` | Delivery attempts for one webhook, with status code, attempts and error | Read | `webhooks:write` | No |
| `get_webhook_delivery` | One attempt in full, including the JSON payload that was sent | Read | `webhooks:write` | No |
| `replay_webhook_delivery` | Re-send a past payload to the current endpoint. Your server really receives it again | Write, destructive | `webhooks:write` | No |
| `list_integrations` | The external services connected to the account. Stored credentials are never returned | Read | `integrations:read` | No |
| `list_integration_catalog` | What can be connected, annotated with what already is | Read | `integrations:read` | No |
| `start_integration_oauth` | Return a console link that connects a sign-in (OAuth) service to the workspace of the person who opens it | Write | `integrations:write` | No |
| `list_prompts` | The managed prompts on the account, with each one's current version | Read | `prompts:read` | No |
| `create_prompt` | Create a managed prompt. Starts empty — add the text as a version | Write | `prompts:write` | No |
| `list_prompt_versions` | The revision log: version, label, pinned model, declared variables. Template text omitted | Read | `prompts:read` | No |
| `get_prompt_version` | One version in full, with its template, variables and pinned model and parameters | Read | `prompts:read` | No |
| `create_prompt_version` | Append a revision. Nothing live changes until a label points at it | Write | `prompts:write` | No |
| `set_prompt_label` | Point a label such as `production` at a version. This is how a prompt is published | Write, destructive | `prompts:write` | No |
| `render_prompt` | Substitute variables and get the text back, plus any you did not supply. No model call | Read | `prompts:read` | No |
| `list_prompt_presets` | Saved presets: a named model plus pinned parameters | Read | `prompts:read` | No |
| `create_prompt_preset` | Save a model and parameters under a name, optionally bound to a prompt version | Write | `prompts:write` | No |
| `list_agent_versions` | An agent's version history with label, status and commit message. Bodies omitted | Read | `bots:read` | No |
| `get_agent_version` | One version in full. Stored channel credentials are redacted | Read | `bots:read` | No |
| `create_agent_version` | Stage a draft from what is live now. Callers are unaffected until it is published | Write | `bots:write` | No |
| `update_agent_version` | Edit the staged draft. Published versions are immutable | Write | `bots:write` | No |
| `publish_agent_version` | Publish the draft onto the live agent. The next caller hears the new behaviour | Write, destructive | `bots:write` | No |
| `rollback_agent_version` | Put an older version back on the live agent. Appends rather than rewriting history | Write, destructive | `bots:write` | No |

`test_webhook` and `replay_webhook_delivery` both make a real outbound HTTP
request to **your own** endpoint, so whatever your server does on that event
happens for real — and a replay makes it happen a second time. Neither takes a
URL: they use the one already stored on the subscription, which is re-checked
against private and internal address ranges on every attempt.

`start_integration_oauth` hands back a console link for the account owner to
open. They sign in to CallMissed and approve access there, and the account is
connected to their own workspace; nothing is connected until they do. Confirm it with
`get_integration_status`, and disconnect with `disconnect_integration` (see
**Connected platforms**). There is no tool that writes a provider credential.

#### More platform configuration tools

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `update_prompt` | Change a prompt's name or description | Write | `prompts:write` | No |
| `update_prompt_preset` | Change a preset's name, model, default params or pinned prompt version | Write | `prompts:write` | No |
| `set_integration_spreadsheets` | Replace the list of spreadsheets agents may use through a Google Sheets connection | Write | `integrations:write` | No |
| `list_usage_logs` | Row-level log of this account's API calls, newest first: service, model, status, latency, token counts (prompt-cache reads and writes separate) and cost in US$ per call | Read | `usage:read` | No |
| `list_models` | The public model catalogue with pricing, context window and feature flags | Read | None | No |
| `get_model` | Pricing, context window and capabilities of one model id from list_models | Read | None | No |
| `get_platform_status` | Current health of each CallMissed API service, as shown on the public status page | Read | None | No |
| `get_task` | Fetch one task in full | Read | `crm_tasks:read` | No |

### Testing and quality

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_eval_suites` | Evaluation suites on the account, with the agent each tests and its scorecard | Read | `evals:read` | No |
| `create_eval_suite` | Create a suite of synthetic conversations for one agent | Write | `evals:write` | No |
| `list_eval_cases` | The cases in a suite: persona, opening line, turn budget, success criteria | Read | `evals:read` | No |
| `create_eval_case` | Add one synthetic conversation to a suite | Write | `evals:write` | No |
| `create_eval_case_from_session` | Turn a real voice session into a test case: its opening, the caller's goal, and suggested outcome checks. The key also needs the stt, tts and llm permissions | Write | `evals:write` | No |
| `run_eval_suite` | Run every case against the agent and report what passed. Takes minutes on a big suite | Write | `evals:write` | Yes |
| `list_eval_runs` | Run history with pass counts, model and credits spent | Read | `evals:read` | No |
| `get_eval_run` | One run in full, including every synthetic transcript | Read | `evals:read` | No |
| `list_experiments` | A/B experiments with status, deciding metric and traffic split | Read | `experiments:read` | No |
| `create_experiment` | Create an experiment on one agent. Starts as a draft taking no traffic | Write | `experiments:write` | No |
| `create_experiment_arm` | Add a configuration to compare: an agent version plus optional overrides | Write | `experiments:write` | No |
| `start_experiment` | Put it live. Real callers are split across the arms from this moment | Write, destructive | `experiments:write` | No |
| `stop_experiment` | Stop splitting traffic. Everything already collected is kept | Write | `experiments:write` | No |
| `conclude_experiment` | Record the winner and close it. Final — it cannot be restarted or changed | Write, destructive | `experiments:write` | No |
| `get_experiment_results` | Per-arm sample sizes and the deciding metric, with a plain-language verdict | Read | `experiments:read` | No |
| `list_voice_alerts` | Metric alerts with their thresholds, windows and whether each is firing | Read | `webhooks:write` | No |
| `create_voice_alert` | Be notified when a call metric crosses a threshold over a rolling window | Write | `webhooks:write` | No |
| `update_voice_alert` | Change a threshold, window, channel or cooldown, or silence an alert | Write | `webhooks:write` | No |

`run_eval_suite` is the one tool here that spends: each case is a real model
conversation of up to its turn budget, and a suite with a scorecard adds a
grading call per case. No real caller is ever contacted — the caller is
synthetic. The balance is checked before anything runs, so a refusal costs
nothing.

Metric alerts use `webhooks:write` rather than a scope of their own. An alert is
a notification subscription that sends mail and fires webhooks, which is exactly
what that scope already governs.

#### More testing and quality tools

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `update_eval_suite` | Rename a suite, change its description or grading scorecard, or turn it on or off | Write | `evals:write` | No |
| `update_eval_case` | Change one case's name, persona, opening line, turn budget, success criteria or position | Write | `evals:write` | No |
| `get_experiment` | Fetch an experiment with its status, metric, traffic split and arms | Read | `experiments:read` | No |
| `update_experiment` | Change an experiment's name, hypothesis, metric (only while a draft) or traffic split | Write | `experiments:write` | No |
| `update_experiment_arm` | Change an arm's name, agent version, overrides or control flag | Write | `experiments:write` | No |
| `list_scorecards` | List the rubrics used to grade calls and eval runs | Read | `conversations:read` | No |
| `create_scorecard` | Create a grading rubric: up to 20 criteria, each a unique label, a relative weight (0-100] and optional guidance | Write | `conversations:write` | No |
| `update_scorecard` | Replace a rubric's name and full criteria list; scores already taken keep their old breakdown | Write | `conversations:write` | No |

### Connected platforms

Check, connect and disconnect the channels and services the account runs on, without opening the console. Listing and re-pointing WhatsApp numbers, Facebook Pages and Instagram accounts is in the Messaging and Facebook and Instagram groups; adding knowledge is in Agents and knowledge.

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `get_integration_status` | Whether an integration is connected, and to which account. Call it after the person approves the `start_integration_oauth` link | Read | `integrations:read` | No |
| `disconnect_integration` | Disconnect an integration and discard its stored credentials. Reconnect with `start_integration_oauth` | Write, destructive | `integrations:write` | No |
| `start_channel_connection` | The link that connects a WhatsApp Business number, Facebook Page, Instagram account or personal WhatsApp number, with the steps to follow | Read | `whatsapp:write` | No |
| `list_whatsapp_accounts` | Connected WhatsApp Business accounts | Read | `whatsapp:read` | No |
| `get_whatsapp_number` | One WhatsApp number's live status | Read | `whatsapp:read` | No |
| `disconnect_whatsapp_number` | Deregister a WhatsApp number from this workspace. History is kept | Write, destructive | `whatsapp:write` | No |
| `get_knowledge_source` | One knowledge source's status, including the error if indexing failed | Read | `knowledge:read` | No |
| `remove_knowledge_source` | Remove a source from an agent's knowledge | Write, destructive | `knowledge:write` | No |
| `list_payment_requests` | Payment links your agents sent to customers during calls and chats, with amount, amount paid and status. The money goes to your own connected Razorpay account | Read | `integrations:read` | No |

Two steps cannot happen inside a chat: Meta's own sign-in window for WhatsApp Business, Facebook and Instagram, and the QR scan that links a personal WhatsApp number. `start_channel_connection` returns the one page that runs that step, and `list_whatsapp_numbers`, `list_facebook_pages` or `list_instagram_accounts` confirms it worked. No tool accepts an access token or provider key.

The disconnect and remove tools are marked destructive, so your client asks before running them. None of them deletes conversation history.

### Facebook and Instagram

Facebook Pages and Instagram accounts: publish and schedule posts, answer comments and DMs, read analytics. Posting, comments and DMs are not billed.

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_facebook_pages` | List your connected Facebook Pages with their ids and linked agent | Read | `whatsapp:read` | No |
| `list_instagram_accounts` | List your connected Instagram professional accounts with their ids and linked agent | Read | `whatsapp:read` | No |
| `set_social_account_agent` | Choose which agent auto-replies to DMs on a Facebook Page or Instagram account; null unlinks it | Write | `whatsapp:write` | No |
| `publish_facebook_post` | Publish to a Facebook Page: text, a link, photos by URL, or a Reel from a video URL | Write | `whatsapp:send` | No |
| `publish_instagram_post` | Publish an image, video, Reel, story or carousel from public media URLs | Write | `whatsapp:send` | No |
| `get_instagram_container_status` | Check whether Instagram has finished processing a post's media | Read | `whatsapp:send` | No |
| `publish_instagram_container` | Publish a processed media container once its status is FINISHED | Write | `whatsapp:send` | No |
| `get_instagram_publishing_limit` | How many posts the account has published in the last 24 hours against Instagram's cap | Read | `whatsapp:send` | No |
| `generate_social_post` | Draft a ready-to-publish post: a caption, hashtags and, unless generate_image is false, an image link | Write | None | Yes |
| `list_social_comments` | List comments on a Facebook post or Instagram media | Read | `whatsapp:read` | No |
| `list_social_comment_replies` | List the replies under one comment | Read | `whatsapp:read` | No |
| `reply_to_social_comment` | Post a public reply to a Facebook or Instagram comment as the Page or account | Write | `whatsapp:send` | No |
| `hide_social_comment` | Hide a comment from everyone but its author, or unhide it | Write | `whatsapp:send` | No |
| `list_social_conversations` | List Facebook Messenger or Instagram DM threads, most recent first | Read | `whatsapp:read` | No |
| `get_social_conversation_messages` | Read the messages in one Facebook or Instagram DM thread | Read | `whatsapp:read` | No |
| `send_social_message` | Send a text reply into an existing Facebook Messenger or Instagram DM thread; Meta only allows it within its reply window | Write | `whatsapp:send` | No |
| `get_social_analytics` | Read Facebook or Instagram DM analytics: the conversation funnel or a daily time series | Read | `whatsapp:read` | No |

### Email

Domains, templates, suppressions, inbound addresses and send history for the Email API. Every tool needs a key with the `email` permission.

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `list_email_domains` | List your sending domains and whether each is verified | Read | `email` permission | No |
| `add_email_domain` | Add a sending domain | Write | `email` permission | No |
| `get_email_domain_records` | List the DNS records a domain needs and whether each is found | Read | `email` permission | No |
| `verify_email_domain` | Check a domain's DNS records now and mark it verified if they pass | Write | `email` permission | No |
| `list_email_sends` | List recent sends with their delivery status | Read | `email` permission | No |
| `get_email_usage` | Read email send volume and spend for the last 30 days | Read | `email` permission | No |
| `list_email_suppressions` | List addresses that will not be emailed (bounces, complaints, manual) | Read | `email` permission | No |
| `add_email_suppression` | Stop all future email to an address | Write | `email` permission | No |
| `list_inbound_email_addresses` | List the addresses that receive email on your verified domains | Read | `email` permission | No |
| `create_inbound_email_address` | Start receiving email at an address on a verified domain, optionally forwarding each message to a webhook URL | Write | `email` permission | No |
| `list_inbound_emails` | List recently received emails, newest first | Read | `email` permission | No |
| `list_email_templates` | List your saved email templates | Read | `email` permission | No |
| `create_email_template` | Save a reusable email template | Write | `email` permission | No |
| `update_email_template` | Change an email template; only the fields given change | Write | `email` permission | No |
| `get_scheduled_email` | Look up scheduled sends by the send id or batchId a send returned | Read | `email` permission | No |
| `cancel_scheduled_email` | Cancel every still-pending send for a send id or batchId | Write, destructive | `email` permission | No |

### Guides and playbooks

Step-by-step playbooks an assistant follows before it builds anything: it interviews you about the use case first. The same playbooks are served as MCP prompts (`build_voice_agent`, `build_whatsapp_agent`, `launch_campaign`, `setup_support_desk`, `setup_crm`).

| Tool | What it does | Access | Scope or permission | Credits |
| --- | --- | --- | --- | --- |
| `get_playbook` | Get a step-by-step playbook | Read | None | No |

### Calling tools are deployment-gated

33 of the 333 tools are gated on whether calling is switched on for your account, in two independent groups.

**20 telephony tools:** `place_call`, `list_calls`, `get_call`, `end_call`, `get_call_recording`, `click_to_call`, `list_phone_numbers`, `search_available_numbers`, `provision_phone_number`, `attach_number_to_agent`, the four voicemail-message tools, `set_agent_voicemail_drop`, `get_number_reputation`, `update_number_call_settings`, `declare_number_for_ai_calls`, `list_telephony_compliance` and `callback_handoff`. With telephony off, `tools/list` returns the other 313.

**13 campaign tools:** the whole **Calling campaigns** table above. With campaigns off, `tools/list` returns the other 320.

The flow, menu and queue tools are **not** in either group. Call routing is authored before calling is switched on, so those routes are always mounted and those tools are always served.

Each group appears only where that feature is switched on. Where neither is, `tools/list` returns the other 300 and those 33 are simply absent, because the REST routes behind them are not mounted there. Calling one by name anyway returns the same `Unknown tool` error as a typo.

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
  "count": 333,
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

The server implements MCP revision `2025-11-25` and also accepts `2025-06-18` and `2025-03-26`. Send the version you speak on every call after initializing:

```
MCP-Protocol-Version: 2025-06-18
```

Omit the header and the server assumes `2025-03-26`, per the specification. Send a version it does not support and the call returns `400`. Batched requests are not accepted, because the `2025-06-18` revision removed them, so send one request object per POST.
