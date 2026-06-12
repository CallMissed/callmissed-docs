---
title: "Authentication"
description: "All API requests require authentication via Bearer token or API key."
slug: "authentication"
breadcrumb: "API Reference"
---

# Authentication

All API requests require authentication via Bearer token or API key.

## Bearer Token

After login or register, you receive an `access_token`. Pass it in every request:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Tokens expire after 24 hours. Use the refresh endpoint to rotate.

## API Key

For server-to-server integrations, create an API key from your Profile page. Keys are prefixed with `cm_`:

```
Authorization: Bearer cm_your_api_key_here
```

API keys never expire but can be revoked at any time.

### Anthropic SDK (x-api-key header)

When using the [Anthropic-compatible endpoint](/docs/anthropic-api) (`/v1/messages`), you can also authenticate with the `x-api-key` header:

```
x-api-key: cm_your_api_key_here
```

Both header styles work on the Anthropic endpoint — use whichever your SDK sends by default.

## Permissions vs. scopes

API keys carry two independent access controls.

### Service permissions

Permissions decide which AI services a key may call. They are enforced on the inference endpoints — a key without the matching permission gets `403 permission_denied`. Set any combination of `llm`, `stt`, `tts`, `search`, `image`, or `*` for all (the default for new keys).

| Permission | Gates |
|------------|-------|
| `llm` | `/v1/chat/completions`, `/v1/messages` (+ `/anthropic/v1/messages`) |
| `stt` | `/v1/audio/transcriptions`, `/v1/audio/translations` |
| `tts` | `/v1/audio/speech` |
| `search` | `/v1/search` |
| `image` | `/v1/images/generations` |

### Resource scopes

Scopes gate the management (dashboard-style) endpoints under `/api/v1/` when you call them with `Bearer cm_*` instead of a dashboard login. Unlike permissions, scopes default to **empty = no resource access** — you opt in explicitly.

| Scope | Gates |
|-------|-------|
| `bots:read` / `bots:write` | View vs. create/update/delete bots |
| `conversations:read` / `conversations:write` | View vs. update conversations |
| `knowledge:read` / `knowledge:write` | View vs. add/remove knowledge entries |
| `webhooks:write` | Manage outbound webhook subscriptions |
| `whatsapp:read` / `whatsapp:write` | Read vs. manage WhatsApp messaging |
| `*` | All resource scopes |

> A key created for plain inference (the common case) needs only service permissions — leave scopes empty. Dashboard JWT sessions always have full access to both services and resources.