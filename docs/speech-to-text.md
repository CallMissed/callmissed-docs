---
title: "Speech to Text"
description: "Transcribe audio to text across 45 STT models — Indic-first saaras, Cartesia Ink, and Deepgram Nova."
slug: "speech-to-text"
breadcrumb: "Speech"
---

# Speech to Text

Transcribe audio to text across 45 STT models — Indic-first saaras, Cartesia Ink, and Deepgram Nova.

:::cards
/docs/stt-realtime | Real-time STT | play | Stream audio for live transcription
/docs/stt-translation | Translation | settings | Transcribe and translate in one call
/docs/models-indic | Indic Models | mic | saaras:v3 and other Indic STT models
:::

## Overview

Transcribe an audio file into text with any of **45 speech-to-text models**. The default, `saaras:v3`, covers 22 Indian languages plus English with automatic language detection — but the `model` field takes any file-transcription STT id, so you can pick per request.

**Endpoint:** `POST /v1/audio/transcriptions`

### Picking a model

| If you need | Use | Why |
|---|---|---|
| Indian languages, or code-mixed Hinglish | `saaras:v3` or `saaras:v4` | Purpose-built for Indic phonetics; v4 adds 24 languages and serves all five modes |
| The widest language coverage | `ink-whisper` | 100 languages, and cheaper per hour than the Indic models |
| English call-centre audio | `deepgram-nova-3`, or `nova-3` on the free tier | Strong on accented and noisy telephony English |
| Turn detection built into the model | `ink-2` or `deepgram-flux-general-en` | Voice sessions only — see the note below |

Four models are on the **free tier**: `saaras:v3`, `saaras:v4`, `nova-3` and
`whisper-large-v3-turbo`.

:::note
Some models appear under two ids at different prices — `nova-3` and
`deepgram-nova-3` are the same underlying model, but only `nova-3` is on the free
tier. If you are on the free plan, use the id listed above; picking the other
spelling of the same model will bill you. Full per-model pricing:
[Models](/docs/models).
:::

### How transcription works

:::flow
icon:app | Your app | Upload an audio file (WAV/MP3) to `POST /v1/audio/transcriptions`
icon:gateway | CallMissed gateway | Validate the key, resolve `model`, detect language (or use `language`), apply `mode`
icon:stt | Your chosen model | Run speech recognition — defaults to `saaras:v3` if `model` is omitted
icon:done | Your app | Receive `text` (plus `duration` in `verbose_json`)
:::

> **Tip:** Leave `model` unset to get `saaras:v3`, and leave `language` unset to let it auto-detect. Set `mode=translate` to get English text out of any supported language in a single call.

