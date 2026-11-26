---
title: "Companies"
description: "The account object — create, create-or-update (upsert), search, look up by your own id and link companies, the domain uniqueness rule, and how contacts attach to them."
slug: "crm-companies"
breadcrumb: "API Reference"
---

# Companies

The account object — create, create-or-update (upsert), search, look up by your own id and link companies, the domain uniqueness rule, and how contacts attach to them.

## Overview

A **company** (an account) is the organisation a contact belongs to. It is the spine the rest of the CRM hangs off: deals point at a company, notes and tasks attach to one, and the [timeline](/docs/crm-lead-scores#timeline) rolls up its activity.

Contacts link to a company through the contact's own `company_id` — list a company's people with [`GET /api/v1/contacts?company_id=…`](/docs/crm-contacts#get-apiv1contacts).

Syncing companies with another system? Use [create-or-update](#post-apiv1companiesupsert) to write, `?external_id=` to look one up by your own id, `?updated_after=` to read what changed, and the [`company.*` webhooks](/docs/webhooks#crm-record-events). The whole flow is in [Connect your CRM](/docs/connect-your-crm).

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, count, get | `companies:read` |
| Create, update, delete | `companies:write` |
| Create-or-update | `companies:write` **and** `companies:read` (it returns the company it matched) |

## The company object

```json
{
  "id": "5c6d…",
  "tenant_id": "a0b1…",
  "name": "Acme Retail",
  "domain": "acme.com",
  "phone": "+919876543210",
  "website": "https://acme.com",
  "industry": "Retail",
  "size": "51-200",
  "notes": "Two brands, one WABA.",
  "external_ids": { "shopify": "gid://shopify/Customer/991" },
  "tags": ["partner"],
  "metadata": null,
  "created_at": "2026-08-04T10:00:00Z",
  "updated_at": "2026-08-16T09:00:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `domain` | `string \| null` | The company's primary email/web domain. **Unique per tenant** when set |
| `external_ids` | `object \| null` | Your own ids for this company in other systems, one per system: `{"hubspot": "8841"}` is system `hubspot`, id `8841`. [Create-or-update](#post-apiv1companiesupsert) matches on them and `?external_id=` finds a company by one |
| `tags` | `string[]` | Your labels, in the order added. Up to 20, each at most 40 characters. See [tags](/docs/crm-import-export#tags) |
| `metadata` | `object \| null` | Free-form JSON attached by the platform |
| `updated_at` | `datetime` | When the company last changed. `?updated_after=` filters on it |

## GET `/api/v1/companies`

Newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `q` | `string` | No | At most 255 characters. Case-insensitive substring match against `name` **or** `domain` |
| `tag` | `string` (repeatable) | No | Only companies carrying **any** of these tags, e.g. `?tag=vip&tag=renewal`. Exact match. At most 20 |
| `external_id` | `string` | No | `system:id`, e.g. `hubspot:8841`. At most 320 characters. Only the company carrying that id in `external_ids`. The split is on the first colon. Anything not shaped `system:id` is `422` |
| `updated_after` | `datetime` | No | ISO 8601, e.g. `2026-11-17T06:30:00Z` (no offset is read as UTC). Only records whose `updated_at` is later than this. The result is then ordered **oldest change first** (`updated_at` ascending, then `id`) and `sort` is ignored. See [incremental sync](/docs/connect-your-crm#3-catch-up-after-downtime) |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/companies?q=acme&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

Look one company up by your own id (an empty array means you have not sent it yet), or read what changed since your last sync, oldest change first:

```bash
curl "https://api.callmissed.com/api/v1/companies?external_id=hubspot:8841" \
  -H "Authorization: Bearer cm_your_api_key"

curl "https://api.callmissed.com/api/v1/companies?updated_after=2026-11-17T06:30:00Z&limit=200" \
  -H "Authorization: Bearer cm_your_api_key"
```

A deleted company is not in the list. Deletions reach you only as the [`company.deleted` webhook](/docs/webhooks#crm-record-events).

## GET `/api/v1/companies/count`

Takes the same `q`, `tag` and `external_id` filters as the list (not `updated_after`) and returns `{"total": 42}` — how many companies match.

## POST `/api/v1/companies`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, not blank |
| `domain` | `string` | No | At most 255 characters. Unique per tenant |
| `phone` | `string` | No | At most 32 characters |
| `website` | `string` | No | At most 512 characters |
| `industry` | `string` | No | At most 128 characters |
| `size` | `string` | No | At most 32 characters |
| `notes` | `string` | No | At most 10,000 characters |
| `external_ids` | `object` | No | Your own ids, one per system, e.g. `{"hubspot": "8841"}` |
| `tags` | `string[]` | No | Up to 20, each at most 40 characters. Trimmed; blanks and repeats dropped |

```bash
curl -X POST https://api.callmissed.com/api/v1/companies \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Acme Retail", "domain": "acme.com", "industry": "Retail" }'
```

Returns `201`.

> `domain` is the natural key. Reusing one returns `409 A company with this domain already exists` — that is what stops two syncs from creating the same account twice. When you are syncing from another system, call [create-or-update](#post-apiv1companiesupsert) instead. Otherwise look the domain up with `?q=` before creating, or use [duplicate detection and merge](/docs/crm-import-export#get-apiv1crmsearchduplicates) to clean up after the fact.

## POST `/api/v1/companies/upsert`

Create-or-update in one call. Scopes `companies:write` and `companies:read`, both required, because the call returns the company it matched. The company that already carries the same external id, domain or name is updated; when there is none, a new company is created.

The body is the [create body](#post-apiv1companies) (`name` is required) plus `match_on`:

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `match_on` | `string[]` | No | 1 to 3 of `external_id`, `domain`, `name`, in order of preference. Default `["external_id", "domain", "name"]` |
| `external_ids` | `object` | No | System name to id, e.g. `{"hubspot": "8841"}`. At most 20 systems. A system name is 1–64 characters with no colon; an id is a string or an integer of 1–255 characters and is stored as a string |
| `name` | `string` | Yes | 1–255 characters, not blank |
| every other create field | | No | Same types and bounds as create. On a match, only the fields you send are written |

`name` is always in the body, so with the default `match_on` a company with the same name is treated as the same company. If two of your accounts can share a name, send `"match_on": ["external_id", "domain"]` or `["external_id"]`.

```bash
curl -X POST https://api.callmissed.com/api/v1/companies/upsert \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "external_ids": { "hubspot": "8841" },
    "name": "Acme Retail",
    "domain": "acme.com",
    "industry": "Retail",
    "match_on": ["external_id", "domain"]
  }'
```

`201` when a company was created, `200` when one was updated. The response is the company object plus `created` and `matched_on`:

```json
{
  "id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "name": "Acme Retail",
  "domain": "acme.com",
  "phone": null,
  "website": null,
  "industry": "Retail",
  "size": null,
  "notes": null,
  "external_ids": { "hubspot": "8841" },
  "tags": [],
  "metadata": null,
  "created_at": "2026-08-04T10:00:00Z",
  "updated_at": "2026-11-17T06:31:10.412000Z",
  "created": false,
  "matched_on": "domain"
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `created` | `boolean` | `true` = a new company was created (`201`). `false` = an existing company was updated (`200`) |
| `matched_on` | `string \| null` | Which key found the existing company: `external_id`, `domain` or `name` — the first one in your `match_on` order that matched. `null` when `created` is `true` |

### How a company is matched

| Key | Two values are the same when |
| --- | --- |
| `external_id` | The company's `external_ids` holds the same id for the same system, for **any** system you sent |
| `domain` | Equal after lower-casing and removing `http://` / `https://`, a leading `www.` and trailing slashes. `https://www.Acme.com/` matches `acme.com` |
| `name` | Equal after trimming, collapsing runs of spaces to one, and lower-casing |

Every key in `match_on` that has a value is looked up. One company found: it is updated. None: a company is created. More than one different company found: nothing is written and you get `409 ambiguous_match`.

On an update only the fields present in the body are written, the stored `domain` is kept as written when it was the matching key, and `external_ids` are merged into the stored ones. A company found by domain or name that already carries a **different** id for one of your systems returns `409 external_id_conflict`.

Two upserts of the same company sent at the same moment do not create two companies: the second waits for the first, then updates it. A retry after a timeout is safe.

### Upsert errors

`409` bodies from this endpoint are objects, so read `detail.code`:

```json
{
  "detail": {
    "code": "ambiguous_match",
    "message": "More than one company matches this record. Merge them, or send match_on with only the key that identifies it.",
    "ids": ["5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f", "7e8f9a0b-1c2d-4e3f-9a4b-5c6d7e8f9a0b"]
  }
}
```

| Status | `detail` | What to do next |
| --- | --- | --- |
| `409` | `code: "ambiguous_match"`, `ids` = the matching companies (up to 10) | Same company twice: [merge](/docs/crm-import-export#post-apiv1crmsearchmerge) `ids`, then retry. Different companies: retry with a narrower `match_on`, e.g. `["external_id"]` |
| `409` | `code: "external_id_conflict"`, `ids` = the one company found | The domain or name belongs to a company already linked to another record in your system. Fix the link on your side, or retry with `"match_on": ["external_id"]` |
| `409` | `"A company with this domain already exists"` (a plain string) | `domain` is not in your `match_on` and is already on another company. Add `domain` to `match_on`, or drop it from the body |
| `422` | `nothing to match on: send a value for one of …` | `match_on` names only keys you sent no value for, e.g. `["external_id"]` without `external_ids` |
| `422` | A field-level validation message | Blank `name`, a malformed `external_ids` object, or a field outside its create bounds |
| `403` | `API key missing required scope: companies:write` (or `companies:read`) | The key needs both scopes. Add the missing one |

## GET / PATCH / DELETE `/api/v1/companies/{company_id}`

`PATCH` accepts the same fields, all optional, with the same bounds. Only the fields you send are changed; `name` cannot be cleared (`422 name must not be blank`). `tags` replaces the whole list (`null` or `[]` clears it); use [bulk tags](/docs/crm-import-export#post-apiv1crmbulkadd-tags) to add or remove one tag across many companies.

`DELETE` returns `204`. Contacts and deals that pointed at the company are kept, with their `company_id` cleared.

`404 Company not found` for an unknown or another tenant's id.

`PATCH` replaces `external_ids` with the object you send. To add one id and keep the rest, use [create-or-update](#post-apiv1companiesupsert), which merges.

## Webhooks

Subscribe a [webhook](/docs/webhooks#crm-record-events) to `company.created`, `company.updated` and `company.deleted` (an API key needs `companies:read` to subscribe) to be told about every change, whoever made it. Created and updated events carry the company object above; a deleted event carries `id` and `external_ids`, plus `merged_into` when the company was folded into another by a merge.

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `companies:read` / `companies:write` |
| `404` | Company not in your tenant |
| `409` | `A company with this domain already exists`; on create-or-update also `ambiguous_match` and `external_id_conflict`, see [upsert errors](#upsert-errors) |
| `422` | `name must not be blank`, an `external_id` filter not shaped `system:id`, an `updated_after` that is not a date-time, nothing to match on in an upsert, more than 20 tags or a tag over 40 characters, or a field over its length limit |

Nothing on this page consumes credits.
