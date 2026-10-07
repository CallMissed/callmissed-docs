---
title: "Handoffs"
description: "The cross-channel queue of conversations waiting for a human: list and count them, assign an owner, resolve them back to the bot, and call a voice customer back."
slug: "handoffs"
breadcrumb: "API Reference"
---

# Handoffs

The cross-channel queue of conversations waiting for a human: list and count them, assign an owner, resolve them back to the bot, and call a voice customer back.

## Overview

A **handoff** is a [conversation](/docs/conversations) that needs a real person. The queue is every conversation in your tenant whose `status` is `escalated`, across WhatsApp, web chat and voice.

A conversation enters the queue when:

- your agent hands off to a human during a conversation (`raised_by: "ai"`, with a `reason`, and on voice a `summary` brief);
- a teammate takes a thread over by turning the bot off with [`POST /api/v1/conversations/{conversation_id}/autoreply`](/docs/conversations) and `enabled: false` (`raised_by: "human"`);
- you set the conversation's status to `escalated` with `PUT /api/v1/conversations/{conversation_id}/status` (no `reason` or `raised_by` is recorded).

When your agent hands off or a teammate takes over, the bot stops auto-replying on that conversation (`ai_autoreply_enabled: false` on the conversation). Setting the status to `escalated` with `PUT …/status` only puts the conversation in the queue; it does not turn the bot off. Resolving the handoff takes it off the queue and, by default, hands the thread back to the bot.

Handoffs are conversations, so there is no create or delete endpoint here, and the path id is the `conversation_id`.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, count | `conversations:read` |
| Assign, resolve, callback | `conversations:write` |

## Enumerations

| Field | Values |
| --- | --- |
| `channel` | `whatsapp`, `voice`, `web` |
| `status` | `active`, `completed`, `escalated`, `failed` — an open handoff is `escalated` |
| `raised_by` | `ai`, `human`, or `null` |

## The handoff object

```json
{
  "conversation_id": "c0ffee00-1111-2222-3333-444455556666",
  "channel": "voice",
  "status": "escalated",
  "external_id": "+919812345678",
  "contact_name": "Asha Rao",
  "reason": "Customer wants to dispute a charge",
  "summary": "Caller was double-charged on order A-4419 and wants a refund. Identity verified.",
  "voice_session_id": "5e6f…",
  "can_callback": true,
  "raised_by": "ai",
  "escalated_at": "2026-08-17T06:10:00Z",
  "assignee": { "id": "b1f2…", "name": "Ravi Kumar" },
  "last_message": "Can I talk to a person please?",
  "last_message_at": "2026-08-17T06:09:41Z",
  "bot_id": "7a8b…"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `conversation_id` | `UUID` | The conversation this handoff is |
| `channel` | `string` | See enumerations |
| `status` | `string` | The conversation's current status |
| `external_id` | `string` | The customer's id on the channel (WhatsApp id, phone number or web visitor id). Personal data — handle it as such |
| `contact_name` | `string \| null` | Name of the linked CRM contact, if any |
| `reason` | `string \| null` | Why the conversation was handed off |
| `summary` | `string \| null` | The agent's short brief for the human taking over (voice handoffs) |
| `voice_session_id` | `string \| null` | Voice handoffs only: the session to read the transcript from with the [Voice Session API](/docs/voice-sessions-api) |
| `can_callback` | `boolean` | `true` only when the `/callback` endpoint below can go through: a voice handoff with a captured customer number, an operator phone set in the console, and an active number of yours to dial from. The customer's number itself is never returned here |
| `raised_by` | `string \| null` | `ai`, `human`, or `null` when no handoff record was written |
| `escalated_at` | `datetime \| null` | When it entered the queue; falls back to the conversation's last update when no handoff time was recorded |
| `assignee` | `object \| null` | `{ "id", "name" }` of the current owner. `null` means unclaimed |
| `last_message` / `last_message_at` | `string \| null` / `datetime \| null` | Preview of the newest message in the thread |
| `bot_id` | `UUID` | The bot that owns the conversation |

## GET `/api/v1/handoffs`

Returns an array of handoff objects, ordered by the conversation's last update, newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `scope` | `string` | No | `open` (default) = conversations currently `escalated`. `all` also includes conversations whose handoff was already resolved |
| `limit` | `integer` | No | `1 <= limit <= 500`, default `100` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/handoffs?scope=open&limit=100" \
  -H "Authorization: Bearer cm_your_api_key"
```

Any `scope` other than `open` or `all` returns `422`.

