---
title: "Campaigns"
description: "Bulk template sends: create a campaign, upload recipients with per-recipient variables, launch it, and track delivery."
slug: "whatsapp-campaigns"
breadcrumb: "WhatsApp"
---

# Campaigns

Bulk template sends: create a campaign, upload recipients with per-recipient variables, launch it, and track delivery.

A campaign sends one approved template to many recipients, each with their own variable values, throttled so WhatsApp does not rate-limit you. It is the right tool for an order-status blast, a restock notice or a renewal reminder. For a single send, use [`POST /messages/template`](/docs/whatsapp-messages#send-a-template-message) instead.

All endpoints are under `https://api.callmissed.com/api/v1/whatsapp`.

## Lifecycle

:::flow
icon:gateway | Create | `POST /campaigns` returns a campaign in `draft`
icon:user | Add recipients | `POST /campaigns/{id}/recipients` in batches of up to 10,000
icon:llm | Launch | `POST /campaigns/{id}/launch` prices the list, holds the credits, and starts the worker
icon:done | Track | `GET /campaigns/{id}` returns live counters and a recipient sample
:::

**Campaign statuses:** `draft`, `scheduled`, `running`, `completed`, `cancelled`, `failed`.
**Recipient statuses:** `pending`, `sent`, `delivered`, `read`, `failed`, `skipped`.

Recipients can only be added while the campaign is `draft`. Once it is `running` the worker is already claiming rows.

## Create a campaign

`POST /api/v1/whatsapp/campaigns` · scope `whatsapp:write`

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_number_id` | UUID | Yes | The **CallMissed** phone id (the `id` from `GET /phone_numbers`), not Meta's id |
| `name` | string, 1 to 255 chars | Yes | Your label for the campaign |
| `template_name` | string, 1 to 255 chars | Yes | An approved template's name |
| `template_language` | string, 2 to 16 chars | No | Template locale. Default `en` |
| `template_components` | array of objects | No | The component **shape**, with `{{N}}` placeholders left in. Default empty |
| `scheduled_at` | datetime | No | When you intend to run it. Recorded on the row; launching is still an explicit call |

`template_components` is a shape, not a finished payload. Leave the `{{1}}`, `{{2}}` tokens in the parameter text and the worker substitutes each recipient's `variables` before sending. Non-text parameters, such as a header image, are passed through untouched.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/campaigns \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
    "name": "April restock notice",
    "template_name": "back_in_stock",
    "template_language": "en_US",
    "template_components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "{{1}}" },
          { "type": "text", "text": "{{2}}" }
        ]
      }
    ]
  }'
```

**Response (201 Created)**

```json
{
  "id": "6d1e8b3a-2c4f-4a5b-8e9d-0f1a2b3c4d5e",
  "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "phone_number_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "name": "April restock notice",
  "template_name": "back_in_stock",
  "template_language": "en_US",
  "status": "draft",
  "scheduled_at": null,
  "started_at": null,
  "completed_at": null,
  "total": 0,
  "sent": 0,
  "delivered": 0,
  "read": 0,
  "failed": 0,
  "created_at": "2026-04-19T12:00:00Z"
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | The campaign id |
| `account_id` | UUID | The WABA it sends from |
| `phone_number_id` | UUID | The sending number |
| `status` | string | Campaign status |
| `scheduled_at` / `started_at` / `completed_at` | datetime, nullable | Timestamps, ISO 8601 UTC |
| `total` | integer | Recipients added |
| `sent` / `delivered` / `read` / `failed` | integer | Live counters, updated by the worker and by delivery webhooks |

`404` with `"phone_number_id not found"` if the number is not on your workspace.

## Add recipients

`POST /api/v1/whatsapp/campaigns/{campaign_id}/recipients` · scope `whatsapp:write`

Up to 10,000 per call. Paginate for larger lists.

| Field | Type | Required | Notes |
|---|---|---|---|
| `recipients` | array, max 10000 | Yes | The batch |
| `recipients[].to_phone` | string, 8 to 32 chars | Yes | Any format. Non-digits are stripped, and the result must be 8 to 15 digits |
| `recipients[].variables` | object of string to string | No | Values keyed by placeholder number, so `{"1": "Priya"}` fills `{{1}}` |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/campaigns/6d1e8b3a-2c4f-4a5b-8e9d-0f1a2b3c4d5e/recipients \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "recipients": [
      { "to_phone": "+91 90000 00000", "variables": { "1": "Priya", "2": "Ethiopia Guji" } },
      { "to_phone": "919000000001", "variables": { "1": "Arun", "2": "Colombia Huila" } },
      { "to_phone": "12", "variables": { "1": "Broken" } }
    ]
  }'
```

**Response (200 OK)**

```json
{
  "inserted": 2,
  "skipped_invalid": 1,
  "skipped_duplicate": 0,
  "total_now": 2
}
```

| Field | Type | Notes |
|---|---|---|
| `inserted` | integer | Recipients added |
| `skipped_invalid` | integer | Numbers that were not 8 to 15 digits after stripping |
| `skipped_duplicate` | integer | Numbers already on the campaign, or repeated inside the batch |
| `total_now` | integer | The campaign's recipient total after this call |

Bad rows are counted and skipped rather than failing the batch, so a 10,000-row paste with a few broken cells still lands the good ones. Adding to a campaign that is not `draft` returns `409` with `Cannot add recipients to a campaign in status=running`.

## Launch

`POST /api/v1/whatsapp/campaigns/{campaign_id}/launch` · scope `whatsapp:write`

No body. Flips the campaign to `running` and starts the send worker.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/campaigns/6d1e8b3a-2c4f-4a5b-8e9d-0f1a2b3c4d5e/launch \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns the campaign object with `status: "running"` and `started_at` set.

### The credit hold

Before anything is sent, the whole pending recipient list is priced against the real rate card, per recipient, using the template's category and each number's region. Those credits are then **held**, so a campaign launched a second later cannot spend them.

If the balance will not cover it the launch is refused with `402` and the campaign stays `draft`, retryable after a top-up. Nothing was sent and nothing was charged.

```json
{
  "detail": "Not enough credits to launch this campaign. It needs about 8631.40 credits for 1200 recipients and you are short by 431.40. Top up your balance and try again."
}
```

Pricing varies by more than tenfold across markets, so a mixed India and Germany list is priced per recipient rather than at a blended rate. If the campaign's template has not been synced locally, it is priced as `MARKETING`, the most expensive category, so a campaign can never start underfunded.

**Failures**

| Code | Meaning |
|---|---|
| `400` | The campaign has no pending recipients to send to |
| `402` | Not enough credits for the priced recipient list. The campaign stays `draft` |
| `404` | No such campaign on your workspace |
| `409` | The campaign is not `draft` or `scheduled`, for example it is already `running` |

Concurrent launch calls are serialised, so a double-click cannot start two workers and double-send.

## Cancel

`POST /api/v1/whatsapp/campaigns/{campaign_id}/cancel` · scope `whatsapp:write`

No body. Works from `draft`, `scheduled` or `running`. A running worker notices within one send, so a few in-flight messages may still go out.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/campaigns/6d1e8b3a-2c4f-4a5b-8e9d-0f1a2b3c4d5e/cancel \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns the campaign object with `status: "cancelled"` and `completed_at` set. `409` from any other status, for example one already `completed`.

## List campaigns

`GET /api/v1/whatsapp/campaigns` · scope `whatsapp:read`

| Param | Type | Default | Notes |
|---|---|---|---|
| `limit` | integer, 1 to 100 | 50 | Max rows, newest first |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/campaigns?limit=50" \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns an array of campaign objects.

## Get one campaign

`GET /api/v1/whatsapp/campaigns/{campaign_id}` · scope `whatsapp:read`

The campaign object plus a sample of up to 50 recent recipient rows, most recently updated first. This is the progress endpoint: poll it while a campaign runs.

```bash
curl https://api.callmissed.com/api/v1/whatsapp/campaigns/6d1e8b3a-2c4f-4a5b-8e9d-0f1a2b3c4d5e \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "id": "6d1e8b3a-2c4f-4a5b-8e9d-0f1a2b3c4d5e",
  "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "phone_number_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "name": "April restock notice",
  "template_name": "back_in_stock",
  "template_language": "en_US",
  "status": "running",
  "scheduled_at": null,
  "started_at": "2026-04-19T12:05:02Z",
  "completed_at": null,
  "total": 1200,
  "sent": 418,
  "delivered": 402,
  "read": 191,
  "failed": 3,
  "created_at": "2026-04-19T12:00:00Z",
  "recipients_sample": [
    {
      "id": "aa11bb22-cc33-4d44-8e55-6f7788990011",
      "to_phone": "919000000000",
      "status": "delivered",
      "wamid": "wamid.HBgMOTE5MDAwMDAwMDAwFQIAERgSN0MyRDFBOEY0RTVCOTAxMgA=",
      "error": null,
      "sent_at": "2026-04-19T12:05:44Z",
      "last_status_at": "2026-04-19T12:05:51Z"
    },
    {
      "id": "bb22cc33-dd44-4e55-9f66-7788990011aa",
      "to_phone": "919000000002",
      "status": "failed",
      "wamid": null,
      "error": "Request rejected by Meta - check the recipient and payload.",
      "sent_at": null,
      "last_status_at": "2026-04-19T12:05:47Z"
    }
  ]
}
```

| Field | Type | Notes |
|---|---|---|
| `recipients_sample[].id` | UUID | Recipient row id |
| `recipients_sample[].to_phone` | string | Normalised to digits only |
| `recipients_sample[].status` | string | `pending`, `sent`, `delivered`, `read`, `failed` or `skipped` |
| `recipients_sample[].wamid` | string, nullable | Meta's message id once sent |
| `recipients_sample[].error` | string, nullable | Why this recipient failed |
| `recipients_sample[].sent_at` | datetime, nullable | When the send left |
| `recipients_sample[].last_status_at` | datetime | Last status change |

The sample is capped at 50 rows and is not paginated. Use the counters on the campaign itself for totals.
