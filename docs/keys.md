---
title: "API Keys"
description: "What an API key controls: service permissions, resource scopes, budget caps and logging."
slug: "keys"
breadcrumb: "API Reference"
---

# API Keys

What an API key controls: service permissions, resource scopes, budget caps and logging.

> Keys are created, rotated and revoked from the **dashboard** — you cannot manage keys with a key. This page explains what the settings on a key actually do.

A key is shown in plaintext **once**, at creation. Store it immediately; if you lose it, the dashboard can re-reveal it after an emailed one-time code, or you can roll a new one.

## Service permissions

Permissions decide which AI services the key may call. A key without the matching permission gets `403 permission_denied`.

| Permission | Gates |
|------------|-------|
| `llm` | `/v1/chat/completions`, `/v1/messages` (+ `/anthropic/v1/messages`) |
| `stt` | `/v1/audio/transcriptions`, `/v1/audio/translations` |
| `tts` | `/v1/audio/speech` |
| `search` | `/v1/search` |
| `image` | `/v1/images/generations` |
| `*` | All services (the default for a new key) |

## Resource scopes

Scopes gate the platform resources the key may touch. They are independent of permissions and default to **empty = no resource access** — you opt in explicitly.

| Scope | Gates |
|-------|-------|
| `bots:read` / `bots:write` | View vs. create/update/delete bots |
| `conversations:read` / `conversations:write` | View vs. update conversations |
| `knowledge:read` / `knowledge:write` | View vs. add/remove knowledge entries |
| `webhooks:write` | Manage outbound webhook subscriptions |
| `whatsapp:read` / `whatsapp:write` / `whatsapp:send` | Read vs. manage vs. send WhatsApp messaging |
| `*` | All resource scopes |

A key created for plain inference — the common case — needs only service permissions. Leave scopes empty.

## Limits on a key

Each key carries its own guardrails, so a key handed to one service or environment cannot spend or reach beyond what you intended.

| Setting | What it does |
|---------|--------------|
| **Budget** | A credit cap for the key. Once spent, calls on that key fail until you raise it — your account balance is untouched by other keys. |
| **RPM limit** | A per-key requests-per-minute ceiling, independent of your plan's overall quota. |
| **Allowed models** | Restricts the key to a named set of model ids, or `*` for everything your plan allows. |
| **Allowed search providers** | Restricts which providers `/v1/search` may use on this key. |
| **Domain allowlist** | Restricts which origins may send the key, for browser-side use. `*` allows any. |

## Logging

Logging is **off** by default on a new key and is opt-in per key:

- **Request logging** records metadata for each call — model, timing, token counts, credits spent.
- **Prompt logging** additionally stores message content. Enable it only when you need to inspect payloads.

## Rotating and revoking

Roll a key from the dashboard whenever it may have been exposed. Revocation takes effect on the next request; in-flight calls already authenticated are not retroactively cancelled. Because budgets, scopes and limits are per key, issuing one key per service or environment keeps a leak contained and makes usage attributable.
