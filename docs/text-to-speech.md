---
title: "Text to Speech"
description: "Convert text to natural-sounding speech — Indic, English and multilingual TTS models behind one OpenAI-compatible endpoint."
slug: "text-to-speech"
breadcrumb: "Speech"
---

# Text to Speech

Convert text to natural-sounding speech — Indic, English and multilingual TTS models behind one OpenAI-compatible endpoint.

## Overview

The Text to Speech API converts text into audio with any of **9 TTS models**. The default, `bulbul:v3`, has 37 voices across 11 Indian languages.

Four models are on the **free tier**: `bulbul:v3`, `aura-2-en`, `aura-2-es` and `melotts`.

**Endpoint:** `POST /v1/audio/speech`

:::flow
icon:app | Your app | Send text + a `voice` and `language` to `POST /v1/audio/speech`
icon:tts | Your chosen model | Synthesize speech in the chosen voice — `bulbul:v3` if `model` is omitted
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

Send a JSON body.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `input` | string | Yes | Text to synthesize, up to 4096 characters. Some models accept less: `bulbul:v3` 2500, `deepgram-aura-2` / `deepgram-aura-1` 2000 |
| `model` | string | No | Default `bulbul:v3`. One of `bulbul:v3`, `sonic-3.6`, `gnani-timbre-v2.0`, `deepgram-aura-2`, `deepgram-aura-1`, `aura-2-en`, `aura-2-es`, `gpt-4o-mini-tts`, `melotts` — see [Models](/docs/models) |
| `voice` | string | No | Voice ID — default `shubh` for `bulbul:v3`, `skylar` for `sonic-3.6` (see [Voices](/docs/tts-voices)). An unrecognized voice falls back to the model's default |
| `language` | string | No | Language code (e.g. `hi-IN`, `ta-IN`; `sonic-3.6` takes base codes like `en`, `hi`). `bulbul:v3` defaults to `en-IN` |
| `speed` | number | No | Speech speed, default 1.0. Must be 0.25–4.0, then each model clamps to its own range: `bulbul:v3` 0.5–2.0, `gpt-4o-mini-tts` 0.25–4.0, `deepgram-aura-2`/`-1` 0.7–1.5, `sonic-3.6` 0.6–1.5, `gnani-timbre-v2.0` 0.85–1.15. Ignored by `aura-2-en`, `aura-2-es`, `melotts` |
| `speech_sample_rate` | integer | No | Default 8000. An integer from 8000 to 48000. `bulbul:v3` accepts only 8000, 16000, 22050, 24000, 32000, 44100 or 48000 (anything else is a `400`); Deepgram PCM output snaps to the nearest of 8000, 16000, 24000, 32000 or 48000 |
| `response_format` | string | No | Default `mp3` — see [Audio Formats](#audio-formats) |
| `temperature` | number | No | Expressiveness, 0.01–2.0. `bulbul:v3` only — higher is more expressive, lower is more consistent. Defaults to 0.9 (warmer than the model's flat default) |
| `instructions` | string | No | Natural-language delivery direction — tone, emotion, accent, pacing. `gpt-4o-mini-tts` only. Max 2000 chars. Example: `"Speak slowly and warmly, like you're reassuring someone."` |
| `humanize` | boolean | No | Default `true`. Shapes your text for natural speech before synthesis — strips markdown and emoji, speaks URLs and emails as words, groups long digit runs into readable chunks. Set `false` to synthesize your text byte-for-byte |
| `stream` | boolean | No | `deepgram-aura-2` / `deepgram-aura-1` only — stream audio as it's generated (lower time-to-first-byte) |

## Audio Formats

`response_format` accepts `mp3`, `opus`, `aac`, `flac`, `wav` or `pcm`, but not every model can produce every format. The `Content-Type` response header always describes the audio actually returned, so read it rather than assuming the format you asked for.

| Model | Formats honoured | Otherwise returns |
|-------|------------------|-------------------|
| `bulbul:v3` | — | Always WAV |
| `aura-2-en`, `aura-2-es`, `melotts` | — | Always MP3 |
| `gpt-4o-mini-tts` | `mp3`, `opus`, `aac`, `flac`, `wav`, `pcm` | MP3 |
| `deepgram-aura-2`, `deepgram-aura-1` | `mp3`, `opus`, `aac`, `flac`, `wav`, `pcm`, plus `mulaw` / `alaw` | WAV |
| `sonic-3.6` | `mp3`, `wav`, `pcm` | WAV |
| `gnani-timbre-v2.0` | `mp3`, `wav`, `pcm`, `opus`, plus `mulaw` | WAV |

## Errors

Errors use the OpenAI envelope: `{"error": {"message", "type", "code"}}`.

| Status | `code` | When |
|--------|--------|------|
| `400` | — | The model rejected a parameter (for example an unsupported `speech_sample_rate` on `bulbul:v3`) |
| `402` | `insufficient_credits` | Credit balance is exhausted |
| `403` | `permission_denied` | The API key does not have the `tts` permission, or the model is not available on your plan |
| `404` | `model_not_found` | `model` is not a known TTS model ID |
| `422` | `invalid_request` | Missing or empty `input`, `input` over 4096 characters, or a field with the wrong type or out of range (`speed`, `speech_sample_rate`, `temperature`, `instructions`) |
| `429` | `quota_exceeded` | Plan usage limit reached |
| `502` | `provider_error`, `upstream_timeout`, `upstream_unavailable` | The model failed to synthesize. The response includes a `request_id` |

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
| Most natural conversational speech | `sonic-3.6` — 44 languages, native-quality Hindi, sub-90ms first audio |
| Direct the emotion in words | `gpt-4o-mini-tts` with `instructions` |
| Indian languages, warm delivery | `bulbul:v3` with `temperature` 0.9–1.2 |
| Pauses and hesitation in text | `deepgram-aura-2` (90 voices, many tagged expressive/cheerful) |
| Lowest cost | `melotts` — no expressive controls; rely on `humanize` |

## Streaming (Deepgram Aura)

For `deepgram-aura-2` and `deepgram-aura-1`, set `"stream": true` to receive audio frames as they're synthesized, as a chunked HTTP response. Ideal for real-time playback where you want the first audio bytes as fast as possible.

Streaming supports **raw encodings only** — `response_format` must be `linear16` (or `pcm`/`wav`), `mulaw`, or `alaw`. Compressed formats (`mp3`, `opus`, `aac`, `flac`) cannot be streamed; if you request one with `stream:true`, the full audio is returned in one buffered response instead.

```bash [cURL]
curl -N -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "deepgram-aura-2", "voice": "thalia", "input": "Streaming hello.", "response_format": "linear16", "stream": true}' \
  --output speech.raw
```

Billing is identical to the non-streaming path (per character). Other providers ignore `stream` and return the full audio in one response.
