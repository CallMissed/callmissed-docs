---
title: "Contacts"
description: "The person object — create, create-or-update (upsert), search, look up by your own id, update and delete contacts, the phone/email/WhatsApp uniqueness rule, consent flags, and the facts your agents remember about each contact."
slug: "crm-contacts"
breadcrumb: "API Reference"
---

# Contacts

The person object — create, create-or-update (upsert), search, look up by your own id, update and delete contacts, the phone/email/WhatsApp uniqueness rule, consent flags, and the facts your agents remember about each contact.

## Overview

A **contact** is one person you talk to — on a call, on WhatsApp, or by email. It is identified by a phone number, an email address or a WhatsApp business-scoped user id (BSUID), and optionally belongs to a [company](/docs/crm-companies) through `company_id`.

Deals, notes and tasks attach to a contact, and the [timeline](/docs/crm-lead-scores#timeline) rolls up its activity.

Syncing contacts with another system? Use [create-or-update](#post-apiv1contactsupsert) to write, [`?external_id=`](#get-apiv1contacts) to look one up by your own id, [`?updated_after=`](#get-apiv1contacts) to read what changed, and the [`contact.*` webhooks](/docs/webhooks#crm-record-events) to hear about changes as they happen. The whole flow is in [Connect your CRM](/docs/connect-your-crm).

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, count, get, read memory | `contacts:read` |
| Create, update, delete, erase memory | `contacts:write` |
| Create-or-update | `contacts:write` **and** `contacts:read` (it returns the contact it matched) |

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
  "lifecycle_stage": "customer",
  "lead_source": "web_form",
  "owner_user_id": null,
  "tags": ["vip", "renewal"],
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
| `external_ids` | `object \| null` | Your own ids for this person in other systems, one per system: `{"hubspot": "123"}` is system `hubspot`, id `123`. [Create-or-update](#post-apiv1contactsupsert) matches on them and `?external_id=` finds a contact by one |
| `lifecycle_stage` | `string \| null` | `subscriber`, `lead`, `qualified`, `opportunity`, `customer` or `churned` |
| `lead_source` | `string \| null` | Same values as a lead's `source` |
| `owner_user_id` | `UUID \| null` | The teammate who owns the contact |
| `tags` | `string[]` | Your labels, in the order added. Up to 20, each at most 40 characters. See [tags](/docs/crm-import-export#tags) |
| `metadata` | `object \| null` | Free-form JSON attached by the platform |
| `updated_at` | `datetime` | When the contact last changed. `?updated_after=` filters on it |

## GET `/api/v1/contacts`

Newest first unless you pass `sort`.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `q` | `string` | No | At most 320 characters. Case-insensitive substring match against `phone`, `email`, `name`, `whatsapp_username` **or** `whatsapp_bsuid` |
| `company_id` | `UUID` | No | Only contacts linked to this company |
| `lifecycle_stage` | `string` | No | `subscriber`, `lead`, `qualified`, `opportunity`, `customer` or `churned` |
| `owner_user_id` | `UUID` | No | Only contacts owned by this teammate |
| `lead_source` | `string` | No | Only contacts from this source (same values as a lead's `source`) |
| `tag` | `string` (repeatable) | No | Only contacts carrying **any** of these tags, e.g. `?tag=vip&tag=renewal`. Exact match. At most 20 |
| `external_id` | `string` | No | `system:id`, e.g. `hubspot:123`. At most 320 characters. Only the contact carrying that id in `external_ids`. The split is on the first colon, so the id itself may contain colons. Anything not shaped `system:id` is `422` |
| `sort` | `string` | No | `newest` (default), `oldest`, `name`, `updated`, or `score` — highest lead score first, unscored contacts last |
| `updated_after` | `datetime` | No | ISO 8601, e.g. `2026-11-17T06:30:00Z` (no offset is read as UTC). Only records whose `updated_at` is later than this. The result is then ordered **oldest change first** (`updated_at` ascending, then `id`) and `sort` is ignored. See [incremental sync](/docs/connect-your-crm#3-catch-up-after-downtime) |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/contacts?q=acme.com&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns an array of contact objects. Each also carries `score` and `score_grade` — the contact's lead score — or `null` when it has never been scored.

Look one contact up by your own id (an empty array means you have not sent it yet):

```bash
curl "https://api.callmissed.com/api/v1/contacts?external_id=hubspot:123" \
  -H "Authorization: Bearer cm_your_api_key"
```

Read what changed since your last sync, oldest change first:

```bash
curl "https://api.callmissed.com/api/v1/contacts?updated_after=2026-11-17T06:30:00Z&limit=200" \
  -H "Authorization: Bearer cm_your_api_key"
```

A deleted contact is not in this list. Deletions reach you only as the [`contact.deleted` webhook](/docs/webhooks#crm-record-events).

## GET `/api/v1/contacts/count`

Takes the same `q`, `company_id`, `lifecycle_stage`, `owner_user_id`, `lead_source`, `tag` and `external_id` filters as the list (not `updated_after`) and returns `{"total": 1234}` — how many contacts match.

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
| `external_ids` | `object` | No | Your own ids, one per system, e.g. `{"hubspot": "123"}` |
| `lifecycle_stage` | `string` | No | `subscriber`, `lead`, `qualified`, `opportunity`, `customer` or `churned` |
| `lead_source` | `string` | No | Same values as a lead's `source` |
| `owner_user_id` | `UUID` | No | Must be a user in your tenant |
| `whatsapp_opt_in` | `boolean` | No | Default `false` |
| `email_opt_in` | `boolean` | No | Default `false` |
| `sms_opt_in` | `boolean` | No | Default `false` |
| `consent_source` | `string` | No | At most 64 characters |
| `tags` | `string[]` | No | Up to 20, each at most 40 characters. Trimmed; blanks and repeats dropped |

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

> `phone`, `email` and `whatsapp_bsuid` are natural keys. Reusing any of them returns `409 A contact with this phone, email or WhatsApp BSUID already exists`. When you are syncing from another system, call [create-or-update](#post-apiv1contactsupsert) instead: it finds the existing contact for you. Otherwise search with `?q=` before creating, or use [duplicate detection and merge](/docs/crm-import-export#get-apiv1crmsearchduplicates) to clean up after the fact.

## POST `/api/v1/contacts/upsert`

Create-or-update in one call. Scopes `contacts:write` and `contacts:read`, both required, because the call returns the contact it matched. The contact that already carries the same external id, phone or email is updated; when there is none, a new contact is created. Use it for every write from another system, so a retry or a second sync run does not create the person twice.

The body is the [create body](#post-apiv1contacts) plus `match_on`:

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `match_on` | `string[]` | No | 1 to 3 of `external_id`, `phone`, `email`, in order of preference. Default `["external_id", "phone", "email"]` |
| `external_ids` | `object` | Conditional | System name to id, e.g. `{"hubspot": "123"}`. At most 20 systems. A system name is 1–64 characters with no colon; an id is a string or an integer of 1–255 characters and is stored as a string |
| `phone` | `string` | Conditional | At most 32 characters |
| `email` | `string` | Conditional | At most 320 characters |
| every other create field | | No | Same types and bounds as create. On a match, only the fields you send are written |

The body must carry a value for at least one key named in `match_on` (`422` otherwise). A key with no value in the body is skipped.

```bash
curl -X POST https://api.callmissed.com/api/v1/contacts/upsert \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "external_ids": { "hubspot": "123" },
    "phone": "+919876543210",
    "email": "asha@acme.com",
    "name": "Asha Menon",
    "whatsapp_opt_in": true,
    "consent_source": "hubspot_sync",
    "tags": ["vip"]
  }'
```

`201` when a contact was created, `200` when one was updated. The response is the contact object plus two fields:

```json
{
  "id": "44110c7e-2a51-4c0f-9d3b-6f1e2a7b8c90",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "company_id": null,
  "phone": "+919876543210",
  "email": "asha@acme.com",
  "name": "Asha Menon",
  "external_ids": { "hubspot": "123" },
  "whatsapp_opt_in": true,
  "email_opt_in": false,
  "sms_opt_in": false,
  "consent_source": "hubspot_sync",
  "consent_updated_at": "2026-11-17T06:31:10.412000Z",
  "whatsapp_bsuid": null,
  "whatsapp_parent_bsuid": null,
  "whatsapp_username": null,
  "lifecycle_stage": null,
  "lead_source": null,
  "owner_user_id": null,
  "tags": ["vip"],
  "metadata": null,
  "created_at": "2026-11-17T06:31:10.412000Z",
  "updated_at": "2026-11-17T06:31:10.412000Z",
  "created": true,
  "matched_on": null
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `created` | `boolean` | `true` = a new contact was created (`201`). `false` = an existing contact was updated (`200`) |
| `matched_on` | `string \| null` | Which key found the existing contact: `external_id`, `phone` or `email` — the first one in your `match_on` order that matched. `null` when `created` is `true` |

### How a contact is matched

| Key | Two values are the same when |
| --- | --- |
| `external_id` | The contact's `external_ids` holds the same id for the same system, for **any** system you sent |
| `phone` | The last ten digits are equal, and the country codes agree when both numbers carry one. `+91 98200 12345` and `098200 12345` match; `+91 98200 12345` and `+1 982 001 2345` do not |
| `email` | Equal after trimming spaces and lower-casing |

Every key in `match_on` that has a value is looked up. One contact found: it is updated. None: a contact is created. **More than one different contact** found (your id is on one contact and the phone on another): nothing is written and you get `409 ambiguous_match`.

### What an update writes

- Only the fields present in the body. A field you leave out keeps its stored value; `tags`, when sent, replaces the whole list.
- The stored `phone` or `email` is kept as written when it was the matching key, so `098200 12345` in your system does not rewrite `+919820012345` here.
- `external_ids` are merged into the stored ones: systems you do not send are kept.
- A contact found by phone or email that already carries a **different** id for one of your systems is another record in that system: nothing is written and you get `409 external_id_conflict`.

Two upserts of the same record sent at the same moment do not create two contacts: the second waits for the first, then updates the contact it created. A retry after a timeout is therefore safe.

### Upsert errors

`409` bodies from this endpoint are objects, so read `detail.code`:

```json
{
  "detail": {
    "code": "ambiguous_match",
    "message": "More than one contact matches this record. Merge them, or send match_on with only the key that identifies it.",
    "ids": ["44110c7e-2a51-4c0f-9d3b-6f1e2a7b8c90", "9922aa01-77b4-4e2c-8a10-0c5d6e7f8a9b"]
  }
}
```

| Status | `detail` | What to do next |
| --- | --- | --- |
| `409` | `code: "ambiguous_match"`, `ids` = the matching contacts (up to 10) | They are the same person: [merge](/docs/crm-import-export#post-apiv1crmsearchmerge) `ids`, then retry. They are different people: retry with `"match_on": ["external_id"]` |
| `409` | `code: "external_id_conflict"`, `ids` = the one contact found | The phone or email belongs to a contact already linked to another record in your system. Fix the link on your side, or retry with `"match_on": ["external_id"]` to create or update by id alone |
| `409` | `"A contact with this phone, email or WhatsApp BSUID already exists"` (a plain string) | A key you left out of `match_on` is already on another contact. Add that key to `match_on`, or drop the field from the body |
| `422` | `nothing to match on: send a value for one of …` | Send `external_ids`, `phone` or `email` for a key in `match_on` |
| `422` | `No contact matched, and a new contact requires a phone, an email or a WhatsApp BSUID` | Nothing matched your `external_ids`, and an id alone cannot create a contact. Add `phone`, `email` or `whatsapp_bsuid` |
| `422` | A field-level validation message | `external_ids` is not an object, has more than 20 systems, a system name with a colon, or an empty or non-string/integer id; or a field is outside its create bounds |
| `403` | `API key missing required scope: contacts:write` (or `contacts:read`) | The key needs both scopes. Add the missing one |
| `404` | `Company not found`, or an owner not found | `company_id` is not one of your companies, or `owner_user_id` is not a user in your tenant |

## GET / PATCH / DELETE `/api/v1/contacts/{contact_id}`

`PATCH` accepts the same fields, all optional, with the same bounds. Only the fields you send are changed. Send `"company_id": null` to un-link the contact from its company. `tags` replaces the whole list (`null` or `[]` clears it); to add or remove one tag across many contacts without touching their other tags, use [bulk tags](/docs/crm-import-export#post-apiv1crmbulkadd-tags).

`DELETE` returns `204`. Deals and conversations that pointed at the contact are kept, with their contact link cleared. The contact's memory is erased with it.

`404 Contact not found` for an unknown or another tenant's id. `404 Company not found` when `company_id` is not one of your companies.

`PATCH` replaces `external_ids` with the object you send. To add one id and keep the rest, use [create-or-update](#post-apiv1contactsupsert), which merges.

## Webhooks

Subscribe a [webhook](/docs/webhooks#crm-record-events) to `contact.created`, `contact.updated` and `contact.deleted` (an API key needs `contacts:read` to subscribe) to be told about every change, whoever made it: your API calls, a teammate in the console, an agent during a call or chat, or an import. Created and updated events carry the contact object above; a deleted event carries `id` and `external_ids`, plus `merged_into` when the contact was folded into another by a merge.

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
| `409` | `A contact with this phone, email or WhatsApp BSUID already exists`; on create-or-update also `ambiguous_match` and `external_id_conflict`, see [upsert errors](#upsert-errors) |
| `422` | No phone, email or BSUID on create, an `external_id` filter not shaped `system:id`, an `updated_after` that is not a date-time, nothing to match on in an upsert, a malformed BSUID or WhatsApp username, more than 20 tags or a tag over 40 characters, or a field over its length limit |

Nothing on this page consumes credits.
