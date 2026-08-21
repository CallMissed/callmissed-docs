---
title: "Search, Bulk & CSV"
description: "Cross-object search, duplicate detection and merge, all-or-nothing bulk updates, and CSV import and export with dry-run validation."
slug: "crm-import-export"
breadcrumb: "API Reference"
---

# Search, Bulk & CSV

Cross-object search, duplicate detection and merge, all-or-nothing bulk updates, and CSV import and export with dry-run validation.

## Overview

Four data-management surfaces on top of the CRM objects:

- **Search** — one query across contacts, companies, deals, notes and tasks.
- **Duplicates & merge** — find records that are the same thing twice, then fold them together.
- **Bulk** — update or delete up to 500 records in one all-or-nothing call.
- **CSV** — export a filtered set, or import with a dry run first.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| Search, duplicates | `crm_search:read` |
| Merge | `crm_search:write` |
| Bulk update, bulk delete | `crm_bulk:write` |
| CSV export | `crm_csv:read` |
| CSV import | `crm_csv:write` |

`crm_bulk` has **no read half** — there is nothing to read, only two destructive writes.

---

# Search

## GET `/api/v1/crm/search`

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `q` | `string` | Yes | 2–320 characters |
| `types` | `string` | No | Comma-separated subset of `contact,company,deal,note,task`. Default: all five |
| `limit` | `integer` | No | `1 <= limit <= 50`, default `10`. **Per type, not overall** |

```bash
curl "https://api.callmissed.com/api/v1/crm/search?q=acme&types=contact,company&limit=5" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "contacts": [
    { "id": "4411…", "type": "contact", "title": "Asha Menon", "subtitle": "asha@acme.com" }
  ],
  "companies": [
    { "id": "5c6d…", "type": "company", "title": "Acme Retail", "subtitle": "acme.com" }
  ],
  "deals": [],
  "notes": [],
  "tasks": []
}
```

Every key is always present — an object type with no hits returns an empty array rather than being omitted, so your client never has to guard for a missing key. With `limit=50` and all five types you get at most 250 rows.

## GET `/api/v1/crm/search/duplicates`

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact` or `company` |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50`. Counts **groups**, not records |

```json
{
  "entity_type": "contact",
  "groups": [
    { "reason": "email", "key": "asha@acme.com", "ids": ["4411…", "9922…"], "count": 2 }
  ]
}
```

`reason` is what made them look alike: `email`, `phone`, `name` for contacts, and `domain` or `name` for companies. `key` is the normalised value they share.

## POST `/api/v1/crm/search/merge`

Folds duplicates into one surviving record.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact` or `company` |
| `primary_id` | `UUID` | Yes | The record that survives. Must not appear in `duplicate_ids` |
| `duplicate_ids` | `UUID[]` | Yes | 1–20 entries, no repeats |

```json
{
  "entity_type": "contact",
  "primary_id": "4411…",
  "merged_ids": ["9922…"],
  "fields_filled": ["email", "company_id"],
  "repointed": { "notes": 4, "tasks": 1, "deals": 2 }
}
```

`fields_filled` lists the fields that were **blank on the primary** and taken from a duplicate — merging never overwrites a value the primary already had. `repointed` counts the related rows moved onto the primary.

> Merging is **destructive and irreversible**. It runs in a single transaction: either every duplicate is folded in and removed, or nothing changes. Preview with `/duplicates` first, and keep `duplicate_ids` small so a mistake is small.

---

# Bulk operations

Both endpoints take up to **500 ids** (de-duplicated before the cap is checked) and are **all-or-nothing**: every id is verified to be in your tenant before a single row is touched.

```json
{ "entity_type": "task", "requested": 3, "affected": 3 }
```

`requested` is the raw count you sent; `affected` is the de-duplicated count actually changed.

## POST `/api/v1/crm/bulk/update`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact`, `company`, `deal` or `task` |
| `ids` | `UUID[]` | Yes | 1–500 entries |
| `changes` | `object` | Yes | Non-empty. At most 16,000 characters serialised |

### What `changes` may contain

| Entity | Settable keys |
| --- | --- |
| `task` | `status` (`open` / `done`), `assignee_user_id` (nullable), `due_at` (nullable) |
| `deal` | `stage_id` (**not** nullable), `owner_user_id` (nullable), `status` (`open` / `won` / `lost`) |
| `contact` | `company_id` (nullable) |
| `company` | `industry` (nullable), `size` (nullable) |

Anything outside the list — including `id`, `tenant_id`, `created_at`, `updated_at` and `created_by_user_id` — returns `422` naming the key and the allowed set.

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/bulk/update \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "entity_type": "task",
    "ids": ["bb33…", "cc44…"],
    "changes": { "status": "done", "assignee_user_id": null }
  }'
