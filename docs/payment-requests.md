---
title: "Payment Requests"
description: "Track the payment links your agents send to customers through your own Razorpay account, and cap how much an agent may request."
slug: "payment-requests"
breadcrumb: "API Reference"
---

# Payment Requests

Track the payment links your agents send to customers through your own Razorpay account, and cap how much an agent may request.

## Overview

A **payment request** is one payment link an agent sent to one of your customers during a call or chat: an EMI instalment, an order payment, an invoice. The link is created with **your own Razorpay account**, so the money is paid into your account. It is never CallMissed credits and never passes through CallMissed.

Agents create payment requests with the `create_payment_link` tool once Razorpay is connected. On a WhatsApp chat the link is sent in the chat; elsewhere (for example on a phone call) Razorpay texts it to the customer's phone and emails it when an email is known. This API is read-only for the requests themselves: you list them, read one, and set the per-link limit.

Status changes arrive from Razorpay and are applied to the request, then delivered to you as `payment_request.*` [webhook events](/docs/webhooks).

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, get, read settings | `integrations:read` |
| Update settings | `integrations:write` |

## Setup: connect Razorpay

1. In Razorpay Dashboard → Account & Settings → API Keys, create a key. Choose a webhook secret (at least 8 characters) and keep it for step 3.
2. Connect it with [`POST /api/v1/integrations`](/docs/integrations):

```bash
curl -X POST https://api.callmissed.com/api/v1/integrations \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "razorpay",
    "name": "Collections",
    "credentials": {
      "key_id": "rzp_live_XXXXXXXXXXXXXX",
      "key_secret": "your_key_secret",
      "webhook_secret": "the_secret_you_chose"
    }
  }'
```

   The key is checked with Razorpay before it is saved. A `rzp_test_…` key connects in test mode.
