---
title: "Leads"
description: "Capture and qualify prospects before they are customers, track follow-ups, and convert a lead into a contact, company and deal in one call."
slug: "crm-leads"
breadcrumb: "API Reference"
---

# Leads

Capture and qualify prospects before they are customers, track follow-ups, and convert a lead into a contact, company and deal in one call.

## Overview

A **lead** is a prospect you have not yet qualified. It is lighter than a contact: a name, one way to reach them, where they came from, who owns the follow-up and what happens next. When a lead is worth pursuing, **convert** it — one call creates the contact, the company and (optionally) a deal, and links them back to the lead.

Leads move through a fixed status lifecycle. Quotes can be raised against a lead, and notes and tasks can be attached to one (`entity_type: "lead"`, see [Notes & Tasks](/docs/crm-notes-tasks)).

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, get, stats | `crm_leads:read` |
| Create, update, delete, convert | `crm_leads:write` |

## The lead object

```json
{
  "id": "7a10…",
  "tenant_id": "a0b1…",
  "name": "Priya Nair",
  "email": "priya@acme.example",
  "phone": "+919800000000",
  "company_name": "Acme Traders",
  "job_title": "Operations Head",
  "source": "web_form",
  "status": "qualified",
  "owner_user_id": "b1f2…",
  "score": 40,
  "estimated_value_minor": 24000000,
  "currency": "INR",
  "next_follow_up_at": "2026-10-12T09:30:00Z",
  "last_contacted_at": "2026-10-08T11:00:00Z",
  "lost_reason": null,
  "message": "Need a voice agent for missed-call follow-ups.",
  "attribution": { "utm_source": "newsletter" },
  "external_ref": null,
  "converted_contact_id": null,
  "converted_company_id": null,
  "converted_deal_id": null,
  "converted_at": null,
  "created_at": "2026-10-07T09:00:00Z",
  "updated_at": "2026-10-08T11:00:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `source` | `string` | `manual`, `web_form`, `book_demo`, `partner`, `whatsapp`, `voice`, `referral`, `import`, `api` or `other`. Default `manual` |
| `status` | `string` | `new`, `contacted`, `qualified`, `unqualified`, `converted` or `lost`. Default `new` |
| `estimated_value_minor` | `integer \| null` | **Integer minor units** of `currency` (paise for `INR`, cents for `USD`). `24000000` is INR 240,000.00 |
| `currency` | `string` | ISO 4217, three letters, upper-cased. Default `INR` |
| `score` | `integer` | `0` to `100`. Default `0` |
| `external_ref` | `string \| null` | Your own de-duplication key, up to 64 characters. Set on create only. Unique per tenant when set |
| `attribution` | `object \| null` | Free-form JSON about where the lead came from |
| `converted_*` | `UUID \| null` | Filled in by [convert](#convert-a-lead) |

### Status lifecycle

| Status | Meaning |
| --- | --- |
| `new` | Just captured |
| `contacted` | You have reached out |
| `qualified` | Worth pursuing |
| `unqualified` | Not a fit |
| `lost` | Pursued, did not close. Record `lost_reason` |
| `converted` | Set by the convert call only. Creating or updating a lead with `status: "converted"` is `422`, and a converted lead's status cannot be changed |

## GET `/api/v1/crm/leads`

Newest first.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `status` | `string` | One of the statuses above. Anything else is `422` |
| `source` | `string` | One of the sources above |
| `owner_user_id` | `UUID` | |
| `q` | `string` | Up to 320 characters. Case-insensitive substring match on name, email, phone and company name |
| `follow_up_before` | `datetime` | Leads whose `next_follow_up_at` is at or before this time |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

## GET `/api/v1/crm/leads/stats`

Total and counts of your leads by status. Every status is present, with `0` when there are none.

```json
{
  "total": 40,
  "by_status": {
    "new": 14,
    "contacted": 9,
    "qualified": 5,
    "unqualified": 3,
    "converted": 7,
    "lost": 2
  }
}
```

## POST `/api/v1/crm/leads`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, not blank |
| `email` | `string` | No | Up to 320 characters. Stored lower-cased |
| `phone` | `string` | No | Up to 32 characters |
| `company_name` | `string` | No | Up to 255 characters |
| `job_title` | `string` | No | Up to 255 characters |
| `source` | `string` | No | Default `manual` |
| `status` | `string` | No | Any status except `converted`. Default `new`. Creating a lead as `contacted` without `last_contacted_at` stamps it with the current time |
| `owner_user_id` | `UUID` | No | Must be a user in your tenant |
| `score` | `integer` | No | `0` to `100`, default `0` |
| `estimated_value_minor` | `integer` | No | `0` to `10^15`, minor units |
| `currency` | `string` | No | Exactly 3 letters (upper-cased), default `INR` |
| `next_follow_up_at` | `datetime` | No | |
| `last_contacted_at` | `datetime` | No | |
| `lost_reason` | `string` | No | Up to 255 characters |
| `message` | `string` | No | The original enquiry. Up to 10,000 characters |
| `attribution` | `object` | No | |
| `external_ref` | `string` | No | Up to 64 characters, unique per tenant |

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/leads \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Priya Nair",
    "email": "priya@acme.example",
    "company_name": "Acme Traders",
    "source": "referral",
    "estimated_value_minor": 24000000,
    "currency": "INR"
  }'
```

