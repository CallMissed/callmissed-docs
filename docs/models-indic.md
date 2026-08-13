---
title: "Indic Models"
description: "Indic STT, TTS, and LLM models — optimized for Indian languages."
slug: "models-indic"
breadcrumb: "LLM & AI"
---

# Indic Models

Indic STT, TTS, and LLM models — optimized for Indian languages.

## LLM

### sarvam-105b

- **Architecture:** 105B MoE, MLA architecture
- **Context:** 128K tokens
- **Training:** Pre-trained on 12T tokens
- **Best for:** Complex reasoning, agentic tasks, long documents
- **Thinking mode:** `reasoning_effort: "low" | "medium" | "high"`

### sarvam-105b-conversations

- **Architecture:** 105B MoE, tuned for conversation and voice
- **Context:** 128K tokens
- **Tool calling:** yes
- **Streaming:** yes
- **Best for:** Multi-turn dialogue, voice agents, assistants that talk
- **Thinking mode:** `reasoning_effort: "low" | "medium" | "high"`

Same family, same price and same 128K window as `sarvam-105b` — tuned for spoken dialogue rather than long-form work. It does not accept image input.

### Thinking Mode

The `sarvam-105b` models support hybrid thinking mode:

```python
response = client.chat.completions.create(
    model="sarvam-105b",
    messages=[{"role": "user", "content": "Solve this complex problem step by step"}],
    extra_body={"reasoning_effort": "high"}
)
```

| Value | Description |
|-------|-------------|
| `"low"` | Minimal reasoning — fastest, cheapest |
| `"medium"` | Balanced reasoning |
| `"high"` | Deep reasoning — best quality, slower |

The `sarvam-*` models reject `"none"` and `"minimal"`; the API maps both
of those values down to `"low"` so an OpenAI-style client that sends
`reasoning_effort: "none"` for thinking-off still works. Full
thinking-disable is available on the direct-routed `kimi-k2.5` / `kimi-k2.6` /
`kimi-k2.7-code` / `gemma-4-26b-a4b-it` models — see the [reasoning_effort matrix](/docs/api-speed#3-reasoning_effort-matrix-per-model--verified-empirically).

## Speech to Text

### saaras:v3

- **Languages:** 23 (22 Indic + English)
- **Output modes:** transcribe, translate, verbatim, translit, codemix
- **Auto language detection:** yes
- **Telephony support:** 8kHz audio
- **Endpoint:** `POST /v1/audio/transcriptions`

Supported languages include: Hindi, Bengali, Gujarati, Kannada, Malayalam, Marathi, Odia, Punjabi, Tamil, Telugu, Urdu, Assamese, Bodo, Dogri, Kashmiri, Konkani, Maithili, Manipuri, Nepali, Sanskrit, Santali, Sindhi, and English.

### saaras:v4

- **Languages:** 24
- **Output modes:** transcribe, translate, verbatim, translit, codemix
- **Auto language detection:** yes
- **Endpoint:** `POST /v1/audio/transcriptions`

Five output modes on one model — standard transcription, English translation, verbatim (fillers kept), Latin-script transliteration, and code-mixed output. Select one with the `mode` form field; `transcribe` is the default.

## Text to Speech

### bulbul:v3

- **Voices:** 39 speakers
- **Languages:** 11
- **Audio codecs:** WAV, MP3, OPUS, FLAC, AAC, Mulaw, Alaw, PCM
- **Pace:** 0.5–2.0 (maps to `speed` parameter)
- **Sample rates:** 8000, 16000, 22050, 24000, 48000 Hz
- **Endpoint:** `POST /v1/audio/speech`

Default voice: `shubh`. See the [Voices](/docs/tts-voices) page for the full list.
