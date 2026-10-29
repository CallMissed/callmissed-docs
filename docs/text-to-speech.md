---
title: "Text to Speech"
description: "Convert text to natural-sounding speech — Indic, English and multilingual TTS models behind one OpenAI-compatible endpoint."
slug: "text-to-speech"
breadcrumb: "Speech"
---

# Text to Speech

Convert text to natural-sounding speech — Indic, English and multilingual TTS models behind one OpenAI-compatible endpoint.

## Overview

The Text to Speech API converts text into audio with any of **9 TTS models**. The default, `bulbul:v3`, has 37 voices across 11 Indian languages. `bulbul:v4-flash` is a lower-latency, lower-cost alternative with 222 persona voices across 9 Indian languages and Hinglish.

Two models are on the **free tier**: `bulbul:v3` and `bulbul:v4-flash`. The retired `aura-2-en`, `aura-2-es` and `melotts` IDs still work and stay free; they are answered by `deepgram-aura-2` (same voices) and `gpt-4o-mini-tts` — see [Retired and deprecated models](/docs/models#retired-and-deprecated-models).

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
| `input` | string | Yes | Text to synthesize, up to 4096 characters. Some models accept less: `bulbul:v3` 2500, `bulbul:v4-flash` 2500, `deepgram-aura-2` / `deepgram-aura-1` 2000 |
| `model` | string | No | Default `bulbul:v3`. One of `bulbul:v3`, `bulbul:v4-flash`, `sonic-3.6`, `gnani-timbre-v3.0`, `deepgram-aura-2`, `deepgram-aura-1`, `gpt-4o-mini-tts`, `mai-voice-2.1`, `mai-voice-2.1-flash` — see [Models](/docs/models) |
| `voice` | string | No | Voice ID — default `shubh` for `bulbul:v3`, `shubh_enhi_ads` for `bulbul:v4-flash` (v3 names such as `shubh` are not valid there), `skylar` for `sonic-3.6` (see [Voices](/docs/tts-voices)). An unrecognized voice falls back to the model's default, except on `gpt-4o-mini-tts`, which returns `400` and lists its 13 voices |
| `language` | string | No | Language code (e.g. `hi-IN`, `ta-IN`; `sonic-3.6` takes base codes like `en`, `hi`). `bulbul:v3` defaults to `en-IN`. `bulbul:v4-flash` covers `hi-IN`, `en-IN`, `bn-IN`, `gu-IN`, `kn-IN`, `mr-IN`, `pa-IN`, `ta-IN` and `te-IN`; `ml-IN` and `od-IN` are not available on it and return `400` (use `bulbul:v3`) |
| `speed` | number | No | Speech speed, default 1.0. Must be 0.25–4.0, then each model clamps to its own range: `bulbul:v3` and `bulbul:v4-flash` 0.5–2.0, `gpt-4o-mini-tts` 0.25–4.0, `deepgram-aura-2`/`-1` 0.7–1.5, `sonic-3.6` 0.6–1.5, `gnani-timbre-v3.0` 0.85–1.15, `mai-voice-2.1` / `mai-voice-2.1-flash` 0.5–2.0. Ignored by `aura-2-en`, `aura-2-es`, `melotts` |
| `speech_sample_rate` | integer | No | Output sample rate in Hz, an integer from 8000 to 48000. Leave it out to get each model's own default: `bulbul:v3` and `bulbul:v4-flash` 22050, `sonic-3.6` 44100, `gnani-timbre-v3.0` 48000, `deepgram-aura-2` / `deepgram-aura-1` 24000 (8000 for `mulaw` / `alaw`). Headerless `pcm` carries no rate, so it defaults to 8000 on every model. Send 8000 for phone audio. `bulbul:v3` accepts only 8000, 16000, 22050, 24000, 32000, 44100 or 48000 and `bulbul:v4-flash` only 8000, 16000, 22050 or 24000 (anything else is a `400`); Deepgram PCM output snaps to the nearest of 8000, 16000, 24000, 32000 or 48000, and its `mp3` / `opus` / `aac` output has a fixed rate. Ignored by `gpt-4o-mini-tts` |
| `response_format` | string | No | Default `mp3` — see [Audio Formats](#audio-formats) |
| `temperature` | number | No | Expressiveness. `bulbul:v3` and `bulbul:v4-flash` only — higher is more expressive, lower is more consistent. The model's range is 0.01–1.0; values up to 2.0 are accepted and applied as 1.0, above 2.0 is a 422. Ignored by other models. Defaults to 0.9 (warmer than the model's flat default) |
| `instructions` | string | No | Natural-language delivery direction — tone, emotion, accent, pacing. `gpt-4o-mini-tts` only. Max 2000 chars. Example: `"Speak slowly and warmly, like you're reassuring someone."` |
| `humanize` | boolean | No | Default `true`. Shapes your text for natural speech before synthesis — strips markdown and emoji, speaks URLs and emails as words, groups long digit runs into readable chunks. Set `false` to synthesize your text byte-for-byte |
| `stream` | boolean | No | `deepgram-aura-2` / `deepgram-aura-1` only — stream audio as it's generated (lower time-to-first-byte) |

## Audio Formats

`response_format` accepts `mp3`, `opus`, `aac`, `flac`, `wav` or `pcm`, but not every model can produce every format. The `Content-Type` response header always describes the audio actually returned, so read it rather than assuming the format you asked for.

`pcm` is headerless 16-bit PCM (`audio/pcm`; `sonic-3.6` and `gnani-timbre-v3.0` also accept the name `raw`), and headerless G.711 is labelled `audio/mulaw` / `audio/alaw`. Audio that arrives in a WAV container is labelled `audio/wav`, as buffered `mulaw` / `alaw` from `deepgram-aura-*` do; use `stream: true` for headerless G.711.

| Model | Formats honoured | Otherwise returns |
|-------|------------------|-------------------|
| `bulbul:v3`, `bulbul:v4-flash` | `mp3`, `aac`, `flac`, `wav`, `pcm`, when `response_format` is sent | WAV (including for `opus`, `mulaw`, `alaw`, and when `response_format` is left out) |
| `gpt-4o-mini-tts` | `mp3`, `opus`, `aac`, `flac`, `wav`, `pcm` | MP3 |
| `deepgram-aura-2`, `deepgram-aura-1` | `mp3`, `opus`, `aac`, `flac`, `wav`, `pcm`, plus `mulaw` / `alaw` | WAV |
| `sonic-3.6` | `mp3`, `wav`, `pcm` | WAV |
| `gnani-timbre-v3.0` | `mp3`, `wav`, `pcm`, `opus`, plus `mulaw` | WAV |
| `mai-voice-2.1`, `mai-voice-2.1-flash` | `mp3` (default, 24 kHz), `wav`, `pcm` (alias `raw`), `opus` (alias `ogg`), `webm`, `mulaw`, `alaw` | `flac` and `aac` return `400` |

## Errors

Errors use the OpenAI envelope: `{"error": {"message", "type", "code"}}`.

| Status | `code` | When |
|--------|--------|------|
| `400` | — | The model rejected a parameter (for example an unsupported `speech_sample_rate` on `bulbul:v3`) |
| `402` | `insufficient_credits` | Credit balance is exhausted |
| `403` | `permission_denied` | The API key does not have the `tts` permission, or the model is not available on your plan |
| `404` | `model_not_found` | `model` is not a known TTS model ID |
| `422` | `invalid_request` | Missing or empty `input`, `input` over the model's character limit (see `input` above), or a field with the wrong type or out of range (`speed`, `speech_sample_rate`, `temperature`, `instructions`) |
| `429` | `quota_exceeded` | Plan usage limit reached |
| `503` | `provider_error`, `upstream_timeout`, `upstream_unavailable` | The model failed to synthesize. The response includes a `request_id` |

## MAI Voice (preview)

`mai-voice-2.1` and `mai-voice-2.1-flash` are Microsoft AI's text-to-speech models. **Both are preview.** `mai-voice-2.1` ($0.2292 per 10K characters) is the most expressive and suits narration and long-form audio. `mai-voice-2.1-flash` ($0.1563 per 10K characters) is built for low latency and is the pick for live calls.

- **Voices and languages.** 97 voices across 23 languages: `cs`, `da`, `de`, `en`, `es`, `fi`, `fr`, `hi`, `hu`, `id`, `it`, `ko`, `nb`, `nl`, `pl`, `pt`, `ro`, `ru`, `sv`, `th`, `tr`, `vi`, `zh`. Voice handles are lowercase `<locale>-<name>` (for example `hi-in-kavya`) and case-insensitive. Each voice speaks only its own locale. The full list is in `voice_names` and `voice_details` on `GET /api/v1/models?category=tts`, and in [Voices](/docs/tts-voices#mai-voice-preview).
- **Default voice.** With no `voice`, the first voice for the request `language` is used (`hi` gives `hi-in-arjun`), otherwise `en-in-priya`.
- **Text.** Plain text only. SSML and tags are not supported; the text is read as written.
- **Formats.** `mp3` (default), `wav`, `pcm`, `opus`, `webm`, `mulaw` and `alaw`; `flac` and `aac` return `400`. `speech_sample_rate` snaps to the nearest supported rate: `mp3` 16000, 24000 or 48000; `wav` 8000, 16000, 24000 or 48000; `pcm` 8000, 16000, 22050, 24000, 44100 or 48000; `opus` 24000 or 48000; `webm` 24000; `mulaw` / `alaw` 8000.
- **Speed.** `speed` is 0.5 to 2.0 and is clamped. Voice agents use `tts_speed`, 0.5 to 2.0.
- **Voice agents.** Set `tts_model` to either id in a voice agent or voice session.
- **Limits.** Up to 4096 characters per request. Not processed in India, so an agent with `hosting: "india"` cannot use them. They are paid-plan only, are never a default, and cannot be used in `tts_fallbacks`.
- **Preview.** Microsoft ships these models as a public preview with no SLA, and may change or retire them at short notice. If one is retired, its ID keeps working and is answered by a named successor, as described in [Retired and deprecated models](/docs/models#retired-and-deprecated-models), so an agent that uses it keeps working but may sound different. Use a generally available model for production-critical agents.

```bash
curl -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "mai-voice-2.1-flash", "voice": "hi-in-kavya", "input": "Namaste, aapka order aaj shaam tak pahunch jayega."}' \
  --output speech.mp3
```

## Making speech sound human

Naturalness comes from three places, in order of impact:

**1. The text you send.** Every engine sounds more human when the input reads like speech rather than like a screen. `humanize` (on by default) handles the mechanical part — markdown, emoji, `https://callmissed.com` → "callmissed dot com", `2039123456` → `203.912.3456`. Beyond that, write short sentences and use contractions; if an LLM generates your text, tell it that its output will be spoken aloud.

**2. Expressiveness parameters**, where the model supports them:

```bash
# bulbul:v3 — temperature is its expressiveness control
curl -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "bulbul:v3", "input": "Bilkul, main abhi check karta hoon.", "voice": "shubh", "temperature": 1.0}' \
  --output speech.mp3

# gpt-4o-mini-tts — direct the performance in plain language
curl -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o-mini-tts", "input": "Your order shipped this morning.", "voice": "nova", "instructions": "Cheerful and upbeat, like sharing good news with a friend."}' \
  --output speech.mp3
```

**3. Pauses in the text.** `deepgram-aura-2` and `deepgram-aura-1` read pause cues written into your text: `...` gives a longer natural pause, a comma or period gives a short one, and `um`/`uh` render as natural hesitation.

```json
{"model": "deepgram-aura-2", "input": "Let me pull that up... okay, found it."}
```

`bulbul:v3` does not support pause markup or SSML — it would speak the dots aloud. Use sentence length and real punctuation for rhythm on that model. `gnani-timbre-v3.0` takes its own audio tags instead — see below.

## Gnani Timbre v3 audio tags

`gnani-timbre-v3.0` (Gnani Timbre v3) streams speech in 12 voices, and every voice speaks all 10 Indian languages (`hi-IN`, `en-IN`, `ta-IN`, `te-IN`, `kn-IN`, `ml-IN`, `mr-IN`, `pa-IN`, `bn-IN`, `gu-IN`) plus `auto` and Hinglish `hi-en`. Write expressive tags straight into `input`:

| Tag | Effect |
|-----|--------|
| `<laugh>` | A full laugh |
| `<giggle>` | A light giggle |
| `<sigh>` | A sigh |
| `<exhales>` | An audible exhale |
| `<sniffles>` | A sniffle |
| `<yawns>` | A yawn |
| `<coughs>` | A cough |
| `<clears throat>` | Clears the throat |
| `<whispers>` | Hushed delivery |
| `<shouting>` | Raised, shouted delivery |
| `<excited>` | Energetic delivery |
| `<slow>` | Drawn-out delivery |
| `<pauses>` | A short silence (not spoken) |

Rules:

- Use the exact lowercase names above, in angle brackets. There is no closing tag.
- A tag applies from its position to the end of the input, or until the next tag.
- Unknown tags are dropped.
- SSML is not supported: a `<speak>` wrapper is read aloud. Square-bracket tags such as `[laugh]` are not supported either.

```bash
curl -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "gnani-timbre-v3.0", "voice": "Shivani", "input": "Let me check that for you. <pauses> Found it <laugh> your order ships tomorrow."}' \
  --output speech.mp3
```

Voice agents that speak with Gnani Timbre v3 add a few of these tags on their own. See [Voice agents](/docs/voice-agent).

:::note
Seven more TTS models (`deepgram-flux-tts`, `rime-coda`, `inworld-tts-2`, `inworld-tts-2-flash`, `fish-audio-drama-3`, `gradium-tts-beta`, `microsoft-neural-tts`) speak only inside voice agents and voice sessions. They are not valid `model` values here and return `400`. Their voices, languages and prices are under [Voice session voices](/docs/tts-voices#voice-session-voices).
:::

## Choosing an expressive voice

| Want | Use |
|------|-----|
| Most natural conversational speech | `sonic-3.6` — 44 languages, native-quality Hindi, sub-90ms first audio |
| Direct the emotion in words | `gpt-4o-mini-tts` with `instructions` |
| Indian languages, warm delivery | `bulbul:v3` with `temperature` 0.9–1.0 |
| Pauses and hesitation in text | `deepgram-aura-2` (90 voices, many tagged expressive/cheerful) |
| Lowest cost | `deepgram-aura-1` or `bulbul:v4-flash` |
| Narration and long-form audio (preview) | `mai-voice-2.1` |
| Live calls on a Microsoft AI voice (preview) | `mai-voice-2.1-flash` |

## Streaming (Deepgram Aura)

For `deepgram-aura-2` and `deepgram-aura-1`, set `"stream": true` to receive audio frames as they're synthesized, as a chunked HTTP response. Ideal for real-time playback where you want the first audio bytes as fast as possible.

Streaming supports **raw encodings only** — `response_format` must be `linear16` (or `pcm`/`wav`), `mulaw`, or `alaw`. Compressed formats (`mp3`, `opus`, `aac`, `flac`) cannot be streamed; if you request one with `stream:true`, the full audio is returned in one buffered response instead.

Streamed audio is headerless and its `Content-Type` carries the sample rate, for example `audio/l16;rate=24000` for `linear16` / `pcm` / `wav` (Deepgram's label; the samples are 16-bit **little-endian**, not the network byte order RFC 2586 assumes for L16), or `audio/mulaw;rate=8000` / `audio/alaw;rate=8000`.

```bash [cURL]
curl -N -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "deepgram-aura-2", "voice": "thalia", "input": "Streaming hello.", "response_format": "linear16", "stream": true}' \
  --output speech.raw
```

Billing is identical to the non-streaming path (per character). Other providers ignore `stream` and return the full audio in one response.
