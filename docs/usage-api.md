---
title: "Usage API"
description: "Read your own metering data — spend summaries, per-request logs, and a CSV export you can drop into a warehouse."
slug: "usage-api"
breadcrumb: "API Reference"
---

# Usage API

Read your own metering data — spend summaries, per-request logs, and a CSV export you can drop into a warehouse.

## Overview

The Usage API returns the same metering rows that back the dashboard's usage charts: one record per billable API call, with the service, model, token counts, latency and the credits it cost you.

Use it to build an internal cost dashboard, attribute spend to a customer via `session_id`, or reconcile an invoice.

| Endpoint | Returns |
| --- | --- |
| `GET /v1/usage/summary` | Rolled-up totals, per-service and per-model breakdowns, a daily series |
| `GET /v1/usage/logs` | Individual request records, newest first |
| `GET /v1/usage/logs.csv` | The same records as a CSV download |
| `GET /v1/credits/balance` | Your current credit balance |

## Authentication

```
Authorization: Bearer cm_your_api_key
```

All three endpoints require the `usage:read` scope. A dashboard session also works.

```json
{ "detail": "API key missing required scope: usage:read. Add it under the key's 'Permissions' section in your dashboard." }
```

## Retention window

Usage history is queryable for the **last 90 days**. Any request for a wider or older window returns `422` — export to CSV on a schedule if you need to keep more.

## GET `/v1/usage/summary`

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `days` | `integer` | No | `1 <= days <= 90`, default `30` |

```bash
curl "https://api.callmissed.com/v1/usage/summary?days=7" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "period_days": 7,
  "totals": {
    "total_requests": 18422,
    "success": 18310,
    "errors": 112,
    "success_rate": 0.9939,
    "total_cost_usd": 41.87,
    "total_input_tokens": 9120334,
    "total_output_tokens": 1844920,
    "total_cache_read_tokens": 4210000,
    "total_cache_creation_tokens": 120500,
    "total_audio_seconds": 6120.5,
    "avg_latency_ms": 812.4
  },
  "by_service": [
    { "service": "llm", "requests": 15980, "cost_usd": 38.11, "input_tokens": 9120334, "output_tokens": 1844920, "cache_read_tokens": 4210000, "cache_creation_tokens": 120500 },
    { "service": "tts", "requests": 1422, "cost_usd": 2.44, "input_tokens": 0, "output_tokens": 0, "cache_read_tokens": 0, "cache_creation_tokens": 0 }
  ],
  "by_model": [
    { "model": "kimi-k2.6", "requests": 9120, "cost_usd": 12.30 }
  ],
  "series": [
    { "date": "2026-08-11", "requests": 2610, "cost_usd": 5.98 }
  ]
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `totals.success_rate` | `number` | Fraction in `0..1`, not a percentage |
| `totals.total_cost_usd` | `number` | What **you** were charged, in US$ at the published rate (1 credit = ₹1, US$1 = ₹96, so credits = US$ × 96). Every `cost_usd` in this response uses the same unit |
| `totals.total_input_tokens` | `integer` | LLM prompt tokens that were **not** served from the prompt cache |
| `totals.total_cache_read_tokens` | `integer` | Prompt tokens served from the prompt cache, billed at the cache-read rate |
| `totals.total_cache_creation_tokens` | `integer` | Prompt tokens written to the prompt cache, billed at the cache-write rate |
| `by_service[].cache_read_tokens`, `by_service[].cache_creation_tokens` | `integer` | The same cache split for one service |
| `by_model` | `array` | Top 10 models by request volume |
| `series[].date` | `string` | `YYYY-MM-DD`, one row per day in the window |

A tenant with no traffic gets zeroed totals and empty arrays — never an error.

## GET `/v1/usage/logs`

Individual request records, newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `days` | `integer` | No | `1 <= days <= 90`, default `7`. Ignored when `start` or `end` is present |
| `start` | `datetime` | No | ISO-8601 UTC |
| `end` | `datetime` | No | ISO-8601 UTC |
| `service` | `string` | No | One of `llm`, `stt`, `tts`, `image`, `search`, `embedding`, `bot`, `whatsapp_message`, `whatsapp_call`, `telephony_call` |
| `model` | `string` | No | At most 255 characters. Exact model id |
| `api_key_id` | `UUID` | No | Restrict to one key |
| `status` | `string` | No | `ok` or `error` (`error` means a status code of 400 or above) |
| `session_id` | `string` | No | At most 64 characters |
| `trace_id` | `string` | No | At most 64 characters |
| `limit` | `integer` | No | `1 <= limit <= 500`, default `100` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/v1/usage/logs?days=1&service=llm&status=error&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "period": { "start": "2026-08-16T00:00:00Z", "end": "2026-08-17T00:00:00Z" },
  "limit": 50,
  "offset": 0,
  "count": 2,
  "logs": [
    {
      "id": "9a1f…",
      "created_at": "2026-08-16T18:04:11Z",
      "service": "llm",
      "endpoint": "/v1/chat/completions",
      "method": "POST",
      "model": "kimi-k2.6",
      "status_code": 429,
      "latency_ms": 41,
      "input_tokens": 0,
      "output_tokens": 0,
      "cache_read_tokens": 0,
      "cache_creation_tokens": 0,
      "audio_seconds": 0.0,
      "cost_usd": 0.0,
      "request_id": "req_01J…",
      "api_key_id": "b1f2…",
      "error_message": "rate limit exceeded",
      "trace_id": null,
      "session_id": "checkout-flow"
    }
  ]
}
```

