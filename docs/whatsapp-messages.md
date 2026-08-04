---
title: "Sending Messages"
description: "Every WhatsApp send endpoint: text, template, media, interactive, location, reaction, contact cards, read receipts, and media upload and download."
slug: "whatsapp-messages"
breadcrumb: "WhatsApp"
---

# Sending Messages

Every WhatsApp send endpoint: text, template, media, interactive, location, reaction, contact cards, read receipts, and media upload and download.

Every send endpoint lives under `https://api.callmissed.com/api/v1/whatsapp`, takes a JSON body, needs the `whatsapp:send` scope (media upload needs `whatsapp:write`, media reads need `whatsapp:read`), and identifies the sending number with `phone_id` or `phone_number_id`. See [WhatsApp API](/docs/whatsapp-api#choosing-the-sending-number) for the selector and the shared error shape.

> **Free-form sends need an open window.** Text, media, interactive and location sends only work inside the 24-hour customer service window that opens when the customer last messaged you. Outside it, send an approved [template](/docs/whatsapp-templates). A closed-window send returns `422`.

## The common response

Every send endpoint returns the same object:

```json
{
  "wamid": "wamid.HBgMOTE5MDAwMDAwMDAwFQIAERgSMkE5N0Y4RDcxMkYzQTJEMQA=",
  "wamids": ["wamid.HBgMOTE5MDAwMDAwMDAwFQIAERgSMkE5N0Y4RDcxMkYzQTJEMQA="],
  "contacts": [
    { "input": "+919000000000", "wa_id": "919000000000" }
  ]
}
```

| Field | Type | Notes |
|---|---|---|
| `wamid` | string | Meta's id for the first message. Keep it to correlate delivery and read status |
| `wamids` | array of strings | Every id the request produced, in send order. More than one when a long text was split across messages |
| `contacts` | array | WhatsApp's resolution of the recipient. Empty when WhatsApp returns none |

Track delivery against `wamids`, not `wamid`: WhatsApp reports status per message, so a split reply produces several status events.

Sends are persisted into the matching conversation thread, so anything you send over the API shows up in the dashboard inbox alongside the agent's own replies. Reactions are the exception, since a reaction is a property of the message it targets rather than a bubble of its own.

## Before WhatsApp is called

Two gates run on every send, before any request reaches Meta.

**Tenant scope.** The sending number is resolved against your workspace. A number you do not own returns `404`, identically to one that does not exist.

**Credit check.** The charge for a WhatsApp message lands after Meta delivers it, so a send you cannot pay for cannot be undone. Sends are therefore priced up front, against the same rate card the delivery charge uses, and refused with `402` when the balance will not cover them. Nothing is sent and nothing is charged.

```json
{
  "detail": "Not enough credits to send this message, so nothing was sent and nothing was charged. It needs at least 7.51 credits and you have 2.00 spendable (balance 12.00, 10.00 held for running campaigns) -- short by 5.51. Top up your balance and try again."
}
```

Template sends are priced from the recipient's region and the template's category, which differ by more than tenfold across markets, so the quoted figure is specific to the message you tried to send. Reactions are not credit-gated, because they are not billed.

## Send a text message

`POST /api/v1/whatsapp/messages` · scope `whatsapp:send`

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_id` | UUID | One of | CallMissed's number id |
| `phone_number_id` | string, max 64 | One of | Meta's number id |
| `to` | string, 5 to 20 chars | Yes | Recipient in E.164, for example `+919000000000` |
| `text` | string, 1 to 65536 chars | Yes | Message body. WhatsApp caps a single message at 4096 characters, so a longer body is split across several messages and every id comes back in `wamids` |
| `preview_url` | boolean | No | Render a link preview for the first URL. Default `false` |

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "text": "Your order AC-10294 shipped this morning. Track it at https://acme.example.com/t/AC-10294",
    "preview_url": true
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
        "text": "Your order AC-10294 shipped this morning.",
        "preview_url": False,
    },
)
resp.raise_for_status()
print(resp.json()["wamid"])
```
```javascript [Node]
const res = await fetch("https://api.callmissed.com/api/v1/whatsapp/messages", {
  method: "POST",
  headers: {
    Authorization: "Bearer cm_your_api_key",
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    phone_number_id: "1234567890",
    to: "+919000000000",
    text: "Your order AC-10294 shipped this morning.",
  }),
});
if (!res.ok) throw new Error((await res.json()).detail);
const { wamid } = await res.json();
```
:::

**Failures**

| Code | Meaning |
|---|---|
| `400` | Neither `phone_id` nor `phone_number_id` was supplied |
| `401` | The number's Meta token is invalid or expired. Reconnect the number |
| `402` | Not enough credits, or a workspace budget cap would be exceeded. Nothing was sent |
| `403` | API key is missing `whatsapp:send` |
| `404` | The sending number is not on your workspace |
| `409` | The number is disconnected, or is not registered on the WhatsApp Business Platform |
| `422` | The 24-hour window is closed, the display name is not approved yet, or WhatsApp rejected the payload |
| `429` | Per-user-pair send rate limit. Retry with backoff |

## Send a template message

`POST /api/v1/whatsapp/messages/template` · scope `whatsapp:send`

The only way to message someone outside the 24-hour window. The template must already be `APPROVED`.

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The sending number |
| `to` | string, 5 to 20 chars | Yes | Recipient in E.164 |
| `template_name` | string, 1 to 512 chars | Yes | The approved template's name |
| `language_code` | string, max 12 | No | Template locale. Default `en_US` |
| `components` | array of objects | No | Header, body and button variable values, passed to WhatsApp unchanged |

`components` follows WhatsApp's own shape, so any combination WhatsApp supports works: header media, body variables, URL button suffixes. Authentication templates (one-time codes) are sent the same way, with the code as a body or button parameter.

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
      },
      {
        "type": "button",
        "sub_type": "url",
        "index": "0",
        "parameters": [{ "type": "text", "text": "AC-10294" }]
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
resp.raise_for_status()
print(resp.json()["wamid"])
```
:::

Returns the common send response. `422` if the template name or locale does not resolve to an approved template on the WABA, and `402` if the priced send exceeds your spendable balance.

## Send media

`POST /api/v1/whatsapp/messages/media` · scope `whatsapp:send`

Reference the file by an uploaded `media_id` (recommended, reusable for 30 days) or by a public `link` that WhatsApp fetches and caches briefly.

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The sending number |
| `to` | string, 5 to 20 chars | Yes | Recipient in E.164 |
| `kind` | enum | Yes | `image`, `audio`, `video`, `document` or `sticker` |
| `media_id` | string, max 64 | One of | From [upload media](#upload-media) |
| `link` | string, max 2048 | One of | A public URL to the file |
| `caption` | string, max 1024 | No | Honoured for `image`, `video` and `document` only, ignored otherwise |
| `filename` | string, max 255 | No | Display filename for documents |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/media \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "kind": "document",
    "media_id": "1079235482913746",
    "filename": "invoice-AC-10294.pdf",
    "caption": "Your invoice"
  }'
```

**Failures**

| Code | Meaning |
|---|---|
| `400` | The MIME type does not match the file. Check the extension and `Content-Type` |
| `413` | The file exceeds 100 MB |
| `422` | The 24-hour window is closed, or WhatsApp rejected the media |

## Send an interactive message

`POST /api/v1/whatsapp/messages/interactive` · scope `whatsapp:send`

Reply buttons, a list menu, or a call-to-action URL button. `interactive_type` selects the variant.

**Common fields**

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The sending number |
| `to` | string, 5 to 20 chars | Yes | Recipient in E.164 |
| `interactive_type` | enum | Yes | `button`, `list` or `cta_url` |
| `body_text` | string, 1 to 1024 chars | Yes | The main message body |
| `footer_text` | string, max 60 | No | Small footer line |
| `header` | object | No | Header block, passed through to WhatsApp. For `list` only its `text` is used |

**Variant fields**

| `interactive_type` | Required | Shape |
|---|---|---|
| `button` | `buttons` | 1 to 3 objects, each `{ "id": string (1-256), "title": string (1-20) }` |
| `list` | `button_text`, `sections` | `button_text` max 20. Each section is `{ "title": string (1-24), "rows": [{ "id": string (1-200), "title": string (1-24), "description"?: string (max 72) }] }`, at least one row per section |
| `cta_url` | `button_text`, `button_url` | `button_url` max 2048 |

:::tabs
```bash [Buttons]
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/interactive \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "interactive_type": "button",
    "body_text": "Your order is out for delivery. Is someone home to receive it?",
    "footer_text": "Acme Coffee",
    "buttons": [
      { "id": "home_yes", "title": "Yes, deliver" },
      { "id": "home_no", "title": "Reschedule" }
    ]
  }'
```
```bash [List]
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/interactive \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "interactive_type": "list",
    "body_text": "What would you like help with?",
    "button_text": "Pick a topic",
    "sections": [
      {
        "title": "Orders",
        "rows": [
          { "id": "track", "title": "Track an order", "description": "Live delivery status" },
          { "id": "return", "title": "Start a return" }
        ]
      },
      {
        "title": "Account",
        "rows": [{ "id": "invoice", "title": "Get an invoice" }]
      }
    ]
  }'
```
```bash [CTA URL]
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/interactive \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "interactive_type": "cta_url",
    "body_text": "Your invoice is ready.",
    "button_text": "View invoice",
    "button_url": "https://acme.example.com/invoices/AC-10294"
  }'
```
:::

Returns the common send response. The customer's tap arrives back on your webhook as an inbound message with `type: "interactive"` or `type: "button"`.

**Failures**

| Code | Meaning |
|---|---|
| `400` | `buttons` missing for `button`, or `button_text` and `sections` missing for `list`, or `button_text` and `button_url` missing for `cta_url` |
| `422` | The 24-hour window is closed, or WhatsApp rejected the layout |

## Send a location

`POST /api/v1/whatsapp/messages/location` · scope `whatsapp:send`

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The sending number |
| `to` | string, 5 to 20 chars | Yes | Recipient in E.164 |
| `latitude` | float, -90 to 90 | Yes | |
| `longitude` | float, -180 to 180 | Yes | |
| `name` | string, max 200 | No | Location label |
| `address` | string, max 300 | No | Street address shown under the name |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/location \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "latitude": 19.076,
    "longitude": 72.8777,
    "name": "Acme Coffee Bandra",
    "address": "Linking Road, Bandra West, Mumbai 400050"
  }'
