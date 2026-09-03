---
title: "Twilio Voice Setup"
description: "Connect a Twilio voice number to CallMissed so an AI agent answers inbound calls in real time."
slug: "twilio-setup"
breadcrumb: "Numbers (PSTN)"
---

# Twilio Voice Setup

Connect a Twilio voice number to CallMissed so an AI agent answers inbound calls in real time.

Connect a [Twilio](https://www.twilio.com/) voice number so an AI agent answers inbound calls — transcribing the caller, generating a reply, and speaking it back with barge-in support.

Twilio connects over **SIP trunking**. CallMissed provisions a trunk against your Twilio account and points it at a per-tenant SIP endpoint; inbound calls land in a room where a voice agent is dispatched to answer them.

## Set it up

The full walkthrough lives in **[Bring Your Own Telephony → Twilio](/docs/bring-your-own-telephony)**: which credentials to copy, what we provision on your account, and how to import your existing numbers.

You will need:

- A **Twilio account** (a trial account works for testing).
- A **voice-capable number** in the [Twilio Console](https://console.twilio.com/) (**Phone Numbers → Manage → Buy a number**, with the *Voice* capability).
- A CallMissed account with the **owner** or **admin** role.

Prefer not to manage a carrier account at all? [Rent a number from us](/docs/telephony-api) instead — KYC and provisioning are handled over the API with your `cm_` key.

## Create the agent that answers

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

Pick the STT, LLM and TTS models on the agent's **Voice** page in the dashboard, then tune the speech knobs your chosen TTS exposes on its **Speech** page. The same agent works over a phone call and over WebRTC, so anything you tune once applies to both.

## How the call runs

:::flow
icon:phone | Caller | Dials your Twilio number
icon:gateway | Twilio | Routes the call over your SIP trunk to CallMissed
icon:stt | STT | Transcribes the caller's audio in real time
icon:llm | LLM | Generates the reply from the bot's system prompt + conversation history
icon:tts | TTS | Synthesizes speech — playback starts before generation finishes
icon:done | Caller | Hears the AI agent respond, and can interrupt it
:::

The exact STT, LLM and TTS in that chain are whichever you selected on the agent — the defaults are Indian-language-first, and every option is listed under [Models](/docs/models).

## Not the TwiML webhook

:::warning
**Do not point your number at `/api/v1/webhooks/twilio/voice`.**

Earlier versions of this page told you to set that URL as the *A call comes in*
webhook. It returns TwiML that opens a media stream to
`wss://api.callmissed.com/ws/call/{call_id}`, and **that streaming pipeline was
never completed** — the socket accepts audio and discards it.

A number wired that way never works. The endpoint returns `503` unless a Twilio
auth token is configured; where one is, the call used to play a hold message and
then stay silent. It now says the number is not set up and hangs up.

If you configured it from the old instructions, that is why your test calls never
connected. Switch to the SIP setup linked above.
:::

> **Tip:** For browser/mobile WebRTC agents (no phone number required) use the [Voice Agent](/docs/voice-agent) and [Voice Sessions API](/docs/voice-sessions-api) instead. [Voice Calling](/docs/voice) compares the telephony paths.
