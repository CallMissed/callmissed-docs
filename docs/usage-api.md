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
    "total_audio_seconds": 6120.5,
    "avg_latency_ms": 812.4
  },
  "by_service": [
    { "service": "llm", "requests": 15980, "cost_usd": 38.11, "input_tokens": 9120334, "output_tokens": 1844920 },
    { "service": "tts", "requests": 1422, "cost_usd": 2.44, "input_tokens": 0, "output_tokens": 0 }
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
| `totals.total_cost_usd` | `number` | What **you** were charged, in USD |
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

`cost_usd` is your price. Failed requests are recorded with `cost_usd: 0.0` — an error is never billed.

### Attributing spend

Send `X-Session-Id` and `X-Trace-Id` headers on your inference calls, then filter here by `session_id` or `trace_id` to attribute spend to one of your own customers, tenants or workflows.

## GET `/v1/usage/logs.csv`

Same filters as `/logs` **minus `limit` and `offset`**, capped at **5,000 rows** per download. Narrow the window or add filters if you need more.

```bash
curl "https://api.callmissed.com/v1/usage/logs.csv?days=30&service=llm" \
  -H "Authorization: Bearer cm_your_api_key" \
  -o usage.csv
```

Returns `text/csv` with `Content-Disposition: attachment; filename="usage-YYYYMMDD.csv"`. Header row:

```
id,created_at,service,endpoint,method,model,status_code,latency_ms,input_tokens,output_tokens,audio_seconds,cost_usd,request_id,api_key_id,error_message[,trace_id][,session_id][,metadata_json]
```

`error_message` is truncated to 500 characters. Text cells are escaped so a spreadsheet cannot interpret a value as a formula.

## Errors

| Status | Detail | Cause |
| --- | --- | --- |
| `403` | `API key missing required scope: usage:read…` | Key lacks the scope |
| `422` | `` `start` must be earlier than `end`. `` | Inverted range |
| `422` | `Requested range exceeds the 90-day maximum.` | `start`/`end` span too wide |
| `422` | `Usage history is available for the last 90 days only.` | `start` older than the window |
| `422` | `Unknown service '…'. Valid: …` | Bad `service` value |

Reading usage never consumes credits.
