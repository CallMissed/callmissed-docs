---
title: "Agent Memories"
description: "Read, teach and erase the standing facts an agent remembers, including the newest ones it carries into every turn without searching."
slug: "agent-memories"
breadcrumb: "API Reference"
---

# Agent Memories

Read, teach and erase the standing facts an agent remembers, including the newest ones it carries into every turn without searching.

> A memory is one short fact an agent was told to keep. These endpoints are the operator's view of that store: see exactly what is remembered, teach a fact directly, and delete anything that should not be there.

:::cards
/docs/bots | Bots | bot | The agent a memory belongs to, and its built-in tool list
/docs/agent-tools | Agent Tools | wrench | The custom REST tools the same agent calls mid-conversation
:::

## The injection window: what the agent reads without asking

Every memory stays searchable forever. Only the newest **12** are handed to the agent at the start of a turn, so it acts on them without deciding to look anything up. That set is the injection window, and `injected: true` marks it on every listed memory.

The window is applied today on the linked personal WhatsApp channel. On every other channel a memory is still remembered and still found by `recall_memory`; it is simply not placed in the prompt ahead of time, so `injected` there tells you where a fact sits in the order rather than that it was already read.

| | Inside the window | Outside it |
| --- | --- | --- |
| Newest ... | 12 facts | Everything older |
| How the agent gets it | Already in front of it, every turn | It calls `recall_memory` and the fact is found by meaning, not by keyword |
| Reliability | Always applied | Applied when the conversation gives the agent a reason to search |

So a fact does not stop existing when it falls out of the window. It stops being free. If a preference must hold on every single reply, keep it near the top by restating it: memories are ordered newest first, and a fresh statement of the same preference moves it back into the window.

<Callout type="info">
  Two more bounds shape the window, and they explain why a very long fact behaves differently from a short one. Each injected fact is trimmed to **200 characters** in the prompt, and the whole block is capped at **1200 characters**, so a run of long facts fills the budget before the twelfth one is reached. `injected` is positional (the newest 12 rows) and does not model the character budget. Write facts as one line each and neither bound will ever bite.
</Callout>

## What is listed here

A memory is stored in one of two forms, and this surface serves only the first.

| | Explicit facts | Captured transcripts |
| --- | --- | --- |
| Where it comes from | The agent's `remember` tool, or a `POST` here | A finished call or chat, captured whole |
| Listed by these endpoints | Yes | No |
| Deletable by these endpoints | Yes | No |
| Reachable by the agent | Injected, then searched | Searched, through `recall_memory` |

Transcripts are deliberately absent: a transcript is not something an operator can meaningfully review or correct one line at a time, so it stays retrieval-only rather than becoming a browsable message archive.

## Credential class: dashboard JWT **or** a `cm_` key

```
Authorization: Bearer cm_your_api_key
```

| Caller | Requirement |
| --- | --- |
| `cm_` API key | `bots:read` to list. `bots:write` to create and delete |
| Dashboard JWT | Listing for any member. Creating and deleting require **owner or admin** (`403 owner or admin role required` otherwise) |

A memory is an attribute of a bot rather than a resource class of its own, so it reuses the `bots:*` scopes and adds none.

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`.

## The memory object

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "fact": "Wholesale orders ship Tuesdays only. Never promise a Friday dispatch.",
  "created_at": "2026-09-08T12:00:00Z",
  "injected": true,
  "captured_by": "remember_tool"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` | Pass it to `DELETE` to forget this one fact |
