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
handshake rather than mid-conversation. A refused handshake fails the WebSocket
upgrade with HTTP `403` whatever the reason; no `Error` frame is sent, because
the socket never opened. Check the key, its permissions and your balance first.

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
| `agent.llm.model` | `gemma-4-31b` | Any LLM id from [`GET /api/v1/voice/models`](#list-available-models) except `deepgram-voice-*` (see [Choosing models](#choosing-models)) |
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
shorthand, e.g. `"llm": "kimi-k2.5"`. Model ids, `voice`, `language` and
`greeting` must be strings; any other type fails `Settings` with
`INVALID_SETTINGS`.

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
accepted too).

Deepgram's own model names are accepted where CallMissed serves the same model,
in `Settings` and in `UpdateListen` / `UpdateSpeak`:

| You send (Deepgram) | Runs as |
| --- | --- |
| `listen`: a Deepgram speech-to-text model, e.g. `flux-general-en`, `flux-general-multi`, `nova-3-medical`, `nova-2-phonecall` | `deepgram-<model>`, the same Deepgram model |
| `speak`: an Aura-2 model, e.g. `aura-2-thalia-en` | `deepgram-aura-2` with voice `thalia` |
| `speak`: an Aura-1 model, e.g. `aura-asteria-en` | `deepgram-aura-1` with voice `asteria` |
| `think`: an OpenAI model CallMissed serves under the same name, e.g. `gpt-4o`, `gpt-4.1`, `gpt-5-mini`, `gpt-5.5` | unchanged |

An id that is already a CallMissed id is never re-routed (`nova-3` stays
`nova-3`). Anything else, including `think` models CallMissed does not serve
(for example `gpt-4o-mini` or Anthropic and Google models) and Flux TTS voices
(`flux-<voice>-<lang>`), fails `Settings` with `INVALID_SETTINGS` naming the id.
It is never swapped for a similar model. `agent.think.prompt`, `agent.think.functions`, `agent.greeting`,
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

Not supported on `/v1/agent/converse`:

- Audio encodings other than `linear16` (no `mulaw`, `alaw`, `opus`, `mp3` and so
  on), and `wav` / `ogg` output containers.
- A reusable agent configuration id in place of the `agent` object.
- Resuming a conversation with `agent.context.messages`, `History` events, and
  `ForceEndTurn` (answered with an `UNKNOWN_MESSAGE` warning).
- `defer_until_eot` on functions, and fallback entries after the first in
  `agent.think` / `agent.speak`.
- Multilingual `ConversationText` language fields.

The update, injection, tool-response and keepalive messages are the same on both
protocols, with two differences on `/v1/agent/converse`, both matching Deepgram:
`UpdatePrompt` **adds** to the current prompt instead of replacing it, and a
second `Settings` is a fatal `SETTINGS_ALREADY_APPLIED` error. On the native
endpoint a second `Settings` gets a `DUPLICATE_SETTINGS` warning and the session
continues.

### Audio format

`linear16` (raw 16-bit PCM, little-endian, mono) in both directions, at
`8000`, `16000`, `24000`, `32000` or `48000` Hz. Output is raw frames with no
container: a WAV or OGG header would be read as audio by most telephony
consumers.

An unsupported encoding, sample rate or container is rejected at `Settings`
time on both endpoints, rather than accepted and quietly changed, so you find
out at the handshake instead of hearing the wrong thing on a call.

Send audio in small frames, as it is captured (20–100 ms per frame is typical).
A single frame carrying more than 5 seconds of audio at your input sample rate
is dropped with an `AUDIO_CHUNK_TOO_LARGE` warning. Use an even number of bytes
per frame: an odd trailing byte is not a whole 16-bit sample and is discarded.

## Choosing models

