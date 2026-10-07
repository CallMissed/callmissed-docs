---
title: "Telephony API"
description: "Complete India KYC, rent Indian phone numbers, link them to a voice-agent bot, place, bridge and hang up outbound PSTN calls, and fetch recordings — all with your cm_ key."
slug: "telephony-api"
breadcrumb: "Numbers (PSTN)"
---

# Telephony API

Complete India KYC, rent Indian phone numbers, link them to a voice-agent bot, place, bridge and hang up outbound PSTN calls, and fetch recordings — all with your cm_ key.

## Overview

The Telephony API is a full lifecycle for **CallMissed Numbers**: submit an India KYC (compliance) application, wait for it to be accepted, search available Indian numbers, **buy** one (a paid action that draws your real credit balance), manage it, and place outbound **PSTN** calls answered by your AI voice agent.

**Base path:** `https://api.callmissed.com/api/v1/telephony`

> **The journey is ordered.** You cannot buy a number until you hold an **accepted** KYC application, and you cannot place a call until you own an **active** number. Follow the flow below top to bottom.

:::flow
icon:app | Submit KYC | Upload your business documents and details once
icon:gateway | CallMissed | Reviews the application; poll or sync until it is `accepted`
icon:done | Buy & call | Search a number, buy it (paid), link a bot, place calls
:::

**Authentication.** Every endpoint accepts both a **JWT** (`Authorization: Bearer <jwt>`) and an **API key** (`Authorization: Bearer cm_<key>`). API-key callers need the `telephony:read` scope for search/list/get and `telephony:write` for every write: buy, release, patch, the AI-call declaration, KYC submit/sync, placing, bridging and hanging up calls, and voicemail drops. Those writes additionally require an **owner/admin** role when called with a JWT.

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
  "plivo_compliance_id": null,
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

Applications move through `draft` → `submitted` → `accepted` / `rejected`, and an accepted application can later become `suspended` or `expired`. Only an **`accepted`** application can back a number purchase.

| Endpoint | Scope | Purpose |
|----------|-------|---------|
| `GET /compliance` | `telephony:read` | List your applications |
| `GET /compliance/{application_id}` | `telephony:read` | Get one application + status |
| `POST /compliance/{application_id}/sync` | `telephony:write` (owner/admin for JWT) | Refresh status from the carrier |

`GET /compliance` accepts `limit` (1–200, default 50) and `offset` (≥ 0). It returns an array of applications, newest first. `POST /compliance/{application_id}/sync` pulls the latest status and, if rejected, populates `rejection_reason`.

```bash
# List applications
curl https://api.callmissed.com/api/v1/telephony/compliance \
  -H "Authorization: Bearer cm_your_api_key"

# Refresh one application's status
curl -X POST https://api.callmissed.com/api/v1/telephony/compliance/{application_id}/sync \
  -H "Authorization: Bearer cm_your_api_key"
```

Once `status` is `accepted`, the application's `plivo_compliance_id` field holds the **compliance reference** you pass as `compliance_application_id` when buying a number (step 4). Note this is that field's value, not the application's `id`. A `GET`/`sync` on an application you do not own returns `404`.

## 3. Search Available Numbers

