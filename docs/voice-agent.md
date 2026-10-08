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
- **Native speech-to-speech:** select a GPT Realtime model and voice.

- **Full-duplex speech-to-speech:** select `gpt-live-1`, which listens and speaks at the same time. It behaves differently enough that it has [its own page](/docs/full-duplex-voice).

An omitted `llm_model` selects `gemma-4-31b` on the STT → LLM → TTS pipeline, with thinking off for the fastest replies. If the selected stack cannot start, the call runs on a backup stack instead of failing, and usage records the model that actually served it. With `gemma-4-31b`, if the model stops answering mid-call, a backup model answers the remaining turns, billed at the `gemma-4-31b` rate. Retired `voice_fallbacks` settings are no longer used.

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

With a GPT Realtime model selected, every turn stays in one speech-to-speech model. With a cascaded model selected (or when the speech-to-speech model is unavailable), every turn streams through STT, LLM, and TTS concurrently to minimize time-to-first-audio:

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
| `voice` | string | `shubh` | A speaker of the selected TTS model (`bulbul:v3` has 37) |
| `language` | string | `en-IN` | Language code for STT and TTS |
| `llm_model` | string | `gemma-4-31b` | One voice-capable model. An id the agent cannot serve is rejected at create time with `422` |
| `tts_provider` | string | *plan-dependent* | Legacy TTS selector; use `tts_model` for per-model selection. Omit it and the server picks by plan: paid plans (starter, pro, enterprise) default to Cartesia `sonic-3.6`, the free plan to Sarvam `bulbul:v3`. An explicit value is always honoured. |
| `max_duration_seconds` | int | 1800 | Max session duration (30-3600) |

`stt_model`, `tts_model`, `greeting`, `variables`, `bot_id` and the rest are covered in the [Voice Session API](/docs/voice-sessions-api#request-body) reference.

## Backup voices and spoken numbers

Two agent settings, saved in the agent's `config` with [`PATCH /api/v1/bots/{bot_id}/config`](/docs/bots) (or the console's **Speaking style** page). They apply to agents that choose their own models; an agent on a [voice tier](/docs/voice-tiers) keeps the tier's voices.

### `tts_fallbacks`

Up to 3 voices to speak through, in order, if the agent's own voice stops working. They are tried before the platform's own backup voice, both when the call starts and in the middle of a call.

```json
{
  "values": {
    "tts_fallbacks": [
      { "tts_model": "sonic-3.6", "voice": "katie" },
      { "tts_model": "bulbul:v3", "voice": "priya" }
    ]
  }
}
```

- `tts_model` is any voice model from the agent's speech models (the `tts_model` values in [`GET /api/v1/bots/config-schema`](/docs/bots)). `voice` is optional; left out, the model's default voice speaks.
- Each entry must speak the agent's `language` (for `language: "multi"`, the opening language), and `voice` must be one of that model's voices. A cloned voice cannot be a backup voice. Anything else is refused with `422`, and so is a later `language` change that a saved backup cannot speak.
- Billing follows the backup voice that spoke. If the agent's voice could not start and a backup voice served the call, the session's `metadata.stack_substitution` names it as `served_tts_model`; if the switch happens mid-call, each turn it speaks is billed at that backup voice's rate. When the platform's own backup voice speaks instead, it is billed at your voice's rate.

### `tts_normalize`

`true` reads numbers the way a person says them: amounts with Indian grouping (`₹1,50,000` → "one lakh fifty thousand rupees"), `$` amounts, dates (`12/03/2026` is read day first), times, percentages, ordinals, ranges and phone numbers (digit by digit). Long digit runs and numbers starting with `0` (order IDs, one-time codes) are read digit by digit.

- English text gets English words; Hindi text in Devanagari gets Hindi words in Devanagari ("एक लाख पचास हज़ार रुपये").
- Links, email addresses and digits joined to letters (`24x7`, `4G`) are left as written. The saved transcript keeps the original digits.
- Off by default. It has no effect on `sonic-3.6`, `bulbul:v3` and `gnani-timbre-v2.0`, which already read numbers aloud themselves, or on `bulbul:v2` with `tts_preprocessing` on.
- `sonic-3.6` is given the agent's regional language (for example `en-IN` or `hi-IN`, not just `en` or `hi`), so it reads amounts and dates the way that region does.

## Turn-taking and speech-to-speech tuning

Advanced agent settings, saved in the agent's `config` with [`PATCH /api/v1/bots/{bot_id}/config`](/docs/bots) (or the console's **Voice** page). Leave them unset unless calls cut people off or feel slow; an unset key keeps the default shown. A number outside its range is saved with a warning in the response and used as the nearest end of the range on calls; a value the key does not list is ignored on calls.