```

## Send a reaction

`POST /api/v1/whatsapp/messages/reaction` · scope `whatsapp:send`

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The sending number |
| `to` | string, 5 to 20 chars | Yes | Recipient in E.164 |
| `message_id` | string, 1 to 128 chars | Yes | The `wamid` of the message to react to |
| `emoji` | string, max 8 | No | The emoji. An empty string removes an existing reaction. Default empty |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/reaction \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "message_id": "wamid.HBgMOTE5MDAwMDAwMDAwFQIAEhggQjc0RTI5RDNBMjJDNjE4RgA=",
    "emoji": "👍"
  }'
```

## Send contact cards

`POST /api/v1/whatsapp/messages/contacts` · scope `whatsapp:send`

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The sending number |
| `to` | string, 5 to 20 chars | Yes | Recipient in E.164 |
| `contacts` | array of objects, 1 to 10 | Yes | WhatsApp contact objects. Each needs a `name` block with at least `formatted_name`, or `first_name` plus `last_name` |
| `context_message_id` | string, max 128 | No | `wamid` of the inbound message this replies to |
| `biz_opaque_callback_data` | string, max 256 | No | Opaque string echoed back on status events for your own correlation |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages/contacts \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "contacts": [
      {
        "name": { "formatted_name": "Acme Support", "first_name": "Acme", "last_name": "Support" },
        "phones": [{ "phone": "+918080247309", "type": "WORK", "wa_id": "918080247309" }],
        "emails": [{ "email": "support@acme.example.com", "type": "WORK" }]
      }
    ],
    "biz_opaque_callback_data": "escalation-4471"
  }'
