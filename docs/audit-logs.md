---
title: "Audit & Usage Logs"
description: "Forensic visibility — the audit log of sensitive actions and the per-request API usage log with cost-by-day rollups and CSV export."
slug: "audit-logs"
breadcrumb: "API Reference"
---

# Audit & Usage Logs

Forensic visibility — the audit log of sensitive actions and the per-request API usage log with cost-by-day rollups and CSV export.

> Two complementary logs: the **audit log** records sensitive actions (key creation, role changes, logins, deactivation) for compliance; the **API usage log** records every billed request (model, latency, tokens, cost, status). Prompt/completion text is only retrievable when prompt logging is enabled on the key. Both export to CSV.

## Credential class: dashboard JWT, owner or admin

Every endpoint here resolves a **dashboard access token** and then checks the caller's role. A `cm_` API key is **rejected with `401`** regardless of its scopes, and an `agent`-role user is rejected with `403`.

```
Authorization: Bearer <jwt_access_token>
```

| Check | Failure |
| --- | --- |
| Valid dashboard access token | `401` |
| Role is `owner` or `admin` | `403 Requires role: admin, owner` |

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`; parameter-bound violations are `422`.

Every query is scoped to your tenant and bounded to **90 days** so a wide window cannot trigger an expensive scan.

---

# Audit events

Sensitive tenant actions. Actions currently written include `budget.update`, `profile.update`, `user.invite`, `user.delete`, `role.change`, `webhook.delete`, and `webhook.delivery.replay`. Treat the action list as open: filter on the values you see rather than hardcoding an enum.

## GET /api/v1/audit/events

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `days` | `integer` | No | `1 <= days <= 90`, default `30` |
| `limit` | `integer` | No | `1 <= limit <= 500`, default `100` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |
| `action` | `string` | No | Max 100 chars. Exact match |
| `resource_type` | `string` | No | Max 50 chars. Exact match, for example `user`, `webhook`, `tenant` |

```bash
curl "https://api.callmissed.com/api/v1/audit/events?days=30&limit=100&action=role.change" \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
[
  {
    "id": "e91a2b3c-4d5e-6f70-8192-a3b4c5d6e7f8",
    "user_id": "u1234567-89ab-cdef-0123-456789abcdef",
    "action": "role.change",
    "resource_type": "user",
    "resource_id": "u7654321-ba98-fedc-3210-fedcba987654",
    "ip_address": "203.0.113.42",
    "user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)",
    "metadata": { "from": "agent", "to": "admin", "ownership_transfer": false },
    "created_at": "2026-08-02T11:20:44+00:00"
  }
]
```

Sorted newest first. `user_id` is `null` for actions taken by a non-user actor.

## GET `/api/v1/audit/events/{event_id}`

`event_id` is a UUID. Returns the same object shape as one row of the list.

```bash
curl https://api.callmissed.com/api/v1/audit/events/e91a2b3c-4d5e-6f70-8192-a3b4c5d6e7f8 \
  -H "Authorization: Bearer <jwt_access_token>"
```

| Status | Cause |
| --- | --- |
| `404` | `Audit event not found` in your tenant |
| `422` | `event_id` is not a valid UUID |

## GET /api/v1/audit/events.csv

Same query as `/events`, streamed as CSV. Capped at **5000 rows**; there is no `limit` or `offset` here.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `days` | `integer` | No | `1 <= days <= 90`, default `30` |
| `action` | `string` | No | Max 100 chars |
| `resource_type` | `string` | No | Max 50 chars |

```bash
curl "https://api.callmissed.com/api/v1/audit/events.csv?days=90" \
  -H "Authorization: Bearer <jwt_access_token>" \
  -o audit.csv
