---
title: "Deals & Pipelines"
description: "Model your sales process as pipelines and stages, then move deals through them — with automatic won/lost status and close stamping."
slug: "crm-deals"
breadcrumb: "API Reference"
---

# Deals & Pipelines

Model your sales process as pipelines and stages, then move deals through them — with automatic won/lost status and close stamping.

## Overview

A **pipeline** is one sales process. Its **stages** are the ordered steps, each with a `position`, an optional win probability, and flags marking it as the won or lost terminus. A **deal** sits in exactly one stage of one pipeline.

Moving a deal onto a stage flagged `is_won` or `is_lost` sets the deal's `status` and stamps `closed_at` for you — you do not manage those by hand.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

Pipelines and deals share one scope pair.

| Operation | Scope |
| --- | --- |
| Read pipelines, stages, deals | `crm_deals:read` |
| Create, update, move, delete | `crm_deals:write` |

---

# Pipelines

```json
{
  "id": "11aa…",
  "tenant_id": "a0b1…",
  "name": "Inbound",
  "is_default": true,
  "created_at": "2026-08-01T09:00:00Z",
  "updated_at": "2026-08-01T09:00:00Z"
}
```

Exactly one pipeline may be the default. Setting `is_default: true` clears the flag on the others in the same transaction.

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/api/v1/crm/pipelines` | Default first, then oldest first. `limit` `1..200` (default `50`), `offset` `0..100000` |
| `POST` | `/api/v1/crm/pipelines` | `name` 1–255 characters, unique per tenant; `is_default` default `false` |
| `GET` | `/api/v1/crm/pipelines/{pipeline_id}` | |
| `PATCH` | `/api/v1/crm/pipelines/{pipeline_id}` | `name`, `is_default` |
| `DELETE` | `/api/v1/crm/pipelines/{pipeline_id}` | `204`. `409 Move or delete the deals here first` if it still holds deals |

## Stages

```json
{
  "id": "22bb…",
  "tenant_id": "a0b1…",
  "pipeline_id": "11aa…",
  "name": "Negotiation",
  "position": 3,
  "probability": 60,
  "is_won": false,
  "is_lost": false,
  "created_at": "2026-08-01T09:01:00Z",
  "updated_at": "2026-08-01T09:01:00Z"
}
```

### GET `/api/v1/crm/pipelines/{pipeline_id}/stages`

Ordered by `position`. **No pagination** — a pipeline's stage list is always returned whole.

### POST `/api/v1/crm/pipelines/{pipeline_id}/stages`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, not blank |
| `position` | `integer` | No | `0 <= position <= 10000`. Omit to append after the current last stage |
| `probability` | `number` | No | `0 <= probability <= 100`, as a percentage |
| `is_won` | `boolean` | No | Default `false` |
| `is_lost` | `boolean` | No | Default `false` |

A stage cannot be both — `422 A stage cannot be both won and lost`.

### PATCH `/api/v1/crm/pipelines/stages/{stage_id}` · DELETE `/api/v1/crm/pipelines/stages/{stage_id}`

`DELETE` returns `204`, or `409 Move or delete the deals here first` when deals still sit on it.

### PATCH `/api/v1/crm/pipelines/{pipeline_id}/stages/reorder`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `stage_ids` | `UUID[]` | Yes | 1–100 entries. Must be **every** stage of this pipeline, exactly once |

New `position` is the array index. A partial or duplicated list returns `422` and changes nothing, so two concurrent reorders cannot interleave.

---

# Deals

## The deal object

```json
{
  "id": "33cc…",
  "tenant_id": "a0b1…",
  "title": "Acme — 3 voice agents",
  "value": 240000.0,
  "currency": "INR",
  "pipeline_id": "11aa…",
  "stage_id": "22bb…",
  "contact_id": "4411…",
  "company_id": "5c6d…",
  "owner_user_id": "b1f2…",
  "status": "open",
  "expected_close_date": "2026-09-15",
  "closed_at": null,
  "last_activity_at": "2026-08-16T12:05:00Z",
  "created_at": "2026-08-05T09:00:00Z",
  "updated_at": "2026-08-16T12:05:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `status` | `string` | `open`, `won` or `lost`. Driven by the stage — see below |
| `currency` | `string` | ISO 4217, three letters, upper-cased. Default `INR` |
| `closed_at` | `datetime \| null` | Stamped when the deal first lands on a won or lost stage, and **never re-stamped** |
| `last_activity_at` | `datetime \| null` | Bumped on every write to the deal |

## GET `/api/v1/crm/deals`

Newest first.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `pipeline_id` | `UUID` | |
| `stage_id` | `UUID` | |
| `status` | `string` | `open`, `won` or `lost` |
| `owner_user_id` | `UUID` | |
| `contact_id` | `UUID` | |
| `company_id` | `UUID` | |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

## POST `/api/v1/crm/deals`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `title` | `string` | Yes | 1–255 characters, not blank |
| `pipeline_id` | `UUID` | Yes | Must be your pipeline |
| `stage_id` | `UUID` | No | Must belong to that pipeline. Omit to start at the lowest-position stage |
| `value` | `number` | No | `0 <= value <= 1e12` |
| `currency` | `string` | No | Exactly 3 letters, default `INR` |
| `contact_id` | `UUID` | No | |
| `company_id` | `UUID` | No | |
| `owner_user_id` | `UUID` | No | Must be a user in your tenant |
| `expected_close_date` | `date` | No | |

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/deals \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Acme — 3 voice agents",
    "pipeline_id": "11aa…",
    "value": 240000,
    "company_id": "5c6d…",
    "expected_close_date": "2026-09-15"
  }'
```

Creating in a pipeline that has no stages yet returns `422 This pipeline has no stages yet` — add stages first.

## POST `/api/v1/crm/deals/{deal_id}/move`

| Field | Type | Required |
| --- | --- | --- |
| `stage_id` | `UUID` | Yes |

The one call you want for a kanban drag. The stage must belong to the deal's pipeline — `422 Stage does not belong to this pipeline` otherwise.

### What a move does to `status`

| Destination stage | Effect |
| --- | --- |
| `is_won` | `status` becomes `won`, `closed_at` stamped if unset |
| `is_lost` | `status` becomes `lost`, `closed_at` stamped if unset |
| Neither | `status` returns to `open` and `closed_at` is cleared |

So re-opening a deal is just moving it back to a working stage.

## GET / PATCH / DELETE `/api/v1/crm/deals/{deal_id}`

`PATCH` accepts the editable fields, all optional, and applies the same stage rules. `DELETE` returns `204`.

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `crm_deals:read` / `crm_deals:write` |
| `404` | Deal, pipeline, stage, contact, company or owner not in your tenant |
| `409` | Duplicate pipeline name, or a pipeline/stage that still holds deals |
| `422` | Blank name/title, a stage outside the deal's pipeline, a stage flagged both won and lost, an incomplete `reorder` list, or an empty pipeline |

Nothing on this page consumes credits.