## GET `/api/v1/handoffs/count`

The number of open handoffs (conversations currently `escalated`).

```bash
curl https://api.callmissed.com/api/v1/handoffs/count \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "count": 3 }
```

## POST `/api/v1/handoffs/{conversation_id}/assign`

Give a handoff an owner, or send it back to the unclaimed pool.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `assignee_id` | `UUID \| null` | No | An active user in your tenant. `null` (or omitted) unassigns |

A JSON body is required; send `{}` to unassign.

```bash
curl -X POST https://api.callmissed.com/api/v1/handoffs/c0ffee00-1111-2222-3333-444455556666/assign \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "assignee_id": "b1f2c3d4-1111-2222-3333-444455556666" }'
```

Returns the updated handoff object.

| Status | Detail |
| --- | --- |
| `404` | `Conversation not found` |
| `404` | `Assignee not found in this workspace` — not a user in your tenant, or deactivated |
| `409` | `This conversation is not an open handoff.` — the conversation's status is not `escalated` |
| `409` | `Already claimed by {name}.` — a signed-in teammate tried to claim (assign to themselves) a handoff someone else holds |

The claim conflict applies only to a signed-in teammate assigning a handoff to themselves. A request made with an API key is treated as a deliberate reassignment and replaces any current assignee.

## POST `/api/v1/handoffs/{conversation_id}/resolve`

Take the conversation off the queue. Its status becomes `active` (the conversation can carry on), the assignee is cleared, and the handoff record is kept and marked resolved, so it still shows under `scope=all`.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `resume_ai` | `boolean` | No | Default `true`: the bot resumes auto-replying. `false` keeps the bot silent so a human stays in control |

A JSON body is required; send `{}` to accept the default.

```bash
curl -X POST https://api.callmissed.com/api/v1/handoffs/c0ffee00-1111-2222-3333-444455556666/resolve \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "resume_ai": true }'
```

Returns the updated handoff object. `404 Conversation not found`.

This endpoint does not check that the conversation is currently `escalated`: calling it on any conversation in your tenant sets that conversation to `active`.

## POST `/api/v1/handoffs/{conversation_id}/callback`

One-click call-back for a **voice** handoff. CallMissed dials your operator phone first, then the customer, and connects the two directly — no AI on the call. Both legs are placed from one of your own active numbers. No request body.

Before calling it, check `can_callback` on the handoff. The preconditions are:

- the handoff is on the `voice` channel and a customer phone number was captured for it;
- an operator phone number is set for your workspace in the console (**Settings**);
- your workspace has at least one active [CallMissed number](/docs/telephony-api) to call from.

```bash
curl -X POST https://api.callmissed.com/api/v1/handoffs/c0ffee00-1111-2222-3333-444455556666/callback \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "status": "calling",
  "room_name": "bridge-…",
  "operator_call_id": "0d1e…",
  "caller_call_id": "2f3a…"
}
```

| Field | Notes |
| --- | --- |
| `status` | Always `calling` on success |
| `room_name` | Opaque identifier of the bridged call |
| `operator_call_id` / `caller_call_id` | The two outbound calls. Read them with `GET /api/v1/telephony/calls/{call_id}` ([Telephony API](/docs/telephony-api), scope `telephony:read`) |

**This endpoint spends credits.** Each leg is an outbound call billed like any other [telephony call](/docs/telephony-api): credits are reserved for both legs before dialling and settled to the real cost after the calls end. If either leg fails to dial, both reservations are refunded.

| Status | Detail |
| --- | --- |
| `400` | `Callback is only available for voice handoffs` |
| `402` | `insufficient credits` — not enough balance to reserve both legs |
| `403` | `This customer's number is on your do-not-call list.` or `Calls to this destination are not permitted.` |
| `404` | `Conversation not found` |
| `409` | `No customer phone number was captured for this handoff.` |
| `409` | `Set an operator phone number in Settings to call customers back.` |
| `409` | `Your workspace has no active phone number to call back from.` |
| `422` | `A phone number for the callback is invalid.` |
| `429` | `Too many calls are in progress right now. Try again shortly.` |

A failure while dialling returns an error status with a message that is safe to show to your operator.

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `conversations:read` / `conversations:write` |
| `404` | Conversation or assignee is not in your tenant |
| `409` | Assigning a conversation that is not an open handoff, a claim conflict, or a callback precondition is missing |
| `422` | Invalid `scope`, `limit` or `offset`, a malformed id, or a missing JSON body on assign/resolve |

Only `/callback` consumes credits; nothing else on this page does.
