---
title: "Payments & Invoices"
description: "Top up credits or upgrade your plan through the checkout gateway, retry a failed order, confirm settlement, and read your payment history."
slug: "payments"
breadcrumb: "API Reference"
---

# Payments & Invoices

Top up credits or upgrade your plan through the checkout gateway, retry a failed order, confirm settlement, and read your payment history.

## Overview

Use `create-order` to start a plan upgrade or a credit top-up, then complete checkout with the gateway SDK using the returned `payment_session_id`. The signed gateway **webhook is the source of truth** for settlement: never grant anything on the browser redirect alone. Successful orders generate a downloadable invoice. **1 credit = ₹1.**

Invoice reading and generation live on [Billing & Credits](/docs/billing).

## Credential class: dashboard JWT only

Every endpoint here resolves a **dashboard access token**. A `cm_` API key is **rejected with `401`**; committing the organization to spend is not part of the API key surface.

```
Authorization: Bearer <jwt_access_token>
```

| Endpoint | Role required |
| --- | --- |
| `POST /api/v1/payments/create-order` | **Owner or admin** |
| `POST /api/v1/payments/retry` | **Owner or admin** |
| `POST /api/v1/payments/verify` | Any member |
| `GET /api/v1/payments/history` | Any member |

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`; validation failures are `422`.

The gateway's own notification callback (`POST /api/v1/payments/webhook`) is not a customer endpoint: it is called by the payment provider with a signed body and is not documented for client use.

## Rate limits

Per client IP, in a 60-second window. Exceeding either returns `429` with a `Retry-After` header.

| Path | Limit / 60s |
| --- | --- |
| `POST /api/v1/payments/create-order` | 10 |
| `POST /api/v1/payments/verify` | 10 |

## Plans and upgrade paths

Prices are monthly, in INR. Downgrades are not self-serve.

| Plan | Price | Can upgrade to |
| --- | --- | --- |
| `free` | 0 | `starter`, `pro`, `enterprise` |
| `starter` | 999 | `pro`, `enterprise` |
| `pro` | 4999 | `enterprise` |
| `enterprise` | 20000 | - |

## Payment statuses

`pending`, `paid`, `expired`, `failed`, `refunded`. A gateway session expires roughly 15 minutes after creation.

---

## POST /api/v1/payments/create-order

Creates an order and returns a checkout session. Owner or admin.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `type` | `string` | No | `plan_upgrade` or `credit_topup`. Default `plan_upgrade` |
| `plan` | `string` | Required when `type` is `plan_upgrade` | `starter`, `pro`, `enterprise`, or `""`. Must be a legal upgrade from your current plan |
| `credits` | `integer` | Required when `type` is `credit_topup` | `0 <= credits <= 100000`, default `0`. Minimum **10** for a top-up |
| `client` | `string` | No | `web` or `console`. Default `web`. Chooses which app the gateway redirects back to; the origins are server-owned |

A `credits` value that matches a pack from [`GET /api/v1/credits/packs`](/docs/billing) is charged at the pack price and credits `total` (base plus bonus). Any other value from 10 upward is charged at ₹1 per credit with **no bonus**.

```bash
curl -X POST https://api.callmissed.com/api/v1/payments/create-order \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"type": "credit_topup", "credits": 5000, "client": "web"}'
```

```json
{
  "order_id": "cm_9f2a1c4e6b8d0a13",
  "cf_order_id": "1877284489",
  "payment_session_id": "session_a1B2c3D4e5F6...",
  "amount": 3999.0,
  "currency": "INR",
  "env": "production"
}
```

For a plan upgrade send `{"type": "plan_upgrade", "plan": "pro"}` instead.

Pass `payment_session_id` to the checkout SDK. Assert that your SDK's mode matches the returned `env` before you open checkout: a sandbox SDK given a production session fails with an opaque `payment_session_id_invalid`.

| Status | `detail` |
| --- | --- |
| `400` | `plan is required for plan_upgrade` |
| `400` | `Cannot upgrade from <current> to <target>` |
| `400` | `Minimum top-up is 10 credits` |
| `400` | `Invalid order type` or `Invalid amount` |
| `403` | `Requires role: admin, owner` |
| `429` | More than 10 requests/min from this IP |
| `502` | `Payment gateway error: ...` |
| `503` | `Payment gateway not configured` |

## POST /api/v1/payments/retry

Re-initiates payment for an order that is no longer usable (failed, expired, or auto-cancelled). Owner or admin. The retry preserves the original order's plan, credits, and amount, so you are billed for exactly what you tried to buy.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `order_id` | `string` | Yes | 1-100 chars. The original `order_id` |
| `client` | `string` | No | `web` or `console`. Default `web` |

```bash
curl -X POST https://api.callmissed.com/api/v1/payments/retry \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"order_id": "cm_9f2a1c4e6b8d0a13", "client": "web"}'
```

Returns the same shape as `create-order`. Two behaviours worth knowing:

- **Still-live session.** If the original order is younger than about 14 minutes and the gateway still reports it active, the **existing** `order_id` and `payment_session_id` are returned rather than a new order. This makes a double-clicked Retry safe.
- **Settled in the meantime.** If the gateway reports the original order as paid, the payment is completed (credits or plan granted) and the call returns `400 This payment just completed — refresh the page to see it.` That is a success, not a failure: re-read the balance.

| Status | `detail` |
| --- | --- |
| `400` | `This payment has already been completed` |
| `400` | `This payment was refunded — start a fresh order` |
| `400` | `This payment just completed — refresh the page to see it.` |
| `403` | `Requires role: admin, owner` |
| `404` | `Payment not found` in your tenant |
| `409` | `Payment amount mismatch — contact support` |
| `422` | `order_id` empty or over 100 chars |
| `502` | `Payment gateway error: ...` |
| `503` | `Payment gateway not configured` |

## POST /api/v1/payments/verify

Polls the gateway for an order's outcome and, on success, grants the plan or credits. Call this after the browser returns from checkout. Any member of the tenant may call it.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `order_id` | `string` | Yes | 1-100 chars |

```bash
curl -X POST https://api.callmissed.com/api/v1/payments/verify \
  -H "Authorization: Bearer <jwt_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"order_id": "cm_9f2a1c4e6b8d0a13"}'
