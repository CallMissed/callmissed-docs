---
title: "WhatsApp Business Setup"
description: "Connect your Meta WhatsApp Business account to CallMissed and deploy a bot in five steps."
slug: "whatsapp-setup"
breadcrumb: "Channels"
---

# WhatsApp Business Setup

Connect your Meta WhatsApp Business account to CallMissed and deploy a bot in five steps.

Connect a Meta (Facebook) WhatsApp Business number to CallMissed so an AI bot answers inbound messages automatically. CallMissed stores your Meta credentials per tenant, verifies the connection, and handles the inbound webhook → AI turn → reply loop.

## Prerequisites

- A **Meta Business account** with a verified business.
- A **WhatsApp Business Account (WABA)** and a phone number registered in the [Meta WhatsApp Manager](https://business.facebook.com/wa/manage/) — the number must not be tied to a personal WhatsApp app.
- An app created at [Meta for Developers](https://developers.facebook.com/) with the **WhatsApp** product added.
- A CallMissed account with the **owner** or **admin** role (only those roles can save integration settings).

## Get your Meta credentials

From your app in the Meta dashboard (**WhatsApp → API Setup**), collect:

- **Phone Number ID** — the ID of the sending number (not the phone number itself).
- **Access token** — generate a **permanent** token from a System User (**Business Settings → System Users → Generate token**) with the `whatsapp_business_messaging` and `whatsapp_business_management` permissions. Temporary tokens expire in 24h and will break delivery.
- **Verify token** — any string you choose (for example a random 32-char value). You will paste the same value into both CallMissed and the Meta webhook config.
- **App secret** (optional but recommended) — from **App Settings → Basic**. CallMissed uses it to validate the `X-Hub-Signature-256` on inbound webhooks.

## Save credentials in CallMissed

:::steps
## Open Integration Settings

In the [Dashboard](https://app.callmissed.com), go to **Settings → Integrations → WhatsApp**. (Programmatically this is `PUT /api/v1/settings/whatsapp`.)

## Enter your credentials

Provide the values you collected from Meta:

```bash
curl -X PUT https://api.callmissed.com/api/v1/settings/whatsapp \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "access_token": "EAAG...permanent-system-user-token",
    "verify_token": "my_chosen_verify_token",
    "app_secret": "your_meta_app_secret"
  }'
```

Stored secrets are write-only — reading the settings back returns masked values.

## Verify the connection

Run a live connectivity check against the Meta Graph API before you rely on the integration:

```bash
curl -X POST https://api.callmissed.com/api/v1/settings/whatsapp/verify \
  -H "Authorization: Bearer cm_your_key"
```

A success response confirms the access token and Phone Number ID are valid.
:::

## Configure the Meta webhook

In the Meta dashboard (**WhatsApp → Configuration → Webhook**), set:

- **Callback URL:**

```
https://api.callmissed.com/api/v1/webhooks/whatsapp
```

- **Verify token:** the exact same `verify_token` you saved in CallMissed.
- **Webhook fields:** subscribe to **messages**.

Meta sends a one-time GET handshake to the callback URL with your verify token. CallMissed echoes the challenge back when the tokens match:

:::flow
icon:user | Meta | Sends `GET /webhooks/whatsapp?hub.verify_token=…&hub.challenge=…`
icon:gateway | CallMissed | Compares the token to your saved `verify_token` and echoes `hub.challenge`
icon:done | Meta | Marks the webhook **Verified** — inbound messages now POST to the same URL
:::

## Create a WhatsApp bot

Create a bot with `type: "whatsapp"`, a system prompt for its persona, and any knowledge-base entries for FAQ/product context:

```bash
curl -X POST https://api.callmissed.com/api/v1/bots \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Support Bot",
    "type": "whatsapp",
    "system_prompt": "You are a helpful support agent for Acme Corp.",
    "config": { "phone_number_id": "1234567890", "language": "en" }
  }'
```

See the [WhatsApp Bot](/docs/whatsapp) guide for the full message flow and bot config reference.

## Test & go live

Send a WhatsApp message to your business number. The end-to-end path is:

:::flow
icon:user | Customer | Sends a WhatsApp message to your business number
icon:gateway | CallMissed | Verifies the signature, finds or creates the conversation, saves the message
icon:llm | LLM | Generates a reply from the system prompt + conversation history + knowledge base
icon:done | Customer | Receives the AI reply via the WhatsApp Cloud API
:::

> **Going to production:** while your number is in Meta's *test* mode you can only message numbers you have added as recipients. Submit your business for verification and request production access in the Meta dashboard to message any customer who opts in.