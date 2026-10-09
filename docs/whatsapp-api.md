---
title: "WhatsApp API"
description: "Base path, authentication scopes, error shapes, connected accounts and numbers, inbound webhook events, and delivery analytics."
slug: "whatsapp-api"
breadcrumb: "WhatsApp"
---

# WhatsApp API

Base path, authentication scopes, error shapes, connected accounts and numbers, inbound webhook events, and delivery analytics.

The WhatsApp API is the programmatic surface for a connected **WhatsApp Business Account (WABA)**. This page covers the parts every other WhatsApp page depends on: authentication, how you pick a sending number, what an error looks like, how to read your connected accounts and numbers, what CallMissed pushes to your own webhook, and the delivery analytics.

**Base URL:** `https://api.callmissed.com`
**Base path:** `/api/v1/whatsapp`

| Area | Page |
|---|---|
| Connect a WABA and register a number | [Business Setup](/docs/whatsapp-setup) |
| Number health, registration, two-step PIN, business profile, blocked users, QR codes | [Number Management](/docs/whatsapp-numbers) |
| Send text, template, media, interactive, location, reaction, contacts | [Sending Messages](/docs/whatsapp-messages) |
| Create, list, delete and sync templates | [Message Templates](/docs/whatsapp-templates) |
| Bulk template sends | [Campaigns](/docs/whatsapp-campaigns) |
| Voice calls over WhatsApp | [Calling](/docs/whatsapp-calling) |

## Authentication

Every endpoint accepts either a `cm_` API key or a dashboard JWT:

```
Authorization: Bearer cm_your_api_key
```

API keys are checked against three WhatsApp scopes. A JWT session is authorized by role instead, so scopes do not apply to it.

| Scope | Grants |
|---|---|
| `whatsapp:read` | List accounts, numbers, templates, campaigns, calls, webhook events, analytics; resolve and download media |
| `whatsapp:write` | Onboard and manage numbers, link bots, template create/delete/sync, campaign create/launch/cancel, upload media, calling settings |
| `whatsapp:send` | Send any message type, mark as read, request call permission, place and terminate calls |

A key without the scope gets `403`:

```json
{
  "detail": "API key missing required scope: whatsapp:send. Add it under the key's Permissions in your dashboard."
}
```

Add scopes when you create the key. See [API Keys](/docs/keys).

## Choosing the sending number

Every endpoint that acts on a specific number needs to know which one. Supply **exactly one** of these. They are interchangeable, and both are always tenant-scoped, so you can only ever act on a number your workspace owns.

