---
title: "Voice Alerts"
description: "Get an email or a signed webhook when a call metric — connect rate, failure rate, hang-ups, volume or p95 latency — crosses a threshold over a rolling window."
slug: "voice-alerts"
breadcrumb: "Voice Agents"
---

# Voice Alerts

Get an email or a signed webhook when a call metric — connect rate, failure rate, hang-ups, volume or p95 latency — crosses a threshold over a rolling window.

## Overview

A **voice alert** watches one call metric over a rolling window and notifies you when it crosses a threshold: "tell me when the connect rate drops **below** 40% over the last hour", or "when p95 turn latency goes **above** 2,500 ms over 15 minutes". Scope an alert to one agent with `bot_id`, or leave it workspace-wide.

Alerts are evaluated about **once a minute**. Firing is edge-triggered: a metric that stays bad notifies **once**, not once per evaluation. The alert re-arms only after the metric recovers, and `cooldown_minutes` stops a flapping metric from notifying again too soon.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Caller | Requirement |
| --- | --- |
| `cm_` API key | Must carry the `webhooks:write` scope — **on every endpoint, including the list**. An alert is a notification subscription, so it shares the webhook scope |
| Dashboard user | Any member can list. Create, update and delete require **owner or admin** (`403 Only owners/admins can manage alerts`) |

Base URL: `https://api.callmissed.com`. Errors are `{"detail": "..."}`.

## Metrics

| `metric` | Unit | Measures |
| --- | --- | --- |
| `connect_rate` | fraction `0..1` | Phone calls that connected ÷ calls attempted |
| `failure_rate` | fraction `0..1` | Phone calls that failed, went unanswered or were busy ÷ calls attempted |
| `call_volume` | count | Phone calls attempted in the window |
| `session_failure_rate` | fraction `0..1` | Voice sessions that failed or timed out ÷ all voice sessions |
| `hangup_rate` | fraction `0..1` | Voice sessions the caller hung up (rather than the agent ending the call) ÷ all voice sessions |
| `latency_p95` | milliseconds | 95th-percentile end-to-end turn latency across voice-session turns |

- Every `*_rate` metric is a **fraction**: `0.25` means 25%. A rate threshold outside `0..1` is rejected with `422` (`"<metric> is a fraction — threshold must be between 0 and 1 (0.25 means 25%)"`). Non-rate thresholds must be `>= 0`.
- Rate metrics need **at least 5 calls or sessions** in the window before they produce a reading. Below that there is no reading: the alert is not evaluated, does not fire, and does not clear.

## The alert object

```json
{
  "id": "a1e2c3d4-5f60-4718-8293-a4b5c6d7e8f9",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "bot_id": null,
  "name": "Connect rate dropped",
  "metric": "connect_rate",
  "comparison": "below",
  "threshold": 0.4,
  "window_minutes": 60,
  "channels": ["email", "webhook"],
  "enabled": true,
  "cooldown_minutes": 60,
  "is_firing": false,
  "last_value": 0.62,
  "last_fired_at": null,
  "last_evaluated_at": "2026-09-30T10:14:00+00:00"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `UUID` | |
| `tenant_id` | `UUID` | Your workspace |
| `bot_id` | `UUID \| null` | `null` = workspace-wide |
| `name` | `string` | |
| `metric` | `string` | One of the metrics above. Cannot be changed after creation |
| `comparison` | `"above" \| "below"` | |
| `threshold` | `number` | |
| `window_minutes` | `integer` | Rolling window the metric is computed over |
| `channels` | `string[]` | Subset of `email`, `webhook` |
| `enabled` | `boolean` | |
| `cooldown_minutes` | `integer` | Minimum gap between two notifications |
| `is_firing` | `boolean` | Read-only. `true` while the condition holds |
| `last_value` | `number \| null` | Read-only. The most recent reading |
| `last_fired_at` | `string \| null` | Read-only. ISO 8601 time of the last notification |
| `last_evaluated_at` | `string \| null` | Read-only. ISO 8601 time of the last evaluation |

---

## GET `/api/v1/voice-alerts`

Every alert in your workspace, newest first. No parameters.

```bash
curl https://api.callmissed.com/api/v1/voice-alerts \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns an array of alert objects.

