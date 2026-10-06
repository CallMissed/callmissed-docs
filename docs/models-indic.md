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
- **Context:** 32K tokens
- **Tool calling:** yes
- **Streaming:** yes
- **Best for:** Multi-turn dialogue, voice agents, assistants that talk
- **Thinking mode:** `reasoning_effort: "low" | "medium" | "high"`

Same family and same price as `sarvam-105b`, with a shorter 32K window — tuned for spoken dialogue rather than long-form work. It does not accept image input.

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
| `"none"` / `"minimal"` | Thinking off — fastest |
| `"low"` | Light reasoning |
| `"high"` | Deep reasoning — better quality, slower |
| `"max"` | Maximum reasoning (`"xhigh"` maps here) |

`"medium"` is not one of the models' levels, so it is dropped and the model
runs at its default. Thinking tokens count toward `max_tokens`; when you omit
it we send 4,096 so the answer is not cut off by the upstream default of
2,048. See the [reasoning_effort matrix](/docs/api-speed#3-reasoning-effort-by-model).

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

- **Voices:** 37 speakers
- **Languages:** 11
- **Audio codecs:** WAV, MP3, OPUS, FLAC, AAC, Mulaw, Alaw, PCM
- **Pace:** 0.5–2.0 (maps to `speed` parameter)
- **Sample rates:** 8000, 16000, 22050, 24000, 48000 Hz
- **Endpoint:** `POST /v1/audio/speech`

Default voice: `shubh`. See the [Voices](/docs/tts-voices) page for the full list.
