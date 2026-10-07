---
title: "Idempotency"
description: "Make POST requests to the inference API safely retryable with an Idempotency-Key, so a network retry never runs or bills a call twice."
slug: "idempotency"
breadcrumb: "Getting Started"
---

# Idempotency

Make POST requests to the inference API safely retryable with an Idempotency-Key, so a network retry never runs or bills a call twice.

## Why Idempotency

If a request times out or the connection drops, you often can't tell whether the server processed it. Retrying blindly risks doing the work twice and paying for it twice. An **idempotency key** lets you retry safely: the server runs the first request and returns the same stored result for any replay with the same key.

## Where it applies

Idempotency keys are honoured on every `POST` under `/v1/` and `/anthropic/v1/` (chat completions, messages, responses, embeddings, speech, transcription, image generation, search, batches, voice sessions), authenticated with a `cm_` API key in either `Authorization: Bearer` or `x-api-key`. On other paths, other methods, or without an API key the header is ignored and the request runs normally. The one exception is the Email API: `POST /api/v1/email/send` (including `messageVersions` batches) accepts its own `Idempotency-Key` of up to 128 characters — see [Send Email](/docs/email-send).

## Using the Header

Send an `Idempotency-Key` header with a unique value of up to 255 characters (a UUID works well):

```bash
curl -X POST https://api.callmissed.com/v1/chat/completions \
  -H "Authorization: Bearer cm_your_key" \
  -H "Idempotency-Key: 3f9c1d2e-0b9a-4c7d-8e1f-2a3b4c5d6e7f" \
  -H "Content-Type: application/json" \
  -d '{"model":"kimi-k2.5","messages":[{"role":"user","content":"Hello"}]}'
```

Reuse the **same** key when retrying the same logical request. Use a **new** key for a genuinely new action. A key longer than 255 characters is ignored.

## Behavior

- Keys are scoped to your account and kept for **24 hours**.
- A replay with the same key, endpoint **and** body returns the original status and body with an `Idempotent-Replay: true` header. The call is not run or billed again.
- Only a successful (`2xx`) response is stored. After an error the key is released, so retrying with the same key runs the request again.
- Streaming (`text/event-stream`) and binary responses (audio, images), and responses over 1 MiB, are not stored. The key is released after them, so a retry runs again.

| Status | `detail` | When |
| --- | --- | --- |
| `409` | `Idempotency-Key reused with a different request body` | Same key, different body |
| `409` | `Idempotency-Key reused for a different request` | Same key sent to a different endpoint |
| `409` | `A request with this Idempotency-Key is already in progress` | The first request has not finished yet. Wait and retry |
| `409` | `... Its response was not retained (zero data retention), so it cannot be replayed.` | The original ran under zero data retention, so there is nothing to replay |
