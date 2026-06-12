---
title: "Conversations"
description: "Access conversation history and messages."
slug: "conversations"
breadcrumb: "API Reference"
---

# Conversations

Access conversation history and messages.

## Overview

A **conversation** is one thread between a customer and a bot on a channel, with an ordered list of **messages**. Every conversation belongs to a bot and is scoped to your tenant. Read access needs the `conversations:read` scope; status changes, AI drafts, and the auto-reply toggle need `conversations:write`. Both a dashboard JWT and a `cm_` API key work.

## List & filter

```bash
curl "https://api.callmissed.com/api/v1/conversations?channel=whatsapp&status=active" \
  -H "Authorization: Bearer cm_your_key"
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
    "created_at": "2026-06-06T11:55:00Z",
    "updated_at": "2026-06-06T12:01:00Z",
    "bot_name": "Support Bot"
  }
]
```

## Messages

`GET /api/v1/conversations/:id/messages` returns the thread in order:

```json
[
  { "id": "...", "conversation_id": "c0ffee00-...", "role": "user", "content": "Where is my order?", "message_type": "text", "tokens_used": null, "created_at": "2026-06-06T11:55:00Z" },
  { "id": "...", "conversation_id": "c0ffee00-...", "role": "assistant", "content": "Let me check that for you.", "message_type": "text", "tokens_used": 18, "created_at": "2026-06-06T11:55:02Z" }
]
```

## Human-in-the-loop

- `POST /api/v1/conversations/:id/ai-draft` returns a suggested reply (`{ "draft": "..." }`) **without sending it** — useful for an agent co-pilot.
- `POST /api/v1/conversations/:id/autoreply` with `{ "enabled": false }` pauses AI auto-replies on a single thread so a human can take over.
- `PUT /api/v1/conversations/:id/status` moves a thread between `active`, `completed`, `escalated`, and `failed`.