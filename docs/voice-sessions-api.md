---
title: "Voice Session API"
description: "REST API for creating and managing WebRTC voice agent sessions."
slug: "voice-sessions-api"
breadcrumb: "Voice Agents"
---

# Voice Session API

REST API for creating and managing WebRTC voice agent sessions.

## Overview

The Voice Session API provides a two-step flow for voice agent interactions:

1. **Create a session** via REST — returns a connection URL + JWT
2. **Connect over WebRTC** — stream audio with the `livekit-client` SDK; the agent joins automatically and handles STT → LLM → TTS

Audio flows over WebRTC; on this API the REST endpoints handle session metadata, token issuance, usage tracking and transcript storage.

Each session runs the voice stack you select. Model ids are checked when you create the session: an id the voice agent cannot serve is rejected with `422` (the error lists the supported ids), and a model under maintenance is rejected with `503`. If the selected stack still cannot be started when the call connects, the call runs on a backup stack instead of failing, and usage and [`/cost`](#session-cost) record the model that actually served it. `voice_fallbacks` is no longer a request field.

If you would rather stream audio straight to us over a plain WebSocket — no WebRTC and no client SDK — use the [Managed Voice Agent](/docs/managed-voice-agent) instead. This page covers the WebRTC session API, which remains the right choice for browser calls with adaptive bitrate.

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
    "tts_model": "bulbul:v3",
    "stt_model": "saaras:v3",
    "max_duration_seconds": 300,
    "…": "…"
  },
  "ws_url": "wss://…",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "started_at": null,
  "ended_at": null,
  "duration_seconds": null,
  "turn_count": 0,
  "total_audio_seconds": 0,
  "end_reason": null,
  "metadata": null,
  "created_at": "2026-04-19T12:00:00Z",
  "analysis": null
}
```

- `config` echoes the resolved session settings, including the defaults the server filled in. It can also carry internal bookkeeping keys; treat it as informational and do not depend on keys this page does not list.
- `ws_url` is the **media server** URL — not the CallMissed API. It is issued per session; read it from the response and pass it straight to the client, do not hardcode it.
- `token` is the **connection JWT** (not an opaque `vs_*` string). TTL is **1 hour**.
- The token is returned **once** on creation and is not fetchable again.

## Request Body

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `bot_id` | uuid | — | Optional agent to load prompt, tools and settings from. Must belong to your workspace (`404` otherwise) |
| `system_prompt` | string | "You are a helpful voice assistant..." | Max 4096 chars. Overrides bot's prompt if both set |
| `greeting` | string | — | Max 500 chars. The exact first line the agent speaks; omit it and the agent opens on its own |
| `voice` | string | `shubh` | Max 50 chars. A speaker of the selected TTS model (`bulbul:v3` has 37); each model's speakers are listed in [`GET /api/v1/voice/models`](/docs/managed-voice-agent#list-available-models) under `tts[].voices` |
| `language` | string | `en-IN` | Max 10 chars. BCP-47 language for STT + TTS |
| `llm_model` | string | `gemma-4-31b` | Any voice-capable model id from [`GET /api/v1/voice/models`](/docs/managed-voice-agent#list-available-models) (`kimi-k2.5`, `sarvam-105b`, `gpt-5.6-luna`, `gpt-live-1`, …). Max 100 chars. An unsupported id returns `422`; a model under maintenance (e.g. `kimi-k2.5-fast`) returns `503` |
| `stt_model` | string | `saaras:v3` | Speech-recognition model id, from the same list. Max 64 chars |
| `tts_model` | string | follows `tts_provider` | Text-to-speech model id, from the same list. Max 64 chars. Defaults to `bulbul:v3` for `sarvam` and `sonic-3.6` for `cartesia` |
| `tts_provider` | string | *plan-dependent* | Legacy selector: `sarvam` or `cartesia`; prefer `tts_model`. `elevenlabs` is not available and returns `422`. Omit it and the server picks by plan: paid plans (starter, pro, enterprise) default to Cartesia `sonic-3.6`, the free plan to Sarvam `bulbul:v3`. An explicit value is always honoured. |
| `tts_engine` | string | `cartesia` | Only for `deepgram-voice-*` models: `deepgram`, `flux`, `cartesia` or `aura-1`. With a `deepgram-voice-*` model and no `stt_model`, recognition defaults to `deepgram-flux-general-multi` |
| `max_duration_seconds` | int | `1800` | 30–3600 |
| `variables` | object | — | Values for `{{token}}` placeholders in the greeting and prompt, e.g. `{"callee_name": "Priya"}`. With `bot_id`, a variable the agent marks required must be supplied (`422` otherwise) |
| `webhook_url` | string | — | Max 2048 chars; must be a public `http`/`https` URL (`400` otherwise). Stored with the session. Session events are delivered to your registered [webhook endpoints](#webhook-events), not to this URL |
| `metadata` | object | — | Arbitrary JSON stored with the session |

## Connect the client

Use `ws_url` + `token` returned by create. **Do not** try to open a WebSocket to the CallMissed API — use the `livekit-client` SDK:

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

Response contains everything from the create response **except** `ws_url` and `token` — those are issued once at creation (they come back as empty strings).

If a part of the requested speech stack (speech recognition, LLM or voice) could not be started, the call is served with a backup for that part only, and `metadata.stack_substitution` records it: `requested_llm` / `requested_stt_model` / `requested_tts_model`, the `served_llm` / `served_stt_model` / `served_tts_model` that ran, and `failed` (the parts replaced). Usage is billed for the models that served.

`analysis` is `null` until post-call analysis has run for the session (it is always `null` on the list endpoint). Once present:

| Field | Type | Notes |
|-------|------|-------|
| `status` | string | Analysis state, e.g. `completed` |
| `summary` | string \| null | Short call summary |
| `sentiment` | string \| null | Caller sentiment |
| `disposition` | string \| null | Call outcome |
| `extracted` | object | Values for the agent's declared analysis variables; `{}` when it declares none |
| `goal_met` | bool \| null | `null` when the agent declares no goal variable |
| `model` | string \| null | Model that produced the analysis |
| `error` | string \| null | Short machine code when analysis failed |
| `created_at` / `updated_at` | datetime | |

## Get Transcript

```bash
curl "https://api.callmissed.com/v1/voice/sessions/{id}/transcript?format=json" \
  -H "Authorization: Bearer cm_your_api_key"
