---
title: "Data Retention & End-User Budgets"
description: "Restrict requests to zero-data-retention routes, and cap what each of your own end users can spend per month."
slug: "gateway-controls"
breadcrumb: "API Reference"
---

# Data Retention & End-User Budgets

Restrict requests to zero-data-retention routes, and cap what each of your own end users can spend per month.

## Overview

Two controls apply to `POST /v1/chat/completions`, `POST /v1/responses`, `POST /v1/messages` and `POST /v1/embeddings`:

- **Zero data retention (ZDR)** — serve a request only on a route whose model provider does not retain prompts or responses, and store no prompt or response content on our side.
- **Per-end-user budgets** — a monthly credit cap per end user of your application, keyed on the `user` field you already send.

## Zero data retention

### Turning it on

Send `provider.zdr` in the request body:

```json
{
  "model": "your-model-id",
  "messages": [{ "role": "user", "content": "Summarise this contract." }],
  "provider": { "zdr": true }
}
```

`provider.zdr` must be a JSON boolean; any other value returns `422`. On `/v1/messages` the same `provider` object is accepted as a CallMissed extension.

ZDR can also be enforced without the flag:

- **Per API key** — turn on *Zero data retention* on the key in the dashboard.
- **Account-wide** — turn on the data-retention policy in the dashboard (owners and admins).

When the key or account policy is on, every request is treated as ZDR. A request can switch ZDR on, never off: `"zdr": false` does not override a key or account policy.

### Which models qualify

Every entry in `GET /v1/models` carries a boolean `zero_data_retention`. It is `true` only when the route serving that model is backed by the provider's own published zero-retention terms. Today no model in the catalogue is marked `true`. A ZDR request is refused rather than served on a route that retains data.

### When no route qualifies

A ZDR request for a model with `zero_data_retention: false` is rejected before any model is called and is not billed:

```json
{
  "error": {
    "message": "Model 'your-model-id' has no zero-data-retention route. ...",
    "type": "invalid_request_error",
    "code": "zdr_unavailable"
  }
}
```

The request is never downgraded to a route that retains data, including through fallbacks: models in your `models` fallback list that have no zero-data-retention route are skipped.

On `/v1/chat/completions`, `bot_id` knowledge-base retrieval also returns `zdr_unavailable` under ZDR, because retrieval sends your latest message to an embedding model.

The [Batch API](/docs/batch) also returns `zdr_unavailable` under ZDR (at upload and at batch create), because a batch stores its input and output files.

### What we do not store for ZDR requests

- **Prompt and response logging** is skipped, even when prompt logging is enabled on the key.
- **The response cache** is neither read nor written.
- **`Idempotency-Key`** (non-streaming requests): the response body is not stored. A retry with the same key returns `409` instead of a replay, so the call is not run or billed a second time.

Usage metering (model, token counts, cost, latency) is still recorded so the call can be billed.

## Per-end-user budgets

### How the cap is applied

Pass your end user's id in the request:

| Endpoint | Field |
| --- | --- |
| `/v1/chat/completions`, `/v1/responses` | `user` (a cap can only be set on an id of at most 256 characters; a longer id is accepted and uncapped) |
| `/v1/embeddings` | `user` (at most 256 characters) |
| `/v1/messages` | `metadata.user_id` (at most 512 characters; a cap can only be set on an id of at most 256) |

If that id has a cap and has already spent its monthly limit, the request is rejected before any model is called:

```json
{
  "error": {
    "message": "Monthly budget for this end user (the `user` field) is exhausted. ...",
    "type": "insufficient_quota",
    "code": "end_user_budget_exceeded"
  }
}
```

The status is `402`. On `/v1/messages` the error uses the Anthropic shape with type `billing_error`.

- Spend is counted per calendar month (UTC) and resets at the start of each month.
- The cap is checked before each call, so requests already in flight when the limit is reached can take the total slightly past it.
- A limit of `0` blocks that end user.
- Ids without a cap are not limited and not tracked.
- Ids are matched exactly and are case-sensitive.

### Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List | `end_user_budgets:read` |
| Create or update, delete | `end_user_budgets:write` |

### GET `/api/v1/gateway/end-user-budgets`

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `end_user` | `string` | No | Exact id, at most 256 characters |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `100` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```json
[
  {
    "id": "5c1d…",
    "end_user": "customer-42",
    "monthly_limit_credits": 50.0,
    "period_start": "2026-10-01T00:00:00Z",
    "period_used_credits": 12.5,
    "remaining_credits": 37.5,
    "created_at": "2026-09-14T08:00:00Z",
    "updated_at": "2026-10-01T09:30:00Z"
  }
]
```

`period_used_credits` is this month's spend. It reads `0` if the id has not been charged yet this month.

### PUT `/api/v1/gateway/end-user-budgets`

Creates the cap, or changes the limit if one exists. Spend already recorded this month is kept.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `end_user` | `string` | Yes | 1–256 printable characters, stored exactly as sent |
| `monthly_limit_credits` | `number` | Yes | `0 <= value <= 10000000` |

```bash
curl -X PUT https://api.callmissed.com/api/v1/gateway/end-user-budgets \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"end_user": "customer-42", "monthly_limit_credits": 50}'
```

The response is one budget object in the same shape as the list items.

### DELETE `/api/v1/gateway/end-user-budgets/{id}`

Removes the cap. The id becomes uncapped and its spend counter is discarded. Returns `204`, or `404` for an id that is not in your account.
