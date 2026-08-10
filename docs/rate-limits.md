---
title: "Rate Limits & Quotas"
description: "How CallMissed limits request rate — per-key RPM, budget caps, response headers, and how to handle 429s."
slug: "rate-limits"
breadcrumb: "Getting Started"
---

# Rate Limits & Quotas

How CallMissed limits request rate — per-key RPM, budget caps, response headers, and how to handle 429s.

## Limit Layers

Requests pass through several limits, in order:

| Layer | Limit | Scope |
| --- | --- | --- |
| Per-key RPM | plan defaults: Free 60 · Starter 500 · Pro 3,000 · Enterprise 10,000 (override per key) | per API key |
| Monthly budget | configurable credit cap | per tenant / per key |
| Plan limits | tier-based caps on LLM/STT/TTS calls, conversations, storage, team size | per tenant |

Abuse protection also runs in front of the API and may throttle traffic that looks automated or hostile, independently of your plan's per-key RPM.

Set a per-key RPM and a [budget cap](/docs/keys) when issuing keys, then track live consumption for each key from the dashboard.

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
