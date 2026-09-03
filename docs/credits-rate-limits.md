---
title: "Credits & Rate Limits"
description: "How CallMissed credits are priced and spent, the per-plan call caps and request rates, and how to handle 402 and 429."
slug: "credits-rate-limits"
breadcrumb: "Getting Started"
---

# Credits & Rate Limits

How CallMissed credits are priced and spent, the per-plan call caps and request rates, and how to handle 402 and 429.

## Credits

One currency for every service. **1 credit = ₹1 = $0.01.**

Every call deducts credits: LLM tokens, STT audio, TTS characters, an image, a
web search. Credits do not expire.

| Grant | Amount |
|-------|--------|
| Signup bonus (once per account) | 1,000 credits |
| Free plan, monthly | 100 credits |
| Starter, monthly | 550 credits |
| Pro, monthly | 6,000 credits |
| Enterprise, monthly | 26,000 credits |

Usage is always metered. There is no unlimited tier — Enterprise removes the
monthly call caps, not the per-call credit cost.

## How each service is metered

| Service | Unit | Worked example |
|---------|------|----------------|
| LLM | per 1M input + 1M output tokens | `kimi-k2.5` at $0.81 in / $4.05 out: 500 in + 200 output tokens = $0.001215 = **0.1215 credits** |
| Speech to text | per audio hour | `saaras:v3` at $0.30/hr: a 4-minute call = **2 credits** |
| Text to speech | per 10,000 characters | `bulbul:v3` at $0.30/10K: a 400-character reply = **1.2 credits** |
| Image generation | per image | `flux-2-klein-9b` at $0.10: one image = **10 credits** |
| Web search | flat | **1 credit** per search, whichever provider serves it |

Per-model rates are in the [model catalog](/docs/models#pricing) and live at
`GET /api/v1/models`.

## Plan call caps

Monthly caps counted per service, reset on the 1st. `-1` means uncapped.

| Plan | LLM | STT | TTS | Image | Conversations | Storage | Team |
|------|-----|-----|-----|-------|---------------|---------|------|
| Free | 100 | 50 | 50 | 50 | 50 | 100 MB | 2 |
| Starter | 5,000 | 2,500 | 2,500 | 500 | 1,000 | 1 GB | 5 |
| Pro | 50,000 | 25,000 | 25,000 | 5,000 | 10,000 | 10 GB | 20 |
| Enterprise | uncapped | uncapped | uncapped | uncapped | uncapped | uncapped | uncapped |

Caps are separate from credits. Exceeding a cap returns `429` even with credits
in the balance; running out of credits returns `402` even under the cap.

## Request rate

Per API key, requests per minute:

| Plan | Default RPM |
|------|-------------|
| Free | 60 |
| Starter | 500 |
| Pro | 3,000 |
| Enterprise | 10,000 |

Override a single key with `rate_limit_rpm` in the dashboard. An explicit
override wins over the plan default.

## Response headers

Every response to a metered endpoint carries the current cap state.

| Header | Meaning |
|--------|---------|
| `X-RateLimit-Limit` | Monthly call cap for that service |
| `X-RateLimit-Remaining` | Calls left this month |
| `X-RateLimit-Reset` | ISO-8601 timestamp when the cap resets (the 1st) |
| `X-Usage-Warning` | Present at 80% (`warning:`) and 95% (`critical:`) of the cap |
| `X-Credits-Balance` | Credits remaining (sent on 402 responses and on search) |

## 402 — out of credits

```json
{
  "error": {
    "message": "Insufficient credits (balance: 0.0). Purchase more at https://console.callmissed.com/org/billing",
    "type": "insufficient_quota",
    "code": "insufficient_credits"
  }
}
```

Do not retry. Top up first.

## 429 — monthly cap reached

```json
{
  "error": {
    "message": "Plan limit exceeded: 100/100 llm calls this month. Upgrade your plan at console.callmissed.com/org/billing",
    "type": "insufficient_quota",
    "code": "quota_exceeded"
  }
}
```

This 429 carries `Retry-After` in seconds until the 1st of next month. Do not
retry inside that window — upgrade the plan instead.

## 429 — too many concurrent requests

```json
{
  "error": {
    "message": "Too many concurrent requests for this API key. Retry shortly.",
    "type": "rate_limit_error",
    "code": "too_many_concurrent_requests"
  }
}
```

This one clears in seconds. Retry with jittered backoff.

```python
import time
from openai import OpenAI, RateLimitError

client = OpenAI(api_key="cm_your_key", base_url="https://api.callmissed.com/v1")

try:
    resp = client.chat.completions.create(
        model="kimi-k2.5",
        messages=[{"role": "user", "content": "Hello"}],
    )
except RateLimitError as e:
    retry_after = int(e.response.headers.get("Retry-After", 0))
    if 0 < retry_after < 120:
        time.sleep(retry_after)
        resp = client.chat.completions.create(
            model="kimi-k2.5",
            messages=[{"role": "user", "content": "Hello"}],
        )
    else:
        raise  # monthly cap — upgrade rather than wait
```

See [Errors](/docs/errors) for the full status and code tables.
