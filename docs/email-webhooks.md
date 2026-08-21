---
title: "Email Webhooks"
description: "Subscribe your own endpoint to email events: bounces, complaints, delivery and engagement, signed with HMAC-SHA256 and logged per attempt."
slug: "email-webhooks"
breadcrumb: "Email"
---

# Email Webhooks

Subscribe your own endpoint to email events: bounces, complaints, delivery and engagement, signed with HMAC-SHA256 and logged per attempt.

## Overview

Register an HTTPS endpoint and we POST each email event to it as it happens, signed with a per-subscription secret. This is how you learn about a bounce or a spam complaint without polling the [send log](/docs/email-logs).

These webhooks are scoped to **your own email events** and are managed with the same `cm_` key you send with. Up to **20 subscriptions** per account.

| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/email/webhooks` | Create a subscription (`201`). The only response that carries the signing secret |
| `GET /api/v1/email/webhooks` | List your subscriptions, newest first |
| `GET /api/v1/email/webhooks/deliveries` | The delivery log: every attempt, its result and its error |
| `PATCH /api/v1/email/webhooks/{id}` | Enable or disable without losing the URL or the secret |
| `DELETE /api/v1/email/webhooks/{id}` | Remove the subscription (`204`) |

Creating, updating and deleting need a write key. Listing and the delivery log are reads, so a read-only key works.

## Create a subscription

**`POST /api/v1/email/webhooks`**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `url` | string | Yes | Your endpoint, 1–2048 chars. Must be a public `http`/`https` URL; an internal or private target is refused at creation with `422 webhook_url_forbidden` |
| `description` | string | No | Your own label, up to 255 chars |
| `events` | array | No | Which events to receive. Omit it (or send an empty list) to receive **every** event. An unknown name is a `422` listing the supported set |

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/email/webhooks \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-app.com/hooks/email",
    "description": "Bounce + complaint handler",
    "events": ["email.bounced", "email.complained"]
  }'
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/email"
h = {"Authorization": "Bearer cm_your_key"}

created = httpx.post(f"{BASE}/webhooks", headers=h, json={
    "url": "https://your-app.com/hooks/email",
    "description": "Bounce + complaint handler",
    "events": ["email.bounced", "email.complained"],
}).json()

secret = created["secret"]          # store this now: it is never returned again
webhook_id = created["webhook"]["id"]
```
```javascript [JavaScript]
const BASE = "https://api.callmissed.com/api/v1/email";
const headers = {
  Authorization: "Bearer cm_your_key",
  "Content-Type": "application/json",
};

const created = await fetch(`${BASE}/webhooks`, {
  method: "POST",
  headers,
  body: JSON.stringify({
    url: "https://your-app.com/hooks/email",
    description: "Bounce + complaint handler",
    events: ["email.bounced", "email.complained"],
  }),
}).then((r) => r.json());

const secret = created.secret; // store this now: it is never returned again
```
:::

The `201` response wraps the subscription and the secret:

```json
{
  "webhook": {
    "id": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
    "url": "https://your-app.com/hooks/email",
    "description": "Bounce + complaint handler",
    "events": ["email.bounced", "email.complained"],
    "secret_prefix": "whsec_A1b2C3",
    "is_active": true,
    "created_at": "2026-08-13T09:41:02.118Z"
  },
  "secret": "whsec_A1b2C3d4E5f6..."
}
```

**The `secret` is returned once, here, and never again.** Every later read exposes only `secret_prefix`. Store it when you create the subscription; if you lose it, delete the subscription and create a new one.

| `WebhookOut` field | Type | Notes |
|--------------------|------|-------|
| `id` | string (UUID) | Use it with `PATCH` / `DELETE` and as the `webhook_id` filter on the delivery log |
| `url` | string | Where we POST |
| `description` | string \| null | Your label |
| `events` | array \| null | The subscribed events. `null` means every event |
| `secret_prefix` | string | The first characters of the secret, for identifying which secret a subscription holds |
| `is_active` | boolean | `false` stops deliveries; the URL and secret are kept |
| `created_at` | string | When it was registered |

