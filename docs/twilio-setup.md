---
title: "Twilio Voice Setup"
description: "Connect a Twilio voice number to CallMissed so an AI agent answers inbound calls in real time."
slug: "twilio-setup"
breadcrumb: "Channels"
---

# Twilio Voice Setup

Connect a Twilio voice number to CallMissed so an AI agent answers inbound calls in real time.

Connect a [Twilio](https://www.twilio.com/) voice number so an AI agent answers inbound calls — transcribing the caller, generating a reply, and speaking it back over a real-time audio stream. CallMissed stores your Twilio credentials per tenant and serves the TwiML that bridges the call to its streaming pipeline.

## Prerequisites

- A **Twilio account** (a trial account works for testing).
- A **voice-capable phone number** purchased in the [Twilio Console](https://console.twilio.com/) (**Phone Numbers → Manage → Buy a number**, with the *Voice* capability).
- A CallMissed account with the **owner** or **admin** role.

## Get your Twilio credentials

From the [Twilio Console](https://console.twilio.com/) home page, copy:

- **Account SID** — starts with `AC…`.
- **Auth Token** — click to reveal it under *Account Info*.
- **Phone Number** — your purchased number in **E.164** format (for example `+14155550123`).

## Save credentials in CallMissed

:::steps
## Open Integration Settings

In the [Dashboard](https://app.callmissed.com), go to **Settings → Integrations → Twilio**. (Programmatically this is `PUT /api/v1/settings/twilio`.)

## Enter your credentials

```bash
curl -X PUT https://api.callmissed.com/api/v1/settings/twilio \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "account_sid": "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "auth_token": "your_twilio_auth_token",
    "phone_number": "+14155550123"
  }'
```

The auth token is stored write-only and returned masked on reads.

## Verify the connection

Confirm the credentials are valid with a live check against the Twilio API:

```bash
curl -X POST https://api.callmissed.com/api/v1/settings/twilio/verify \
  -H "Authorization: Bearer cm_your_key"
```
:::

## Create a voice bot

Create a bot with `type: "inbound_call"` and a system prompt for the agent's persona:

```bash
curl -X POST https://api.callmissed.com/api/v1/bots \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Reception Agent",
    "type": "inbound_call",
    "system_prompt": "You are the front-desk agent for Acme Clinic. Be concise and friendly."
  }'
```

## Point your number at CallMissed

In the Twilio Console, open your number (**Phone Numbers → Manage → Active numbers → your number**). Under **Voice Configuration → A call comes in**, set:

- **Webhook**, HTTP **POST**, URL:

```
https://api.callmissed.com/api/v1/webhooks/twilio/voice
```

Save. Twilio will now POST to CallMissed on every inbound call, and CallMissed returns TwiML that opens a media stream into its STT → LLM → TTS pipeline.

## Test the call

Call your Twilio number. The real-time path is:

:::flow
icon:phone | Caller | Dials your Twilio number
icon:gateway | Twilio | POSTs to `/webhooks/twilio/voice`; CallMissed returns TwiML opening a media stream
icon:stt | STT (saaras:v3) | Transcribes the caller's audio in real time
icon:llm | LLM | Generates the reply from the bot's system prompt + conversation history
icon:tts | TTS (bulbul:v3) | Synthesizes speech — playback starts before generation finishes
icon:done | Caller | Hears the AI agent respond
:::

> **Tip:** For browser/mobile WebRTC agents (no phone number required) use the LiveKit-based [Voice Agent](/docs/voice-agent) and [Voice Sessions API](/docs/voice-sessions-api) instead. See the [Voice Calling](/docs/voice) guide for the full telephony protocol.