| Field | Type | Where it comes from |
|---|---|---|
| `phone_id` | UUID | The `id` field from `GET /phone_numbers` (CallMissed's id) |
| `phone_number_id` | string, max 64 | Meta's `phone_number_id` for the same number |

On send endpoints they go in the JSON body. On `GET` endpoints they are query parameters. On `POST /media` they are multipart form fields.

Omitting both returns `400`:

```json
{ "detail": "Either phone_id (UUID) or phone_number_id (Meta) is required" }
```

## Error shape

Every error is a single JSON object with a `detail` string:

```json
{ "detail": "The 24-hour customer service window is closed. Send a template message instead, or wait for the user to message you." }
```

The one exception is request-body validation, where `detail` is an array of field-level errors instead of a string:

```json
{
  "detail": [
    {
      "type": "string_too_short",
      "loc": ["body", "to"],
      "msg": "String should have at least 5 characters",
      "input": "+91"
    }
  ]
}
```

Upstream WhatsApp errors are never echoed verbatim. They are mapped to a short, actionable message and a status code that tells you whether to retry.

### Platform errors

These come from CallMissed before any WhatsApp call is made.

| Code | Meaning | Fix |
|---|---|---|
| `400` | Neither `phone_id` nor `phone_number_id` supplied, or a variant-specific field is missing | Add the missing field |
| `401` | Missing, malformed or expired credentials | Check the `Authorization` header |
| `402` | Not enough credits to pay for the send or campaign, or a workspace budget cap would be exceeded. Nothing was sent and nothing was charged | Top up, or raise the cap |
| `403` | API key is missing the required WhatsApp scope, or the workspace is inactive | Add the scope to the key |
| `404` | The number, template, campaign or call does not exist on your workspace | Verify the id |
| `409` | The number is disconnected, or has no stored access token. Or `recipient_opted_out`: the recipient opted out of your WhatsApp messages, see [Opt-outs](/docs/whatsapp-messages#opt-outs) | Reconnect the number. Do not retry an opted-out recipient |
| `422` | Request body failed validation | Read the `loc` path in `detail` |
| `429` | Rate limit exceeded. Media upload is limited more tightly than other calls | Back off and retry |
| `503` | Stored credentials could not be decrypted on this server | Contact support |

A `404` is deliberately identical whether the resource does not exist or belongs to another workspace, so the API cannot be used to probe for ids.

### Running out of credits

Sends, campaign launches and outbound calls are checked against your balance **before** WhatsApp is called, because the actual charge lands after delivery and there is nothing to un-send. The refusal is a `402` that tells you the shortfall:

```json
{
  "detail": "Not enough credits to send this message, so nothing was sent and nothing was charged. It needs at least 7.51 credits and you have 2.00 spendable (balance 12.00, 10.00 held for running campaigns) -- short by 5.51. Top up your balance and try again."
}
```

A separate `402` covers a self-imposed monthly budget cap, where the fix is raising the cap rather than topping up. Credits held for a running campaign or an in-flight call are reserved, not spent, and are released when the work settles.

### WhatsApp errors

Errors returned by Meta are translated. These are the mappings you will actually hit:

| Code | When | Retryable |
|---|---|---|
| `400` | Media MIME type does not match the file, or the request was rejected outright | No, fix the payload |
| `401` | The number's Meta access token is invalid or expired | No, reconnect the number |
| `403` | The app lacks advanced access for this WABA | No, contact support |
| `409` | Number is not registered on the WhatsApp Business Platform, was recently deleted, or calling is not enabled on it | No, finish the setup step named in `detail` |
| `413` | Media file exceeds 100 MB | No, shrink the file |
| `422` | 24-hour window closed, display name not yet approved by Meta, no call permission from the user, or a generic WhatsApp rejection | No, follow the instruction in `detail` |
| `429` | Per-user-pair send rate limit, or too many registration attempts in a short window | Yes, with backoff |
| `502` | WhatsApp returned a server error | Yes |
| `503` | The WhatsApp integration is not configured on this server | No, contact support |

## Accounts

A WABA is the Meta-side container for your numbers and templates.

### List accounts

`GET /api/v1/whatsapp/accounts` · scope `whatsapp:read`

No parameters. Returns every WABA on your workspace, newest first.

```bash
curl https://api.callmissed.com/api/v1/whatsapp/accounts \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
[
  {
    "id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
    "waba_id": "102290129340398",
    "business_id": "441329482726",
    "name": "Acme Coffee",
    "currency": "INR",
    "review_status": "APPROVED",
    "account_status": "ACTIVE",
    "account_restriction_reason": null,
    "payment_setup_complete": true,
    "is_active": true,
    "created_at": "2026-04-19T12:00:00Z"
  }
]
```

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | CallMissed's account id, used as `account_id` on template endpoints |
| `waba_id` | string | Meta's WABA id |
| `business_id` | string, nullable | Meta business id |
| `name` | string, nullable | WABA display name, filled in from Meta |
| `currency` | string, nullable | ISO 4217, the currency Meta bills the WABA in |
| `review_status` | string | Meta's business verification state |
| `account_status` | string | `ACTIVE` while healthy |
| `account_restriction_reason` | string, nullable | Set when Meta restricts the account, null otherwise |
| `payment_setup_complete` | boolean | `false` means Meta has no payment method on the WABA and sends will fail |
| `is_active` | boolean | Whether the account is live on your workspace |
| `created_at` | datetime | ISO 8601 UTC |

### Refresh an account from Meta

`POST /api/v1/whatsapp/accounts/{account_id}/refresh` · scope `whatsapp:write`

No body. Re-reads the WABA from Meta and updates `name`, `review_status` and `payment_setup_complete`. Meta sends no event when a payment method is added, so call this after adding one in WhatsApp Manager. A delivered paid message also sets `payment_setup_complete` to `true`, and a send that fails for a payment reason sets it back to `false`.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/accounts/1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d/refresh \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns the updated account object. `404` if the account is not on your workspace, `400` if no number on it has a stored token (reconnect one), `422` if Meta rejected the read, `502` if Meta failed.

### Delete an account

`DELETE /api/v1/whatsapp/accounts/{account_id}` · scope `whatsapp:write`

Permanently removes the WABA and everything under it: phone numbers, templates, campaigns and call records all cascade. Conversation history is preserved. CallMissed first tries to deregister each number and unsubscribe from the WABA's webhooks upstream, then deletes locally whether or not that succeeded.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/whatsapp/accounts/1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{ "ok": true, "error": null }
```

`error` is a short string when the upstream cleanup partly failed. The local delete still happened.

## Phone numbers

### List numbers

`GET /api/v1/whatsapp/phone_numbers` · scope `whatsapp:read`

No parameters. Newest first.

```bash
curl https://api.callmissed.com/api/v1/whatsapp/phone_numbers \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
[
  {
    "id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
    "account_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
    "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
    "phone_number_id": "1234567890",
    "display_phone_number": "+91 80802 47309",
    "verified_name": "Acme Coffee",
    "code_verification_status": "VERIFIED",
    "quality_rating": "GREEN",
    "messaging_limit_tier": "TIER_1K",
    "throughput_level": "STANDARD",
    "registration_status": "REGISTERED",
    "registration_error": null,
    "name_status": "APPROVED",
    "ai_autoreply_enabled": true,
    "is_active": true,
    "created_at": "2026-04-19T12:00:00Z"
  }
]
```

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Use as `phone_id` anywhere a sending number is needed |
| `account_id` | UUID | The owning WABA |
| `bot_id` | UUID, nullable | The linked agent. `null` means no auto-reply |
| `phone_number_id` | string | Meta's id for the number |
| `display_phone_number` | string | Human-readable number |
| `verified_name` | string, nullable | The business name shown to customers |
| `code_verification_status` | string | Meta's number verification state |
| `quality_rating` | string | `GREEN`, `YELLOW`, `RED` or `UNKNOWN` |
| `messaging_limit_tier` | string | Meta's 24-hour send cap tier, for example `TIER_1K` |
| `throughput_level` | string | Meta's throughput class, `STANDARD` by default |
| `registration_status` | string | `PENDING`, `REGISTERED`, `FAILED` or `DEREGISTERED` |
| `registration_error` | string, nullable | Why registration failed |
| `name_status` | string | Display-name approval. `APPROVED` or `AVAILABLE_WITHOUT_REVIEW` means sends work. Anything else blocks free-form sends |
| `ai_autoreply_enabled` | boolean | Master AI auto-reply switch for the number |
| `is_active` | boolean | `false` after a disconnect |
| `created_at` | datetime | ISO 8601 UTC |

### Get one number

`GET /api/v1/whatsapp/phone_numbers/{phone_id}` · scope `whatsapp:read`

Same object as a list element. `404` if the number is not on your workspace.

```bash
curl https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d \
  -H "Authorization: Bearer cm_your_api_key"
```

### Refresh metadata from Meta

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/refresh` · scope `whatsapp:write`

No body. Re-pulls live values from Meta and updates `display_phone_number`, `verified_name`, `code_verification_status`, `quality_rating`, `messaging_limit_tier`, `throughput_level` and `name_status`. Use it when a freshly onboarded number is still missing its display number, or after Meta approves a display name or raises your tier.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/refresh \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns the updated phone-number object. `400` if the number has no stored token (reconnect it), `422` if Meta rejected the read, `502` if Meta failed.

### Link or unlink a bot

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/link-bot` · scope `whatsapp:write`

This is what turns a number into an AI agent. Without a link the number stores inbound messages and never replies.

| Field | Type | Required | Notes |
|---|---|---|---|
| `bot_id` | UUID or null | Yes | The bot to link. Pass `null` to unlink |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/link-bot \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d" }'
```

Returns the updated phone-number object. `404` if the bot does not belong to your workspace.

### Pause or resume auto-reply

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/autoreply` · scope `whatsapp:write`

Master switch for the whole number. Set `false` and the agent stays silent on every conversation on that number until you set it back. Messages are still received and stored.

| Field | Type | Required | Notes |
|---|---|---|---|
| `enabled` | boolean | Yes | `false` pauses the agent for this number |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/autoreply \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "enabled": false }'
```

Returns the updated phone-number object.

### Conversational automation

In-chat guidance on a number: **ice breakers**, the tappable openers a customer sees before they have said anything, and **commands**, the slash-command hints that appear while they type. Both are per number.

`GET /api/v1/whatsapp/phone_numbers/{phone_id}/conversational_automation` · scope `whatsapp:read`

```bash
curl https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/conversational_automation \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "enable_welcome_message": true,
  "commands": [
    { "command_name": "track", "command_description": "Track your latest order" },
    { "command_name": "invoice", "command_description": "Get an invoice by order id" }
  ],
  "prompts": ["Track my order", "Talk to a human", "Store timings"]
}
```

A number that has never been configured returns empty lists rather than an error.

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/conversational_automation` · scope `whatsapp:write`

| Field | Type | Required | Notes |
|---|---|---|---|
| `commands` | array of objects, max 30 | No | Each is `{ "command_name": string (1-32), "command_description": string (1-256) }` |
| `prompts` | array of strings, max 4 | No | The ice breakers. Each max 80 characters |
| `enable_welcome_message` | boolean | No | Show a welcome message on a brand-new chat. Forwarded to WhatsApp only when you set it explicitly, and never defaulted |

Every field is optional and updates are partial, so posting only `commands` leaves your ice breakers untouched, and vice versa. A field you do send replaces the whole list, so send the full set you want, not just the additions.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/conversational_automation \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "prompts": ["Track my order", "Talk to a human", "Store timings"],
    "commands": [
      { "command_name": "track", "command_description": "Track your latest order" },
      { "command_name": "invoice", "command_description": "Get an invoice by order id" }
    ],
    "enable_welcome_message": true
  }'
```

Returns the configuration as WhatsApp reports it after the update, in the same shape as the read. `422` when a cap is exceeded: more than 30 commands, more than 4 ice breakers, an ice breaker over 80 characters, a `command_name` over 32, or a `command_description` over 256.

A tapped ice breaker or command arrives as an ordinary inbound text message, so your agent answers it with no extra wiring. Point the copy at things the agent can actually do.

### Disconnect a number

`POST /api/v1/whatsapp/phone_numbers/{phone_id}/disconnect` · scope `whatsapp:write`

No body. Reversible teardown: CallMissed tries to deregister the number and unsubscribe from the WABA's webhooks, then always marks the local row `is_active: false` with `registration_status: "DEREGISTERED"`, even if the upstream calls failed. Reconnect later by onboarding the number again.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/disconnect \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{ "ok": true, "error": null }
```

### Delete a number

`DELETE /api/v1/whatsapp/phone_numbers/{phone_id}` · scope `whatsapp:write`

Permanent, unlike disconnect. Best-effort deregister upstream first, then the row is removed regardless. Campaigns and call records that reference the number cascade. Conversation history is preserved.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{ "ok": true, "error": null }
```

## Raw webhook events

`GET /api/v1/whatsapp/webhook_events` · scope `whatsapp:read`

An audit peek at the raw events Meta delivered for your WABAs, newest first. Useful for debugging "did that message actually arrive". This is **not** the way to consume inbound messages, see [Inbound events you receive](#inbound-events-you-receive) for that.

| Param | Type | Default | Notes |
|---|---|---|---|
| `limit` | integer, 1 to 100 | 20 | Max rows |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/webhook_events?limit=20" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
[
  {
    "id": "b7d4e2f1-0a3c-4d5e-8f6a-9b0c1d2e3f4a",
    "received_at": "2026-04-19T12:04:11Z",
    "signature_valid": true,
    "event_type": "messages",
    "waba_id": "102290129340398",
    "processed": true,
    "process_error": null,
    "sender_wa_id": "919000000000",
    "body_preview": "Where is my order AC-10294?"
  }
]
```

Full raw payloads are deliberately not exposed here, because they carry customer message bodies and phone numbers. `event_type` mirrors Meta's webhook field name, for example `messages`, `message_template_status_update`, `account_update`, `phone_number_quality_update` or `calls`.

## You never configure a webhook in Meta

Meta delivers your WABA's events to CallMissed, not to you. Connecting a number subscribes the CallMissed app to the WABA's webhooks automatically, so there is nothing to set up in Meta's dashboard. Each delivery is signature-verified before it is processed. You see what arrived through [`GET /webhook_events`](#raw-webhook-events), and you consume the messages themselves through your own subscription below.

## Inbound events you receive

You never poll for inbound messages. Register an HTTPS endpoint and CallMissed pushes to it.

### Subscribe

```bash
curl -X POST https://api.callmissed.com/api/v1/webhooks \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-app.example.com/hooks/callmissed",
    "events": ["message.received"]
  }'
```

`message.received` is the event the WhatsApp channel emits for inbound messages; `message.status` reports the progress of messages you send (see [Delivery status](#delivery-status-and-proof-of-service)). See [Webhooks](/docs/webhooks) for the full catalogue of event types across the platform, delivery retries and replay.

### The request you receive

```
POST /hooks/callmissed HTTP/1.1
Content-Type: application/json
X-CallMissed-Event: message.received
X-CallMissed-Signature: sha256=9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
X-CallMissed-Delivery: 4c8d2e10-6b7a-4f3d-9e21-0a5b6c7d8e9f
```

```json
{
  "event": "message.received",
  "data": {
    "conversation_id": "2f6c9a11-3b4d-4e5f-8a9b-0c1d2e3f4a5b",
    "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
    "channel": "whatsapp",
    "from": "919000000000",
    "message_id": "wamid.HBgMOTE5MDAwMDAwMDAwFQIAEhggQjc0RTI5RDNBMjJDNjE4RgA=",
    "type": "text",
    "text": "Where is my order AC-10294?"
  },
  "timestamp": "2026-04-19T12:04:11.512340+00:00"
}
```

| Field | Type | Notes |
|---|---|---|
| `event` | string | Always `message.received` for this subscription |
| `timestamp` | string | ISO 8601 UTC, when the event was dispatched |
| `data.conversation_id` | UUID | The CallMissed conversation thread |
| `data.bot_id` | UUID | The bot that owns the conversation |
| `data.channel` | string | `whatsapp` |
| `data.from` | string | The customer's WhatsApp id, digits only, no `+` |
| `data.message_id` | string | Meta's `wamid` for the inbound message |
| `data.type` | string | WhatsApp message type, for example `text`, `image`, `audio`, `interactive`, `button`, `location` |
| `data.text` | string, nullable | Body text. Null for non-text types |

### Verify the signature

`X-CallMissed-Signature` is `sha256=` followed by the hex HMAC-SHA256 of the **raw request body**, keyed with the webhook's secret. Compare in constant time and reject a mismatch.

```python
import hashlib, hmac

def verify(raw_body: bytes, header: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", header or "")
```

```javascript
import crypto from "node:crypto";

function verify(rawBody, header, secret) {
  const expected =
    "sha256=" + crypto.createHmac("sha256", secret).update(rawBody).digest("hex");
  return (
    header?.length === expected.length &&
    crypto.timingSafeEqual(Buffer.from(header), Buffer.from(expected))
  );
}
```

Return `2xx` quickly. Do your own work after acknowledging.

## Delivery status and proof of service

Every message you send gets a `wamid`, returned as `wamid` (and `wamids` when a long text is split into several bubbles) by the [send endpoints](/docs/whatsapp-messages). As WhatsApp reports progress on that message, CallMissed records each status with WhatsApp's own timestamp and, on failure, WhatsApp's error code. You can receive the statuses as a webhook, look them up by `wamid`, or both. These records are kept for the life of your account ([Data Residency & Retention](/docs/data-residency)).

| Status | Meaning |
|---|---|
| `sent` | WhatsApp accepted the message and sent it toward the recipient |
| `delivered` | The message reached the recipient's device |
| `read` | The recipient opened it. Only reported when the recipient has read receipts on |
| `failed` | WhatsApp could not deliver it. `error` carries the code and reason |

Statuses can arrive out of order, for example `read` before `delivered`, and not every status always arrives. A `read` with no `delivered` leaves `delivered_at` null. CallMissed never invents a timestamp WhatsApp did not report.

### The `message.status` webhook

Subscribe to `message.status` on the same [webhook](/docs/webhooks) subscription as `message.received`:

```bash
curl -X POST https://api.callmissed.com/api/v1/webhooks \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-app.example.com/hooks/callmissed",
    "events": ["message.received", "message.status"]
  }'
```

It fires **once per message per status**. WhatsApp sometimes reports the same status more than once, and those repeats are dropped. The request is signed exactly like `message.received` ([verify the signature](#verify-the-signature)).

```json
{
  "event": "message.status",
  "data": {
    "message_id": "wamid.HBgMOTE5MDAwMDAwMDAwFQIAERgSQjc0RTI5RDNBMjJDNjE4RgA=",
    "status": "failed",
    "timestamp": "2026-04-19T12:04:15+00:00",
    "recipient": "919000000000",
    "phone_number_id": "106540352242922",
    "conversation_id": "2f6c9a11-3b4d-4e5f-8a9b-0c1d2e3f4a5b",
    "campaign_id": null,
    "error": {
      "code": 131026,
      "title": "Message undeliverable",
      "message": "Message undeliverable",
      "details": "Message Undeliverable."
    },
    "pricing": null
  },
  "timestamp": "2026-04-19T12:04:16.204113+00:00"
}
```

| Field | Type | Notes |
|---|---|---|
| `data.message_id` | string | The `wamid` the send endpoint returned |
| `data.status` | string | `sent`, `delivered`, `read` or `failed` |
| `data.timestamp` | string | When WhatsApp says the status happened, ISO 8601 UTC. The outer `timestamp` is when CallMissed dispatched the webhook |
| `data.recipient` | string | The recipient's WhatsApp id, digits only, no `+` |
| `data.phone_number_id` | string | Your sending number's Meta phone number id |
| `data.conversation_id` | UUID, nullable | The CallMissed conversation, when the message is stored in one |
| `data.campaign_id` | UUID, nullable | Set when the message was sent by a [campaign](/docs/whatsapp-campaigns) |
| `data.error` | object, nullable | Only on `failed`: WhatsApp's `code`, `title`, `message` and `details`. Look the code up in Meta's [error code reference](https://developers.facebook.com/docs/whatsapp/cloud-api/support/error-codes) |
| `data.pricing` | object, nullable | WhatsApp's pricing info for the message when it reports it: `billable`, `type` and `category` |

### Look up a message

`GET /api/v1/whatsapp/messages/{wamid}` · scope `whatsapp:read`

Returns the current status of one message and every status recorded for it. A `wamid` that does not belong to your workspace returns `404`.

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/messages/wamid.HBgMOTE5MDAwMDAwMDAwFQIAERgSQjc0RTI5RDNBMjJDNjE4RgA=" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "wamid": "wamid.HBgMOTE5MDAwMDAwMDAwFQIAERgSQjc0RTI5RDNBMjJDNjE4RgA=",
  "direction": "outbound",
  "type": "template",
  "to": "919000000000",
  "from": null,
  "status": "read",
  "created_at": "2026-04-19T12:04:10.881020+00:00",
  "sent_at": "2026-04-19T12:04:11+00:00",
  "delivered_at": "2026-04-19T12:04:12+00:00",
  "read_at": "2026-04-19T12:09:40+00:00",
  "failed_at": null,
  "error": null,
  "conversation_id": "2f6c9a11-3b4d-4e5f-8a9b-0c1d2e3f4a5b",
  "campaign_id": null,
  "statuses": [
    { "status": "sent", "timestamp": "2026-04-19T12:04:11+00:00", "error": null },
    { "status": "delivered", "timestamp": "2026-04-19T12:04:12+00:00", "error": null },
    { "status": "read", "timestamp": "2026-04-19T12:09:40+00:00", "error": null }
  ]
}
```

| Field | Type | Notes |
|---|---|---|
| `direction` | string | `outbound` or `inbound` |
| `type` | string, nullable | Message type, when CallMissed stored the message |
| `to` / `from` | string, nullable | The other party's WhatsApp id, on the matching side |
| `status` | string | The furthest status reached: `sent`, `delivered` or `read`, or `failed` |
| `created_at` | string, nullable | When CallMissed accepted the send |
| `sent_at`, `delivered_at`, `read_at`, `failed_at` | string, nullable | WhatsApp's timestamp for each status. Null if that status was never reported |
| `error` | object, nullable | `{code, title}` from the failure |
| `statuses` | array | Every status recorded, oldest first. This is the audit trail: one entry per status, each with WhatsApp's timestamp and, for `failed`, the error |

Every send is recorded the same way, whether it went out through an AI agent, a direct API call or a campaign. A message that has not yet reported any status can return `404` for a few seconds after the send.

To audit a whole campaign, page through [`GET /campaigns/{id}/recipients`](/docs/whatsapp-campaigns#list-recipients).

## Analytics

Three read-only aggregations over the last N days from CallMissed's own delivery and billing records, plus [Meta insights](#meta-insights), a read-through to WhatsApp's own figures. All require `whatsapp:read`. On the first three, `days` is bounded to 1 to 90.

### Delivery funnel

`GET /api/v1/whatsapp/analytics/funnel` · scope `whatsapp:read`

| Param | Type | Default | Notes |
|---|---|---|---|
| `days` | integer, 1 to 90 | 7 | Window size |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/analytics/funnel?days=7" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "days": 7,
  "inbound": 412,
  "outbound": 508,
  "delivered": 341,
  "read": 146,
  "replied": 96,
  "delivery_rate": 0.671,
  "read_rate": 0.428
}
```

| Field | Type | Notes |
|---|---|---|
| `days` | integer | Echoes the window |
| `inbound` | integer | Inbound message events from customers |
| `outbound` | integer | Messages you sent: conversation replies plus campaign sends |
| `delivered` | integer | Real delivered acks, from message status plus campaign delivery counters |
| `read` | integer | Real read acks, from the same two sources |
| `replied` | integer | Conversations you sent at least one agent reply in during the window |
| `delivery_rate` | float | `delivered / outbound`, 3 decimals, clamped to `[0, 1]` |
| `read_rate` | float | `read / delivered`, 3 decimals, clamped to `[0, 1]` |

The counts come from real delivery data, not from a ratio. Where a figure genuinely cannot be sourced it stays `0` rather than being estimated. If no WABA is connected, every counter is `0`.

### Event time series

`GET /api/v1/whatsapp/analytics/timeseries` · scope `whatsapp:read`

One row per day and event type, ready to stack in a chart.

| Param | Type | Default | Notes |
|---|---|---|---|
| `days` | integer, 1 to 90 | 14 | Window size |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/analytics/timeseries?days=14" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "days": 14,
  "points": [
    { "date": "2026-04-18", "event_type": "messages", "count": 63 },
    { "date": "2026-04-18", "event_type": "message_template_status_update", "count": 2 },
    { "date": "2026-04-19", "event_type": "messages", "count": 71 }
  ]
}
```

| Field | Type | Notes |
|---|---|---|
| `points[].date` | string | `YYYY-MM-DD` in UTC |
| `points[].event_type` | string | Meta webhook field name, or `unknown` |
| `points[].count` | integer | Events that day |

### Cost breakdown

`GET /api/v1/whatsapp/analytics/costs` · scope `whatsapp:read`

| Param | Type | Default | Notes |
|---|---|---|---|
| `days` | integer, 1 to 90 | 30 | Window size |

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/analytics/costs?days=30" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "days": 30,
  "llm_tokens": 1284300,
  "llm_credits": 998.4412,
  "whatsapp_message_credits": 512.0,
  "whatsapp_call_credits": 43.75,
  "total_credits": 1554.1912,
  "ledger_source": "wa_usage_events"
}
```

| Field | Type | Notes |
|---|---|---|
| `days` | integer | Echoes the window |
| `llm_tokens` | integer | Tokens the agent consumed. Context only, it does not drive any credit figure |
| `llm_credits` | float | Real LLM spend, from the usage ledger |
| `whatsapp_message_credits` | float | Real WhatsApp message spend, priced per delivered message |
| `whatsapp_call_credits` | float | Real WhatsApp call spend |
| `total_credits` | float | Sum of the three credit figures |
| `ledger_source` | string | How the figures were sourced, so you can tell a per-event ledger match from an aggregate |

Every credit figure here traces to a per-event billing record, so it reconciles with what was actually deducted. For the wallet balance and the platform-wide usage feed, see [Credits & Rate Limits](/docs/credits-rate-limits).

### Meta insights

The three routes above count what CallMissed recorded. The seven routes below read **WhatsApp's own insights** live for one WhatsApp Business Account. They will not match the figures above exactly: WhatsApp describes its analytics as approximate, and they lag delivery. Nothing here is stored.

Cost figures are **not included** on any of these routes (`cost_included` is always `false`). For what you were charged, use [Cost breakdown](#cost-breakdown).

All seven need `whatsapp:read` and pick the account the same way as the template endpoints: pass exactly one of `account_id` (UUID, from `GET /accounts`) or `waba_id` (Meta's id, max 64 chars). Omitting both returns `400`, and a disconnected account returns `409`.

| Route | What it returns | `days` | `granularity` | Extra params |
|---|---|---|---|---|
| `GET /api/v1/whatsapp/analytics/meta/messaging` | Messages `sent` and `delivered` per bucket | 1 to 365, default 30 | `HALF_HOUR`, `DAY` (default), `MONTH` | |
| `GET /api/v1/whatsapp/analytics/meta/conversations` | `conversation` count per bucket | 1 to 365, default 30 | `HALF_HOUR`, `DAILY` (default), `MONTHLY` | `dimensions`: any of `CONVERSATION_CATEGORY`, `CONVERSATION_DIRECTION`, `CONVERSATION_TYPE`, `COUNTRY`, `PHONE` |
| `GET /api/v1/whatsapp/analytics/meta/pricing` | Billable message `volume` per bucket | 1 to 365, default 30 | `HALF_HOUR`, `DAILY` (default), `MONTHLY` | `dimensions`: any of `COUNTRY`, `PHONE`, `PRICING_CATEGORY`, `PRICING_TYPE`, `TIER`. Default `PRICING_CATEGORY` + `COUNTRY` |
| `GET /api/v1/whatsapp/analytics/meta/calls` | Call `count` and `average_duration` per bucket | 1 to 365, default 30 | `HALF_HOUR`, `DAILY` (default), `MONTHLY` | `directions`: `USER_INITIATED`, `BUSINESS_INITIATED` |
| `GET /api/v1/whatsapp/analytics/meta/templates` | Per template: `sent`, `delivered`, `read`, `clicked` | 1 to 90, default 30 | Fixed `DAILY` | `template_ids` (required, up to 10), `after` (cursor, max 512) |
| `GET /api/v1/whatsapp/analytics/meta/template-groups` | Per template group: `sent`, `delivered`, `read` | 1 to 90, default 30 | Fixed `DAILY` | `template_group_ids` (required, up to 10), `after` |
| `GET /api/v1/whatsapp/analytics/meta/groups` | Per WhatsApp group: `sent`, `delivered`, `read`, `joined`, `left` | 1 to 90, default 30 | Fixed `DAILY` | `group_ids` (required, exactly 1 today), `after` |

Note the granularity spellings: `messaging` takes `DAY` / `MONTH`, while every other surface takes `DAILY` / `MONTHLY`. A value the surface does not accept returns `400` naming the accepted set. `HALF_HOUR` windows are capped at 7 days. Repeat a list parameter to pass several values, for example `?dimensions=COUNTRY&dimensions=PHONE`.

```bash
curl "https://api.callmissed.com/api/v1/whatsapp/analytics/meta/pricing?account_id=1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d&days=7&granularity=DAILY&dimensions=PRICING_CATEGORY&dimensions=COUNTRY" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "source": "meta",
  "surface": "pricing_analytics",
  "granularity": "DAILY",
  "days": 7,
  "start": 1790553600,
  "end": 1791158400,
  "cost_included": false,
  "points": [
    {
      "start": 1790553600,
      "end": 1790640000,
      "volume": 184,
      "country": "IN",
      "tier": null,
      "pricing_type": "REGULAR",
      "pricing_category": "MARKETING"
    }
  ],
  "paging": null
}
```

| Field | Type | Notes |
|---|---|---|
| `source` | string | Always `meta` |
| `surface` | string | `analytics`, `conversation_analytics`, `pricing_analytics`, `call_analytics`, `template_analytics`, `template_group_analytics` or `group_analytics` |
| `granularity` | string | The bucket size used |
| `days` | integer | The window actually queried, ending at the next UTC midnight. Smaller than requested when a `HALF_HOUR` window was capped |
| `start`, `end` | integer | Window bounds as UNIX timestamps (UTC) |
| `cost_included` | boolean | Always `false` |
| `points` | object[] | One entry per bucket, with `start` / `end` plus the metrics and dimensions listed above. Metrics WhatsApp omits come back as `null` |
| `paging` | object, nullable | WhatsApp's cursors, on the `templates`, `template-groups` and `groups` surfaces. Pass `paging.cursors.after` back as `after` |

Template analytics needs a one-time insights opt-in on the account and is not available for accounts in the European Union or Japan; without them these surfaces return `422` saying so. Template `read` and `clicked` counts only exist for 7 days after a send, and clicks only for marketing and utility templates, so older days in a 30-day window can show sends with no reads. A WhatsApp analytics rate limit returns `429`; back off and retry.
