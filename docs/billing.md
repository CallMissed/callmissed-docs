---
title: "Billing & Credits"
description: "Credit balance, packs, ledger, the monthly budget cap, and invoices: all dashboard-session endpoints."
slug: "billing"
breadcrumb: "API Reference"
---

# Billing & Credits

Credit balance, packs, ledger, the monthly budget cap, and invoices: all dashboard-session endpoints.

> Credits are the universal billing unit. **1 credit = ₹1.** Every API call (LLM, STT, TTS, search, image) deducts credits based on usage.

## Credential class: dashboard JWT only

Every endpoint on this page resolves a **dashboard access token**. A `cm_` API key is **rejected with `401`** - `credits`, `invoices`, and `audit` are not part of the API key surface, no matter which scopes the key carries.

```
Authorization: Bearer <jwt_access_token>
```

Get that token from [`POST /api/v1/auth/login`](/docs/auth-api). If you need programmatic access to your own spend from a server, run the login/refresh flow and hold the JWT; there is no key-based equivalent today.

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`; validation failures are `422` with FastAPI's array-of-errors shape.

Buying credits (`create-order`, `retry`, `verify`, `history`) lives on [Payments & Invoices](/docs/payments).

---

# Credits

## GET /api/v1/credits/balance

No parameters.

```bash
curl https://api.callmissed.com/api/v1/credits/balance \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "balance": 4820.5,
  "real_balance": 3820.5,
  "currency": "INR",
  "rate": "1 credit = ₹1"
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `balance` | `number` | Total spendable credits |
| `real_balance` | `number` | The **paid** portion only: total minus the unspent signup bonus. Phone-number rentals must come out of this |
| `currency` | `string` | Always `INR` |
| `rate` | `string` | Always `1 credit = ₹1` |

## GET /api/v1/credits/packs

Purchasable credit packs and the bounds on a custom amount. No parameters.

```bash
curl https://api.callmissed.com/api/v1/credits/packs \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "packs": [
    { "credits": 100,  "bonus": 0,   "total": 100,  "price": 99,   "label": "100",   "popular": false },
    { "credits": 500,  "bonus": 10,  "total": 510,  "price": 449,  "label": "510",   "popular": true  },
    { "credits": 1000, "bonus": 50,  "total": 1050, "price": 849,  "label": "1,050", "popular": false },
    { "credits": 5000, "bonus": 500, "total": 5500, "price": 3999, "label": "5,500", "popular": false }
  ],
  "costs": { "...": "per-service credit costs" },
  "custom_min": 10,
  "custom_max": 100000,
  "custom_rate": 1.0
}
```

`total` is `credits + bonus` and is what actually lands in the balance. A custom top-up (any integer from `custom_min` to `custom_max`) is charged at `custom_rate` (₹1 per credit) and earns **no bonus**. Pass the `credits` value to [`POST /api/v1/payments/create-order`](/docs/payments).

## GET /api/v1/credits/transactions

Cursor-paginated credit ledger for the tenant, newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `before` | `string` | No | ISO-8601 timestamp. Returns rows created strictly **before** it. Pass the previous response's `next_cursor` |
| `since_days` | `integer` | No | `1 <= since_days <= 365`. Inclusive lower bound on `created_at` |
| `type` | `string` | No | One of `signup_bonus`, `purchase`, `deduction`, `refund`, `plan_grant`, `admin`, `coupon`, `number_rental`, `call_charge` |

```bash
curl "https://api.callmissed.com/api/v1/credits/transactions?limit=50&since_days=30&type=deduction" \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "data": [
    {
      "id": "3c1f8e22-9a04-4d71-b6ee-71a2c9f00d15",
      "amount": -0.42,
      "balance_after": 4820.5,
      "type": "deduction",
      "description": "LLM call: glm-5.2",
      "reference_id": "req_9f2a1c4e6b8d0a13",
      "created_at": "2026-08-04T09:58:31.482119+00:00"
    }
  ],
  "next_cursor": "2026-08-04T09:58:31.482119+00:00",
  "summary_by_type": {
    "deduction": { "count": 1532, "total": -612.44 },
    "purchase":  { "count": 3,    "total": 5510.0 }
  }
}
```

`next_cursor` is `null` on the last page. `summary_by_type` aggregates the same filter window ignoring the `before` cursor, so the totals stay stable while you page.

| Status | Cause |
| --- | --- |
| `400` | `before` is not parseable ISO-8601, or `type` is not one of the allowed values |
| `422` | `limit` or `since_days` outside its bounds |

## GET /api/v1/credits/budget

Current monthly budget cap and period usage. No parameters.

```bash
curl https://api.callmissed.com/api/v1/credits/budget \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "monthly_budget_credits": 5000.0,
  "period_start": "2026-08",
  "period_used_credits": 612.44,
  "remaining_credits": 4387.56,
  "alerts_enabled": true
}
```

