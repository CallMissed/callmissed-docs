---
title: "Telephony API"
description: "Complete India KYC, rent Indian phone numbers, link them to a voice-agent bot, place outbound PSTN calls, and fetch recordings — all with your cm_ key."
slug: "telephony-api"
breadcrumb: "Numbers (PSTN)"
---

# Telephony API

Complete India KYC, rent Indian phone numbers, link them to a voice-agent bot, place outbound PSTN calls, and fetch recordings — all with your cm_ key.

## Overview

The Telephony API is a full lifecycle for **CallMissed Numbers**: submit an India KYC (compliance) application, wait for it to be accepted, search available Indian numbers, **buy** one (a paid action that draws your real credit balance), manage it, and place outbound **PSTN** calls answered by your AI voice agent.

**Base path:** `https://api.callmissed.com/api/v1/telephony`

> **The journey is ordered.** You cannot buy a number until you hold an **accepted** KYC application, and you cannot place a call until you own an **active** number. Follow the flow below top to bottom.

:::flow
icon:app | Submit KYC | Upload your business documents and details once
icon:gateway | CallMissed | Reviews the application; poll or sync until it is `accepted`
icon:done | Buy & call | Search a number, buy it (paid), link a bot, place calls
:::

**Authentication.** Every endpoint accepts both a **JWT** (`Authorization: Bearer <jwt>`) and an **API key** (`Authorization: Bearer cm_<key>`). API-key callers need the `telephony:read` scope for search/list/get and `telephony:write` for buy, release, patch, KYC submit/sync, and originating calls. Money-affecting and destructive actions (buy, release, patch, KYC submit, originate call) additionally require an **owner/admin** role when called with a JWT.

> **Availability.** Telephony is **India-only** and enabled per tenant. When the feature is not enabled for your tenant, the routes are unmounted and every call returns `404`.

## 1. Submit KYC (Compliance)

Every rented Indian number must be backed by an **accepted** KYC application. Submission is a **multipart form** carrying your business details plus the two **mandatory** documents:

- **Registration certificate** — Certificate of Incorporation (CIN) or Udyam certificate
- **GST certificate**

Files must be **PDF, JPEG, or PNG**, up to **5 MB each**. The legal business name must match **exactly** on both documents or the application is rejected upstream.

`POST /compliance` · scope `telephony:write` (owner/admin for JWT)

**Form fields:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `alias` | string (1–128) | Yes | A label for this application |
| `business_name` | string (1–100) | Yes | Legal name, exactly as printed on **both** documents |
| `registration_number` | string (1–64) | Yes | CIN or Udyam number |
| `email` | string (3–254) | Yes | Business contact email |
| `address_line1` | string (1–255) | Yes | |
| `address_line2` | string (0–255) | No | |
| `city` | string (1–100) | Yes | |
| `state` | string (1–100) | Yes | |
| `postal_code` | string (1–16) | Yes | |
| `registration_cert` | file | Yes | COI or Udyam certificate (PDF/JPEG/PNG, ≤ 5 MB) |
| `gst_cert` | file | Yes | GST certificate (PDF/JPEG/PNG, ≤ 5 MB) |

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/telephony/compliance \
  -H "Authorization: Bearer cm_your_api_key" \
  -F 'alias=Acme India KYC' \
  -F 'business_name=ACME TECHNOLOGIES PRIVATE LIMITED' \
  -F 'registration_number=U72900KA2020PTC000000' \
  -F 'email=compliance@acme.in' \
  -F 'address_line1=123 MG Road' \
  -F 'address_line2=Suite 400' \
  -F 'city=Bengaluru' \
  -F 'state=Karnataka' \
  -F 'postal_code=560001' \
  -F 'registration_cert=@certificate-of-incorporation.pdf' \
  -F 'gst_cert=@gst-certificate.pdf'
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/telephony"
headers = {"Authorization": "Bearer cm_your_api_key"}

data = {
    "alias": "Acme India KYC",
    "business_name": "ACME TECHNOLOGIES PRIVATE LIMITED",
    "registration_number": "U72900KA2020PTC000000",
    "email": "compliance@acme.in",
    "address_line1": "123 MG Road",
    "address_line2": "Suite 400",
    "city": "Bengaluru",
    "state": "Karnataka",
    "postal_code": "560001",
}
files = {
    "registration_cert": ("coi.pdf", open("coi.pdf", "rb"), "application/pdf"),
    "gst_cert": ("gst.pdf", open("gst.pdf", "rb"), "application/pdf"),
}

