---
title: "Error Codes"
description: "Every HTTP status and error code the CallMissed API returns, what causes it, and how to recover."
slug: "errors"
breadcrumb: "Resources"
---

# Error Codes

Every HTTP status and error code the CallMissed API returns, what causes it, and how to recover.

## Error Format

The API returns errors in one of two shapes, depending on the surface. Neither ever contains an upstream provider's raw error or a stack trace.

**Inference endpoints** (`/v1/*`, `/anthropic/v1/*`) use the OpenAI error envelope, so existing SDK error handling works unchanged. `code` is stable and machine-readable; `message` is for humans.

```json
{
  "error": {
    "message": "Insufficient credits (balance: 0.0). Purchase more at https://console.callmissed.com/org/billing",
    "type": "insufficient_quota",
    "code": "insufficient_credits"
  }
}
```

**Platform endpoints** (`/api/v1/*`: agents, conversations, CRM, support, knowledge, webhooks, telephony and the rest) return a `detail` string:

```json
{ "detail": "API key missing required scope: contacts:write. Add it under the key's 'Permissions' section in your dashboard." }
```

A request that fails schema validation returns `422` with `detail` as a list of the fields that failed:

```json
{
  "detail": [
    { "loc": ["body", "k"], "msg": "Input should be less than or equal to 50", "type": "less_than_equal" }
  ]
}
```

## HTTP Status Codes

| Status | Meaning | Typical cause |
| --- | --- | --- |
| `200` / `201` / `202` / `204` | Success | `201` created, `202` accepted for async work, `204` deleted with no body |
| `400` | Bad Request | Invalid parameter value, unsupported option for this model |
| `401` | Unauthorized | Missing, unknown, revoked or expired API key |
| `402` | Payment Required | Out of credits, the key's own budget is spent, or the account has no payment method on file |
| `403` | Forbidden | Key lacks the permission, scope or model; origin not allowlisted; account inactive |
| `404` | Not Found | The id does not exist, or belongs to another account |
| `409` | Conflict | Duplicate resource, or an `Idempotency-Key` reused for a different request |
| `413` | Payload Too Large | Upload over the endpoint's size cap |
| `422` | Unprocessable Entity | Schema validation failed (bad enum, out-of-range number, missing field) |
| `429` | Too Many Requests | Per-key rate limit, monthly plan cap or budget cap, or too many requests in flight |
| `500` | Server Error | Unexpected failure. Safe to retry |
| `503` | Service Unavailable | The model or service failed upstream, timed out, or is temporarily unavailable or under maintenance. Carries `Retry-After`. Safe to retry |

## Error codes on the inference endpoints

| Code | Status | Meaning |
| --- | --- | --- |
| `invalid_api_key` | 401 | The `cm_` key is missing, unknown or revoked |
| `api_key_expired` | 401 | The key passed its expiry date. Issue a new one |
| `insufficient_credits` | 402 | Your credit balance is too low. Top up |
| `budget_exceeded` | 402 | This key's own budget is spent. Raise it on the key |
| `payment_method_required` | 402 | The account has no verified payment method. Add a card or UPI Autopay on the billing page of the console, then retry |
| `permission_denied` | 403 | The key lacks the service permission (`llm`, `stt`, `tts`, `image`, `search`) |
| `model_not_available` | 403 | The model needs a paid plan (the Claude models need Pro or higher) |
| `model_not_allowed` | 403 | The model is outside the key's allowed-models list |
| `search_provider_not_allowed` | 403 | The key's allowed search providers exclude the one requested |
| `domain_not_allowed` | 403 | The request origin is not in the key's domain allowlist |
| `account_inactive` / `account_terminated` | 403 | The account is not active. Contact support |
| `model_not_found` | 404 | No model with that id. See `GET /api/v1/models` |
| `context_length_exceeded` | 400 | The input is longer than the model's context window |
| `content_policy_violation` | 400 | An image-generation prompt was refused by the content policy. Rephrase it |
| `rate_limit_exceeded` | 429 | Over the key's requests-per-minute limit |
| `quota_exceeded` | 429 | The plan's monthly call cap for this service, or your monthly budget cap, is reached. `Retry-After` gives the seconds until the 1st |
| `too_many_concurrent_requests` | 429 | Too many requests from this key are still in flight. Retry after a few seconds |
| `upstream_error` / `provider_error` | 503 | The model failed to answer. Retry with backoff, or switch model |
| `model_under_maintenance` | 503 | The model is temporarily out of service |

## Notice header

While an account is inside its grace period for adding a payment method, successful responses carry `X-CallMissed-Notice: payment_method_required; deadline=YYYY-MM-DD; …`. Add a payment method before that date; after it, the same calls return `402 payment_method_required`.

## Retrying

- On **429** with `Retry-After`, wait at least that many seconds. Without the header, back off exponentially with jitter. A `quota_exceeded` 429 lasts until the 1st of next month, so upgrade or raise the cap rather than wait.
- On **500 and 503**, retry once or twice with jittered backoff. Send an [`Idempotency-Key`](/docs/idempotency) on inference `POST`s so a retry never runs or bills a call twice.
- On **401, 402 and 403**, do **not** retry. Fix the key, credits, permission or scope first.
