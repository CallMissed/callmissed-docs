---
title: "Calling"
description: "Voice calls over WhatsApp: enable calling, request permission, place a business-initiated call answered by your agent, and read call logs with transcripts."
slug: "whatsapp-calling"
breadcrumb: "WhatsApp"
---

# Calling

Voice calls over WhatsApp: enable calling, request permission, place a business-initiated call answered by your agent, and read call logs with transcripts.

WhatsApp Calling lets a customer call your business number, and lets you call them, over WhatsApp itself rather than the phone network. Calls are answered by the same agent that handles the number's messages, with the same voice, model and system prompt, so a call produces a transcript and per-turn AI cost exactly like any other voice session.

All endpoints are under `https://api.callmissed.com/api/v1/whatsapp`.

## Before you can call

Three things have to be true, and each has its own failure code:

1. **Calling is enabled on the number.** Turn it on with [call settings](#call-settings), or in WhatsApp Manager. Otherwise sends fail with `409` telling you calling is not enabled.
2. **The number's messaging limit is 2000 or above.** WhatsApp requires it. Below that you get `409`.
3. **The customer granted call permission.** Inbound calls need nothing, but a business-initiated call is permission-gated and returns `403` without one.

## Call settings

### Read settings

`GET /api/v1/whatsapp/calling/settings` · scope `whatsapp:read`

| Query param | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The number |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/calling/settings?phone_number_id=1234567890" \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns WhatsApp's own settings object for the number, unchanged.

### Update settings

`POST /api/v1/whatsapp/calling/settings` · scope `whatsapp:write`

Send only the fields you want to change. At least one is required, or you get `400` with `"No calling settings provided"`.

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The number |
| `status` | string | No | `ENABLED` or `DISABLED`. The master switch for calling on this number |
| `call_icon_visibility` | string, max 32 | No | Where WhatsApp shows the call button |
| `callback_permission_status` | string | No | `ENABLED` or `DISABLED` |
| `call_hours` | object | No | Your calling hours, forwarded to WhatsApp unchanged |
| `sip` | object | No | SIP configuration, forwarded unchanged |
| `audio` | object | No | Audio configuration, forwarded unchanged |
| `voicemail` | object | No | Voicemail configuration, forwarded unchanged |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/calling/settings \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "status": "ENABLED",
    "callback_permission_status": "ENABLED"
  }'
```

The nested objects are passed to WhatsApp exactly as you send them and validated there, so any option WhatsApp supports works without waiting on a CallMissed release. Returns WhatsApp's response.

## Call permission

WhatsApp requires an explicit grant from the customer before a business may call them. A grant is `temporary` or `permanent`; temporary grants expire, so re-check before relying on one.

### Check one user's permission

`GET /api/v1/whatsapp/calling/permissions` · scope `whatsapp:read`

| Query param | Type | Required | Notes |
|---|---|---|---|
| `user` | string, 5 to 20 chars | Yes | The customer's WhatsApp id |
| `phone_id` / `phone_number_id` | UUID / string | One of | Your number |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/calling/permissions?phone_number_id=1234567890&user=919000000000" \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns WhatsApp's permission object, which carries a `permission.status` of `no_permission`, `temporary` or `permanent`. The result is also recorded locally, so a number that has granted permission shows up in [allowed numbers](#list-allowed-numbers) even before you call it.

### Ask for permission

`POST /api/v1/whatsapp/calling/permission-request` · scope `whatsapp:send`

Sends the customer an interactive message asking them to allow calls. It only works inside an open 24-hour customer service window. Outside it, send an approved `call_permission_request` template instead.

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | Your number |
| `to` | string, 5 to 20 chars | Yes | The customer in E.164 |
| `body_text` | string, 1 to 1024 chars | Yes | Why you want to call |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/calling/permission-request \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "body_text": "Can we call you about order AC-10294? It will take about two minutes."
  }'
```

**Response (200 OK)**

```json
{ "wamid": "wamid.HBgMOTE5MDAwMDAwMDAwFQIAERgSQjE1RDNBOEY0RTVCOTAxMgA=" }
```

