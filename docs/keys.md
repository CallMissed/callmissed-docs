---
title: "API Keys"
description: "Create and manage API keys with granular scopes and request logging."
slug: "keys"
breadcrumb: "API Reference"
---

# API Keys

Create and manage API keys with granular scopes and request logging.

> API keys are shown in plaintext **once** at creation. Each key carries two access controls: **service permissions** — which AI services it can call (`llm`, `stt`, `tts`, `search`, `image`, or `*` for all) — and **scopes** — which platform resources it can touch (`bots:read`, `bots:write`, `conversations:read`, `conversations:write`, `knowledge:read`, `knowledge:write`, `webhooks:write`, `whatsapp:read`, `whatsapp:write`, `whatsapp:send`). A key also supports a **domain allowlist**, a per-key **RPM** limit, a **budget** cap, allowed **models**, and allowed **search providers**. Request and prompt logging are opt-in per key.

## Create a key

Key management uses your **dashboard session (JWT)**. This example mints an LLM-only key that can read conversations, capped at 200 req/min and a 5,000-credit budget:

```bash
curl https://api.callmissed.com/api/v1/keys \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Production LLM key",
    "permissions": "llm",
    "scopes": "conversations:read",
    "rate_limit_rpm": 200,
    "budget_limit": 5000
  }'
```

The plaintext `key` is returned **once** — store it now:

```json
{
  "id": "k1234567-89ab-cdef-0123-456789abcdef",
  "name": "Production LLM key",
  "key": "cm_live_3f9a...redacted",
  "permissions": "llm",
  "scopes": "conversations:read",
  "allowed_domains": "*",
  "is_active": true,
  "logs_enabled": false,
  "prompt_logging_enabled": false,
  "budget_limit": 5000.0,
  "budget_used": 0.0,
  "budget_remaining": 5000.0,
  "rate_limit_rpm": 200,
  "allowed_models": "*",
  "created_at": "2026-06-06T12:15:00Z"
}
```

If you lose it later, `POST /api/v1/keys/:id/reveal/request` emails an OTP, then `.../reveal/verify` returns the stored plaintext. New keys have **logging off** by default — flip `logs_enabled` (and `prompt_logging_enabled` for message content) only when you need it.