### Turn-taking (agents with separate listening, thinking and speaking models)

| Key | Values | Default | What it does |
|-----|--------|---------|--------------|
| `min_endpointing_delay` | 0.2–1.5 s | 0.3 | How long the agent waits after the caller stops before replying. |
| `endpointing_mode` | `fixed`, `dynamic` | `fixed` | `fixed` waits the same pause every turn. `dynamic` learns how long this caller pauses mid-thought and waits that long, between `min_endpointing_delay` and `max_endpointing_delay`. |
| `max_endpointing_delay` | 1.0–6.0 s | 3.0 (2.5 with `audio_turn_detector`) | The longest the agent waits when the caller sounds unfinished. It is the ceiling for `dynamic` and for `audio_turn_detector`; with neither, the agent never waits this long. |
| `audio_turn_detector` | `true`, `false` | `false` | At each pause, listens to how the caller sounds, not just how long the pause is, and waits longer when they sound unfinished. Only for agents whose `language` is English or Hindi. Not used with Flux or `ink-2` listening models, which do this themselves. |
| `flux_eot_threshold` | 0.5–0.9 | 0.7 | Flux listening models only. How sure the model must be that the caller finished. Higher waits longer and cuts in less. |
| `flux_eager_eot_threshold` | 0.3–0.9 | 0.4 | Flux listening models only. The agent starts preparing its reply once the model is this sure. Lower is faster. Never above `flux_eot_threshold`: a higher value is lowered to it. |

### Speech-to-speech models (`gpt-realtime` family)

For `gpt-realtime`, `gpt-realtime-mini`, `gpt-realtime-1.5`, `gpt-realtime-2`, `gpt-realtime-2.1` and `gpt-realtime-2.1-mini`. The model decides when the caller has finished, so the turn-taking keys above do not apply.

| Key | Values | Default | What it does |
|-----|--------|---------|--------------|
| `realtime_vad` | `server_vad`, `semantic_vad` | `server_vad` | `server_vad` replies after a set pause. `semantic_vad` replies when the caller's words sound finished. |
| `realtime_eagerness` | `low`, `medium`, `high`, `auto` | `auto` | With `semantic_vad` only. `low` waits longest (up to 8 s), `high` replies fastest, `auto` is `medium`. |
| `realtime_vad_threshold` | 0.0–1.0 | 0.5 | With `server_vad` only. How loud the caller must be to count as speech. Raise it on noisy lines. |
| `realtime_silence_ms` | 200–2000 ms | 200 | With `server_vad` only. The pause before the agent replies. |
| `realtime_noise_reduction` | `near_field`, `far_field` | off | Cleans the caller's audio before the model hears it. `near_field` for a phone held to the ear or a headset, `far_field` for a speakerphone or laptop. |
| `realtime_speed` | 0.25–1.5 | 1.0 | Speaking pace. |

Setting `realtime_vad_threshold` or `realtime_silence_ms` without `realtime_vad` uses `server_vad`. The caller can always interrupt a speech-to-speech agent.

```json
{
  "values": {
    "realtime_vad": "semantic_vad",
    "realtime_eagerness": "low",
    "realtime_noise_reduction": "near_field"
  }
}
```

## Features

- **Interruption handling** — speak while the agent is talking and it stops immediately, listens to you
- **Server-side turn detection** — voice-activity detection finds where speech starts and ends; no client-side VAD needed
- **Noise filtering (coming soon)** — the `noise_suppression` agent setting will filter steady background noise such as fans, traffic and line hiss out of the caller's audio. Not available yet: we're testing it on real calls first, and setting it has no effect today
- **Preemptive generation** — LLM starts generating before STT fully confirms the transcript
- **Streaming pipeline** — each stage streams to the next, no buffering between stages
- **Session management** — REST API for creating, listing, deleting sessions and retrieving transcripts
- **Per-model pricing** — each model is billed at its own catalogue rate (see the [model catalogue](/docs/models)), and [`GET /v1/voice/sessions/{id}/cost`](/docs/voice-sessions-api#session-cost) itemises a call
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

Send a config message after connecting (`{"type": "config", "system_prompt": "…", "voice": "shubh", "language": "en-IN", "llm_model": "sarvam-105b"}`), then stream binary 16-bit PCM, 16 kHz mono. The agent's speech comes back as binary MP3 chunks between `audio_start` and `audio_end` events. A key without `stt`, `tts` and `llm` permissions is refused. This is a direct-WebSocket pipeline, separate from the WebRTC path above. See the [Session API](/docs/voice-sessions-api) for the recommended WebRTC approach.
