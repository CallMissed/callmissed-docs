---
title: "Webhooks"
description: "Create outbound webhook subscriptions, fire test deliveries, and inspect or replay the delivery log."
slug: "webhooks"
breadcrumb: "API Reference"
---

# Webhooks

Create outbound webhook subscriptions, fire test deliveries, and inspect or replay the delivery log.

> Subscribe an HTTPS endpoint to platform events. Every payload is signed with HMAC-SHA256 using the subscription's secret. Deliveries are logged, inspectable, and replayable.

## Credential class: dashboard JWT **or** a `cm_` key with `webhooks:write`

These routes accept either credential.

```
Authorization: Bearer <jwt_access_token>
# or
Authorization: Bearer cm_your_api_key
```

| Caller | Requirement |
| --- | --- |
| `cm_` API key | Must carry the `webhooks:write` scope. Every endpoint on this page checks it, **including the read-only list and delivery endpoints** - there is no `webhooks:read` scope |
| Dashboard JWT | Read endpoints work for any member. Create, update, delete, test, and replay require **owner or admin** (`403 Only owners/admins can manage webhooks` otherwise) |

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`.

## Event types

Pass these in the `events` array. An unrecognised value returns `422` listing the valid set.

| Category | Events |
| --- | --- |
| Conversations & messages | `conversation.started`, `conversation.ended`, `message.received`, `message.sent` |
| Voice sessions | `voice_session.started`, `voice_session.ended`, `voice_session.failed` |
| Telephony call lifecycle | `call.started`, `call.completed`, `call.failed` |
| Post-call analysis | `voice_analysis.completed` |
| Outbound campaigns | `campaign.started`, `campaign.completed`, `campaign.failed` |
| Metric alerts | `voice_alert.triggered` |
| Billing | `budget.alert`, `budget.exceeded`, `credits.low` |
| Keys | `api_key.expired` |
| Invoices & payments | `invoice.created`, `payment.succeeded`, `payment.failed` |

## Signing and verification

Every delivery, including the test delivery, carries:

```
Content-Type: application/json
X-CallMissed-Signature: sha256=<hex digest>
```

The digest is `HMAC-SHA256(secret, raw_request_body)`. Compute it over the **raw bytes** you received, before any JSON parsing, and compare with a constant-time function.

```python
import hashlib, hmac

def verify(raw_body: bytes, header: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", header)
```

The full `secret` is returned **only once**, in the `POST` create response. Every later read masks it as `first4 + "****" + last4`. Store it when you create the subscription.

## URL rules

The `url` is validated on create, on update, before a test delivery, and on every real delivery attempt. Endpoints resolving to private, loopback, link-local, or shared address space are rejected. Use a public HTTPS URL.

---

## GET /api/v1/webhooks

Lists your tenant's subscriptions, newest first. No parameters.

```bash
curl https://api.callmissed.com/api/v1/webhooks \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "id": "w1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8",
    "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
    "url": "https://hooks.acme.com/callmissed",
    "secret": "Xk4p****9dQz",
    "events": ["conversation.started", "message.received"],
    "is_active": true,
    "bot_id": null,
    "created_at": "2026-07-19T08:00:00Z"
  }
]
```

`403` when a `cm_` key lacks `webhooks:write`.

## POST /api/v1/webhooks

Creates a subscription and generates its secret. Returns `201`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `url` | `string` | Yes | 1-2048 chars. Must pass the URL rules above |
| `events` | `string[]` | Yes | At least one entry, each from the event table |
| `bot_id` | `UUID \| null` | No | Scope the subscription to one agent. Omit for a workspace-wide subscription |

```bash
curl -X POST https://api.callmissed.com/api/v1/webhooks \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://hooks.acme.com/callmissed",
    "events": ["conversation.started", "message.received", "payment.succeeded"],
    "bot_id": null
  }'
```

```json
{
  "id": "w1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "url": "https://hooks.acme.com/callmissed",
  "secret": "Xk4pR7tYb2Lm8sNcVhJ3wQeZ1aFgD6uO9dQz",
  "events": ["conversation.started", "message.received", "payment.succeeded"],
  "is_active": true,
  "bot_id": null,
  "created_at": "2026-08-04T10:40:00Z"
}
```

This response is the **only** place the full `secret` appears.

| Status | Cause |
| --- | --- |
| `400` | The URL failed validation |
| `403` | Key missing `webhooks:write`, or JWT caller is not owner/admin |
| `404` | `Agent not found` when `bot_id` is not an agent in your tenant |
| `422` | Unknown event type, empty `events`, or `url` over 2048 chars |

## PUT `/api/v1/webhooks/{webhook_id}`

Partial update. Every field is optional; omitted fields are unchanged. Returns the subscription with a **masked** secret.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `url` | `string \| null` | No | 1-2048 chars. Re-validated when present |
| `events` | `string[] \| null` | No | Replaces the whole list |
| `is_active` | `boolean \| null` | No | Pause or resume deliveries |
| `bot_id` | `UUID \| null` | No | Re-scope to an agent in your tenant |
| `clear_bot_id` | `boolean` | No | Default `false`. Send `true` to widen a scoped subscription back to the whole workspace. Takes precedence over `bot_id` |

```bash
curl -X PUT https://api.callmissed.com/api/v1/webhooks/w1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8 \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"events": ["message.received"], "is_active": false}'
```

`404` `Webhook not found`; `404` `Agent not found` for a foreign `bot_id`; `403`, `400`, and `422` as for create.

## DELETE `/api/v1/webhooks/{webhook_id}`

Returns `204` with an empty body. Writes a `webhook.delete` [audit event](/docs/audit-logs).

```bash
curl -X DELETE https://api.callmissed.com/api/v1/webhooks/w1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8 \
  -H "Authorization: Bearer cm_your_api_key"
