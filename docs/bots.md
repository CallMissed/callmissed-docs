---
title: "Bots"
description: "Create and manage AI communication agents, verify channel deployment, and load their knowledge base."
slug: "bots"
breadcrumb: "API Reference"
---

# Bots

Create and manage AI communication agents, verify channel deployment, and load their knowledge base.

## Overview

A **bot** is a configured AI agent bound to a channel. The `type` is fixed at creation and determines how the bot is reached:

| type | Channel |
| --- | --- |
| `whatsapp` | WhatsApp text conversations |
| `whatsapp_voice` | WhatsApp voice notes |
| `inbound_call` | Phone calls customers place to you |
| `outbound_call` | Calls the bot places (see [Voice](/docs/voice) for availability) |
| `ivr` | Smart IVR menu flows |

## Credential class: dashboard JWT **or** a `cm_` key with scopes

Both credentials work.

```
Authorization: Bearer <jwt_access_token>
# or
Authorization: Bearer cm_your_api_key
```

| Endpoints | Scope needed by a `cm_` key |
| --- | --- |
| List, get, tool catalog | `bots:read` |
| Create, update, delete, toggle, deploy verify | `bots:write` |
| Knowledge list | `knowledge:read` |
| Knowledge add, upload, delete | `knowledge:write` |

A dashboard JWT bypasses the scope check. One extra role rule applies to JWT callers: `POST /{bot_id}/deploy/verify` requires **owner or admin**.

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`; validation failures are `422`. A missing scope returns `403` naming the scope.

## Credential redaction in `config`

`config` is a free-form JSON object. On the way out, any key whose name looks like a credential (`access_token`, `api_key`, `auth_token`, `app_secret`, `secret`, `password`, `token`, `verify_token`, and similar, matched case-insensitively at any depth) is replaced with `"[REDACTED]"`. The stored value is unchanged; you simply cannot read it back.

Two `config` sub-keys are schema-validated at write time and return `422` when malformed: `analysis_variables` (post-call extraction) and `input_variables` (per-call `{{token}}` declarations). Everything else in `config` is stored as-is.

---

## GET /api/v1/bots

Lists your tenant's bots, newest first. No parameters.

```bash
curl https://api.callmissed.com/api/v1/bots \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
    "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
    "name": "Support Bot",
    "type": "whatsapp",
    "system_prompt": "You are a friendly support agent for Acme.",
    "config": { "phone_number_id": "109876543210987", "access_token": "[REDACTED]" },
    "is_active": true,
    "created_at": "2026-06-06T12:00:00Z",
    "updated_at": "2026-06-06T12:00:00Z",
    "conversation_count": 42
  }
]
```

`403` when a `cm_` key lacks `bots:read`.

## GET /api/v1/bots/tool-catalog

Agent tools a bot can enable. Write the chosen `name` values into `config.tools`; the runtime resolves them from the registry. No parameters. Requires `bots:read`.

```bash
curl https://api.callmissed.com/api/v1/bots/tool-catalog \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "name": "get_order_status",
    "description": "Look up the status of a customer order by id.",
    "category": "commerce",
    "parameters": {
      "type": "object",
      "properties": { "order_id": { "type": "string" } },
      "required": ["order_id"]
    }
  }
]
```

`parameters` is a JSON Schema object, empty (`{"type": "object", "properties": {}}`) for tools that take no arguments.

## POST /api/v1/bots

Returns `201`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1-255 chars |
| `type` | `string` | Yes | One of `whatsapp`, `inbound_call`, `outbound_call`, `ivr`, `whatsapp_voice`. Immutable after creation |
| `system_prompt` | `string` | No | Max 50000 chars, default `""` |
| `config` | `object \| null` | No | Free-form JSON. `analysis_variables` and `input_variables` are validated |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Support Bot",
    "type": "whatsapp",
    "system_prompt": "You are a friendly support agent for Acme. Keep replies under 3 sentences.",
    "config": { "tools": ["get_order_status"] }
  }'
```

```json
{
  "id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "name": "Support Bot",
  "type": "whatsapp",
  "system_prompt": "You are a friendly support agent for Acme. Keep replies under 3 sentences.",
  "config": { "tools": ["get_order_status"] },
  "is_active": true,
  "created_at": "2026-08-04T11:00:00Z",
  "updated_at": "2026-08-04T11:00:00Z",
  "conversation_count": 0
}
```

`403` without `bots:write`; `422` on an unknown `type`, a name outside 1-255 chars, a prompt over 50000 chars, or a malformed `analysis_variables` / `input_variables` block.

## GET `/api/v1/bots/{bot_id}`

Same object shape as one row of the list, with an accurate `conversation_count`. Requires `bots:read`.

```bash
curl https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab \
  -H "Authorization: Bearer cm_your_api_key"
```

`404` `Bot not found` when the id is not in your tenant; `422` when it is not a valid UUID.

## PUT `/api/v1/bots/{bot_id}`

