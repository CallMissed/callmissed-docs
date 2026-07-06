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
| `stream` | boolean | Deepgram Aura-2 only — stream audio as it's generated (lower time-to-first-byte) |

## Audio Formats

Supported values for `response_format`: `mp3`, `opus`, `aac`, `flac`, `wav`, `pcm`

## Streaming (Deepgram Aura-2)

For `deepgram-aura-2`, set `"stream": true` to receive audio frames as they're synthesized over Deepgram's low-latency WebSocket, relayed to you as a chunked HTTP response. Ideal for real-time playback where you want the first audio bytes as fast as possible.

Streaming supports **raw encodings only** — `response_format` must be `linear16` (or `pcm`/`wav`), `mulaw`, or `alaw`. Compressed formats (`mp3`, `opus`, `aac`, `flac`) are not WebSocket-streamable; if you request one with `stream:true`, the full audio is returned in one buffered response instead.

```bash [cURL]
curl -N -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "deepgram-aura-2", "voice": "thalia", "input": "Streaming hello.", "response_format": "linear16", "stream": true}' \
  --output speech.raw
```

Billing is identical to the non-streaming path (per character). Other providers ignore `stream` and return the full audio in one response.