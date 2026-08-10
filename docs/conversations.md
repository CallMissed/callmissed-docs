---
title: "Conversations"
description: "Read conversation history and messages, move threads through statuses, draft AI replies, and take a thread over from the bot."
slug: "conversations"
breadcrumb: "API Reference"
---

# Conversations

Read conversation history and messages, move threads through statuses, draft AI replies, and take a thread over from the bot.

## Overview

A **conversation** is one thread between a customer and a bot on a channel, with an ordered list of **messages**. Every conversation belongs to a bot and is scoped to your tenant.

Conversations are created by the platform when a customer reaches one of your bots. There is no create or delete endpoint on this surface.

## Credential class: dashboard JWT **or** a `cm_` key with scopes

Both credentials work.

```
Authorization: Bearer <jwt_access_token>
# or
Authorization: Bearer cm_your_api_key
```

| Endpoints | Scope needed by a `cm_` key |
| --- | --- |
| List, get, messages | `conversations:read` |
| Status update, AI draft, mark read, autoreply toggle | `conversations:write` |

A dashboard JWT bypasses the scope check. No extra role rule applies here: any member of the tenant can use every endpoint on this page.

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`; validation failures are `422`. A missing scope returns `403` naming the scope.

## Enumerations

| Field | Values |
| --- | --- |
| `channel` | `whatsapp`, `voice`, `web` |
| `status` | `active`, `completed`, `escalated`, `failed` |
| `role` (message) | `user` for the customer, `assistant` for the bot or agent |
| `status` (message) | `sent`, `delivered`, `read`, `failed`. Inbound rows carry `sent` |

---

## GET /api/v1/conversations

Lists conversations, newest first. Requires `conversations:read`.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `channel` | `string` | No | `whatsapp`, `voice`, or `web`. An unknown value returns `422` with a generic invalid-input message |
| `status` | `string` | No | `active`, `completed`, `escalated`, or `failed`. Same behaviour on an unknown value |
| `bot_id` | `UUID` | No | Restrict to one agent |
| `limit` | `integer` | No | `1 <= limit <= 500`, default `200` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/conversations?channel=whatsapp&status=active&limit=200" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "id": "c0ffee00-1111-2222-3333-444455556666",
    "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
    "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
    "channel": "whatsapp",
    "external_id": "+919876543210",
    "status": "active",
    "duration_seconds": null,
    "metadata": null,
    "ai_autoreply_enabled": true,
    "created_at": "2026-08-04T11:55:00Z",
    "updated_at": "2026-08-04T12:01:00Z",
    "bot_name": "Support Bot",
    "last_message": "Where is my order?",
    "unread_count": 2,
    "last_read_at": "2026-08-04T11:58:00Z"
  }
]
```

| Field | Type | Notes |
| --- | --- | --- |
| `external_id` | `string` | The customer's channel identity, for example the WhatsApp phone number |
| `duration_seconds` | `integer \| null` | Populated for voice threads |
| `metadata` | `object \| null` | Free-form JSON attached by the platform |
| `ai_autoreply_enabled` | `boolean` | Whether the bot still answers automatically on this thread |
| `bot_name` | `string` | `""` when the bot has been deleted |
| `last_message` | `string \| null` | Preview of the most recent message |
| `unread_count` | `integer` | `user` messages created after `last_read_at` |
| `last_read_at` | `string \| null` | `null` means the thread was never opened |

## GET `/api/v1/conversations/{conversation_id}`

One conversation, same shape as a list row. Requires `conversations:read`.

```bash
curl https://api.callmissed.com/api/v1/conversations/c0ffee00-1111-2222-3333-444455556666 \
  -H "Authorization: Bearer cm_your_api_key"
```

`404` `Conversation not found` when the id is not in your tenant; `422` when it is not a valid UUID.

## GET `/api/v1/conversations/{conversation_id}/messages`