Search for Indian numbers before buying. Rates are returned **after** your tenant markup, with **18% GST added on top**. `rental_credits` is the total you will be charged per month; `rental_base_credits` and `rental_gst_credits` break it down.

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
    "rental_base_credits": 250,
    "rental_gst_credits": 45,
    "gst_rate": 0.18,
    "rental_credits": 295,
    "voice_enabled": true,
    "sms_enabled": false
  }
]
```

## 4. Buy a Number

Rent one of the searched numbers. This is a **paid action**: the first month (rental plus 18% GST) is drawn from your **real (paid) credit balance**. The signup bonus does **not** cover the purchase; if your paid balance is short, you get a `402` telling you to top up. Renewals are different. They draw on your whole credit balance, free credits included (see [Billing](#billing)).

`POST /numbers` · scope `telephony:write` (owner/admin for JWT)

**Request body**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `e164` | string | Yes | The number to buy, e.g. `+918080247309` |
| `compliance_application_id` | string | Yes | The `plivo_compliance_id` of your **accepted** KYC application |

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
  "outbound_bot_id": null,
  "alias": null,
  "monthly_rental_rate_usd": 2.5,
  "rental_credits": 295,
  "added_on": "2026-04-19",
  "renewal_date": "2026-05-19",
  "config": null,
  "metadata": null,
  "a2p_declared_at": null,
  "a2p_series": null,
  "a2p_tsp_reference": null,
  "created_at": "2026-04-19T12:00:00Z",
  "release_on": null
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
| `PATCH /numbers/{number_id}` | `telephony:write` (owner/admin for JWT) | Update alias / linked bot / per-number call config |
| `DELETE /numbers/{number_id}?confirm=true` | `telephony:write` (owner/admin for JWT) | Release a number (permanent) |

`GET /numbers` accepts `status` (e.g. `active`), `limit` (1–200, default 50), and `offset` (0–100000), newest first. `PATCH` updates only the fields you send; a `bot_id` or `outbound_bot_id` that is not yours returns `404`. `bot_id` is the agent that answers inbound calls. It also places outbound calls from the number unless you set `outbound_bot_id` to a different agent; set `outbound_bot_id` back to `null` to use one agent for both again. A number moves through `pending` → `active` → `suspended` (unpaid) → `released`.

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

The optional `config` object holds per-number call-handling overrides that **win over the linked bot's config on this number's calls** — so two numbers can share one bot yet greet and speak differently. The object **replaces** the stored overrides on every write (send the full set each time; `{}` clears every override). A key set to `null` or `""` is dropped, so the bot's own value applies again. Unknown keys return `422`.

| Key | Type | Notes |
|-----|------|-------|
| `voice_model` | string (≤ 100) | Voice LLM model id |
| `voice` | string (≤ 64) | Voice / speaker id |
| `language` | string (≤ 16) | e.g. `hi-IN` |
| `stt_model` | string (≤ 64) | Speech-to-text model id |
| `tts_model` | string (≤ 64) | Text-to-speech model id |
| `tts_provider` | string (≤ 32) | TTS provider id |
| `tts_engine` | string (≤ 32) | TTS engine id |
| `greeting` | string (≤ 500) | Opening line spoken on the call |
| `system_prompt` | string (≤ 8000) | Overrides the bot's persona for this number |
| `max_call_duration_seconds` | integer | 30–14400 |
| `allow_interruptions` | boolean | `false` turns barge-in off on this number, e.g. for a scripted disclosure. A speech-to-speech model always lets the caller interrupt, so `false` is ignored on one |
| `tools` | array of strings | Up to 20 agent tool names |
| `queue_id` | string | The call queue this number routes through. Normally set by [attaching a queue](/docs/call-handling-api#how-a-number-picks-one); keep it in the set you send, or the binding is cleared |
| `menu_id` / `flow_id` | string | Accepted only as `null` or `""`, to clear an old binding. Any other value returns `422`, because inbound menus and flows are not available yet |

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

`confirm=true` is **required** — releasing a number is permanent, stops its monthly rental charge, and is **not refunded**. Requires `telephony:write` (owner/admin for JWT). Returns `204 No Content` on success, `400` if `confirm` is omitted, and `404` if the number is not found or not owned by your tenant.

### Renew a number now

```bash
curl -X POST "https://api.callmissed.com/api/v1/telephony/numbers/{number_id}/renew" \
  -H "Authorization: Bearer cm_your_api_key"
```

Pays the next monthly rental straight away instead of waiting for the automatic renewal. It works for a `suspended` number, which is reactivated immediately, and for an `active` number whose `renewal_date` is within the next 7 days. The charge is `rental_credits`, drawn from your whole credit balance (free credits included), and `renewal_date` moves forward 30 days. Requires `telephony:write` (owner/admin for JWT). Returns the updated number (`200`), `402` if your balance can't cover the rental, `409` if the number is not due for renewal, and `404` if the number is not found or not owned by your tenant.

A suspended number carries `release_on`, the date it is released if the rental is still unpaid. It is `null` for every other status.

### Declare a number for AI calls (India)

TRAI requires a business making automated or AI calls to declare that use, and the caller IDs it uses, to its telecom provider first ([press release 119/2026](https://www.trai.gov.in/sites/default/files/2026-09/PR_No119of2026.pdf)). After you have made that declaration with your provider, record it on the number. Campaigns check it before they dial; see [TRAI readiness](/docs/voice-campaigns#trai-readiness-for-ai-calls-india).

```bash
curl -X PUT https://api.callmissed.com/api/v1/telephony/numbers/{number_id}/a2p-declaration \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"series": "1600", "tsp_reference": "DECL-2026-0042", "confirm_declared_to_tsp": true}'
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `series` | `140` \| `1600` \| `1601` \| `standard` | Yes | `140`, `1600` and `1601` are TRAI's commercial-call series; `standard` is an ordinary number |
| `tsp_reference` | string (≤64) | No | Your telecom provider's reference for the declaration |
| `confirm_declared_to_tsp` | boolean | Yes | Must be `true`. You confirm the declaration was made to your provider |

