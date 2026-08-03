---
title: "WhatsApp API"
description: "Programmatic WhatsApp Cloud API — onboard WABAs, send all message types, manage templates and campaigns, and read delivery analytics."
slug: "whatsapp-api"
breadcrumb: "WhatsApp"
---

# WhatsApp API

Programmatic WhatsApp Cloud API — onboard WABAs, send all message types, manage templates and campaigns, and read delivery analytics.

## Overview

The WhatsApp API is the programmatic send-and-manage surface for a connected **WhatsApp Business Account (WABA)**. Once a number is connected (see [WhatsApp Business Setup](/docs/whatsapp-setup)), you can send every WhatsApp message type — text, templates, media, interactive buttons/lists, location pins, reactions, and contact cards — manage your message templates, upload media, and read the connected accounts and phone numbers on your workspace.

**Base path:** `https://api.callmissed.com/api/v1/whatsapp`

> **The 24-hour customer service window.** Free-form messages (text, media, interactive, location) can only be sent inside the 24-hour window that opens when a user last messaged you. Outside that window you must send an approved **template** (see [Send a template message](#send-a-template-message)). A closed-window send returns `422` with an actionable message.

**Authentication.** Every endpoint is authenticated with your `cm_` API key (`Authorization: Bearer cm_...`); JWT sessions work too. API keys need a WhatsApp scope matching the action:

| Scope | Grants |
|-------|--------|
| `whatsapp:read` | List accounts, phone numbers, templates, webhook events; resolve/download media |
| `whatsapp:write` | Create/delete/sync templates; upload media; onboarding and number management |
| `whatsapp:send` | Send messages (text, template, media, interactive, location, reaction, contacts); mark read |

## Choosing the sending number

Every send endpoint needs to know **which connected number to send from**. Provide **one** of these two fields in the request body — one is required:

| Field | Type | Notes |
|-------|------|-------|
| `phone_id` | UUID | CallMissed's internal id for the connected number (from `GET /phone_numbers`) |
| `phone_number_id` | string | Meta's `phone_number_id` for the same number |

The lookup is always scoped to your workspace — you can only send from a number you own. A number that is disconnected or missing its stored credentials returns `409` (reconnect via Embedded Signup).

## Send a text message

Send a free-form text message. Only valid inside the 24-hour customer service window.

`POST /messages` · scope `whatsapp:send`

**Request body**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `phone_id` / `phone_number_id` | UUID / string | Yes (one) | The sending number — see [Choosing the sending number](#choosing-the-sending-number) |
| `to` | string (5–20) | Yes | Recipient in E.164, e.g. `+919000000000` |
| `text` | string (1–4096) | Yes | The message body |
| `preview_url` | boolean | No | Render a link preview for the first URL in the text (default `false`) |

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "text": "Hi! Thanks for reaching out — how can we help?",
    "preview_url": false
  }'
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/whatsapp"
headers = {"Authorization": "Bearer cm_your_api_key"}

resp = httpx.post(
    f"{BASE}/messages",
    headers=headers,
    json={
        "phone_number_id": "1234567890",
        "to": "+919000000000",
        "text": "Hi! Thanks for reaching out — how can we help?",
    },
)
print(resp.json()["wamid"])  # e.g. "wamid.HBgM...=="
```
:::

**Response (200 OK)** — the common send shape shared by every send endpoint:

```json
{
  "wamid": "wamid.HBgMOTE5MDAwMDAwMDAwFQIAERgS...==",
  "contacts": [
    { "input": "+919000000000", "wa_id": "919000000000" }
  ]
}
```

The `wamid` is Meta's message id — keep it to correlate delivery/read status events on your webhook.

**Status codes**

| Code | Meaning |
|------|---------|
| `200` | Message accepted by WhatsApp |
| `401` | Meta access token invalid or expired — reconnect the number |
| `403` | Missing `whatsapp:send` scope |
| `404` | Sending number not found on your workspace |
| `409` | Number disconnected, or not registered on the WhatsApp Business Platform |
| `422` | The 24-hour window is closed (send a template), or the request was rejected by WhatsApp |

## Send a template message

Send a pre-approved message template. Templates are the **only** way to message a user outside the 24-hour window (e.g. order updates, reminders, one-time codes). Create and manage templates via the [template endpoints](#manage-templates).

`POST /messages/template` · scope `whatsapp:send`

**Request body**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `phone_id` / `phone_number_id` | UUID / string | Yes (one) | The sending number |
| `to` | string (5–20) | Yes | Recipient in E.164 |
| `template_name` | string (1–512) | Yes | The approved template's name |
| `language_code` | string (≤ 12) | No | Template locale, e.g. `en_US`, `hi_IN` (default `en_US`) |
| `components` | array of objects | No | Header/body/button variable values — WhatsApp's `components` array |

The `components` array is passed through to WhatsApp unchanged, so it accepts any combination WhatsApp supports (header media, body variables, URL button suffixes). Authentication-category templates (one-time codes) are sent the same way — supply the code as the body/button variable in `components`.

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/template \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "template_name": "order_shipped",
    "language_code": "en_US",
    "components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "Priya" },
          { "type": "text", "text": "AC-10294" }
        ]
      }
    ]
  }'
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/whatsapp"
headers = {"Authorization": "Bearer cm_your_api_key"}

resp = httpx.post(
    f"{BASE}/messages/template",
    headers=headers,
    json={
        "phone_number_id": "1234567890",
        "to": "+919000000000",
        "template_name": "order_shipped",
        "language_code": "en_US",
        "components": [
            {
                "type": "body",
                "parameters": [
                    {"type": "text", "text": "Priya"},
                    {"type": "text", "text": "AC-10294"},
                ],
            }
        ],
    },
)
print(resp.json()["wamid"])
```
:::

Returns the common send shape (`wamid` + `contacts`). A template send works regardless of the 24-hour window as long as the template is `APPROVED`.

## Send a media message

Send an image, audio clip, video, document, or sticker. Reference the media by a `media_id` (recommended — upload once via [`POST /media`](#upload-media)) or by a public `link` (WhatsApp fetches and caches it for a short time).

`POST /messages/media` · scope `whatsapp:send`

**Request body**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `phone_id` / `phone_number_id` | UUID / string | Yes (one) | The sending number |
| `to` | string (5–20) | Yes | Recipient in E.164 |
| `kind` | enum | Yes | One of `image`, `audio`, `video`, `document`, `sticker` |
| `media_id` | string (≤ 64) | One of | An uploaded media id |
| `link` | string (≤ 2048) | One of | A public URL to the file |
| `caption` | string (≤ 1024) | No | Caption text — honored for `image`, `video`, `document` only |
| `filename` | string (≤ 255) | No | Display filename (documents) |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/media \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "kind": "image",
    "media_id": "1079h3f...meta-media-id",
    "caption": "Your receipt"
  }'
```

Returns the common send shape (`wamid` + `contacts`).

## Send an interactive message

Send reply buttons, a list menu, or a call-to-action URL button. Select the variant with `interactive_type`.

`POST /messages/interactive` · scope `whatsapp:send`

**Common fields**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `phone_id` / `phone_number_id` | UUID / string | Yes (one) | The sending number |
| `to` | string (5–20) | Yes | Recipient in E.164 |
| `interactive_type` | enum | Yes | One of `button`, `list`, `cta_url` |
| `body_text` | string (1–1024) | Yes | The main message body |
| `footer_text` | string (≤ 60) | No | Small footer line |
| `header` | object | No | Optional header (text/image/video/document) |

**Variant-specific fields**

| `interactive_type` | Required fields | Notes |
|--------------------|-----------------|-------|
| `button` | `buttons` | Up to 3 reply buttons, each `{ id, title }` (title ≤ 20 chars) |
| `list` | `button_text`, `sections` | `button_text` (≤ 20) opens the list; each section is `{ title, rows: [{ id, title, description? }] }` |
| `cta_url` | `button_text`, `button_url` | Renders a button that opens `button_url` |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/interactive \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "interactive_type": "button",
    "body_text": "Would you like to continue?",
    "buttons": [
      { "id": "yes", "title": "Yes, continue" },
      { "id": "no", "title": "No, thanks" }
    ]
  }'
```

Returns the common send shape (`wamid` + `contacts`). The user's tap arrives back as an inbound interactive reply on your webhook.

## Send a location

Send a static location pin.

`POST /messages/location` · scope `whatsapp:send`

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `phone_id` / `phone_number_id` | UUID / string | Yes (one) | The sending number |
| `to` | string (5–20) | Yes | Recipient in E.164 |
| `latitude` | number (−90 to 90) | Yes | |
| `longitude` | number (−180 to 180) | Yes | |
| `name` | string (≤ 200) | No | Location label |
| `address` | string (≤ 300) | No | Street address shown under the name |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/location \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "latitude": 19.076,
    "longitude": 72.8777,
    "name": "Our Mumbai office",
    "address": "MG Road, Mumbai"
  }'
```

## Send a reaction

React to an inbound message with an emoji, or remove a reaction by sending an empty string.

`POST /messages/reaction` · scope `whatsapp:send`

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `phone_id` / `phone_number_id` | UUID / string | Yes (one) | The sending number |
| `to` | string (5–20) | Yes | Recipient in E.164 |
| `message_id` | string (1–128) | Yes | The `wamid` of the inbound message to react to |
| `emoji` | string (≤ 8) | No | The emoji; empty string removes an existing reaction |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/reaction \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "message_id": "wamid.HBgMOTE5...==",
    "emoji": "👍"
  }'
```

## Send contact cards

Send one or more contact cards (WhatsApp's `contacts` message type).

`POST /messages/contacts` · scope `whatsapp:send`

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `phone_id` / `phone_number_id` | UUID / string | Yes (one) | The sending number |
| `to` | string (5–20) | Yes | Recipient in E.164 |
| `contacts` | array of objects (1–10) | Yes | WhatsApp contact objects — each needs a `name` block with at least `formatted_name` or `first_name` + `last_name` |
| `context_message_id` | string (≤ 128) | No | `wamid` of the inbound message this is a reply to |
| `biz_opaque_callback_data` | string (≤ 256) | No | Opaque string echoed back on status events for correlation |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/contacts \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "contacts": [
      {
        "name": { "formatted_name": "Acme Support", "first_name": "Acme" },
        "phones": [ { "phone": "+918080247309", "type": "WORK" } ]
      }
    ]
  }'
```

## Mark a message as read

Mark an inbound message as read (blue ticks) and optionally show a typing indicator.

`POST /messages/{message_id}/read` · scope `whatsapp:send`

`{message_id}` is the `wamid` of the inbound message. The body carries the sending-number selector plus an optional typing flag.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `phone_id` / `phone_number_id` | UUID / string | Yes (one) | The sending number |
| `typing_indicator` | boolean | No | Show a typing bubble to the user (default `false`) |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/wamid.HBgM...==/read \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "phone_number_id": "1234567890", "typing_indicator": true }'
```

**Response (200 OK):**

```json
{ "success": true }
```

## Media

### Upload media

Upload a file and get back a reusable `media_id` (valid for 30 days). Send it as a multipart form; the MIME type is taken from the file's `Content-Type`.

`POST /media` · scope `whatsapp:write`

**Form fields**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `file` | file | Yes | The media file to upload |
| `phone_id` / `phone_number_id` | UUID / string | Yes (one) | The number the media is scoped to |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/media \
  -H "Authorization: Bearer cm_your_api_key" \
  -F 'phone_number_id=1234567890' \
  -F 'file=@receipt.jpg;type=image/jpeg'
```

**Response (200 OK):**

```json
{
  "media_id": "1079h3f...meta-media-id",
  "mime_type": "image/jpeg",
  "size_bytes": 84213
}
```

Pass the `media_id` to [`POST /messages/media`](#send-a-media-message).

### Resolve a media id to a download URL

Resolve a `media_id` to a temporary download URL. Most useful for **inbound** media — the `media.id` inside an incoming message event.

`GET /media/{media_id}` · scope `whatsapp:read`

**Query parameters**

| Param | Type | Required | Notes |
|-------|------|----------|-------|
| `phone_id` / `phone_number_id` | UUID / string | Yes (one) | The owning number |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/media/1079h3f...?phone_number_id=1234567890" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK):**

```json
{
  "url": "https://lookaside.fbsbx.com/whatsapp_business/attachments/...",
  "mime_type": "image/jpeg",
  "sha256": "b1946ac92492d2347c6235b4d2611184...",
  "file_size": 84213
}
```

The `url` is short-lived — fetch it immediately.

### Stream inbound media bytes

Stream the raw bytes of a media asset back through CallMissed, so browser and mobile clients can render inbound images inline without handling short-lived upstream URLs. Responds with the file bytes and the upstream content type.

`GET /media/{media_id}/content` · scope `whatsapp:read`

**Query parameters**

| Param | Type | Required | Notes |
|-------|------|----------|-------|
| `phone_id` / `phone_number_id` | UUID / string | Yes (one) | The owning number |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/media/1079h3f.../content?phone_number_id=1234567890" \
  -H "Authorization: Bearer cm_your_api_key" \
  --output inbound-image.jpg
```

## Manage templates

Message templates are created on and approved by WhatsApp, then mirrored locally. A template transitions through `PENDING` → `APPROVED` / `REJECTED` via status events; only `APPROVED` templates can be sent.

### Create a template

`POST /templates` · scope `whatsapp:write`

**Request body**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `account_id` / `waba_id` | UUID / string | Yes (one) | The WABA to create the template under |
| `name` | string (1–512) | Yes | Template name (lowercase, underscores) |
| `category` | string | Yes | One of `MARKETING`, `UTILITY`, `AUTHENTICATION` |
| `language` | string (2–12) | Yes | Template locale, e.g. `en_US`, `hi_IN` |
| `components` | array of objects | Yes | Header/body/footer/buttons spec — passed through to WhatsApp |
| `parameter_format` | string | No | `POSITIONAL` or `NAMED` — variable syntax |
| `allow_category_change` | boolean | No | Let WhatsApp re-categorize the template (default on) |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/templates \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "name": "order_shipped",
    "category": "UTILITY",
    "language": "en_US",
    "components": [
      {
        "type": "BODY",
        "text": "Hi {{1}}, your order {{2}} has shipped.",
        "example": { "body_text": [["Priya", "AC-10294"]] }
      }
    ]
  }'
```

**Response (200 OK):**

```json
{
  "template_id": "1234567890123456",
  "status": "PENDING",
  "template": {
    "id": "3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e",
    "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
    "template_id": "1234567890123456",
    "name": "order_shipped",
    "language": "en_US",
    "category": "UTILITY",
    "status": "PENDING",
    "quality_score": "UNKNOWN",
    "rejection_reason": null,
    "components": [],
    "last_meta_synced_at": null,
    "created_at": "2026-04-19T12:00:00Z",
    "updated_at": "2026-04-19T12:00:00Z"
  }
}
```

> **Authentication templates.** Use `category: "AUTHENTICATION"` for one-time passcode / verification templates. Once approved, send the code with [`POST /messages/template`](#send-a-template-message), passing the code as the body/button variable in `components`.

### List templates

`GET /templates` · scope `whatsapp:read`

Returns templates from the local mirror, newest first.

**Query parameters**

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `account_id` | UUID | — | Filter to one connected account |
| `waba_id` | string (≤ 64) | — | Filter by Meta WABA id |
| `status` | string | — | Filter by status, e.g. `APPROVED`, `PENDING`, `REJECTED` |
| `category` | string | — | `MARKETING` / `UTILITY` / `AUTHENTICATION` |
| `language` | string (≤ 12) | — | Filter by locale |
| `limit` | integer (1–500) | 100 | Max rows |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/templates?status=APPROVED&limit=100" \
  -H "Authorization: Bearer cm_your_api_key"
```

### Get one template

`GET /templates/{template_uuid}` · scope `whatsapp:read`

Fetch a single template by its CallMissed UUID. Returns a `404` if it does not belong to your workspace.

```bash
curl https://api.callmissed.com/api/v1/whatsapp/templates/{template_uuid} \
  -H "Authorization: Bearer cm_your_api_key"
```

### Delete a template

`DELETE /templates/{template_uuid}` · scope `whatsapp:write`

Deletes the template on WhatsApp and drops the local row. Returns `204 No Content`.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/whatsapp/templates/{template_uuid} \
  -H "Authorization: Bearer cm_your_api_key"
```

> Deleting an `APPROVED` template starts a 30-day cooldown before the same **name** can be reused.

### Sync templates

`POST /templates/sync` · scope `whatsapp:write`

Pull every template for a WABA from WhatsApp and upsert the local mirror. A background sweep does this hourly; call this endpoint for an immediate refresh after editing templates in WhatsApp Manager.

**Request body**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `account_id` / `waba_id` | UUID / string | Yes (one) | The WABA to reconcile |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/templates/sync \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "waba_id": "102290129340398" }'
```

**Response (200 OK):**

```json
{ "waba_id": "102290129340398", "fetched": 12, "inserted": 2, "updated": 10 }
```

## Accounts & phone numbers

Read the WhatsApp Business Accounts and phone numbers connected to your workspace. Both require `whatsapp:read`.

| Endpoint | Purpose |
|----------|---------|
| `GET /accounts` | List connected WhatsApp Business Accounts |
| `GET /phone_numbers` | List connected phone numbers (with quality, messaging limit, registration status) |
| `GET /phone_numbers/{phone_id}` | Get one phone number |
| `GET /webhook_events` | Recent inbound webhook events for your WABAs (audit peek) |

```bash
# List connected phone numbers
curl https://api.callmissed.com/api/v1/whatsapp/phone_numbers \
  -H "Authorization: Bearer cm_your_api_key"
```

**`GET /phone_numbers` response (200 OK)** — an array of numbers:

```json
[
  {
    "id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
    "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
    "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
    "phone_number_id": "1234567890",
    "display_phone_number": "+91 80802 47309",
    "verified_name": "Acme Support",
    "code_verification_status": "VERIFIED",
    "quality_rating": "GREEN",
    "messaging_limit_tier": "TIER_1K",
    "registration_status": "REGISTERED",
    "name_status": "APPROVED",
    "ai_autoreply_enabled": true,
    "is_active": true,
    "created_at": "2026-04-19T12:00:00Z"
  }
]
```

Use `id` (the `phone_id`) or `phone_number_id` from this list as the sending-number selector on any send endpoint.

`GET /webhook_events` accepts `limit` (1–100, default 20) and returns a slim projection — id, `received_at`, `signature_valid`, `event_type`, `waba_id`, `processed`, plus a best-effort `sender_wa_id` and `body_preview`. Full raw payloads are not surfaced here.

## Receiving inbound messages

Inbound WhatsApp events — a user's reply, a delivery/read status, a button tap — are delivered to **your** configured endpoints as CallMissed webhook events. Subscribe to `message.received`, `message.sent`, `conversation.started`, and `conversation.ended` (and verify the `X-CallMissed-Signature` HMAC header) exactly as documented on the [Webhooks](/docs/webhooks) page. You do not poll for inbound messages — configure a webhook and receive them in real time.

## Connecting a number

Sending requires a connected WABA. The simplest path is the dashboard: **Settings → Integrations → WhatsApp** walks you through connecting a Meta WhatsApp Business number and registering it. See [WhatsApp Business Setup](/docs/whatsapp-setup) for the full walkthrough, and the [WhatsApp Bot](/docs/whatsapp) guide for wiring an AI bot to auto-reply on the number.
