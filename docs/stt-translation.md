---
title: "Audio Translation"
description: "Translate audio in any supported language to English text. OpenAI-compatible endpoint."
slug: "stt-translation"
breadcrumb: "Speech"
---

# Audio Translation

Translate audio in any supported language to English text. OpenAI-compatible endpoint.

## Overview

Translates speech in any of 23 supported languages to **English text**. OpenAI-compatible `/v1/audio/translations`.

Unlike [Speech to Text](/docs/speech-to-text) (which transcribes in the original language), this endpoint always outputs English.

**Endpoint:** `POST /v1/audio/translations`

**Supported input languages (23, with `saaras:v3` / `saaras:v4`):** Hindi, Bengali, Tamil, Telugu, Kannada, Malayalam, Marathi, Gujarati, Punjabi, Odia, Assamese, Urdu, Nepali, Konkani, Kashmiri, Sindhi, Sanskrit, Santali, Manipuri, Bodo, Maithili, Dogri, English. The source language is always auto-detected — this endpoint takes no `language` field.

:::warning
**Only three models translate.** `saaras:v3`, `saaras:v4` and `whisper-large-v3-turbo` return English. Any other STT model accepted here returns a transcript in the **original** language, billed at that model's rate. Stick to those three for translation.
:::

## Basic Usage

:::tabs
```python [Python]
from openai import OpenAI

client = OpenAI(
    api_key="cm_your_key",
    base_url="https://api.callmissed.com/v1"
)

# Translate Hindi audio to English text
with open("hindi_audio.wav", "rb") as f:
    translation = client.audio.translations.create(
        model="saaras:v3",
        file=f,
    )

print(translation.text)
```
```javascript [JavaScript]
import OpenAI from "openai";
import fs from "fs";

const client = new OpenAI({
  apiKey: "cm_your_key",
  baseURL: "https://api.callmissed.com/v1",
});

const translation = await client.audio.translations.create({
  model: "saaras:v3",
  file: fs.createReadStream("hindi_audio.wav"),
});

console.log(translation.text);
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/audio/translations \
  -H "Authorization: Bearer cm_your_key" \
  -F model=saaras:v3 \
  -F file=@hindi_audio.wav
```
:::

**Response:**

```json
{"text": "Hello, how are you? I wanted to discuss the project."}
```

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file` | file | Yes | Audio file (WAV, MP3, AAC, OGG, FLAC, WebM, M4A), up to 25 MB. An empty file returns `400`. The same per-model size and duration limits as [transcription](/docs/speech-to-text) apply |
| `model` | string | No | Default `saaras:v3`. `saaras:v4` and `whisper-large-v3-turbo` also translate to English — see the warning above |
| `response_format` | string | No | `json` (default), `text`, or `verbose_json`. Any other value is answered as `json` |
| `temperature` | float | No | 0–2. Accepted for OpenAI SDK compatibility; currently not forwarded to the model |
| `prompt` | string | No | Accepted for OpenAI SDK compatibility; currently not forwarded to the model |

Errors match [Speech to Text](/docs/speech-to-text#errors): `400` empty file, `402` insufficient credits, `403` missing `stt` permission or model not on your plan, `404` unknown model, `422` invalid form field, `429` plan limit, `502` model failure.

## Response Formats

### json (default)

```json
{"text": "Hello, how are you?"}
```

### text

Returns plain text with no JSON wrapping.

### verbose_json

`duration` is the billed audio length in seconds. `segments` and `words` are always empty.

```json
{
  "task": "translate",
  "language": "en",
  "duration": 4.52,
  "text": "Hello, how are you?",
  "segments": [],
  "words": []
}
```

> **Tip:** For transcription in the original language (not translated), use [Speech to Text](/docs/speech-to-text) instead. For output modes like transliteration or code-mixing, use the `mode` parameter on the transcription endpoint.
