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
/docs/stt-batch | Batch | file-text | Process large audio files asynchronously
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
icon:done | Your app | Receive `text` (plus word timestamps in `verbose_json`)
:::

> **Tip:** Leave `model` unset to get `saaras:v3`, and leave `language` unset to let it auto-detect. Set `mode=translate` to get English text out of any supported language in a single call.

:::warning
**Streaming-only models cannot transcribe files.** `ink-2` and the Deepgram Flux
models do turn detection as part of the model, which only makes sense on a live
stream. Sending one here returns a 400 naming the file-transcription
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

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | `saaras:v3`, `saaras:v4`, `ink-whisper`, or any other file-transcription STT model ID. `ink-2` is **not** valid here — see [Cartesia Ink models](#cartesia-ink-models) |
| `file` | file | Audio file (WAV, MP3, etc.) |
| `language` | string | Language code (auto-detected if omitted) |
| `mode` | string | Output mode — see below |
| `response_format` | string | `json`, `text`, or `verbose_json` |
| `timestamp_granularities[]` | array | `["word"]` for word-level timestamps (OpenAI-compatible) |

## Output Modes

| Mode | Description |
|------|-------------|
| `transcribe` | Standard transcription (default) |
| `translate` | Transcribe and translate to English |
| `verbatim` | Exact transcription including filler words |
| `translit` | Transliteration to Latin script |
| `codemix` | Code-mixed output (Indic + English) |

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
| `ink-whisper` | $0.18 / hr | 100 (incl. Hindi, Urdu, Tamil) | Yes | Yes |
| `ink-2` | $0.54 / hr | English only (`en`) | **No** | Yes |

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
`/v1/audio/transcriptions` returns `400` rather than quietly substituting a
different model:

```json
{
  "detail": "ink-2 is a streaming-only model and is not available for file transcription. Use ink-whisper here, or ink-2 on a voice session."
}
```

Select it on a [voice session](/docs/voice-agent) or the
[Managed Voice Agent](/docs/managed-voice-agent) instead.

**2. It is English only.** The model accepts `en` and nothing else. Sending it
Hindi (or any other language) does **not** raise an upstream error — it silently
produces poor output. Our agent logs a warning and transcribes as `en`. For
non-English speech use `ink-whisper` (100 languages) or an Indic model such as
`saaras:v3` / `saaras:v4`.

## Deepgram feature parameters

When you select a Deepgram model (`deepgram-nova-3`, `deepgram-nova-2`, `deepgram-flux-general-en`, etc.), these extra form fields are accepted. They are ignored for non-Deepgram models. Model-restricted features are dropped automatically when the chosen model doesn't support them.

| Parameter | Type | Description |
|-----------|------|-------------|
| `diarize` | boolean | Label each speaker (`[Speaker 0]`, `[Speaker 1]`, …) |
| `utterances` | boolean | Segment the transcript into utterances |
| `utt_split` | number | Silence gap (seconds) used to split utterances |
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
| `keyterm` | string | Boost recognition of a term/phrase (Nova-3 + Flux; repeatable) |
| `keywords` | string | `keyword:intensifier` boost/suppress (Nova-2 / legacy; repeatable) |
| `search` | string | Phonetically search the audio for a term (repeatable) |
| `replace` | string | `find:replacement` substitution (repeatable) |

### Dialects & locales

Deepgram models accept locale-specific language codes so you can pin a dialect for best accuracy. Pass the code in the `language` field. Each model's exact dialect list is published in the `dialects` array on `GET /v1/models`. Examples:

- **English:** `en-US`, `en-GB`, `en-IN`, `en-AU`, `en-NZ`, `en-CA`, `en-IE`
- **Spanish:** `es`, `es-419` (Latin America)
- **Portuguese:** `pt-BR`, `pt-PT`
- **Chinese:** `zh-CN`, `zh-TW`, `zh-HK` (Cantonese)
- **Multilingual code-switching:** `multi` (Nova-3, Nova-2, Flux multilingual)
