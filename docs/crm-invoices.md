---
title: "Invoices & Payments"
description: "Issue numbered invoices with tax, send them to customers, record payments, collect online through your connected payment gateway, and track what is overdue."
slug: "crm-invoices"
breadcrumb: "API Reference"
---

# Invoices & Payments

Issue numbered invoices with tax, send them to customers, record payments, collect online through your connected payment gateway, and track what is overdue.

## Overview

An **invoice** is a numbered request for payment. You can create one directly, or have it created from an accepted [quote](/docs/crm-quotes). Customers receive a hosted link where they can view the invoice, download a PDF and, where available, pay online.

The quote to payment flow:

1. A [quote](/docs/crm-quotes) is accepted, which creates an invoice (or you create an invoice yourself as a `draft`).
2. **Issue** the invoice. It gets its number, your billing details are frozen onto it and status becomes `issued`.
3. **Send** it by email and/or WhatsApp. The customer receives a hosted link to view and pay.
4. Payments are recorded against it, either manually by you or automatically when the customer pays online. Status moves to `partially_paid` and then `paid`.
5. If the due date passes with money outstanding, the invoice becomes `overdue` automatically.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, get, summary, payments list, PDF | `crm_invoices:read` |
| Create, edit, issue, send, record payment, create payment link, void, delete | `crm_invoices:write` |

A key missing the scope gets `403`.

## Money and tax

