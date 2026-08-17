---
title: "Managed Voice Agent"
description: "Stream audio over a WebSocket and get a speaking agent back. Two protocols: CallMissed-native and Deepgram Voice Agent compatible."
slug: "managed-voice-agent"
breadcrumb: "Voice Agents"
---

# Managed Voice Agent

Stream audio over a WebSocket and get a speaking agent back. Two protocols: CallMissed-native and Deepgram Voice Agent compatible.

## Overview

The Managed Voice Agent is a full speech-to-speech pipeline behind a single
WebSocket. You stream microphone audio in; you get synthesized speech and
conversation events back. Speech recognition, the language model, text-to-speech,
turn-taking and interruption handling are all run and tuned for you.

Unlike the [Voice Session API](/docs/voice-sessions-api), there is no WebRTC and
no client SDK to install — a plain WebSocket and raw PCM is the whole integration.

Two wire protocols are served, backed by the same engine:

| Endpoint | Protocol |
| --- | --- |
| `wss://voice.callmissed.com/v2/voice/agent` | CallMissed-native |
| `wss://voice.callmissed.com/v1/agent/converse` | Deepgram Voice Agent compatible |

If you already have an integration written against Deepgram's Voice Agent API,
point it at the second URL and it will work unchanged.

## Authentication

Both endpoints take an API key with `stt`, `tts` and `llm` permissions.

```http
Authorization: Token cm_your_api_key
```

Browsers cannot set headers on a WebSocket, so the subprotocol form is also
accepted:

```js
new WebSocket(url, ["token", "cm_your_api_key"])
```

Plan limits, the concurrent-session cap and your credit balance are all checked
before the socket is accepted, so an over-limit connection fails at the
handshake rather than mid-conversation.

## Session flow

1. Connect. The server sends `Welcome` with a `request_id`.
2. Send `Settings` as your first message.
3. Wait for `SettingsApplied`, then start streaming audio.
4. Send raw PCM as **binary** frames; receive synthesized PCM as binary frames
   and events as JSON on the same socket.

```json
{ "type": "Welcome", "request_id": "fc553ec9-5874-49ca-a47c-b670d525a4b1" }
```

## Settings (native)

The native shape is organised around what you actually choose: one model id per
role.

```json
{
  "type": "Settings",
  "audio": {
    "input":  { "encoding": "linear16", "sample_rate": 24000 },
    "output": { "encoding": "linear16", "sample_rate": 24000 }
  },
  "agent": {
    "prompt": "You are a concise support agent for an Indian retail brand.",
    "greeting": "Hi, how can I help?",
    "language": "en-IN",
    "llm": { "model": "gpt-oss-120b", "temperature": 0.4 },
    "stt": { "model": "saaras:v3" },
    "tts": { "model": "bulbul:v3", "voice": "shubh" }
  },
  "tags": ["support"]
}
```

Every other client message — updates, injection, tool responses, keepalive — is
identical across both protocols. Switching protocols means rewriting one message.

### Audio format

`linear16` (raw 16-bit PCM, little-endian, mono) in both directions, at
`8000`, `16000`, `24000`, `32000` or `48000` Hz. Output is raw frames with no
container: a WAV or OGG header would be read as audio by most telephony
consumers.

An unsupported encoding, sample rate or container is rejected at `Settings`
time rather than accepted and quietly changed, so you find out at the handshake
instead of hearing the wrong thing on a call.

## Choosing models

