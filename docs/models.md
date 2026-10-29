---
title: "Models"
description: "Every model CallMissed serves — Indic STT/TTS/LLM, fast direct-routed LLMs, first-party flagships, realtime voice and image — through one OpenAI-compatible API, plus 300+ more we deploy on demand."
slug: "models"
breadcrumb: "LLM & AI"
---

# Models

Every model CallMissed serves — Indic STT/TTS/LLM, fast direct-routed LLMs, first-party flagships, realtime voice and image — through one OpenAI-compatible API, plus 300+ more we deploy on demand.

:::cards
/docs/model-access | Model Access by Plan | key | Free, Starter, Pro, and Enterprise model tiers
/docs/models-indic | Indic Models | mic | Indic STT, TTS, and LLM models
/docs/models-kimi-fast | Fast LLMs | zap | High-throughput Kimi tier for voice-agent latency
/docs/api-speed | API Speed | gauge | Latency benchmarks and reasoning-effort matrix
:::

## Overview

134 models, one OpenAI-compatible API. Same auth, same request shape — change
the `model` field and nothing else.

| Group | What it is |
|-------|------------|
| **Fast LLMs** | Gemma 4 31B, the default for voice agents, and Kimi K2.5 at up to ~414 tok/s. |
| **Indic models** | STT, TTS and LLM built for 22 Indian languages. |
| **Direct-routed LLMs** | Sub-2s open-weights models: Kimi K2.5/K2.6/K2.7 Code and GPT-OSS. |
| **First-party** | `gpt-6-*`, `gpt-6.1-sol`, `gpt-5.6-*`, `grok-4.3`, `DeepSeek-V4-*`, realtime voice, plus first-party STT/TTS. Older GPT models are [deprecated](#retired-and-deprecated-models). |
| **On demand** | [300+ more we deploy on request](#models-on-demand). |

## Models API

List all available models programmatically. **No authentication required.**

```bash
# List all models
curl https://api.callmissed.com/api/v1/models

# Filter by category: llm, stt, tts, image, embedding
curl https://api.callmissed.com/api/v1/models?category=llm

# Filter free-plan models only
curl https://api.callmissed.com/api/v1/models?free=true

# Get a specific model
curl https://api.callmissed.com/api/v1/models/sarvam-105b

# Which models each plan tier can call
curl https://api.callmissed.com/api/v1/models/access
```

Each entry includes `id`, `name`, `description`, `category`, `owned_by`, `context_window`, `max_output_tokens` (only where the model documents an output limit), `pricing` (US$), `free`, `supports_streaming`, `supports_tools`, `supports_reasoning`, `supports_vision`, `zero_data_retention`, `processing` (`"india"`, `"india_preferred"` or `"outside_india"`; see [Data residency](/docs/data-residency)), `hosted_in_india` (kept for compatibility; `true` only when `processing` is `"india"`), `voice_capable` (valid for `POST /v1/voice/sessions`), `voice_agent_only`, and `speed_tier`. A model that is temporarily unavailable also carries `status: "maintenance"` and a `status_message` naming the alternative. Embedding models add `dimensions`; TTS models add their voice list. The response is `{object: "list", total, data}`.

The OpenAI-compatible listing at `GET /v1/models` (requires `Authorization: Bearer cm_*`) returns the same capability fields plus `context_length` (an alias of `context_window` for OpenAI-style clients), but a **shorter list**: it hides models that are only valid for voice sessions (`gpt-realtime*`, `gpt-live-1`, `deepgram-voice-*`). Use `GET /api/v1/models` for the full catalog. The Anthropic-shape listing at `GET /anthropic/v1/models` returns the same set inside Anthropic's `{data, has_more, first_id, last_id}` envelope.

## Free Plan Models

The free tier includes **11 models** across four categories. Use `GET /api/v1/models?free=true` to list them, or see the [Model Access by Plan](/docs/model-access) page for the full breakdown.

### LLM (6 models)
| Model ID | Description |
|----------|-------------|
| `sarvam-105b` | 105B MoE — complex reasoning, Indic languages |
| `sarvam-105b-conversations` | 105B MoE tuned for conversation and voice — 32K context, tool calling |
| `kimi-k2.5` | Moonshot K2.5 — 262K context, reasoning, vision |
| `kimi-k2.6` | Moonshot K2.6 — improved reasoning + coding, 262K context |
| `kimi-k2.7-code` | Moonshot K2.7 Code — frontier 1T-param agentic coding, 262K context, vision + tools |
| `gpt-oss-120b` | GPT-OSS 120B — open-weights large model |

### STT (1 model)
| Model ID | Description |
|----------|-------------|
| `saaras:v4` | 23 langs (22 Indic + English) — five output modes: transcribe, translate, verbatim, transliterate, code-mix |

### TTS (2 models)
| Model ID | Description |
|----------|-------------|
| `bulbul:v3` | 37 voices, 11 Indian languages |
| `bulbul:v4-flash` | 222 persona voices, 9 Indian languages + Hinglish, low latency |

### Embedding (2 models)
| Model ID | Description |
|----------|-------------|
| `text-embedding-3-small` | 1536 dimensions, 8,192-token inputs — best price/performance |
| `text-embedding-3-large` | 3072 dimensions, 8,192-token inputs — highest accuracy |

A free-plan key calling a paid model gets `403 model_not_available` — it is not billed, it is refused. Upgrade to Starter or above first.

The Claude models (`claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`) are the exception: they need the Pro plan or higher. A free or Starter key gets `403 model_not_available` with the message `Model '<id>' requires the Pro plan or higher. Upgrade at https://console.callmissed.com/org/plans`.

All other models — including `kimi-k2.5-fast`, `glm-5.3`, `deepseek-v4.1-flash`, `gemma-4-31b`, the `gemini-*` chat models, first-party IDs (`gpt-4o`, `gpt-4.1`, `gpt-5-mini`, `gpt-5.5`, `gpt-5.6-*`, `gpt-6-*`, `gpt-6.1-sol`, `grok-4.3`, `DeepSeek-V4-*`, `gpt-realtime*`, `gpt-live-1`, first-party STT/TTS), the other speech models (`gnani-*`, `ink-*`, `sonic-3.6`, `assemblyai-*`, `inworld-*`, `rime-coda`, `fish-audio-*`, `gradium-*`, `deepgram-flux-tts`, `microsoft-*`), the Deepgram direct line (`deepgram-nova-3`, `deepgram-flux-general-en/multi`, `deepgram-nova-2*`, `deepgram-enhanced*`, `deepgram-base*`, `deepgram-whisper-*`, `deepgram-aura-2`, `deepgram-aura-1`, Deepgram Voice Agent `deepgram-voice-*` ids, the `deepgram-summarize/topics/sentiment/intents` Audio Intelligence features, and the `deepgram-text-summarize/topics/sentiment/intents` Text Intelligence features), and paid image models (`flux-2-pro`, `flux-1.1-pro`, `gpt-image-2.5-*`, `gpt-image-2`, `gpt-image-1.5`, `nano-banana-*`, `gemini-3.1-flash-lite-image`) — require Starter, Pro, or Enterprise.

### Pricing

All models are pay-per-use. Pricing is in US$ at US$1 = ₹96; you pay in credits, where 1 credit = ₹1 ≈ US$0.0104.

| Model | Input / 1M tokens | Output / 1M tokens |
|-------|-------------------|-------------------|
| `kimi-k2.5-fast` | $0.8438 | $4.219 |
| `sarvam-105b` | $0.1563 (₹15) | $0.625 (₹60) |
| `sarvam-105b-conversations` | $0.1563 (₹15) | $0.625 (₹60) |
| `gpt-5.6-sol` | $4.167 | $20.83 |
| `gpt-5.6-terra` | $2.083 | $12.5 |
| `gpt-5.6-luna` | $0.2083 | $1.25 |
| `gpt-6-sol` | $2.083 | $10.42 |
| `gpt-6-luna` | $0.1042 | $0.5208 |
| `gpt-6.1-sol` | $2.083 | $10.42 |
| `claude-opus-5-5` *(Pro plan and up)* | $4.167 | $20.83 |
| `claude-sonnet-5-5` *(Pro plan and up)* | $2.083 | $10.42 |
| `claude-haiku-5-5` *(Pro plan and up)* | $0.1042 ($0.5208 for prompts over 100K tokens) | $0.5208 ($2.604 for prompts over 100K tokens) |
| `gpt-realtime` | $4.167 | $16.67 |
| `gpt-realtime-mini` | $0.625 | $2.5 |
| `gpt-realtime-2` | $4.167 | $25 |
| `gpt-realtime-1.5` | $4.167 | $16.67 |
| `gpt-realtime-2.1` | $4.167 | $25 |
| `gpt-realtime-2.1-mini` | $0.625 | $2.5 |
| `gpt-live-1` | per-minute only | $0.05208/min *(voice-agent only)* |
| `deepgram-voice-*` | per-minute Voice Agent tier | Standard $0.07813/min, Advanced $0.1698/min *(voice-agent only)* |

| STT Model | Price |
|-----------|-------|
| `saaras:v4` | $0.3125 / hour (₹30/hr) |
| `gnani-prisma-v2.5` | $0.2813 / hour |
| `ink-whisper` | $0.1875 / hour |
| `ink-2` | $0.5625 / hour *(voice sessions only)* |
| `assemblyai-universal-3.6-pro` | $0.4688 / hour *(voice sessions only)* |
| `inworld-stt-1` | $0.1563 / hour *(voice sessions only)* |
| `microsoft-speech-stt` | $1.0417 / hour *(voice sessions only)* |
| `deepgram-nova-3` | $0.3021 / hour |
| `deepgram-nova-3-medical` | $0.3021 / hour |
| `deepgram-flux-general-en` | $0.4063 / hour |
| `deepgram-flux-general-multi` | $0.4896 / hour |
| `deepgram-nova-2` (+ domain variants) | $0.3646 / hour |
| `deepgram-nova` / `deepgram-whisper-*` | $0.3646 / hour |
| `deepgram-enhanced` (+ variants) | $1.031 / hour |
| `deepgram-base` (+ variants) | $0.9063 / hour |

| TTS Model | Price |
|-----------|-------|
| `bulbul:v3` | $0.3125 / 10K chars (₹30/10K) |
| `bulbul:v4-flash` | $0.2083 / 10K chars (₹20/10K) |
| `gnani-timbre-v3.0` | $0.2813 / 10K chars |
| `deepgram-aura-2` | $0.3125 / 10K chars |
| `deepgram-aura-1` | $0.1563 / 10K chars |
| `sonic-3.6` | $0.5208 / 10K chars |
| `deepgram-flux-tts` | $0.4688 / 10K chars *(voice sessions only)* |
| `rime-coda` | $0.5208 / 10K chars *(voice sessions only)* |
| `inworld-tts-2` | $0.2604 / 10K chars *(voice sessions only)* |
| `inworld-tts-2-flash` | $0.1563 / 10K chars *(voice sessions only)* |
| `fish-audio-drama-3` | $0.5208 / 10K chars *(voice sessions only)* |
| `gradium-tts-beta` | $0.50 / 10K chars *(voice sessions only)* |
| `microsoft-neural-tts` | $0.1563 / 10K chars *(voice sessions only)* |

| Intelligence feature (not a model ID) | Price |
|-------------------------------|-------|
| `deepgram-summarize` / `-topics` / `-sentiment` / `-intents` (audio) | $0.0003125 / 1K input + $0.000625 / 1K output tokens |
| `deepgram-text-summarize` / `-text-topics` / `-text-sentiment` / `-text-intents` | $0.0003125 / 1K input + $0.000625 / 1K output tokens |

These eight values go in the `features` field, not in `model`. They are not
catalog models — `GET /api/v1/models/deepgram-summarize` returns 404.

Full pricing for all models is available via the API: `GET /api/v1/models`

```python
import requests

# List all LLM models
models = requests.get("https://api.callmissed.com/api/v1/models?category=llm").json()
for m in models["data"]:
    print(f"{m['id']} — {m['name']} ({m['context_window']} tokens) {'FREE' if m['free'] else 'PAID'}")
```

## Fast LLMs

High-throughput Kimi K2.5 inference tier optimized for voice-agent latency.

| Model ID | Status | Context | Best For |
|----------|--------|---------|----------|
| `kimi-k2.5-fast` | **Under maintenance** — fall back to `kimi-k2.5` | 256K | Voice agents, fast inference, reasoning tasks |

While `kimi-k2.5-fast` is in maintenance (returns HTTP 503), use `kimi-k2.5`:

```python
response = client.chat.completions.create(
    model="kimi-k2.5",
    messages=[{"role": "user", "content": "Hello"}]
)
```

## Indic Models

### Speech to Text

| Model | Description | Languages |
|-------|-------------|-----------|
| `saaras:v4` | Five output modes on one model — transcribe, translate, verbatim, transliterate, code-mix | 23 languages (22 Indic + English) |
| `gnani-prisma-v2.5` | India-first telephony STT — code-switching, sub-4% WER on Indian English | 10 Indian languages |

For 99-language general-purpose transcription and translation, see `whisper`. For diarization + smart-format on calls, see `deepgram-nova-3`. Both live under the [audio model routes](#audio-models).

`ink-whisper` also covers Hindi, Urdu and Tamil as part of its 100-language set at $0.1875 / hr — cheaper than the Indic-specialist models, though without their code-mix output modes. `ink-2` covers Hindi among its five languages (`en`, `fr`, `hi`, `ja`, `es`) but no other Indic language. `microsoft-speech-stt` (voice sessions only, processed in India) hears 81 languages, including Hindi, Bengali, Tamil, Telugu, Marathi, Gujarati, Kannada, Malayalam, Punjabi, Odia, Assamese, Urdu, Bhojpuri and Indian English, but only one language per call. `assemblyai-universal-3.6-pro` (voice sessions only) hears Hindi, Marathi and Urdu, plus Indian English (`en-IN`), among its 32 languages.

### Text to Speech

| Model | Description | Voices |
|-------|-------------|--------|
| `bulbul:v3` | Natural TTS — 37 voices, 11 Indian languages | shubh (default) + 36 more |
| `bulbul:v4-flash` | Low-latency TTS — 9 Indian languages + Hinglish; no Malayalam or Odia (use `bulbul:v3`) | 222 persona voices, `shubh_enhi_ads` (default) |
| `gnani-timbre-v3.0` | Gnani Timbre v3 — India-first neural TTS, streaming; every voice speaks all 10 Indian languages plus Hinglish; supports expressive audio tags in the text | 12 voices (Shivani default) |
| `sonic-3.6` | Cartesia Sonic 3.6 — most natural conversational speech, 44 languages with native-quality Hindi | Searchable public library + 16 featured aliases (skylar default) |
| `rime-coda` | Rime Coda — voice sessions only; 8 languages including Hindi | 13 voices, two native Hindi (nadi default) |
| `inworld-tts-2` / `inworld-tts-2-flash` | Inworld TTS-2 — voice sessions only; 94 languages including Hindi, Marathi, Bengali, Tamil, Telugu, Gujarati, Kannada, Malayalam, Punjabi, Urdu and Odia | 14 voices (riya default) |
| `microsoft-neural-tts` | Microsoft Neural voices — voice sessions only; processed in India; 13 languages including Hindi, Bengali, Tamil, Telugu, Marathi, Gujarati, Kannada, Malayalam, Punjabi, Urdu, Odia and Assamese | 57 voices (en-in-neerja default) |

For low-latency English / Spanish voice agents, see `deepgram-aura-2`. For steerable multilingual delivery, see `gpt-4o-mini-tts`.

### Chat Completion (LLM)

| Model | Params | Context | Best For |
|-------|--------|---------|----------|
| `sarvam-105b` | 105B MoE | 128K tokens | Complex reasoning, agentic tasks, long documents |
| `sarvam-105b-conversations` | 105B MoE | 32K tokens | Conversation and voice agents, tool calling |
| `glm-5.3` | Z.ai GLM 5.3 | 1M tokens | Long-context reasoning, tool calling (text only) |
| `deepseek-v4.1-flash` | DeepSeek V4.1 Flash | 1M tokens | Long-document analysis, agentic workflows, tool calling (text only) |
| `gemma-4-31b` | Google Gemma 4 31B | 128K tokens | Fast answers with thinking off, image input, tool calling, voice agents |

`glm-5.3` and `deepseek-v4.1-flash` always reason. `reasoning_effort` takes `"low"`, `"high"` or `"max"` (the default when omitted); `"none"` and `"minimal"` map to `"low"`, `"medium"` to `"high"` and `"xhigh"` to `"max"`. Reasoning tokens bill as output. `gemma-4-31b` answers without thinking by default. `reasoning_effort: "low"`, `"medium"` or `"high"` switch thinking on; the level does not reliably change how much it thinks. `"none"` and `"minimal"` keep it off. Both are paid-plan models.

Both `sarvam-105b` models support hybrid thinking via `reasoning_effort: "low" | "high" | "max"`, and `"xhigh"` maps to `"max"`. `"none"` and `"minimal"` switch thinking fully off. `"medium"` is not one of their levels, so it is dropped and the model runs at its default. Other models with a full off switch: `kimi-k2.5`, `kimi-k2.6`, `gemma-4-31b`, `DeepSeek-V4-Pro`, `DeepSeek-V4-Flash`, or a GPT-5.5 / GPT-5.6 / GPT-6 model with `reasoning_effort: "none"`. See the [per-model matrix](/docs/api-speed#3-reasoning-effort-by-model).

## Audio Models

See the [Pricing](/docs/pricing) page for current rates.

### Speech to Text

| Model | Languages | Best for | Price |
|-------|-----------|----------|-------|
| `whisper` | 99 with auto-detect | Whisper batch + translate | $0.4167 / hour |
| `gpt-4o-transcribe` | Streaming | Higher-accuracy OpenAI transcription | $0.4167 / hour |
| `gpt-4o-mini-transcribe` | Streaming | Low-cost OpenAI transcription | $0.25 / hour |
| `gpt-4o-transcribe-diarize` | File transcription with speaker labels (not available for voice sessions) | Multi-speaker meetings / calls | $0.4167 / hour |

**Deepgram (direct)** — the full Deepgram speech-to-text line, billed per audio hour at the rates below:

| Model | Languages | Best for | Price |
|-------|-----------|----------|-------|
| `deepgram-flux-general-en` | English | Conversational voice agents — model-native turn detection, ultra-low latency | $0.4063 / hour |
| `deepgram-flux-general-multi` | 10 (multilingual) | Multilingual voice agents with code-switching | $0.4896 / hour |
| `deepgram-nova-3` | 45+ incl. `multi` | Flagship general-purpose ASR, keyterm prompting, PII redaction | $0.3021 / hour |
| `deepgram-nova-3-medical` | English | Clinical / medical terminology | $0.3021 / hour |
| `deepgram-nova-2` | 36 incl. `multi` | High-accuracy ASR + filler-word detection | $0.3646 / hour |
| `deepgram-nova-2-{meeting,phonecall,finance,conversationalai,voicemail,video,medical,drivethru,automotive,atc}` | English | Domain-tuned Nova-2 variants | $0.3646 / hour |
| `deepgram-nova` / `-phonecall` / `-medical` | en/es/hi | Legacy Nova-1 | $0.3646 / hour |
| `deepgram-enhanced` (+ meeting/phonecall/finance) | 13 | Legacy, keyword boosting | $1.031 / hour |
| `deepgram-base` (+ 6 variants) | 17 | Legacy, high-volume batch | $0.9063 / hour |
| `deepgram-whisper-{tiny,base,small,medium,large}` | 99 | Whisper in five sizes, batch transcription | $0.3646 / hour |

### Text to Speech

| Model | Languages | Voices | Price |
|-------|-----------|--------|-------|
| `deepgram-aura-2` | en/es/de/fr/nl/it/ja | 90+ (thalia default) | $0.3125 / 10K chars |
| `deepgram-aura-1` | English | 12 (asteria default) | $0.1563 / 10K chars |
| `gpt-4o-mini-tts` | Multilingual steerable | 6 OpenAI voices | $0.2083 / 10K chars |

Aura 2 returns linear16 PCM streamed at 24 kHz for low-latency playback. Output formats may vary as models are updated.

Deepgram Flux TTS is a voice-agent-first model and is **not** available on this `/v1/audio/speech` endpoint. It runs in two places: as `deepgram-flux-tts` on voice agents and voice sessions (see [Voice-session-only models](#voice-session-only-models)), and in the managed Voice Agent (see the Voice Sessions API), selectable with `tts_engine: "flux"`, where it is billed inside the per-minute voice rate.

### Voice-session-only models

These models run inside voice agents and [voice sessions](/docs/voice-sessions-api) only: set them as `stt_model` or `tts_model`. `POST /v1/audio/transcriptions` and `POST /v1/audio/speech` return `400` for them. Voices, languages and markup support are on the [Voices](/docs/tts-voices#voice-session-voices) and [Speech to Text](/docs/speech-to-text#voice-session-only-models) pages.

| Model | Type | Languages | Price |
|-------|------|-----------|-------|
| `assemblyai-universal-3.6-pro` | STT, built-in turn detection | 50, including Hindi, Marathi, Urdu and Indian English | $0.4688 / hour |
| `inworld-stt-1` | STT | English only | $0.1563 / hour |
| `microsoft-speech-stt` | STT, processed in India | 81 (148 locales), including 13 Indian languages, one per call | $1.0417 / hour |
| `deepgram-flux-tts` | TTS, 16 voices | English only | $0.4688 / 10K chars |
| `rime-coda` | TTS, 13 voices | 8, including Hindi | $0.5208 / 10K chars |
| `inworld-tts-2` | TTS, 14 voices | 94, including 11 Indian languages | $0.2604 / 10K chars |
| `inworld-tts-2-flash` | TTS, 14 voices, faster | 94, including 11 Indian languages | $0.1563 / 10K chars |
| `fish-audio-drama-3` | TTS, 7 voices | 13, no Indian languages | $0.5208 / 10K chars |
| `gradium-tts-beta` | TTS, 13 voices | 5, no Indian languages | $0.50 / 10K chars |
| `microsoft-neural-tts` | TTS, 57 voices, processed in India | 13, including 12 Indian languages | $0.1563 / 10K chars |

### Audio Intelligence (Deepgram)

Deepgram Audio Intelligence runs analysis over an uploaded audio file via `POST /v1/audio/intelligence` (English only, 150K input-token limit, 25 MB upload limit). Billed at $0.0003125/1K input + $0.000625/1K output tokens, plus the transcription the analysis runs on at the per-hour rate of the matching `deepgram-*` STT model (e.g. `model=nova-3` bills at the `deepgram-nova-3` rate, $0.3021 / hour).

| Feature | Model ID | Returns |
|---------|----------|---------|
| Summarization | `deepgram-summarize` | A concise `summary` of the audio |
| Topic Detection | `deepgram-topics` | Per-segment `topics` with confidence |
| Sentiment Analysis | `deepgram-sentiment` | Per-segment + average `sentiments` |
| Intent Recognition | `deepgram-intents` | Per-segment `intents` with confidence |

Request multiple features in one call with a comma-separated `features` form field (e.g. `features=deepgram-summarize,deepgram-sentiment`).

### Text Intelligence (Deepgram)

Deepgram Text Intelligence runs the same four analyses over **text** input (a string or a hosted text URL) via `POST /v1/text/intelligence` (English only, 150K input-token limit). Token-billed at $0.0003125/1K input + $0.000625/1K output. Requires the `llm` key permission.

| Feature | Model ID | Returns |
|---------|----------|---------|
| Summarization | `deepgram-text-summarize` | A concise `summary` of the text |
| Topic Detection | `deepgram-text-topics` | Per-segment `topics` with confidence |
| Sentiment Analysis | `deepgram-text-sentiment` | Per-segment + average `sentiments` |
| Intent Recognition | `deepgram-text-intents` | Per-segment `intents` with confidence |

Send a JSON body with `features` (array or comma-separated string) and exactly one of `text` or `url`:

```json
{
  "features": ["deepgram-text-summarize", "deepgram-text-sentiment"],
  "text": "Your text to analyze here."
}
```

## Direct-Routed LLMs

Low-latency models routed directly through CallMissed — sub-2s end-to-end on small prompts and free-tier eligible per the [reasoning_effort matrix](/docs/api-speed#3-reasoning-effort-by-model).

| Model ID | Creator | Context |
|----------|---------|---------|
| `kimi-k2.5` | Moonshot AI | 262K |
| `kimi-k2.6` | Moonshot AI | 262K |
| `kimi-k2.7-code` | Moonshot AI | 262K |
| `gpt-oss-120b` | OpenAI (open-weights) | 131K |

## Retired and Deprecated Models

**Retired on 2026-10-17.** These IDs still work. Each one is answered by the
model in the second column, and the response `model` field keeps the ID you
sent. They are no longer listed by `GET /api/v1/models`. Requests are billed
at the price shown.

| Retired ID | Answered by | Billed at |
|------------|-------------|-----------|
| `glm-4.7-flash` | `gpt-oss-120b` | Unchanged ($0.5208 in / $2.083 out per 1M) |
| `glm-5.2` | `kimi-k2.7-code` | Unchanged ($1.969 in / $6.188 out per 1M) |
| `nemotron-3-super` | `kimi-k2.6` | Unchanged ($1.563 in / $6.25 out per 1M) |
| `gemma-4-26b-a4b-it` | `kimi-k2.5` | `kimi-k2.5` price ($0.8438 in / $4.219 out per 1M) |
| `mistral-small-3.1` | `kimi-k2.5` | `kimi-k2.5` price ($0.8438 in / $4.219 out per 1M) |
| `flux-2-klein-9b`, `phoenix-1.0` | `flux-2-pro` | Unchanged ($0.1042 / image) |
| `lucid-origin` | `flux-2-pro` | Unchanged ($0.08333 / image) |
| `sdxl-lightning`, `dreamshaper-8-lcm` | `flux-2-pro` | `flux-2-pro` price ($0.1042 / image) |
| `aura-2-en`, `aura-2-es` | `deepgram-aura-2` (same voices) | Unchanged ($0.4167 / 10K chars) |
| `melotts` | `gpt-4o-mini-tts` | `gpt-4o-mini-tts` price ($0.2083 / 10K chars) |
| `nova-3` | `deepgram-nova-3` | Unchanged ($0.5208 / hour) |
| `whisper-large-v3-turbo` | `whisper` (also translates) | `whisper` price ($0.4167 / hour) |

The retired LLM, speech-to-text and text-to-speech IDs stay on the free plan.
The retired image IDs do not: the free plan has no image model, and a free key
gets `403 model_not_available`.

**Deprecated, sunset 2026-11-16.** These models are still listed and served as
themselves until the sunset date. The catalogue marks them with
`deprecated: true`, `sunset_date` and `successor`, and every API response to a
request that names one carries `Deprecation`, `Sunset` and
`Link: <…/changelog>; rel="deprecation"` headers. From 2026-11-16 a request
that names one is answered by, and billed as, its successor.

| Deprecated ID | Successor |
|---------------|-----------|
| `gpt-4o`, `gpt-4.1`, `gpt-5.5`, `gpt-5.6-sol`, `gpt-5.6-terra` | `gpt-6-sol` |
| `gpt-5-mini`, `gpt-5.6-luna` | `gpt-6-luna` |
| `gemini-3.5-flash`, `gemini-3.6-flash`, `gemini-3.7-flash` | `gemini-3.8-flash` |
| `gemini-3.1-flash-lite` | `gemini-3.5-flash-lite` |
| `gpt-realtime`, `gpt-realtime-1.5`, `gpt-realtime-2` | `gpt-realtime-2.1` |
| `gpt-realtime-mini` | `gpt-realtime-2.1-mini` |
| `gpt-image-1.5`, `gpt-image-2` | `gpt-image-2.5-flare` |
| `flux-1.1-pro` | `flux-2-pro` |
| `deepgram-nova`, `deepgram-nova-2`, `deepgram-enhanced`, `deepgram-base` | `deepgram-nova-3` |
| `deepgram-nova-medical`, `deepgram-nova-2-medical` | `deepgram-nova-3-medical` |
| `deepgram-nova-phonecall`, `deepgram-enhanced-phonecall`, `deepgram-base-phonecall` | `deepgram-nova-2-phonecall` |
| `deepgram-enhanced-meeting`, `deepgram-base-meeting` | `deepgram-nova-2-meeting` |
| `deepgram-enhanced-finance`, `deepgram-base-finance` | `deepgram-nova-2-finance` |
| `deepgram-base-conversationalai`, `deepgram-base-voicemail`, `deepgram-base-video` | the matching `deepgram-nova-2-*` model |

Where a successor costs more (for example `gpt-4.1` output, `gemini-3.1-flash-lite`), the request bills at the successor's price from the sunset date; switch earlier to test.

## Models on Demand

`GET /api/v1/models` lists everything in the catalogue today: **134** model IDs.
Models marked `status: "maintenance"` return `503` until they are back.

Beyond that we deploy **300+ further models on demand** on CallMissed
infrastructure. Send the model you need and your expected throughput to
`sales@callmissed.com`. Once deployed it appears in your `GET /api/v1/models`
response with a plain CallMissed ID and published per-token pricing, on the
same `/v1/chat/completions` endpoint as every other model. Same key, same
credit balance, no new SDK.

Enterprise accounts get dedicated capacity. Starter and Pro get shared capacity
where the model allows it.

## First-Party Models

Credit-covered first-party models. Use the bare model ID in API requests — e.g. `gpt-4o`.

| Model ID | Type | Notes |
|----------|------|-------|
| `gpt-4o` | LLM | Multimodal text + vision, 128K context — *deprecated, sunset 2026-11-16 → `gpt-6-sol`* |
| `gpt-4.1` | LLM | Long-context (300K) multimodal — *deprecated, sunset 2026-11-16 → `gpt-6-sol`* |
| `gpt-5-mini` | LLM | Fast reasoning, 400K context — *deprecated, sunset 2026-11-16 → `gpt-6-luna`* |
| `gpt-5.5` | LLM | GPT-5.5 reasoning flagship, 1.05M context, vision + tools — *deprecated, sunset 2026-11-16 → `gpt-6-sol`* |
| `gpt-5.6-sol` | LLM | GPT-5.6 flagship, 1.05M context, vision + tools — *deprecated, sunset 2026-11-16 → `gpt-6-sol`* |
| `gpt-5.6-terra` | LLM | GPT-5.6 balanced intelligence/cost, 1.05M context — *deprecated, sunset 2026-11-16 → `gpt-6-sol`* |
| `gpt-5.6-luna` | LLM | GPT-5.6 fast + affordable, 1.05M context — *deprecated, sunset 2026-11-16 → `gpt-6-luna`* |
| `gpt-6-sol` | LLM | GPT-6 frontier reasoning, 1.05M context, vision + tools |
| `gpt-6-luna` | LLM | GPT-6 efficient, high-volume, 1.05M context |
| `gpt-6.1-sol` | LLM | GPT-6.1 near-frontier for complex coding and professional work, 1.05M context, vision + tools |
| `claude-opus-5-5` | LLM | Anthropic Claude Opus 5.5 for long-running agentic coding and knowledge work, 1M context, vision + tools (Pro plan and up) |
| `claude-sonnet-5-5` | LLM | Anthropic Claude Sonnet 5.5, the best combination of speed and intelligence, 1M context, vision + tools (Pro plan and up) |
| `claude-haiku-5-5` | LLM | Anthropic Claude Haiku 5.5 for high-volume, latency-sensitive work, 1M context, vision + tools (Pro plan and up) |
| `grok-4.3` | LLM | xAI Grok, 200K context |
| `DeepSeek-V4-Pro` | LLM | Flagship DeepSeek reasoning, 1M context, tools (text-only) |
| `DeepSeek-V4-Flash` | LLM | Fast DeepSeek reasoning, 1M context, tools |
| `gpt-realtime` | Realtime voice | OpenAI flagship speech-to-speech (10 concurrent), live — *deprecated, sunset 2026-11-16 → `gpt-realtime-2.1`* |
| `gpt-realtime-mini` | Realtime voice | Lowest-cost realtime, ~3× cheaper than gpt-realtime, live — *deprecated, sunset 2026-11-16 → `gpt-realtime-2.1-mini`* |
| `gpt-realtime-2` | Realtime voice | Answered by gpt-realtime-2.1 — *deprecated, sunset 2026-11-16 → `gpt-realtime-2.1`* |
| `gpt-realtime-1.5` | Realtime voice | Pinned 1.5 snapshot of gpt-realtime, live — *deprecated, sunset 2026-11-16 → `gpt-realtime-2.1`* |
| `gpt-realtime-2.1` | Realtime voice | Latest realtime — better recognition, silence/interrupt handling, live |
| `gpt-realtime-2.1-mini` | Realtime voice | Distilled low-cost 2.1 realtime, live |
| `gpt-live-1` | Realtime voice | GPT Live speech-to-speech — audio + text only, 14 voices, $0.05208/min, live |
| `whisper` | STT | OpenAI Whisper — 99 langs |
| `gpt-4o-transcribe` | STT | Streaming transcription |
| `gpt-4o-mini-transcribe` | STT | Low-cost streaming STT |
| `gpt-4o-transcribe-diarize` | STT | Speaker diarization |
| `gpt-4o-mini-tts` | TTS | Steerable OpenAI TTS, 13 voices |

See [Credits & Rate Limits](/docs/credits-rate-limits) for per-model USD pricing.

## Full Model Catalog

A curated, representative slice of the **134** models (64 LLM · 45 STT · 14 TTS · 9 image · 2 embedding) served by `GET /api/v1/models` as of the latest deploy — the per-domain variants of the direct Deepgram STT line and the `deepgram-voice-*` managed LLM ids are covered in their own sections above rather than repeated below. For live pricing and capability flags (`supports_vision`, `supports_tools`, `free`), query the API — it always reflects the current catalog.

### LLM (40 models)

| Model ID | Description | Context | Free | Pricing |
|----------|-------------|---------|------|---------|
| `sarvam-105b` | 105B MoE. Complex reasoning, agentic tasks, long documents. | 131K | Yes | $0.1563 in / $0.625 out per 1M |
| `sarvam-105b-conversations` | 105B MoE tuned for conversation and voice. Tool calling. | 32K | Yes | $0.1563 in / $0.625 out per 1M |
| `gpt-4o` | Multimodal text + vision. | 128K | No | $2.604 in / $10.42 out per 1M |
| `gemini-3.8-flash` | Fast multimodal flagship. Thinking low/medium/high. | 1M | No | $1.563 in / $7.813 out per 1M |
| `gemini-3.7-flash` | Fast multimodal. Thinking low/medium/high. | 1M | No | $1.563 in / $7.813 out per 1M |
| `gemini-3.6-flash` | Fast multimodal. Thinking minimal→high. | 1M | No | $1.563 in / $7.813 out per 1M |
| `gemini-3.5-flash` | Balanced multimodal workhorse. | 1M | No | $1.563 in / $9.375 out per 1M |
| `gemini-3.5-flash-lite` | Cheapest 1M-context Gemini. | 1M | No | $0.3125 in / $2.604 out per 1M |
| `gemini-3.1-pro-preview` | Reasoning-heavy Gemini tier. | 1M | No | $2.083 in / $12.5 out per 1M |
| `gemini-3.1-flash-lite` | Low-cost multimodal, tool use. | 1M | No | $0.2604 in / $1.563 out per 1M |
| `gpt-4.1` | Long-context multimodal. Strong instruction following. | 300K | No | $2.083 in / $8.333 out per 1M |
| `gpt-5-mini` | Fast, affordable reasoning. | 400K | No | $0.2604 in / $2.083 out per 1M |
| `gpt-5.5` | Reasoning flagship. Vision, tools, prompt caching. | 1.05M | No | $5.208 in / $31.25 out per 1M |
| `gpt-5.6-sol` | Frontier model for complex professional work. Vision, reasoning, tools. | 1.05M | No | $4.167 in / $20.83 out per 1M |
| `gpt-5.6-terra` | Balances intelligence and cost. Vision, reasoning, tools. | 1.05M | No | $2.083 in / $12.5 out per 1M |
| `gpt-5.6-luna` | Cost-sensitive, high-volume workloads. Vision, reasoning, tools. | 1.05M | No | $0.2083 in / $1.25 out per 1M |
| `gpt-6-sol` | Frontier reasoning for enterprise agents, coding and complex knowledge work. Vision, reasoning, tools. | 1.05M | No | $2.083 in / $10.42 out per 1M |
| `gpt-6-luna` | Efficient GPT-6 for high-volume, cost-sensitive workloads. Vision, reasoning, tools. | 1.05M | No | $0.1042 in / $0.5208 out per 1M |
| `gpt-6.1-sol` | Near-Astra performance for complex coding, computer use and professional work at a lower cost. Vision, reasoning, tools. | 1.05M | No | $2.083 in / $10.42 out per 1M |
| `claude-opus-5-5` | Anthropic Claude Opus 5.5 for long-running agentic coding and knowledge work. 128K max output. Vision, reasoning, tools. Pro plan and up. | 1M | No | $4.167 in / $20.83 out per 1M ($0.2083 cached input) |
| `claude-sonnet-5-5` | Anthropic Claude Sonnet 5.5, the best combination of speed and intelligence. 128K max output. Vision, reasoning, tools. Pro plan and up. | 1M | No | $2.083 in / $10.42 out per 1M ($0.1042 cached input) |
| `claude-haiku-5-5` | Anthropic Claude Haiku 5.5 for high-volume, latency-sensitive tasks such as classification, extraction and routing. 128K max output. Vision, reasoning, tools. Pro plan and up. | 1M | No | $0.1042 in / $0.5208 out per 1M ($0.01042 cached input); prompts over 100K tokens (cached tokens included) bill $0.5208 / $2.604 ($0.05208 cached) |
| `grok-4.3` | xAI Grok 4.3. Reasoning + vision (images of at least 512 pixels in total; smaller returns `400`). Reasoning tokens bill as output. | 200K | No | $3.646 in / $15.63 out per 1M |
| `DeepSeek-V4-Pro` | Flagship DeepSeek reasoning. Tools. Text-only. | 1M | No | $1.813 in / $4.125 out per 1M |
| `DeepSeek-V4-Flash` | Fast, affordable DeepSeek reasoning. Tools. Text-only. | 1M | No | $0.4583 in / $1.375 out per 1M |
| `kimi-k2.5` | Strong on coding and math. Vision. | 262K | Yes | $0.8438 in / $4.219 out per 1M |
| `kimi-k2.5-fast` *(maintenance)* | Kimi K2.5 at ~414 tok/s for voice-agent latency. | 256K | No | $0.8438 in / $4.219 out per 1M |
| `kimi-k2.6` | Improved reasoning and coding over K2.5. Vision. | 262K | Yes | $1.333 in / $5.625 out per 1M |
| `kimi-k2.7-code` | 1T-param agentic coding. Vision + tools. | 262K | Yes | $1.333 in / $5.625 out per 1M |
| `glm-5.3` | Z.ai GLM 5.3. Tools + reasoning (low / high / max). Text-only. | 1M | No | $2.125 in / $6.667 out per 1M |
| `deepseek-v4.1-flash` | DeepSeek V4.1 Flash. Tools + reasoning (low / high / max). Text-only. | 1M | No | $0.427 in / $1.771 out per 1M |
| `gpt-oss-120b` | Open-weight 120B MoE. Reasoning-grade at lower cost. | 131K | Yes | $1.042 in / $4.167 out per 1M |
| `gemma-4-31b` | Gemma 4 31B instruct. Fast with thinking off, vision, tools. | 128K | No | $0.3646 in / $0.9375 out per 1M |
| `gpt-realtime` | OpenAI flagship realtime speech-to-speech model — STT + reasoning + function calling + TTS in one. 10 concurrent. | 32K | No | $4.167 in / $16.67 out per 1M • $0.3906/min |
| `gpt-realtime-mini` | Lowest-cost realtime — same single-model shape as gpt-realtime, ~3× cheaper. 20 concurrent. | 32K | No | $0.625 in / $2.5 out per 1M • $0.1229/min |
| `gpt-realtime-2` | Newest realtime with stronger tool calling. 32K input, 4K output. | 32K | No | $4.167 in / $25 out per 1M • $0.3906/min |
| `gpt-realtime-1.5` | Pinned 1.5 snapshot of gpt-realtime. Use when you want version stability. | 32K | No | $4.167 in / $16.67 out per 1M • $0.3906/min |
| `gpt-realtime-2.1` | Latest realtime speech-to-speech — better alphanumeric recognition, silence/noise + interruption handling. Voice-agent only. | 32K | No | $4.167 in / $25 out per 1M • $0.3906/min |
| `gpt-realtime-2.1-mini` | Distilled, lower-cost realtime for faster voice interactions. Voice-agent only. | 32K | No | $0.625 in / $2.5 out per 1M • $0.1219/min |
| `gpt-live-1` | GPT Live speech-to-speech — audio + text in and out, function calling, 14 voices (default `marin`). No image or video input. Voice-agent only. | — | No | $0.05208/min (billed per second) |

### Speech to Text (11 models)

| Model ID | Description | Context | Free | Pricing |
|----------|-------------|---------|------|---------|
| `saaras:v4` | 23 languages (22 Indic + English). Five output modes: transcribe, translate, verbatim, transliterate, code-mix. | — | Yes | $0.3125 / hr |
| `gnani-prisma-v2.5` | India-first telephony STT. 10 Indian languages, code-switching. | — | No | $0.2813 / hr |
| `ink-whisper` | Cartesia Ink Whisper — 100 languages including Hindi, Urdu and Tamil. Dynamic chunking reduces hallucination across pauses and silence. File transcription + streaming. | — | No | $0.1875 / hr |
| `ink-2` | Cartesia Ink 2 — top-ranked for voice agents (8% WER on AppTek's 14-accent call-centre benchmark, vs 10% Deepgram Flux and 12% ElevenLabs). Self-detects turns. **Languages: `en`, `fr`, `hi`, `ja`, `es`. Voice sessions only — not available for file transcription.** | — | No | $0.5625 / hr |
| `assemblyai-universal-3.6-pro` | AssemblyAI Universal-3.6 Pro — newest streaming STT with built-in turn detection. 32 languages including Hindi, Marathi, Urdu and Indian English. **Voice sessions only — not available for file transcription.** | — | No | $0.4688 / hr |
| `inworld-stt-1` | Inworld STT 1 — low-cost streaming STT, English only. **Voice sessions only — not available for file transcription.** | — | No | $0.1563 / hr |
| `microsoft-speech-stt` | Microsoft Speech — real-time streaming STT, processed in India. 81 languages (148 locales) including Hindi, Bengali, Tamil, Telugu, Marathi, Gujarati, Kannada, Malayalam, Punjabi, Odia, Assamese, Urdu, Bhojpuri and Indian English; one language per call. **Voice sessions only — not available for file transcription.** | — | No | $1.0417 / hr |
| `whisper` | 99 languages. Transcription + translation to English. | — | No | $0.4167 / hr |
| `gpt-4o-transcribe` | Higher accuracy than Whisper. Streaming. | — | No | $0.4167 / hr |
| `gpt-4o-mini-transcribe` | Cheaper, faster streaming transcription. | — | No | $0.25 / hr |
| `gpt-4o-transcribe-diarize` | File transcription with speaker labels. Not available for voice sessions. | — | No | $0.4167 / hr |

### Text to Speech (12 models)

| Model ID | Description | Voices | Free | Pricing |
|----------|-------------|--------|------|---------|
| `bulbul:v3` | Indic TTS across 11 Indian languages. | 37 | Yes | $0.3125 / 10K chars |
| `bulbul:v4-flash` | Low-latency Indic TTS: 9 Indian languages + Hinglish. | 222 | Yes | $0.2083 / 10K chars |
| `gnani-timbre-v3.0` | Gnani Timbre v3 — India-first neural TTS, 10 Indian languages + Hinglish, streaming, expressive audio tags. | 12 | No | $0.2813 / 10K chars |
| `sonic-3.6` | Cartesia Sonic 3.6 — most natural conversational TTS. 44 languages, native-quality Hindi + Hinglish, sub-90ms first audio. | Searchable library | No | $0.5208 / 10K chars |
| `deepgram-flux-tts` | Deepgram Flux TTS — conversational, English only. **Voice sessions only.** | 16 | No | $0.4688 / 10K chars |
| `rime-coda` | Rime Coda — 8 languages including Hindi, two native Hindi voices. **Voice sessions only.** | 13 | No | $0.5208 / 10K chars |
| `inworld-tts-2` | Inworld TTS-2 — 94 languages including 11 Indian languages. **Voice sessions only.** | 14 | No | $0.2604 / 10K chars |
| `inworld-tts-2-flash` | Inworld TTS-2 Flash — faster, lower-cost TTS-2, same voices and languages. **Voice sessions only.** | 14 | No | $0.1563 / 10K chars |
| `fish-audio-drama-3` | Fish Audio Drama 3 — expressive, 13 languages, no Indian languages. **Voice sessions only.** | 7 | No | $0.5208 / 10K chars |
| `gradium-tts-beta` | Gradium TTS beta — English, French, German, Spanish and Portuguese. **Voice sessions only.** | 13 | No | $0.50 / 10K chars |
| `microsoft-neural-tts` | Microsoft Neural voices — neural TTS, processed in India. 13 languages including 12 Indian languages, plain text only. **Voice sessions only.** | 57 | No | $0.1563 / 10K chars |
| `gpt-4o-mini-tts` | Steerable — takes an `instructions` field to direct tone. | 6 | No | $0.2083 / 10K chars |

### Image Generation (9 models)

| Model ID | Description | Free | Pricing |
|----------|-------------|------|---------|
| `flux-2-pro` | Flux 2 Pro. Flagship BFL fidelity. | No | $0.1042 / image |
| `flux-1.1-pro` | Flux 1.1 Pro. Fast, production-grade. | No | $0.05208 / image |
| `gpt-image-2.5-sunburst` | Most capable generation + editing. Inpainting, quality tiers to `max`. | No | $0.2604 / image |
| `gemini-3.1-flash-lite-image` | Low-cost text-to-image with reference edits. 1K resolution. | No | $0.035 / image |
| `gpt-image-2.5-flare` | Fast, high-quality everyday generation. On-image text + edits. | No | $0.2604 / image |
| `gpt-image-2` | Accurate on-image text rendering. | No | $0.2604 / image |
| `gpt-image-1.5` | Precise image editing. Strong logo/face preservation. | No | $0.2604 / image |
| `nano-banana-2` | Fast multimodal image generation. | No | $0.06979 / image |
| `nano-banana-pro` | Flagship typography and fidelity. | No | $0.1396 / image |

### Embeddings (2 models)

| Model ID | Description | Dimensions | Free | Pricing |
|----------|-------------|------------|------|---------|
| `text-embedding-3-small` | Fast, low-cost embeddings. Best price/performance for large corpora. | 1536 | Yes | $0.02083 / 1M input tokens |
| `text-embedding-3-large` | Highest-accuracy embeddings. | 3072 | Yes | $0.1354 / 1M input tokens |

Both accept 8,192-token inputs and support shortening the vector with `dimensions`. See [Embeddings](/docs/embeddings).

> **Tip:** Filter programmatically — `GET /api/v1/models?category=llm`, `?category=stt`, `?category=tts`, `?category=image`, `?category=embedding`, or `?free=true` for free-plan models only.

## Model Selection

Pass the model ID in your request:

```python
# Indic LLM
response = client.chat.completions.create(
    model="sarvam-105b",
    messages=[{"role": "user", "content": "Hello in Hindi"}]
)

# Indic LLM with thinking mode
response = client.chat.completions.create(
    model="sarvam-105b",
    messages=[{"role": "user", "content": "Solve this step by step"}],
    extra_body={"reasoning_effort": "high"}
)

# First-party flagship model
response = client.chat.completions.create(
    model="gpt-6-luna",
    messages=[{"role": "user", "content": "Hello"}]
)
```

The API automatically routes to the correct backend based on the model ID:
- Bare names (`kimi-k2.5`, `gpt-6-sol`, `DeepSeek-V4-Pro`, `gpt-oss-120b`, …) → direct-routed or first-party
- `sarvam-*` prefix → Indic LLMs
- `saaras:*` / `bulbul:*` / `deepgram-*` / image IDs → the matching speech or image backend

Every ID is a plain CallMissed ID with no vendor prefix — including models we
[deploy on demand](#models-on-demand).
