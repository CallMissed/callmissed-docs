---
title: "Models"
description: "All available models on CallMissed — Indic STT/TTS/LLM, fast direct-routed models, and 300+ frontier text models. All accessible through one OpenAI-compatible API."
slug: "models"
breadcrumb: "Getting Started"
---

# Models

All available models on CallMissed — Indic STT/TTS/LLM, fast direct-routed models, and 300+ frontier text models. All accessible through one OpenAI-compatible API.

:::cards
/docs/model-access | Model Access by Plan | key | Free, Starter, Pro, and Enterprise model tiers
/docs/models-indic | Indic Models | mic | Indic STT, TTS, and LLM models
/docs/models-frontier | Frontier Models | boxes | 300+ frontier LLMs from every major provider
/docs/api-speed | API Speed | gauge | Latency benchmarks and reasoning-effort matrix
:::

## Overview

CallMissed provides access to a tiered model catalog through a single OpenAI-compatible API:

- **Fast LLMs** — Kimi K2.5 at up to ~414 tokens/second on high-throughput GPUs. The default and fastest LLM for voice agents.
- **Indic Models** — purpose-built for Indian languages. STT, TTS, and LLM optimized for Hindi, Tamil, Telugu, Bengali, and 19 more.
- **Direct-Routed LLMs** — sub-2s open-weights models (Kimi K2.5/K2.6, GPT-OSS, Gemma-4, GLM, Nemotron, Mistral Small).
- **First-Party Models** — first-party OpenAI, xAI, DeepSeek, and Amazon voice deployments (`gpt-4o`, `gpt-4.1`, `gpt-5-mini`, `grok-4.3`, `DeepSeek-V4-Pro`, `nova-sonic-2`, realtime voice) plus first-party STT/TTS.
- **Frontier Models** — frontier models from OpenAI, Anthropic, Google, xAI, Qwen, Mistral, and more through one endpoint.

All models use the same authentication and request format. Just change the `model` field.

## Models API

List all available models programmatically. **No authentication required.**

```bash
# List all models
curl https://api.callmissed.com/api/v1/models

# Filter by category: llm, stt, tts
curl https://api.callmissed.com/api/v1/models?category=llm

# Filter free-plan models only
curl https://api.callmissed.com/api/v1/models?free=true

# Get a specific model
curl https://api.callmissed.com/api/v1/models/sarvam-30b

# Which models each plan tier can call
curl https://api.callmissed.com/api/v1/models/access
```

Response includes: `id`, `name`, `description`, `category`, `owned_by`, `context_window`, `context_length` (alias of `context_window` for OpenAI-style clients), `pricing`, `free`, `supports_streaming`, `supports_tools`, `supports_reasoning`, and `supports_vision`.

The OpenAI-compatible listing at `GET /v1/models` (requires `Authorization: Bearer cm_*`) returns the same enriched fields, and the Anthropic-shape listing at `GET /anthropic/v1/models` surfaces them inside Anthropic's `{data, has_more, first_id, last_id}` envelope.

## Free Plan Models

The free tier includes **25 models** across four categories. Use `GET /api/v1/models?free=true` to list them, or see the [Model Access by Plan](/docs/model-access) page for the full breakdown.

### LLM (12 models)
| Model ID | Description |
|----------|-------------|
| `auto` | Free auto-router — picks a capable free model per request |
| `sarvam-30b` | 30B MoE — Indic languages, cost-efficient |
| `sarvam-105b` | 105B MoE — complex reasoning, Indic languages |
| `kimi-k2.5` | Moonshot K2.5 — 262K context, reasoning |
| `kimi-k2.6` | Moonshot K2.6 — improved reasoning + coding, 262K context |
| `kimi-k2.7-code` | Moonshot K2.7 Code — frontier 1T-param agentic coding, 262K context, vision + tools |
| `glm-4.7-flash` | GLM 4.7 Flash — fast inference |
| `glm-5.2` | GLM 5.2 — Z.ai flagship agentic coding, 262K context, tools + reasoning |
| `gpt-oss-120b` | GPT-OSS 120B — open-weights large model |
| `nemotron-3-super` | Nvidia Nemotron 3 Super |
| `gemma-4-26b-a4b-it` | Google Gemma 4 26B |
| `mistral-small-3.1` | Mistral Small 3.1 — 24B instruct, tool use |

