---
title: "Voice Client Libraries"
description: "Which packages you actually install to build a voice client, and how to wire them to the Voice Session API."
slug: "voice-sdk"
breadcrumb: "Voice Agents"
---

# Voice Client Libraries

Which packages you actually install to build a voice client, and how to wire them to the Voice Session API.

## What you install

There is no CallMissed-branded voice package on PyPI or npm. Voice is two plain pieces: a JSON REST call to create the session, and a standard WebRTC client in the browser to carry the audio.

| Layer | What to use | Why |
|-------|-------------|-----|
| Create a session (server side) | Any HTTP client: `httpx`, `requests`, `fetch`, `curl` | `POST /v1/voice/sessions` is a plain JSON endpoint |
| Browser audio | `livekit-client` (npm) | The create response hands you a WebRTC URL and token that this package consumes |
| Transcripts and session records | Any HTTP client | Plain JSON `GET` endpoints |

For the text LLM, speech-to-text and text-to-speech APIs there is likewise no bespoke package: those surfaces are OpenAI and Anthropic compatible, so you use the official `openai` or `anthropic` SDK with our base URL. See [Libraries & SDKs](/docs/sdks).

**Authentication:** every REST call on this page takes `Authorization: Bearer cm_your_api_key`. To create a voice session the key needs the `stt`, `tts` and `llm` permissions; a key missing any of them gets `403`.

## Step 1: create the session

Mint the session on your server, never in the browser, so your API key is never shipped to a client.

```bash
curl -X POST https://api.callmissed.com/v1/voice/sessions \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "system_prompt": "You are a helpful assistant. Keep replies to one or two sentences.",
    "greeting": "Hi, how can I help?",
    "voice": "shubh",
    "language": "en-IN",
    "max_duration_seconds": 1800
  }'
```

```python
import httpx

async with httpx.AsyncClient() as client:
    r = await client.post(
        "https://api.callmissed.com/v1/voice/sessions",
        headers={"Authorization": "Bearer cm_your_api_key"},
        json={
            "system_prompt": "You are a helpful assistant.",
            "greeting": "Hi, how can I help?",
            "voice": "shubh",
            "language": "en-IN",
        },
    )
    r.raise_for_status()
    session = r.json()

# Hand session["ws_url"] and session["token"] to your browser client.
print(session["id"], session["ws_url"])
```

```typescript
// Server-side route handler. Returns only ws_url + token to the browser.
const res = await fetch("https://api.callmissed.com/v1/voice/sessions", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.CALLMISSED_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    system_prompt: "You are a helpful assistant.",
    greeting: "Hi, how can I help?",
    voice: "shubh",
    language: "en-IN",
  }),
});

const session = await res.json();
return Response.json({ wsUrl: session.ws_url, token: session.token });
```

### Fields used above

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `system_prompt` | string | a concise built-in assistant prompt | Max 4096 characters |
| `greeting` | string | agent decides its own opener | Max 500 characters. The exact first line the agent speaks |
| `voice` | string | `shubh` | Max 50 characters |
| `language` | string | `en-IN` | Max 10 characters, BCP-47 |
| `llm_model` | string | resolved server side | Omit it to take the platform default voice stack |
| `max_duration_seconds` | int | `1800` | Between 30 and 3600. Hard ceiling for one active call |
| `variables` | object | none | Values for `{{token}}` placeholders in the greeting and prompt |
| `metadata` | object | none | Arbitrary JSON stored with the session |

`bot_id`, `webhook_url`, `tts_provider`, `tts_model`, `stt_model` and `tts_engine` are also accepted. Each call uses one selected stack without automatic model or provider substitution. The [Voice Session API](/docs/voice-sessions-api) page is the full reference for the request body, the other endpoints and the webhook events.

## Step 2: what the response gives you

```json
{
  "id": "7c2b9e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
  "tenant_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
  "bot_id": null,
  "status": "created",
  "config": { "system_prompt": "You are a helpful assistant.", "voice": "shubh", "language": "en-IN" },
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

The three fields your client needs:

| Field | Use |
|-------|-----|
| `ws_url` | The media server URL, issued per session. Read it from the response and pass it through; do not hardcode it |
| `token` | The connection credential. Returned once at creation and never fetchable again. It expires one hour after issue |
| `id` | The session id, for fetching the transcript afterwards |

`analysis` is `null` at creation and is populated later, once post-call analysis has run.

## Step 3: connect the browser

The transport package is the third-party `livekit-client`:

```bash
npm install livekit-client
```

```javascript
import { Room, RoomEvent, Track } from "livekit-client";

// wsUrl + token come from your own server route (step 1).
const { wsUrl, token } = await fetch("/api/voice/session").then((r) => r.json());

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
    console.log(`${who}: ${seg.text}`);
  }
});

await room.connect(wsUrl, token);
await room.localParticipant.setMicrophoneEnabled(true);
```

The agent joins the room on its own and runs the speech pipeline. Speech boundaries are detected server side, and interruptions are handled for you: talk while the agent is speaking and it stops and listens.

Call `room.disconnect()` to end the call from the client.

## Streaming audio from a server, not a browser

`livekit-client` is a browser package. If the thing holding the microphone is a Python, Go or Node process rather than a browser tab, do not reach for a WebRTC client at all: use the [Managed Voice Agent](/docs/managed-voice-agent) instead. It takes raw audio over a single plain WebSocket with no client SDK on your side.

## Step 4: read the transcript

```bash
curl "https://api.callmissed.com/v1/voice/sessions/{id}/transcript?format=json" \
  -H "Authorization: Bearer cm_your_api_key"
```

`format` accepts `json` (the default, a structured turn list), `txt` (alternating plain text) or `srt` (subtitles).
