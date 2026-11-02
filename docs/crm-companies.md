---
title: "Companies"
description: "The account object — create, search and link companies, the domain uniqueness rule, and how contacts attach to them."
slug: "crm-companies"
breadcrumb: "API Reference"
---

# Companies

The account object — create, search and link companies, the domain uniqueness rule, and how contacts attach to them.

## Overview

A **company** (an account) is the organisation a contact belongs to. It is the spine the rest of the CRM hangs off: deals point at a company, notes and tasks attach to one, and the [timeline](/docs/crm-lead-scores#timeline) rolls up its activity.

Contacts link to a company through the contact's own `company_id` — list a company's people with [`GET /api/v1/contacts?company_id=…`](/docs/crm-contacts#get-apiv1contacts).

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, get | `companies:read` |
| Create, update, delete | `companies:write` |

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
| `external_ids` | `object \| null` | Your own foreign keys into other systems. Free-form JSON |
| `tags` | `string[]` | Your labels, in the order added. Up to 20, each at most 40 characters. See [tags](/docs/crm-import-export#tags) |
| `metadata` | `object \| null` | Free-form JSON attached by the platform |

## GET `/api/v1/companies`

Newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `q` | `string` | No | At most 255 characters. Case-insensitive substring match against `name` **or** `domain` |
| `tag` | `string` (repeatable) | No | Only companies carrying **any** of these tags, e.g. `?tag=vip&tag=renewal`. Exact match. At most 20 |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/companies?q=acme&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

## GET `/api/v1/companies/count`

Takes the same `q` and `tag` filters as the list and returns `{"total": 42}` — how many companies match.

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
| `external_ids` | `object` | No | Free-form JSON |
| `tags` | `string[]` | No | Up to 20, each at most 40 characters. Trimmed; blanks and repeats dropped |

```bash
curl -X POST https://api.callmissed.com/api/v1/companies \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Acme Retail", "domain": "acme.com", "industry": "Retail" }'
```

Returns `201`.

> `domain` is the natural key. Reusing one returns `409 A company with this domain already exists` — that is what stops two syncs from creating the same account twice. Look the domain up with `?q=` before creating, or use [duplicate detection and merge](/docs/crm-import-export#get-apiv1crmsearchduplicates) to clean up after the fact.

## GET / PATCH / DELETE `/api/v1/companies/{company_id}`

`PATCH` accepts the same fields, all optional, with the same bounds. Only the fields you send are changed; `name` cannot be cleared (`422 name must not be blank`). `tags` replaces the whole list (`null` or `[]` clears it); use [bulk tags](/docs/crm-import-export#post-apiv1crmbulkadd-tags) to add or remove one tag across many companies.

`DELETE` returns `204`. Contacts and deals that pointed at the company are kept, with their `company_id` cleared.

`404 Company not found` for an unknown or another tenant's id.

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `companies:read` / `companies:write` |
| `404` | Company not in your tenant |
| `409` | `A company with this domain already exists` |
| `422` | `name must not be blank`, more than 20 tags or a tag over 40 characters, or a field over its length limit |

Nothing on this page consumes credits.