:::warning
**Streaming-only models cannot transcribe files.** `ink-2` and the Deepgram Flux
models do turn detection as part of the model, which only makes sense on a live
stream. Sending one here returns an error naming the file-transcription
alternative rather than silently substituting a different model — see
[Cartesia Ink models](#cartesia-ink-models).
:::

## Basic Usage

:::tabs
```python [Python]
from openai import OpenAI

client = OpenAI(
    api_key="cm_your_key",
    base_url="https://api.callmissed.com/v1"
)

with open("audio.wav", "rb") as f:
    response = client.audio.transcriptions.create(
        model="saaras:v3",
        file=f
    )

print(response.text)
```
```javascript [JavaScript]
import OpenAI from "openai";
import fs from "fs";

const client = new OpenAI({
  apiKey: "cm_your_key",
  baseURL: "https://api.callmissed.com/v1",
});

const response = await client.audio.transcriptions.create({
  model: "saaras:v3",
  file: fs.createReadStream("audio.wav"),
});

console.log(response.text);
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/audio/transcriptions \
  -H "Authorization: Bearer cm_your_key" \
  -F file=@audio.wav \
  -F model=saaras:v3
```
:::

## Parameters

Send as `multipart/form-data`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file` | file | Yes | Audio file (WAV, MP3, etc.), up to 25 MB. An empty file returns `400`. Some models accept less per request: `saaras:v3` and `saaras:v4` take at most 30 seconds of audio, `gnani-prisma-v2.5` at most 60 seconds, and `whisper`, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe` and `gpt-4o-transcribe-diarize` at most 25,000,000 bytes |
| `model` | string | No | Default `saaras:v3`. Any file-transcription STT model ID from [Models](/docs/models). `ink-2` and the Flux models are **not** valid here — see [Cartesia Ink models](#cartesia-ink-models) |
| `language` | string | No | Language code; auto-detected if omitted. For `saaras:v3` / `saaras:v4` use a locale such as `hi-IN` or `ta-IN` — a code the model does not support falls back to auto-detection |
| `mode` | string | No | Output mode — see below. Default `transcribe`; any other value returns `422` |
| `response_format` | string | No | `json` (default), `text`, `verbose_json`, or `diarized_json`. Any other value is answered as `json` |
| `temperature` | number | No | 0–2. Accepted for OpenAI SDK compatibility; currently not forwarded to the model |
| `prompt` | string | No | Accepted for OpenAI SDK compatibility; currently not forwarded to the model |

## Response

`json` (default):

```json
{"text": "Namaste, aap kaise hain?"}
```

`text` returns the transcript as `text/plain`. `verbose_json` adds the billed
audio duration in seconds. `words` is always an empty array — word-level
timestamps are not returned. `segments` is empty except on
`gpt-4o-transcribe-diarize`, where each segment carries `id`, `speaker`,
`start`, `end` and `text`. `diarized_json` returns `task`, `duration`, `text`
and those speaker `segments`.

```json
{
  "task": "transcribe",
  "language": "hi-IN",
  "duration": 4.52,
  "text": "Namaste, aap kaise hain?",
  "segments": [],
  "words": []
}
```

`language` echoes the `language` you sent, or `"unknown"` when you let the model
auto-detect.

## Errors

Errors use the OpenAI envelope: `{"error": {"message", "type", "code"}}`.

| Status | `code` | When |
|--------|--------|------|
| `400` | `invalid_request` | `file` is empty; the audio or a parameter was rejected (format, size, language); or the model is streaming-only or not available for file transcription. The message says which |
| `400` | `audio_too_long` | The audio is longer than the model accepts per request (see `file` above). Use a model without a duration limit, such as `deepgram-nova-3` |
| `402` | `insufficient_credits` | Credit balance is exhausted |
| `403` | `permission_denied` | The API key does not have the `stt` permission, or the model is not available on your plan |
| `404` | `model_not_found` | `model` is not a known STT model ID |
| `422` | — | A form field fails validation (for example an unknown `mode`, or `temperature` outside 0–2) |
| `413` | `file_too_large` | The file is larger than 25 MB, or than the model accepts (see `file` above) |
| `429` | `quota_exceeded` | Plan usage limit reached |
| `429` | `rate_limit_exceeded` | Transcription is rate limited right now. Retry with backoff |
| `503` | `service_unavailable` | Transcription is temporarily unavailable. Retry shortly. `/v1/audio/speech` uses the same code |
| `503` | `provider_error`, `upstream_timeout`, `upstream_unavailable` | The model failed to transcribe the file. The response includes a `request_id` |

`/v1/audio/translations` returns the same errors.

## Output Modes

| Mode | Description |
|------|-------------|
| `transcribe` | Standard transcription (default) |
| `translate` | Transcribe and translate to English |
| `verbatim` | Exact transcription including filler words |
| `translit` | Transliteration to Latin script |
| `codemix` | Code-mixed output (Indic + English) |

`mode` is applied by `saaras:v3` and `saaras:v4`. `whisper-large-v3-turbo` also
honours `translate`. Every other model ignores `mode` and returns a plain
transcript.

`saaras:v4` serves all five modes on one model across 24 languages, and is free-tier like `saaras:v3`:

```bash
curl -X POST https://api.callmissed.com/v1/audio/transcriptions \
  -H "Authorization: Bearer cm_your_key" \
  -F file=@audio.wav \
  -F model=saaras:v4 \
  -F mode=codemix
```

## Cartesia Ink models

Two Cartesia STT models, and they are **not interchangeable** — one transcribes
files, the other only runs on a live voice session.

| Model | Price | Languages | File transcription | Voice sessions |
|-------|-------|-----------|--------------------|----------------|
| `ink-whisper` | $0.1875 / hr | 100 (incl. Hindi, Urdu, Tamil) | Yes | Yes |
| `ink-2` | $0.5625 / hr | 5: `en`, `fr`, `hi`, `ja`, `es` | **No** | Yes |

### `ink-whisper` — the cheapest 100-language option

Cartesia's fastest and most affordable STT, with better accuracy than baseline
Whisper. Its dynamic chunking cuts hallucination during pauses and silence, so
audio with dead air transcribes cleanly instead of inventing text to fill gaps.

```bash
curl -X POST https://api.callmissed.com/v1/audio/transcriptions \
  -H "Authorization: Bearer cm_your_key" \
  -F file=@audio.wav \
  -F model=ink-whisper \
  -F language=hi
```

### `ink-2` — voice agents only

Cartesia's top-ranked STT for voice agents: **8% WER** on AppTek's 14-accent
call-centre benchmark, against 10% for Deepgram Flux and 12% for ElevenLabs. It
also self-detects turns, so a voice agent needs no separate turn detector on top.

Two limits decide whether you can use it at all:

**1. It cannot transcribe files.** `ink-2` is streaming-only. POSTing it to
`/v1/audio/transcriptions` fails rather than quietly substituting a
different model. The request is not transcribed, and the error message names
the alternative:

```json
{
  "error": {
    "message": "ink-2 is a streaming-only model and is not available for file transcription. Use ink-whisper here, or ink-2 on a voice session.",
    "type": "invalid_request_error",
    "code": "invalid_request",
    "request_id": "stt-…"
  }
}
```

The HTTP status is `400`. `deepgram-flux-general-en` and `deepgram-flux-general-multi` fail the same way and point you to `deepgram-nova-3`.

Select it on a [voice session](/docs/voice-agent) or the
[Managed Voice Agent](/docs/managed-voice-agent) instead.

**2. It covers five languages.** The model accepts `en`, `fr`, `hi`, `ja` and
`es`. Sending it any other language (Tamil, for example) does **not** raise an
upstream error — it silently produces poor output. Our agent logs a warning and
transcribes as `en`. For other languages use `ink-whisper` (100 languages) or an
Indic model such as `saaras:v3` / `saaras:v4`.

## Deepgram feature parameters

When you select a Deepgram file-transcription model (`deepgram-nova-3`, `deepgram-nova-2`, `deepgram-enhanced`, etc.), these extra form fields are accepted. They are ignored for non-Deepgram models. Model-restricted features are dropped automatically when the chosen model doesn't support them.

| Parameter | Type | Description |
|-----------|------|-------------|
| `diarize` | boolean | Label each speaker (`[Speaker 0]`, `[Speaker 1]`, …) |
| `utterances` | boolean | Segment the transcript into utterances |
| `utt_split` | number | Silence gap (seconds, 0–10) used to split utterances |
| `paragraphs` | boolean | Split the transcript into paragraphs |
| `numerals` | boolean | Write numbers as digits (e.g. "five" → "5") |
| `measurements` | boolean | Abbreviate measurement units (English) |
| `dictation` | boolean | Convert spoken "comma"/"period" to punctuation (English) |
| `profanity_filter` | boolean | Mask recognized profanity with `****` |
| `filler_words` | boolean | Keep "uh"/"um" (Nova / Nova-2 / Nova-3) |
| `multichannel` | boolean | Transcribe each audio channel independently |
| `detect_entities` | boolean | Tag entities like names and locations (English) |
| `detect_language` | string | `true` to auto-detect, or repeat with codes to restrict |
| `redact` | string | `pci`, `pii`, `phi`, `numbers`, or a specific entity type (repeatable) |
| `keyterm` | string | Boost recognition of a term/phrase (Nova-3; repeatable) |
| `keywords` | string | `keyword:intensifier` boost/suppress (Nova-2 / legacy; repeatable) |
| `search` | string | Phonetically search the audio for a term (repeatable) |
| `replace` | string | `find:replacement` substitution (repeatable) |

### Dialects & locales

Deepgram models accept locale-specific language codes so you can pin a dialect for best accuracy. Pass the code in the `language` field. Each model's exact dialect list is published in the `dialects` array on `GET /v1/models`. Examples:

- **English:** `en-US`, `en-GB`, `en-IN`, `en-AU`, `en-NZ`, `en-CA`, `en-IE`
- **Spanish:** `es`, `es-419` (Latin America)
- **Portuguese:** `pt-BR`, `pt-PT`
- **Chinese:** `zh-CN`, `zh-TW`, `zh-HK` (Cantonese)
- **Multilingual code-switching:** `multi` (Nova-3, Nova-2)
