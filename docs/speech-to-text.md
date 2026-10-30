---
title: "Speech to Text"
description: "Transcribe audio to text across 47 STT models — Indic-first saaras, Cartesia Ink, and Deepgram Nova."
slug: "speech-to-text"
breadcrumb: "Speech"
---

# Speech to Text

Transcribe audio to text across 47 STT models — Indic-first saaras, Cartesia Ink, and Deepgram Nova.

:::cards
/docs/stt-realtime | Real-time STT | play | Stream audio for live transcription
/docs/stt-translation | Translation | settings | Transcribe and translate in one call
/docs/models-indic | Indic Models | mic | saaras:v4 and other Indic STT models
:::

## Overview

Transcribe an audio file into text with any of **47 speech-to-text models**. The default, `saaras:v4`, covers 22 Indian languages plus English with automatic language detection — but the `model` field takes any file-transcription STT id, so you can pick per request.

**Endpoint:** `POST /v1/audio/transcriptions`

### Picking a model

| If you need | Use | Why |
|---|---|---|
| Indian languages, or code-mixed Hinglish | `saaras:v4` | Purpose-built for Indic phonetics; 23 languages, all five output modes |
| The widest language coverage | `ink-whisper` | 100 languages, and cheaper per hour than the Indic models |
| English call-centre audio | `deepgram-nova-3` | Strong on accented and noisy telephony English |
| Clinical or pharmaceutical audio | `deepgram-nova-3-medical` or `deepgram-nova-3-pharma` | Tuned for medical terms and drug names; `deepgram-nova-3-pharma` is English only |
| Long recordings, up to 2 hours per file (preview) | `mai-transcribe-2` | 60 languages with automatic detection — see [MAI-Transcribe-2](#mai-transcribe-2-preview) |
| Turn detection built into the model | `ink-2`, `assemblyai-universal-3.6-pro` or `deepgram-flux-general-en` | Voice sessions only — see the note below |

One model is on the **free tier**: `saaras:v4`. The retired `nova-3` and
`whisper-large-v3-turbo` IDs still work and stay free; they are answered by
`deepgram-nova-3` and `whisper` — see
[Retired and deprecated models](/docs/models#retired-and-deprecated-models).

:::note
The retired `nova-3` ID and `deepgram-nova-3` are the same model at different
prices: `nova-3` stays on the free plan at its old rate. Full per-model pricing:
[Models](/docs/models).
:::

### How transcription works

:::flow
icon:app | Your app | Upload an audio file (WAV/MP3) to `POST /v1/audio/transcriptions`
icon:gateway | CallMissed gateway | Validate the key, resolve `model`, detect language (or use `language`), apply `mode`
icon:stt | Your chosen model | Run speech recognition — defaults to `saaras:v4` if `model` is omitted
icon:done | Your app | Receive `text` (plus `duration` in `verbose_json`)
:::

> **Tip:** Leave `model` unset to get `saaras:v4`, and leave `language` unset to let it auto-detect. Set `mode=translate` to get English text out of any supported language in a single call.

:::warning
**Streaming-only models cannot transcribe files.** `ink-2`, the Deepgram Flux
models, `assemblyai-universal-3.6-pro` and `inworld-stt-1` do turn detection as part of the model, which only makes sense on a live
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
        model="saaras:v4",
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
  model: "saaras:v4",
  file: fs.createReadStream("audio.wav"),
});

console.log(response.text);
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/audio/transcriptions \
  -H "Authorization: Bearer cm_your_key" \
  -F file=@audio.wav \
  -F model=saaras:v4
