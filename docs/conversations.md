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
| List, search, get, messages | `conversations:read` |
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

Lists conversations. Requires `conversations:read`.

By default the newest **conversation** comes first (`sort=created`) and you page with `offset`. Pass `sort=last_activity` to get the inbox order instead: the conversation with the newest **message** first. That order changes every time a message arrives, so it pages with a cursor, not an offset: send each row's `cursor` value from the last row of a page as `cursor=` to get the next page.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `channel` | `string` | No | `whatsapp`, `voice`, or `web`. An unknown value returns `422` with a generic invalid-input message |
| `status` | `string` | No | `active`, `completed`, `escalated`, or `failed`. Same behaviour on an unknown value |
| `bot_id` | `UUID` | No | Restrict to one agent |
| `limit` | `integer` | No | `1 <= limit <= 500`, default `200` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0`. Only with `sort=created`: a non-zero offset with `sort=last_activity` returns `400` |
| `sort` | `string` | No | `created` (default) or `last_activity` |
| `cursor` | `string` | No | The `cursor` of the last row of the previous page. Only with `sort=last_activity`, otherwise `400`. A malformed cursor returns `400` |
| `unread` | `boolean` | No | `true`: only threads with unread customer messages; `false`: only the rest |
| `assigned` | `string` | No | `unassigned`, or a teammate's user id |
| `label` | `string` | No | Threads carrying this label. Case-insensitive; surrounding spaces and invisible characters are ignored, as they are when labels are saved |

```bash
# Inbox order, first page, then the next one
curl "https://api.callmissed.com/api/v1/conversations?sort=last_activity&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
curl "https://api.callmissed.com/api/v1/conversations?sort=last_activity&limit=50&cursor=WyIyMDI2LTA4LTA0VDEyOjAxOjAwKzAwOjAwIiwiYzBmZmVlMDAiXQ" \
  -H "Authorization: Bearer cm_your_api_key"
```

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
    "last_message_at": "2026-08-04T12:01:00Z",
    "unread_count": 2,
    "last_read_at": "2026-08-04T11:58:00Z",
    "last_message_preview": "Where is my order?",
    "last_message_type": "text",
    "last_message_direction": "in",
    "last_message_status": null,
    "display_name": "Asha Rao",
    "contact_name": "Asha Rao",
    "profile_name": "Asha",
    "cursor": null
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
| `last_message` | `string \| null` | The most recent message as stored |
| `last_message_at` | `string \| null` | When the most recent message was sent. `null` when the thread has none |
| `unread_count` | `integer` | `user` messages created after `last_read_at` |
| `last_read_at` | `string \| null` | `null` means the thread was never opened |
| `last_message_preview` | `string \| null` | What an inbox shows for the most recent message: its text, or for media and interactive messages the caption, file name or a label such as `Photo` or `Voice message (0:07)` |
| `last_message_type` | `string \| null` | `text`, `image`, `audio`, `video`, `document`, `sticker`, `location`, `contacts`, `interactive`, `button`, `reaction`, `template` or `system` |
| `last_message_direction` | `string \| null` | `in` from the customer, `out` from your business |
| `last_message_status` | `string \| null` | Outbound only: `sent`, `delivered`, `read` or `failed` |
| `display_name` | `string \| null` | The CRM contact's name, else the customer's WhatsApp profile name |
| `contact_name` | `string \| null` | The linked CRM contact's name |
| `profile_name` | `string \| null` | The name the customer set on WhatsApp |
| `cursor` | `string \| null` | With `sort=last_activity`: pass the last row's value as `cursor=` for the next page. `null` otherwise |

## GET `/api/v1/conversations/search`

Full-text search over your conversations' messages, newest first. Requires `conversations:read`.

It searches what an inbox shows: a text message's words, or a media or interactive message's caption, file name, contact names or button titles. Every word you send must appear; each matches as a prefix, so `ord` finds `order`. Words match as typed, with no stemming, so Hindi, Hinglish and order numbers work.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `q` | `string` | Yes | 1 to 200 characters |
| `limit` | `integer` | No | `1 <= limit <= 50`, default `20` |
| `cursor` | `string` | No | `next_cursor` from the previous page |
| `conversation_id` | `UUID` | No | Search within one conversation |

```bash
curl "https://api.callmissed.com/api/v1/conversations/search?q=refund&limit=20" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "results": [
    {
      "conversation_id": "c0ffee00-1111-2222-3333-444455556666",
      "message_id": "m1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8",
      "created_at": "2026-08-04T11:55:00Z",
      "direction": "in",
      "message_type": "text",
      "snippet": "Can I get a refund for order 5512?",
      "highlights": [{ "start": 12, "end": 18 }],
      "channel": "whatsapp",
      "external_id": "+919876543210",
      "display_name": "Asha Rao",
      "cursor": "WyIyMDI2LTA4LTA0VDExOjU1OjAwKzAwOjAwIiwibTFhMmIzYzQiXQ"
    }
  ],
  "next_cursor": null
}
```

`highlights` are offsets into `snippet` in UTF-16 code units, the same as JavaScript string indices. `next_cursor` is `null` on the last page.

| Status | Cause |
| --- | --- |
| `400` | `Invalid cursor`, or `Search is too broad. Add more letters or another word.` |
| `403` | Key missing `conversations:read` |
| `422` | `q` empty or over 200 characters, or `limit` out of range |

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

Returns the updated conversation, same shape as a list row.

`403` without `conversations:write`; `404` when not found; `422` on an invalid status value.

## POST `/api/v1/conversations/{conversation_id}/read`

Marks the thread read as of now by bumping `last_read_at`. Idempotent, safe to call on every inbox open. No required body. Requires `conversations:write`.

```bash
curl -X POST https://api.callmissed.com/api/v1/conversations/c0ffee00-1111-2222-3333-444455556666/read \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns the conversation, same shape as a list row, with `unread_count` recomputed after the bump (normally `0`).

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

Returns the conversation, same shape as a list row, with the new `ai_autoreply_enabled`. `403` without `conversations:write`; `404` when not found.

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
