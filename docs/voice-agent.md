---
title: "Voice Agent"
description: "Real-time voice AI agent over WebRTC — native speech-to-speech with Nova 2 Sonic by default, plus STT→LLM→TTS fallback."
slug: "voice-agent"
breadcrumb: "Voice Agents"
---

# Voice Agent

Real-time voice AI agent over WebRTC — native speech-to-speech with Nova 2 Sonic by default, plus STT→LLM→TTS fallback.

:::cards
/docs/voice-sessions-api | Voice Sessions API | key | Create sessions and generate connection tokens
/docs/voice-sdk | Voice SDK | package | Client SDK for browser and mobile WebRTC
/docs/stt-realtime | Real-time STT | mic | Streaming speech-to-text over WebSocket
/docs/text-to-speech | Text to Speech | volume2 | Indic TTS for agent responses
:::

## Overview

The Voice Agent is a real-time conversational AI that streams over **WebRTC**. By default it uses a native speech-to-speech model:

```
Mic (WebRTC) → Nova 2 Sonic (speech-to-speech) → Speaker (WebRTC)
```

Nova 2 Sonic handles speech understanding, reasoning, turn-taking, function calling, and speech output in one model. If the speech-to-speech model is unavailable, the agent falls back to a cascaded STT→LLM→TTS stack automatically so sessions still connect.

**Default stack:**
- **LLM / voice:** `nova-sonic-2` (Amazon Nova 2 Sonic, speech-to-speech, 16 voices)
- **Fallback STT:** `saaras:v3` (streaming, 23 languages)
- **Fallback LLM:** `gpt-oss-120b` (fast no-think pipeline model)
- **Fallback TTS:** `bulbul:v3` (streaming, 37 voices)
- **Transport:** WebRTC (via the `livekit-client` SDK)

## Architecture

You create a session over REST and receive a connection URL + token. Your client connects with the `livekit-client` SDK; the CallMissed voice agent joins automatically and handles the speech pipeline. Audio flows over WebRTC.

This is the WebRTC path. For a plain WebSocket you stream raw audio to — no client SDK, no media hop — see the [Managed Voice Agent](/docs/managed-voice-agent), which runs the same tuned pipeline over `wss://api.callmissed.com`.

:::flow
icon:app | Browser (livekit-client SDK) | Captures mic audio and streams it over WebRTC
icon:server | Connection | WebRTC transport that connects your client to the voice agent
icon:bot | CallMissed voice agent | Runs Nova Sonic speech-to-speech, or falls back to the STT → LLM → TTS loop
icon:done | Browser | Receives synthesized speech back over WebRTC and plays it
:::

### One conversational turn

With Nova Sonic selected, every turn stays in one speech-to-speech model. With a cascaded model selected (or when the speech-to-speech model is unavailable), every turn streams through STT, LLM, and TTS concurrently to minimize time-to-first-audio:

:::flow
icon:stt | STT | Streams partial transcripts as the user speaks, finalizes on end-of-speech
icon:llm | LLM | Generates the reply at high throughput and pushes sentence chunks downstream
icon:tts | TTS | Synthesizes each sentence chunk as it arrives — playback starts before generation finishes
:::

## Quickstart

**1. Create a session:**

```bash
curl -X POST https://api.callmissed.com/v1/voice/sessions \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "system_prompt": "You are a helpful assistant.",
    "voice": "shubh",
    "language": "en-IN",
    "llm_model": "kimi-k2.5"
  }'
```

**Response:**
```json
{
  "id": "uuid",
  "ws_url": "wss://…",
  "token": "eyJhbGciOi...",
  "status": "created"
}
```

Read `ws_url` from this response and pass it straight to the client — it is issued per session. Do not hardcode it.

**2. Connect with the client SDK:**

```javascript
import { Room, RoomEvent, Track } from "livekit-client";

const room = new Room();

room.on(RoomEvent.TrackSubscribed, (track, pub, participant) => {
  if (track.kind === Track.Kind.Audio) {
    const el = track.attach();
    document.body.appendChild(el);
  }
});

room.on(RoomEvent.TranscriptionReceived, (segments, participant) => {
  for (const seg of segments) {
    if (seg.final) {
      const who = participant?.isLocal ? "You" : "Agent";
      console.log(who + ": " + seg.text);
    }
  }
});

await room.connect(session.ws_url, session.token);
await room.localParticipant.setMicrophoneEnabled(true);
```

The agent joins automatically, greets the user, and responds to speech.

## Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `system_prompt` | string | "You are a helpful voice assistant..." | System prompt for LLM |
| `voice` | string | `shubh` | TTS voice ID (37 voices available) |
| `language` | string | `en-IN` | Language code for STT and TTS |
| `llm_model` | string | `kimi-k2.5` | LLM model (`kimi-k2.5`, `sarvam-105b`, or any catalog model). `kimi-k2.5-fast` is currently under maintenance. |
| `tts_provider` | string | `sarvam` | TTS provider (currently only `sarvam`) |
| `max_duration_seconds` | int | 300 | Max session duration (30-3600) |

## Features

- **Interruption handling** — speak while the agent is talking and it stops immediately, listens to you
- **STT-based turn detection** — server-side VAD detects speech start/end with low-latency (~50ms) endpointing
- **Preemptive generation** — LLM starts generating before STT fully confirms the transcript
- **Streaming pipeline** — each stage streams to the next, no buffering between stages
- **Session management** — REST API for creating, listing, deleting sessions and retrieving transcripts
- **Per-model pricing** — usage tracked and billed per model ($0.81/$4.05 per 1M tokens)

## Legacy WebSocket

The direct WebSocket endpoint is still available for backward compatibility:

```
WS /ws/voice-agent
Sec-WebSocket-Protocol: token, cm_your_api_key
```

Authenticate with the subprotocol header. It is a request header, so the key stays out of access logs and proxy history, which a query string does not. In the browser, pass it as the constructor's second argument, with the literal `token` first and the key second:

```javascript
new WebSocket("wss://api.callmissed.com/ws/voice-agent", [
  "token",
  "cm_your_api_key",
]);
```

Clients that can set headers may send `Authorization: Bearer cm_your_api_key` instead. The `?key=cm_your_api_key` query parameter is **deprecated** and still accepted for existing integrations.

Send a config message after connecting, then stream PCM audio. This is a direct-WebSocket pipeline, separate from the WebRTC path above. See the [Session API](/docs/voice-sessions-api) for the recommended WebRTC approach.