Partial update. Omitted fields are unchanged. `type` cannot be changed. Requires `bots:write`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string \| null` | No | 1-255 chars |
| `system_prompt` | `string \| null` | No | Max 50000 chars |
| `config` | `object \| null` | No | **Replaces** the whole object, not a deep merge |

```bash
curl -X PUT https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"system_prompt": "You are a concise support agent for Acme."}'
```

Returns the updated bot. Note that `conversation_count` is `0` in the update response; read the bot to get the real count.

`403` without `bots:write`; `404` when not found; `422` on validation failure.

## DELETE `/api/v1/bots/{bot_id}`

Returns `204` with an empty body. Requires `bots:write`.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab \
  -H "Authorization: Bearer cm_your_api_key"
```

`403` without `bots:write`; `404` when not found.

## POST `/api/v1/bots/{bot_id}/toggle`

Flips `is_active`. No request body. Requires `bots:write`.

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/toggle \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns the bot with the new `is_active`. `403` without `bots:write`; `404` when not found.

## POST `/api/v1/bots/{bot_id}/deploy/verify`

Live connectivity check against the bot's channel, using credentials stored in the bot's own `config`. No request body. Requires `bots:write`; a JWT caller must also be **owner or admin**.

Which credentials are read depends on the bot `type`:

| Bot type | Reads from `config` | Checked against |
| --- | --- | --- |
| `whatsapp`, `whatsapp_voice` | `phone_number_id`, `access_token` | Meta Graph API |
| `inbound_call`, `outbound_call`, `ivr` | `account_sid`, `auth_token` | Twilio REST API |

The result is also persisted back into `config` as `channel_verified`, `last_verified_at`, and `error_message`.

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/deploy/verify \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "channel_verified": true,
  "last_verified_at": "2026-08-04T11:05:33.129004+00:00",
  "error_message": null
}
```

A failed check is still `200`, with `channel_verified: false` and a reason such as `Missing phone_number_id or access_token`, `WhatsApp API returned 401`, `Twilio API returned 401`, or `Provider API timeout — try again`.

| Status | Cause |
| --- | --- |
| `403` | Key missing `bots:write`, or `Only owners/admins can verify deployments` for a JWT caller |
| `404` | `Bot not found` |

---

# Knowledge base

Entries attached to one bot. See [Knowledge Base](/docs/knowledge) for retrieval behaviour.

## GET `/api/v1/bots/{bot_id}/knowledge`

Lists entries for the bot, newest first. Requires `knowledge:read`.

```bash
curl https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/knowledge \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "id": "k9e8d7c6-b5a4-4321-9876-543210fedcba",
    "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
    "name": "Refund policy",
    "content": "Refunds are issued within 7 business days...",
    "metadata": { "source": "handbook" },
    "file_url": null,
    "file_size_bytes": null,
    "format": null,
    "status": "indexed",
    "error_message": null,
    "created_at": "2026-07-11T09:00:00Z"
  }
]
```

`403` without `knowledge:read`; `404` `Bot not found`.

## POST `/api/v1/bots/{bot_id}/knowledge`

Adds a text entry. Returns `201`. Requires `knowledge:write`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1-255 chars |
| `content` | `string` | Yes | 1-100000 chars |
| `metadata` | `object \| null` | No | Free-form JSON |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/knowledge \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Refund policy",
    "content": "Refunds are issued within 7 business days of the return being received.",
    "metadata": { "source": "handbook" }
  }'
```

Returns the created entry. `403` without `knowledge:write`; `404` `Bot not found`; `422` when `content` is empty or over 100000 chars.

## POST `/api/v1/bots/{bot_id}/knowledge/upload`

Uploads a document as `multipart/form-data` and extracts its text synchronously. Returns `201`. Requires `knowledge:write`.

| Part | Type | Required | Constraints |
| --- | --- | --- | --- |
| `file` | file | Yes | Extension must be `pdf`, `docx`, or `txt`. Max **20 MB** (20,971,520 bytes) |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/knowledge/upload \
  -H "Authorization: Bearer cm_your_api_key" \
  -F "file=@handbook.pdf"
```

```json
{
  "id": "k1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8",
  "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
  "name": "handbook.pdf",
  "content": "Acme employee handbook\n\n1. Refunds...",
  "metadata": null,
  "file_url": null,
  "file_size_bytes": 184203,
  "format": "pdf",
  "status": "indexed",
  "error_message": null,
  "created_at": "2026-08-04T11:12:00Z"
}
```

Check `status` on the response. When extraction yields nothing usable the entry is still created with `status: "failed"` and an `error_message` such as `No text content could be extracted`. The `name` is the uploaded filename.

| Status | Cause |
| --- | --- |
| `400` | `File exceeds 20MB limit`, or `Unsupported format: <ext>. Use PDF, DOCX, or TXT.` |
| `403` | Key missing `knowledge:write` |
| `404` | `Bot not found` |

## DELETE `/api/v1/bots/{bot_id}/knowledge/{entry_id}`

Returns `204` with an empty body. Requires `knowledge:write`.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/knowledge/k9e8d7c6-b5a4-4321-9876-543210fedcba \
  -H "Authorization: Bearer cm_your_api_key"
```

`403` without `knowledge:write`; `404` `Knowledge entry not found` when the entry is not on that bot in your tenant.