## POST `/api/v1/voice-alerts`

Creates an alert. Returns `201` with the alert object.

| Field | Type | Required | Default | Constraints |
| --- | --- | --- | --- | --- |
| `name` | `string` | Yes | — | 1–120 characters |
| `metric` | `string` | Yes | — | One of the six metrics above |
| `comparison` | `string` | No | `"above"` | `above` or `below` |
| `threshold` | `number` | Yes | — | `0..1` for rate metrics, `>= 0` otherwise |
| `window_minutes` | `integer` | No | `60` | `5..1440` |
| `bot_id` | `UUID` | No | `null` | Must be one of your agents (`404 Agent not found` otherwise) |
| `channels` | `string[]` | No | `["email"]` | At least one of `email`, `webhook` |
| `enabled` | `boolean` | No | `true` | |
| `cooldown_minutes` | `integer` | No | `60` | `5..10080` (7 days) |

```bash
curl -X POST https://api.callmissed.com/api/v1/voice-alerts \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Connect rate dropped",
    "metric": "connect_rate",
    "comparison": "below",
    "threshold": 0.4,
    "window_minutes": 60,
    "channels": ["email", "webhook"]
  }'
```

A workspace can hold at most **50** alerts; the 51st returns `409 Alert limit reached (50)`.

## PUT `/api/v1/voice-alerts/{alert_id}`

Partial update — send only the fields you want to change. Accepts `name`, `comparison`, `threshold`, `window_minutes`, `channels`, `enabled` and `cooldown_minutes`, with the same constraints as create. `metric` and `bot_id` are fixed; create a new alert to change them.

Changing `threshold`, `comparison` or `window_minutes` changes what the alert means, so it resets `is_firing` to `false`.

```bash
curl -X PUT https://api.callmissed.com/api/v1/voice-alerts/a1e2c3d4-5f60-4718-8293-a4b5c6d7e8f9 \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"threshold": 0.35, "cooldown_minutes": 120}'
```

Returns the updated alert object.

## DELETE `/api/v1/voice-alerts/{alert_id}`

Deletes the alert. Returns `204` with no body.

---

## Notifications

| Channel | Delivery |
| --- | --- |
| `email` | Sent to the workspace **owner's** registered email address |
| `webhook` | A `voice_alert.triggered` event to every active [webhook](/docs/webhooks) subscribed to it. An alert with a `bot_id` reaches workspace-wide subscriptions and subscriptions scoped to that agent |

Webhook deliveries are signed and retried exactly like every other event. The body:

```json
{
  "event": "voice_alert.triggered",
  "data": {
    "alert_id": "a1e2c3d4-5f60-4718-8293-a4b5c6d7e8f9",
    "name": "Connect rate dropped",
    "metric": "connect_rate",
    "comparison": "below",
    "threshold": 0.4,
    "value": 0.31,
    "window_minutes": 60,
    "bot_id": null,
    "scope": "workspace"
  },
  "timestamp": "2026-09-30T10:15:00+00:00"
}
```

`scope` is `"bot"` when the alert has a `bot_id`, otherwise `"workspace"`. A failure on one channel does not block the other.

## Errors

| Status | Cause |
| --- | --- |
| `403` | Key missing `webhooks:write`, or a dashboard user who is not owner/admin writing an alert |
| `404` | `Alert not found`, or `Agent not found` for a `bot_id` outside your workspace |
| `409` | `Alert limit reached (50)` |
| `422` | Unknown `metric`, `comparison` or channel; a threshold out of range; a window or cooldown out of bounds |
