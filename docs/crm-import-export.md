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
- **CSV** — export a filtered set, preview a file's column mapping, or import with a dry run first.

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
| CSV inspect, CSV import | `crm_csv:write` |

`crm_bulk` has **no read half** — there is nothing to read, only two destructive writes.

---

# Search

## GET `/api/v1/crm/search`

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `q` | `string` | Yes | 2–320 characters |
| `types` | `string` | No | At most 128 characters. Comma-separated subset of `contact,company,deal,note,task`. Default: all five |
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

`reason` is what made them look alike: `email`, `phone`, `name` for contacts, and `domain` or `name` for companies. `key` is the normalised value they share — a lower-cased email, the last 10 digits of a phone number, a lower-cased name with whitespace collapsed, or a domain with any `http(s)://`, leading `www.` and trailing `/` removed. A record can appear in more than one group. Groups are largest first, and `limit` caps the total across all reasons.

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
  "repointed": {
    "conversations": 3,
    "crm_deals": 2,
    "commerce_events": 0,
    "crm_notes": 4,
    "crm_tasks": 1,
    "contact_memory": 0
  }
}
```

`fields_filled` lists the fields that were **blank on the primary** and taken from a duplicate — merging never overwrites a value the primary already had. `external_ids` maps are combined key by key, with the primary winning any conflict.

`repointed` counts the related rows moved onto the primary. For a contact merge the keys are `conversations`, `crm_deals`, `commerce_events`, `crm_notes`, `crm_tasks` and `contact_memory`; for a company merge they are `contacts`, `crm_deals`, `crm_notes` and `crm_tasks`.

`404 Contact not found` (or Company) when the primary or any duplicate is not in your tenant — nothing is merged.

> Merging is **destructive and irreversible**. It runs in a single transaction: either every duplicate is folded in and removed, or nothing changes. Preview with `/duplicates` first, and keep `duplicate_ids` small so a mistake is small.

---

# Bulk operations

Both endpoints take **1–500 ids** per request (repeated ids are collapsed) and are **all-or-nothing**: every id is verified to be in your tenant before a single row is touched.

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

Anything outside the list — including `id`, `tenant_id`, `created_at`, `updated_at` and `created_by_user_id` — returns `422` naming the key and the allowed set. A value of the wrong type returns `422 Invalid value for '<key>'`.

Ids inside `changes` must also be yours: an unknown `assignee_user_id`, `owner_user_id`, `company_id` or `stage_id` returns `404 Assignee not found` (or Owner / Company / Stage). A deal `stage_id` must belong to the pipeline of **every** deal in `ids` — `422 Stage does not belong to this pipeline` otherwise — and it closes or re-opens the deals exactly as a [move](/docs/crm-deals#post-apiv1crmdealsdeal_idmove) does, unless you also send `status`. Setting a task's `status` stamps or clears `completed_at`; setting a deal's `status` stamps or clears `closed_at`.

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
| `q` | `string` | No | Contacts (matches phone, email, name) and companies (matches name, domain). At most 320 characters |
| `company_id` | `UUID` | No | Contacts and deals |
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

## POST `/api/v1/crm/csv/inspect`

Preview how a file's columns map onto CallMissed fields before you import it. Writes nothing and keeps no copy of the file. Requires `crm_csv:write`.

`multipart/form-data`.

| Part | Type | Required | Constraints |
| --- | --- | --- | --- |
| `file` | file | Yes | UTF-8 CSV with a header row. At most **5 MB** |
| `entity_type` | text | Yes | `contact`, `company` or `deal` |
| `use_ai` | text | No | Default `true`. When `true`, headers that no alias or value pattern matched are also tried with an AI pass. Send `false` for a purely rule-based mapping |

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/csv/inspect \
  -H "Authorization: Bearer cm_your_api_key" \
  -F "file=@contacts.csv" \
  -F "entity_type=contact"
```