Call [`GET /api/v1/voice/models`](#list-available-models) for the models you can
use, each with its measured latency. `Settings` accepts any id in that response
except the managed `deepgram-voice-*` language models, which run only on
[Voice Agent](/docs/voice-agent) (WebRTC) sessions; naming one here fails
`Settings` with `INVALID_SETTINGS`. Filter on `eligible` to pick models measured fast enough to
hold a conversation. Any speech-to-text × language model × text-to-speech
combination is valid.

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
| `FunctionCallCancelled` | `functions[]` (`id`, `name`) | A request you already received was abandoned. Stop the work and do not reply. |
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

An interrupted reply's assistant `ConversationText` holds only the part the
caller heard. If the caller barges in within the first fraction of a second of
a reply, before any of it was heard, no assistant `ConversationText` is sent for
that turn.

Latency fields are omitted when a stage was not measured, rather than reported
as `0`.

### Error codes

Where Deepgram has its own name for a code, `/v1/agent/converse` sends
Deepgram's name; the native endpoint keeps its own.

| Native `code` | On `/v1/agent/converse` | Cause |
| --- | --- | --- |
| `SETTINGS_TIMEOUT` | `CLIENT_MESSAGE_TIMEOUT` | No `Settings` within 15 seconds of `Welcome` |
| `NON_SETTINGS_MESSAGE_BEFORE_SETTINGS` | same | The first message was not `Settings` |
| `INVALID_JSON` | `UNPARSABLE_CLIENT_MESSAGE` | `Settings` was not valid JSON, or not a JSON object |
| `INVALID_SETTINGS` | same | A field failed validation, a model id cannot be served, or `Settings` is too large |
| — | `SETTINGS_ALREADY_APPLIED` | A second `Settings` arrived |
| `CONCURRENCY_LIMIT` | same | Your plan's concurrent-session cap was reached while the session started |
| `MODEL_UNAVAILABLE` | same | No voice engine could be built for this configuration |
| `INSUFFICIENT_CREDITS` | same | Your balance ran out mid-session |
| `API_KEY_BUDGET_EXCEEDED` | same | The API key reached its spend limit, or was revoked or expired, mid-session |
| `SESSION_TIMEOUT` | `MAXIMUM_SESSION_LENGTH_REACHED` | The 2-hour session cap was reached |
| `AGENT_ERROR` | same | The voice engine could not continue |
| `INTERNAL_ERROR` | `INTERNAL_SERVER_ERROR` | The voice engine could not start |

Common `Warning` codes: `INVALID_UPDATE`, `THINK_UPDATE_FAILED`,
`LISTEN_UPDATE_FAILED`, `SPEAK_UPDATE_FAILED`, `FUNCTION_CALL_TIMEOUT`,
`UNMATCHED_FUNCTION_RESPONSE`, `FUNCTION_ENDPOINT_UNSUPPORTED`,
`UNKNOWN_MESSAGE`, `DUPLICATE_SETTINGS` (native only), `INVALID_JSON`
(`UNPARSABLE_CLIENT_MESSAGE` on converse), `MESSAGE_TOO_LARGE`,
`AUDIO_CHUNK_TOO_LARGE` (the frame is dropped), `PROMPT_TOO_LONG`,
`STACK_SUBSTITUTED`, `INTERNAL_ERROR` (`INTERNAL_SERVER_ERROR` on converse: one
message could not be applied, the session continues), and, on
`/v1/agent/converse` only, `MAXIMUM_SESSION_LENGTH_APPROACHING` five minutes
before the 2-hour cap.

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

If the agent abandons a request it already sent you, you receive
`FunctionCallCancelled` with that request's `id` and `name`. Stop the work and
do not reply; a late `FunctionCallResponse` for it is dropped.

## Updating a live session

| Message | Body | Effect | Acknowledged by |
| --- | --- | --- | --- |
| `UpdatePrompt` | `prompt` | Native: **replaces** the system prompt. `/v1/agent/converse`: **adds** to it, as Deepgram does. The result is capped at 25,000 characters (`PROMPT_TOO_LONG`) | `PromptUpdated` |
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
`Warning`; it is never swapped for a different one behind your back. A
`model`, `voice` or `language` that is not a string returns an `INVALID_UPDATE`
warning and changes nothing.

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
| Prompt length | 25,000 characters |
| Audio in one binary frame | 5 seconds at your input sample rate |
| Concurrent sessions | Per plan |

JSON control messages are small; one far larger than any legitimate update is
dropped with a `MESSAGE_TOO_LARGE` warning. A `Settings` message whose prompt is
within the 25,000-character limit always fits.

Connection attempts are also admission-controlled before the API key is
checked. Opening many sessions at once, or reconnecting in a tight loop, from
one server can be refused at the handshake before your plan's concurrent-session
cap is reached. Reconnect with backoff, and contact sales@callmissed.com if you
need sustained high concurrency from one server.

There is no idle timeout. A connected session keeps running, and keeps
transcribing whatever audio you send, until you close the socket or it reaches
the 2-hour cap, so close it when the conversation ends.

Usage is billed per turn across speech recognition, the language model and
text-to-speech, at each model's own rate. If your balance runs out mid-session
you receive an `INSUFFICIENT_CREDITS` `Error` frame and the socket closes,
rather than the call continuing unbilled. An API key with a spend limit is
checked after every billed turn as well; once the key reaches its limit you
receive `API_KEY_BUDGET_EXCEEDED` and the socket closes.

### Not available on this endpoint

A session here is configured entirely by its `Settings` message; there is no
`bot_id`. Features loaded from a saved agent are not applied: knowledge base,
the built-in [agent tools](/docs/voice-agent-tools), pronunciations,
`{{variables}}`, PII redaction, background sound, noise suppression, and the
no-input nudge and hang-up. For those, create the session with the
[Voice Session API](/docs/voice-sessions-api) and a `bot_id`.
