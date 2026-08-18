---
title: "Flows"
description: "Build native in-chat forms — create a Flow from its screen JSON, publish it, and read the submissions customers send back."
slug: "whatsapp-flows"
breadcrumb: "WhatsApp"
---

# Flows

Build native in-chat forms — create a Flow from its screen JSON, publish it, and read the submissions customers send back.

## Overview

A **Flow** is a multi-screen form WhatsApp renders **inside the conversation** — no browser, no link-out. Customers pick dates, confirm an address or answer a survey without leaving the chat, and the submission comes back to you as structured JSON.

Typical uses: cash-on-delivery confirmation, address capture, lead qualification, appointment booking, and post-conversation surveys.

## Lifecycle

```
create (DRAFT) ──▶ publish (PUBLISHED) ──▶ send ──▶ read responses
```

1. **Create** with a `flow_json` screen document and one or more categories. It starts as `DRAFT`.
2. **Publish** it. One way, and after publishing the screen document is frozen — a change means a new Flow.
3. **Send** it with [`POST /api/v1/whatsapp/messages/interactive`](/docs/whatsapp-messages#flow) using `interactive_type: "flow"`. That is the billed, window-aware send path for every interactive message.
4. **Read** the submissions here.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List, get, read responses | `wa_flows:read` |
| Create, publish, delete | `wa_flows:write` |

Your tenant also needs a connected WhatsApp number and Business Account. Without one you get `409 No connected WhatsApp number for tenant`.

## Statuses

`DRAFT`, `PUBLISHED`, `DEPRECATED`, `BLOCKED`, `THROTTLED`. Only the first two are ever set from this API; the rest can appear when WhatsApp changes a Flow's state on its side.

## Categories

Every Flow declares 1–8 categories: `SIGN_UP`, `SIGN_IN`, `APPOINTMENT_BOOKING`, `LEAD_GENERATION`, `CONTACT_US`, `CUSTOMER_SUPPORT`, `SURVEY`, `OTHER`.

## The flow object

```json
{
  "id": "f1a2…",
  "tenant_id": "a0b1…",
  "flow_id": "1122334455667788",
  "name": "Appointment booking",
  "categories": ["APPOINTMENT_BOOKING"],
  "status": "PUBLISHED",
  "flow_json": { "version": "7.0", "screens": [] },
  "endpoint_uri": null,
  "created_at": "2026-08-09T09:00:00Z",
  "updated_at": "2026-08-09T09:30:00Z"
}
```

> There are **two ids**. `id` is the CallMissed record and is what every path parameter on this page takes. `flow_id` is WhatsApp's id — that is the one you pass to the send endpoint.

`endpoint_uri` decides the Flow's kind:

| `endpoint_uri` | Kind | Behaviour |
| --- | --- | --- |
| `null` | **Static** | Every screen is defined up front in `flow_json` |
| Set | **Endpoint-backed** | Screens are served from your endpoint at runtime via `data_exchange` |

Start static. It needs no server, no encryption key and no runtime availability on your side.

## GET `/api/v1/commerce/flows`

Newest first.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `status` | `string` | One of the five statuses |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

## POST `/api/v1/commerce/flows`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, not blank |
| `categories` | `string[]` | Yes | 1–8 entries from the category list |
| `flow_json` | `object` | Yes | The screen document. At most 10 MB serialised |
| `endpoint_uri` | `string` | No | At most 512 characters. Omit for a static Flow |

```bash
curl -X POST https://api.callmissed.com/api/v1/commerce/flows \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Appointment booking",
    "categories": ["APPOINTMENT_BOOKING"],
    "flow_json": { "version": "7.0", "screens": [] }
  }'
```

Returns `201` with `status: "DRAFT"`.

The Flow is created at WhatsApp **first**, then mirrored locally — so a rejected screen document never leaves a phantom record behind. WhatsApp's validation message is passed through verbatim, which is what you want when a screen definition is malformed.

## GET `/api/v1/commerce/flows/{flow_uuid}`

One Flow by its CallMissed `id`. `404 Flow not found`.

## POST `/api/v1/commerce/flows/{flow_uuid}/publish`

No body. Moves `DRAFT` to `PUBLISHED`.

```bash
curl -X POST https://api.callmissed.com/api/v1/commerce/flows/f1a2…/publish \
  -H "Authorization: Bearer cm_your_api_key"
```

One way, and irreversible. Publishing freezes the screen document — iterate while the Flow is still a draft. WhatsApp validates the whole document at this point, so this is where a structural mistake surfaces.

## DELETE `/api/v1/commerce/flows/{flow_uuid}`

Returns `204`.

WhatsApp refuses to delete a `PUBLISHED` Flow and its refusal is passed through. If the Flow is already gone upstream, the local record is still cleared, so a stale mirror can always be tidied.

## Responses

A customer's submission arrives on your inbound webhook and is recorded here, correlated by the `flow_token` you set when sending. Recording is idempotent per WhatsApp message id, so a webhook redelivery never doubles a submission.

### GET `/api/v1/commerce/flows/responses`

Newest first.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `flow_id` | `string` | WhatsApp's flow id, at most 64 characters |
| `flow_token` | `string` | At most 128 characters — the identifier you sent |
| `contact_id` | `UUID` | |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

```json
[
  {
    "id": "r9c8…",
    "tenant_id": "a0b1…",
    "flow_id": "1122334455667788",
    "flow_token": "booking-4471",
    "wa_message_id": "wamid.HBg…",
    "contact_id": "4411…",
    "conversation_id": "c0ff…",
    "response": { "date": "2026-08-22", "slot": "10:30", "branch": "Kothrud" },
    "created_at": "2026-08-17T07:41:00Z"
  }
]
```

`response` is the screen data the customer submitted, exactly as your `flow_json` defined the field names.

Filtering by your own `flow_token` is the reliable way to tie a submission back to the order, booking or ticket you sent it for — set it to something meaningful when you send.

### GET `/api/v1/commerce/flows/responses/{response_id}`

One submission by its CallMissed id. `404 Flow response not found`.

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `wa_flows:read` / `wa_flows:write` |
| `404` | Flow or response not in your tenant |
| `409` | No connected WhatsApp number or Business Account for your tenant |
| `422` | Blank name, no categories, an unknown category, or a `flow_json` over 10 MB |
| `502` | WhatsApp accepted the call but returned no flow id |

Errors originating at WhatsApp keep their status code and message, so a validation failure reads the same as it would against the Cloud API directly.

Creating, publishing, deleting and reading Flows do not consume credits. Sending a Flow message is billed on the [interactive message endpoint](/docs/whatsapp-messages#flow).
