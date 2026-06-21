---
title: "Text to Speech"
description: "Convert text to natural-sounding speech with our Indic TTS."
slug: "text-to-speech"
breadcrumb: "API Guides & Tutorials"
---

# Text to Speech

Convert text to natural-sounding speech with our Indic TTS.

## Overview

The Text to Speech API converts text into audio — Indic languages, 37 voices across 11 languages.

**Endpoint:** `POST /v1/audio/speech`

:::flow
icon:app | Your app | Send text + a `voice` and `language` to `POST /v1/audio/speech`
icon:tts | bulbul:v3 | Synthesize speech in the chosen voice and `response_format`
icon:done | Your app | Receive the audio stream and play or save it
:::

## Basic Usage

:::tabs
```python [Python]
from openai import OpenAI

client = OpenAI(
    api_key="cm_your_key",
    base_url="https://api.callmissed.com/v1"
)

response = client.audio.speech.create(
    model="bulbul:v3",
    voice="shubh",
    input="Namaste, kaise hain aap?"
)

response.stream_to_file("speech.mp3")
```
```javascript [JavaScript]
import OpenAI from "openai";
import fs from "fs";

const client = new OpenAI({
  apiKey: "cm_your_key",
  baseURL: "https://api.callmissed.com/v1",
});

const response = await client.audio.speech.create({
  model: "bulbul:v3",
  voice: "shubh",
  input: "Namaste, kaise hain aap?",
});

const buffer = Buffer.from(await response.arrayBuffer());
fs.writeFileSync("speech.mp3", buffer);
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "bulbul:v3", "input": "Namaste, kaise hain aap?", "voice": "shubh"}' \
  --output speech.mp3
```
:::

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | `bulbul:v3` |
| `input` | string | Text to synthesize |
| `voice` | string | Voice ID — default `shubh` (37 voices available) |
| `language` | string | Language code (e.g. `hi-IN`, `ta-IN`) |
| `speed` | number | Speech speed 0.5–2.0 (default 1.0) |
| `speech_sample_rate` | integer | 8000, 16000, 22050, 24000, or 48000 Hz |
| `response_format` | string | Output format — see below |

## Audio Formats

Supported values for `response_format`: `mp3`, `opus`, `aac`, `flac`, `wav`, `pcm`