resp = httpx.post(f"{BASE}/compliance", headers=headers, data=data, files=files)
application = resp.json()
print(application["id"], application["status"])  # e.g. "...", "submitted"
```
:::

**Response (200 OK)** — a compliance application:

```json
{
  "id": "3f9a1c20-7d8e-4b1a-9c2f-5e6a7b8c9d0e",
  "tenant_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "alias": "Acme India KYC",
  "country_iso": "IN",
  "number_type": "local",
  "user_type": "business",
  "status": "submitted",
  "rejection_reason": null,
  "business_name": "ACME TECHNOLOGIES PRIVATE LIMITED",
  "registration_number": "U72900KA2020PTC000000",
  "created_at": "2026-04-19T12:00:00Z"
}
```

**Status codes**

| Code | Meaning |
|------|---------|
| `200` | Application created and submitted |
| `403` | Missing `telephony:write` scope, or JWT caller is not an owner/admin |
| `413` | A document exceeds the 5 MB limit |
| `422` | A document is missing, empty, or not a PDF/JPEG/PNG |
| `404` | Telephony not enabled for your tenant |

## 2. Check KYC Status

Applications move through `draft` → `submitted` → `accepted` / `rejected`. Only an **`accepted`** application can back a number purchase.

| Endpoint | Scope | Purpose |
|----------|-------|---------|
| `GET /compliance` | `telephony:read` | List your applications |
| `GET /compliance/{application_id}` | `telephony:read` | Get one application + status |
| `POST /compliance/{application_id}/sync` | `telephony:write` | Refresh status from the carrier |

`GET /compliance` accepts `limit` (1–200, default 50) and `offset` (≥ 0). It returns an array of applications, newest first. `POST /compliance/{application_id}/sync` pulls the latest status and, if rejected, populates `rejection_reason`.

```bash
# List applications
curl https://api.callmissed.com/api/v1/telephony/compliance \
  -H "Authorization: Bearer cm_your_api_key"

# Refresh one application's status
curl -X POST https://api.callmissed.com/api/v1/telephony/compliance/{application_id}/sync \
  -H "Authorization: Bearer cm_your_api_key"
```

Once `status` is `accepted`, the application carries a **compliance reference** you pass as `compliance_application_id` when buying a number (step 4). A `GET`/`sync` on an application you do not own returns `404`.

## 3. Search Available Numbers

Search for Indian numbers before buying. Rates are returned **after** your tenant markup — `rental_credits` is what you will actually be charged per month.

`GET /numbers/search` · scope `telephony:read`

**Query parameters**

| Param | Values | Default |
|-------|--------|---------|
| `country_iso` | 2-letter ISO (`IN`) | `IN` |
| `type` | `local` / `mobile` / `tollfree` | — |
| `pattern` | digit substring to match (max 32 chars) | — |
| `limit` | 1–20 | 20 |

```bash
curl "https://api.callmissed.com/api/v1/telephony/numbers/search?country_iso=IN&type=local&pattern=80802&limit=10" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)** — an array of search hits:

```json
[
  {
    "number": "+918080247309",
    "number_type": "local",
    "country": "IN",
    "region": "Mumbai",
    "monthly_rental_rate_usd": 2.5,
    "rental_credits": 250,
    "voice_enabled": true,
    "sms_enabled": false
  }
]
```

## 4. Buy a Number

Rent one of the searched numbers. This is a **paid action** — it draws your **real (paid) credit balance**. The signup bonus does **not** cover a number rental; if your paid balance is short, you get a `402` telling you to top up.

`POST /numbers` · scope `telephony:write` (owner/admin for JWT)

**Request body**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `e164` | string | Yes | The number to buy, e.g. `+918080247309` |
| `compliance_application_id` | string | Yes | The compliance reference from your **accepted** KYC application |

The purchase runs in a strict, money-safe order: it verifies your KYC application is accepted (else `409`), confirms the number is still available and prices it live (else `422`), deducts the rental credits, and only then rents the number. If the rent fails, the credits are refunded automatically.

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/telephony/numbers \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "e164": "+918080247309",
    "compliance_application_id": "your-accepted-compliance-reference"
  }'
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/telephony"
headers = {"Authorization": "Bearer cm_your_api_key"}