```

## Mark a message as read

`POST /api/v1/whatsapp/messages/{message_id}/read` · scope `whatsapp:send`

`{message_id}` is the `wamid` of the inbound message. Shows blue ticks, and optionally a typing bubble while you prepare a reply.

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The sending number |
| `typing_indicator` | boolean | No | Show a typing bubble. Default `false` |

```bash
curl -X POST "https://api.callmissed.com/api/v1/whatsapp/messages/wamid.HBgMOTE5MDAwMDAwMDAwFQIAEhggQjc0RTI5RDNBMjJDNjE4RgA=/read" \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "phone_number_id": "1234567890", "typing_indicator": true }'
```

**Response (200 OK)**

```json
{ "success": true }
```

The agent does this automatically for messages it answers.

## Media

### Upload media

`POST /api/v1/whatsapp/media` · scope `whatsapp:write`

Multipart upload. The MIME type is read from the file part's `Content-Type`, so set it explicitly, and it is validated against WhatsApp's allowlist before the request reaches Meta. The returned `media_id` is reusable for 30 days and is scoped to the number you uploaded it against. Maximum upload size is 100 MB, and this path has a tighter rate limit than the rest of the API at 60 requests per minute.

| Form field | Type | Required | Notes |
|---|---|---|---|
| `file` | file | Yes | The media file. Must carry a `Content-Type` |
| `phone_id` | UUID | One of | CallMissed's number id |
| `phone_number_id` | string | One of | Meta's number id |

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/whatsapp/media \
  -H "Authorization: Bearer cm_your_api_key" \
  -F 'phone_number_id=1234567890' \
  -F 'file=@invoice-AC-10294.pdf;type=application/pdf'
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/whatsapp"
headers = {"Authorization": "Bearer cm_your_api_key"}

with open("invoice-AC-10294.pdf", "rb") as fh:
    resp = httpx.post(
        f"{BASE}/media",
        headers=headers,
        data={"phone_number_id": "1234567890"},
        files={"file": ("invoice-AC-10294.pdf", fh, "application/pdf")},
    )
resp.raise_for_status()
media_id = resp.json()["media_id"]
```
:::