Returns `201` with the lead object. A reused `external_ref` returns `409`. An `owner_user_id` outside your tenant returns `404`.

## GET / PATCH / DELETE `/api/v1/crm/leads/{lead_id}`

`PATCH` accepts every create field except `external_ref`, all optional. Explicit `null` is `422` for `name`, `source`, `status`, `score` and `currency`. Moving a lead to `contacted` stamps `last_contacted_at` with the current time unless you send it. Changing the status of a `converted` lead is `422`. `DELETE` permanently deletes the lead and returns `204`; contacts, companies and deals created by a conversion are kept.

## Convert a lead

### POST `/api/v1/crm/leads/{lead_id}/convert`

In one transaction: reuses a contact or creates one from the lead (an existing contact in your account with the same phone, then the same email, is reused), reuses a company with the same name (case-insensitive) or creates one when `company_name` is set, and optionally creates a deal. Sets the lead to `converted`, stamps `converted_at` and fills the three `converted_*` ids. The contact's lifecycle stage moves to `opportunity` when a deal is created, otherwise `qualified`, and is never moved backwards.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `create_deal` | `boolean` | No | Default `true` |
| `pipeline_id` | `UUID` | No | Pipeline for the new deal. Omit to use the pipeline of `stage_id`, else your default pipeline |
| `stage_id` | `UUID` | No | Must belong to that pipeline. Omit to use the pipeline's first stage |
| `deal_title` | `string` | No | 1–255 characters, not blank. Defaults to the lead's name |
| `deal_value_minor` | `integer` | No | `0` to `10^15`, minor units. Defaults to the lead's `estimated_value_minor` |
| `existing_contact_id` | `UUID` | No | Link to a contact you already have instead of creating one |

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/leads/7a10…/convert \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "create_deal": true, "deal_title": "Acme — voice agents" }'
```

```json
{
  "lead": { "id": "7a10…", "status": "converted", "converted_contact_id": "4411…", "converted_at": "2026-10-09T10:00:00Z", "…": "full lead object" },
  "contact_id": "4411…",
  "company_id": "5c6d…",
  "deal_id": "33cc…"
}
```

`lead` is the full lead object. `company_id` and `deal_id` are `null` when no company or deal was created or linked.

Converting a lead that is already `converted` returns `409`, as does a new contact that would collide on phone or email. A `pipeline_id`, `stage_id` or `existing_contact_id` outside your tenant returns `404`. With `create_deal: true`, `422` is returned when you have no pipeline (create one or send `create_deal: false`), the pipeline has no stages, the stage is outside the pipeline, or the deal value is too large.

---

## Errors

| Status | When |
| --- | --- |
| `403` | API key is missing `crm_leads:read` / `crm_leads:write` (dashboard logins are not scope-checked) |
| `404` | Lead, owner, pipeline, stage or contact not in your tenant |
| `409` | Duplicate `external_ref`, converting an already converted lead, or a phone/email collision on the new contact |
| `422` | Blank name, unknown `status` or `source`, `converted` as a status, a null on a required field in `PATCH`, a convert problem listed above, or a value outside the bounds above |

Nothing on this page consumes credits.