For LLM rows, `input_tokens` is the part of the prompt that was not cached;
`cache_read_tokens` (served from the prompt cache) and `cache_creation_tokens`
(written to it) are counted separately, so the whole prompt is the sum of the
three. Rows recorded before cache tracking was added show `0` for both cache fields.

`cost_usd` is your price for the call, in US$ at the published rate (1 credit = ₹1, US$1 = ₹96), the same unit as `/v1/usage/summary`. Failed requests are recorded with `cost_usd: 0.0` — an error is never billed.

### Attributing spend

On `POST /v1/messages`, put `trace_id` and/or `session_id` (each at most 64 characters) inside the `metadata` object, then filter here by `session_id` or `trace_id` to attribute spend to one of your own customers, tenants or workflows. Other `metadata` keys (up to 16, string/number/boolean values) are stored with the row and exported in the CSV's `metadata_json` column.

**This needs request logging on the key that makes the call.** `trace_id`, `session_id` and the rest of `metadata` are stored only when **Request logging** is turned on for that API key, and logging is off by default on a new key. On a key with logging off the request is still metered and billed, but the row keeps `trace_id`, `session_id` and `metadata_json` empty (as it does `model`, `latency_ms`, `request_id` and `error_message`), so filtering by `session_id` or `trace_id` will not find it. Request logging is a console setting: turn it on when you create the key, or later from the key's edit dialog under **Developer → API keys** in the [console](https://console.callmissed.com/developer/keys). It cannot be changed with an API key, and it applies only to requests made after you turn it on. See [Logging](/docs/keys#logging).

```json
{
  "model": "kimi-k2.6",
  "max_tokens": 512,
  "metadata": { "session_id": "checkout-flow", "trace_id": "tr_8812", "feature": "summary" },
  "messages": [{ "role": "user", "content": "Summarise this order history." }]
}
```

On `/v1/chat/completions` and `/v1/responses` the `metadata`, `trace_id` and `session_id` fields are accepted and validated but are **not yet recorded** on the usage row, so those rows show `null` for both ids. Attribute that traffic with a separate API key per customer and filter by `api_key_id`.

## GET `/v1/usage/logs.csv`

Same filters as `/logs` **minus `limit` and `offset`**, capped at **5,000 rows** per download. Narrow the window or add filters if you need more.

```bash
curl "https://api.callmissed.com/v1/usage/logs.csv?days=30&service=llm" \
  -H "Authorization: Bearer cm_your_api_key" \
  -o usage.csv
```

Returns `text/csv` with `Content-Disposition: attachment; filename="usage-YYYYMMDD.csv"`. Header row:

```
id,created_at,service,endpoint,method,model,status_code,latency_ms,input_tokens,output_tokens,cache_read_tokens,cache_creation_tokens,audio_seconds,cost_usd,request_id,api_key_id,error_message[,trace_id][,session_id][,metadata_json]
```

`cost_usd` is in US$, as on `/logs`. `error_message` is truncated to 500 characters. Text cells are escaped so a spreadsheet cannot interpret a value as a formula.

## GET `/v1/credits/balance`

Your account's current credit balance. Any valid `cm_` key can call it — no scope or service permission is needed.

```bash
curl https://api.callmissed.com/v1/credits/balance \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "balance": 1843.25 }
```

`balance` is in credits (1 credit = ₹1). Errors use the OpenAI envelope, e.g. `401 invalid_api_key`.

## Errors

| Status | Detail | Cause |
| --- | --- | --- |
| `403` | `API key missing required scope: usage:read…` | Key lacks the scope |
| `422` | `` `start` must be earlier than `end`. `` | Inverted range |
| `422` | `Requested range exceeds the 90-day maximum.` | `start`/`end` span too wide |
| `422` | `Usage history is available for the last 90 days only.` | `start` older than the window |
| `422` | `Unknown service '…'. Valid: …` | Bad `service` value |

Reading usage never consumes credits.
