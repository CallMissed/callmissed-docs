---
title: "Lead Scoring & Timeline"
description: "Score contacts and companies from behavioural signals with your own rules, read the per-rule breakdown, and pull a unified activity timeline."
slug: "crm-lead-scores"
breadcrumb: "API Reference"
---

# Lead Scoring & Timeline

Score contacts and companies from behavioural signals with your own rules, read the per-rule breakdown, and pull a unified activity timeline.

## Overview

**Lead scoring** turns behaviour into a number. You define rules — each a signal, an operator, a value and a point award — and the API sums the ones that match a record, producing a score and an A–D grade with a per-rule breakdown so you can see exactly why.

**The timeline** is the other half of the same question: a single merged feed of everything that has happened to a contact or company.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| Read rules and scores | `crm_scores:read` |
| Create/update/delete rules, recompute | `crm_scores:write` |
| Read the timeline | `crm_timeline:read` |

---

# Lead scoring

## Signals

Signals are a **closed allowlist** per entity type — an unknown signal is `422`, never a silently-false rule.

### Contact

| Signal | Kind |
| --- | --- |
| `has_email`, `has_phone`, `whatsapp_opt_in`, `email_opt_in`, `sms_opt_in`, `has_company` | boolean |
| `conversation_count`, `deal_count`, `deal_value_total`, `days_since_last_conversation` | number |

### Company

| Signal | Kind |
| --- | --- |
| `has_domain`, `has_phone` | boolean |
| `industry`, `size` | string |
| `contact_count`, `deal_count`, `deal_value_total` | number |

## Operators

`eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `contains`, `is_set`, `is_not_set` — but **which ones are legal depends on the signal's kind**:

| Kind | Allowed operators |
| --- | --- |
| boolean | `eq`, `neq`, `is_set`, `is_not_set` |
| number | `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `is_set`, `is_not_set` |
| string | `eq`, `neq`, `contains`, `is_set`, `is_not_set` |

`is_set` and `is_not_set` take no `value` — one sent is dropped.

## Grades

| Grade | Score |
| --- | --- |
| A | 75 and above |
| B | 50–74 |
| C | 25–49 |
| D | below 25 |

## The rule object

```json
{
  "id": "77aa…",
  "tenant_id": "a0b1…",
  "entity_type": "contact",
  "name": "Opted in on WhatsApp",
  "signal": "whatsapp_opt_in",
  "operator": "eq",
  "value": true,
  "points": 20,
  "is_active": true,
  "created_at": "2026-08-06T09:00:00Z",
  "updated_at": "2026-08-06T09:00:00Z"
}
```

## GET `/api/v1/crm/lead-scores/rules`

Oldest first — the order you built them in.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `entity_type` | `string` | `contact` or `company` |
| `is_active` | `boolean` | |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

## POST `/api/v1/crm/lead-scores/rules`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact` or `company` |
| `name` | `string` | Yes | 1–255 characters, unique per entity type |
| `signal` | `string` | Yes | From the allowlist for that entity type |
| `operator` | `string` | Yes | Legal for the signal's kind |
| `value` | any | Conditional | Required unless the operator is `is_set` / `is_not_set`. String values at most 255 characters |
| `points` | `integer` | Yes | `-1000 <= points <= 1000`. Negative points subtract |
| `is_active` | `boolean` | No | Default `true` |

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/lead-scores/rules \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "entity_type": "contact",
    "name": "Stale — no contact in 30 days",
    "signal": "days_since_last_conversation",
    "operator": "gt",
    "value": 30,
    "points": -15
  }'
```

Errors name the exact problem, for example `operator 'contains' is not valid for the boolean signal 'has_email'. Allowed: eq, neq, is_set, is_not_set`.

## PATCH / DELETE `/api/v1/crm/lead-scores/rules/{rule_id}`

`PATCH` takes every field except `entity_type`, all optional. The signal / operator / value triple is re-validated against the **merged** result, so you cannot change the signal in one call and leave an illegal operator behind.

---

## Reading scores

### GET `/api/v1/crm/lead-scores`

Highest score first — the ranked list.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `entity_type` | `string` | `contact` or `company` |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

### GET `/api/v1/crm/lead-scores/{entity_type}/{entity_id}`

```json
{
  "id": "88bb…",
  "tenant_id": "a0b1…",
  "entity_type": "contact",
  "entity_id": "4411…",
  "score": 62,
  "grade": "B",
  "breakdown": [
    { "rule_id": "77aa…", "name": "Opted in on WhatsApp", "signal": "whatsapp_opt_in", "operator": "eq", "value": true, "points": 20 },
    { "rule_id": "99cc…", "name": "Has an open deal", "signal": "deal_count", "operator": "gte", "value": 1, "points": 42 }
  ],
  "computed_at": "2026-08-17T05:00:00Z"
}
```

`breakdown` lists only the rules that **matched**, which is what makes a score explainable to a salesperson.

`404 No score has been computed for this record` when the record has never been scored — call recompute first.

### POST `/api/v1/crm/lead-scores/recompute`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact` or `company` |
| `entity_ids` | `UUID[]` | Yes | 1–**200** entries |

```json
{ "entity_type": "contact", "requested": 3, "computed": 3 }
```

Idempotent, and ids are de-duplicated before the 200 cap is checked. If any id is not yours you get `404 2 of 50 contact ids were not found in this tenant; nothing was scored` — nothing is partially computed.

Scores are **not** recomputed automatically after you change a rule. Re-run the affected records yourself.

---

# Timeline

## GET `/api/v1/crm/timeline`

A merged, newest-first feed of everything attached to one record.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact` or `company`. **Deals are not supported here** |
| `entity_id` | `UUID` | Yes | |
| `types` | `string` | No | Comma-separated subset of `conversation,message,call,note,task,deal`. Omit for all |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/crm/timeline?entity_type=contact&entity_id=4411…&types=note,task" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "id": "note:aa22…",
    "type": "note",
    "occurred_at": "2026-08-16T12:00:00Z",
    "title": "Note added",
    "summary": "Renewal call went well — wants a Hindi voice agent.",
    "actor": "Ravi K",
    "ref_id": "aa22…",
    "meta": {}
  }
]
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `string` | Prefixed composite such as `note:<uuid>` — unique across types, good as a render key |
| `ref_id` | `UUID` | The underlying record's own id, for a follow-up fetch |
| `summary` | `string \| null` | Truncated to 280 characters |
| `actor` | `string \| null` | Who caused it, when known |
| `meta` | `object` | Type-specific extras. Defaults to `{}` |

> An unknown or another tenant's `entity_id` returns an **empty array, not a 404**. The timeline never confirms whether a record exists — do not use it as an existence check.

Read-only: there is no write scope and no way to post to a timeline. It is assembled from the underlying records.

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `crm_scores:*` / `crm_timeline:read` |
| `404` | Rule not found, no score computed yet, or an id outside your tenant during recompute |
| `409` | Duplicate rule name |
| `422` | Unknown signal, an operator illegal for that signal's kind, a value of the wrong type, `points` outside `-1000..1000`, over 200 recompute ids, or an unknown timeline type |

Nothing on this page consumes credits.