3. Call [`GET /api/v1/payment-requests/settings`](#get-apiv1payment-requestssettings). In Razorpay Dashboard → Webhooks, add a webhook whose URL is the returned `webhook_url`, whose secret is the one from step 1, and whose events are the returned `webhook_events`. Without this webhook, requests stay at `created` and no `payment_request.*` events are sent.

If more than one Razorpay integration is connected, the oldest one is used.

## Statuses

| `status` | Meaning |
| --- | --- |
| `creating` | Saved, link being created with Razorpay |
| `created` | Link created and sent; nothing paid yet |
| `partially_paid` | Part of the amount paid (`amount_paid_minor` is less than `amount_minor`) |
| `paid` | Paid in full. `paid_at` is set |
| `expired` | The link expired unpaid |
| `cancelled` | The link was cancelled |
| `failed` | Razorpay refused the link, or the result could not be confirmed. See `error` |

Statuses only move forward: a late or repeated event from Razorpay never moves a request back. `paid`, `expired` and `cancelled` are final. A `failed` request can still become `partially_paid` or `paid` if Razorpay reports a payment on its link.

## The payment request object

```json
{
  "id": "5c7d…",
  "provider": "razorpay",
  "status": "paid",
  "amount_minor": 149900,
  "amount_paid_minor": 149900,
  "currency": "INR",
  "description": "EMI 3 of 12",
  "external_reference": "LOAN-2291-03",
  "short_url": "https://rzp.io/i/AbC123",
  "customer_name": "Priya Sharma",
  "customer_phone": "+919812345678",
  "customer_email": null,
  "contact_id": "4411…",
  "bot_id": "c3d4…",
  "channel": "voice",
  "conversation_id": null,
  "voice_session_id": "8e9f…",
  "delivered_via": "sms",
  "error": null,
  "paid_at": "2026-09-26T10:42:00Z",
  "created_at": "2026-09-26T10:31:00Z",
  "updated_at": "2026-09-26T10:42:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `provider` | `string` | `razorpay` |
| `amount_minor` | `integer` | Requested amount in the smallest currency unit (paise): `149900` = ₹1,499.00 |
| `amount_paid_minor` | `integer` | Paid so far, in paise |
| `currency` | `string` | `INR` |
| `description` | `string` | What the payment is for, as shown to the customer |
| `external_reference` | `string \| null` | Your own reference (invoice, EMI or order number), if the agent was given one |
| `short_url` | `string \| null` | The payment link. `null` until the link is created |
| `customer_name` / `customer_phone` / `customer_email` | `string \| null` | Who the link was sent to. Phone in E.164 |
| `contact_id` | `UUID \| null` | The CRM contact, when known. Status changes are also added as a note on this contact |
| `bot_id` | `UUID \| null` | The agent that created the request |
| `channel` | `string \| null` | Where the agent was, e.g. `voice` or `whatsapp` |
| `conversation_id` / `voice_session_id` | `UUID \| null` | The chat or call it came from |
| `delivered_via` | `string \| null` | Comma-separated channels the link was sent through: `whatsapp`, `sms`, `email` |
| `error` | `string \| null` | Why the request `failed`, safe to show |
| `paid_at` | `datetime \| null` | When it became `paid` |

## GET `/api/v1/payment-requests`

Returns an array of payment request objects, newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `status` | `string` | No | One of the seven statuses |
| `contact_id` | `UUID` | No | One contact's requests |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/payment-requests?status=paid&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

An unknown `status` returns `422 status must be one of: creating, created, partially_paid, paid, expired, cancelled, failed`.

## GET `/api/v1/payment-requests/{request_id}`

One payment request. `404 Payment request not found`.

```bash
curl https://api.callmissed.com/api/v1/payment-requests/{request_id} \
  -H "Authorization: Bearer cm_your_api_key"
```

## GET `/api/v1/payment-requests/settings`

The connected Razorpay account, the per-link limit and the webhook to configure in Razorpay.

```bash
curl https://api.callmissed.com/api/v1/payment-requests/settings \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "connected": true,
  "integration_id": "6a1e…",
  "account": "rzp_live_XXXXXXXXXXXXXX",
  "mode": "live",
  "max_amount_inr": 10000,
  "platform_max_amount_inr": 500000,
  "webhook_url": "https://api.callmissed.com/api/v1/webhooks/razorpay/6a1e…",
  "webhook_events": [
    "payment_link.cancelled",
    "payment_link.expired",
    "payment_link.paid",
    "payment_link.partially_paid"
  ]
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `connected` | `boolean` | `false` when no Razorpay integration is connected. `integration_id`, `account`, `mode` and `webhook_url` are then `null` |
| `account` | `string \| null` | The connected Razorpay key id |
| `mode` | `string \| null` | `live` for a `rzp_live_…` key, otherwise `test` |
| `max_amount_inr` | `integer` | The largest amount, in whole rupees, an agent may request in one link. Default `10000` |
| `platform_max_amount_inr` | `integer` | The highest value `max_amount_inr` can be set to: `500000` |
| `webhook_url` | `string \| null` | Paste this exactly as returned into Razorpay's webhook settings |
| `webhook_events` | `string[]` | The Razorpay events to enable on that webhook |

## PUT `/api/v1/payment-requests/settings`

Sets the largest amount an agent may request in one payment link. Larger collections are left to a person.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `max_amount_inr` | `integer` | Yes | Whole rupees, `1 <= max_amount_inr <= 500000` |

```bash
curl -X PUT https://api.callmissed.com/api/v1/payment-requests/settings \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "max_amount_inr": 25000 }'
```

Returns the settings object. `404 Connect Razorpay first` when no Razorpay integration is connected; `422` when the value is out of range.

## Webhook events

Subscribe a [webhook](/docs/webhooks) to these events to hear about payments as they happen:

| Event | When |
| --- | --- |
| `payment_request.partially_paid` | A part payment was made |
| `payment_request.paid` | Paid in full |
| `payment_request.expired` | The link expired |
| `payment_request.cancelled` | The link was cancelled |

The event's `data` object:

```json
{
  "payment_request_id": "5c7d…",
  "status": "paid",
  "provider": "razorpay",
  "amount": 1499.0,
  "amount_paid": 1499.0,
  "currency": "INR",
  "description": "EMI 3 of 12",
  "reference": "LOAN-2291-03",
  "short_url": "https://rzp.io/i/AbC123",
  "contact_id": "4411…",
  "customer_phone": "+919812345678",
  "customer_email": null,
  "bot_id": "c3d4…",
  "paid_at": "2026-09-26T10:42:00.481203+00:00"
}
```

Note the units: in event payloads `amount` and `amount_paid` are in **rupees**, while the API's `amount_minor` and `amount_paid_minor` are in **paise**. `reference` is the API's `external_reference`.

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `integrations:read` / `integrations:write` |
| `404` | Payment request not in your account, or settings updated before Razorpay is connected |
| `422` | Unknown `status` filter, or `max_amount_inr` outside `1`–`500000` |

Nothing on this page consumes credits.
