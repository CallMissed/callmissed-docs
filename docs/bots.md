---
title: "Bots"
description: "Create and manage AI communication agents."
slug: "bots"
breadcrumb: "API Reference"
---

# Bots

Create and manage AI communication agents.

## Overview

A **bot** is a configured AI agent bound to a channel. The `type` is fixed at creation and determines how the bot is reached:

| type | Channel |
| --- | --- |
| `whatsapp` | WhatsApp text conversations |
| `whatsapp_voice` | WhatsApp voice notes |
| `inbound_call` | Phone calls customers place to you |
| `outbound_call` | Calls the bot places (see [Voice](/docs/voice) for availability) |
| `ivr` | Smart IVR menu flows |

Bots accept either a **dashboard session (JWT)** or a **`cm_` API key** with the `bots:read` / `bots:write` scopes.

## Create a bot

```bash
curl https://api.callmissed.com/api/v1/bots \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Support Bot",
    "type": "whatsapp",
    "system_prompt": "You are a friendly support agent for Acme. Keep replies under 3 sentences."
  }'
```

```json
{
  "id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "name": "Support Bot",
  "type": "whatsapp",
  "system_prompt": "You are a friendly support agent for Acme. Keep replies under 3 sentences.",
  "config": null,
  "is_active": true,
  "created_at": "2026-06-06T12:00:00Z",
  "updated_at": "2026-06-06T12:00:00Z",
  "conversation_count": 0
}
```

`PUT /api/v1/bots/:id` updates `name`, `system_prompt`, or `config` (not `type`). `POST /api/v1/bots/:id/toggle` flips `is_active`. Use `POST /api/v1/bots/:id/deploy/verify` (owner/admin) to confirm the WhatsApp/Twilio channel is connected before going live.