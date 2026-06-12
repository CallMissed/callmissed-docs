---
title: "Error Codes"
description: "Every HTTP status and error code the CallMissed API returns, what causes it, and how to recover."
slug: "errors"
breadcrumb: "Resources"
---

# Error Codes

Every HTTP status and error code the CallMissed API returns, what causes it, and how to recover.

## Error Format

All errors return a JSON body with a stable machine-readable `code` and a human message. We never leak upstream provider errors or stack traces to clients.

```json
{
  "error": {
    "code": "insufficient_credits",
    "message": "Your credit balance is too low to complete this request.",
    "type": "billing_error"
  }
}
```

The OpenAI-compatible endpoints (`/v1/*`) return the standard OpenAI error envelope so existing SDK error handling works unchanged.

## HTTP Status Codes

| Status | Meaning | Typical cause |
| --- | --- | --- |
| `200` | OK | Success |
| `400` | Bad Request | Malformed JSON, invalid parameter value |
| `401` | Unauthorized | Missing/invalid `Authorization` header or expired token |
| `402` | Payment Required | Insufficient credits or monthly budget cap reached |
| `403` | Forbidden | Key lacks the required scope/permission, or domain not allowlisted |
| `404` | Not Found | Resource ID does not exist or belongs to another tenant |
| `409` | Conflict | Duplicate resource, or replayed `Idempotency-Key` with a different body |
| `422` | Unprocessable Entity | Schema validation failed (bad enum, out-of-range number) |
| `429` | Too Many Requests | Per-IP or per-key rate limit exceeded — back off and retry |
| `500` | Internal Server Error | Unexpected server error — safe to retry once |
| `501` | Not Implemented | Endpoint exists but the feature is not yet live (e.g. embeddings) |
| `503` | Service Unavailable | Upstream provider temporarily unavailable |

## Common Error Codes

| Code | Status | Meaning |
| --- | --- | --- |
| `invalid_api_key` | 401 | The `cm_` key is unknown or revoked |
| `token_expired` | 401 | JWT access token expired — refresh it |
| `insufficient_credits` | 402 | Top up credits to continue |
| `budget_exceeded` | 402 | Monthly credit budget cap reached |
| `permission_denied` | 403 | Key is missing the required service permission |
| `search_provider_not_allowed` | 403 | Key's allowed web-search providers excludes the requested provider |
| `domain_not_allowed` | 403 | Request origin is not in the key's domain allowlist |
| `not_found` | 404 | Resource does not exist in your tenant |
| `rate_limit_exceeded` | 429 | Slow down — see `Retry-After` |
| `provider_error` | 503 | Upstream model/provider failed |

## Retrying

- On **429**, honor the `Retry-After` header (seconds) and use exponential backoff.
- On **500/503**, retry once or twice with jittered backoff. To make a mutating request safely retryable, send an `Idempotency-Key` header — replays with the same key and body return the original result instead of duplicating the action.
- On **402/403**, do **not** retry — fix the underlying credit/permission issue first.