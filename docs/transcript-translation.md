---
title: "Transcript Translation"
description: "Translate a finished call's transcript into Hindi or English, turn by turn, with numbers, amounts and phone numbers written as digits kept exactly as they appear. On request, or automatically after every call."
slug: "transcript-translation"
breadcrumb: "Voice Agents"
---

# Transcript Translation

Translate a finished call's transcript into Hindi or English, turn by turn, with numbers, amounts and phone numbers written as digits kept exactly as they appear. On request, or automatically after every call.

## Overview

A call transcript is in whatever language the caller spoke. Transcript translation gives your team the same call in **Hindi** (`hi`) or **English** (`en`):

- **Every turn**, caller and agent, in call order. Speaker sides never move.
- **Values kept exactly.** Numbers, amounts, phone numbers, times, dates, order ids, emails and links written as digits or symbols are taken out before the text is translated and put back afterwards, so a translation cannot change them. A number the transcript spells out in words ("nine eight seven") is translated as words. Names are kept as written.
- **Stored.** Asking again returns the stored translation and costs nothing. Regenerate with `force`.
- **On request or automatic.** Translate any finished call yourself, or set an agent to translate every call after it ends.

The source can be any language the call was transcribed in, including mixed languages such as Hinglish. Lines already in the target language come back unchanged.

Machine translation can still miss nuance. Treat it as a reading aid for your team, not as a record of what was agreed. AI behavior is not guaranteed and may vary.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

The same access as reading the [transcript](/docs/voice-sessions-api): an API key with the `stt`, `tts` and `llm` permissions reads the sessions it created; a session another key or the console created also needs the `telephony:read` scope. Base URL: `https://api.callmissed.com`. Errors are `{"detail": "..."}`.

## Translate a call

```
POST /v1/voice/sessions/{session_id}/translations
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `language` | string | yes | `hi` (Hindi, Devanagari script) or `en` (English) |
| `force` | boolean | no | Translate again even if a translation is stored. Default `false` |

```bash
curl -X POST https://api.callmissed.com/v1/voice/sessions/SESSION_ID/translations \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"language": "en"}'
```

```json
{
  "session_id": "6f1c2a8e-3b7d-4e0f-9a21-5c8d7e6f4b30",
  "language": "en",
  "status": "completed",
  "turns": [
    {"turn_index": 0, "user_transcript": null, "agent_response": "Hello, how can I help you?", "kept_original": false},
    {"turn_index": 1, "user_transcript": "Where is my order ORD-1042? I paid ₹1,500.", "agent_response": "It ships on 12/11. Call 98765 43210 for help.", "kept_original": false}
  ],
  "source": "request",
  "model": "gpt-6-luna",
  "error": null,
  "cost_credits": 0.04,
  "translated_at": "2026-11-20T10:15:02Z",
  "created_at": "2026-11-20T10:14:58Z",
  "updated_at": "2026-11-20T10:15:02Z"
}
```

- Only a **finished** call can be translated.
- A call that already has a translation in that language returns it unchanged and is **not billed again**, unless `force` is `true`.
- `kept_original: true` marks a turn where a line is shown in its original language because it could not be translated with its numbers and other values intact.
- `source` is `request` or `auto` (translated after the call by the agent setting below).
- The request waits for the translation; a long call can take tens of seconds.

## Get a stored translation

```
GET /v1/voice/sessions/{session_id}/translations/{language}
```

Free. Returns `{"session_id", "language", "translation"}`, with `translation` set to the object above, or `null` when the call has not been translated into that language.

## Translate every call automatically

Set the agent's `transcript_translation_language` to `hi` or `en` and each call is translated after it ends. Leave it unset (the default) to translate only when asked.

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/BOT_ID/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"values": {"transcript_translation_language": "hi"}}'
```

- Applies to calls that **end after you save it**. Turning it on never translates calls you already have.
- Calls where nobody but the agent spoke are skipped.
- Each automatic translation is billed like one made on request.
- In the console: the agent's **Analysis** tab, **Translate transcripts**.

## Billing

Each translation runs one or more language-model completions and is billed in credits at that model's per-token [LLM rate](/docs/pricing); `cost_credits` on the result is what it cost. Reading a stored translation is free.

## Privacy

- A call under [zero data retention](/docs/voice-data-retention) keeps no transcript, so it cannot be translated.
- When a retention window deletes a call's transcript, its translations are deleted with it.
- If the agent redacts card, Aadhaar, PAN, OTP and CVV numbers from transcripts, the translation is made from the redacted text and the placeholders stay as they are.
- Translations are not sent in webhooks or call exports. Read them with the endpoints above.

## Errors

| Status | When |
| --- | --- |
| `402` | Not enough credits, or the monthly budget cap is reached |
| `403` | The API key cannot read this session |
| `404` | No such session in your workspace |
| `409` | The call is still in progress, has no transcript, is under zero data retention, its transcript was deleted, is too long to translate, or is already being translated |
| `422` | `language` is not `hi` or `en` |
| `503` | Translation is unavailable right now. Retry |

## MCP

Two tools on the [Account MCP server](/docs/agent-tools-mcp): `get_transcript_translation` (free) and `translate_call_transcript` (costs credits).