### STT (3 models)
| Model ID | Description |
|----------|-------------|
| `saaras:v3` | 23 langs (22 Indic + English), best for code-mixed |
| `whisper-large-v3-turbo` | Whisper — 99 langs with auto-detect, transcribe + translate |
| `nova-3` | Nova 3 — 11 langs, diarization, smart-format, streaming-capable |

### TTS (4 models)
| Model ID | Description |
|----------|-------------|
| `bulbul:v3` | 37 voices, 11 Indian languages |
| `aura-2-en` | Aura 2 — 40 English voices, low-latency streaming |
| `aura-2-es` | Aura 2 — 10 Spanish voices, low-latency streaming |
| `melotts` | MeloTTS — en + fr, cheapest TTS available |

### Image (6 free + 5 paid)
| Model ID | Description |
|----------|-------------|
| `flux-2-klein-9b` | Flux 2 Klein — highest quality |
| `flux-2-dev` | Flux 2 Dev — flagship fidelity |
| `lucid-origin` | Lucid Origin — cinematic |
| `phoenix-1.0` | Phoenix — photorealistic |
| `sdxl-lightning` | SDXL Lightning — fast |
| `dreamshaper-8-lcm` | DreamShaper 8 LCM — fast |
| `flux-2-pro` | Flux 2 Pro — flagship BFL quality *(paid)* |
| `flux-1.1-pro` | Flux 1.1 Pro — fast high-quality *(paid)* |
| `nano-banana-2` | Google Gemini 3.1 Flash Image — multimodal, highest LM-Arena Elo (paid) |
| `nano-banana-pro` | Google Gemini 3 Pro Image — flagship typography + fidelity (paid) |

All other models — including `kimi-k2.5-fast`, first-party IDs (`gpt-4o`, `gpt-4.1`, `gpt-5-mini`, `grok-4.3`, `DeepSeek-V4-*`, `gpt-realtime*`, `nova-sonic*`, first-party STT/TTS), the Deepgram direct line (`deepgram-nova-3`, `deepgram-flux-general-en/multi`, `deepgram-nova-2*`, `deepgram-enhanced*`, `deepgram-base*`, `deepgram-whisper-*`, `deepgram-aura-2`, Deepgram Voice Agent `deepgram-voice-*` ids, the `deepgram-summarize/topics/sentiment/intents` Audio Intelligence features, and the `deepgram-text-summarize/topics/sentiment/intents` Text Intelligence features), slash-prefixed frontier IDs (`openai/*`, `anthropic/*`, `google/*`, `x-ai/*`, `qwen/*`, `mistralai/*`), and paid image models (`flux-2-pro`, `nano-banana-*`) — require Starter, Pro, or Enterprise.

### Pricing

All models are pay-per-use. Pricing is in USD.

| Model | Input / 1M tokens | Output / 1M tokens |
|-------|-------------------|-------------------|
| `kimi-k2.5-fast` | $0.81 | $4.05 |
| `sarvam-30b` | $0.35 (₹30) | $0.35 (₹30) |
| `sarvam-105b` | $0.35 (₹30) | $0.35 (₹30) |
| `openai/gpt-5.4-mini` | $1.00 | $6.00 |
| `openai/gpt-5.4` | $3.50 | $20.00 |
| `openai/gpt-5.4-pro` | $40.00 | $240.00 |
| `anthropic/claude-sonnet-4.6` | $4.00 | $20.00 |
| `anthropic/claude-opus-4.6` | $7.00 | $35.00 |
| `google/gemini-3.1-pro-preview` | $2.00 | $12.00 |
| `google/gemini-3-flash-preview` | $0.50 | $3.00 |
| `google/gemini-3.5-flash` | $1.50 | $9.00 |
| `google/gemini-3.1-flash-lite` | $0.25 | $1.50 |
| `nova-sonic-2` | $4.00 | $15.00 |
| `nova-sonic` | $4.50 | $17.00 |
| `gpt-realtime` | $4.00 | $16.00 |
| `gpt-realtime-mini` | $0.60 | $2.40 |
| `gpt-realtime-2` | $4.00 | $24.00 |
| `gpt-realtime-1.5` | $4.00 | $16.00 |
| `deepgram-voice-*` | per-minute Voice Agent tier | Standard $0.075/min, Advanced $0.163/min *(voice-agent only; runtime bridge in progress)* |