```

Response is `Content-Type: text/csv` with `Content-Disposition: attachment; filename="audit-YYYYMMDD.csv"`. Header row:

```
id,created_at,action,resource_type,resource_id,user_id,ip_address,user_agent,metadata_json
```

`user_agent` is truncated to 200 chars; `metadata_json` is the compact JSON encoding of the event metadata.

---

# API usage logs

One row per authenticated inference request: which key, which model, status, latency, tokens, and cost.

## GET /api/v1/audit/api-logs

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `days` | `integer` | No | `1 <= days <= 90`, default `7` |
| `limit` | `integer` | No | `1 <= limit <= 500`, default `100` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |
| `service` | `string` | No | One of `llm`, `stt`, `tts`, `bot`, `search`, `image`, `embedding`, `whatsapp_message`. An unknown value returns `400` |
| `status` | `string` | No | `ok` (status code &lt; 400) or `error` (status code &gt;= 400). Anything else is `422` |
| `key_id` | `UUID` | No | Narrow to one API key |
| `min_cost` | `number` | No | `0 <= min_cost <= 1000`, in **USD**. Filters rows only; the summary block still covers the whole window |

```bash
curl "https://api.callmissed.com/api/v1/audit/api-logs?days=7&status=error&service=llm&limit=100" \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "data": [
    {
      "id": "f0a1b2c3-d4e5-6789-0abc-def123456789",
      "api_key_id": "k1234567-89ab-cdef-0123-456789abcdef",
      "key_name": "Production LLM key",
      "service": "llm",
      "model": "glm-5.2",
      "endpoint": "/v1/chat/completions",
      "method": "POST",
      "status_code": 429,
      "latency_ms": 118,
      "input_tokens": 0,
      "output_tokens": 0,
      "audio_seconds": 0.0,
      "cost_usd": 0.0,
      "request_id": "req_9f2a1c4e6b8d0a13",
      "ip_address": "203.0.113.42",
      "error_message": "rate_limit_exceeded",
      "has_prompt_log": false,
      "created_at": "2026-08-04T09:58:31.482119+00:00"
    }
  ],
  "summary": {
    "total_calls": 1532,
    "total_cost_usd": 7.3812,
    "error_count": 12,
    "period_days": 7
  }
}
```

`key_name` is `null` when the key has since been deleted; the historical rows are kept. `has_prompt_log` tells you whether the content endpoint below will return anything. The `summary` block applies the `service` and `key_id` filters but ignores `status` and `min_cost`, so "12 of 1532 calls failed" stays meaningful.

| Status | Cause |
| --- | --- |
| `400` | `Invalid service filter. Allowed: [...]` |
| `403` | Caller is not owner or admin |
| `422` | A numeric bound was exceeded, or `status` is not `ok`/`error` |

## GET /api/v1/audit/api-logs/cost-by-day

Calls, cost, and errors grouped by UTC calendar day. Empty days are backfilled with zeros so a chart axis stays continuous.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `days` | `integer` | No | `1 <= days <= 90`, default `7` |
| `service` | `string` | No | Same allowlist as `/api-logs` |
| `key_id` | `UUID` | No | Narrow to one API key |

```bash
curl "https://api.callmissed.com/api/v1/audit/api-logs/cost-by-day?days=7" \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "days": [
    { "day": "2026-07-28", "calls": 210, "cost_usd": 0.9814, "errors": 0 },
    { "day": "2026-07-29", "calls": 0,   "cost_usd": 0.0,    "errors": 0 },
    { "day": "2026-07-30", "calls": 388, "cost_usd": 1.7702, "errors": 4 }
  ],
  "period_days": 7
}
```

Rows run oldest to newest. `400` on an invalid `service`.

## GET `/api/v1/audit/api-logs/{usage_id}/content`

Returns the stored prompt and completion text for one usage row. Content exists **only** when the calling key had both `logs_enabled` and `prompt_logging_enabled` at the time of the request (see [API Keys](/docs/keys)). Otherwise nothing was written and this returns `404`.

```bash
curl https://api.callmissed.com/api/v1/audit/api-logs/f0a1b2c3-d4e5-6789-0abc-def123456789/content \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "usage_id": "f0a1b2c3-d4e5-6789-0abc-def123456789",
  "prompt_text": "Summarise this support thread in two lines.",
  "completion_text": "The customer's order shipped on 2 Aug and is out for delivery.",
  "truncated": false,
  "created_at": "2026-08-04T09:58:31.482119+00:00"
}
```

`truncated` is `true` when the stored text was clipped.

| Status | Cause |
| --- | --- |
| `404` | `No prompt content captured for this request.` Also returned for a usage id belonging to another tenant |
| `422` | `usage_id` is not a valid UUID |

## GET /api/v1/audit/api-logs.csv

Same query as `/api-logs`, streamed as CSV. Capped at **5000 rows**. `min_cost`, `limit`, and `offset` are not accepted here.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `days` | `integer` | No | `1 <= days <= 90`, default `7` |
| `service` | `string` | No | Same allowlist as `/api-logs` |
| `status` | `string` | No | `ok` or `error` |
| `key_id` | `UUID` | No | Narrow to one API key |

```bash
curl "https://api.callmissed.com/api/v1/audit/api-logs.csv?days=30&status=error" \
  -H "Authorization: Bearer <jwt_access_token>" \
  -o api-logs.csv
```

Response is `Content-Type: text/csv` with `Content-Disposition: attachment; filename="api-logs-YYYYMMDD.csv"`. Header row:

```
id,created_at,key_name,service,model,endpoint,method,status_code,latency_ms,input_tokens,output_tokens,audio_seconds,cost_usd,request_id,ip_address,error_message
```

`audio_seconds` is formatted to 2 decimals, `cost_usd` to 6, and `error_message` is truncated to 200 chars.
