---
title: "Telephony API"
description: "Rent Indian phone numbers, complete India KYC, place outbound PSTN calls, and fetch call recordings — all with your cm_ key."
slug: "telephony-api"
breadcrumb: "Channels"
---

# Telephony API

Rent Indian phone numbers, complete India KYC, place outbound PSTN calls, and fetch call recordings — all with your cm_ key.

## Overview

The Telephony API lets you rent **Indian phone numbers**, place and receive **PSTN voice calls**, and retrieve **call recordings** programmatically. Numbers can be linked to a voice-agent bot so inbound calls are answered by your AI agent.

> **Availability.** Telephony is **India-only** and gated per tenant. Every number purchase requires an accepted **India KYC (compliance) application**. If the feature is not yet enabled for your tenant, endpoints return `403 permission_denied`.

**Authentication:** All endpoints accept both **JWT** (`Authorization: Bearer <jwt>`) and **API key** (`Authorization: Bearer cm_<key>`). Keys need the `telephony:read` scope for search/list/get and `telephony:write` for buy/release/patch, compliance submission, and originating calls.

**Base path:** `/api/v1/telephony`

## Search Numbers

Search available Indian numbers before renting one.

```bash
curl "https://api.callmissed.com/api/v1/telephony/numbers/search?country_iso=IN&type=local&pattern=80802&limit=10" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Query parameters:**

| Param | Values | Default |
|-------|--------|---------|
| `country_iso` | `IN` | `IN` |
| `type` | `local` / `mobile` / `tollfree` | — |
| `pattern` | digit substring to match | — |
| `limit` | 1–20 | 10 |

**Response (200 OK):**
```json
[
  {
    "number": "+918080247309",
    "number_type": "local",
    "region": "Mumbai",
    "monthly_rental_rate_usd": 2.50,
    "rental_credits": 250,
    "voice_enabled": true,
    "sms_enabled": false
  }
]
```

## Buy a Number

Rent a searched number. Requires an **accepted** compliance application ID.

```bash
curl -X POST https://api.callmissed.com/api/v1/telephony/numbers \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "e164": "+918080247309",
    "compliance_application_id": "3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e"
  }'
```

**Response (201 Created):** a `PhoneNumber` object.
```json
{
  "id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "e164": "+918080247309",
  "number_type": "local",
  "region": "Mumbai",
  "status": "active",
  "alias": null,
  "bot_id": null,
  "voice_enabled": true,
  "sms_enabled": false,
  "monthly_rental_credits": 250,
  "rented_at": "2026-04-19T12:00:00Z",
  "created_at": "2026-04-19T12:00:00Z"
}
```

## Manage Numbers

```bash
# List your rented numbers
curl https://api.callmissed.com/api/v1/telephony/numbers \
  -H "Authorization: Bearer cm_your_api_key"

# Get one
curl https://api.callmissed.com/api/v1/telephony/numbers/{id} \
  -H "Authorization: Bearer cm_your_api_key"

# Update alias / linked bot
curl -X PATCH https://api.callmissed.com/api/v1/telephony/numbers/{id} \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"alias": "Support line", "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d"}'
```

**Release a number (irreversible):**

```bash
curl -X DELETE "https://api.callmissed.com/api/v1/telephony/numbers/{id}?confirm=true" \
  -H "Authorization: Bearer cm_your_api_key"
```

The `confirm=true` query parameter is **required** — releasing a number is permanent and stops its monthly rental billing.

## Compliance & KYC

Every rented Indian number must be backed by an accepted KYC application. Submit one, then poll or sync its status until it is `accepted` before buying.

Submission is a **multipart form** carrying your business details plus the two **mandatory** documents: the registration certificate (Certificate of Incorporation or Udyam certificate) and the GST certificate. Files must be PDF, JPEG, or PNG, up to 5 MB each — and the legal business name must match **exactly** on both documents or the application is rejected.

```bash
curl -X POST https://api.callmissed.com/api/v1/telephony/compliance \
  -H "Authorization: Bearer cm_your_api_key" \
  -F 'alias=Acme India KYC' \
  -F 'business_name=ACME TECHNOLOGIES PRIVATE LIMITED' \
  -F 'registration_number=U72900KA2020PTC000000' \
  -F 'email=compliance@acme.in' \
  -F 'address_line1=123 MG Road' \
  -F 'city=Bengaluru' \
  -F 'state=Karnataka' \
  -F 'postal_code=560001' \
  -F 'registration_cert=@certificate-of-incorporation.pdf' \
  -F 'gst_cert=@gst-certificate.pdf'
```

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/telephony/compliance` | List your KYC applications |
| `GET /api/v1/telephony/compliance/{id}` | Get one application + status |
| `POST /api/v1/telephony/compliance/{id}/sync` | Refresh status from the carrier |

Applications move through `draft` → `submitted` → `accepted` / `rejected`. Only `accepted` applications can back a number purchase.

## Place a Call

Originate an outbound PSTN call from one of your rented numbers. Link a `bot_id` to have your AI voice agent handle the call.

```bash
curl -X POST https://api.callmissed.com/api/v1/telephony/calls \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "from_number_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
    "to_e164": "+919000000000",
    "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d"
  }'
```

**Response (201 Created):** a `TelephonyCall` object.
```json
{
  "id": "5e6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c",
  "direction": "outbound",
  "status": "initiated",
  "from_e164": "+918080247309",
  "to_e164": "+919000000000",
  "from_number_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
  "duration_seconds": null,
  "call_credits": null,
  "started_at": null,
  "ended_at": null,
  "created_at": "2026-04-19T12:00:00Z"
}
```

## List Calls

```bash
curl "https://api.callmissed.com/api/v1/telephony/calls?direction=outbound&status=completed&limit=50&offset=0" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Query parameters:**

| Param | Values | Default |
|-------|--------|---------|
| `direction` | `inbound` / `outbound` | — |
| `status` | `initiated` / `ringing` / `in_progress` / `completed` / `failed` / `no_answer` | — |
| `limit` | 1–200 | 50 |
| `offset` | ≥ 0 | 0 |

Returns an array of `TelephonyCall` objects. `GET /api/v1/telephony/calls/{id}` fetches a single call.

## Recordings

If a call was recorded, fetch a short-lived signed URL for its audio:

```bash
curl https://api.callmissed.com/api/v1/telephony/calls/{id}/recording \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK):**
```json
{ "url": "https://media.callmissed.com/recordings/....?token=..." }
```

The URL is time-limited — fetch it on demand rather than storing it. Returns `404` if the call has no recording.

## Scopes

| Scope | Grants |
|-------|--------|
| `telephony:read` | Search numbers, list/get numbers, list/get compliance applications, list/get calls, fetch recording URLs |
| `telephony:write` | Buy/release/update numbers, submit & sync compliance applications, originate calls |

## Billing

| Charge | Type | When |
|--------|------|------|
| **Number rental** | `number_rental` transaction | Monthly, in credits, per active number (`monthly_rental_credits`) |
| **Call usage** | `telephony_call` service / `call_charge` transaction | Per call, reconciled from the call record after the call completes |

Releasing a number stops its monthly rental charge. Call charges are metered from your credit balance; ensure sufficient credits before originating calls.

## Webhooks

Telephony call lifecycle and recording-ready events are delivered to the webhook endpoints you configure via the [Webhooks API](/docs/webhooks). Payloads are HMAC-SHA256 signed — verify the `X-CallMissed-Signature` header exactly as shown on the [Webhooks](/docs/webhooks) page before trusting a payload.