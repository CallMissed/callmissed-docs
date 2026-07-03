---
title: "Voice Session API"
description: "REST API for creating and managing LiveKit-based voice agent sessions."
slug: "voice-sessions-api"
breadcrumb: "API Guides & Tutorials > Voice Agent"
---

# Voice Session API

REST API for creating and managing LiveKit-based voice agent sessions.

## Overview

The Voice Session API provides a two-step flow for voice agent interactions:

1. **Create a session** via REST — returns a LiveKit room URL + JWT
2. **Connect via LiveKit WebRTC** — stream audio with the `livekit-client` SDK; the agent joins automatically and handles STT → LLM → TTS

Audio flows over WebRTC to the LiveKit room. There is **no direct WebSocket between the browser and the CallMissed API** — the REST endpoints handle session metadata, token issuance, usage tracking, and transcript storage.

**Authentication:** All REST endpoints accept both **JWT** (`Authorization: Bearer <jwt>`) and **API key** (`Authorization: Bearer cm_<key>`). API keys must have `stt`, `tts`, and `llm` permissions to create a session.

## Create Session

```bash
curl -X POST https://api.callmissed.com/v1/voice/sessions \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "system_prompt": "You are a helpful assistant.",
    "voice": "shubh",
    "language": "en-IN",
    "llm_model": "kimi-k2.5",
    "tts_provider": "sarvam",
    "max_duration_seconds": 300,
    "webhook_url": "https://your-app.com/webhooks/voice"
  }'
```

**Response (201 Created):**
```json
{
  "id": "7c2b9e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "tenant_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
  "bot_id": null,
  "status": "created",
  "config": {
    "system_prompt": "You are a helpful assistant.",
    "voice": "shubh",
    "language": "en-IN",
    "llm_model": "kimi-k2.5",
    "tts_provider": "sarvam",
    "max_duration_seconds": 300,
    "livekit_room": "voice-<random>"
  },
  "ws_url": "wss://livekit.callmissed.com",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "started_at": null,
  "ended_at": null,
  "duration_seconds": null,
  "turn_count": 0,
  "total_audio_seconds": 0,
  "end_reason": null,
  "metadata": null,
  "created_at": "2026-04-19T12:00:00Z"
}
```

- `ws_url` is the **LiveKit server** URL — not the CallMissed API.
- `token` is a **LiveKit JWT** (not an opaque `vs_*` string). TTL is **1 hour**.
- The token is returned **once** on creation and is not fetchable again.

## Request Body

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `bot_id` | uuid | — | Optional bot to load prompt/knowledge from |
| `system_prompt` | string | "You are a helpful voice assistant..." | Max 4096 chars. Overrides bot's prompt if both set |
| `voice` | string | `shubh` | TTS voice ID (37 voices) |
| `language` | string | `en-IN` | BCP-47 language for STT + TTS |
| `llm_model` | string | `kimi-k2.5` | Any catalog model (`sarvam-30b`, `sarvam-105b`, `openai/*`, etc.). `kimi-k2.5-fast` is currently under maintenance. |
| `tts_provider` | string | `sarvam` | Currently `sarvam` only |
| `max_duration_seconds` | int | `300` | 30–3600 |
| `webhook_url` | string | — | Receives session events (see below) |
| `metadata` | object | — | Arbitrary JSON stored with the session |

## Connect via LiveKit

Use `ws_url` + `token` returned by create. **Do not** try to open a WebSocket to the CallMissed API — use the LiveKit client:

```bash
npm install livekit-client
```

```javascript
import { Room, RoomEvent, Track } from "livekit-client";

const session = await fetch("https://api.callmissed.com/v1/voice/sessions", {
  method: "POST",
  headers: {
    "Authorization": "Bearer cm_your_api_key",
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    system_prompt: "You are a helpful assistant.",
    voice: "shubh",
    language: "en-IN",
    llm_model: "kimi-k2.5",
  }),
}).then(r => r.json());

const room = new Room();

room.on(RoomEvent.TrackSubscribed, (track) => {
  if (track.kind === Track.Kind.Audio) {
    document.body.appendChild(track.attach());
  }
});

room.on(RoomEvent.TranscriptionReceived, (segments, participant) => {
  for (const seg of segments) {
    if (!seg.final) continue;
    const who = participant?.isLocal ? "You" : "Agent";
    console.log(who + ": " + seg.text);
  }
});

await room.connect(session.ws_url, session.token);
await room.localParticipant.setMicrophoneEnabled(true);
```