**Response (200 OK)**

```json
{
  "media_id": "1079235482913746",
  "mime_type": "application/pdf",
  "size_bytes": 84213
}
```

Pass `media_id` to [send media](#send-media).

**Failures**

| Code | Meaning |
|---|---|
| `400` | No `Content-Type` on the file part, or the MIME type does not match the bytes |
| `413` | The file exceeds 100 MB |

### Resolve inbound media to a URL

`GET /api/v1/whatsapp/media/{media_id}` · scope `whatsapp:read`

Turns a media id into a temporary download URL. Mostly used for **inbound** media, where the webhook gives you a media id and you want the file. The URL is valid for about five minutes and requires WhatsApp's own auth, so fetch it immediately or use [the content proxy](#stream-inbound-media-bytes) instead.

| Query param | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The owning number |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/media/1079235482913746?phone_number_id=1234567890" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "url": "https://lookaside.fbsbx.com/whatsapp_business/attachments/?mid=1079235482913746",
  "mime_type": "image/jpeg",
  "sha256": "b1946ac92492d2347c6235b4d2611184a3e0f5b1c2d3e4f5a6b7c8d9e0f1a2b3",
  "file_size": 84213
}
```

| Field | Type | Notes |
|---|---|---|
| `url` | string | Short-lived download URL |
| `mime_type` | string | Falls back to `application/octet-stream` |
| `sha256` | string, nullable | Checksum, when WhatsApp provides one |
| `file_size` | integer, nullable | Bytes, when WhatsApp provides it |

### Stream inbound media bytes

`GET /api/v1/whatsapp/media/{media_id}/content` · scope `whatsapp:read`

Streams the raw file back through CallMissed with the upstream content type, so your browser or mobile client can render inbound images without handling short-lived URLs or WhatsApp credentials. Responses carry `Cache-Control: private, max-age=300`.

| Query param | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The owning number |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/media/1079235482913746/content?phone_number_id=1234567890" \
  -H "Authorization: Bearer cm_your_api_key" \
  --output inbound-image.jpg
```

Returns the file bytes on `200`, or `404` when the media id no longer resolves.
