---
title: "Authentication"
description: "Every API request authenticates with a CallMissed API key."
slug: "authentication"
breadcrumb: "API Reference"
---

# Authentication

Every API request authenticates with a CallMissed API key.

## API Key

Every request authenticates with an API key. Create one from your Profile page in the dashboard — keys are prefixed with `cm_` and are passed as a Bearer token:

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

Scopes gate the resource endpoints under `/api/v1/` — bots, conversations, knowledge, webhooks. Unlike permissions, scopes default to **empty = no resource access** — you opt in explicitly.

| Scope | Gates |
|-------|-------|
| `bots:read` / `bots:write` | View vs. create/update/delete bots |
| `conversations:read` / `conversations:write` | View vs. update conversations |
| `knowledge:read` / `knowledge:write` | View vs. add/remove knowledge entries |
| `webhooks:write` | Manage outbound webhook subscriptions |
| `whatsapp:read` / `whatsapp:write` | Read vs. manage WhatsApp messaging |
| `*` | All resource scopes |

> A key created for plain inference (the common case) needs only service permissions — leave scopes empty.
