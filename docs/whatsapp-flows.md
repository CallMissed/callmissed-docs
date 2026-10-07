---
title: "Flows"
description: "Build native in-chat forms: create a Flow from its screen JSON, preview, publish, deprecate and send it."
slug: "whatsapp-flows"
breadcrumb: "WhatsApp"
---

# Flows

Build native in-chat forms: create a Flow from its screen JSON, preview, publish, deprecate and send it.

## Overview

A **Flow** is a multi-screen form WhatsApp renders **inside the conversation** — no browser, no link-out. Customers pick dates, confirm an address or answer a survey without leaving the chat, and the submission comes back to you as structured JSON.

Typical uses: cash-on-delivery confirmation, address capture, lead qualification, appointment booking, and post-conversation surveys.

## Two APIs for the same Flows

Flows live on your WhatsApp Business Account (WABA). CallMissed gives you two ways to manage them. Both act on the same WABA, and a Flow made with either one is sent the same way.

| | [WhatsApp Flow API](#whatsapp-flow-api) | [Flow records API](#flow-records-api) |
| --- | --- | --- |
| Base path | `/api/v1/whatsapp/flows` | `/api/v1/commerce/flows` |
| Scopes | `whatsapp:read` / `whatsapp:write` | `wa_flows:read` / `wa_flows:write` |
| Ids in paths | WhatsApp's Flow id | CallMissed's record id (UUID) |
| What a list returns | Every Flow on the WABA, read live from WhatsApp, including ones made in WhatsApp Manager | Only the Flows created through this API, from CallMissed's stored copy |
| Operations | Create, list, get, preview, update metadata, replace the Flow JSON, publish, deprecate, delete | Create, list, get, publish, delete, read submissions |
| Choosing the WABA | `account_id` or `waba_id` on every call | Your workspace's first connected number |

Use the **WhatsApp Flow API** to build and iterate on a Flow: it is the only one that can replace the Flow JSON of a draft, give you a preview link, or deprecate a published Flow. Use the **Flow records API** when you want CallMissed to keep the Flow JSON alongside a record id, and to read submissions.

## Lifecycle

```
create (DRAFT) ──▶ publish (PUBLISHED) ──▶ send ──▶ read responses
                                       └──▶ deprecate (DEPRECATED)
```

1. **Create** with a Flow JSON screen document and one or more categories. It starts as `DRAFT`.
2. **Iterate** while it is a draft: replace the Flow JSON and open the preview link (WhatsApp Flow API).
3. **Publish** it. One way, and after publishing the screen document is frozen. A change means a new Flow.
4. **Send** it with [`POST /api/v1/whatsapp/messages/interactive`](/docs/whatsapp-messages#send-an-interactive-message) using `interactive_type: "flow"` and WhatsApp's Flow id. That is the billed, window-aware send path for every interactive message.
5. **Read** the submissions with the [Flow records API](#responses) (not populated yet, see that section).
6. **Deprecate** a published Flow you no longer want sent. A published Flow cannot be deleted.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| WhatsApp Flow API: list, get, preview | `whatsapp:read` |
| WhatsApp Flow API: create, update, replace JSON, publish, deprecate, delete | `whatsapp:write` |
| Flow records API: list, get, read responses | `wa_flows:read` |
| Flow records API: create, publish, delete | `wa_flows:write` |

Your workspace also needs a connected WhatsApp number and Business Account. Without one, the Flow records API returns `409 No connected WhatsApp number for tenant` and the WhatsApp Flow API returns `404 WhatsApp account not found`.

## Statuses

`DRAFT`, `PUBLISHED`, `DEPRECATED`, `BLOCKED`, `THROTTLED`. Only the first two are ever set from this API; the rest can appear when WhatsApp changes a Flow's state on its side.

## Categories

Every Flow declares 1–8 categories: `SIGN_UP`, `SIGN_IN`, `APPOINTMENT_BOOKING`, `LEAD_GENERATION`, `CONTACT_US`, `CUSTOMER_SUPPORT`, `SURVEY`, `OTHER`.

## WhatsApp Flow API

Every call reads or writes the Flow at WhatsApp directly. Nothing is cached, so a status WhatsApp changed on its side (`BLOCKED`, `THROTTLED`) shows up on the next read.

### Choosing the WABA

Every route takes the WABA as a selector: `account_id` (CallMissed's UUID from [List accounts](/docs/whatsapp-api#list-accounts)) or `waba_id` (Meta's id, at most 64 characters). On `POST /flows` it goes in the body; on every other route it is a query parameter. One of the two is required, otherwise `400 Either account_id (UUID) or waba_id (Meta) is required`.

Before acting on a Flow id, CallMissed checks that the Flow is on that WABA. A Flow id that is not returns the same `404 Flow not found` as one that does not exist.

### The Flow object

```json
{
  "id": "1122334455667788",
  "name": "Appointment booking",
  "status": "DRAFT",
  "categories": ["APPOINTMENT_BOOKING"],
  "validation_errors": [],
  "json_version": "7.0",
  "data_api_version": null,
  "endpoint_uri": null
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | WhatsApp's Flow id. Use it in paths here and as `flow_id` when sending |
| `name` | string, nullable | |
| `status` | string, nullable | One of the five statuses |
| `categories` | string[] | |
| `validation_errors` | object[] | Problems WhatsApp found in your Flow JSON, as WhatsApp reports them (message, line and column spans). A Flow cannot be published while this is non-empty |
| `json_version` | string, nullable | The Flow JSON version |
| `data_api_version` | string, nullable | Set on endpoint-backed Flows |
| `endpoint_uri` | string, nullable | Set on endpoint-backed Flows |

### Create a Flow

`POST /api/v1/whatsapp/flows` · scope `whatsapp:write`

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `account_id` / `waba_id` | UUID / string | One of the two | The WABA to create the Flow on |
| `name` | string, 1 to 512 chars | Yes | |
| `categories` | string[] | Yes | At least one, from the category list |
| `flow_json` | object or string | No | The screen document, as an object or an already-encoded JSON string. Can be added later with [Replace the Flow JSON](#replace-the-flow-json) |
| `clone_flow_id` | string, max 64 | No | Start from a copy of an existing Flow on the same WABA |
| `endpoint_uri` | string, max 2048 | No | Your data-exchange endpoint. Omit for a static Flow |
| `publish` | boolean | No | Publish on creation. Irreversible. Default off |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/flows \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "name": "Appointment booking",
    "categories": ["APPOINTMENT_BOOKING"],
    "flow_json": { "version": "7.0", "screens": [] }
  }'
```

**Response (200 OK)**

```json
{
  "id": "1122334455667788",
  "success": true,
  "validation_errors": []
}
```

A `200` with entries in `validation_errors` means the Flow **was** created and its JSON needs fixing before it can be published. The response carries the `id` either way.

### List Flows

`GET /api/v1/whatsapp/flows` · scope `whatsapp:read`

| Param | Type | Default | Notes |
| --- | --- | --- | --- |
| `account_id` / `waba_id` | UUID / string | | One of the two is required |
| `limit` | integer, 1 to 100 | 50 | Page size |
| `after` | string, max 512 | | `next_cursor` from the previous page |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/flows?waba_id=102290129340398&limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "flows": [
    {
      "id": "1122334455667788",
      "name": "Appointment booking",
      "status": "PUBLISHED",
      "categories": ["APPOINTMENT_BOOKING"],
      "validation_errors": [],
      "json_version": null,
      "data_api_version": null,
      "endpoint_uri": null
    }
  ],
  "next_cursor": null
}
```

`next_cursor` is `null` on the last page.

### Get a Flow

`GET /api/v1/whatsapp/flows/{flow_id}?waba_id=...` · scope `whatsapp:read`

Returns one Flow object, including `json_version`, `data_api_version`, `endpoint_uri` and `validation_errors`.

### Preview a Flow

`GET /api/v1/whatsapp/flows/{flow_id}/preview?waba_id=...` · scope `whatsapp:read`

| Param | Type | Default | Notes |
| --- | --- | --- | --- |
| `invalidate` | boolean | `false` | `true` issues a new link and stops the previous one working |

```json
{
  "preview_url": "<the preview link WhatsApp returns>",
  "expires_at": "2026-08-24T09:00:00+0000"
}
```

Open `preview_url` in a browser to click through the Flow as a customer would. Leave `invalidate` off unless a link has been shared too widely.

### Update Flow metadata

`PATCH /api/v1/whatsapp/flows/{flow_id}?waba_id=...` · scope `whatsapp:write`

| Field | Type | Notes |
| --- | --- | --- |
| `name` | string, 1 to 512 chars | |
| `categories` | string[] | |
| `endpoint_uri` | string, max 2048 | |

Send at least one field, otherwise `400`. Only the fields you send change. Returns `{ "success": true }`. A published Flow cannot be modified, and WhatsApp's refusal comes back as `422`.

### Replace the Flow JSON

`POST /api/v1/whatsapp/flows/{flow_id}/assets?waba_id=...` · scope `whatsapp:write`

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `flow_json` | object or string | Yes | The full screen document. WhatsApp limits it to 10 MB |

```bash
curl -X POST "https://api.callmissed.com/api/v1/whatsapp/flows/1122334455667788/assets?waba_id=102290129340398" \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "flow_json": { "version": "7.0", "screens": [] } }'
```

```json
{ "success": true, "validation_errors": [] }
```

As with create, a `200` with `validation_errors` means the upload landed and the document has problems to fix before you can publish.

### Publish a Flow

`POST /api/v1/whatsapp/flows/{flow_id}/publish?waba_id=...` · scope `whatsapp:write`

No body. Returns `{ "success": true }`. **Irreversible**: a published Flow cannot be updated or deleted, only deprecated. WhatsApp refuses to publish while validation errors are outstanding.

### Deprecate a Flow

`POST /api/v1/whatsapp/flows/{flow_id}/deprecate?waba_id=...` · scope `whatsapp:write`

No body. Returns `{ "success": true }`. A deprecated Flow can no longer be sent or opened. It does not undo publishing.

### Delete a Flow

`DELETE /api/v1/whatsapp/flows/{flow_id}?waba_id=...` · scope `whatsapp:write`

Returns `204`. Only a `DRAFT` Flow can be deleted. WhatsApp's refusal for any other status comes back as `422`; deprecate a published Flow instead.

### Errors

| Status | When |
| --- | --- |
| `400` | No `account_id` or `waba_id`, an empty `PATCH` body, or WhatsApp rejected the name, categories or JSON |
| `401` | The WABA's access token is no longer valid. Reconnect the number |
| `403` | Key is missing `whatsapp:read` / `whatsapp:write`, or the WABA has not granted the permission this needs |
| `404` | `WhatsApp account not found`, or `Flow not found` on that WABA |
| `409` | The WABA is disconnected, or has no usable token. Reconnect it |
| `422` | WhatsApp refused the request for the Flow's current status (for example, editing or deleting a published Flow) |
| `502` | WhatsApp did not respond correctly. Retry |

Managing Flows does not consume credits.

## Flow records API

CallMissed keeps a copy of each Flow created here, with the Flow JSON you sent and its own record id. The WABA is your workspace's first connected number's account.

### The flow record

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

> There are **two ids**. `id` is the CallMissed record and is what every path parameter in the Flow records API takes. `flow_id` is WhatsApp's id — that is the one you pass to the send endpoint.

`endpoint_uri` decides the Flow's kind:

| `endpoint_uri` | Kind | Behaviour |
| --- | --- | --- |
| `null` | **Static** | Every screen is defined up front in `flow_json` |
| Set | **Endpoint-backed** | Screens are served from your endpoint at runtime via `data_exchange` |

Start static. It needs no server, no encryption key and no runtime availability on your side.

### GET `/api/v1/commerce/flows`

Newest first.

| Parameter | Type | Constraints |
| --- | --- | --- |
| `status` | `string` | One of the five statuses |
| `limit` | `integer` | `1 <= limit <= 200`, default `50` |
| `offset` | `integer` | `0 <= offset <= 100000`, default `0` |

### POST `/api/v1/commerce/flows`

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

### GET `/api/v1/commerce/flows/{flow_uuid}`

One Flow by its CallMissed `id`. `404 Flow not found`.

### POST `/api/v1/commerce/flows/{flow_uuid}/publish`

No body. Moves `DRAFT` to `PUBLISHED`.

```bash
curl -X POST https://api.callmissed.com/api/v1/commerce/flows/f1a2…/publish \
  -H "Authorization: Bearer cm_your_api_key"
```

One way, and irreversible. Publishing freezes the screen document — iterate while the Flow is still a draft. WhatsApp validates the whole document at this point, so this is where a structural mistake surfaces.

### DELETE `/api/v1/commerce/flows/{flow_uuid}`

Returns `204`.

WhatsApp refuses to delete a `PUBLISHED` Flow and its refusal is passed through. If the Flow is already gone upstream, the local record is still cleared, so a stale mirror can always be tidied.

## Responses

These endpoints are designed to hold each customer's submission, correlated by the `flow_token` you set when sending and idempotent per WhatsApp message id.

> **Not populated yet.** Submissions are not written to this store today, so both endpoints currently return an empty result. Until capture is live, the submitted screen data is not available through the API.

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

## Flow records errors

| Status | When |
| --- | --- |
| `403` | Key is missing `wa_flows:read` / `wa_flows:write` |
| `404` | Flow or response not in your tenant |
| `409` | No connected WhatsApp number or Business Account for your tenant |
| `422` | Blank name, no categories, an unknown category, or a `flow_json` over 10 MB |
| `502` | WhatsApp accepted the call but returned no flow id |

Errors originating at WhatsApp keep their status code and message, so a validation failure reads the same as it would against the Cloud API directly.

Creating, publishing, deleting and reading Flows do not consume credits. Sending a Flow message is billed on the [interactive message endpoint](/docs/whatsapp-messages#send-an-interactive-message).