The agent joins the room, greets the user, listens for speech, and responds. Server-side VAD detects speech boundaries; interruptions are handled automatically (speak while the agent is talking and it stops and listens).

## List Sessions

```bash
curl "https://api.callmissed.com/v1/voice/sessions?status=completed&limit=50&offset=0" \
  -H "Authorization: Bearer cm_your_api_key"
```

**Query parameters:**

| Param | Values | Default |
|-------|--------|---------|
| `status` | `created` / `active` / `completed` / `failed` / `timeout` | — |
| `limit` | 1–200 | 50 |
| `offset` | ≥ 0 | 0 |

Returns an array of `VoiceSessionOut` objects (same shape as Get Session).

## Get Session

```bash
curl https://api.callmissed.com/v1/voice/sessions/{id} \
  -H "Authorization: Bearer cm_your_api_key"
```

Response contains everything from the create response **except** `ws_url` and `token` — those are issued once at creation.

## Get Transcript

```bash
curl "https://api.callmissed.com/v1/voice/sessions/{id}/transcript?format=json" \
  -H "Authorization: Bearer cm_your_api_key"
```

**`format` query param** — defaults to `json`:

| Format | Content-Type | Shape |
|--------|--------------|-------|
| `json` | application/json | Array of turns: `turn_index`, `user_transcript`, `agent_response`, `interrupted`, `stt_ms`, `first_token_ms`, `first_audio_ms`, `total_ms`, `llm_model`, `created_at` |
| `txt` | text/plain | Human-readable alternating `User:` / `Agent:` lines |
| `srt` | application/x-subrip | SubRip subtitles with timing derived from per-turn durations |

## Delete Session

```bash
curl -X DELETE https://api.callmissed.com/v1/voice/sessions/{id} \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns `204 No Content`. Sessions in `created` or `active` state are marked `completed` with `end_reason = "api_delete"`; already-finished sessions are left unchanged.

## Limits

| Limit | Value | Behavior on exceed |
|-------|-------|--------------------|
| Session create rate | 10 / minute / tenant | HTTP 429 |
| Concurrent active sessions (free) | 1 | HTTP 429 |
| Concurrent active sessions (starter) | 5 | HTTP 429 |
| Concurrent active sessions (pro) | 20 | HTTP 429 |
| Concurrent active sessions (enterprise) | unlimited | — |
| Minimum credit balance to create | server-configured | HTTP 402 |
| Max session duration | 3600s (capped by `max_duration_seconds`) | session auto-ends |
| LiveKit token TTL | 3600s (1 hour) | reconnect requires a new session |

## Webhook Events

If `webhook_url` is set on session creation, the following events are delivered as `POST` requests with JSON body and HMAC-SHA256 signature:

| Event | When |
|-------|------|
| `voice_session.started` | Session created (token issued) |
| `voice_session.ended` | Session marked completed (normal finish or `DELETE`) |
| `voice_session.failed` | Session entered failed state |

**Delivery headers:**

| Header | Value |
|--------|-------|
| `X-CallMissed-Event` | Event name (e.g. `voice_session.started`) |
| `X-CallMissed-Delivery` | Delivery UUID (unique per attempt batch) |
| `X-CallMissed-Signature` | `sha256=<hex>` HMAC of the raw body using your webhook secret |

**Verify the signature:**

```python
import hmac, hashlib

raw_body = await request.body()                            # bytes — do not re-serialize
received = request.headers["X-CallMissed-Signature"]       # e.g. "sha256=abc123..."
expected = "sha256=" + hmac.new(
    webhook_secret.encode(), raw_body, hashlib.sha256
).hexdigest()
assert hmac.compare_digest(expected, received)
```

```javascript
import crypto from "node:crypto";

const received = req.headers["x-callmissed-signature"];   // "sha256=..."
const expected = "sha256=" + crypto
  .createHmac("sha256", webhookSecret)
  .update(rawBody)                                        // raw Buffer / string
  .digest("hex");

if (!crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(received))) {
  return res.status(401).send("invalid signature");
}
```