```

```json
{ "status": "paid", "plan": "topup_5500" }
```

| `status` | Meaning |
| --- | --- |
| `paid` | Settled. `plan` echoes what was bought: a plan name, or `topup_<total_credits>` |
| `pending` | Not settled yet. Poll again, or wait for the webhook |
| `expired` | The session expired or was terminated. Use `retry` |
| `refunded` | Already reversed |

The call is idempotent and takes a row lock, so it is safe to race against the gateway webhook. The order amount the gateway reports must match the recorded amount within 1 paisa or nothing is granted.

| Status | `detail` |
| --- | --- |
| `400` | `Invalid JSON body` or `order_id is required` |
| `404` | `Payment not found` in your tenant |
| `409` | `Payment amount mismatch — contact support` |
| `429` | More than 10 requests/min from this IP |
| `502` | `Unable to verify payment status` |

## GET /api/v1/payments/history

The tenant's **50 most recent** payments, newest first. No parameters and no pagination.

```bash
curl https://api.callmissed.com/api/v1/payments/history \
  -H "Authorization: Bearer <jwt_access_token>"
```

```json
[
  {
    "order_id": "cm_9f2a1c4e6b8d0a13",
    "amount": 3999.0,
    "currency": "INR",
    "status": "paid",
    "plan_from": "pro",
    "plan_to": "topup_5500",
    "payment_method": "upi",
    "created_at": "2026-07-28T06:12:44+00:00"
  }
]
```

| Field | Type | Notes |
| --- | --- | --- |
| `order_id` | `string` | Pass to `verify`, `retry`, or `POST /api/v1/invoices/generate` |
| `status` | `string` | `pending`, `paid`, `expired`, `failed`, or `refunded` |
| `plan_from` | `string` | The plan at the time the order was created |
| `plan_to` | `string` | A plan name, `topup_<total_credits>`, or `coupon_<code>` |
| `payment_method` | `string \| null` | Reported by the gateway after settlement |

`payment.succeeded` and `payment.failed` [webhooks](/docs/webhooks) fire on settlement, and `invoice.created` when the invoice is issued.
