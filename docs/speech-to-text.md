---
title: "Speech to Text"
description: "Transcribe audio to text with our Indic saaras model and 22 Indic language support."
slug: "speech-to-text"
breadcrumb: "API Guides & Tutorials"
---

# Speech to Text

Transcribe audio to text with our Indic saaras model and 22 Indic language support.

:::cards
/docs/stt-realtime | Real-time STT | play | Stream audio for live transcription
/docs/stt-translation | Translation | settings | Transcribe and translate in one call
/docs/stt-batch | Batch | file-text | Process large audio files asynchronously
/docs/models-indic | Indic Models | mic | saaras:v3 and other Indic STT models
:::

## Overview

The Speech to Text API transcribes audio files into text. Uses our saaras:v3 model with support for 22 Indian languages + English. Supports auto language detection.

**Endpoint:** `POST /v1/audio/transcriptions`

### How transcription works

:::flow
icon:app | Your app | Upload an audio file (WAV/MP3) to `POST /v1/audio/transcriptions`
icon:gateway | CallMissed gateway | Validate the key, detect language (or use `language`), apply `mode`
icon:stt | saaras:v3 | Run speech recognition across 22 Indic languages + English
icon:done | Your app | Receive `text` (plus word timestamps in `verbose_json`)
:::

> **Tip:** Leave `language` unset and saaras:v3 auto-detects it. Set `mode=translate` to get English text out of any supported language in a single call.

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
| `model` | string | `saaras:v3` |
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