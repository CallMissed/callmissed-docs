---
title: "Idempotency"
description: "Make mutating requests safely retryable with an Idempotency-Key so network retries never duplicate an action."
slug: "idempotency"
breadcrumb: "Getting Started"
---

# Idempotency

Make mutating requests safely retryable with an Idempotency-Key so network retries never duplicate an action.

## Why Idempotency

If a request times out or the connection drops, you often can't tell whether the server processed it. Retrying blindly risks doing the action twice — charging a card twice, creating two bots. An **idempotency key** lets you retry safely: the server processes the first request and returns the same stored result for any replay with the same key.

## Using the Header

Send an `Idempotency-Key` header with a unique value (a UUID works well) on any mutating request (`POST`, `PUT`, `PATCH`):

```bash
curl -X POST https://api.callmissed.com/api/v1/bots \
  -H "Authorization: Bearer cm_your_key" \
  -H "Idempotency-Key: 3f9c1d2e-0b9a-4c7d-8e1f-2a3b4c5d6e7f" \
  -H "Content-Type: application/json" \
  -d '{"name":"Support Bot","type":"whatsapp"}'
```

Reuse the **same** key when retrying the same logical request. Use a **new** key for a genuinely new action.

## Behavior

- A replay with the same key **and** the same body returns the original response — the action runs only once.
- A replay with the same key but a **different** body returns `409 Conflict`.
- Keys are scoped to your tenant and retained for a limited window, then expire.
- Idempotency is most important for [payments](/docs/payments) and resource-creation endpoints.