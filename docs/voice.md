---
title: "Voice Calling"
description: "How an AI agent answers a PSTN call, and which integration path to use."
slug: "voice"
breadcrumb: "Numbers (PSTN)"
---

# Voice Calling

How an AI agent answers a PSTN call, and which integration path to use.

An AI voice agent answers an inbound phone call, transcribes the caller, generates a reply and speaks it back — with barge-in, so the caller can interrupt mid-sentence.

There are two ways to get a phone number onto an agent. Both run the same voice pipeline.

## Choose a path

**You want us to supply the number.** Complete KYC, rent an Indian number, and bind it to a bot — all over the API with your `cm_` key. See [Telephony API](/docs/telephony-api).

**You already own numbers.** Connect a carrier account you control (Twilio, Plivo, or any SIP provider), import your existing numbers, and point them at an agent. See [Bring Your Own Telephony](/docs/bring-your-own-telephony).

Moving an existing deployment across? [Migrate from Twilio](/docs/migrate-from-twilio) covers the number-by-number cutover.

## How a call actually runs

Both paths converge on the same flow:

```
Inbound PSTN call
  → your carrier's SIP trunk
  → CallMissed SIP endpoint (per-tenant inbound trunk)
  → a room is created and a voice agent is dispatched into it
  → the agent runs STT → LLM → TTS on the live audio
  → speech is streamed back to the caller
```

The agent pipeline is the same one WebRTC voice sessions use, so an agent you have already tuned in the dashboard behaves identically on a phone call. Turn-taking, interruption handling and the model stack all carry over.

Speaking style is per agent, not per call: pick the STT, LLM and TTS on the agent's **Voice** page, and tune the speech knobs your chosen TTS exposes on its **Speech** page.

## Outbound calling

Outbound is available through the [Telephony API](/docs/telephony-api) — place a call, attach an agent, and fetch the recording afterwards.

## Legacy: the TwiML media-stream route

:::warning
**Not implemented — do not build against this.**

Earlier versions of these docs described a Twilio TwiML route that opened
`wss://api.callmissed.com/ws/call/{call_id}` and ran a streaming
STT → LLM → TTS pipeline over it. **That pipeline was never completed.** The
WebSocket accepts audio and discards it: nothing is transcribed, no reply is
generated, and no audio is sent back. A number pointed at
`/api/v1/webhooks/twilio/voice` plays a hold message and then goes silent for
the rest of the call.

It is documented here only so that anyone who wired it up from the old
instructions knows why their calls were quiet. Use one of the two supported
paths above instead.
:::