All money fields end in `_minor` and are **integers in minor units** of the invoice `currency` (paise for `INR`, cents for `USD`; `JPY` has none). `1180.50` is `118050`. Rates are in **basis points**: `1800` is 18%. Totals are computed by the server and are authoritative. Tax modes (CGST + SGST, IGST, standard or none) work as described in [Tax regimes](/docs/crm-products#tax-regimes). Tax is calculated only from the rates you configure; consult your tax advisor for how it applies to you.

## The invoice object

```json
{
  "id": "d5e6…",
  "tenant_id": "a0b1…",
  "number": "INV/26-27/0001",
  "status": "partially_paid",
  "title": "Voice agents for Acme",
  "contact_id": "4411…",
  "company_id": "5c6d…",
  "deal_id": "33cc…",
  "quote_id": "a1b2…",
  "owner_user_id": "b1f2…",
  "currency": "INR",
  "issue_date": "2026-10-09",
  "due_date": "2026-10-24",
  "buyer": { "name": "Acme Traders", "state_code": "27", "country": "IN", "tax_id": "27AAAAA0000A1Z5" },
  "seller": { "legal_name": "…", "tax_id": "…" },
  "tax_mode": "in_intra",
  "place_of_supply": "27",
  "subtotal_minor": 2500000,
  "discount_minor": 0,
  "tax_minor": 450000,
  "total_minor": 2950000,
  "amount_paid_minor": 1000000,
  "amount_due_minor": 1950000,
  "tax_breakdown": [
    { "label": "CGST 9%", "rate_bps": 900, "taxable_minor": 2500000, "tax_minor": 225000 },
    { "label": "SGST 9%", "rate_bps": 900, "taxable_minor": 2500000, "tax_minor": 225000 }
  ],
  "lines": [],
  "notes": null,
  "terms": "Payment due within 15 days.",
  "public_url": "https://…/invoice/…",
  "payment_request_id": null,
  "sent_at": "2026-10-09T10:00:00Z",
  "viewed_at": null,
  "issued_at": "2026-10-09T10:00:00Z",
  "paid_at": null,
  "voided_at": null,
  "void_reason": null,
  "created_at": "2026-10-09T09:50:00Z",
  "updated_at": "2026-10-10T08:00:00Z"
}
```

`amount_due_minor` is the amount still owed: `total_minor - amount_paid_minor`, or `0` on a `draft` or `void` invoice. `lines` is returned on create, get, update and the action responses, and omitted from list rows. `buyer`, `seller`, `tax_*`, `lines` and the totals follow the same shapes and rules as the [quote object](/docs/crm-quotes#the-quote-object). `public_url` is the hosted link for the customer; treat it as a secret.

`number` is `null` on a draft. Numbers are assigned when the invoice is **issued**, so numbering has no gaps, and a voided invoice keeps its number.

### Status lifecycle

| Status | Meaning | Moves to |
| --- | --- | --- |
| `draft` | Editable, no number | `issued`, `void` |
| `issued` | Numbered and sent or ready to send | `partially_paid`, `paid`, `overdue`, `void` |
| `partially_paid` | Some money received | `paid`, `overdue` |
| `overdue` | Past `due_date` with a balance. Set automatically, or when a partial payment lands after the due date | `paid`, `void` |
| `paid` | Fully paid | final |
| `void` | Cancelled. Only allowed with no payments recorded | final |

Any move not listed returns `422`. Header fields and lines can be edited **only in `draft`**.

## GET `/api/v1/crm/invoices`

Newest first. Each row carries the header fields, `buyer`, `total_minor`, `due_date`, `amount_paid_minor`, `quote_id`, `sent_at`, a computed `amount_due_minor` and a computed `buyer_name`. It omits `lines`, `seller`, the tax fields and `public_url`.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `status` | `string` | One of the statuses above, otherwise `422` |
| `contact_id` | `UUID` | |
| `deal_id` | `UUID` | |
| `q` | `string` | Substring on number, title and buyer name. Up to 255 characters |
| `overdue` | `boolean` | Default `false`. `true` returns only `overdue` invoices plus `issued` or `partially_paid` ones already past their due date |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

## GET `/api/v1/crm/invoices/summary`

Outstanding, overdue and recently paid totals, one row per currency because amounts in different currencies are never added together. `outstanding_minor` is the balance on `issued`, `partially_paid` and `overdue` invoices, `overdue_minor` is the overdue part of that, and `paid_30d_minor` is the payments recorded in the last 30 days. Rows are sorted by currency.

```json
{
  "currencies": [
    {
      "currency": "INR",
      "outstanding_minor": 1950000,
      "overdue_minor": 450000,
      "paid_30d_minor": 5900000
    }
  ]
}
```

## POST `/api/v1/crm/invoices`

Creates a `draft`. The body and line fields are the same as [creating a quote](/docs/crm-quotes#post-apiv1crmquotes), with these differences: there is no `lead_id` or `valid_until`; `due_date` (default `issue_date` plus `payment_terms_days`) and `quote_id` (optional, must be your quote or `404`) are accepted. `due_date` before `issue_date` returns `422`. Every line needs `name`, `quantity` and `unit_price_minor`.

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/invoices \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "contact_id": "4411…",
    "deal_id": "33cc…",
    "lines": [
      { "name": "Voice agent setup", "quantity": 1, "unit_price_minor": 2500000,
        "tax_name": "GST 18%", "tax_rate_bps": 1800 }
    ]
  }'
```

Returns `201` with the full invoice, its `lines` and `public_url`.

## GET / PATCH / DELETE `/api/v1/crm/invoices/{invoice_id}`

`PATCH` accepts the create fields, all optional (except `quote_id`); `lines`, when present, replaces every line and recalculates totals. An explicit `null` for `currency`, `issue_date` or `due_date` is ignored. `PATCH` and `DELETE` (`204`, no body) work only on a `draft`, otherwise `422`.

## Issue and send

### POST `/api/v1/crm/invoices/{invoice_id}/issue`

`draft` to `issued`. Assigns the number, freezes your billing details and stamps `issued_at`. Returns the invoice. Any other status, or a draft with no lines, returns `422`.

### POST `/api/v1/crm/invoices/{invoice_id}/send`

Same body and per-channel response as [sending a quote](/docs/crm-quotes#sending) (`channels`, `to_email`, `to_phone`, `message`). Sending a `draft` issues it first (`422` if it has no lines). An `issued`, `partially_paid`, `overdue` or `paid` invoice is delivered again without a status change; a `void` invoice returns `422`. The customer receives a hosted link where they can view the invoice, download the PDF and pay online when it is available. The same recipient rule for API keys (`403`) and daily send limit (`429`) apply as for [quotes](/docs/crm-quotes#sending).

## Payments

```json
{
  "id": "e7f8…",
  "invoice_id": "d5e6…",
  "amount_minor": 1000000,
  "method": "bank_transfer",
  "reference": "UTR 4821",
  "paid_at": "2026-10-10T08:00:00Z",
  "payment_request_id": null,
  "recorded_by_user_id": null,
  "note": null,
  "created_at": "2026-10-10T08:01:00Z"
}
```

### POST `/api/v1/crm/invoices/{invoice_id}/payments`

Records a payment received outside the platform.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `amount_minor` | `integer` | Yes | `> 0`, at most `10^15`, minor units. Cannot exceed the balance |
| `method` | `string` | No | `cash`, `bank_transfer`, `upi`, `card`, `cheque` or `other`. Default `bank_transfer`. `online` is not accepted: online payments are recorded for you |
| `paid_at` | `datetime` | No | Default now |
| `reference` | `string` | No | Up to 128 characters |
| `note` | `string` | No | Up to 500 characters |

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/invoices/d5e6…/payments \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "amount_minor": 1000000, "method": "bank_transfer", "reference": "UTR 4821" }'
```

Returns `201` with the payment. The invoice moves to `partially_paid` (or stays or becomes `overdue` when the due date has passed and a balance remains), or to `paid` once the balance reaches zero (and `paid_at` is stamped; the linked contact's lifecycle stage becomes `customer`). A payment larger than the balance, or on a `draft`, `paid` or `void` invoice, returns `422`.

When a payment settles the invoice, the `invoice.paid` [webhook event](/docs/webhooks) is emitted once the request commits.

`GET /api/v1/crm/invoices/{invoice_id}/payments` (scope `crm_invoices:read`) lists the payments, oldest `paid_at` first.

### Online payment

### POST `/api/v1/crm/invoices/{invoice_id}/payment-link`

Creates, or reuses when one is still live for the same balance, a payment link for the amount due through **your connected payment gateway**. When the customer pays, the payment is recorded on the invoice for you (method `online`), the status updates, and the `invoice.paid` [webhook event](/docs/webhooks) is emitted once the invoice is settled. Takes no body and returns `200`. While a link for the invoice is still being created, a second call returns `409`; retry after a few seconds.

The gateway does not message the customer itself: share `url`, or [send the invoice](#issue-and-send), whose hosted page has a Pay button. Recording a payment or voiding the invoice cancels any unpaid link for it at the gateway, so an old link cannot collect a stale amount. If money still arrives that cannot be recorded (the invoice was settled or voided meanwhile), it is written as a [note](/docs/crm-notes-tasks) on the invoice so you can refund or reconcile it.

Online payment is available for invoices in `INR` only, in `issued`, `partially_paid` or `overdue` status with a balance due, and only when you have connected your own Razorpay account. The invoice buyer needs an email or phone number. The amount due is capped at INR 5,00,000 per link; above it the call returns `422` and you should collect by bank transfer instead. For any other currency, share your bank details (they are printed on the invoice) and record the payment manually.

```json
{ "url": "https://…" }
```

## Void and PDF

- `POST /api/v1/crm/invoices/{invoice_id}/void` cancels an invoice. `reason` is **required** (1–500 characters, not blank). Allowed from `draft`, `issued` or `overdue`, and only with **no payments** recorded, otherwise `422`. Returns the invoice.
- `GET /api/v1/crm/invoices/{invoice_id}/pdf` (scope `crm_invoices:read`) returns the invoice as `application/pdf`, rendered on demand. Documents with `tax_mode` `in_intra` or `in_inter` are titled "Tax Invoice".

## Overdue

An invoice is past due from the day after its `due_date` in your workspace timezone: the `overdue=true` list filter, the summary and a partial payment use that date. The stored status of `issued` or `partially_paid` invoices moves to `overdue` automatically once the due date has passed in every timezone, so it can lag by up to a day. Recording the remaining payment moves an overdue invoice to `paid`; a partial payment on an overdue invoice leaves it `overdue`.

---

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `crm_invoices:read` / `crm_invoices:write`, or a key sends to a recipient other than the buyer |
| `404` | Invoice, quote, contact, company, deal, owner or product not in your tenant |
| `409` | A payment link for the invoice is still being created |
| `422` | Invalid status transition, editing or deleting a non-draft invoice, a line or invoice total above 10^15 minor units, issuing or sending a draft with no lines, sending a `void` invoice, overpayment, payment on a `draft`, `paid` or `void` invoice, voiding an invoice that has payments, a payment link on a non-payable or non-`INR` invoice, with no gateway connected, with no buyer email or phone, or above the ceiling, or a value outside the bounds above |
| `429` | The workspace's daily send limit for that channel is used up |

Nothing on this page consumes credits. Email and WhatsApp deliveries follow the normal charges of those channels.