The customer's answer arrives as an inbound event, and their permission state is reflected on the next permission check.

### List allowed numbers

`GET /api/v1/whatsapp/calling/allowed-numbers` · scope `whatsapp:read`

Numbers that have granted call permission, so you can pick one and dial. WhatsApp exposes no bulk lookup, so this is derived from the permission state recorded whenever you check permission, request it, or place a call. Deduplicated per number, most recent first.

| Query param | Type | Default | Notes |
|---|---|---|---|
| `phone_id` | UUID | none | Restrict to one of your numbers |
| `limit` | integer, 1 to 500 | 100 | Max rows |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/calling/allowed-numbers?limit=100" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
[
  {
    "to_number": "919000000000",
    "permission_status": "permanent",
    "last_call_at": "2026-04-19T12:31:08Z"
  }
]
```

| Field | Type | Notes |
|---|---|---|
| `to_number` | string | The customer's number |
| `permission_status` | string | `temporary` or `permanent` |
| `last_call_at` | datetime, nullable | When we last saw this number |

A `temporary` grant expires. Re-check with the permissions endpoint before relying on one.

## Place a call

`POST /api/v1/whatsapp/calling/initiate` · scope `whatsapp:send`

Places a business-initiated call. Permission is verified first, the credits are held, the media bridge is provisioned, and the agent picks up when the customer answers.

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_id` / `phone_number_id` | UUID / string | One of | The number to call from |
| `to` | string, 5 to 20 chars | Yes | The customer in E.164 |
| `reason` | string, max 512 | No | Your own note on why the call was placed. Stored on the call log, not sent to WhatsApp |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/calling/initiate \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "reason": "Delivery address could not be verified for AC-10294"
  }'