```

**`format` query param** — defaults to `json`:

| Format | Content-Type | Shape |
|--------|--------------|-------|
| `json` | application/json | Array of turns: `id`, `turn_index`, `user_transcript`, `agent_response`, `interrupted`, `stt_ms`, `first_token_ms`, `tts_ttfb_ms`, `eou_delay_ms`, `first_audio_ms`, `total_ms`, `llm_model`, `created_at`. Timing fields are milliseconds and `null` when a stage was not measured |
| `txt` | text/plain | Human-readable alternating `User:` / `Agent:` lines |
| `srt` | application/x-subrip | SubRip subtitles with timing derived from per-turn durations |

Transcripts are saved for every session, whether or not the call is recorded.

## Get Recording

A session's audio is recorded only when its agent has call recording turned on (`record_calls: true` in the agent's config; off by default). The agent then speaks a short recording notice before its greeting unless `recording_notice_enabled` is `false`; `recording_notice` sets the wording. The recording is one MP3 with both sides of the call, available a few seconds after the call ends. This covers every way the agent takes calls (phone in and out, WhatsApp, web) and every voice model. A [managed voice agent](/docs/managed-voice-agent) session is recorded when its `Settings` set `agent.record_calls: true`.

```bash
curl https://api.callmissed.com/v1/voice/sessions/{id}/recording \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "url": "https://...signed-url..." }
```

The URL is private and time-limited — fetch it on demand rather than storing it. Returns `404` when the session has no recording, and `503` if recording storage is temporarily unavailable. A phone call's recording is also returned by `GET /api/v1/telephony/calls/{call_id}/recording`.

## Delete Session

```bash
curl -X DELETE https://api.callmissed.com/v1/voice/sessions/{id} \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns `204 No Content`. Sessions in `created` or `active` state are marked `completed` with `end_reason = "api_delete"`; already-finished sessions are left unchanged.