| `fact` | `string` | The memory **in full**. Read from the memory's stored text, not from the 120-character title kept for display, so a long fact comes back whole here even though the agent reads only its first 200 characters in the prompt |
| `created_at` | `string` | When it was remembered. The list is ordered by this, newest first |
| `injected` | `boolean` | `true` for the newest 12. See [the injection window](#the-injection-window-what-the-agent-reads-without-asking) |
| `captured_by` | `string` | How the fact was captured: `remember_tool` when the agent saved it mid-conversation, `remember_command` when the owner saved it with a chat command, `console` when it was taught through `POST`. `null` when the memory records no capture source |

## Memories belong to one agent

A memory is scoped to a workspace **and** a bot. Two agents in the same workspace do not share a fact, and re-pointing a phone number at a different agent gives it a different set of memories, not the same set under a new name. Move a fact by `POST`ing it to the second agent.

## GET /api/v1/agent-memories

Everything one agent has been told to remember, newest first. Requires `bots:read`.

| Parameter | Type | Notes |
| --- | --- | --- |
| `bot_id` | `uuid` | **Required.** The agent whose memories to list |
| `limit` | `integer` | 1 to 200, default 100 |

```bash
curl "https://api.callmissed.com/api/v1/agent-memories?bot_id=b1f2c3d4-5678-90ab-cdef-1234567890ab&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
  "memories": [
    {
      "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "fact": "Wholesale orders ship Tuesdays only. Never promise a Friday dispatch.",
      "created_at": "2026-09-08T12:00:00Z",
      "injected": true,
      "captured_by": "remember_tool"
    }
  ],
  "max_injected": 12
}
```

`max_injected` is the size of the injection window. Read it instead of hardcoding 12, so a change to the window does not silently make your own display wrong.

## POST /api/v1/agent-memories

Teach the agent a fact directly, without waiting for it to come up in a conversation. Returns `201` and the memory object. Requires `bots:write`, or an owner or admin JWT.

| Field | Type | Notes |
| --- | --- | --- |
| `bot_id` | `uuid` | Required. The agent that should remember this |
| `fact` | `string` | Required. 1 to 2000 characters. Surrounding whitespace is stripped |

```bash
curl -X POST https://api.callmissed.com/api/v1/agent-memories \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
    "fact": "Wholesale orders ship Tuesdays only. Never promise a Friday dispatch."
  }'
```

The new memory always comes back with `injected: true`, because it is the newest, and `captured_by: "console"`, which is how a taught fact is distinguished from one the agent chose to keep.

<Callout type="info">
  Write one fact per call, in one line, the way you would say it out loud. The agent is given these verbatim, so "Wholesale orders ship Tuesdays only" steers a reply and a pasted paragraph of policy does not. Anything longer belongs in the [knowledge base](/docs/knowledge), which the agent searches instead of carrying.
</Callout>

| Status | Meaning |
| --- | --- |
| `201` | Remembered |
| `403` | The key lacks `bots:write`, or the JWT is neither owner nor admin |
| `404` | `bot_id` is not an agent in this workspace. Nothing is written |
| `422` | `fact` is empty after stripping, or longer than 2000 characters |

## DELETE `/api/v1/agent-memories/{memory_id}`

Forget one fact. Returns `204` and no body. Requires `bots:write`, or an owner or admin JWT.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/agent-memories/7c9e6679-7425-40de-944b-e07fc1f90ae7 \
  -H "Authorization: Bearer cm_your_api_key"
```

Deletion is **idempotent**: `204` whether the memory was there or not, so a retry after a dropped connection is safe. It is also scoped to memories, so this route cannot be used to remove an ordinary knowledge document by guessing at its id. The fact and its search index go together; there is no way to read a deleted memory back.

<Callout type="warn">
  These facts are asserted about real people and are put in front of the agent, so treat this endpoint as the erasure path it is. A memory a customer asks you to remove is removed here, in full, not edited into something softer. There is no update: delete the old fact and `POST` the corrected one, which also moves it back into the injection window.
</Callout>

## Errors

| Status | When |
| --- | --- |
| `401` | No credential, or one that is not valid |
| `403` | The key is missing `bots:read` (listing) or `bots:write` (writing), or the dashboard JWT is not an owner or admin |
| `404` | `bot_id` does not name an agent in this workspace |
| `422` | `bot_id` is missing or not a UUID, `limit` is outside 1 to 200, or `fact` is empty or too long |

<Callout type="warn">
  A `bot_id` belonging to another workspace returns `404`, never `403`. The ownership check runs before anything else, so a caller cannot tell "not yours" apart from "does not exist" and cannot use this endpoint to confirm that an id exists somewhere else.
</Callout>

## Billing

A memory is embedded when it is written, so that the agent can find it later by meaning rather than by keyword. That embedding is billed like any other embedding on your account, at the rate on the [pricing page](/docs/pricing). Listing and deleting are free.

A memory whose embedding did not complete still appears in the list, with `fact` falling back to its first 120 characters, because an un-embedded memory is exactly the thing you need to see rather than one that quietly vanishes.
