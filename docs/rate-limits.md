---
title: "Rate Limits & Quotas"
description: "How CallMissed limits request rate — global per-IP limits, per-key RPM, response headers, and how to handle 429s."
slug: "rate-limits"
breadcrumb: "Getting Started"
---

# Rate Limits & Quotas

How CallMissed limits request rate — global per-IP limits, per-key RPM, response headers, and how to handle 429s.

## Limit Layers

Requests pass through several limits, in order:

| Layer | Limit | Scope |
| --- | --- | --- |
| Global middleware | 200 requests / minute | per IP |
| Auth endpoints | 5–10 requests / minute | per IP (login, register, refresh, OTP) |
| Per-key RPM | plan defaults: Free 60 · Starter 500 · Pro 3,000 · Enterprise 10,000 (override per key) | per API key |
| Monthly budget | configurable credit cap | per tenant / per key |
| Plan limits | tier-based caps on LLM/STT/TTS calls, conversations, storage, team size | per tenant |

Set a per-key RPM and a [budget cap](/docs/payments#budget) when issuing keys, and check live consumption with `GET /api/v1/keys/:id/rate-state`.

## Response Headers

Rate-limited responses include standard headers so you can pace requests:

| Header | Meaning |
| --- | --- |
| `Retry-After` | Seconds to wait before retrying (on 429) |
| `X-RateLimit-Limit` | The ceiling for the current window |
| `X-RateLimit-Remaining` | Requests left in the window |

## Handling 429

When you receive `429 Too Many Requests`:

1. Read `Retry-After` and wait at least that long.
2. Use exponential backoff with jitter for repeated 429s.
3. Spread bursty workloads across time, or request a higher per-key RPM.

See [Error Codes](/docs/errors) for the full status/code reference.