## Session Cost

```bash
curl https://api.callmissed.com/v1/voice/sessions/{id}/cost \
  -H "Authorization: Bearer cm_your_api_key"
```

The AI cost of a session, one line per `(service, model)` pair it actually used — so a call that ran on a backup stack shows the model that served it. Credits are what was actually charged.

```json
{
  "session_id": "7c2b9e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "total_credits": 1.84,
  "items": [
    { "service": "stt", "model": "saaras:v3", "credits": 0.42, "input_tokens": 0, "output_tokens": 0, "audio_seconds": 50.3 },
    { "service": "llm", "model": "kimi-k2.5", "credits": 0.61, "input_tokens": 5120, "output_tokens": 410, "audio_seconds": 0.0 },
    { "service": "tts", "model": "bulbul:v3", "credits": 0.81, "input_tokens": 0, "output_tokens": 0, "audio_seconds": 38.9 }
  ]
}
```

Items are ordered STT, LLM, TTS. This covers the AI layer only; phone-carrier charges for a call on a rented number are billed separately.

## Limits

| Limit | Value | Behavior on exceed |
|-------|-------|--------------------|
| Session create rate | 10 / minute / workspace | HTTP 429 |
| API key spend cap / per-key rate limit | as set on the key | HTTP 402 / HTTP 429 |
| Concurrent active sessions (free) | 1 | HTTP 429 |
| Concurrent active sessions (starter) | 5 | HTTP 429 |
| Concurrent active sessions (pro) | 20 | HTTP 429 |
| Concurrent active sessions (enterprise) | unlimited | — |
| Minimum credit balance to create | server-configured | HTTP 402 (also when your monthly budget cap is exhausted) |
| Max session duration | 3600s (capped by `max_duration_seconds`) | session auto-ends |
| Connection token TTL | 3600s (1 hour) | reconnect requires a new session |

## Webhook Events

Session events go to the webhook endpoints you register with the [Webhooks API](/docs/webhooks) — subscribe an endpoint to the events below. Each delivery is a `POST` with the JSON body `{"event": "<name>", "data": {…}, "timestamp": "<ISO 8601>"}`, signed with that endpoint's secret. An endpoint scoped to one agent receives only that agent's sessions.

| Event | When | `data` |
|-------|------|--------|
| `voice_session.started` | Session created (token issued) | `session_id`, `bot_id`, `llm`, `voice`, `language` |
| `voice_session.ended` | Session completed: the call finished normally or was ended with `DELETE` | `session_id`, `bot_id`, `duration_seconds`, `turn_count`, `end_reason` |
| `voice_session.failed` | Session failed (`end_reason` `agent_error`), timed out, or its token was never used before it expired (`timeout` / `token_expired`) | `session_id`, `bot_id`, `end_reason`, `duration_seconds` |
| `voice_analysis.completed` | Post-call analysis finished | `analysis_id`, `session_id`, `bot_id`, `disposition`, `sentiment`, `goal_met` |

Each session fires exactly one of `voice_session.ended` or `voice_session.failed`, once, after the end is saved. The full `end_reason` table is on [Webhooks](/docs/webhooks#voice-session-end-reasons).

**Delivery headers:**

| Header | Value |
|--------|-------|
| `X-CallMissed-Event` | Event name (e.g. `voice_session.started`) |
| `X-CallMissed-Delivery` | Delivery UUID (unique per attempt batch) |
| `X-CallMissed-Signature` | `sha256=<hex>` HMAC-SHA256 of the raw body using the endpoint's webhook secret |

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
