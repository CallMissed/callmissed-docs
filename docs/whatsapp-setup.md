---
title: "Business Setup"
description: "Connect a Meta WhatsApp Business Account to CallMissed, register the number, link an agent, and let AI write the first system prompt and templates."
slug: "whatsapp-setup"
breadcrumb: "WhatsApp"
---

# Business Setup

Connect a Meta WhatsApp Business Account to CallMissed, register the number, link an agent, and let AI write the first system prompt and templates.

Connecting a number binds your **WhatsApp Business Account (WABA)** to CallMissed, subscribes CallMissed to the WABA's webhooks, and registers the number on the WhatsApp Cloud API. After that, inbound messages flow to your agent and you can send from the API.

There are two ways to connect, and one thing you never have to do: **you do not configure a webhook in Meta**. Connecting subscribes the CallMissed app to your WABA automatically. See [the Meta-facing webhook](/docs/whatsapp-api#the-meta-facing-webhook) if you want to know what that endpoint is.

## Prerequisites

- A **Meta Business account** with a verified business.
- A **WhatsApp Business Account** and a phone number in [WhatsApp Manager](https://business.facebook.com/wa/manage/). The number must not be tied to a personal WhatsApp app.
- A payment method on the WABA in WhatsApp Manager. Until Meta has one, sends fail.
- A CallMissed workspace. The manual path additionally needs the **owner** or **admin** role.

## Option 1: connect from the dashboard

Go to **Settings → Integrations → WhatsApp** in the [dashboard](https://console.callmissed.com) and follow Embedded Signup. Meta's popup handles the account selection and consent, and CallMissed does the rest: exchanging the authorisation code, subscribing to webhooks, and registering the number with a two-step verification PIN.

This is the recommended path. It is also the only path that registers a brand-new number for you.

## Option 2: connect an existing WABA over the API

Use this when the number was set up outside Embedded Signup, for example registered directly in the Meta dashboard or migrated from another provider.

`POST /api/v1/whatsapp/onboarding/manual`

**Owner or admin dashboard login only.** This endpoint accepts a long-lived business token in the body, so an API key cannot call it: even a key with `whatsapp:write` gets `403`. Authenticate with a dashboard session JWT.

| Field | Type | Required | Notes |
|---|---|---|---|
| `waba_id` | string, 1 to 64 chars | Yes | Meta's WABA id |
| `phone_number_id` | string, 1 to 64 chars | Yes | Meta's phone number id, not the phone number itself |
| `access_token` | string, 20 to 4096 chars | Yes | A long-lived business token. A System User token is strongly recommended, since 24-hour tokens break delivery when they expire |
| `business_id` | string, max 64 | No | Meta business id |
| `bot_id` | UUID | No | Link an agent to the number in the same call |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/onboarding/manual \
  -H "Authorization: Bearer eyJhbGciOi...your-dashboard-session-jwt" \
  -H "Content-Type: application/json" \
  -d '{
    "waba_id": "102290129340398",
    "phone_number_id": "1234567890",
    "business_id": "441329482726",
    "access_token": "EAAG...long-lived-system-user-token",
    "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d"
  }'
```

The token is encrypted at rest. Meta's `register` step is skipped, because a number provisioned outside Embedded Signup is already registered and calling it again would fail and burn your registration quota. Webhook subscription is still attempted, so events flow.

**Response (200 OK)**

```json
{
  "account": {
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
  },
  "phone_number": {
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
  },
  "fully_provisioned": true,
  "onboarding_error": null
}
```

| Field | Type | Notes |
|---|---|---|
| `account` | object | The connected WABA |
| `phone_number` | object | The connected number. Its `id` is the `phone_id` you use everywhere else |
| `fully_provisioned` | boolean | `false` means the rows exist but a setup step did not complete. The connection is recoverable, so retry rather than starting over |
| `onboarding_error` | string, nullable | A short reason when `fully_provisioned` is `false` |

**Failures**

| Code | Meaning |
|---|---|
| `403` | Not an owner or admin, or called with an API key instead of a dashboard session |
| `404` | The `bot_id` does not belong to your workspace |
| `409` | The WABA or number is already connected to a different workspace |
| `422` | Meta rejected the token or the ids |

## Completing Embedded Signup yourself

If you are building your own Embedded Signup flow rather than using the dashboard, post Meta's callback data to:

`POST /api/v1/whatsapp/onboarding/exchange` · scope `whatsapp:write`

| Field | Type | Required | Notes |
|---|---|---|---|
| `code` | string, 1 to 2048 chars | Yes | The exchangeable code from Meta's login callback. It expires in about 30 seconds, so post it immediately |
| `waba_id` | string, 1 to 64 chars | Yes | From the signup event data |
| `phone_number_id` | string, 1 to 64 chars | Yes | From the signup event data |
| `business_id` | string, max 64 | No | From the signup event data |
| `bot_id` | UUID | No | Link an agent in the same call |
| `data_localization_region` | string, exactly 2 chars | No | ISO 3166-1 alpha-2 for data-at-rest residency, from Meta's supported list |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/onboarding/exchange \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "AQBx-hBsH...code-from-meta",
    "waba_id": "102290129340398",
    "phone_number_id": "1234567890",
    "business_id": "441329482726",
    "data_localization_region": "IN"
  }'
```

CallMissed exchanges the code for a business token, saves the account and number, subscribes to the WABA's webhooks, and registers the number with a two-step verification PIN it generates and stores. Returns the same object as the manual path.

The rows are saved before registration is attempted, so a failure at the last step leaves a recoverable connection rather than losing the WABA association. Check `fully_provisioned` and retry if it is `false`.

> **Reconnecting a number keeps its original PIN.** WhatsApp has no way to disable two-step verification, so a reconnect reuses the stored PIN. If it cannot be read, you get `409` asking you to reset the PIN in WhatsApp Manager first. Guessing would burn Meta's limit of registration attempts.

## Link an agent

A connected number stores inbound messages but stays silent until an agent is linked.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/link-bot \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d" }'
```

See [Phone numbers](/docs/whatsapp-api#phone-numbers) for linking, pausing auto-reply, refreshing metadata from Meta, disconnecting and deleting.

## Let AI write the first setup

Two endpoints turn a description of your business into a working agent. Both are billed to your workspace.

### Bootstrap the whole number

`POST /api/v1/whatsapp/ai/bootstrap` · scope `whatsapp:write`

Generates a system prompt, a persona, a welcome message and a set of starter templates from a plain-language description, and optionally applies them to the number's agent.

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone_number_id` | UUID | Yes | The **CallMissed** phone id, from `GET /phone_numbers` |
| `company_description` | string, 10 to 2000 chars | Yes | What the business does and how it wants to sound |
| `mode` | enum | No | `suggest`, `auto_apply` or `autonomous`. Default `auto_apply` |
| `language` | string, 2 to 8 chars | No | Default `en` |

`suggest` changes nothing and returns the plan for review. `auto_apply` and `autonomous` write the system prompt and create the templates.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/ai/bootstrap \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
    "company_description": "Acme Coffee is an Indian D2C roastery. We sell single-origin beans by subscription, ship in 2 to 3 days, and get asked about order status, roast dates and returns.",
    "mode": "suggest",
    "language": "en"
  }'
```

**Response (200 OK)**

```json
{
  "bootstrap_id": null,
  "mode": "suggest",
  "applied": false,
  "system_prompt": "You are the support agent for Acme Coffee, an Indian D2C roastery...",
  "persona": {
    "name": "Acme Coffee Support",
    "tone": "warm and concise",
    "signature": "Acme Coffee"
  },
  "welcome_message": "Hi, this is Acme Coffee. Ask me about an order, a roast date, or a return.",
  "templates": [
    {
      "name": "order_shipped",
      "category": "UTILITY",
      "body": "Hi {{1}}, order {{2}} shipped today and should arrive in 2 to 3 days.",
      "applied": false,
      "template_id": null
    }
  ]
}
```

| Field | Type | Notes |
|---|---|---|
| `bootstrap_id` | UUID, nullable | Present when the plan was applied. Pass it to undo |
| `mode` | string | Echoes the requested mode |
| `applied` | boolean | Whether the plan was written to the agent |
| `system_prompt` | string | The generated prompt |
| `persona` | object | `name`, `tone` and `signature` |
| `welcome_message` | string | Suggested opening line |
| `templates[]` | array | Each with `name`, `category`, `body`, `applied` and `template_id` when created |

`400` when the plan cannot be applied, for example the number does not exist. `422` when the model cannot produce a usable plan.

### Undo a bootstrap

`POST /api/v1/whatsapp/ai/bootstrap/{bootstrap_id}/undo` · scope `whatsapp:write`

No body. Restores the agent's previous system prompt and removes the still-pending templates the bootstrap created, within a 7-day window. Templates Meta has already approved are kept, because deleting them would break sends already relying on them.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/ai/bootstrap/7b3c9d10-2e4f-4a5b-8c6d-9e0f1a2b3c4d/undo \
  -H "Authorization: Bearer cm_your_api_key"
```

**Response (200 OK)**

```json
{
  "undone": true,
  "reverted_at": "2026-04-19T12:44:03+00:00",
  "reason": "ok"
}
```

| Field | Type | Notes |
|---|---|---|
| `undone` | boolean | Whether anything was reverted |
| `reverted_at` | string, nullable | ISO 8601 timestamp of the revert, `null` when nothing was reverted |
| `reason` | string | `ok` on success. Otherwise `bootstrap_id not found` or `undo window expired` |

A refusal is still a `200`: read `undone`, not the status code.

### Write or improve a system prompt

`POST /api/v1/whatsapp/ai/build_system_prompt` · scope `whatsapp:read`

Read-only. Returns markdown you review and save on the agent yourself.

| Field | Type | Required | Notes |
|---|---|---|---|
| `intent` | string, 10 to 8000 chars | Yes | The business and what the agent should do |
| `language` | string, 2 to 8 chars | No | Default `en` |
| `tone` | string, max 40 | No | For example `friendly`, `formal` |
| `existing_prompt` | string, max 8000 | No | Set this to improve an existing prompt instead of starting fresh |

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/ai/build_system_prompt \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "intent": "Support agent for Acme Coffee. Handle order status, roast dates and returns. Escalate anything about a refund over 5000 rupees to a human.",
    "tone": "warm and concise"
  }'
```

**Response (200 OK)**

```json
{
  "system_prompt": "## Role\nYou are the support agent for Acme Coffee...\n\n## Rules\n- Answer in under 60 words..."
}
```

`422` when the model cannot produce a usable prompt. Shorten or clarify the intent and retry.

To draft message templates the same way, see [Draft a template with AI](/docs/whatsapp-templates#draft-a-template-with-ai).

## Verify it works

Message your business number from a personal WhatsApp account. Then:

1. `GET /api/v1/whatsapp/webhook_events` shows the raw event arriving with `signature_valid: true`.
2. Your own webhook subscription receives a `message.received` event. See [Inbound events](/docs/whatsapp-api#inbound-events-you-receive).
3. The agent replies, and the thread appears in the dashboard inbox.

If the message arrives but nothing replies, walk the [auto-reply checklist](/docs/whatsapp#when-the-bot-replies). The usual cause is a number that was never linked to a bot.

> **Going to production.** While your number is in Meta's test mode you can only message numbers you have added as recipients. Submit your business for verification and request production access in the Meta dashboard to message any customer who opts in. Sends also fail until Meta approves a display name for the number, which shows as `name_status` on the phone-number object.
