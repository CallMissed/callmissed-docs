---
title: "Quotes"
description: "Draft priced quotes with line items and tax, send them to customers by email or WhatsApp, track acceptance, and convert an accepted quote into an invoice."
slug: "crm-quotes"
breadcrumb: "API Reference"
---

# Quotes

Draft priced quotes with line items and tax, send them to customers by email or WhatsApp, track acceptance, and convert an accepted quote into an invoice.

## Overview

A **quote** is a priced offer to a customer. You build it as a draft, send it, and the customer receives a hosted link where they can view it, accept it or decline it. Accepting can issue an [invoice](/docs/crm-invoices) automatically.

The flow end to end:

1. Create a quote (`draft`) with line items. Totals and tax are calculated by the server.
2. **Send** it. It is numbered, your billing details are frozen onto it, and the customer gets a link by email and/or WhatsApp. Status becomes `sent`.
3. The customer opens the link (`viewed`) and accepts or declines. You can also record the decision yourself.
4. On acceptance the quote becomes `accepted`, or `converted` when [`auto_invoice_on_accept`](/docs/crm-products#billing-profile) is on and an invoice is issued for you.
5. The invoice is paid, see [Invoices](/docs/crm-invoices).

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, get, PDF | `crm_quotes:read` |
| Create, edit, send, accept, decline, convert, revise, void, delete | `crm_quotes:write` |

Converting a quote creates an invoice, so [convert](#convert-to-an-invoice) needs **both** `crm_quotes:write` and `crm_invoices:write`. A key missing a required scope gets `403`.

Attaching a quote to a contact, company, deal or lead, or using a product, also requires that the record is in your tenant.

## Money and tax

All money fields end in `_minor` and are **integers in minor units** of the quote `currency` (paise for `INR`, cents for `USD`; `JPY` has none). `1180.50` rupees is `118050`. Rates are in **basis points**: `1800` is 18%. Totals are computed by the server and are authoritative; amounts you send for `subtotal`, `tax` or `total` are ignored.

Per line: `subtotal = quantity x unit_price_minor - discount`, `tax = subtotal x tax_rate_bps / 10000`, rounded half up to the minor unit.

## The quote object

```json
{
  "id": "a1b2…",
  "tenant_id": "a0b1…",
  "number": "QT/26-27/0001",
  "status": "sent",
  "title": "Voice agents for Acme",
  "contact_id": "4411…",
  "company_id": "5c6d…",
  "deal_id": "33cc…",
  "lead_id": null,
  "owner_user_id": "b1f2…",
  "currency": "INR",
  "issue_date": "2026-10-09",
  "valid_until": "2026-11-08",
  "buyer": {
    "name": "Acme Traders",
    "email": "accounts@acme.example",
    "phone": "+919800000000",
    "address": "44 Park Street",
    "city": "Mumbai",
    "state": "Maharashtra",
    "state_code": "27",
    "postal_code": "400001",
    "country": "IN",
    "tax_id": "27AAAAA0000A1Z5"
  },
  "seller": { "legal_name": "…", "tax_id": "…" },
  "tax_mode": "in_intra",
  "place_of_supply": "27",
  "subtotal_minor": 2500000,
  "discount_minor": 0,
  "tax_minor": 450000,
  "total_minor": 2950000,
  "tax_breakdown": [
    { "label": "CGST 9%", "rate_bps": 900, "taxable_minor": 2500000, "tax_minor": 225000 },
    { "label": "SGST 9%", "rate_bps": 900, "taxable_minor": 2500000, "tax_minor": 225000 }
  ],
  "lines": [
    {
      "id": "c3d4…",
      "position": 0,
      "product_id": "8b20…",
      "name": "Voice agent setup",
      "description": null,
      "hsn_sac": "998313",
      "unit": "unit",
      "quantity": 1,
      "unit_price_minor": 2500000,
      "discount_bps": 0,
      "tax_name": "GST 18%",
      "tax_rate_bps": 1800,
      "subtotal_minor": 2500000,
      "tax_minor": 450000,
      "total_minor": 2950000
    }
  ],
  "notes": null,
  "terms": "Payment due within 15 days.",
  "public_url": "https://…/quote/…",
  "sent_at": "2026-10-09T10:00:00Z",
  "viewed_at": null,
  "accepted_at": null,
  "declined_at": null,
  "accepted_by_name": null,
  "decline_reason": null,
  "converted_invoice_id": null,
  "created_at": "2026-10-09T09:50:00Z",
  "updated_at": "2026-10-09T10:00:00Z"
}
```

| Field | Notes |
| --- | --- |
| `number` | `null` until the quote is first sent. Format in [Numbering](/docs/crm-products#numbering) |
| `lines` | Returned on create, get, update and the action responses. List rows omit it |
| `buyer` | A snapshot of the customer's details at the time of the quote, so later edits to the contact do not change the document |
| `seller` | Your [billing profile](/docs/crm-products#billing-profile), frozen when the quote is sent. `null` on a draft |
| `tax_mode` | `in_intra` (CGST + SGST), `in_inter` (IGST), `standard` or `none`. See [Tax regimes](/docs/crm-products#tax-regimes) |
| `public_url` | The hosted link customers use. Treat it as a secret: anyone holding it can view the quote |
| `lines[].position` | Zero-based order of the line |
| `lines[].quantity` | Number, up to 3 decimal places |

### Status lifecycle

| Status | Meaning | Moves to |
| --- | --- | --- |
| `draft` | Editable. No number yet | `sent`, `void` |
| `sent` | Delivered, awaiting a decision | `viewed`, `accepted`, `declined`, `expired`, `void`, or `draft` via revise |
| `viewed` | Customer first opened the link | `accepted`, `declined`, `expired`, `void`, or `draft` via revise |
| `accepted` | Customer or you accepted | `converted`, `void` |
| `declined` | Customer or you declined | `draft` via revise, `void` |
| `expired` | Past `valid_until` without a decision. Set automatically | `draft` via revise, `void` |
| `converted` | An invoice was created from it | final |
| `void` | Cancelled | final |

Any move not listed returns `422`. Header fields and lines can be edited **only in `draft`**. Accepting also returns `422` once `valid_until` has passed.

## GET `/api/v1/crm/quotes`

Newest first. Each row carries the header fields, `buyer`, `total_minor`, `sent_at`, `valid_until`, `converted_invoice_id` and a computed `buyer_name`. It omits `lines`, `seller`, the tax fields and `public_url`.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `status` | `string` | One of the statuses above, otherwise `422` |
| `contact_id` | `UUID` | |
| `deal_id` | `UUID` | |
| `lead_id` | `UUID` | |
| `q` | `string` | Substring on number, title and buyer name. Up to 255 characters |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

## POST `/api/v1/crm/quotes`

Creates a `draft`. Currency, validity, notes and terms default from your billing profile.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `title` | `string` | No | Up to 255 characters |
| `contact_id` | `UUID` | No | Must be your contact |
| `company_id` | `UUID` | No | Must be your company |
| `deal_id` | `UUID` | No | Must be your deal |
| `lead_id` | `UUID` | No | Must be your lead |
| `owner_user_id` | `UUID` | No | Must be a user in your account |
| `currency` | `string` | No | 3-letter ISO 4217 code. Default: profile `default_currency` |
| `issue_date` | `date` | No | Default today |
| `valid_until` | `date` | No | Default `issue_date` plus `quote_validity_days`. Before `issue_date` returns `422` |
| `buyer` | `object` | No | Overrides the details taken from the contact or company. Keys: `name`, `email`, `phone`, `address`, `city`, `state`, `state_code`, `postal_code`, `country` (2 letters), `tax_id` |
| `notes`, `terms` | `string` | No | Up to 5000 characters. Default from your billing profile |
| `lines` | `object[]` | No | Up to 200. See below |

**Line fields**

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `product_id` | `UUID` | No | Must be your product. When `tax_rate_bps` is omitted the line takes the product's tax rate, and omitted `hsn_sac` and `unit` come from the product. `name` and `unit_price_minor` are still required |
| `name` | `string` | Yes | 1–255 characters, not blank |
| `description` | `string` | No | Up to 5000 characters |
| `hsn_sac` | `string` | No | Up to 16 characters |
| `unit` | `string` | No | Up to 32 characters |
| `quantity` | `number` | Yes | `> 0`, at most `1000000000`. Rounded half up to 3 decimals |
| `unit_price_minor` | `integer` | Yes | `0..10^15`, minor units |
| `discount_bps` | `integer` | No | `0..10000`, default `0` |
| `tax_name` | `string` | No | Up to 64 characters |
| `tax_rate_bps` | `integer` | No | `0..10000`, default `0` |

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/quotes \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Voice agents for Acme",
    "contact_id": "4411…",
    "deal_id": "33cc…",
    "lines": [
      { "product_id": "8b20…", "name": "Voice agent setup", "quantity": 1,
        "unit_price_minor": 2500000 },
      { "name": "Extra call minutes", "quantity": 500, "unit_price_minor": 600,
        "tax_name": "GST 18%", "tax_rate_bps": 1800 }
    ]
  }'
```

Returns `201` with the full quote, its `lines` and `public_url`.

## GET / PATCH / DELETE `/api/v1/crm/quotes/{quote_id}`

`GET` returns the quote with `lines`. `PATCH` accepts the create fields, all optional; when `lines` is present it **replaces** every line and totals are recalculated. An explicit `null` for `currency`, `issue_date` or `valid_until` is ignored. Changing `contact_id`, `company_id` or `buyer` rebuilds the buyer snapshot and recalculates tax. `DELETE` returns `204` with no body. `PATCH` and `DELETE` work only on a `draft`, otherwise `422`. A revised quote is a draft that keeps its number, so it cannot be deleted (`422`): void it instead, which keeps the number sequence complete.

## Sending

### POST `/api/v1/crm/quotes/{quote_id}/send`

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `channels` | `string[]` | Yes | One or two of `email`, `whatsapp` |
| `to_email` | `string` | No | Valid email, up to 320 characters. Defaults to the buyer email |
| `to_phone` | `string` | No | Up to 32 characters. Defaults to the buyer phone |
| `message` | `string` | No | Up to 1000 characters. Added to the message body |

Sending a `draft` first numbers it, freezes your billing details and moves it to `sent`; a draft with no lines, or whose `valid_until` is in the past, returns `422`. A quote that is already `sent`, `viewed`, `accepted` or `converted` is delivered again without a status change. A `declined`, `expired` or `void` quote returns `422` (revise it first). The status change is saved before delivery starts, so a failed channel does not undo it. The customer receives a hosted link where they can view, accept or decline the quote.

With an API key, `to_email` and `to_phone` may only repeat the buyer's own email and phone on the quote; any other recipient returns `403`. Leave them out to use the buyer's details, or change the buyer on the draft first. Signed-in console users can send to any recipient.

Each workspace can send up to 200 quotes and invoices per channel per day (UTC), counting every email or WhatsApp delivery attempted. Past that, the call returns `429` before anything is numbered or sent; try again the next day. Channels with no recipient (`skipped`) do not count.

The response reports each channel separately:

```json
{ "email": "sent", "whatsapp": "outside_window" }
```

A channel you did not request is `null`.

| Value | Meaning |
| --- | --- |
| `sent` | Handed off for delivery |
| `skipped` | No recipient address for the channel, or no WhatsApp number connected |
| `failed` | Delivery was attempted and failed |
| `outside_window` | WhatsApp only. The customer has not messaged you in the last 24 hours, so a free-form WhatsApp message cannot be sent. Use email or share the `public_url` yourself |

## Accepting

### POST `/api/v1/crm/quotes/{quote_id}/accept`

Records an acceptance on the customer's behalf, for example after a phone approval. Returns the quote.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, not blank. Stored as `accepted_by_name` |

Valid from `sent` or `viewed`, and only while `valid_until` has not passed in your workspace timezone. Emits the `quote.accepted` [webhook event](/docs/webhooks) once the request commits. When `auto_invoice_on_accept` is on in your billing profile, an invoice is created and issued from the quote (issue date today, due date from `payment_terms_days`), the quote becomes `converted` and `converted_invoice_id` is set. Otherwise the quote stays `accepted` and you can [convert it](#convert-to-an-invoice) when ready.

### POST `/api/v1/crm/quotes/{quote_id}/decline`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `reason` | `string` | No | Up to 500 characters |

Valid from `sent` or `viewed`. Returns the quote. Send `{}` when you have no reason.

## Convert to an invoice

### POST `/api/v1/crm/quotes/{quote_id}/convert`

Creates an invoice from an `accepted` quote, copying its lines, and moves the quote to `converted`. Requires both `crm_quotes:write` and `crm_invoices:write`. The invoice is dated today and due after `payment_terms_days` from your billing profile.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `issue` | `boolean` | No | `true` numbers and issues the invoice immediately. `false` leaves it a `draft`. Default `true` |

Send `{}` for the default. Returns `201` with the new [invoice](/docs/crm-invoices#the-invoice-object). Any status other than `accepted` returns `422`.

## Revise, void and PDF

- `POST /api/v1/crm/quotes/{quote_id}/revise` returns a `sent`, `viewed`, `declined` or `expired` quote to `draft` so it can be edited. It keeps its number and clears the viewed and declined timestamps and the decline reason. Returns the quote.
- `POST /api/v1/crm/quotes/{quote_id}/void` cancels a quote and takes no body. Allowed from `draft`, `sent`, `viewed`, `accepted`, `declined` and `expired`; a `converted` or `void` quote returns `422`. Returns the quote.
- `GET /api/v1/crm/quotes/{quote_id}/pdf` (scope `crm_quotes:read`) returns the quote as `application/pdf`, rendered on demand.

## Expiry

A quote can be accepted until the end of its `valid_until` date in your workspace timezone (the timezone in your workspace settings; India time when none is set and your billing country is India). From the next day, accepting returns `422` and the hosted page shows the quote as no longer acceptable. The stored status moves from `sent` or `viewed` to `expired` automatically once that date has passed in every timezone, so it can lag by up to a day.

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `crm_quotes:read` / `crm_quotes:write`, or `crm_invoices:write` for convert, or a key sends to a recipient other than the buyer |
| `404` | Quote, contact, company, deal, lead, owner or product not in your tenant |
| `422` | Invalid status transition, editing or deleting a non-draft quote, deleting a revised (numbered) draft, accepting an expired quote, a line or quote total above 10^15 minor units, sending a draft with no lines or a past `valid_until`, sending a `declined`, `expired` or `void` quote, a line missing `name`, `quantity` or `unit_price_minor`, or a value outside the bounds above |
| `429` | The workspace's daily send limit for that channel is used up |

Nothing on this page consumes credits. Email and WhatsApp deliveries follow the normal charges of those channels.
