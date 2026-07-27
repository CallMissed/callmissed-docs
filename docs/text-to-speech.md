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
| `speed` | number | Speech speed (default 1.0). `bulbul:v3` 0.5–2.0, `gpt-4o-mini-tts` 0.25–4.0, `deepgram-aura-2`/`-1` 0.7–1.5. Not supported by `aura-2-en`, `aura-2-es`, `melotts`, `gnani-timbre-v2.0` |
| `speech_sample_rate` | integer | 8000, 16000, 22050, 24000, or 48000 Hz |
| `response_format` | string | Output format — see below |
| `temperature` | number | Expressiveness, 0.01–2.0. `bulbul:v3` only — higher is more expressive, lower is more consistent. Defaults to 0.9 (warmer than the model's flat default) |
| `instructions` | string | Natural-language delivery direction — tone, emotion, accent, pacing. `gpt-4o-mini-tts` only. Max 2000 chars. Example: `"Speak slowly and warmly, like you're reassuring someone."` |
| `humanize` | boolean | Default `true`. Shapes your text for natural speech before synthesis — strips markdown and emoji, speaks URLs and emails as words, groups long digit runs into readable chunks. Set `false` to synthesize your text byte-for-byte |
| `stream` | boolean | Deepgram Aura-2 only — stream audio as it's generated (lower time-to-first-byte) |

## Audio Formats

Supported values for `response_format`: `mp3`, `opus`, `aac`, `flac`, `wav`, `pcm`

## Making speech sound human

Naturalness comes from three places, in order of impact:

**1. The text you send.** Every engine sounds more human when the input reads like speech rather than like a screen. `humanize` (on by default) handles the mechanical part — markdown, emoji, `https://callmissed.com` → "callmissed dot com", `2039123456` → `203.912.3456`. Beyond that, write short sentences and use contractions; if an LLM generates your text, tell it that its output will be spoken aloud.

**2. Expressiveness parameters**, where the model supports them:

```bash
# bulbul:v3 — temperature is its expressiveness control
curl -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "bulbul:v3", "input": "Bilkul, main abhi check karta hoon.", "voice": "shubh", "temperature": 1.1}' \
  --output speech.mp3

# gpt-4o-mini-tts — direct the performance in plain language
curl -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o-mini-tts", "input": "Your order shipped this morning.", "voice": "nova", "instructions": "Cheerful and upbeat, like sharing good news with a friend."}' \
  --output speech.mp3
```

**3. Pauses in the text.** `deepgram-aura-2`, `deepgram-aura-1`, `aura-2-en` and `aura-2-es` read pause cues written into your text: `...` gives a longer natural pause, a comma or period gives a short one, and `um`/`uh` render as natural hesitation.

```json
{"model": "deepgram-aura-2", "input": "Let me pull that up... okay, found it."}
```

`bulbul:v3` and `gnani-timbre-v2.0` do not support pause markup or SSML — they would speak the dots aloud. Use sentence length and real punctuation for rhythm on those models.

## Choosing an expressive voice

| Want | Use |
|------|-----|
| Direct the emotion in words | `gpt-4o-mini-tts` with `instructions` |
| Indian languages, warm delivery | `bulbul:v3` with `temperature` 0.9–1.2 |
| Pauses and hesitation in text | `deepgram-aura-2` (91 voices, many tagged expressive/cheerful) |
| Lowest cost | `melotts` — no expressive controls; rely on `humanize` |

## Streaming (Deepgram Aura-2)

For `deepgram-aura-2` and `deepgram-aura-1`, set `"stream": true` to receive audio frames as they're synthesized over Deepgram's low-latency WebSocket, relayed to you as a chunked HTTP response. Ideal for real-time playback where you want the first audio bytes as fast as possible.

Streaming supports **raw encodings only** — `response_format` must be `linear16` (or `pcm`/`wav`), `mulaw`, or `alaw`. Compressed formats (`mp3`, `opus`, `aac`, `flac`) are not WebSocket-streamable; if you request one with `stream:true`, the full audio is returned in one buffered response instead.

```bash [cURL]
curl -N -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "deepgram-aura-2", "voice": "thalia", "input": "Streaming hello.", "response_format": "linear16", "stream": true}' \
  --output speech.raw
```

Billing is identical to the non-streaming path (per character). Other providers ignore `stream` and return the full audio in one response.