---
title: "Payments"
description: "Take UPI payments inside a WhatsApp chat: create and manage payment configurations on a WABA, then send order details and order status messages."
slug: "whatsapp-payments"
breadcrumb: "WhatsApp"
---

# Payments

Take UPI payments inside a WhatsApp chat: create and manage payment configurations on a WABA, then send order details and order status messages.

WhatsApp Payments lets a customer pay an itemised bill from the chat itself with any UPI app. Two pieces: a **payment configuration** on the WABA that says where the money lands, and the two [order messages](/docs/whatsapp-messages#send-an-order-details-message) that bill the customer and then settle the order.

All endpoints are under `https://api.callmissed.com/api/v1/whatsapp`.

> **India and UPI only.** These endpoints implement the India flow, with `payment_type: "upi"` and `INR`. Any other `payment_type` on a send returns `501`, because other regions use a different request shape rather than a variation of this one.

## How it fits together

:::flow
icon:gateway | Configure | `POST /payment_configurations` registers a UPI VPA or a payment gateway on the WABA
icon:user | Link | For a gateway, the merchant opens the returned `oauth_url` to finish linking. A VPA is usable immediately
icon:send | Bill | `POST /messages/order_details` sends the itemised bill. The customer pays in their UPI app
icon:done | Settle | `POST /messages/order_status` moves the order off "Order pending"
:::

## Choosing the WABA

A payment configuration belongs to a **WhatsApp Business Account**, not to a phone number, so these endpoints take the same account selector as [templates](/docs/whatsapp-templates#choosing-the-waba). Supply exactly one:

| Field | Type | Where it comes from |
|---|---|---|
| `account_id` | UUID | The `id` from `GET /accounts` |
| `waba_id` | string, max 64 | Meta's WABA id |

The account is always resolved against your workspace, so naming a WABA you do not own returns the same `404` as one that does not exist.

The two order sends are per-**number** instead, and take `phone_id` or `phone_number_id` like every other send.

## Providers

`provider_name` picks how the money is collected.

| `provider_name` | What it is | Ready when |
|---|---|---|
| `upi_vpa` | A UPI VPA handle you own, linked directly | Immediately |
| `razorpay` | Payment gateway | After the merchant completes the OAuth link |
| `payu` | Payment gateway | After the merchant completes the OAuth link |
| `zaakpay` | Payment gateway | After the merchant completes the OAuth link |

A gateway configuration exists as soon as you create it but **cannot take a payment** until the merchant visits the `oauth_url` the create returns. Until then, an order details message quoting it will not be payable.

## Create a payment configuration

`POST /api/v1/whatsapp/payment_configurations` · scope `whatsapp:write`

| Field | Type | Required | Notes |
|---|---|---|---|
| `account_id` / `waba_id` | UUID / string | One of | The WABA to configure |
| `configuration_name` | string, 1 to 60 chars | Yes | The name you quote as `payment_configuration` when sending an order |
| `provider_name` | string, 1 to 32 chars | Yes | One of the providers above |
| `merchant_vpa` | string, max 256 | For `upi_vpa` | The VPA handle to collect into |
| `merchant_category_code` | string, max 32 | No | Your MCC |
| `purpose_code` | string, max 32 | No | Purpose code, where your provider requires one |
| `redirect_url` | string, max 2048 | No | Where to send the merchant after they finish the OAuth link |

:::tabs
```bash [UPI VPA]
curl -X POST https://api.callmissed.com/api/v1/whatsapp/payment_configurations \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "configuration_name": "acme-upi",
    "provider_name": "upi_vpa",
    "merchant_vpa": "acmecoffee@okhdfcbank"
  }'
```
```bash [Gateway]
curl -X POST https://api.callmissed.com/api/v1/whatsapp/payment_configurations \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "configuration_name": "acme-razorpay",
    "provider_name": "razorpay",
    "redirect_url": "https://acme.example.com/payments/linked"
  }'
```
:::

**Response (200 OK)**

```json
{
  "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "waba_id": "102290129340398",
  "configuration_name": "acme-razorpay",
  "success": true,
  "oauth_url": "https://business.example.com/payments/link?token=...",
  "expiration": 1776000000
}
```

| Field | Type | Notes |
|---|---|---|
| `account_id` | UUID | The WABA's CallMissed id |
| `waba_id` | string | Meta's WABA id |
| `configuration_name` | string | Echoes the name you created |
| `success` | boolean | Whether the configuration was created |
| `oauth_url` | string, nullable | Present for a gateway provider only. The merchant must visit it to finish linking |
| `expiration` | integer, nullable | When that link stops working |

## List payment configurations

`GET /api/v1/whatsapp/payment_configurations` · scope `whatsapp:read`

Read live, with no cached fallback, so you never see a status we stored earlier and never refreshed.

| Query param | Type | Required | Notes |
|---|---|---|---|
| `account_id` / `waba_id` | UUID / string | One of | The WABA to read |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/payment_configurations?waba_id=102290129340398" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "waba_id": "102290129340398",
  "payment_configurations": [
    {
      "configuration_name": "acme-upi",
      "status": "Active",
      "provider_name": "upi_vpa",
      "provider_mid": null,
      "merchant_vpa": "acmecoffee@okhdfcbank",
      "merchant_category_code": { "code": "5814", "description": "Restaurants" },
      "purpose_code": null,
      "created_timestamp": 1774000000,
      "updated_timestamp": 1774000000
    }
  ]
}
```

### The payment configuration object

| Field | Type | Notes |
|---|---|---|
| `configuration_name` | string | The name you quote when sending an order |
| `status` | string, nullable | `Active`, `Needs_Connecting` or `Needs_Testing`. Only `Active` can take a payment |
| `provider_name` | string, nullable | The provider it was created with |
| `provider_mid` | string, nullable | The gateway's merchant id, where the provider issues one |
| `merchant_vpa` | string, nullable | The VPA handle, for a `upi_vpa` configuration |
| `merchant_category_code` | string or object, nullable | Reported either as a plain code or as `{ code, description }` |
| `purpose_code` | string or object, nullable | Same, when set |
| `created_timestamp` / `updated_timestamp` | integer, nullable | Epoch seconds |

Fields are broadly optional because the read endpoints and the status webhook each report a different subset.

## Get one payment configuration

`GET /api/v1/whatsapp/payment_configurations/{configuration_name}` · scope `whatsapp:read`

| Query param | Type | Required | Notes |
|---|---|---|---|
| `account_id` / `waba_id` | UUID / string | One of | The WABA to read |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/payment_configurations/acme-upi?waba_id=102290129340398" \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns a single [payment configuration object](#the-payment-configuration-object), or `404` when no configuration on that WABA carries the name.

Poll this after creating a gateway configuration to see it move to `Active` once the merchant has finished linking.

## Regenerate the OAuth link

`POST /api/v1/whatsapp/payment_configurations/{configuration_name}/oauth_link` · scope `whatsapp:write`

The link a gateway create returns expires. This is how a merchant who never finished linking, or whose link went stale, gets a fresh one without recreating the configuration.

| Field | Type | Required | Notes |
|---|---|---|---|
| `account_id` / `waba_id` | UUID / string | One of | The WABA |
| `redirect_url` | string | No | Where to send the merchant afterwards |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/payment_configurations/acme-razorpay/oauth_link \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "redirect_url": "https://acme.example.com/payments/linked"
  }'
```

**Response (200 OK)**

```json
{
  "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "waba_id": "102290129340398",
  "configuration_name": "acme-razorpay",
  "oauth_url": "https://business.example.com/payments/link?token=...",
  "expiration": 1776000000
}
```

Only meaningful for a gateway provider. A `upi_vpa` configuration has nothing to link.

## Delete a payment configuration

`DELETE /api/v1/whatsapp/payment_configurations/{configuration_name}` · scope `whatsapp:write`

| Query param | Type | Required | Notes |
|---|---|---|---|
| `account_id` / `waba_id` | UUID / string | One of | The WABA |

```bash
curl -X DELETE "https://api.callmissed.com/api/v1/whatsapp/payment_configurations/acme-razorpay?waba_id=102290129340398" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "waba_id": "102290129340398",
  "configuration_name": "acme-razorpay",
  "success": true
}
```

> **Stop sending first.** Make sure no new order messages quote this configuration before you unlink it, or those bills will have nowhere to collect into.

## Billing a customer

The two sends live with the rest of the send reference:

- [Send an order details message](/docs/whatsapp-messages#send-an-order-details-message) — the itemised bill, with the money rules and the full `order` shape
- [Send an order status update](/docs/whatsapp-messages#send-an-order-status-update) — the update that settles it

Both need the `whatsapp:send` scope and both are window-limited like any other free-form send. Tie the two together with `reference_id`: it is unique per order details message, and quoting it on an order status update is what moves that specific order off "Order pending".
