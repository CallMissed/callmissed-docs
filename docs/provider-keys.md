---
title: "Bring Your Own Key"
description: "Store your own model-provider credentials, verify them, rotate them, and deactivate them — the secret is never returned."
slug: "provider-keys"
breadcrumb: "API Reference"
---

# Bring Your Own Key

Store your own model-provider credentials, verify them, rotate them, and deactivate them — the secret is never returned.

## Overview

Bring Your Own Key (BYOK) lets you store your own credential for a model provider so inference runs on your account with that provider instead of ours.

The stored secret is **write-only**. No endpoint on this page returns it, and there is no reveal route. You can see the provider, an optional label, the last four characters, whether it is active, and when it was last verified — nothing more. If you lose the original secret, delete the record and add a new one.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List | `provider_keys:read` |
| Create, update, delete, verify | `provider_keys:write` |

## Supported providers

`openai`, `anthropic`, `google`, `sarvam`, `deepgram`, `elevenlabs`, and the other providers accepted by the create endpoint. Provider ids are lowercase. An unsupported value returns `422` and lists the accepted set.

## The uniqueness rule

A credential occupies one slot per `(provider, label)`. A record with no label is that provider's **default** slot. Adding a second key to an occupied slot returns `409`:

```json
{ "detail": "A default openai key already exists. Delete it first to replace the secret." }
```

To hold several keys for one provider, give each a distinct `label` (for example `eu`, `us`, `batch`).

**Rotation is delete-then-create.** `PATCH` deliberately refuses a `key` field, so a secret can never be silently swapped under a record that other systems believe they know.

## GET `/api/v1/gateway/provider-keys`

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `provider` | `string` | No | At most 32 characters, case-insensitive, must be a supported provider |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `100` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```json
[
  {
    "id": "1a2b…",
    "provider": "openai",
    "label": "eu",
    "key_last4": "9f3a",
    "is_active": true,
    "last_verified_at": "2026-08-17T09:12:00Z",
    "created_at": "2026-08-02T11:00:00Z",
    "updated_at": "2026-08-17T09:12:00Z"
  }
]
```

Newest first.

## POST `/api/v1/gateway/provider-keys`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `provider` | `string` | Yes | 1–32 characters, lowercased, must be supported |
| `key` | `string` | Yes | 8–512 characters, not blank. Write-only — stored encrypted, never returned |
| `label` | `string` | No | At most 128 characters. Blank is stored as no label (the default slot) |

Unknown fields are rejected with `422` rather than ignored, so a typo in a field name fails loudly instead of quietly dropping your secret.

```bash
curl -X POST https://api.callmissed.com/api/v1/gateway/provider-keys \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "provider": "openai", "key": "sk-…", "label": "eu" }'
```

Returns `201` with the record — note the response has no `key` field.

## PATCH `/api/v1/gateway/provider-keys/{key_id}`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `label` | `string` | No | At most 128 characters |
| `is_active` | `boolean` | No | `false` takes the credential out of use without deleting it |

Sending `key` returns `422`. Moving a label into an occupied slot returns `409`.

Use `is_active: false` when you suspect a credential and want traffic to fall back immediately while you investigate.

## DELETE `/api/v1/gateway/provider-keys/{key_id}`

Returns `204`. `404 Provider key not found` for an unknown or another tenant's id.

## POST `/api/v1/gateway/provider-keys/{key_id}/verify`

Performs a live check against the provider. No request body.

```json
{
  "id": "1a2b…",
  "provider": "openai",
  "ok": true,
  "detail": "Credential accepted by the provider.",
  "last_verified_at": "2026-08-17T09:12:00Z"
}
```

`detail` is one of exactly two strings — `Credential accepted by the provider.` or `The provider rejected this credential or could not be reached.` The upstream status code and body are never passed through, so a verify call cannot be used to probe a provider. A successful check stamps `last_verified_at`.

| Status | Detail | Cause |
| --- | --- | --- |
| `400` | `Liveness verification is not available for {provider} keys.` | That provider has no cheap check |
| `404` | `Provider key not found` | Unknown or another tenant's id |
| `409` | `This stored credential can no longer be read. Delete and re-add it.` | The stored secret is unreadable — re-add it |

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `provider_keys:read` / `provider_keys:write` |
| `404` | Record not found in your tenant |
| `409` | Slot already occupied, or an unreadable stored credential |
| `422` | Unsupported provider, blank/short secret, an unknown field, or a `key` sent to `PATCH` |

Storing, verifying and deleting provider keys never consumes credits.
