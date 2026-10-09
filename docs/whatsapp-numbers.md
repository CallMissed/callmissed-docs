---
title: "Number Management"
description: "Check account and number health, register or deregister a number, verify it, rotate its two-step PIN, edit its business profile, block users, and manage QR codes and short links."
slug: "whatsapp-numbers"
breadcrumb: "WhatsApp"
---

# Number Management

Check account and number health, register or deregister a number, verify it, rotate its two-step PIN, edit its business profile, block users, and manage QR codes and short links.

These endpoints manage a WhatsApp number **after** it is connected. Connecting, listing, linking a bot, pausing auto-reply, disconnecting and deleting are on [WhatsApp API](/docs/whatsapp-api#phone-numbers); ice breakers and commands are under [Conversational automation](/docs/whatsapp-api#conversational-automation).

**Base path:** `/api/v1/whatsapp`

Every path below takes CallMissed's ids, never Meta's: `{account_id}` is the `id` from `GET /accounts` and `{phone_id}` is the `id` from `GET /phone_numbers`. Both are scoped to your workspace, and an id from another workspace returns the same `404` as one that does not exist.

| Area | Endpoints | Scope |
|---|---|---|
| [Health](#health) | Account health, refresh, number health | `whatsapp:read`; refresh needs `whatsapp:write` |
| [Registration](#registration-and-verification) | Register, deregister, request code, verify code, two-step PIN | `whatsapp:write` |
| [Business profile](#business-profile) | Read and update | `whatsapp:read` / `whatsapp:write` |
| [Blocked users](#blocked-users) | List, block, unblock | `whatsapp:read` / `whatsapp:write` |
| [QR codes and short links](#qr-codes-and-short-links) | List, create, get, update, delete | `whatsapp:read` / `whatsapp:write` |

### Errors common to every route on this page

| Code | Meaning |
|---|---|
| `400` | No business token is on file for the number. Reconnect it through Embedded Signup |
| `403` | The API key is missing the scope in the route's signature line |
| `404` | The account, number or QR code is not on your workspace |
| `422` | The body failed validation, or WhatsApp rejected the request. The `detail` is a short, actionable sentence |
| `429` | WhatsApp's rate limit for this number or action was reached. Wait, then retry |
| `500` | The stored credentials for the number could not be read. Reconnect the number |
| `502` | WhatsApp failed. Retry; contact support if it persists |

WhatsApp's raw error text and numeric codes are never echoed back. Each failure carries one of our own sentences in `detail`.

## Health

Health tells you which link in the chain (app, business, WhatsApp Business Account, phone number) is blocking sends, and why. The two `GET` routes return the **last stored** verdict without calling WhatsApp, so they are cheap to poll. `POST .../health/refresh` re-reads it live.

### Get account health

`GET /api/v1/whatsapp/accounts/{account_id}/health` · scope `whatsapp:read`

```bash
curl https://api.callmissed.com/api/v1/whatsapp/accounts/1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d/health \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "waba_id": "102290129340398",
  "health_status": {
    "can_send_message": "AVAILABLE",
    "entities": [
      { "entity_type": "BUSINESS", "id": "441329482726", "can_send_message": "AVAILABLE" },
      { "entity_type": "WABA", "id": "102290129340398", "can_send_message": "AVAILABLE" }
    ]
  },
  "health_checked_at": "2026-09-30T08:15:00Z",
  "business_verification_status": "verified",
  "account_review_status": "APPROVED",
  "account_status": "ACTIVE",
  "account_restriction_reason": null,
  "payment_setup_complete": true
}
```

| Field | Type | Notes |
|---|---|---|
| `account_id` | UUID | CallMissed's account id |
| `waba_id` | string | Meta's WABA id |
| `health_status` | object, nullable | WhatsApp's health document, passed through as WhatsApp returned it, including the per-entity breakdown and any reason. `null` until the first refresh. New entity types can appear, so do not switch exhaustively on it |
| `health_checked_at` | datetime, nullable | When health was last read successfully. `null` means it has never been read, **not** that the account is healthy |
| `business_verification_status` | string, nullable | Meta business verification. Required before you can create authentication templates. Verification itself is done in Meta Business Manager, not over any API |
| `account_review_status` | string | `PENDING`, `APPROVED` or `REJECTED` |
| `account_status` | string | `ACTIVE`, or a restricted state |
| `account_restriction_reason` | string, nullable | Set when WhatsApp restricts the account. `null` while healthy |
| `payment_setup_complete` | boolean | Whether a payment method is attached to the WABA. Payment methods are added in WhatsApp Manager |

### Refresh account health

`POST /api/v1/whatsapp/accounts/{account_id}/health/refresh` · scope `whatsapp:write`

No body. Re-reads, live from WhatsApp, the account's health, its business verification and review status, and the health of **every number** under the account. Each read is independent: if some succeed and others fail, the successful ones are saved and returned, and `health_checked_at` only moves for what was actually read.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/accounts/1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d/health/refresh \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns the account health object above. `409` if no number on the account has a usable business token (reconnect one), `502` if every read failed.

It needs `whatsapp:write` rather than `whatsapp:read` because it spends the account's WhatsApp rate-limit budget and updates stored state.

### Get number health

`GET /api/v1/whatsapp/phone_numbers/{phone_id}/health` · scope `whatsapp:read`

A number can be blocked while its account is healthy, and the other way round, so it has its own health. Refresh it with the account refresh above.

```bash
curl https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/health \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "phone_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "phone_number_id": "1234567890",
  "display_phone_number": "+91 80802 47309",
  "health_status": { "can_send_message": "AVAILABLE" },
  "health_checked_at": "2026-09-30T08:15:00Z",
  "quality_rating": "GREEN",
  "name_status": "APPROVED",
  "code_verification_status": "VERIFIED",
  "throughput_level": "STANDARD",
  "messaging_limit": null,
  "messaging_limit_tier": "TIER_1K",
  "registration_status": "REGISTERED",
  "registration_error": null,
  "is_active": true
}
```

| Field | Type | Notes |
|---|---|---|
| `phone_id` | UUID | CallMissed's number id |
| `phone_number_id` | string | Meta's phone number id |
| `display_phone_number` | string | The number as customers see it |
| `health_status` | object, nullable | WhatsApp's health document for this number, passed through. `null` until the first refresh |
| `health_checked_at` | datetime, nullable | When it was last read successfully |
| `quality_rating` | string | `GREEN`, `YELLOW`, `RED`, `NA` or `UNKNOWN`. Treat it as an open string |
| `name_status` | string | Display-name approval. Free-form sends only work while this is `APPROVED` or `AVAILABLE_WITHOUT_REVIEW` |
| `code_verification_status` | string | `VERIFIED` once the number has been verified |
| `throughput_level` | string | `STANDARD` today |
| `messaging_limit` | string, nullable | Reserved for WhatsApp's portfolio-level messaging limit, which is shared by every number in the same business portfolio. May be `null`; read `messaging_limit_tier` until it is set |
| `messaging_limit_tier` | string | Business-initiated messaging limit tier, for example `TIER_1K` |
| `registration_status` | string | `PENDING`, `REGISTERED`, `FAILED` or `DEREGISTERED` |
| `registration_error` | string, nullable | Why the last registration failed |
| `is_active` | boolean | `false` once the number is disconnected |

## Registration and verification

### Register a number

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/register` · scope `whatsapp:write`

Registers a connected number on the WhatsApp Cloud API. Use it to retry a registration that failed during onboarding (`registration_status` is `FAILED`), or to re-register after a [deregister](#deregister-a-number).

> **WhatsApp allows 10 registrations per number in any rolling 72 hours.** The 11th attempt locks the number for 72 hours and returns `429`. Call this only from a deliberate user action, never from a retry loop. Deregister counts against the same quota.

If the number already has a two-step verification PIN on file, it is reused, because WhatsApp requires the existing PIN and a wrong one burns quota. Otherwise a new PIN is generated and kept for future re-registrations. To set your own PIN, use [Rotate the two-step PIN](#rotate-the-two-step-pin).

| Field | Type | Required | Notes |
|---|---|---|---|
| `data_localization_region` | string, exactly 2 chars | No | ISO 3166-1 alpha-2 country for data-at-rest residency, from WhatsApp's supported list. It cannot be changed in place: deregister, then register with the new value |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/register \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "data_localization_region": "IN" }'
```

**Response (200 OK)**

```json
{ "ok": true, "error": null }
```

On success `registration_status` becomes `REGISTERED`. On failure it becomes `FAILED` with `registration_error` set.

| Code | Meaning |
|---|---|
| `409` | The stored two-step PIN cannot be used, or the number was recently deleted and WhatsApp has not finished removing it. Reset the PIN in WhatsApp Manager, or wait about 5 minutes |
| `429` | The 10-per-72-hours registration limit was reached. Wait until WhatsApp unblocks the number |

### Deregister a number

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/deregister` · scope `whatsapp:write`

No body. Deregisters the number on WhatsApp **only**: the number stays connected and active in your workspace, and `registration_status` moves to `DEREGISTERED`. Use it mid-workflow, for example to change `data_localization_region` by deregistering and registering again.

To stop using a number altogether, use [Disconnect](/docs/whatsapp-api#disconnect-a-number) instead, which also unsubscribes webhooks and deactivates the number locally.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/deregister \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns `{ "ok": true, "error": null }`. Shares the registration quota above.

### Request a verification code

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/request_code` · scope `whatsapp:write`

Asks WhatsApp to send a verification code to the number by SMS or voice call. This is number verification, separate from Cloud API registration, and does not use the registration quota.

| Field | Type | Required | Notes |
|---|---|---|---|
| `code_method` | string | No | `SMS` (default) or `VOICE`. Case-insensitive |
| `language` | string, 2 to 10 chars | No | Language of the code message, for example `en` or `en_US`. Default `en_US`. Passed to WhatsApp as given |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/request_code \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "code_method": "SMS", "language": "en_US" }'
```

Returns `{ "ok": true, "error": null }`. `409` if the number is already verified (`code_verification_status` is `VERIFIED`), so no code is sent.

### Submit the verification code

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/verify_code` · scope `whatsapp:write`

| Field | Type | Required | Notes |
|---|---|---|---|
| `code` | string, 1 to 16 chars | Yes | The code WhatsApp sent to the number |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/verify_code \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "code": "123456" }'
```

Returns `{ "ok": true, "error": null }` and sets `code_verification_status` to `VERIFIED`. A wrong or expired code returns `422`. The code is never stored or echoed back.

### Rotate the two-step PIN

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/two_step_pin` · scope `whatsapp:write`

Changes the number's two-step verification PIN in place. It does not deregister the number and does not use the registration quota. The new PIN is kept only once WhatsApp accepts it, and is used for later re-registrations. It is never returned by any endpoint.

| Field | Type | Required | Notes |
|---|---|---|---|
| `pin` | string, exactly 6 digits | Yes | The new PIN |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/two_step_pin \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "pin": "482913" }'
```

Returns `{ "ok": true, "error": null }`.

> Two-step verification cannot be **disabled** through any API. Once enabled it stays enabled, and this endpoint can only change the PIN. To turn it off, use WhatsApp Manager.

## Business profile

The profile is what a customer sees when they tap your business name in a chat: about line, address, description, email, websites, industry and profile picture. It belongs to a **number**, not the account, so two numbers on one account have independent profiles.

### Get the profile

`GET /api/v1/whatsapp/phone_numbers/{phone_id}/business_profile` · scope `whatsapp:read`

Read live from WhatsApp, so edits made in WhatsApp Manager show up immediately.

```bash
curl https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/business_profile \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "phone_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "phone_number_id": "1234567890",
  "about": "Fresh coffee, delivered.",
  "address": "12 MG Road, Bengaluru 560001",
  "description": "Specialty coffee roasted in small batches and shipped across India.",
  "email": "support@acme.example",
  "vertical": "RETAIL",
  "websites": ["https://acme.example"],
  "profile_picture_url": "https://pps.whatsapp.net/v/t61.24694-24/...",
  "last_synced_at": "2026-09-30T08:20:00Z",
  "stale": false
}
```

| Field | Type | Notes |
|---|---|---|
| `about` | string, nullable | The short about line |
| `address` | string, nullable | Business address |
| `description` | string, nullable | Longer description |
| `email` | string, nullable | Contact email |
| `vertical` | string, nullable | Industry, one of the values listed under [Update the profile](#update-the-profile) |
| `websites` | string[], nullable | Website URLs |
| `profile_picture_url` | string, nullable | Read-only. A temporary URL for the current picture |
| `last_synced_at` | datetime, nullable | When the profile was last read from WhatsApp |
| `stale` | boolean | `true` when WhatsApp could not be reached and the last stored copy was returned instead. Without a stored copy the error is returned |

### Update the profile

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/business_profile` · scope `whatsapp:write`

A partial update. Only the fields you send change; omitted fields are left as they are. Send a field as `null` to clear it. At least one field is required (`400` otherwise).

| Field | Type | Required | Notes |
|---|---|---|---|
| `about` | string or null | No | The short about line |
| `address` | string or null, max 256 chars | No | Business address |
| `description` | string or null | No | Longer description |
| `email` | string or null | No | Contact email |
| `vertical` | string or null | No | One of `OTHER`, `AUTO`, `BEAUTY`, `APPAREL`, `EDU`, `ENTERTAIN`, `EVENT_PLAN`, `FINANCE`, `GROCERY`, `GOVT`, `HOTEL`, `HEALTH`, `NONPROFIT`, `PROF_SERVICES`, `RETAIL`, `TRAVEL`, `RESTAURANT`, `ALCOHOL`, `ONLINE_GAMBLING`, `PHYSICAL_GAMBLING`, `OTC_DRUGS`. Case-insensitive. `UNDEFINED` and `NOT_A_BIZ` are rejected |
| `websites` | string[] or null | No | Website URLs |
| `profile_picture_handle` | string or null | No | An upload handle for the new picture. Get one by uploading a JPEG or PNG to [`POST /media/resumable`](/docs/whatsapp-templates). Passing a picture URL here does not work |

WhatsApp validates the other lengths itself, and its rejection comes back as `422`.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/business_profile \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "about": "Fresh coffee, delivered.",
    "vertical": "RETAIL",
    "websites": ["https://acme.example"]
  }'
```

Returns the profile object as WhatsApp reports it after the update.

## Blocked users

A blocked user cannot message the number, and the number cannot message them. WhatsApp only accepts a block for a user who messaged the number in the last 24 hours, and a number can hold up to 64,000 blocked users.

### List blocked users

`GET /api/v1/whatsapp/phone_numbers/{phone_id}/blocked_users` · scope `whatsapp:read`

| Param | Type | Default | Notes |
|---|---|---|---|
| `limit` | integer, 1 to 1000 | WhatsApp's default | Page size |
| `after` | string, max 1024 | | Cursor from a previous page's `after` |
| `before` | string, max 1024 | | Cursor from a previous page's `before` |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/blocked_users?limit=100" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "data": [{ "wa_id": "919000000000" }, { "wa_id": "919111111111" }],
  "after": "MjQZD",
  "before": null
}
```

Each entry carries only the `wa_id`; WhatsApp does not report when or by whom a user was blocked.

### Block users

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/blocked_users` · scope `whatsapp:write`

| Field | Type | Required | Notes |
|---|---|---|---|
| `users` | string[], 1 to 1000 items | Yes | Phone numbers in international format |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/blocked_users \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "users": ["+919000000000", "+919222222222"] }'
```

**Response (200 OK)**

A batch can partly succeed, so the response always lists both outcomes:

```json
{
  "blocked": [{ "input": "+919000000000", "wa_id": "919000000000" }],
  "failed": [
    {
      "input": "+919222222222",
      "wa_id": "919222222222",
      "reason": "This user has not messaged you in the last 24 hours, so Meta will not accept a block for them yet."
    }
  ]
}
```

| Code | Meaning |
|---|---|
| `400` | The business number tried to block itself |
| `409` | The blocklist is full (64,000), or another change to it is still in progress |
| `429` | Too many blocklist requests for this number. Wait, then retry |

### Unblock users

`DELETE /api/v1/whatsapp/phone_numbers/{phone_id}/blocked_users` · scope `whatsapp:write`

A `DELETE` with a JSON body, same shape as block.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/blocked_users \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "users": ["+919000000000"] }'
```

```json
{
  "unblocked": [{ "input": "+919000000000", "wa_id": "919000000000" }],
  "failed": []
}
```

## QR codes and short links

A QR code and its `wa.me` short link open a chat with your number with a message already typed. Use them on packaging, posters and receipts.

### The QR code object

| Field | Type | Notes |
|---|---|---|
| `code` | string | The code's id |
| `prefilled_message` | string | The message pre-typed for the customer |
| `deep_link_url` | string | The short link, for example `https://wa.me/message/4O4YGZEG3RIVE1` |
| `qr_image_url` | string, nullable | The QR image. Returned **only** on create; `null` on list, get and update. Keep it from the create response |

