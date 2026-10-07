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
| `wss://api.callmissed.com/v2/voice/agent` | CallMissed-native |
| `wss://api.callmissed.com/v1/agent/converse` | Deepgram Voice Agent compatible |

If you already have an integration written against Deepgram's Voice Agent API,
point it at the second URL and it will work unchanged.

Both live on `api.callmissed.com`, the same host as the rest of the API — there
is no separate hostname to allowlist.

## Authentication

Both endpoints take an API key with `stt`, `tts` and `llm` permissions.

```http
Authorization: Token cm_your_api_key
```

`Authorization: Bearer cm_your_api_key` is accepted too.

Browsers cannot set headers on a WebSocket, so the subprotocol form is also
accepted:

```js
new WebSocket(url, ["token", "cm_your_api_key"])
```

A missing or invalid key, a key without those permissions, an exhausted plan
limit, the concurrent-session cap and a credit balance below the minimum are all
checked before the socket is accepted, so a refused connection fails at the
handshake rather than mid-conversation.

## Session flow

1. Connect. The server sends `Welcome` with a `request_id`.
2. Send `Settings` as your first message, within 15 seconds. Anything else
   first (including audio) ends the session with an `Error`.
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
    "tts": { "model": "bulbul:v3", "voice": "shubh" },
    "tools": [
      {
        "name": "lookup_order",
        "description": "Look up an order by id",
        "parameters": {
          "type": "object",
          "properties": { "order_id": { "type": "string" } },
          "required": ["order_id"]
        }
      }
    ]
  },
  "tags": ["support"]
}
```

| Field | Default | Notes |
| --- | --- | --- |
| `audio.input` / `audio.output` | `linear16` at `24000` Hz | See [Audio format](#audio-format) |
| `agent.prompt` | a concise built-in assistant prompt | String. Longer than 25,000 characters is truncated, with a `PROMPT_TOO_LONG` warning |
| `agent.greeting` | — | The first line the agent speaks |
| `agent.language` | `en-IN` | BCP-47. Falls back to `agent.stt.language`, then `agent.tts.language` |
| `agent.llm.model` | `gemma-4-31b` | Any id from [`GET /api/v1/voice/models`](#list-available-models) |
| `agent.llm.temperature` | model default | Number, 0–2 |
| `agent.stt.model` | `saaras:v3` | |
| `agent.tts.model` | plan-dependent | Free plan `bulbul:v3`; paid plans (starter, pro, enterprise) `sonic-3.6` |
| `agent.tts.voice` | model default | With no `agent.tts.model`: free plan `shubh`, paid plans `skylar`. With a model set, that model's own default speaker. A voice you set must be a speaker of the chosen model |
| `agent.tools` | `[]` | Functions the agent may call: `name` (required), `description`, `parameters` (JSON Schema) |
| `agent.record_calls` | `false` | Boolean. Records the session as one MP3 with both sides; fetch it afterwards with [`GET /v1/voice/sessions/{id}/recording`](/docs/voice-sessions-api#get-recording). The session is listed by `GET /v1/voice/sessions`. Transcripts are saved either way |
| `agent.recording_notice_enabled` | `true` | Boolean. With recording on, the agent first says a short notice that the call is recorded; `false` skips it |
| `agent.recording_notice` | a short default notice | String, at most 300 characters. The notice's wording |
| `tags` | `[]` | Array of strings |

`agent.llm`, `agent.stt` and `agent.tts` also accept a bare model id string as
shorthand, e.g. `"llm": "kimi-k2.5"`.

A model id the voice agent cannot serve fails `Settings` with an
`INVALID_SETTINGS` error naming it; it is never swapped for another model.
If a servable stack still cannot start (for example, a voice that is not a
speaker of the chosen model), the session runs on a backup stack. You get a
`STACK_SUBSTITUTED` warning naming the models in use, and the session is billed
for those models.

## Settings (Deepgram-compatible)

On `/v1/agent/converse`, send Deepgram's Voice Agent `Settings` shape unchanged,
with CallMissed model ids in `agent.listen.provider.model`,
`agent.think.provider.model` and `agent.speak.provider.model` (`model_id` is
accepted too). `agent.think.prompt`, `agent.think.functions`, `agent.greeting`,
the `audio` block and `tags` behave as on the native surface, and so do
`agent.record_calls`, `agent.recording_notice_enabled` and
`agent.recording_notice` (CallMissed additions; Deepgram has no equivalent).

Differences from Deepgram, each reported rather than silently dropped:

- A fallback array in `agent.think` or `agent.speak` uses only its first entry
  (`THINK_FALLBACK_IGNORED` / `SPEAK_FALLBACK_IGNORED` warning).
- Fields accepted but not applied (for example `listen.provider.keyterms`,
  `speak.provider.speed`, `think.provider.credentials`) are listed in one
  `SETTINGS_FIELDS_IGNORED` warning.
- `audio.output.container` must be `none` (raw frames).
- A function with a server-side `endpoint` is dispatched to your client instead
  (see [Tool calling](#tool-calling)).

Every other client message — updates, injection, tool responses, keepalive — is
identical across both protocols. Switching protocols means rewriting one message.
`Settings` is applied once; sending it again returns a `DUPLICATE_SETTINGS`
warning.

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
use, each with its measured latency. `Settings` accepts any id in that response;
filter on `eligible` to pick models measured fast enough to hold a conversation.
Any speech-to-text × language model × text-to-speech combination is valid.

Eligibility is based on **measured** performance from real traffic, not on
vendor claims. A model that is too slow to hold a conversation is marked
ineligible and listed with the reason, so you can see why rather than wondering
where it went.

Two speech-to-text models are streaming-capable here that you cannot use for file
transcription in the same way:

- **`ink-2`** ($0.5625 / hr) — Cartesia's top-ranked voice-agent STT: 8% WER on
  AppTek's 14-accent call-centre benchmark, vs 10% Deepgram Flux and 12%
  ElevenLabs. It self-detects turns. **English only** — set
  `"language": "en"`. Sending non-English audio does not error; it just
  transcribes badly. This is the only surface `ink-2` runs on: the file
  transcription endpoint rejects it with a `400`.
- **`ink-whisper`** ($0.1875 / hr) — 100 languages including Hindi, Urdu and Tamil.
  Cartesia's cheapest STT, with dynamic chunking that reduces hallucination
  across pauses. Use this instead of `ink-2` for any non-English call.

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

| Field | Meaning |
| --- | --- |
| `turn_budget_ms` | The voice-to-voice target the three stage budgets add up to |
| `id` / `label` | Model id to send, and its display name |
| `eligible` | `true` when `verdict` is `eligible` — the field to filter on |
| `p50_ms` / `samples` | Median measured latency for the model's stage over the last 30 days, and the number of measured turns behind it |
| `budget_ms` | The stage budget the model is judged against |
| `reason` | Why the model got its verdict |
| `languages` | BCP-47 codes where recorded; `[]` means not recorded |
| `voices` | Selectable speakers (TTS models only; `[]` otherwise) |

## Server events

| Event | Fields | Meaning |
| --- | --- | --- |
| `Welcome` | `request_id` | Socket open. |
| `SettingsApplied` | — | Configuration accepted; start streaming. |
| `ConversationText` | `role` (`user` / `assistant`), `content` | A finished turn. |
| `UserStartedSpeaking` | — | **Barge-in.** Stop playback and clear your buffer. |
| `AgentThinking` | `content` | The model is working. |
| `AgentStartedSpeaking` | `total_latency`, `tts_latency`, `ttt_latency` | First audio of a turn. Seconds. |
| `AgentAudioDone` | — | Last audio chunk *sent* for this turn. |
| `FunctionCallRequest` | `functions[]` | Run a tool and reply. |
| `LatencyReport` | `eou_latency`, `stt_latency`, `ttt_latency`, `tts_latency`, `total_latency` | Per-stage timings for the turn. Seconds. |
| `PromptUpdated` / `ThinkUpdated` / `ListenUpdated` / `SpeakUpdated` | — | The matching update was applied. |
| `InjectionRefused` | — | An `InjectAgentMessage` was refused (see below). |
| `Warning` | `code`, `description` | Non-fatal; the session continues. |
| `Error` | `code`, `description` | Fatal; the socket closes. Reconnect to start a new session. |

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

### Error codes

| `code` | Cause |
| --- | --- |
| `SETTINGS_TIMEOUT` | No `Settings` within 15 seconds of `Welcome` |
| `NON_SETTINGS_MESSAGE_BEFORE_SETTINGS` | The first message was not `Settings` |
| `INVALID_JSON` | `Settings` was not valid JSON |
| `INVALID_SETTINGS` | A field failed validation, or a model id cannot be served |
| `INSUFFICIENT_CREDITS` | Your balance ran out mid-session |
| `SESSION_TIMEOUT` | The 2-hour session cap was reached |
| `AGENT_ERROR` / `INTERNAL_ERROR` | The voice engine could not continue or could not start |

Common `Warning` codes: `INVALID_UPDATE`, `THINK_UPDATE_FAILED`,
`LISTEN_UPDATE_FAILED`, `SPEAK_UPDATE_FAILED`, `FUNCTION_CALL_TIMEOUT`,
`UNMATCHED_FUNCTION_RESPONSE`, `FUNCTION_ENDPOINT_UNSUPPORTED`,
`UNKNOWN_MESSAGE`, `DUPLICATE_SETTINGS`, `INVALID_JSON`, `MESSAGE_TOO_LARGE`,
`AUDIO_CHUNK_TOO_LARGE` (the frame is dropped), `PROMPT_TOO_LONG`,
`STACK_SUBSTITUTED`.

## Tool calling

Declare tools in `Settings` (`agent.tools` on the native surface,
`agent.think.functions` on the Deepgram-compatible one), then answer requests
over the same socket.

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

Reply with the result. `arguments` is a JSON **string**, not an object. Echo the
`id`; `content` should be a string (anything else is JSON-encoded for you).

```json
{
  "type": "FunctionCallResponse",
  "id": "fc_01H...",
  "name": "lookup_order",
  "content": "{\"status\":\"shipped\",\"eta\":\"2 days\"}"
}
```

A tool that does not answer within 30 seconds does not hang the call — the agent
tells the caller it could not complete the action, and a `FUNCTION_CALL_TIMEOUT`
warning is emitted.

Server-side execution (a `functions[].endpoint`) is **not** supported. Declaring
one returns a `Warning` and the call is dispatched to your client instead, so an
unsupported mode is never silently ignored.

## Updating a live session

| Message | Body | Effect | Acknowledged by |
| --- | --- | --- | --- |
| `UpdatePrompt` | `prompt` | **Replaces** the system prompt | `PromptUpdated` |
| `UpdateThink` | `model`, optional `temperature`, `prompt` | Switch the language model (and optionally the prompt) | `ThinkUpdated` |
| `UpdateListen` | `model`, optional `language` | Switch speech recognition | `ListenUpdated` |
| `UpdateSpeak` | `model`, `voice`, optional `language` | Switch voice or text-to-speech model | `SpeakUpdated` |
| `InjectAgentMessage` | `message`, optional `behavior` | Make the agent say something now | `InjectionRefused` if refused |
| `InjectUserMessage` | `content` | Inject text as if the caller said it | — |
| `KeepAlive` | — | Hold an idle socket open (does not extend the 2-hour cap) | — |

```json
{ "type": "UpdateSpeak", "model": "sonic-3.6", "voice": "skylar" }
```

The update bodies can be sent flat as above, or nested the Deepgram way
(`{"type": "UpdateSpeak", "speak": {"provider": {"model": "sonic-3.6"}}}`) on
either surface.

`InjectAgentMessage` `behavior` is `default`, `queue` or `interrupt`. `default`
is refused while either side is speaking, `queue` only while the caller is
speaking, and `interrupt` is never refused.

A model id the service cannot serve keeps the current model and returns a
`Warning`; it is never swapped for a different one behind your back.

## Latency

The fast path targets **sub-1s** from end of your speech to first audio back.
That budget is the sum of three serial stages:

| Stage | Budget |
| --- | --- |
| Speech recognition, final transcript | 300 ms |
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
| Time to send `Settings` | 15 seconds |
| Tool response timeout | 30 seconds |
| Concurrent sessions | Per plan |

Usage is billed per turn across speech recognition, the language model and
text-to-speech, at each model's own rate. If your balance runs out mid-session
you receive an `INSUFFICIENT_CREDITS` `Error` frame and the socket closes,
rather than the call continuing unbilled.
