---
title: "Contacts"
description: "The person object — create, search, update and delete contacts, the phone/email/WhatsApp uniqueness rule, consent flags, and the facts your agents remember about each contact."
slug: "crm-contacts"
breadcrumb: "API Reference"
---

# Contacts

The person object — create, search, update and delete contacts, the phone/email/WhatsApp uniqueness rule, consent flags, and the facts your agents remember about each contact.

## Overview

A **contact** is one person you talk to — on a call, on WhatsApp, or by email. It is identified by a phone number, an email address or a WhatsApp business-scoped user id (BSUID), and optionally belongs to a [company](/docs/crm-companies) through `company_id`.

Deals, notes and tasks attach to a contact, and the [timeline](/docs/crm-lead-scores#timeline) rolls up its activity.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, get, read memory | `contacts:read` |
| Create, update, delete, erase memory | `contacts:write` |

## The contact object

```json
{
  "id": "4411…",
  "tenant_id": "a0b1…",
  "company_id": "5c6d…",
  "phone": "+919876543210",
  "email": "asha@acme.com",
  "name": "Asha Menon",
  "external_ids": { "shopify": "gid://shopify/Customer/991" },
  "whatsapp_opt_in": true,
  "email_opt_in": false,
  "sms_opt_in": false,
  "consent_source": "checkout_form",
  "consent_updated_at": "2026-08-04T10:00:00Z",
  "whatsapp_bsuid": null,
  "whatsapp_parent_bsuid": null,
  "whatsapp_username": null,
  "metadata": null,
  "created_at": "2026-08-04T10:00:00Z",
  "updated_at": "2026-08-16T09:00:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `phone` | `string \| null` | **Unique per tenant** when set |
| `email` | `string \| null` | **Unique per tenant** when set |
| `whatsapp_bsuid` | `string \| null` | WhatsApp business-scoped user id, for a user who hides their phone number. **Unique per tenant** when set |
| `whatsapp_parent_bsuid` | `string \| null` | The parent BSUID, when WhatsApp supplies one |
| `whatsapp_username` | `string \| null` | WhatsApp username, stored lower-case without a leading `@` |
| `whatsapp_opt_in`, `email_opt_in`, `sms_opt_in` | `boolean` | Consent per channel. Default `false` |
| `consent_source` | `string \| null` | Where consent was captured, in your own words |
| `consent_updated_at` | `datetime \| null` | **Server-managed.** Stamped when any opt-in flag or `consent_source` changes |
| `external_ids` | `object \| null` | Your own foreign keys into other systems. Free-form JSON |
| `metadata` | `object \| null` | Free-form JSON attached by the platform |

## GET `/api/v1/contacts`

Newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `q` | `string` | No | At most 320 characters. Case-insensitive substring match against `phone`, `email`, `name`, `whatsapp_username` **or** `whatsapp_bsuid` |
| `company_id` | `UUID` | No | Only contacts linked to this company |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/contacts?q=acme.com&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns an array of contact objects.

## POST `/api/v1/contacts`

At least one of `phone`, `email` or `whatsapp_bsuid` is required — `422 a contact requires a phone, an email or a WhatsApp BSUID` otherwise.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `phone` | `string` | Conditional | At most 32 characters. Unique per tenant |
| `email` | `string` | Conditional | At most 320 characters. Unique per tenant |
| `whatsapp_bsuid` | `string` | Conditional | A 2-letter upper-case country code, a period, an optional `ENT.`, then 1–128 letters/digits, e.g. `US.13491208655302741918`. Unique per tenant |
| `whatsapp_parent_bsuid` | `string` | No | Same format as `whatsapp_bsuid` |
| `whatsapp_username` | `string` | No | 3–35 of `a-z`, `0-9`, `.`, `_`, with at least one letter; may not start with `www` or `.`, end with `.`, or contain `..`. A leading `@` is stripped and the value is lower-cased |
| `name` | `string` | No | At most 255 characters |
| `company_id` | `UUID` | No | Must be a company in your tenant |
| `external_ids` | `object` | No | Free-form JSON |
| `whatsapp_opt_in` | `boolean` | No | Default `false` |
| `email_opt_in` | `boolean` | No | Default `false` |
| `sms_opt_in` | `boolean` | No | Default `false` |
| `consent_source` | `string` | No | At most 64 characters |

```bash
curl -X POST https://api.callmissed.com/api/v1/contacts \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "+919876543210",
    "email": "asha@acme.com",
    "name": "Asha Menon",
    "company_id": "5c6d…",
    "whatsapp_opt_in": true,
    "consent_source": "checkout_form"
  }'
```

Returns `201` with the contact object.

> `phone`, `email` and `whatsapp_bsuid` are natural keys. Reusing any of them returns `409 A contact with this phone, email or WhatsApp BSUID already exists`. Search with `?q=` before creating, or use [duplicate detection and merge](/docs/crm-import-export#get-apiv1crmsearchduplicates) to clean up after the fact.

## GET / PATCH / DELETE `/api/v1/contacts/{contact_id}`

`PATCH` accepts the same fields, all optional, with the same bounds. Only the fields you send are changed. Send `"company_id": null` to un-link the contact from its company.

`DELETE` returns `204`. Deals and conversations that pointed at the contact are kept, with their contact link cleared. The contact's memory is erased with it.

`404 Contact not found` for an unknown or another tenant's id. `404 Company not found` when `company_id` is not one of your companies.

---

# Contact memory

Your AI agents can remember durable facts about a contact across conversations. These endpoints let you see exactly what is stored and erase any of it.

## GET `/api/v1/contacts/{contact_id}/memory`

Scope `contacts:read`.

```json
{
  "contact_id": "4411…",
  "facts": [
    {
      "key": "3f9a1c2b7d4e5f60",
      "fact": "Prefers to be contacted in Hindi",
      "confidence": 0.9,
      "source_conversation_id": "c0d1…",
      "created_at": "2026-08-10T09:00:00+00:00",
      "updated_at": "2026-08-16T12:00:00+00:00",
      "seen_count": 3,
      "injected": true
    }
  ],
  "max_facts": 50,
  "updated_at": "2026-08-16T12:00:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `facts` | `array` | Ranked in the order the agent sees them. Empty (not `404`) when nothing has been learned yet |
| `facts[].key` | `string` | Stable handle for deleting this one fact |
| `facts[].seen_count` | `integer` | How many separate conversations confirmed the fact |
| `facts[].injected` | `boolean` | Whether the fact is among those the agent is currently given |
| `max_facts` | `integer` | How many facts can be stored per contact |

## DELETE `/api/v1/contacts/{contact_id}/memory`

Scope `contacts:write`. Erases every fact about the contact. Returns `204`, and is idempotent — erasing a contact with no memory is also `204`.

## DELETE `/api/v1/contacts/{contact_id}/memory/{fact_key}`

Scope `contacts:write`. Erases one fact, addressed by its `key` (1–64 characters) from the `GET` response. Returns `204`, or `404 Fact not found` when the key matches nothing.

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `contacts:read` / `contacts:write` |
| `404` | Contact, linked company, or memory fact not in your tenant |
| `409` | `A contact with this phone, email or WhatsApp BSUID already exists` |
| `422` | No phone, email or BSUID on create, a malformed BSUID or WhatsApp username, or a field over its length limit |

Nothing on this page consumes credits.