```
:::

## Parameters

Send as `multipart/form-data`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file` | file | Yes | Audio file (WAV, MP3, etc.), up to 25 MB. An empty file returns `400`. Some models accept less per request: `saaras:v4` takes at most 30 seconds of audio, `gnani-prisma-v2.5` at most 60 seconds, and `whisper`, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe` and `gpt-4o-transcribe-diarize` at most 25,000,000 bytes |
| `model` | string | No | Default `saaras:v4`. Any file-transcription STT model ID from [Models](/docs/models). `ink-2` and the Flux models are **not** valid here — see [Cartesia Ink models](#cartesia-ink-models) |
| `language` | string | No | Language code; auto-detected if omitted. For `saaras:v4` use a locale such as `hi-IN` or `ta-IN` — a code the model does not support falls back to auto-detection |
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

`mode` is applied by `saaras:v4`. `whisper` also
honours `translate`. Every other model ignores `mode` and returns a plain
transcript.

`saaras:v4` serves all five modes on one model across 23 languages, and is on the free tier:

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
Indic model such as `saaras:v4`.

## Voice-session-only models

Three more speech-to-text models run only on a live voice session, selected with `stt_model` on an agent or a [voice session](/docs/voice-sessions-api). POSTing any of them to `/v1/audio/transcriptions` returns `400`. For files, use `saaras:v4`, `ink-whisper` or `deepgram-nova-3`.

| Model | Price | Languages | Turn detection |
|-------|-------|-----------|----------------|
| `assemblyai-universal-3.6-pro` | $0.4688 / hr | 50 | Built into the model |
| `inworld-stt-1` | $0.1563 / hr | English only (`en`, `en-US`) | Not built in |
| `microsoft-speech-stt` | $1.0417 / hr | 81 (148 locales), one per call | Not built in |

`assemblyai-universal-3.6-pro` is AssemblyAI's newest streaming model. It ends the caller's turn itself, so the agent needs no separate turn detector on top, and it auto-detects and code-switches, so the language you set is a hint. Languages include English (with `en-IN`, `en-US`, `en-GB`, `en-AU`, `en-CA`, `en-NZ`), Spanish, French, German, Italian, Portuguese, Turkish, Dutch, Swedish, Norwegian, Danish, Finnish, Hindi, Marathi, Urdu, Vietnamese, Arabic, Hebrew, Japanese, Chinese, Cantonese, Russian, Korean, Catalan, Galician, Romanian, Estonian, Persian, Afrikaans, Zulu and Xhosa.

`inworld-stt-1` is Inworld's low-cost streaming model and hears English only. For any other language use `assemblyai-universal-3.6-pro`, `saaras:v4` or `deepgram-nova-3`.

`microsoft-speech-stt` is Microsoft's real-time streaming recognizer, processed in India. It hears 81 languages (148 locales), including Hindi, Bengali, Tamil, Telugu, Marathi, Gujarati, Kannada, Malayalam, Punjabi, Odia, Assamese, Urdu, Bhojpuri and Indian English. It recognises one language per call: it does not auto-detect or follow a caller who switches language, so set the agent's language to the one the caller speaks. It has no turn detection of its own, so the agent's turn detection applies. Keyterms you set on the agent (`stt_keyterms`) are applied as a phrase list.

## `mai-transcribe-2` (preview)

`mai-transcribe-2` is Microsoft AI's MAI-Transcribe-2, $0.375 / hour. **It is preview.** It transcribes files on `POST /v1/audio/transcriptions` only: it is not available in voice sessions, and `POST /v1/audio/translations` refuses it.

- **Languages.** 60: `af`, `ar`, `as`, `az`, `bg`, `bn`, `bs`, `ca`, `cs`, `da`, `de`, `el`, `en`, `es`, `et`, `fa`, `fi`, `fil`, `fr`, `gl`, `gu`, `he`, `hi`, `hu`, `hy`, `id`, `is`, `it`, `ja`, `kk`, `kn`, `ko`, `lt`, `lv`, `mk`, `ml`, `mr`, `ms`, `nb`, `ne`, `nl`, `or`, `pa`, `pl`, `pt`, `ro`, `ru`, `sk`, `sl`, `sv`, `sw`, `ta`, `te`, `th`, `tr`, `uk`, `ur`, `vi`, `yue`, `zh`.
- **Language detection.** Leave `language` out and the language is detected automatically, including Hinglish and other code-switching. `language` takes one code; `hi` and `hi-IN` both work. A language not in the list returns `400`.
- **Files.** Up to 2 hours of audio per file and the route's 25 MB upload limit. WAV, MP3, FLAC and Ogg are accepted. `response_format` is `json`, `text` or `verbose_json`.
- **Plan and hosting.** Paid plans only, never a default, and not processed in India.
- **Preview.** Microsoft ships this model as a public preview with no SLA, and may change or retire it at short notice. If it is retired, the ID keeps working and is answered by a named successor, as described in [Retired and deprecated models](/docs/models#retired-and-deprecated-models), so the transcript may differ. Use a generally available model for production-critical workloads.

```bash
curl -X POST https://api.callmissed.com/v1/audio/transcriptions \
  -H "Authorization: Bearer cm_your_key" \
  -F "model=mai-transcribe-2" \
  -F "file=@meeting.mp3"
```

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
| `keyterm` | string | Boost recognition of a term/phrase (Nova-3, including medical and pharma; repeatable) |
| `language_hint` | string | Steer `language=multi` toward the languages you expect, e.g. `language_hint=hi` and `language_hint=en`. `deepgram-nova-3` only, and only with `language=multi`; ignored otherwise, as are codes Deepgram does not list. Repeatable |
| `keywords` | string | `keyword:intensifier` boost/suppress (Nova-2 / legacy; repeatable) |
| `search` | string | Phonetically search the audio for a term (repeatable) |
| `replace` | string | `find:replacement` substitution (repeatable) |

### Dialects & locales

Deepgram models accept locale-specific language codes so you can pin a dialect for best accuracy. Pass the code in the `language` field. Each model's exact dialect list is published in the `dialects` array on `GET /v1/models`. Examples:

- **English:** `en-US`, `en-GB`, `en-IN`, `en-AU`, `en-NZ`, `en-CA`, `en-IE`
- **Spanish:** `es`, `es-419` (Latin America)
- **Portuguese:** `pt-BR`, `pt-PT`
- **Chinese:** `zh-CN`, `zh-TW`, `zh-HK` (Cantonese)
- **Multilingual code-switching:** `multi` (Nova-3, Nova-3 Medical, Nova-2). Nova-3 Medical covers 10 languages in `multi`; `deepgram-nova-3-pharma` is English only
