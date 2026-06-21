---
title: "Voice Calling"
description: "AI-powered inbound voice call agents via Twilio."
slug: "voice"
breadcrumb: "Channels"
---

# Voice Calling

AI-powered inbound voice call agents via Twilio.

## Setup

1. Create a bot with `type: "inbound_call"`
2. Configure Twilio credentials in Settings
3. Set your Twilio phone number webhook to:

```
https://api.callmissed.com/api/v1/webhooks/twilio/voice
```

## Call Flow

```
Incoming call → Twilio
  → POST /api/v1/webhooks/twilio/voice
  → Returns TwiML to open WebSocket stream
  → WebSocket /ws/call/{call_id} receives audio
  → Audio chunks → STT → text
  → Text → LLM → response
  → Response → TTS → audio
  → Audio streamed back to caller
```

## WebSocket Streaming

Connect to the voice WebSocket for real-time audio. Requires an API key:

```
wss://api.callmissed.com/ws/call/{call_id}?api_key=cm_your_key
```

The WebSocket implements a full STT → LLM → TTS pipeline:

1. **Audio in** — Twilio sends 8kHz mulaw audio chunks
2. **STT** — saaras:v3 transcribes in real-time
3. **LLM** — Generates response using bot's system prompt + conversation history
4. **TTS** — bulbul:v3 synthesizes speech (MP3 at 24kHz)
5. **Audio out** — Streamed back to the caller

LLM and TTS run concurrently for minimum latency — audio playback begins while the LLM is still generating.

## Outbound Calling

The `outbound_call` bot type exists, but a public API to **initiate** outbound calls is not yet available — today the voice pipeline is driven by inbound Twilio calls (and LiveKit voice sessions). Programmatic outbound dialing is on the roadmap. [Talk to us](/docs/talk-to-us) if you need it.