resp = httpx.post(
    f"{BASE}/numbers",
    headers=headers,
    json={
        "e164": "+918080247309",
        "compliance_application_id": "your-accepted-compliance-reference",
    },
)

if resp.status_code == 402:
    print("Top up your paid balance before buying a number")
elif resp.status_code == 409:
    print("Your KYC application is not accepted yet")
else:
    number = resp.json()
    print(number["id"], number["status"])  # e.g. "...", "active"
```
:::

**Response (200 OK)** — the rented number:

```json
{
  "id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "tenant_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "e164": "+918080247309",
  "country_iso": "IN",
  "number_type": "local",
  "status": "active",
  "bot_id": null,
  "alias": null,
  "monthly_rental_rate_usd": 2.5,
  "rental_credits": 250,
  "added_on": "2026-04-19",
  "renewal_date": "2026-05-19",
  "config": null,
  "metadata": null,
  "created_at": "2026-04-19T12:00:00Z"
}
```

**Status codes**

| Code | Meaning |
|------|---------|
| `200` | Number rented and active |
| `402` | Insufficient **real** balance — top up to buy |
| `403` | Missing `telephony:write` scope, or JWT caller is not an owner/admin |
| `409` | KYC not accepted, or you already hold this number |
| `422` | Number no longer available, or `e164` is malformed |
| `404` | Telephony not enabled for your tenant |

## 5. Manage Numbers

| Endpoint | Scope | Purpose |
|----------|-------|---------|
| `GET /numbers` | `telephony:read` | List your rented numbers |
| `GET /numbers/{number_id}` | `telephony:read` | Get one number |
| `PATCH /numbers/{number_id}` | `telephony:write` | Update alias / linked bot / per-number call config |
| `DELETE /numbers/{number_id}?confirm=true` | `telephony:write` | Release a number (permanent) |

`GET /numbers` accepts `status` (e.g. `active`), `limit` (1–200, default 50), and `offset` (≥ 0). A number moves through `pending` → `active` → `suspended` (unpaid) → `released`.

```bash
# List your rented numbers
curl https://api.callmissed.com/api/v1/telephony/numbers \
  -H "Authorization: Bearer cm_your_api_key"

# Get one
curl https://api.callmissed.com/api/v1/telephony/numbers/{number_id} \
  -H "Authorization: Bearer cm_your_api_key"

# Update the alias and link a voice-agent bot
curl -X PATCH https://api.callmissed.com/api/v1/telephony/numbers/{number_id} \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"alias": "Support line", "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d"}'
```

### Per-number call overrides

The optional `config` object holds per-number call-handling overrides that **win over the linked bot's config on this number's calls** — so two numbers can share one bot yet greet and speak differently. The object **replaces** the stored overrides on every write (send the full set each time; `{}` clears every override). Unknown keys return `422`.

| Key | Type | Notes |
|-----|------|-------|
| `voice_model` | string | Voice LLM model id |
| `voice` | string | Voice / speaker id |
| `language` | string | e.g. `hi-IN` |
| `stt_model` | string | Speech-to-text model id |
| `tts_model` | string | Text-to-speech model id |
| `tts_provider` | string | TTS provider id |
| `tts_engine` | string | TTS engine id |
| `greeting` | string | Opening line spoken on the call |
| `system_prompt` | string | Overrides the bot's persona for this number |
| `max_call_duration_seconds` | integer | 30–14400 |
| `voice_fallbacks` | array of strings | Up to 2 fallback model ids |
| `tools` | array of strings | Up to 20 agent tool names |

```bash
curl -X PATCH https://api.callmissed.com/api/v1/telephony/numbers/{number_id} \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"config": {"greeting": "Namaste! Aap Support line par pahunche hain.", "language": "hi-IN", "max_call_duration_seconds": 600}}'
```

### Release a number

```bash
curl -X DELETE "https://api.callmissed.com/api/v1/telephony/numbers/{number_id}?confirm=true" \
  -H "Authorization: Bearer cm_your_api_key"