### List QR codes

`GET /api/v1/whatsapp/phone_numbers/{phone_id}/qr_codes` · scope `whatsapp:read`

```bash
curl https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/qr_codes \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns an array of QR code objects.

### Create a QR code

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/qr_codes` · scope `whatsapp:write`

| Field | Type | Required | Notes |
|---|---|---|---|
| `prefilled_message` | string, 1 to 140 chars | Yes | The message pre-typed for the customer |
| `generate_qr_image` | string | No | `SVG` (default) or `PNG`. Case-insensitive |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/qr_codes \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "prefilled_message": "Hi, I want to reorder my coffee", "generate_qr_image": "PNG" }'
```

**Response (200 OK)**

```json
{
  "code": "4O4YGZEG3RIVE1",
  "prefilled_message": "Hi, I want to reorder my coffee",
  "deep_link_url": "https://wa.me/message/4O4YGZEG3RIVE1",
  "qr_image_url": "https://scontent.xx.fbcdn.net/..."
}
```

### Get one QR code

`GET /api/v1/whatsapp/phone_numbers/{phone_id}/qr_codes/{code}` · scope `whatsapp:read`

Returns one QR code object, with `qr_image_url` as `null`. `404` if WhatsApp does not report that code on the number.

### Update a QR code

`PATCH /api/v1/whatsapp/phone_numbers/{phone_id}/qr_codes/{code}` · scope `whatsapp:write`

Changes only the prefilled message. The `code` and `deep_link_url` stay the same, so anything already printed keeps working.

| Field | Type | Required | Notes |
|---|---|---|---|
| `prefilled_message` | string, 1 to 140 chars | Yes | The new message |

```bash
curl -X PATCH https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/qr_codes/4O4YGZEG3RIVE1 \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "prefilled_message": "Hi, I want to track my order" }'
```

Returns the updated QR code object.

### Delete a QR code

`DELETE /api/v1/whatsapp/phone_numbers/{phone_id}/qr_codes/{code}` · scope `whatsapp:write`

```bash
curl -X DELETE https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/qr_codes/4O4YGZEG3RIVE1 \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "ok": true }
```

The short link stops working once the code is deleted.