| STT Model | Price |
|-----------|-------|
| `saaras:v3` | $0.53 / hour (₹45/hr) |
| `gnani-prisma-v2.5` | $0.27 / hour |
| `whisper-large-v3-turbo` | $0.06 / hour |
| `nova-3` | $0.50 / hour |
| `deepgram-nova-3` | $0.29 / hour |
| `deepgram-nova-3-medical` | $0.29 / hour |
| `deepgram-flux-general-en` | $0.39 / hour |
| `deepgram-flux-general-multi` | $0.47 / hour |
| `deepgram-nova-2` (+ domain variants) | $0.35 / hour |
| `deepgram-nova` / `deepgram-whisper-*` | $0.35 / hour |
| `deepgram-enhanced` (+ variants) | $0.99 / hour |
| `deepgram-base` (+ variants) | $0.87 / hour |

| TTS Model | Price |
|-----------|-------|
| `bulbul:v3` | $0.53 / 10K chars (₹45/10K) |
| `gnani-timbre-v2.0` | $0.27 / 10K chars |
| `aura-2-en` | $0.40 / 10K chars |
| `aura-2-es` | $0.40 / 10K chars |
| `deepgram-aura-2` | $0.30 / 10K chars |
| `melotts` | $0.05 / 10K chars |

| Audio / Text Intelligence (Deepgram) | Price |
|-------------------------------|-------|
| `deepgram-summarize` / `-topics` / `-sentiment` / `-intents` (audio) | $0.0003 / 1K input + $0.0006 / 1K output tokens |
| `deepgram-text-summarize` / `-text-topics` / `-text-sentiment` / `-text-intents` | $0.0003 / 1K input + $0.0006 / 1K output tokens |

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
| `kimi-k2.5-fast` | **Under maintenance** — fall back to `kimi-k2.5` | 262K | Voice agents, fast inference, reasoning tasks |

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
| `saaras:v3` | Latest STT — best accuracy on Indian + code-mixed | 23 languages (22 Indic + English) |
| `gnani-prisma-v2.5` | India-first telephony STT — code-switching, sub-4% WER on Indian English | 10 Indian languages |

