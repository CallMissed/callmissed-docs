---
title: "CSAT & NPS Surveys"
description: "Mint a survey link after a conversation, host the response page yourself against the public token endpoints, and read CSAT and NPS statistics."
slug: "csat"
breadcrumb: "API Reference"
---

# CSAT & NPS Surveys

Mint a survey link after a conversation, host the response page yourself against the public token endpoints, and read CSAT and NPS statistics.

## Overview

Create a survey, send its link to the customer, and read the score back.

Two halves, and they authenticate differently:

| Half | Endpoints | Credential |
| --- | --- | --- |
| **Management** — mint surveys, read results | `/api/v1/csat/surveys`, `/api/v1/csat/stats` | Your `cm_` key |
| **Response** — what the customer's browser hits | `/api/v1/csat/r/{token}` | The token in the URL. No API key |

The response endpoints are public on purpose: they are what your survey page calls from the customer's device, where an API key must never be present. The token *is* the credential, and it authorises exactly one survey.

Two scales are supported: `csat_5` (satisfaction, 1–5) and `nps_10` (recommendation, 0–10).

---

# Management endpoints

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, get, stats | `csat:read` |
| Create, delete | `csat:write` |

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

Returns `201`. Build the customer's link from the returned `token` — point it at your own survey page, which then calls the public endpoints below.

## GET `/api/v1/csat/surveys`

Newest first.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `channel` | `string` | One of the four channels |
| `answered` | `boolean` | `true` = has a response, `false` = still open |
| `created_after` | `datetime` | Inclusive |
| `created_before` | `datetime` | Exclusive |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

## GET `/api/v1/csat/stats`

| Parameter | Type | Constraints |
| --- | --- | --- |
| `days` | `integer` | `1 <= days <= 365`, default `30` |

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
| `response_rate` | Fraction in `0..1`, not a percentage. `0.0` when nothing was sent |
| `average_rating` | Across all scales. `null` when there are no responses |
| `promoters` / `passives` / `detractors` | `nps_10` only. Promoter is 9–10, passive 7–8, detractor 0–6 |
| `nps` | `(promoters − detractors) / nps_responses × 100`. `null` when there are no `nps_10` answers in the window |

## GET / DELETE `/api/v1/csat/surveys/{survey_id}`

`GET` returns one survey. `DELETE` returns `204` and removes its response along with it. `404 Survey not found`.

---

# Public response endpoints

**No `Authorization` header. No scope.** These are the two calls your survey page makes from the customer's browser or app.

The token is validated globally: an unknown, deleted or malformed token returns the same `404 Survey not found`, so the endpoints cannot be used to discover which tokens exist.

Both endpoints are rate limited per client. Handle `429` by asking the customer to retry shortly.

## GET `/api/v1/csat/r/{token}`

Fetch what to render.

```bash
curl https://api.callmissed.com/api/v1/csat/r/0Yb3k…
```

```json
{
  "question": "How likely are you to recommend us to a friend or colleague?",
  "scale": "nps_10",
  "answered": false,
  "expired": false
}
```

The projection is deliberately minimal — no tenant, contact, conversation or ticket id is exposed to the customer's device.

| Field | Type | Notes |
| --- | --- | --- |
| `question` | `string` | Falls back to the scale's standard wording |
| `scale` | `string` | `csat_5` or `nps_10` — decides the rating range you render |
| `answered` | `boolean` | Show a thank-you instead of the form when `true` |
| `expired` | `boolean` | `true` still returns `200` here, so you can show a "this survey has closed" page rather than a hard error |

## POST `/api/v1/csat/r/{token}`

Submit the answer.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `rating` | `integer` | Yes | `1`–`5` for `csat_5`, `0`–`10` for `nps_10` |
| `comment` | `string` | No | At most 2,000 characters. An empty string is stored as no comment |

```bash
curl -X POST https://api.callmissed.com/api/v1/csat/r/0Yb3k… \
  -H "Content-Type: application/json" \
  -d '{ "rating": 9, "comment": "Fast and clear, thanks." }'
```

```json
{ "status": "recorded" }
```

The acknowledgement is contentless by design — a submitter learns nothing about your tenant, the survey or your score distribution.

**One answer per survey**, enforced by a uniqueness constraint rather than a convention. A second submit, including a double-tap race, returns `409` and never a `500`.

| Status | Detail | Meaning |
| --- | --- | --- |
| `404` | `Survey not found` | Unknown, deleted or malformed token |
| `410` | `This survey has closed.` | Past `expires_at` |
| `422` | `rating must be between {low} and {high}` | Rating outside the survey's scale |
| `409` | `This survey has already been answered.` | Already submitted |
| `429` | `Too many requests — try again shortly.` | Rate limited |

Checks run in that order, so an expired survey reports `410` rather than leaking whether it was already answered.

---

## Errors on the management half

| Status | When |
| --- | --- |
| `403` | Key is missing `csat:read` / `csat:write` |
| `404` | Survey, conversation or contact not in your tenant |
| `422` | Unknown channel, blank question, or `days` outside `1..365` |

Nothing on this page consumes credits.
