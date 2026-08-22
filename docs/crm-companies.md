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

Contacts link to a company through the contact's own `company_id`.

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
  "metadata": null,
  "created_at": "2026-08-04T10:00:00Z",
  "updated_at": "2026-08-16T09:00:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `domain` | `string \| null` | The company's primary email/web domain. **Unique per tenant** when set |
| `external_ids` | `object \| null` | Your own foreign keys into other systems. Free-form JSON |
| `metadata` | `object \| null` | Free-form JSON attached by the platform |

## GET `/api/v1/companies`

Newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `q` | `string` | No | At most 255 characters. Case-insensitive substring match against `name` **or** `domain` |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/companies?q=acme&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

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

```bash
curl -X POST https://api.callmissed.com/api/v1/companies \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Acme Retail", "domain": "acme.com", "industry": "Retail" }'
```

Returns `201`.

> `domain` is the natural key. Reusing one returns `409 A company with this domain already exists` — that is what stops two syncs from creating the same account twice. Look the domain up with `?q=` before creating, or use [duplicate detection and merge](/docs/crm-import-export#duplicates-and-merge) to clean up after the fact.

## GET / PATCH / DELETE `/api/v1/companies/{company_id}`

`PATCH` accepts the same fields, all optional, with the same bounds. `DELETE` returns `204`.

`404 Company not found` for an unknown or another tenant's id.

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `companies:read` / `companies:write` |
| `404` | Company not in your tenant |
| `409` | `A company with this domain already exists` |
| `422` | `name must not be blank`, or a field over its length limit |

Nothing on this page consumes credits.
