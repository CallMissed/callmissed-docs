---
title: "Products, Tax Rates & Billing Profile"
description: "Your catalogue of billable items, the named tax rates applied to them, and the billing profile that drives numbering and tax mode on quotes and invoices."
slug: "crm-products"
breadcrumb: "API Reference"
---

# Products, Tax Rates & Billing Profile

Your catalogue of billable items, the named tax rates applied to them, and the billing profile that drives numbering and tax mode on quotes and invoices.

## Overview

Three small resources sit behind [quotes](/docs/crm-quotes) and [invoices](/docs/crm-invoices):

- **Products** are reusable line items with a price, unit and optional tax rate.
- **Tax rates** are named percentages you define, such as "GST 18%" or "VAT 20%".
- The **billing profile** is your seller identity (legal name, address, tax ID) plus numbering and default terms. There is exactly one per account.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| Read products and tax rates | `crm_products:read` |
| Create, update, delete products and tax rates | `crm_products:write` |
| Read the billing profile | `crm_billing:read` |
| Replace the billing profile | `crm_billing:write` |

One scope pair covers both products and tax rates. Dashboard logins are not scope-checked, except that creating, updating or deleting tax rates and replacing the billing profile require an owner or admin role.

## Money and rates

- **Money is always an integer in minor units** of the document's currency, never a decimal. INR `1180.50` is `118050`. Currencies with no minor unit (for example `JPY`) use exponent 0; a few (for example `KWD`) use exponent 3; everything else uses 2.
- **Tax rates are in basis points** (1 bps = 0.01%). `1800` is 18%. Allowed range `0..10000`.

---

# Products

```json
{
  "id": "8b20…",
  "tenant_id": "a0b1…",
  "name": "Voice agent setup",
  "description": "One-time onboarding",
  "sku": "VA-SETUP",
  "unit": "unit",
  "unit_price_minor": 2500000,
  "currency": "INR",
  "tax_rate_id": "9c30…",
  "hsn_sac": "998313",
  "archived": false,
  "created_at": "2026-10-07T09:00:00Z",
  "updated_at": "2026-10-07T09:00:00Z"
}
```

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/api/v1/crm/products` | `q` (up to 255 characters; case-insensitive name or SKU substring), results ordered by name, `include_archived` (default `false`), `limit` `1..200` (default `50`), `offset` `0..100000` |
| `POST` | `/api/v1/crm/products` | `201` |
| `GET` | `/api/v1/crm/products/{product_id}` | |
| `PATCH` | `/api/v1/crm/products/{product_id}` | Any create field plus `archived`, all optional. A null `unit`, `unit_price_minor`, `currency` or `archived` is ignored; a blank `name` is `422` |
| `DELETE` | `/api/v1/crm/products/{product_id}` | `204`. Archives the product if any quote or invoice line references it; otherwise deletes it |

### POST `/api/v1/crm/products`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, not blank |
| `description` | `string` | No | Up to 5000 characters |
| `sku` | `string` | No | Up to 64 characters, unique per tenant when set (`409` otherwise) |
| `unit` | `string` | No | 1–32 characters, default `unit` |
| `unit_price_minor` | `integer` | No | `0` to `10^15`, default `0` |
| `currency` | `string` | No | Exactly 3 letters (upper-cased). Defaults to your billing profile's `default_currency` |
| `tax_rate_id` | `UUID` | No | Must be your tax rate (`404` otherwise) |
| `hsn_sac` | `string` | No | Up to 16 characters. HSN (goods) or SAC (services) code, used on Indian GST documents |

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/products \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Voice agent setup",
    "sku": "VA-SETUP",
    "unit_price_minor": 2500000,
    "currency": "INR",
    "tax_rate_id": "9c30…",
    "hsn_sac": "998313"
  }'
```

Create returns `201` with the product object. Archived products stay on existing documents but are hidden from the default list.

---

# Tax rates

```json
{
  "id": "9c30…",
  "tenant_id": "a0b1…",
  "name": "GST 18%",
  "rate_bps": 1800,
  "is_default": true,
  "archived": false,
  "created_at": "2026-10-07T09:00:00Z",
  "updated_at": "2026-10-07T09:00:00Z"
}
```

| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/api/v1/crm/tax-rates` | `include_archived` (default `false`), `limit` `1..200` (default `50`), `offset` `0..100000`. Ordered by `rate_bps`, then name |
| `POST` | `/api/v1/crm/tax-rates` | `201`. `name` 1–64 characters, unique per tenant (`409`). `rate_bps` `0..10000`, required. `is_default` default `false` |
| `PATCH` | `/api/v1/crm/tax-rates/{rate_id}` | `name`, `rate_bps`, `is_default`, `archived`, all optional. A null `rate_bps`, `is_default` or `archived` is ignored |
| `DELETE` | `/api/v1/crm/tax-rates/{rate_id}` | `204`. Archives the rate (and clears its default flag) rather than removing it, so existing documents keep their history |

Only one rate is the default: setting `is_default` on one clears it on the others, and archiving a rate clears its default flag. Writes need an owner or admin role for dashboard logins. Changing a rate does not rewrite existing quotes or invoices. Each line stores the tax name and rate it was created with.

---

# Billing profile

One profile per account. `GET` creates the profile with defaults on first read if you have not saved one yet.

```json
{
  "id": "1d40…",
  "tenant_id": "a0b1…",
  "legal_name": "Acme Traders Pvt Ltd",
  "address_line1": "12 MG Road",
  "address_line2": null,
  "city": "Pune",
  "state": "Maharashtra",
  "state_code": "27",
  "postal_code": "411001",
  "country": "IN",
  "tax_id": "27ABCDE1234F1Z5",
  "tax_regime": "in_gst",
  "default_currency": "INR",
  "invoice_prefix": "INV",
  "quote_prefix": "QT",
  "numbering_period": "fy_april",
  "payment_terms_days": 15,
  "quote_validity_days": 30,
  "default_notes": null,
  "default_terms": "Payment due within 15 days.",
  "bank_details": "Account name, number, IFSC",
  "reply_to_email": "billing@acme.example",
  "auto_invoice_on_accept": true,
  "created_at": "2026-10-07T09:00:00Z",
  "updated_at": "2026-10-07T09:00:00Z"
}
```

| Method | Path |
| --- | --- |
| `GET` | `/api/v1/crm/billing-profile` |
| `PUT` | `/api/v1/crm/billing-profile` |

`PUT` replaces the whole profile: every field is optional, but an omitted field is reset to its default (`null` for text fields). With an API key it needs `crm_billing:write`; with a dashboard login only an owner or admin may change it (`403` otherwise). Returns the saved profile.

| Field | Type | Constraints |
| --- | --- | --- |
| `legal_name` | `string` | Up to 255 characters |
| `address_line1`, `address_line2` | `string` | Up to 255 characters each |
| `city`, `state` | `string` | Up to 120 characters each |
| `state_code` | `string` | Up to 4 characters. For India, the 2-digit GST state code |
| `postal_code` | `string` | Up to 20 characters |
| `country` | `string` | Exactly 2 letters (upper-cased), ISO 3166, default `IN` |
| `tax_id` | `string` | Up to 32 characters. GSTIN, VAT number or equivalent |
| `tax_regime` | `string` | `in_gst`, `standard` or `none`. Default `in_gst`. See below |
| `default_currency` | `string` | Exactly 3 letters (upper-cased), ISO 4217, default `INR` |
| `invoice_prefix`, `quote_prefix` | `string` | 1–5 characters: letters, digits and `-` only. Defaults `INV` and `QT` |
| `numbering_period` | `string` | `fy_april`, `calendar` or `none`. Default `fy_april` |
| `payment_terms_days` | `integer` | `0..365`, default `15`. Sets the invoice due date |
| `quote_validity_days` | `integer` | `1..365`, default `30`. Sets the quote `valid_until` |
| `default_notes`, `default_terms` | `string` | Up to 5000 characters each. Copied onto new documents |
| `bank_details` | `string` | Up to 2000 characters. Copied onto new documents |
| `reply_to_email` | `string` | Up to 320 characters, must be a valid-looking email address. Replies to documents you send go here |
| `auto_invoice_on_accept` | `boolean` | Default `true`. See [quotes](/docs/crm-quotes#accepting) |

## Tax regimes

| `tax_regime` | Effect |
| --- | --- |
| `in_gst` | Indian GST. If the buyer's state code matches yours the tax is split into **CGST** and **SGST** (half each); otherwise it is a single **IGST** line. A buyer outside India is treated as inter-state |
| `standard` | One named tax row per rate, such as "VAT 20%". No country rules are applied |
| `none` | All line tax is forced to `0` |

The mode is resolved for you from the seller and buyer details and saved on each document as `tax_mode` (`in_intra`, `in_inter`, `standard` or `none`) with a `place_of_supply`. CallMissed does not look up rates for you: tax is calculated from the rates you configure. For tax treatment, such as zero-rating exports, consult your tax advisor.

## Numbering

Document numbers look like `INV/26-27/0001`: prefix, period, then a gapless counter that restarts each period.

| `numbering_period` | Period segment |
| --- | --- |
| `fy_april` | Financial year starting 1 April, such as `26-27` |
| `calendar` | Calendar year, such as `2026` |
| `none` | No period segment, such as `INV/0001` |

Quotes are numbered when first sent. Invoices are numbered when issued, so drafts never consume a number.

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing the scope, or a dashboard user who is not an owner or admin creates, updates or deletes tax rates or replaces the billing profile |
| `404` | Product, tax rate or referenced tax rate not in your tenant |
| `409` | Duplicate SKU or tax rate name |
| `422` | Blank name, `rate_bps` outside `0..10000`, unknown `tax_regime` or `numbering_period`, an invalid `country`, currency, prefix or `reply_to_email`, or a value outside the bounds above |

Nothing on this page consumes credits.
