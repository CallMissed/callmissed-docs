---
title: "Status API"
description: "Public service-health endpoints — current status, historical uptime, and the incident feed that power status.callmissed.com."
slug: "status-api"
breadcrumb: "API Reference"
---

# Status API

Public service-health endpoints — current status, historical uptime, and the incident feed that power status.callmissed.com.

> These endpoints are **public** (no auth) and back [status.callmissed.com](https://status.callmissed.com). Use them to embed live status in your own dashboards or to gate automated jobs on platform health.

## Credential class: none

Send no `Authorization` header. A dashboard JWT or a `cm_` API key is accepted but ignored: these routes have no auth dependency and return identical data either way. They are **not** tenant-scoped, so nothing here reveals your account.

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`; out-of-range query parameters are `422`.

All three endpoints share the same status vocabulary:

| `status` value | Meaning |
| --- | --- |
| `operational` | Healthy |
| `degraded` | Working with reduced quality |
| `down` | Not serving |
| `not_configured` | Not enabled on this deployment |
| `unknown` | No data recorded for that day (uptime only) |

---

## GET /api/v1/status

Live health snapshot plus the catalogue of route groups customers integrate with. No parameters.

```bash
curl https://api.callmissed.com/api/v1/status
```

```json
{
  "overall": "operational",
  "checked_at": "2026-08-04T10:31:07.512004+00:00",
  "services": [
    {
      "name": "CallMissed API",
      "group": "infrastructure",
      "status": "operational",
      "description": "REST API, dashboard, webhooks, and API keys",
      "latency_ms": 4
    },
    {
      "name": "AI, speech & language",
      "group": "ai",
      "status": "operational",
      "description": "Chat, speech-to-text, text-to-speech, and embeddings-compatible routes"
    },
    {
      "name": "Billing & checkout",
      "group": "payments",
      "status": "operational",
      "description": "Subscriptions and one-time payments"
    },
    {
      "name": "WhatsApp Business",
      "group": "channels",
      "status": "operational",
      "description": "Meta WhatsApp Cloud API"
    },
    {
      "name": "Voice & SMS",
      "group": "channels",
      "status": "operational",
      "description": "PSTN voice and SMS (Twilio)"
    },
    {
      "name": "Transactional email",
      "group": "notifications",
      "status": "operational",
      "description": "Account notifications and receipts"
    }
  ],
  "user_api_surface": [
    { "group": "openai_compat", "title": "OpenAI- & Anthropic-compatible APIs (Bearer API key)", "prefixes": ["/v1", "/anthropic/v1"] }
  ]
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `overall` | `string` | Derived from the `infrastructure` group only: `down` if any infra row is down, else `degraded` if any is degraded, else `operational` |
| `checked_at` | `string` | ISO-8601 UTC timestamp of this check |
| `services[].name` | `string` | Human label |
| `services[].group` | `string` | One of `infrastructure`, `ai`, `payments`, `channels`, `notifications` |
| `services[].status` | `string` | See the vocabulary above |
| `services[].description` | `string` | What the row covers |
| `services[].latency_ms` | `integer` | **Optional.** Present only where a live latency was measured |
| `user_api_surface[]` | `array` | Route-prefix catalogue, each entry with `group`, `title`, and `prefixes` |

To gate a job on platform health, poll this and require `overall == "operational"`.

## GET /api/v1/status/uptime

Per-service daily uptime for a rolling window ending today.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `days` | `integer` | No | `1 <= days <= 90`, default `90` |

```bash
curl "https://api.callmissed.com/api/v1/status/uptime?days=30"
```

```json
{
  "days": 30,
  "services": [
    {
      "service_name": "CallMissed API",
      "uptime_percent": 99.94,
      "daily": [
        {
          "date": "2026-07-06",
          "total_checks": 288,
          "operational_checks": 288,
          "degraded_checks": 0,
          "down_checks": 0,
          "worst_status": "operational"
        },
        {
          "date": "2026-07-07",
          "total_checks": 0,
          "operational_checks": 0,
          "degraded_checks": 0,
          "down_checks": 0,
          "worst_status": "unknown"
        }
      ]
    }
  ]
}
```

`daily` always contains exactly `days` entries, oldest first. Days with no recorded checks are returned as zero-filled placeholders with `worst_status: "unknown"` so a chart axis stays continuous.

`uptime_percent` counts a degraded check as half an operational one:

```
uptime_percent = round(((operational + 0.5 * degraded) / total) * 100, 2)
```

It is `0.0` when the window contains no checks at all. Services are sorted by name.

`422` when `days` is outside `1 .. 90`.

## GET /api/v1/status/incidents

Incidents overlapping the requested window, newest first. An incident is included when it started before now and either has not ended or ended inside the window.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `days` | `integer` | No | `1 <= days <= 90`, default `7` |

```bash
curl "https://api.callmissed.com/api/v1/status/incidents?days=7"
```

```json
{
  "days": 7,
  "incidents": [
    {
      "id": "b7c8d9e0-f1a2-4b3c-8d4e-5f6a7b8c9d0e",
      "service_name": "CallMissed API",
      "severity": "degraded",
      "title": "Elevated latency on the CallMissed API",
      "summary": "Requests were slower than normal for roughly 20 minutes.",
      "started_at": "2026-08-01T04:12:00+00:00",
      "ended_at": "2026-08-01T04:33:00+00:00",
      "resolved": true
    }
  ]
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `string` | UUID |
| `service_name` | `string` | Matches a `services[].name` from `/status` |
| `severity` | `string` | `degraded` or `down` |
| `title` | `string` | Generated from service and severity when no custom title was written |
| `summary` | `string \| null` | Optional detail |
| `started_at` | `string` | ISO-8601 UTC |
| `ended_at` | `string \| null` | `null` while the incident is ongoing |
| `resolved` | `boolean` | Whether the incident is closed |

`incidents` is an empty array when the window is clean. `422` when `days` is outside `1 .. 90`.