Returns the number with `a2p_declared_at`, `a2p_series` and `a2p_tsp_reference` set. Declaring again updates them. `confirm_declared_to_tsp: false` returns `422`, and so does a `tsp_reference` with characters other than letters, digits and `_ . / -`. `DELETE` on the same path withdraws the declaration and returns the number. Both need `telephony:write` (owner/admin for JWT). A released number returns `409`.

## 6. Place a Call

Originate an outbound PSTN call from one of your **active** numbers. Link a `bot_id` to have your AI voice agent handle the call; if you omit it, the number's outbound agent is used (`outbound_bot_id`, or `bot_id` when the number uses one agent for both).

`POST /calls` · scope `telephony:write` (owner/admin for JWT)

**Request body**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `from_number_id` | UUID | Yes | An active number you own |
| `to_e164` | string | Yes | The destination, e.g. `+919000000000` |
| `bot_id` | UUID | No | Voice-agent bot to answer the call |
| `reason` | string (≤ 500) | No | Plain-language purpose; spoken on the outbound greeting |
| `greeting_mode` | `outbound` \| `inbound` | No | Default follows the call: the agent introduces itself and states `reason`. `inbound` makes it greet in character from its own greeting, as it would answer an inbound caller, and not speak `reason` (still stored on the call). Useful for test calls to yourself |
| `variables` | object | No | Per-call `{{token}}` values, merged over the agent's declared input-variable defaults and rendered into its greeting and prompt. Up to 50 entries; keys match `^[a-zA-Z_][a-zA-Z0-9_]{0,63}$`; each value ≤ 200 characters |
| `status_callback_url` | string (≤ 2048) | No | A public `https` URL we `POST` this call's lifecycle payload to on every status change — the same shape as the account-level `call.*` [webhook](/docs/webhooks) events. Internal or private addresses are refused with `400` |

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
  "duration_seconds": null,
  "cost_credits": null,
  "ai_cost_credits": null,
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
| `400` | `status_callback_url` is not a valid public `https` URL |
| `402` | Insufficient credits to reserve the call |
| `403` | Missing `telephony:write` scope, JWT caller is not an owner/admin, the destination is on your [do-not-call list](/docs/voice-campaigns#do-not-call-list), or the destination country / number range is not allowed |
| `404` | `from_number_id` not found/active, or an explicit `bot_id` not found |
| `422` | `to_e164` is malformed, `variables` breaks the limits above, or the agent declares a required input variable you did not supply |
| `429` | Concurrent-call limit reached (up to 10 live calls per tenant), or a burst of calls that looks like toll fraud or robocalling was held |

The call row carries two costs. `cost_credits` is the **phone-line** leg only: it is filled in after the call by reconciliation (typically within the hour) and stays `null` for inbound calls. `ai_cost_credits` is the **voice agent** (STT, LLM and TTS) for the same call. `duration_seconds` is the best duration available right now, filled the moment the call ends.

## 7. List & Fetch Calls

| Endpoint | Scope | Purpose |
|----------|-------|---------|
| `GET /calls` | `telephony:read` | List your calls |
| `GET /calls/{call_id}` | `telephony:read` | Get one call |
| `GET /calls/{call_id}/recording` | `telephony:read` | Signed recording URL |
| `DELETE /calls/{call_id}` | `telephony:write` (owner/admin for JWT) | Hang up a live call |

**`GET /calls` query parameters**

| Param | Values | Default |
|-------|--------|---------|
| `direction` | `inbound` / `outbound` | — |
| `status` | `initiated` / `ringing` / `in_progress` / `completed` / `failed` / `no_answer` / `busy` | — |
| `limit` | 1–200 | 50 |
| `offset` | 0–100000 | 0 |

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

### Hang up a call

```bash
curl -X DELETE https://api.callmissed.com/api/v1/telephony/calls/{call_id} \
  -H "Authorization: Bearer cm_your_api_key"
```

Ends a live call and returns the call, now `completed`. Idempotent: hanging up a call that has already ended returns it unchanged. `404` if the call is not yours.

## 8. Click-to-call (bridge a person to a contact)

`POST /click-to-call` · scope `telephony:write` (owner/admin for JWT)

Rings **your** phone first, and once you pick up, dials the contact and puts you both on one line. No AI agent joins. Both legs are outbound calls from your active number and are billed as such; if you do not answer, nothing is charged and the contact is never dialled.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `destination` | string | One of these two | The contact's number, in E.164 |
| `contact_id` | UUID | One of these two | A CRM contact of yours; its phone number is dialled |
| `agent_number` | string | No | The phone to ring first, in E.164. Omit to use the operator phone set in your account settings |
| `bot_id` | UUID | No | Picks which of your numbers is used as the caller ID, and is recorded on both call rows |

```bash
curl -X POST https://api.callmissed.com/api/v1/telephony/click-to-call \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"agent_number": "+919000000001", "destination": "+919000000002"}'
```

**Response (200 OK):**

```json
{
  "status": "bridged",
  "room_name": "…",
  "agent_call_id": "5e6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c",
  "destination_call_id": "6f7a8b9c-0d1e-2f3a-4b5c-6d7e8f9a0b1c",
  "destination_e164": "+919000000002",
  "contact_id": null
}
```

`bridged` means both legs connected. A leg answered by voicemail also counts as connected, so it does not prove a person is listening. Both call ids work with `GET /calls/{call_id}` and `DELETE /calls/{call_id}`.

| Code | Meaning |
|------|---------|
| `402` | Insufficient credits to reserve both legs |
| `403` | The destination is on your do-not-call list, or not an allowed destination |
| `404` | `bot_id` or `contact_id` not found |
| `409` | No `agent_number` and no operator phone configured, or no active number to call from |
| `422` | Neither `destination` nor `contact_id` given, or a number is not valid E.164 |
| `429` | Too many calls in progress (a bridge uses two of the 10 concurrent lines) |
| `504` | Nobody answered your phone, so the contact was not dialled |

## 9. Voicemail drop

`POST /calls/{call_id}/voicemail-drop` · scope `telephony:write` (owner/admin for JWT)

Prepares the voicemail message for a call and returns where its audio is. Body: `{"template_id": "<uuid>"}`, or `{}` to use the template the call's campaign or agent is configured with. Templates are managed on the [Call Handling API](/docs/call-handling-api#voicemail-messages).

```json
{
  "action": "voicemail_drop",
  "template_id": "2f8a7c10-5b6d-4e3f-8a1b-7c9d0e1f2a3b",
  "audio_url": "https://…",
  "hangup_after": true
}
```

A text-only template is turned into speech, cached and billed the first time it is used. `audio_url` is short-lived. `404` if the call or template is not yours, `409` if no template is configured for the call or it has nothing to say, `503` if speech synthesis or audio storage is unavailable.

## Scopes

| Scope | Grants |
|-------|--------|
| `telephony:read` | Search numbers, list/get numbers, list/get compliance applications, list/get calls, fetch recording URLs, list BYO providers and connections |
| `telephony:write` | Buy/release/update numbers, the AI-call declaration, submit & sync compliance applications, place / bridge / hang up calls, voicemail drops, BYO connections and imports |

Every `telephony:write` action also requires an **owner/admin** role when called with a JWT (an API key carrying `telephony:write` is sufficient on its own).

## Billing

| Charge | When |
|--------|------|
| **Number rental** | Monthly, in credits, per active number (`rental_credits`, 18% GST included). The first month is paid from your **real** balance, not the signup bonus. Each renewal on `renewal_date` draws on your whole credit balance, free credits included. |
| **Call usage** | Reserved when a call is placed, then settled to the real cost from the call record after the call completes. |

Releasing a number stops its monthly rental charge (no refund for the current period). Ensure sufficient credits before buying numbers or placing calls, or those calls return `402`.

**Renewal reminders.** For the 7 days before a number's `renewal_date`, the account owner gets one email a day if the credit balance can't cover the rentals due. If a renewal still can't be paid, the number is suspended (calls stop) and you get a daily reminder until it is renewed or released at `release_on`. Add credits or pick a plan, then call [renew](#renew-a-number-now) to bring it back immediately.

## Webhooks

Telephony call-lifecycle and recording-ready events are delivered to the endpoints you configure via the [Webhooks API](/docs/webhooks). For one call, `status_callback_url` on `POST /calls` delivers the same lifecycle payload to a URL of your choice. Payloads are HMAC-SHA256 signed — verify the `X-CallMissed-Signature` header exactly as shown on the [Webhooks](/docs/webhooks) page before trusting a payload.
