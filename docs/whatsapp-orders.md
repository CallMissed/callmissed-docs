---
title: "Catalog & Orders"
description: "Turn on the WhatsApp cart and storefront, send catalog and product messages, and the order read API."
slug: "whatsapp-orders"
breadcrumb: "WhatsApp"
---

# Catalog & Orders

Turn on the WhatsApp cart and storefront, send catalog and product messages, and the order read API.

## Overview

Selling from a WhatsApp catalog takes three steps:

1. **Turn the cart and storefront on** for the sending number with [commerce settings](#commerce-settings).
2. **Show products** in the chat with a [catalog, product or product list message](#send-product-messages).
3. **Read the orders** customers send back from their cart with the [Orders API](#orders).

The catalog itself (products, prices, images) is managed in Meta Commerce Manager and connected to your WhatsApp Business Account there. These endpoints use it; they do not edit it.

## Commerce settings

Two switches per phone number, read and written live at WhatsApp. `phone_id` is CallMissed's id for the number, from [List numbers](/docs/whatsapp-api#list-numbers).

### Read the settings

`GET /api/v1/whatsapp/phone_numbers/{phone_id}/commerce_settings` · scope `whatsapp:read`

```bash
curl https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/commerce_settings \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "phone_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "phone_number_id": "1234567890",
  "is_cart_enabled": true,
  "is_catalog_visible": false
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `is_cart_enabled` | boolean, nullable | Customers can add products to a cart and send it as an order |
| `is_catalog_visible` | boolean, nullable | The storefront icon shows in the chat header |

`null` means WhatsApp did not report the switch, which is not the same as `false`. WhatsApp's defaults are cart **on** and storefront icon **hidden**.

### Update the settings

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/commerce_settings` · scope `whatsapp:write`

| Field | Type | Notes |
| --- | --- | --- |
| `is_cart_enabled` | boolean | Optional |
| `is_catalog_visible` | boolean | Optional |

Send at least one, otherwise `400`. Only the switches you send change.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/commerce_settings \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "is_catalog_visible": true }'
```

Returns the settings object, re-read from WhatsApp after the write.

| Status | When |
| --- | --- |
| `400` | Neither switch sent, or no business token on file for the number |
| `403` | Key is missing `whatsapp:read` / `whatsapp:write` |
| `404` | `Phone number not found` |

## Send product messages

Three interactive message types that show catalog products in the chat. All three take `whatsapp:send`, pick the sending number with `phone_id` or `phone_number_id` (one is required, the same as [every send](/docs/whatsapp-api#choosing-the-sending-number)), and return the [common send response](/docs/whatsapp-messages#the-common-response). They are free-form messages, so they need an open 24-hour customer service window, and they are priced and credit-checked like any other send.

`product_retailer_id` is the product's SKU, shown as **Content ID** in Commerce Manager.

### Send a catalog message

`POST /api/v1/whatsapp/messages/catalog` · scope `whatsapp:send`

A bubble with a **View catalog** button that opens the catalog connected to the sending number.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `to` | string, 5 to 20 chars | Yes | The customer's number |
| `body_text` | string, 1 to 1024 chars | Yes | |
| `footer_text` | string, max 60 | No | |
| `thumbnail_product_retailer_id` | string, max 256 | No | The product whose image becomes the header. Without it WhatsApp uses the first catalog item |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/catalog \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "body_text": "Our new single-origin range is in. Tap below to browse.",
    "thumbnail_product_retailer_id": "SKU-114"
  }'
```

### Send a single product

`POST /api/v1/whatsapp/messages/product` · scope `whatsapp:send`

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `to` | string, 5 to 20 chars | Yes | |
| `catalog_id` | string, 1 to 256 chars | Yes | Meta's catalog id |
| `product_retailer_id` | string, 1 to 256 chars | Yes | |
| `body_text` | string, max 1024 | No | |
| `footer_text` | string, max 60 | No | |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/product \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "catalog_id": "998877665544",
    "product_retailer_id": "SKU-114",
    "body_text": "The one you asked about."
  }'
```

### Send a product list

`POST /api/v1/whatsapp/messages/product_list` · scope `whatsapp:send`

Up to 30 products, grouped in sections.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `to` | string, 5 to 20 chars | Yes | |
| `catalog_id` | string, 1 to 256 chars | Yes | |
| `header_text` | string, 1 to 60 chars | Yes | Text header, required for this type |
| `body_text` | string, 1 to 1024 chars | Yes | |
| `footer_text` | string, max 60 | No | |
| `sections` | object[] | Yes | At least one |
| `sections[].title` | string, max 24 | When more than one section | |
| `sections[].product_items` | object[] | Yes | At least one `{ "product_retailer_id": "..." }`. At most 30 products across all sections |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/product_list \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "catalog_id": "998877665544",
    "header_text": "Bestsellers",
    "body_text": "Our three most-ordered coffees this month.",
    "sections": [
      { "title": "Beans", "product_items": [ { "product_retailer_id": "SKU-114" }, { "product_retailer_id": "SKU-220" } ] },
      { "title": "Ground", "product_items": [ { "product_retailer_id": "SKU-305" } ] }
    ]
  }'
```

### Send errors

| Status | When |
| --- | --- |
| `400` | Neither `phone_id` nor `phone_number_id` sent, more than 30 products, or a missing section title when there is more than one section |
| `402` | Not enough credits, or a workspace budget cap would be exceeded. Nothing was sent |
| `403` | Key is missing `whatsapp:send` |
| `404` | `Phone number not found` |
| `409` | The number is disconnected. Reconnect it |
| `422` | The body failed validation (a missing required field or an over-long one), or WhatsApp rejected the message, for example because the 24-hour window is closed |

## Orders

When a customer builds a cart from your WhatsApp catalog and sends it, the order is recorded against your workspace. These endpoints read those orders and their line items.

> **Not populated yet.** Inbound cart orders are not written to this store today, so both read endpoints currently return an empty result. The request and response shapes below are stable; use them once order capture is live.

Orders are created by the customer's action on WhatsApp. There is no create endpoint.

### Authentication

```
Authorization: Bearer cm_your_api_key
```

Reading orders requires `wa_commerce:read`.

```json
{ "detail": "API key missing required scope: wa_commerce:read. Add it under the key's 'Permissions' section in your dashboard." }
```

### Statuses

`pending`, `processing`, `partially_shipped`, `shipped`, `completed`, `canceled`.

`completed` and `canceled` are terminal.

### The order object

```json
{
  "id": "o1a2…",
  "tenant_id": "a0b1…",
  "conversation_id": "c0ff…",
  "contact_id": "4411…",
  "wa_message_id": "wamid.HBg…",
  "catalog_id": "998877665544",
  "reference_id": "ORD-4471",
  "status": "pending",
  "currency": "INR",
  "subtotal": 2499.0,
  "note": "Please deliver after 6pm",
  "created_at": "2026-08-17T07:20:00Z",
  "updated_at": "2026-08-17T07:20:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `wa_message_id` | `string` | The WhatsApp message that carried the cart |
| `reference_id` | `string \| null` | Your own bill reference, once one has been attached |
| `subtotal` | `number` | Sum of the line items, in `currency` |
| `note` | `string \| null` | Free-text the customer typed with the order |

### GET `/api/v1/commerce/orders`

Newest first.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `status` | `string` | No | One of the six statuses |
| `contact_id` | `UUID` | No | One customer's orders |
| `created_from` | `datetime` | No | Inclusive lower bound on `created_at` |
| `created_to` | `datetime` | No | Inclusive upper bound on `created_at` |
| `limit` | `integer` | No | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | No | `0 <= offset <= 100000`, default `0` |

```bash
curl "https://api.callmissed.com/api/v1/commerce/orders?status=pending&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

An unknown status returns `422 status must be one of: pending, processing, partially_shipped, shipped, completed, canceled`.

### GET `/api/v1/commerce/orders/{order_id}`

The order plus its line items, oldest first.

```json
{
  "id": "o1a2…",
  "status": "pending",
  "currency": "INR",
  "subtotal": 2499.0,
  "items": [
    {
      "id": "i9b8…",
      "product_retailer_id": "SKU-114",
      "quantity": 2,
      "item_price": 999.0,
      "currency": "INR",
      "created_at": "2026-08-17T07:20:00Z"
    },
    {
      "id": "i7c6…",
      "product_retailer_id": "SKU-220",
      "quantity": 1,
      "item_price": 501.0,
      "currency": "INR",
      "created_at": "2026-08-17T07:20:00Z"
    }
  ]
}
```

`product_retailer_id` is your own SKU as it appears in the catalog — join on it to look the product up in your system.

`404 Order not found` for an unknown id or another tenant's order.

### Orders errors

| Status | When |
| --- | --- |
| `403` | Key is missing `wa_commerce:read` |
| `404` | `Order not found` |
| `422` | Unknown `status` value |

Reading orders does not consume credits.
