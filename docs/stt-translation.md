---
title: "Audio Translation"
description: "Translate audio in any supported language to English text. OpenAI-compatible endpoint."
slug: "stt-translation"
breadcrumb: "Speech to Text"
---

# Audio Translation

Translate audio in any supported language to English text. OpenAI-compatible endpoint.

## Overview

The Audio Translation API translates speech in any of 24 supported languages to **English text**. This is the OpenAI-compatible `/v1/audio/translations` endpoint.

Unlike [Speech to Text](/docs/speech-to-text) (which transcribes in the original language), this endpoint always outputs English.

**Endpoint:** `POST /v1/audio/translations`

**Supported input languages:** Hindi, Bengali, Tamil, Telugu, Kannada, Malayalam, Marathi, Gujarati, Punjabi, Odia, Assamese, Urdu, Nepali, Konkani, Kashmiri, Sindhi, Sanskrit, Santali, Manipuri, Brij, Maithili, Dogri, English — plus auto-detection.

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
| `file` | file | Yes | Audio file (WAV, MP3, AAC, OGG, FLAC, WebM, M4A) |
| `model` | string | No | Model ID (default: `saaras:v3`) |
| `response_format` | string | No | `json` (default), `text`, or `verbose_json` |
| `temperature` | float | No | Sampling temperature |
| `prompt` | string | No | Prompt to guide transcription style |

## Response Formats

### json (default)

```json
{"text": "Hello, how are you?"}
```

### text

Returns plain text with no JSON wrapping.

### verbose_json

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