```

`403` for insufficient permission; `404` when the subscription is not in your tenant.

## POST `/api/v1/webhooks/{webhook_id}/test`

Sends a real signed POST to the configured URL and reports the result. No request body. The stored URL is re-validated first, so a hostname repointed at a private address after creation is rejected.

The delivered payload:

```json
{
  "event": "test",
  "timestamp": "2026-08-04T10:45:12.004921+00:00",
  "data": { "message": "This is a test webhook delivery from CallMissed" }
}
```

```bash
curl -X POST https://api.callmissed.com/api/v1/webhooks/w1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8/test \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "success": true,
  "status_code": 200,
  "latency_ms": 214,
  "error": null
}
```

`success` is true only for a 2xx from your endpoint. On a network failure the response is still `200` with `success: false`, `status_code: null`, and a short `error` string (truncated to 500 chars). The 10-second request timeout applies.

`403` for insufficient permission; `404` when the subscription is not in your tenant.

## GET `/api/v1/webhooks/{webhook_id}/deliveries`

Delivery log for one subscription, newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |

```bash
curl "https://api.callmissed.com/api/v1/webhooks/w1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8/deliveries?limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "id": "d5e6f708-1920-4a3b-8c4d-5e6f708192a3",
    "webhook_id": "w1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8",
    "event": "message.received",
    "status_code": 500,
    "attempts": 3,
    "success": false,
    "error": "HTTP 500",
    "created_at": "2026-08-04T09:12:00Z",
    "delivered_at": null
  }
]
```

`status_code` and `error` are `null` when not applicable; `delivered_at` is `null` until a delivery succeeds. The payload body is **not** in this list response - fetch one delivery for that.

`403` when a `cm_` key lacks `webhooks:write`; `404` when the subscription is not in your tenant; `422` when `limit` is outside `1 .. 200`.

## GET `/api/v1/webhooks/{webhook_id}/deliveries/{delivery_id}`

Full inspector view of one delivery, including the stored payload.

```bash
curl https://api.callmissed.com/api/v1/webhooks/w1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8/deliveries/d5e6f708-1920-4a3b-8c4d-5e6f708192a3 \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "id": "d5e6f708-1920-4a3b-8c4d-5e6f708192a3",
  "webhook_id": "w1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8",
  "event": "message.received",
  "status_code": 500,
  "attempts": 3,
  "success": false,
  "error": "HTTP 500",
  "payload": {
    "event": "message.received",
    "data": { "conversation_id": "c0ffee00-1111-2222-3333-444455556666", "role": "user" }
  },
  "created_at": "2026-08-04T09:12:00+00:00",
  "delivered_at": null
}
```

| Status | Cause |
| --- | --- |
| `403` | Key missing `webhooks:write` |
| `404` | `Webhook not found` or `Delivery not found` |
| `422` | Either path id is not a valid UUID |

## POST `/api/v1/webhooks/{webhook_id}/deliveries/{delivery_id}/replay`

Re-sends a past delivery's payload against the subscription's **current** URL and secret. No request body. A new delivery row is created; the original is preserved. Returns `202`.

```bash
curl -X POST https://api.callmissed.com/api/v1/webhooks/w1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8/deliveries/d5e6f708-1920-4a3b-8c4d-5e6f708192a3/replay \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "delivery_id": "a7b8c9d0-e1f2-4304-8516-27a8b9c0d1e2",
  "status": "queued"
}
```

`status: "queued"` means the replay was accepted, not that it succeeded. Poll the deliveries list for the new row's outcome. Writes a `webhook.delivery.replay` audit event.

| Status | Cause |
| --- | --- |
| `400` | `Webhook is inactive` - re-enable it with `PUT` first |
| `403` | Key missing `webhooks:write`, or JWT caller is not owner/admin |
| `404` | `Webhook not found` or `Delivery not found` |
