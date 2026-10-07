---
title: "Rate Limits & Quotas"
description: "How CallMissed limits request rate — per-key RPM, plan caps, budget caps, response headers, and how to handle 429s."
slug: "rate-limits"
breadcrumb: "Getting Started"
---

# Rate Limits & Quotas

How CallMissed limits request rate — per-key RPM, plan caps, budget caps, response headers, and how to handle 429s.

## Limit Layers

Requests pass through several limits:

| Layer | Limit | Scope | When exceeded |
| --- | --- | --- | --- |
| Per-key RPM | Plan defaults: Free 60 · Starter 500 · Pro 3,000 · Enterprise 10,000 (override per key) | per API key | `429 rate_limit_exceeded` |
| In-flight requests | A cap on simultaneous chat-completion requests per key | per API key | `429 too_many_concurrent_requests` |
| Key budget | Optional credit cap on one key | per API key | `402 budget_exceeded` |
| Monthly budget cap | Optional credit cap for the whole account | per account | `429 quota_exceeded` until the 1st |
| Plan call caps | Monthly caps on LLM/STT/TTS/image calls, see [Credits & Rate Limits](/docs/credits-rate-limits#plan-call-caps) | per account | `429 quota_exceeded` until the 1st |

Abuse protection also runs in front of the API and may throttle traffic that looks automated or hostile, independently of your plan's per-key RPM.

Set a per-key RPM and a [budget cap](/docs/keys) when issuing keys, then track live consumption for each key from the console.

## Response Headers

| Header | Sent on | Meaning |
| --- | --- | --- |
| `X-RateLimit-Limit` | Metered inference responses | The monthly call cap for that service |
| `X-RateLimit-Remaining` | Metered inference responses | Calls left this month |
| `X-RateLimit-Reset` | Metered inference responses | ISO-8601 time the monthly cap resets |
| `X-Usage-Warning` | At 80% and 95% of a monthly cap | `warning: …` or `critical: …` with the count used |
| `Retry-After` | `quota_exceeded`, `too_many_concurrent_requests`, and the per-key RPM 429 on `/api/v1/*` | Seconds to wait before retrying |

The `X-RateLimit-*` headers describe the **monthly plan cap**, not the per-minute limit. A per-key RPM 429 from an inference endpoint (`/v1/*`) carries no `Retry-After`: the window is a rolling minute, so back off and retry within it.

## Handling 429

Read the error `code` first (on `/api/v1/*` routes, the `detail` message):

1. `rate_limit_exceeded`: wait (`Retry-After` when present, otherwise exponential backoff with jitter, starting at about a second) and retry. Spread bursts over time, or raise the key's RPM.
2. `too_many_concurrent_requests`: wait the `Retry-After` seconds and retry. Lower your client's parallelism.
3. `quota_exceeded`: do not retry. The cap lasts until the 1st of next month. Upgrade the plan or raise the budget cap.

See [Error Codes](/docs/errors) for the full status/code reference.