```

If any id is missing you get `404 2 of 50 task ids were not found in this tenant; nothing was changed` — the count tells you how badly your list has drifted, and nothing was half-applied.

## POST `/api/v1/crm/bulk/delete`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact`, `company`, `deal` or `task` |
| `ids` | `UUID[]` | Yes | 1–500 entries |

`409 One or more company records are still referenced; nothing was deleted` when something points at them — clear the references first.

---

# CSV

## GET `/api/v1/crm/csv/export`

| Parameter | Type | Required | Applies to |
| --- | --- | --- | --- |
| `entity_type` | `string` | Yes | `contact`, `company` or `deal` |
| `q` | `string` | No | Contacts and companies. At most 320 characters |
| `company_id` | `UUID` | No | |
| `contact_id` | `UUID` | No | Deals |
| `pipeline_id` | `UUID` | No | Deals |
| `stage_id` | `UUID` | No | Deals |
| `owner_user_id` | `UUID` | No | Deals |
| `status` | `string` | No | Deals — `open`, `won` or `lost` |

```bash
curl "https://api.callmissed.com/api/v1/crm/csv/export?entity_type=contact&q=acme" \
  -H "Authorization: Bearer cm_your_api_key" \
  -o contacts.csv
```

Streams `text/csv` with `Content-Disposition: attachment; filename="contacts-YYYYMMDD.csv"`. Capped at **50,000 rows** — narrow the filters if you need more. Every cell is escaped so a spreadsheet cannot execute a value as a formula.

### Export columns

| Entity | Columns |
| --- | --- |
| `contact` | `id, name, email, phone, company_id, whatsapp_opt_in, email_opt_in, sms_opt_in, consent_source, created_at, updated_at` |
| `company` | `id, name, domain, phone, website, industry, size, notes, created_at, updated_at` |
| `deal` | `id, title, value, currency, status, pipeline_id, stage_id, contact_id, company_id, owner_user_id, expected_close_date, closed_at, last_activity_at, created_at, updated_at` |

## POST `/api/v1/crm/csv/import`

`multipart/form-data`.

| Part | Type | Required | Constraints |
| --- | --- | --- | --- |
| `file` | file | Yes | UTF-8 CSV with a header row. At most **5 MB** and **10,000 rows** |
| `entity_type` | text | Yes | `contact`, `company` or `deal` |
| `dry_run` | text | No | **Defaults to `true`** |

> `dry_run` defaults to **true**. A plain import call validates and reports without writing anything — send `dry_run=false` explicitly to commit. Run the dry pass first, read the errors, then commit the same file.

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/csv/import \
  -H "Authorization: Bearer cm_your_api_key" \
  -F "file=@contacts.csv" \
  -F "entity_type=contact" \
  -F "dry_run=true"
```

```json
{
  "entity_type": "contact",
  "dry_run": true,
  "total": 1200,
  "created": 1140,
  "updated": 52,
  "skipped": 8,
  "errors": [
    { "row": 42, "field": "email", "message": "value must be an email address" }
  ],
  "errors_truncated": false
}
```

`row` is the 1-based line in the file, so the **first data row is `2`** — it lines up with what a spreadsheet shows. At most 100 errors are reported; `errors_truncated` tells you there were more.

### Upsert keys

| Entity | Matched on, in order |
| --- | --- |
| `contact` | `id`, else phone, else email |
| `company` | `id`, else `domain` |
| `deal` | `id` only |

### Import columns

| Entity | Accepted columns |
| --- | --- |
| `contact` | `id, name, email, phone, company_id, company_domain, whatsapp_opt_in, email_opt_in, sms_opt_in, consent_source` |
| `company` | `id, name, domain, phone, website, industry, size, notes` |
| `deal` | `id, title, value, currency, status, pipeline_id, stage_id, contact_id, company_id, owner_user_id, expected_close_date` |

`company_domain` on a contact row resolves to an existing company by domain — handy when your source system has no CallMissed ids.

A header with no recognised columns returns `422` and lists what was expected. A concurrent change during the write returns `409 Import conflicted with a concurrent change; nothing was written. Retry.`

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing the matching `crm_search:*` / `crm_bulk:write` / `crm_csv:*` scope |
| `404` | An id in your list is not in your tenant — nothing was changed |
| `409` | Records still referenced, or a concurrent import |
| `422` | Query too short, unknown `types`, over 500 ids, a non-updatable change key, a file over 5 MB or 10,000 rows, or an unrecognised CSV header |

Nothing on this page consumes credits.
