---
title: "Orders"
description: "Read the orders customers place from your WhatsApp catalog — filter by status, contact or date, and fetch line items."
slug: "whatsapp-orders"
breadcrumb: "WhatsApp"
---

# Orders

Read the orders customers place from your WhatsApp catalog — filter by status, contact or date, and fetch line items.

## Overview

When a customer builds a cart from your WhatsApp catalog and sends it, the order is recorded against your tenant. These endpoints read those orders and their line items.

Orders are created by the customer's action on WhatsApp — there is no create endpoint here.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

Reading orders requires `wa_commerce:read`.

```json
{ "detail": "API key missing required scope: wa_commerce:read. Add it under the key's 'Permissions' section in your dashboard." }
```

## Statuses

`pending`, `processing`, `partially_shipped`, `shipped`, `completed`, `canceled`.

`completed` and `canceled` are terminal.

## The order object

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

## GET `/api/v1/commerce/orders`

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

## GET `/api/v1/commerce/orders/{order_id}`

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

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `wa_commerce:read` |
| `404` | `Order not found` |
| `422` | Unknown `status` value |

Reading orders does not consume credits.