```json
{
  "headers": ["Full Name", "Mobile No.", "Email ID", "Notes"],
  "sample_rows": [
    { "Full Name": "Asha Menon", "Mobile No.": "+919876543210", "Email ID": "asha@acme.com", "Notes": "" }
  ],
  "proposals": [
    { "header": "Full Name", "field": "name", "source": "alias" },
    { "header": "Mobile No.", "field": "phone", "source": "alias" },
    { "header": "Email ID", "field": "email", "source": "alias" },
    { "header": "Notes", "field": null, "source": null }
  ],
  "unresolved": ["Notes"],
  "importable_fields": ["id", "name", "email", "phone", "company_id", "company_domain", "whatsapp_opt_in", "email_opt_in", "sms_opt_in", "consent_source"]
}
```

| Field | Notes |
| --- | --- |
| `headers` | Your header row, in its original spelling |
| `sample_rows` | Up to 5 data rows, echoed for review |
| `proposals[].source` | `alias` (exact name match), `heuristic` (matched on the sample values) or `ai`. `null` when unmapped |
| `unresolved` | Headers still unmapped — map them yourself or leave them out |
| `importable_fields` | Every valid mapping target for this `entity_type` |

Turn the confirmed proposals into the `column_map` you send to `/import`, e.g. `{"Full Name": "name", "Mobile No.": "phone", "Email ID": "email"}`.

## POST `/api/v1/crm/csv/import`

`multipart/form-data`.

| Part | Type | Required | Constraints |
| --- | --- | --- | --- |
| `file` | file | Yes | UTF-8 CSV with a header row. At most **5 MB** and **10,000 rows** |
| `entity_type` | text | Yes | `contact`, `company` or `deal` |
| `dry_run` | text | No | **Defaults to `true`** |
| `column_map` | text | No | A JSON object string mapping your headers to import columns, e.g. `{"Mobile No.": "phone"}`. Usually built from [`/inspect`](#post-apiv1crmcsvinspect). Targets must be import columns for this `entity_type`, and two headers may not map to the same column |

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
    { "row": 42, "field": "company_domain", "message": "company_domain does not match a company in this account" }
  ],
  "errors_truncated": false
}
```

`row` is the 1-based line in the file, so the **first data row is `2`** — it lines up with what a spreadsheet shows. Bad rows are counted in `skipped` and do not stop the rest of the file. At most 100 errors are reported; `errors_truncated` tells you there were more. The report has the same shape for a dry run and a real import.

### Upsert keys

| Entity | Matched on, in order |
| --- | --- |
| `contact` | `id`, else phone, else email |
| `company` | `id`, else `domain` |
| `deal` | `id` only — a row without `id` always creates |

A matched row is updated, never duplicated. A contact row with none of `id`, `phone`, `email` (or a company row with neither `id` nor `domain`) is reported as an error rather than inserted.

### Import columns

| Entity | Accepted columns |
| --- | --- |
| `contact` | `id, name, email, phone, company_id, company_domain, whatsapp_opt_in, email_opt_in, sms_opt_in, consent_source` |
| `company` | `id, name, domain, phone, website, industry, size, notes` |
| `deal` | `id, title, value, currency, status, pipeline_id, stage_id, contact_id, company_id, owner_user_id, expected_close_date` |

`company_domain` on a contact row resolves to an existing company by domain — handy when your source system has no CallMissed ids.

Whole-file problems return `422` and nothing is written: a file that is not UTF-8 CSV, an empty file or missing header, a header with no recognised columns (the message lists what was expected), more than 10,000 rows, more than 5 MB, an upload that is not a CSV content type, or an invalid `column_map`. A concurrent change during the write returns `409 Import conflicted with a concurrent change; nothing was written. Retry.`

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing the matching `crm_search:*` / `crm_bulk:write` / `crm_csv:*` scope |
| `404` | An id in your list (or inside `changes`, or a merge id) is not in your tenant — nothing was changed |
| `409` | Records still referenced, or a concurrent import |
| `422` | Query too short, unknown `types`, over 500 ids, a non-updatable change key, a deal stage from another pipeline, a file over 5 MB or 10,000 rows, an unrecognised CSV header, or a bad `column_map` |

Nothing on this page consumes credits.
