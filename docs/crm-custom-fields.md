---
title: "Custom Fields & Saved Views"
description: "Extend contacts, companies and deals with typed custom fields, and store filtered views your team can share."
slug: "crm-custom-fields"
breadcrumb: "API Reference"
---

# Custom Fields & Saved Views

Extend contacts, companies and deals with typed custom fields, and store filtered views your team can share.

## Overview

**Custom fields** add typed columns to `contact`, `company` and `deal` records without changing the API shape of those objects. A field is defined once, then given a value per record.

**Saved views** store a filter, sort and column set for an object type, either private to you or shared with the team.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| Read field definitions and values | `crm_custom_fields:read` |
| Create, update, delete definitions and values | `crm_custom_fields:write` |
| Read saved views | `crm_views:read` |
| Create, update, delete saved views | `crm_views:write` |

---

# Custom fields

## Field definitions

```json
{
  "id": "44dd…",
  "tenant_id": "a0b1…",
  "entity_type": "company",
  "key": "account_tier",
  "label": "Account tier",
  "field_type": "select",
  "options": ["bronze", "silver", "gold"],
  "is_required": false,
  "position": 2,
  "created_at": "2026-08-02T09:00:00Z",
  "updated_at": "2026-08-02T09:00:00Z"
}
```

| Field type | Accepted `value` |
| --- | --- |
| `text` | A non-blank string, at most 2,000 characters |
| `number` | A finite number |
| `boolean` | `true` or `false` |
| `date` | An ISO 8601 date |
| `select` | One of the definition's `options` |

### GET `/api/v1/crm/custom-fields`

Ordered by entity type, then position, then key.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `entity_type` | `string` | `contact`, `company` or `deal` |
| `limit` | `integer` | `1 <= limit <= 200`, **default `100`** |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

### POST `/api/v1/crm/custom-fields`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact`, `company` or `deal` |
| `key` | `string` | Yes | At most 64 characters, matching `^[a-z][a-z0-9_]{0,63}$`. Unique per entity type |
| `label` | `string` | Yes | 1–255 characters, not blank |
| `field_type` | `string` | Yes | One of the five types above |
| `options` | `string[]` | Conditional | **Required for `select`, rejected otherwise.** At most 100 entries, each at most 128 characters, no blanks or duplicates |
| `is_required` | `boolean` | No | Default `false` |
| `position` | `integer` | No | `0 <= position <= 10000`, default `0` |

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/custom-fields \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "entity_type": "company",
    "key": "account_tier",
    "label": "Account tier",
    "field_type": "select",
    "options": ["bronze", "silver", "gold"]
  }'
```

A duplicate key returns `409 A custom field with this key already exists for this entity type`.

### PATCH `/api/v1/crm/custom-fields/{def_id}`

Accepts `label`, `options`, `is_required` and `position`.

> `key`, `entity_type` and `field_type` are **immutable** — changing a field's type would silently invalidate every stored value. Create a new field and migrate instead.

### DELETE `/api/v1/crm/custom-fields/{def_id}`

Returns `204` and **cascades to every value stored for that field**.

## Field values

```json
{
  "id": "55ee…",
  "tenant_id": "a0b1…",
  "field_def_id": "44dd…",
  "entity_type": "company",
  "entity_id": "5c6d…",
  "value": "gold",
  "key": "account_tier",
  "label": "Account tier",
  "field_type": "select",
  "created_at": "2026-08-04T10:00:00Z",
  "updated_at": "2026-08-16T09:00:00Z"
}
```

The row carries the definition's `key`, `label` and `field_type` alongside the value, so one call renders a record's custom section without a second lookup.

### GET `/api/v1/crm/custom-fields/values`

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact`, `company` or `deal` |
| `entity_id` | `UUID` | Yes | |
| `limit` | `integer` | No | `1 <= limit <= 200`, **default `100`** |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

Ordered by the definition's `position`, then `key` — the order you defined for display.

### PUT `/api/v1/crm/custom-fields/values`

Upsert. Always returns `200`, whether it created or replaced.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `field_def_id` | `UUID` | Yes | |
| `entity_id` | `UUID` | Yes | Must exist in your tenant |
| `entity_type` | `string` | No | Optional cross-check against the definition |
| `value` | any | Yes | Typed by the definition |

```bash
curl -X PUT https://api.callmissed.com/api/v1/crm/custom-fields/values \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "field_def_id": "44dd…", "entity_id": "5c6d…", "value": "gold" }'
```

> To clear a field, **delete the value** — sending `value: null` returns `422 value must not be null; delete the value to clear it`. The distinction keeps "never set" and "deliberately empty" from collapsing into one state.

Type mismatches return a specific `422`: `value must be a number`, `value must be an ISO 8601 date`, `value must be one of the field's options`, and so on. A simultaneous write from elsewhere returns `409 This field was updated concurrently; retry`.

### DELETE `/api/v1/crm/custom-fields/values/{value_id}`

Returns `204`.

---

# Saved views

```json
{
  "id": "66ff…",
  "tenant_id": "a0b1…",
  "entity_type": "deal",
  "name": "My open enterprise deals",
  "filters": { "status": "open", "value_gte": 100000 },
  "sort": { "field": "expected_close_date", "dir": "asc" },
  "columns": ["title", "value", "stage_id", "expected_close_date"],
  "layout": "kanban",
  "is_shared": false,
  "created_by_user_id": "b1f2…",
  "created_at": "2026-08-10T09:00:00Z",
  "updated_at": "2026-08-10T09:00:00Z"
}
```

`filters` is free-form JSON — the API stores and returns it; your client decides what the keys mean.

## Visibility

A view is reachable only when `is_shared` is `true`, **or** you created it. A teammate's private view returns `404`, not `403` — its existence is not disclosed.

> API keys have no user identity, so a view created with a key has `created_by_user_id: null`. **Set `is_shared: true` on views you want a key to read back**, otherwise the key will not see them.

## GET `/api/v1/crm/saved-views`

Newest first.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `entity_type` | `string` | `contact`, `company`, `deal` or `task` |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

## POST `/api/v1/crm/saved-views`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact`, `company`, `deal` or `task` |
| `name` | `string` | Yes | 1–255 characters, unique per entity type |
| `filters` | `object` | No | Default `{}`. At most 16,000 characters serialised |
| `sort` | `object` | No | `{ "field": "...", "dir": "asc" \| "desc" }`. `field` at most 64 characters, `dir` defaults to `desc` |
| `columns` | `string[]` | No | At most 100, each non-blank and at most 128 characters |
| `layout` | `string` | No | `table` (default) or `kanban` |
| `is_shared` | `boolean` | No | Default `false` |

## GET / PATCH / DELETE `/api/v1/crm/saved-views/{view_id}`

`PATCH` accepts every field except `entity_type` — a view belongs to one object type for life. `DELETE` returns `204`.

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `crm_custom_fields:*` / `crm_views:*` |
| `404` | Definition, value or view not visible to you |
| `409` | Duplicate field key or view name, or a concurrent value write |
| `422` | Bad key pattern, `options` on a non-select field (or missing on a select), a value that does not match the field type, `value: null`, or oversized `filters` |

Nothing on this page consumes credits.
