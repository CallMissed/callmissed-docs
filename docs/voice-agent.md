---
title: "Voice Agent"
description: "Real-time voice agents over WebRTC with one selected speech-to-speech or STT-to-LLM-to-TTS stack."
slug: "voice-agent"
breadcrumb: "Voice Agents"
---

# Voice Agent

Real-time voice agents over WebRTC with one selected speech-to-speech or STT-to-LLM-to-TTS stack.

:::cards
/docs/voice-sessions-api | Voice Sessions API | key | Create sessions and generate connection tokens
/docs/voice-sdk | Voice SDK | package | Client SDK for browser and mobile WebRTC
/docs/voice-agent-tools | Voice Agent Tools | wrench | What the agent can call mid-conversation
/docs/full-duplex-voice | Full-Duplex Voice | radio | `gpt-live-1` listens and speaks at once
/docs/stt-realtime | Real-time STT | mic | Streaming speech-to-text over WebSocket
/docs/text-to-speech | Text to Speech | volume2 | Indic TTS for agent responses
:::

## Overview

The Voice Agent streams conversations over **WebRTC**. Choose one configuration for the call:

- **CallMissed-managed pipeline:** select one speech-recognition model, one language model, and one speech-generation model and voice.
- **Deepgram-managed pipeline:** select a `deepgram-voice-*` model and its supported recognition and voice settings.
- **Native speech-to-speech:** select a GPT Realtime or Nova Sonic model and voice.

- **Full-duplex speech-to-speech:** select `gpt-live-1`, which listens and speaks at the same time. It behaves differently enough that it has [its own page](/docs/full-duplex-voice).

An omitted `llm_model` selects `deepgram-voice-open-ai-gpt-5.4-nano`. Calls do not switch to another model or provider on failure. Same-provider transient retries remain; if the selected stack cannot run, the session fails or ends instead of substituting another stack. Retired `voice_fallbacks` settings are no longer used.

## Architecture

You create a session over REST and receive a connection URL + token. Your client connects with the `livekit-client` SDK; the CallMissed voice agent joins automatically and handles the speech pipeline. Audio flows over WebRTC.

This is the WebRTC path. For a plain WebSocket you stream raw audio to — no client SDK, no media hop — see the [Managed Voice Agent](/docs/managed-voice-agent), which runs the same tuned pipeline over `wss://api.callmissed.com`.

:::flow
icon:app | Browser (livekit-client SDK) | Captures mic audio and streams it over WebRTC
icon:server | Connection | WebRTC transport that connects your client to the voice agent
icon:bot | CallMissed voice agent | Runs the selected speech-to-speech model or STT → LLM → TTS pipeline
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
| `llm_model` | string | resolved server side | One supported voice model. Unavailable selections are not replaced. |
| `tts_provider` | string | *plan-dependent* | Legacy TTS selector; use `tts_model` for per-model selection. Omit it and the server picks by plan: paid plans (starter, pro, enterprise) default to Cartesia `sonic-3.6`, the free plan to Sarvam `bulbul:v3`. An explicit value is always honoured. |
| `max_duration_seconds` | int | 1800 | Max session duration (30-3600) |

## Features

- **Interruption handling** — speak while the agent is talking and it stops immediately, listens to you
- **STT-based turn detection** — server-side VAD detects speech start/end with low-latency (~50ms) endpointing
- **Preemptive generation** — LLM starts generating before STT fully confirms the transcript
- **Streaming pipeline** — each stage streams to the next, no buffering between stages
- **Session management** — REST API for creating, listing, deleting sessions and retrieving transcripts
- **Per-model pricing** — usage tracked and billed per model ($0.81/$4.05 per 1M tokens)
- **Tool calling** — built-in tools, your own REST endpoints and MCP servers, mid-conversation. See [Voice Agent Tools](/docs/voice-agent-tools)

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