```

`confirm=true` is **required** — releasing a number is permanent, stops its monthly rental charge, and is **not refunded**. Returns `204 No Content` on success, `400` if `confirm` is omitted, and `404` if the number is not found or not owned by your tenant.

## 6. Place a Call

Originate an outbound PSTN call from one of your **active** numbers. Link a `bot_id` to have your AI voice agent handle the call; if you omit it, the number's persistently bound bot is used.

`POST /calls` · scope `telephony:write` (owner/admin for JWT)

**Request body**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `from_number_id` | UUID | Yes | An active number you own |
| `to_e164` | string | Yes | The destination, e.g. `+919000000000` |
| `bot_id` | UUID | No | Voice-agent bot to answer the call |
| `reason` | string (≤ 500) | No | Plain-language purpose; spoken on the outbound greeting |

```bash
curl -X POST https://api.callmissed.com/api/v1/telephony/calls \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "from_number_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
    "to_e164": "+919000000000",
    "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
    "reason": "Confirming your appointment for tomorrow"
  }'
```

**Response (200 OK)** — the created call (status advances via webhooks):

```json
{
  "id": "5e6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c",
  "tenant_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "phone_number_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
  "voice_session_id": "7f8a9b0c-1d2e-3f4a-5b6c-7d8e9f0a1b2c",
  "direction": "outbound",
  "status": "initiated",
  "remote_e164": "+919000000000",
  "bill_duration_seconds": null,
  "billed_duration_seconds": null,
  "cost_credits": null,
  "hangup_cause_code": null,
  "hangup_source": null,
  "recording_id": null,
  "metadata": null,
  "created_at": "2026-04-19T12:00:00Z"
}
```

**Status codes**

| Code | Meaning |
|------|---------|
| `200` | Call created and dialing |
| `402` | Insufficient credits to reserve the call |
| `403` | Missing `telephony:write` scope, or JWT caller is not an owner/admin |
| `404` | `from_number_id` not found/active, or an explicit `bot_id` not found |
| `422` | `to_e164` is malformed |
| `429` | Concurrent-call limit reached (up to 10 live calls per tenant) |

## 7. List & Fetch Calls

| Endpoint | Scope | Purpose |
|----------|-------|---------|
| `GET /calls` | `telephony:read` | List your calls |
| `GET /calls/{call_id}` | `telephony:read` | Get one call |
| `GET /calls/{call_id}/recording` | `telephony:read` | Signed recording URL |

**`GET /calls` query parameters**

| Param | Values | Default |
|-------|--------|---------|
| `direction` | `inbound` / `outbound` | — |
| `status` | `initiated` / `ringing` / `in_progress` / `completed` / `failed` / `no_answer` / `busy` | — |
| `limit` | 1–200 | 50 |
| `offset` | ≥ 0 | 0 |

```bash
curl "https://api.callmissed.com/api/v1/telephony/calls?direction=outbound&status=completed&limit=50&offset=0" \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns an array of calls, newest first. `GET /calls/{call_id}` fetches a single call (`404` if not owned).

### Recordings

If a call was recorded, fetch a short-lived signed URL for its audio:

```bash
curl https://api.callmissed.com/api/v1/telephony/calls/{call_id}/recording \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK):**

```json
{ "url": "https://media.callmissed.com/recordings/....?token=..." }
```

The URL is time-limited — fetch it on demand rather than storing it. Returns `404` if the call has no recording, and `503` if recording storage is temporarily unavailable.

## Scopes

| Scope | Grants |
|-------|--------|
| `telephony:read` | Search numbers, list/get numbers, list/get compliance applications, list/get calls, fetch recording URLs |
| `telephony:write` | Buy/release/update numbers, submit & sync compliance applications, originate calls |

Buy, release, patch, KYC submit, and originate-call also require an **owner/admin** role when called with a JWT (an API key carrying `telephony:write` is sufficient on its own).

## Billing

| Charge | When |
|--------|------|
| **Number rental** | Monthly, in credits, per active number (`rental_credits`). Paid from your **real** balance — not the signup bonus. Renews on `renewal_date`. |
| **Call usage** | Reserved when a call is placed, then settled to the real cost from the call record after the call completes. |

Releasing a number stops its monthly rental charge (no refund for the current period). Ensure sufficient credits before buying numbers or placing calls, or those calls return `402`.

## Webhooks

Telephony call-lifecycle and recording-ready events are delivered to the endpoints you configure via the [Webhooks API](/docs/webhooks). Payloads are HMAC-SHA256 signed — verify the `X-CallMissed-Signature` header exactly as shown on the [Webhooks](/docs/webhooks) page before trusting a payload.