For 99-language general-purpose transcription, see `whisper-large-v3-turbo`. For diarization + smart-format on calls, see `nova-3`. Both are free-tier and live under the [audio model routes](#audio-models).

### Text to Speech

| Model | Description | Voices |
|-------|-------------|--------|
| `bulbul:v3` | Natural TTS — 37 voices, 11 Indian languages | shubh (default) + 36 more |
| `gnani-timbre-v2.0` | India-first neural TTS — context-aware tone, low-latency | 24 voices (English + Hindi) |

For low-latency English / Spanish voice agents, see `aura-2-en` / `aura-2-es`. For ultra-cheap en/fr notification audio, see `melotts`. All three are free-tier.

### Chat Completion (LLM)

| Model | Params | Context | Best For |
|-------|--------|---------|----------|
| `sarvam-30b` | 30B MoE (2.4B active) | 64K tokens | Real-time chat, Indic languages, cost-efficient |
| `sarvam-105b` | 105B MoE | 128K tokens | Complex reasoning, agentic tasks, long documents |

Both `sarvam-30b` and `sarvam-105b` support **hybrid thinking mode** via `reasoning_effort: "low" | "medium" | "high"`. `"none"` and `"minimal"` are mapped down to `"low"` (verified 2026-05-01) so OpenAI-style clients sending `reasoning_effort: "none"` for thinking-off still get a 200. Full thinking-disable is available on the direct-routed `kimi-k2.5` / `kimi-k2.6` / `kimi-k2.7-code` / `gemma-4-26b-a4b-it` models.

## Audio Models

Free-tier on every plan. See the [Pricing](/docs/pricing) page for current rates.

### Speech to Text

| Model | Languages | Best for | Price |
|-------|-----------|----------|-------|
| `whisper-large-v3-turbo` | 99 with auto-detect | Multilingual general-purpose; transcribe + translate | $0.06 / hour |
| `nova-3` | 11 BCP-47 incl. `multi` auto-detect | Diarization, smart-format, streaming voice agents | $0.50 / hour |
| `whisper` | 99 with auto-detect | Whisper batch + translate | $0.40 / hour |
| `gpt-4o-transcribe` | Streaming | Higher-accuracy OpenAI transcription | $0.40 / hour |
| `gpt-4o-mini-transcribe` | Streaming | Low-cost OpenAI transcription | $0.24 / hour |
| `gpt-4o-transcribe-diarize` | Streaming + diarization | Multi-speaker meetings / calls | $0.40 / hour |

**Deepgram (direct)** — the full Deepgram speech-to-text line, billed pass-through at Deepgram's published rates:

| Model | Languages | Best for | Price |
|-------|-----------|----------|-------|
| `deepgram-flux-general-en` | English | Conversational voice agents — model-native turn detection, ultra-low latency | $0.39 / hour |
| `deepgram-flux-general-multi` | 10 (multilingual) | Multilingual voice agents with code-switching | $0.47 / hour |
| `deepgram-nova-3` | 45+ incl. `multi` | Flagship general-purpose ASR, keyterm prompting, PII redaction | $0.29 / hour |
| `deepgram-nova-3-medical` | English | Clinical / medical terminology | $0.29 / hour |
| `deepgram-nova-2` | 36 incl. `multi` | High-accuracy ASR + filler-word detection | $0.35 / hour |
| `deepgram-nova-2-{meeting,phonecall,finance,conversationalai,voicemail,video,medical,drivethru,automotive,atc}` | English | Domain-tuned Nova-2 variants | $0.35 / hour |
| `deepgram-nova` / `-phonecall` / `-medical` | en/es/hi | Legacy Nova-1 | $0.35 / hour |
| `deepgram-enhanced` (+ meeting/phonecall/finance) | 13 | Legacy, keyword boosting | $0.99 / hour |
| `deepgram-base` (+ 6 variants) | 17 | Legacy, high-volume batch | $0.87 / hour |
| `deepgram-whisper-{tiny,base,small,medium,large}` | 99 | Deepgram-managed Whisper Cloud | $0.35 / hour |

### Text to Speech

| Model | Languages | Voices | Price |
|-------|-----------|--------|-------|
| `aura-2-en` | English | 40 (luna default) | $0.40 / 10K chars |
| `aura-2-es` | Spanish | 10 (aquila default) | $0.40 / 10K chars |
| `deepgram-aura-2` | en/es/de/fr/nl/it/ja | 90+ (thalia default) | $0.30 / 10K chars |
| `melotts` | English + French | 1 per language | $0.05 / 10K chars |
| `gpt-4o-mini-tts` | Multilingual steerable | 6 OpenAI voices | $0.20 / 10K chars |

Aura 2 returns linear16 PCM streamed at 24 kHz for low-latency playback. MeloTTS returns base64 MP3. Output formats may vary as models are updated.

### Audio Intelligence (Deepgram)

Deepgram Audio Intelligence runs analysis over an uploaded audio file via `POST /v1/audio/intelligence` (English only, 150K input-token limit). Token-billed at $0.0003/1K input + $0.0006/1K output.

| Feature | Model ID | Returns |
|---------|----------|---------|
| Summarization | `deepgram-summarize` | A concise `summary` of the audio |
| Topic Detection | `deepgram-topics` | Per-segment `topics` with confidence |
| Sentiment Analysis | `deepgram-sentiment` | Per-segment + average `sentiments` |
| Intent Recognition | `deepgram-intents` | Per-segment `intents` with confidence |

Request multiple features in one call with a comma-separated `features` form field (e.g. `features=deepgram-summarize,deepgram-sentiment`).

### Text Intelligence (Deepgram)

Deepgram Text Intelligence runs the same four analyses over **text** input (a string or a hosted text URL) via `POST /v1/text/intelligence` (English only, 150K input-token limit). Token-billed at $0.0003/1K input + $0.0006/1K output. Requires the `llm` key permission.

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

Low-latency models routed directly through CallMissed — sub-2s end-to-end on small prompts and free-tier eligible per the [reasoning_effort matrix](/docs/api-speed#3-reasoning-effort-matrix-per-model-verified-empirically).

| Model ID | Creator | Context |
|----------|---------|---------|
| `kimi-k2.5` | Moonshot AI | 262K |
| `kimi-k2.6` | Moonshot AI | 262K |
| `kimi-k2.7-code` | Moonshot AI | 262K |
| `gpt-oss-120b` | OpenAI (open-weights) | 128K |
| `gemma-4-26b-a4b-it` | Google | 128K |
| `glm-4.7-flash` | Zhipu | 128K |
| `glm-5.2` | Z.ai | 262K |
| `nemotron-3-super` | NVIDIA | 128K |
| `mistral-small-3.1` | Mistral | 128K |

## Frontier Models

Access frontier models via the same `/v1/chat/completions` endpoint. Use the slash-prefixed model ID as the `model` field.

> **"300+ models" — what that means.** CallMissed maintains a curated catalog of ~67 first-party models (Indic STT/TTS/LLM, direct-routed fast models, realtime speech-to-speech voice, image, and the popular frontier IDs below). On top of that, *any* of 300+ and growing frontier models is reachable as a passthrough by sending its slash-prefixed ID (e.g. `openai/gpt-5.4`) even if it isn't in our curated list. Passthrough models are billed at the listed per-model rate and aren't guaranteed to appear in `GET /v1/models`.

### Popular Models

| Model ID | Creator | Context |
|----------|---------|---------|
| `openai/gpt-5.4-pro` | OpenAI | 1M |
| `openai/gpt-5.4` | OpenAI | 1M |
| `openai/gpt-5.4-nano` | OpenAI | 1M |
| `openai/gpt-5.4-mini` | OpenAI | 1M |
| `anthropic/claude-opus-4.6` | Anthropic | 1M |
| `anthropic/claude-sonnet-4.6` | Anthropic | 1M |
| `anthropic/claude-haiku-4.5` | Anthropic | 200K |
| `google/gemini-3.1-pro-preview` | Google | 1M |
| `google/gemini-3-flash-preview` | Google | 1M |
| `google/gemini-3.5-flash` | Google | 1M |
| `google/gemini-3.1-flash-lite` | Google | 1M |
| `x-ai/grok-4.20` | xAI | 256K |
| `qwen/qwen3.5-plus` | Qwen | 256K |
| `qwen/qwen3.5-flash` | Qwen | 256K |
| `mistralai/mistral-small-2603` | Mistral | 128K |
| `auto` | Auto Router | — |

Use `auto` to let CallMissed select the best free model for your prompt automatically.


## First-Party Models

Credit-covered first-party models. Use the bare model ID in API requests — e.g. `gpt-4o`.

| Model ID | Type | Notes |
|----------|------|-------|
| `gpt-4o` | LLM | Multimodal text + vision, 128K context |
| `gpt-4.1` | LLM | Long-context (1M) multimodal |
| `gpt-5-mini` | LLM | Fast reasoning, 400K context |
| `grok-4.3` | LLM | xAI Grok, 200K context |
| `DeepSeek-V4-Pro` | LLM | Flagship DeepSeek reasoning, 1M context |
| `DeepSeek-V4-Flash` | LLM | Fast DeepSeek reasoning |
| `nova-sonic-2` | Realtime voice | Default speech-to-speech voice model — 16 voices, Hindi + en-IN, live |
| `nova-sonic` | Realtime voice | First-generation Amazon speech-to-speech voice model |
| `gpt-realtime` | Realtime voice | OpenAI flagship speech-to-speech (10 concurrent), live |
| `gpt-realtime-mini` | Realtime voice | Lowest-cost realtime, ~3× cheaper than gpt-realtime, live |
| `gpt-realtime-2` | Realtime voice | Newest realtime with stronger tool calling, live |
| `gpt-realtime-1.5` | Realtime voice | Pinned 1.5 snapshot of gpt-realtime, live |
| `whisper` | STT | OpenAI Whisper — 99 langs |
| `gpt-4o-transcribe` | STT | Streaming transcription |
| `gpt-4o-mini-transcribe` | STT | Low-cost streaming STT |
| `gpt-4o-transcribe-diarize` | STT | Speaker diarization |
| `gpt-4o-mini-tts` | TTS | Steerable OpenAI TTS, 6 voices |

See [Credits & Rate Limits](/docs/credits-rate-limits) for per-model USD pricing.

## Full Model Catalog

A curated, representative slice of the **132** models (72 LLM · 42 STT · 7 TTS · 11 image) served by `GET /api/v1/models` as of the latest deploy — the per-domain variants of the direct Deepgram STT line and the `deepgram-voice-*` managed LLM ids are covered in their own sections above rather than repeated below. For live pricing and capability flags (`supports_vision`, `supports_tools`, `free`), query the API — it always reflects the current catalog.

### LLM (42 models)

| Model ID | Description | Context | Free | Pricing |
|----------|-------------|---------|------|---------|
| `sarvam-30b` | 30B MoE (2.4B active params), 64K context. Best for real-time chat an… | 65K | Yes | $0.35 in / $0.35 out per 1M |
| `sarvam-105b` | 105B MoE, 128K context. Flagship model for complex reasoning and agen… | 131K | Yes | $0.35 in / $0.35 out per 1M |
| `openai/gpt-5.4-pro` | OpenAI's most capable model. 1M context, best for complex multi-step … | 1M | No | $40.00 in / $240.00 out per 1M |
| `openai/gpt-5.4` | OpenAI's flagship model. 1M context, strong general-purpose performance. | 1M | No | $3.50 in / $20.00 out per 1M |
| `openai/gpt-5.4-mini` | Fast and affordable. Great balance of speed and quality. | 1M | No | $1.00 in / $6.00 out per 1M |
| `openai/gpt-5.4-nano` | Fastest and cheapest OpenAI model. Best for simple tasks. | 1M | No | $0.27 in / $1.70 out per 1M |
| `gpt-4o` | OpenAI GPT-4o. Multimodal (text + vision), 128K context. Hos… | 128K | No | $2.50 in / $10.00 out per 1M |
| `gpt-4.1` | OpenAI GPT-4.1. Long-context (1M) multimodal model with stro… | 1M | No | $2.00 in / $8.00 out per 1M |
| `gpt-5-mini` | OpenAI GPT-5 Mini. Fast, affordable reasoning model. 400K co… | 400K | No | $0.25 in / $2.00 out per 1M |
| `gpt-5.5` | OpenAI GPT-5.5. Flagship reasoning, 1M context, multimodal + tool calling + prompt caching. | 1M | Yes | $5.00 in / $30.00 out per 1M |
| `grok-4.3` | xAI Grok 4.3. Strong reasoning and real-ti… | 200K | No | $3.50 in / $15.00 out per 1M |
| `DeepSeek-V4-Pro` | DeepSeek V4 Pro. Flagship reasoning model … | 1M | No | $1.00 in / $3.00 out per 1M |
| `DeepSeek-V4-Flash` | DeepSeek V4 Flash. Fast, affordable reason… | 131K | No | $0.30 in / $1.20 out per 1M |
| `anthropic/claude-opus-4.6` | Anthropic's most capable model. 1M context, excellent for coding and … | 1M | No | $7.00 in / $35.00 out per 1M |
| `anthropic/claude-sonnet-4.6` | Fast and intelligent. 1M-token context — best balance of speed and ca… | 1M | No | $4.00 in / $20.00 out per 1M |
| `anthropic/claude-haiku-4.5` | Anthropic's fastest, most cost-efficient model. 200K context, strong … | 200K | No | $1.35 in / $6.75 out per 1M |
| `google/gemini-3.1-pro-preview` | Google's flagship model. 1M context, agentic reasoning, multimodal. P… | 1M | No | $2.00 in / $12.00 out per 1M |
| `google/gemini-3-flash-preview` | Google's fast frontier model. 1M context, optimized for low latency. … | 1M | No | $0.50 in / $3.00 out per 1M |
| `google/gemini-3.5-flash` | Google's latest fast frontier model. 1M context, multimodal, tools + reasoning + caching. | 1M | No | $1.50 in / $9.00 out per 1M |
| `google/gemini-3.1-flash-lite` | Most affordable Gemini 3.x model. 1M context, low-latency. Pass-throu… | 1M | No | $0.25 in / $1.50 out per 1M |
| `x-ai/grok-4.20` | xAI's flagship model. 256K context, strong on reasoning and real-time… | 262K | No | $1.69 in / $3.38 out per 1M |
| `qwen/qwen3.5-plus` | Alibaba's flagship model. 256K native context, excellent multilingual… | 262K | No | $0.40 in / $2.40 out per 1M |
| `qwen/qwen3.5-flash` | Fast Qwen model. 256K native context, optimized for speed and cost. | 262K | No | $0.09 in / $0.35 out per 1M |
| `kimi-k2.5` | Moonshot AI's latest model. 256K context, strong on coding and math. | 262K | Yes | $0.81 in / $4.05 out per 1M |
| `kimi-k2.5-fast` *(maintenance)* | Kimi K2.5 tuned for sub-second latency — 414 tok/s inference, 256K co… | 262K | No | $0.81 in / $4.05 out per 1M |
| `kimi-k2.6` | Moonshot AI's latest K2.6 release. 256K context, improved reasoning a… | 262K | Yes | $1.28 in / $5.40 out per 1M |
| `kimi-k2.7-code` | Moonshot AI's frontier 1T-param agentic-coding model. 262K context, t… | 262K | Yes | $1.28 in / $5.40 out per 1M |
| `glm-4.7-flash` | Z.ai's fast, cost-efficient bilingual model. Strong tool use and 128K… | 131K | Yes | $0.50 in / $2.00 out per 1M |
| `glm-5.2` | Z.ai's flagship agentic coding model. 262K context, tools + reasoning. | 262K | Yes | $1.89 in / $5.94 out per 1M |
| `gpt-oss-120b` | OpenAI's open-weight 120B MoE. Reasoning-grade quality, lower cost th… | 131K | Yes | $1.00 in / $4.00 out per 1M |
| `nemotron-3-super` | NVIDIA Nemotron 3 — 120B MoE tuned for long-context reasoning. 1M-tok… | 1M | Yes | $1.50 in / $6.00 out per 1M |
| `gemma-4-26b-a4b-it` | Google Gemma 4 — 26B MoE (4B active). Efficient instruct model for ge… | 131K | Yes | $0.40 in / $1.60 out per 1M |
| `mistral-small-3.1` | Mistral Small 3.1 — 24B instruct, 128K context. Strong tool use, fast… | 131K | Yes | $0.47 in / $0.76 out per 1M |
| `mistralai/mistral-small-2603` | Mistral's efficient model. Fast and cost-effective for everyday tasks. | 131K | No | $0.20 in / $0.80 out per 1M |
| `auto` | Auto-router. Automatically selects a model per request, filtering by … | 200K | Yes | $0.30 in / $0.50 out per 1M |
| `openrouter/auto` | Automatically selects the best model for your prompt. | — | No | Pricing varies — depends on the model auto-selected for your prompt |
| `nova-sonic-2` | Amazon Nova 2 Sonic. Native speech-to-speech voice model — STT, reasoning, and TTS in one; 16 voices across 8 languages including Hindi + en-IN. | 32K | No | $4.00 in / $15.00 out per 1M • $0.064/min |
| `nova-sonic` | Amazon Nova Sonic 1.0. Native speech-to-speech voice model with 11 voices across English, Spanish, French, Italian, and German. | 32K | No | $4.50 in / $17.00 out per 1M • $0.071/min |
| `gpt-realtime` | OpenAI flagship realtime speech-to-speech model — STT + reasoning + function calling + TTS in one. 10 concurrent. | 32K | No | $4.00 in / $16.00 out per 1M • $0.375/min |
| `gpt-realtime-mini` | Lowest-cost realtime — same single-model shape as gpt-realtime, ~3× cheaper. 20 concurrent. | 32K | No | $0.60 in / $2.40 out per 1M • $0.118/min |
| `gpt-realtime-2` | Newest realtime with stronger tool calling. 128K text context. | 128K | No | $4.00 in / $24.00 out per 1M • $0.375/min |
| `gpt-realtime-1.5` | Pinned 1.5 snapshot of gpt-realtime. Use when you want version stability. | 32K | No | $4.00 in / $16.00 out per 1M • $0.375/min |

### Speech to Text (8 models)

| Model ID | Description | Context | Free | Pricing |
|----------|-------------|---------|------|---------|
| `saaras:v3` | Latest Indic STT model. 23 languages (22 Indic + English), best ac… | — | Yes | $0.53 / hr |
| `gnani-prisma-v2.5` | India-first telephony STT. 10 Indian languages, code-switching… | — | No | $0.27 / hr |
| `whisper-large-v3-turbo` | OpenAI's Whisper Large v3 Turbo. 100+ languages with auto-detect, tra… | — | Yes | $0.06 / hr |
| `nova-3` | Nova 3 — production-grade STT with diarization, punctuation,… | — | Yes | $0.50 / hr |
| `whisper` | OpenAI Whisper. 99 languages, transcription + translation to… | — | No | $0.40 / hr |
| `gpt-4o-transcribe` | OpenAI gpt-4o-transcribe — higher accuracy than Whisper, sup… | — | No | $0.40 / hr |
| `gpt-4o-mini-transcribe` | Cheaper, faster gpt-4o-mini-transcribe. Streaming transcript… | — | No | $0.24 / hr |
| `gpt-4o-transcribe-diarize` | gpt-4o-transcribe with speaker diarization — labels who spok… | — | No | $0.40 / hr |

### Text to Speech (6 models)

| Model ID | Description | Context | Free | Pricing |
|----------|-------------|---------|------|---------|
| `bulbul:v3` | Natural Indic TTS. 37 voices, 11 Indic languages. | — | Yes | — |
| `gnani-timbre-v2.0` | India-first neural TTS. 24 voices, English + Hindi, context-aware… | — | No | $0.27 / 10K chars |
| `aura-2-en` | Aura 2 — natural, conversational English TTS. 40 voices incl… | — | Yes | — |
| `aura-2-es` | Aura 2 Spanish — 10 native voices including aquila, sirio, d… | — | Yes | — |
| `melotts` | MyShell MeloTTS — fast, lightweight multilingual TTS. English + Frenc… | — | Yes | — |
| `gpt-4o-mini-tts` | Steerable TTS — accepts an `instructions` field to control t… | — | No | — |

### Image Generation (11 models)

| Model ID | Description | Context | Free | Pricing |
|----------|-------------|---------|------|---------|
| `flux-2-klein-9b` | Black Forest Labs' Flux 2 — high-quality text-to-image. 1024×1024 by … | — | Yes | $0.10 / image |
| `flux-2-pro` | Black Forest Labs' FLUX.2 Pro — flagship text-to-image with high fide… | — | No | $0.10 / image |
| `flux-1.1-pro` | Black Forest Labs' FLUX 1.1 Pro — fast, production-grade text-to-imag… | — | No | $0.05 / image |
| `flux-2-dev` | Black Forest Labs' Flux 2 Dev — higher fidelity, 50-step inference. | — | Yes | $0.12 / image |
| `lucid-origin` | Leonardo Lucid Origin — vibrant, cinematic compositions. Great for ma… | — | Yes | $0.08 / image |
| `phoenix-1.0` | Leonardo Phoenix 1.0 — strong prompt adherence, photorealistic portra… | — | Yes | $0.10 / image |
| `sdxl-lightning` | ByteDance distilled SDXL — 4-step inference, fastest for iterative pr… | — | Yes | $0.04 / image |
| `dreamshaper-8-lcm` | Lykon Dreamshaper 8 via LCM — stylised illustrations, fast generation. | — | Yes | $0.04 / image |
| `nano-banana-2` | Google Gemini 3.1 Flash Image — fast multimodal image generation with… | — | No | $0.07 / image |
| `nano-banana-pro` | Google Gemini 3 Pro Image — flagship image model with the highest qua… | — | No | $0.134 / image |

> **Tip:** Filter programmatically — `GET /api/v1/models?category=llm`, `?category=stt`, `?category=tts`, `?category=image`, or `?free=true` for free-plan models only.

## Model Selection

Pass the model ID in your request:

```python
# Indic LLM
response = client.chat.completions.create(
    model="sarvam-30b",
    messages=[{"role": "user", "content": "Hello in Hindi"}]
)

# Indic LLM with thinking mode
response = client.chat.completions.create(
    model="sarvam-105b",
    messages=[{"role": "user", "content": "Solve this step by step"}],
    extra_body={"reasoning_effort": "high"}
)

# Frontier model
response = client.chat.completions.create(
    model="openai/gpt-5.4-mini",
    messages=[{"role": "user", "content": "Hello"}]
)
```

The API automatically routes to the correct backend based on the model ID:
- Bare names (`kimi-k2.5`, `gpt-4o`, `DeepSeek-V4-Pro`, `mistral-small-3.1`, …) → direct-routed or first-party
- `sarvam-*` prefix → Indic LLMs
- Slash-prefixed (`openai/`, `anthropic/`, `google/`, …) → frontier catalog