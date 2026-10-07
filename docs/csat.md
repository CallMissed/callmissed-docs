---
title: "CSAT & NPS Surveys"
description: "Mint a survey link after a conversation, send the hosted response page to your customer, and read CSAT and NPS statistics."
slug: "csat"
breadcrumb: "API Reference"
---

# CSAT & NPS Surveys

Mint a survey link after a conversation, send the hosted response page to your customer, and read CSAT and NPS statistics.

## Overview

Create a survey, send its link to the customer, and read the score back.

Every endpoint on this page is called with your `cm_` key. The customer never sees the API: they answer on a hosted response page addressed by the survey's `token`, so no key ever reaches their device.

Two scales are supported: `csat_5` (satisfaction, 1–5) and `nps_10` (recommendation, 0–10).

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, get, stats | `csat:read` |
| Create, delete, mark sent | `csat:write` |

## The survey object

```json
{
  "id": "aa11…",
  "tenant_id": "a0b1…",
  "conversation_id": "c0ff…",
  "ticket_id": "e5d4…",
  "contact_id": "4411…",
  "channel": "whatsapp",
  "token": "0Yb3k…",
  "question": "How satisfied were you with this conversation?",
  "scale": "csat_5",
  "sent_at": null,
  "expires_at": "2026-08-24T00:00:00Z",
  "created_at": "2026-08-17T07:00:00Z",
  "updated_at": "2026-08-17T07:00:00Z"
}
```

`token` is minted server-side — you cannot supply or choose it. Treat it as a secret: anyone holding it can answer that survey once.

## POST `/api/v1/csat/surveys`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `channel` | `string` | Yes | `whatsapp`, `email`, `web` or `voice` |
| `scale` | `string` | No | `csat_5` (default) or `nps_10` |
| `conversation_id` | `UUID` | No | Must exist in your tenant |
| `contact_id` | `UUID` | No | Must exist in your tenant |
| `ticket_id` | `UUID` | No | Stored as an opaque reference, not validated |
| `question` | `string` | No | At most 255 characters. Defaults to the scale's standard wording |
| `expires_at` | `datetime` | No | Omit for a link that never expires |

```bash
curl -X POST https://api.callmissed.com/api/v1/csat/surveys \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "channel": "whatsapp",
    "scale": "nps_10",
    "conversation_id": "c0ffee00-1111-2222-3333-444455556666",
    "expires_at": "2026-08-24T00:00:00Z"
  }'
```

Returns `201` with the survey object. `404 Conversation not found` / `Contact not found` when a linked id is not in your tenant.

## Sending the survey

Creating a survey does not deliver it. Build the customer's link from the returned `token`:

```
https://console.callmissed.com/csat/{token}
```

Send that link on your own channel (WhatsApp, email, SMS, your app), then call `POST /api/v1/csat/surveys/{survey_id}/sent` (below) to record when it went out.

The hosted page shows the survey's `question`, renders the rating range for its `scale` (`1`–`5` for `csat_5`, `0`–`10` for `nps_10`) and takes an optional comment of up to 2,000 characters. It accepts **one answer per survey**; a second attempt is refused. Once `expires_at` has passed it shows the survey as closed instead of the form. No tenant, contact, conversation or ticket id is shown to the customer.

The survey object does not carry the customer's rating or comment. Use the `answered` filter below to see which surveys have a response, and `GET /api/v1/csat/stats` for the aggregates.

## GET `/api/v1/csat/surveys`

Returns an array of survey objects, newest first.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `channel` | `string` | One of the four channels |
| `answered` | `boolean` | `true` = has a response, `false` = still open |
| `created_after` | `datetime` | Inclusive |
| `created_before` | `datetime` | Exclusive |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/csat/surveys?channel=whatsapp&answered=false" \
  -H "Authorization: Bearer cm_your_api_key"
```

An unknown `channel` returns `422 channel must be one of: whatsapp, email, web, voice`.

## GET `/api/v1/csat/stats`

| Parameter | Type | Constraints |
| --- | --- | --- |
| `days` | `integer` | `1 <= days <= 365`, default `30` |

The window covers surveys **created** in the last `days` days.

```bash
curl "https://api.callmissed.com/api/v1/csat/stats?days=30" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "days": 30,
  "surveys_sent": 1840,
  "responses": 611,
  "response_rate": 0.3321,
  "average_rating": 4.31,
  "promoters": 302,
  "passives": 118,
  "detractors": 61,
  "nps": 49.75
}
```

| Field | Notes |
| --- | --- |
| `surveys_sent` | Surveys created in the window, whether or not you called `/sent` for them |
| `response_rate` | `responses / surveys_sent` as a fraction in `0..1`, not a percentage. `0.0` when no survey was created |
| `average_rating` | Across all scales. `null` when there are no responses |
| `promoters` / `passives` / `detractors` | `nps_10` only. Promoter is 9–10, passive 7–8, detractor 0–6 |
| `nps` | `(promoters − detractors) / nps_responses × 100`. `null` when there are no `nps_10` answers in the window |

## GET / DELETE `/api/v1/csat/surveys/{survey_id}`

`GET` returns one survey. `DELETE` returns `204` and removes its response along with it. `404 Survey not found`.

```bash
curl https://api.callmissed.com/api/v1/csat/surveys/{survey_id} \
  -H "Authorization: Bearer cm_your_api_key"

curl -X DELETE https://api.callmissed.com/api/v1/csat/surveys/{survey_id} \
  -H "Authorization: Bearer cm_your_api_key"
```

## POST `/api/v1/csat/surveys/{survey_id}/sent`

Creating a survey does not deliver it: you send the link on your own channel, then call this to record when it went out. Scope: `csat:write`. No body. Returns the survey with `sent_at` set. Idempotent — calling it again keeps the first `sent_at`. `404 Survey not found`.

```bash
curl -X POST https://api.callmissed.com/api/v1/csat/surveys/{survey_id}/sent \
  -H "Authorization: Bearer cm_your_api_key"
```

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `csat:read` / `csat:write` |
| `404` | Survey, conversation or contact not in your tenant |
| `422` | Unknown channel, blank question, or `days` outside `1..365` |

Nothing on this page consumes credits.