## Events

| Event | Fires when | Status |
|-------|-----------|--------|
| `email.bounced` | A recipient's mail server rejected the message. The address is also added to your [suppression list](/docs/email-logs#suppressions) | **Live** |
| `email.complained` | A recipient marked the message as spam. The address is suppressed too | **Live** |
| `email.sent` | The message was accepted for delivery | Subscribable; not emitted yet |
| `email.delivered` | Delivery to the recipient's mailbox was confirmed | Subscribable; not emitted yet |
| `email.opened` | A tracked message was opened | Subscribable; not emitted yet |
| `email.received` | Inbound mail arrived at one of your receiving addresses | Subscribable; not emitted yet |

You can subscribe to any of the six today. The four marked *not emitted yet* are accepted so your subscription does not have to be rewritten when they start firing — until then they simply deliver nothing. For inbound mail right now, use the per-address `forward_url` on [Receive Email](/docs/email-inbound), which is live; for opens and clicks, read the aggregates from [engagement metrics](/docs/email-logs#engagement-metrics).

### Payload

Every delivery is a POST with this envelope:

```json
{
  "type": "email.bounced",
  "created_at": "2026-08-13T09:41:02.118431+00:00",
  "data": {
    "email_id": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
    "message_id": "<1a2b3c4d@acme.com>",
    "recipient": "customer@example.com",
    "detail": "550 5.1.1 recipient address rejected",
    "domain": "acme.com"
  }
}
```

| Field | Notes |
|-------|-------|
| `type` | The event name |
| `created_at` | When we generated the event, ISO 8601 UTC. This value **is** covered by the signature, so it is the timestamp to trust for an age check |
| `data.email_id` | The send id from `POST /send`; `null` if the event could not be matched to a send |
| `data.message_id` | The RFC 5322 `Message-ID` |
| `data.recipient` | The address that bounced or complained |
| `data.detail` | The reported reason, when one was given |
| `data.domain` | Your sending domain the message went out on |

`email.bounced` and `email.complained` carry the shape above.

### Headers

```
Content-Type: application/json
X-CallMissed-Signature: sha256=<hex digest>
X-CallMissed-Event: email.bounced
X-CallMissed-Delivery: 3f1c9a2d-4b5e-6a7f-8c9d-0e1f2a3b4c5d
```

`X-CallMissed-Delivery` is the delivery id, so a row in the [delivery log](#delivery-log) can be matched to the request your handler saw. Use it to make your handler idempotent: a retried delivery reuses the same id.

## Verifying the signature

The digest is `HMAC-SHA256(secret, raw_request_body)`. Compute it over the **raw bytes** you received, before any JSON parsing, and compare with a constant-time function. Re-serialising the parsed JSON will not reproduce the signed bytes.

:::tabs
```python [Python]
import hashlib, hmac

def verify(raw_body: bytes, header: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", header)
```
```javascript [JavaScript]
import crypto from "node:crypto";

function verify(rawBody, header, secret) {
  const expected =
    "sha256=" + crypto.createHmac("sha256", secret).update(rawBody).digest("hex");
  const a = Buffer.from(expected);
  const b = Buffer.from(header ?? "");
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```
:::

There is deliberately **no timestamp header**: a timestamp the signature does not cover could be rewritten, so an age check based on it would not be trustworthy. Read `created_at` from the signed body instead.

## Delivery behaviour

- **Retries.** A non-2xx response or a transport error is retried up to **3 attempts** with exponential backoff. Anything in the 2xx range counts as success, so answer `200` as soon as you have accepted the event and do your work afterwards.
- **Timeout.** Each attempt allows 10 seconds for a response.
- **Redirects are not followed.** Point the subscription at its final URL.
- **The URL is re-checked on every attempt.** A URL that resolves to a private or internal address at send time is refused and the delivery is marked `blocked` rather than retried, even if it passed validation when you created the subscription.
- **Events are not delayed by your endpoint.** Delivery runs outside the request that produced the event, so a slow handler never slows a send or the processing of a bounce.
- **Order is not guaranteed.** Use `created_at` from the payload if you need to sequence events.

## Delivery log

**`GET /api/v1/email/webhooks/deliveries`** returns every attempt chain, newest first, so you can tell "we never sent it" from "my handler returned 500".

| Query param | Notes |
|-------------|-------|
| `webhook_id` | Optional UUID; restrict the log to one subscription |
| `limit` | 1–200, default 50 |
| `offset` | ≥0, default 0 |

| `WebhookDeliveryOut` field | Type | Notes |
|----------------------------|------|-------|
| `id` | string (UUID) | Matches the `X-CallMissed-Delivery` header your handler received |
| `webhook_id` | string (UUID) | Which subscription this went to |
| `event` | string | The event name |
| `send_id` | string (UUID) \| null | The send the event was about, when it could be matched |
| `status` | string | `pending`, `delivered`, `failed` or `blocked` — see below |
| `attempt_count` | integer | How many POSTs were made. `0` on a `blocked` row, because no connection was opened |
| `response_code` | integer \| null | The last HTTP status your endpoint returned; `null` on a transport error |
| `error_detail` | string \| null | Why the last attempt failed |
| `created_at` | string | When the event was generated |
| `last_attempt_at` | string \| null | When we last tried |
| `delivered_at` | string \| null | When your endpoint accepted it |

| `status` | Meaning |
|----------|---------|
| `pending` | Created, not yet attempted |
| `delivered` | Your endpoint answered 2xx |
| `failed` | Non-2xx or a transport error, and the retries are exhausted |
| `blocked` | The URL resolved somewhere we refuse to POST to, so no request was sent |

```bash
# Everything, newest first
curl "https://api.callmissed.com/api/v1/email/webhooks/deliveries?limit=50&offset=0" \
  -H "Authorization: Bearer cm_your_key"

# Just one subscription
curl "https://api.callmissed.com/api/v1/email/webhooks/deliveries?webhook_id=9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e" \
  -H "Authorization: Bearer cm_your_key"
```

## Pause, resume and delete

**`PATCH /api/v1/email/webhooks/{id}`** takes one field, `is_active` (boolean), and returns the updated `WebhookOut`. This is how you stop a noisy endpoint without re-registering and redeploying a new secret.

```bash
# Stop deliveries, keep the URL, secret and history
curl -X PATCH https://api.callmissed.com/api/v1/email/webhooks/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"is_active": false}'

# Resume
curl -X PATCH https://api.callmissed.com/api/v1/email/webhooks/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"is_active": true}'

# Remove it entirely
curl -X DELETE https://api.callmissed.com/api/v1/email/webhooks/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e \
  -H "Authorization: Bearer cm_your_key"
```

`DELETE` is a hard delete and **also removes that subscription's delivery history**, so a deleted subscription leaves none of your payloads behind. To stop deliveries while keeping the audit trail, `PATCH is_active=false` instead.

## Common failures on these routes

| Status | Body | Meaning |
|--------|------|---------|
| 401 | string `detail` | Missing, malformed or unrecognised `Authorization` header |
| 403 | string `detail` | The API key is read-only and this route writes |
| 404 | string `detail` | The webhook id is not yours |
| 422 | `reason: webhook_url_forbidden` | The `url` is not a permitted public URL |
| 422 | `reason: too_many_webhooks` | You already have 20 subscriptions |
| 422 | schema array `detail` | An unknown event name in `events` |

The two `reason` bodies are nested under `detail` alongside an `error` string. Every shape is spelled out on [Limits, Quotas & Errors](/docs/email-limits#response-shapes).

## Related

- [Delivery, Suppressions & Usage](/docs/email-logs) for the send log, the suppression list and engagement metrics.
- [Receive Email](/docs/email-inbound) for inbound mail, which is forwarded per address rather than through this subsystem.
- [Domains & Senders](/docs/email-domains#tracking-and-sending-toggles) to turn open and click tracking on.
