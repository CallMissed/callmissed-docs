---
title: "API Keys"
description: "What an API key controls: service permissions, resource scopes, budget caps and logging."
slug: "keys"
breadcrumb: "API Reference"
---

# API Keys

What an API key controls: service permissions, resource scopes, budget caps and logging.

> Keys are created, rotated and revoked in the **console** (Developer → API keys) — you cannot manage keys with a key, so there is no API for it. This page explains what the settings on a key actually do.

A key is shown in plaintext **once**, at creation. Store it immediately; if you lose it, the console can re-reveal it after an emailed one-time code, or you can rotate it.

## Service permissions

Permissions decide which AI services the key may call. A key without the matching permission gets `403 permission_denied`.

| Permission | Gates |
|------------|-------|
| `llm` | `/v1/chat/completions`, `/v1/responses`, `/v1/messages` (+ `/anthropic/v1/messages`), `/v1/embeddings`, `/v1/batches` and `/v1/files` |
| `stt` | `/v1/audio/transcriptions`, `/v1/audio/translations` |
| `tts` | `/v1/audio/speech` |
| `search` | `/v1/search` |
| `image` | `/v1/images/generations`, `/v1/images/history` |
| `email` | The [Email API](/docs/email) |
| `*` | All services (the default for a new key) |

Voice sessions and the Managed Voice Agent sockets need **all three** of `stt`, `tts` and `llm`.

## Resource scopes

Scopes gate the platform resources the key may touch. They are independent of permissions and default to **empty = no resource access** — you opt in explicitly.

| Scope | Gates |
|-------|-------|
| `bots:read` / `bots:write` | View vs. create/update/delete agents |
| `conversations:read` / `conversations:write` | View vs. update conversations and handoffs |
| `knowledge:read` / `knowledge:write` | View vs. add/remove knowledge sources |
| `webhooks:write` | Manage webhook subscriptions and read their delivery log |
| `whatsapp:read` / `whatsapp:write` / `whatsapp:send` | Read vs. manage vs. send WhatsApp messaging |
| `*` | All resource scopes |

CRM, support desk, campaigns, telephony, integrations, gateway and voice-agent tooling each have their own pairs. The full list, with the page each one unlocks, is on [Authentication](/docs/authentication#resource-scopes).

A key created for plain inference — the common case — needs only service permissions. Leave scopes empty.

## Limits on a key

Each key carries its own guardrails, so a key handed to one service or environment cannot spend or reach beyond what you intended.

| Setting | What it does |
|---------|--------------|
| **Budget** | A credit cap for the key. Once spent, calls on that key fail with `402` until you raise it. Other keys keep working. |
| **RPM limit** | A per-key requests-per-minute ceiling, independent of your plan's overall quota. |
| **Allowed models** | Restricts the key to a named set of model ids, or `*` for everything your plan allows. |
| **Allowed search providers** | Restricts which providers `/v1/search` may use on this key. |
| **Domain allowlist** | Restricts which origins may send the key, for browser-side use. `*` allows any. |

## Logging

Logging is **off** by default on a new key and is opt-in per key:

- **Request logging** records metadata for each call — model, timing, token counts, credits spent.
- **Prompt logging** additionally stores message content. Enable it only when you need to inspect payloads.

## Expiry

A key can be given an expiry (1-365 days) when it is created. After that it gets `401`, and an `api_key.expired` [webhook](/docs/webhooks) fires. A key with no expiry set never expires.

## Rotating and revoking

Rotate a key in the console whenever it may have been exposed. Rotation issues a new secret for the **same** key: its id, name, permissions, scopes and budget carry over, and the old secret stops working within about a minute. There is no overlap window, so deploy the new secret straight away. Revocation likewise takes effect within about a minute; in-flight calls already authenticated are not retroactively cancelled. Because budgets, scopes and limits are per key, issuing one key per service or environment keeps a leak contained and makes usage attributable.