Call [`GET /api/v1/voice/models`](#list-available-models) for the models you can
use, each with its measured latency. Any eligible speech-to-text × language model
× text-to-speech combination is valid.

Models are offered based on **measured** performance from real traffic, not on
vendor claims. A model that is too slow to hold a conversation is not offered —
it is listed with the reason, so you can see why rather than wondering where it
went.

### List available models

```bash
curl https://api.callmissed.com/api/v1/voice/models \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "turn_budget_ms": 1000,
  "llm": [
    {
      "id": "gpt-oss-120b",
      "label": "GPT-OSS 120B",
      "eligible": true,
      "verdict": "eligible",
      "p50_ms": 180.0,
      "samples": 240,
      "budget_ms": 400,
      "reason": "p50 180ms within the 400ms budget",
      "languages": ["en"],
      "voices": []
    }
  ],
  "stt": [],
  "tts": []
}
```

| `verdict` | Meaning |
| --- | --- |
| `eligible` | Measured within its stage budget. Selectable. |
| `too_slow` | Measured over budget. Not offered; `reason` says by how much. |
| `unsupported` | Structurally unavailable (e.g. under maintenance). |
| `unmeasured` | Not enough measured turns yet to give a verdict. |

`p50_ms` is `null` when a model has no measurement. It is never reported as `0`
— zero would read as "instant" for a stage that was simply never measured.

## Server events

| Event | Meaning |
| --- | --- |
| `Welcome` | Socket open, carries `request_id`. |
| `SettingsApplied` | Configuration accepted; start streaming. |
| `ConversationText` | A finished turn — `role` is `user` or `assistant`. |
| `UserStartedSpeaking` | **Barge-in.** Stop playback and clear your buffer. |
| `AgentThinking` | The model is working. |
| `AgentStartedSpeaking` | First audio of a turn; carries latency numbers. |
| `AgentAudioDone` | Last audio chunk *sent* for this turn. |
| `FunctionCallRequest` | Run a tool and reply. |
| `LatencyReport` | Per-stage timings for the turn. |
| `Warning` | Non-fatal; the session continues. |
| `Error` | Fatal; reconnect. |

<Callout type="warn">
`UserStartedSpeaking` is the **only** barge-in signal — there is no separate
flush message. When you receive it, stop playback and discard whatever audio you
have buffered. If you don't, the caller keeps hearing the interrupted sentence
for as long as your playback buffer is deep, which is the most common reason
interruption appears not to work.
</Callout>

<Callout>
`AgentAudioDone` means the last chunk was **sent**, not that the caller heard it.
Your own playback buffer may still be draining.
</Callout>

Latency fields are omitted when a stage was not measured, rather than reported
as `0`.

## Tool calling

Declare tools in `Settings`, then answer requests over the same socket.

```json
{
  "type": "FunctionCallRequest",
  "functions": [
    {
      "id": "fc_01H...",
      "name": "lookup_order",
      "arguments": "{\"order_id\":\"A-1042\"}",
      "client_side": true
    }
  ]
}
```

Reply with the result. `arguments` is a JSON **string**, not an object.

```json
{
  "type": "FunctionCallResponse",
  "id": "fc_01H...",
  "name": "lookup_order",
  "content": "{\"status\":\"shipped\",\"eta\":\"2 days\"}"
}
```

A tool that does not answer within 30 seconds does not hang the call — the turn
continues and a `Warning` is emitted.

Server-side execution (a `functions[].endpoint`) is **not** supported. Declaring
one returns a `Warning` and the call is dispatched to your client instead, so an
unsupported mode is never silently ignored.

## Updating a live session

| Message | Effect |
| --- | --- |
| `UpdatePrompt` | Append to the system prompt. |
| `UpdateThink` | Switch the language model. |
| `UpdateListen` | Switch speech recognition. |
| `UpdateSpeak` | Switch voice or text-to-speech model. |
| `InjectAgentMessage` | Make the agent say something now. |
| `InjectUserMessage` | Inject text as if the caller said it. |
| `KeepAlive` | Hold an idle socket open. |

A model id the service cannot serve keeps the current model and returns a
`Warning`; it is never swapped for a different one behind your back.

## Latency

The fast path targets **sub-1s** from end of your speech to first audio back.
That budget is the sum of three serial stages:

| Stage | Budget |
| --- | --- |
| Turn detection | ~300 ms |
| Language model, first token | 400 ms |
| Text-to-speech, first byte | 300 ms |

`LatencyReport` gives you the real numbers per turn, so you can measure rather
than take our word for it.

<Callout type="warn">
Sub-200ms end-to-end voice-to-voice is not achievable with a speech-to-text →
language model → speech pipeline, by anyone. Detecting that you stopped speaking
alone costs more than that. Treat sub-second as the realistic target and measure
the rest with `LatencyReport`.
</Callout>

## Limits

| Limit | Value |
| --- | --- |
| Maximum session length | 2 hours |
| Tool response timeout | 30 seconds |
| Concurrent sessions | Per plan |

Usage is billed per turn across speech recognition, the language model and
text-to-speech. If your balance runs out mid-session you receive an `Error`
frame and the socket closes, rather than the call continuing unbilled.