```

**Response (200 OK)**

```json
{
  "id": "c4a7f210-3b8e-4d1f-9a2c-5e6b7d8f9012",
  "session_id": "e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b",
  "status": "INITIATED"
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | The CallMissed call row |
| `session_id` | UUID | The linked voice session, where the transcript and AI cost land |
| `status` | string | Always `INITIATED` at this point |

WhatsApp's own `call_id` does not exist yet. It is minted moments later as the call is placed, and appears on the call log once the handshake completes. Correlate by `session_id` until then.

**Failures**

| Code | Meaning |
|---|---|
| `402` | Not enough credits to cover the call. Nothing was placed |
| `403` | The customer has not granted call permission. Send a permission request first |
| `404` | The calling number is not on your workspace |
| `409` | Calling is not enabled on the number, or its messaging limit is below 2000 |
| `503` | The calling media bridge is unavailable right now |

### The credit hold

The network leg is charged when the call ends, so an unfundable call cannot be undone once placed. Before the call is provisioned, its worst-case cost is held: the agent's own maximum call duration, priced at the recipient's regional per-minute rate. A shortfall returns `402` and nothing is placed, no room is created and WhatsApp is never asked to dial.

Over-holding is self-correcting. When the call ends, the real charge settles and the remainder is released. Inbound calls are not held at all, because WhatsApp does not charge for user-initiated calls.

## Call logs

### List calls

`GET /api/v1/whatsapp/calling/calls` · scope `whatsapp:read`

Most recent first.

| Query param | Type | Default | Notes |
|---|---|---|---|
| `phone_id` | UUID | none | Restrict to one of your numbers |
| `limit` | integer, 1 to 200 | 50 | Max rows |
| `offset` | integer, 0 to 100000 | 0 | Pagination offset |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/calling/calls?limit=50&offset=0" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
[
  {
    "id": "c4a7f210-3b8e-4d1f-9a2c-5e6b7d8f9012",
    "call_id": "wacid.HBgMOTE5MDAwMDAwMDAwFQIAERgSQTBGOEQ3MTJGM0EyRDFDNQA=",
    "direction": "BUSINESS_INITIATED",
    "status": "COMPLETED",
    "from_wa_id": "1234567890",
    "to_number": "919000000000",
    "duration_seconds": 96,
    "cost_credits": 4.8,
    "voice_session_id": "e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b",
    "permission_status": "permanent",
    "created_at": "2026-04-19T12:31:08Z",
    "ai_cost_credits": 2.1374
  }
]
```

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | The CallMissed call row |
| `call_id` | string | WhatsApp's call id. Use it on the detail and terminate endpoints |
| `direction` | string | `BUSINESS_INITIATED` or `USER_INITIATED` |
| `status` | string | Lifecycle state, for example `INITIATED`, `RINGING`, `ACCEPTED`, `COMPLETED`, `TERMINATED`, `FAILED`. Stored as WhatsApp reports it, so new values can appear |
| `from_wa_id` | string, nullable | The calling side |
| `to_number` | string, nullable | The called side |
| `duration_seconds` | integer, nullable | Call length. Falls back to the voice session's duration when the call was ended by the agent |
| `cost_credits` | float, nullable | The WhatsApp network leg only. Zero for inbound calls, which WhatsApp does not charge for |
| `voice_session_id` | UUID, nullable | The linked voice session |
| `permission_status` | string, nullable | The permission snapshot when the call was placed |
| `created_at` | datetime, nullable | ISO 8601 UTC |
| `ai_cost_credits` | float, nullable | Speech, model and voice cost for the call. Separate from `cost_credits` |

Permission-check marker rows are excluded, so this list is real calls only.

### Get one call with its transcript

`GET /api/v1/whatsapp/calling/calls/{call_id}` · scope `whatsapp:read`

`{call_id}` is WhatsApp's call id. Only `A-Z a-z 0-9 _ . : = -` are accepted in the path, up to 128 characters.

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/calling/calls/wacid.HBgMOTE5MDAwMDAwMDAwFQIAERgSQTBGOEQ3MTJGM0EyRDFDNQA=" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "id": "c4a7f210-3b8e-4d1f-9a2c-5e6b7d8f9012",
  "call_id": "wacid.HBgMOTE5MDAwMDAwMDAwFQIAERgSQTBGOEQ3MTJGM0EyRDFDNQA=",
  "direction": "BUSINESS_INITIATED",
  "status": "COMPLETED",
  "from_wa_id": "1234567890",
  "to_number": "919000000000",
  "duration_seconds": 96,
  "cost_credits": 4.8,
  "voice_session_id": "e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b",
  "permission_status": "permanent",
  "created_at": "2026-04-19T12:31:08Z",
  "ai_cost_credits": 2.1374,
  "transcript": [
    {
      "turn_index": 0,
      "user_transcript": null,
      "agent_response": "Hi, this is Acme Coffee calling about order AC-10294.",
      "interrupted": false
    },
    {
      "turn_index": 1,
      "user_transcript": "Yes, go ahead.",
      "agent_response": "We could not verify the delivery address. Is flat 4B still correct?",
      "interrupted": false
    }
  ]
}
```

Every field from the list response, plus:

| Field | Type | Notes |
|---|---|---|
| `transcript[].turn_index` | integer | Turn order, starting at 0 |
| `transcript[].user_transcript` | string, nullable | What the customer said |
| `transcript[].agent_response` | string, nullable | What the agent said |
| `transcript[].interrupted` | boolean | Whether the customer spoke over the agent |

`transcript` is empty until the agent has persisted turns, so it is normally empty while a call is still running. `404` if the call is not on your workspace.

### Terminate a live call

`POST /api/v1/whatsapp/calling/calls/{call_id}/terminate` · scope `whatsapp:send`

No body. Hangs up on WhatsApp's side and immediately marks the local row `TERMINATED`, so your call log updates without waiting for the webhook.

```bash
curl -X POST "https://api.callmissed.com/api/v1/whatsapp/calling/calls/wacid.HBgMOTE5MDAwMDAwMDAwFQIAERgSQTBGOEQ3MTJGM0EyRDFDNQA=/terminate" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{ "success": true }
```

`404` if the call is not on your workspace.