`monthly_budget_credits` and `remaining_credits` are `null` when no cap is set. The used counter resets when the period rolls over.

## PUT /api/v1/credits/budget

Sets or clears the monthly cap. **Owner or admin only.**

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `monthly_budget_credits` | `number \| null` | No | `0 <= value <= 10000000`. Send `null` to remove the cap. Default `null` |
| `alerts_enabled` | `boolean` | No | Default `true` |

```bash
curl -X PUT https://api.callmissed.com/api/v1/credits/budget \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"monthly_budget_credits": 5000, "alerts_enabled": true}'
```

Returns the same object as `GET /budget`. Writes a `budget.update` audit event.

| Status | Cause |
| --- | --- |
| `403` | `Only owners/admins can change the budget` |
| `404` | Tenant not found |
| `422` | `monthly_budget_credits` outside `0 .. 10000000` |

Budget events fire the `budget.alert` and `budget.exceeded` [webhooks](/docs/webhooks).

---

# Invoices

An invoice row is created when a payment settles. Every route is scoped to your tenant, so another tenant's invoice number returns `404`.

## GET /api/v1/invoices

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `limit` | `integer` | No | `1 <= limit <= 100`, default `20` |
| `offset` | `integer` | No | `0 <= offset <= 10000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/invoices?limit=20&offset=0" \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
[
  {
    "invoice_number": "CM-2026-000482",
    "amount": 3999.0,
    "currency": "INR",
    "description": "Credit top-up (5500 credits)",
    "payment_method": "upi",
    "order_id": "cm_9f2a1c4e6b8d0a13",
    "plan": null,
    "credits": 5500,
    "pdf_url": "https://.../CM-2026-000482.pdf",
    "issued_at": "2026-07-28T06:14:02+00:00"
  }
]
```

Sorted by `issued_at` descending. `pdf_url` may be `null` if the stored copy is unavailable; use the PDF endpoint below, which always renders.

## GET `/api/v1/invoices/{invoice_number}`

Full detail for one invoice, including the gateway payment id.

```bash
curl https://api.callmissed.com/api/v1/invoices/CM-2026-000482 \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
{
  "invoice_number": "CM-2026-000482",
  "amount": 3999.0,
  "currency": "INR",
  "description": "Credit top-up (5500 credits)",
  "payment_method": "upi",
  "cf_payment_id": "5114909345",
  "order_id": "cm_9f2a1c4e6b8d0a13",
  "plan": null,
  "credits": 5500,
  "pdf_url": "https://.../CM-2026-000482.pdf",
  "issued_at": "2026-07-28T06:14:02+00:00",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "user_id": "u1234567-89ab-cdef-0123-456789abcdef"
}
```

`404` with `Invoice not found` when the number does not exist in your tenant.

## GET `/api/v1/invoices/{invoice_number}/pdf`

Streams the invoice as a PDF, regenerated on demand. Works even when `pdf_url` is `null`.

Response is `Content-Type: application/pdf` with `Content-Disposition: attachment; filename="<invoice_number>.pdf"`.

```bash
curl -L https://api.callmissed.com/api/v1/invoices/CM-2026-000482/pdf \
  -H "Authorization: Bearer <jwt_access_token>" \
  -o CM-2026-000482.pdf
```

| Status | Cause |
| --- | --- |
| `404` | `Invoice not found` |
| `500` | `Unable to render invoice PDF` |

## POST /api/v1/invoices/generate

Creates an invoice for a settled payment that does not have one yet. Idempotent: if an invoice already exists for the order, the existing one is returned unchanged.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `order_id` | `string` | Yes | 1-100 chars. The `order_id` from `POST /api/v1/payments/create-order` |

```bash
curl -X POST https://api.callmissed.com/api/v1/invoices/generate \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"order_id": "cm_9f2a1c4e6b8d0a13"}'
```

```json
{
  "invoice_number": "CM-2026-000482",
  "amount": 3999.0,
  "currency": "INR",
  "description": "Credit top-up (5500 credits)",
  "pdf_url": "https://.../CM-2026-000482.pdf",
  "issued_at": "2026-07-28T06:14:02+00:00",
  "order_id": "cm_9f2a1c4e6b8d0a13"
}
```

The `description` is derived from the order: `Credit top-up (N credits)`, `Coupon credit grant (...)`, or `Plan upgrade to <Plan>`.

| Status | Cause |
| --- | --- |
| `400` | `Invoice can only be generated for paid transactions` |
| `404` | `Payment not found` in your tenant |
| `422` | `order_id` empty or over 100 chars |

An `invoice.created` [webhook](/docs/webhooks) fires when an invoice is issued.