The full thread in chronological order. No pagination parameters: the whole thread is returned. Requires `conversations:read`.

```bash
curl https://api.callmissed.com/api/v1/conversations/c0ffee00-1111-2222-3333-444455556666/messages \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "id": "m1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8",
    "conversation_id": "c0ffee00-1111-2222-3333-444455556666",
    "role": "user",
    "content": "Where is my order?",
    "message_type": "text",
    "tokens_used": null,
    "status": "sent",
    "created_at": "2026-08-04T11:55:00Z"
  },
  {
    "id": "m2b3c4d5-e6f7-4081-92a3-b4c5d6e7f809",
    "conversation_id": "c0ffee00-1111-2222-3333-444455556666",
    "role": "assistant",
    "content": "Let me check that for you.",
    "message_type": "text",
    "tokens_used": 18,
    "status": "delivered",
    "created_at": "2026-08-04T11:55:02Z"
  }
]
```

`404` `Conversation not found`; `403` without `conversations:read`.

## PUT `/api/v1/conversations/{conversation_id}/status`

Moves a thread between statuses. Requires `conversations:write`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `status` | `string` | Yes | Exactly one of `active`, `completed`, `escalated`, `failed` |

```bash
curl -X PUT https://api.callmissed.com/api/v1/conversations/c0ffee00-1111-2222-3333-444455556666/status \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"status": "completed"}'
```

Returns the updated conversation. In this response `bot_name` is `""` and the preview fields are not recomputed; re-read the conversation if you need them.

`403` without `conversations:write`; `404` when not found; `422` on an invalid status value.

## POST `/api/v1/conversations/{conversation_id}/read`

Marks the thread read as of now by bumping `last_read_at`. Idempotent, safe to call on every inbox open. No required body. Requires `conversations:write`.

```bash
curl -X POST https://api.callmissed.com/api/v1/conversations/c0ffee00-1111-2222-3333-444455556666/read \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns the conversation with `unread_count` recomputed after the bump (normally `0`).

`403` without `conversations:write`; `404` when not found.

## POST `/api/v1/conversations/{conversation_id}/autoreply`

Per-thread gate on the bot's automatic replies. Set `enabled: false` when a human takes the thread over; the inbound handler then skips the bot reply until it is flipped back. Requires `conversations:write`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `enabled` | `boolean` | Yes | - |

```bash
curl -X POST https://api.callmissed.com/api/v1/conversations/c0ffee00-1111-2222-3333-444455556666/autoreply \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"enabled": false}'
```

Returns the conversation with the new `ai_autoreply_enabled`. `403` without `conversations:write`; `404` when not found.

## POST `/api/v1/conversations/{conversation_id}/ai-draft`

Generates a suggested reply for a human to review. **It never sends anything.** Pair it with `autoreply: false` to use the model as a co-pilot on a thread a human has taken over. Requires `conversations:write`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `instruction` | `string \| null` | No | Max 1000 chars. Extra steering for this draft only, for example `shorter` or `apologise for the delay` |

```bash
curl -X POST https://api.callmissed.com/api/v1/conversations/c0ffee00-1111-2222-3333-444455556666/ai-draft \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"instruction": "apologise for the delay and offer to check the tracking"}'
```

```json
{
  "draft": "Sorry about the wait. Let me pull up your tracking details right now and get back to you in a minute.",
  "used_default_prompt": false
}
```

`used_default_prompt` is `true` when the linked bot has no system prompt, in which case a built-in support-agent persona is used. The draft is clamped to 4096 characters so it is always sendable on WhatsApp.

**This call is billed** at the same LLM token rates as the auto-reply path.

| Status | Cause |
| --- | --- |
| `400` | `No messages in this conversation yet — nothing to draft from.` |
| `403` | Key missing `conversations:write` |
| `404` | `Conversation not found` |
| `502` | `AI draft failed; please retry.` or `AI returned an empty draft — please retry.` |
