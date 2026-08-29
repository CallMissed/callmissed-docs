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

122 models, one OpenAI-compatible API. Same auth, same request shape — change
the `model` field and nothing else.

| Group | What it is |
|-------|------------|
| **Fast LLMs** | Kimi K2.5 at up to ~414 tok/s. The default for voice agents. |
| **Indic models** | STT, TTS and LLM built for 22 Indian languages. |
| **Direct-routed LLMs** | Sub-2s open-weights models: Kimi K2.5/K2.6/K2.7 Code, GPT-OSS, Gemma 4, GLM, Nemotron, Mistral Small. |
| **First-party** | `gpt-4o`, `gpt-4.1`, `gpt-5-mini`, `gpt-5.5`, `gpt-5.6-*`, `grok-4.3`, `DeepSeek-V4-*`, realtime voice, plus first-party STT/TTS. |
| **On demand** | [300+ more we deploy on request](#models-on-demand). |

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
curl https://api.callmissed.com/api/v1/models/sarvam-105b

# Which models each plan tier can call
curl https://api.callmissed.com/api/v1/models/access
```

Response includes: `id`, `name`, `description`, `category`, `owned_by`, `context_window`, `context_length` (alias of `context_window` for OpenAI-style clients), `pricing`, `free`, `supports_streaming`, `supports_tools`, `supports_reasoning`, and `supports_vision`.

The OpenAI-compatible listing at `GET /v1/models` (requires `Authorization: Bearer cm_*`) returns the same fields but a **shorter list**: it hides models that are only valid for voice sessions (`nova-sonic*`, `gpt-realtime*`, `deepgram-voice-*`). Use `GET /api/v1/models` for the full catalog. The Anthropic-shape listing at `GET /anthropic/v1/models` returns the same set inside Anthropic's `{data, has_more, first_id, last_id}` envelope.

## Free Plan Models

The free tier includes **27 models** across five categories. Use `GET /api/v1/models?free=true` to list them, or see the [Model Access by Plan](/docs/model-access) page for the full breakdown.

### LLM (11 models)
| Model ID | Description |
|----------|-------------|
| `sarvam-105b` | 105B MoE — complex reasoning, Indic languages |
| `sarvam-105b-conversations` | 105B MoE tuned for conversation and voice — 128K context, tool calling |
| `kimi-k2.5` | Moonshot K2.5 — 256K context, reasoning |
| `kimi-k2.6` | Moonshot K2.6 — improved reasoning + coding, 262K context |
| `kimi-k2.7-code` | Moonshot K2.7 Code — frontier 1T-param agentic coding, 262K context, vision + tools |
| `glm-4.7-flash` | GLM 4.7 Flash — fast inference |
| `glm-5.2` | GLM 5.2 — Z.ai flagship agentic coding, 262K context, tools + reasoning |
| `gpt-oss-120b` | GPT-OSS 120B — open-weights large model |
| `nemotron-3-super` | Nvidia Nemotron 3 Super |
| `gemma-4-26b-a4b-it` | Google Gemma 4 26B |
| `mistral-small-3.1` | Mistral Small 3.1 — 24B instruct, tool use |

### STT (4 models)
| Model ID | Description |
|----------|-------------|
| `saaras:v3` | 23 langs (22 Indic + English), best for code-mixed |
| `saaras:v4` | 24 langs — five output modes: transcribe, translate, verbatim, transliterate, code-mix |
| `whisper-large-v3-turbo` | Whisper — 99 langs with auto-detect, transcribe + translate |
| `nova-3` | Nova 3 — 11 langs, diarization, smart-format, streaming-capable |

### TTS (4 models)
| Model ID | Description |
|----------|-------------|
| `bulbul:v3` | 37 voices, 11 Indian languages |
| `aura-2-en` | Aura 2 — 40 English voices, low-latency streaming |
| `aura-2-es` | Aura 2 — 10 Spanish voices, low-latency streaming |
| `melotts` | MeloTTS — en + fr, cheapest TTS available |

### Image (6 free + 6 paid)
| Model ID | Description |
|----------|-------------|
| `flux-2-klein-9b` | Flux 2 Klein — highest quality |
| `flux-2-dev` | Flux 2 Dev — flagship fidelity |
| `lucid-origin` | Lucid Origin — cinematic |
| `phoenix-1.0` | Phoenix — photorealistic |
| `sdxl-lightning` | SDXL Lightning — fast |
| `dreamshaper-8-lcm` | DreamShaper 8 LCM — fast |

### Embedding (2 models)
| Model ID | Description |
|----------|-------------|
| `text-embedding-3-small` | 1536 dimensions, 8,192-token inputs — best price/performance |
| `text-embedding-3-large` | 3072 dimensions, 8,192-token inputs — highest accuracy |
| `flux-2-pro` | Flux 2 Pro — flagship BFL quality *(paid)* |
| `flux-1.1-pro` | Flux 1.1 Pro — fast high-quality *(paid)* |
| `gpt-image-2` | OpenAI GPT Image 2 — accurate on-image text *(paid)* |
| `gpt-image-1.5` | OpenAI GPT Image 1.5 — precise image editing, strong logo/face preservation *(paid)* |
| `nano-banana-2` | Google Gemini 3.1 Flash Image — multimodal, highest LM-Arena Elo *(paid · maintenance)* |
| `nano-banana-pro` | Google Gemini 3 Pro Image — flagship typography + fidelity *(paid · maintenance)* |

A free-plan key calling a paid model gets `403 model_not_available` — it is not billed, it is refused. Upgrade to Starter or above first.

All other models — including `kimi-k2.5-fast`, first-party IDs (`gpt-4o`, `gpt-4.1`, `gpt-5-mini`, `gpt-5.5`, `gpt-5.6-*`, `grok-4.3`, `DeepSeek-V4-*`, `gpt-realtime*`, `nova-sonic*`, first-party STT/TTS), the Deepgram direct line (`deepgram-nova-3`, `deepgram-flux-general-en/multi`, `deepgram-nova-2*`, `deepgram-enhanced*`, `deepgram-base*`, `deepgram-whisper-*`, `deepgram-aura-2`, `deepgram-aura-1`, Deepgram Voice Agent `deepgram-voice-*` ids, the `deepgram-summarize/topics/sentiment/intents` Audio Intelligence features, and the `deepgram-text-summarize/topics/sentiment/intents` Text Intelligence features), and paid image models (`flux-2-pro`, `gpt-image-2`, `gpt-image-1.5`, `nano-banana-*`) — require Starter, Pro, or Enterprise.

### Pricing

All models are pay-per-use. Pricing is in USD.

| Model | Input / 1M tokens | Output / 1M tokens |
|-------|-------------------|-------------------|
| `kimi-k2.5-fast` | $0.81 | $4.05 |
| `sarvam-105b` | $0.35 (₹30) | $0.35 (₹30) |
| `sarvam-105b-conversations` | $0.35 (₹30) | $0.35 (₹30) |
| `gpt-5.6-sol` | $5.00 | $30.00 |
| `gpt-5.6-terra` | $2.00 | $12.00 |
| `gpt-5.6-luna` | $0.20 | $1.20 |
| `nova-sonic-2` | $4.00 | $15.00 |
| `nova-sonic` | $4.50 | $17.00 |
| `gpt-realtime` | $4.00 | $16.00 |
| `gpt-realtime-mini` | $0.60 | $2.40 |
| `gpt-realtime-2` | $4.00 | $24.00 |
| `gpt-realtime-1.5` | $4.00 | $16.00 |
| `gpt-realtime-2.1` | $4.00 | $24.00 |
| `gpt-realtime-2.1-mini` | $0.60 | $2.40 |
| `deepgram-voice-*` | per-minute Voice Agent tier | Standard $0.075/min, Advanced $0.163/min *(voice-agent only)* |

| STT Model | Price |
|-----------|-------|
| `saaras:v3` | $0.30 / hour (₹30/hr) |
| `saaras:v4` | $0.30 / hour (₹30/hr) |
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
| `bulbul:v3` | $0.30 / 10K chars (₹30/10K) |
| `gnani-timbre-v2.0` | $0.27 / 10K chars |
| `aura-2-en` | $0.40 / 10K chars |
| `aura-2-es` | $0.40 / 10K chars |
| `deepgram-aura-2` | $0.30 / 10K chars |
| `deepgram-aura-1` | $0.15 / 10K chars |
| `sonic-3.6` | $0.50 / 10K chars |
| `melotts` | $0.05 / 10K chars |

| Intelligence feature (not a model ID) | Price |
|-------------------------------|-------|
| `deepgram-summarize` / `-topics` / `-sentiment` / `-intents` (audio) | $0.0003 / 1K input + $0.0006 / 1K output tokens |
| `deepgram-text-summarize` / `-text-topics` / `-text-sentiment` / `-text-intents` | $0.0003 / 1K input + $0.0006 / 1K output tokens |

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
| `saaras:v3` | Latest STT — best accuracy on Indian + code-mixed | 23 languages (22 Indic + English) |
| `saaras:v4` | Five output modes on one model — transcribe, translate, verbatim, transliterate, code-mix | 24 languages |
| `gnani-prisma-v2.5` | India-first telephony STT — code-switching, sub-4% WER on Indian English | 10 Indian languages |

For 99-language general-purpose transcription, see `whisper-large-v3-turbo`. For diarization + smart-format on calls, see `nova-3`. Both are free-tier and live under the [audio model routes](#audio-models).

### Text to Speech

| Model | Description | Voices |
|-------|-------------|--------|
| `bulbul:v3` | Natural TTS — 37 voices, 11 Indian languages | shubh (default) + 36 more |
| `gnani-timbre-v2.0` | India-first neural TTS — context-aware tone, low-latency | 73 voices (English + Hindi + Indic) |
| `sonic-3.6` | Cartesia Sonic 3.6 — most natural conversational speech, 44 languages with native-quality Hindi | 16 voices incl. 5 native-Hindi (skylar default) |

For low-latency English / Spanish voice agents, see `aura-2-en` / `aura-2-es`. For ultra-cheap en/fr notification audio, see `melotts`. All three are free-tier.

### Chat Completion (LLM)

| Model | Params | Context | Best For |
|-------|--------|---------|----------|
| `sarvam-105b` | 105B MoE | 128K tokens | Complex reasoning, agentic tasks, long documents |
| `sarvam-105b-conversations` | 105B MoE | 128K tokens | Conversation and voice agents, tool calling |

Both Sarvam models support hybrid thinking via `reasoning_effort: "low" | "medium" | "high"`. `"none"` and `"minimal"` map down to `"low"`, so an OpenAI-style client sending `"none"` gets a 200 rather than an error — but thinking stays on. To turn thinking fully off, use `kimi-k2.5`, `kimi-k2.6`, `glm-4.7-flash`, `glm-5.2`, or `gemma-4-26b-a4b-it` with `reasoning_effort: "none"`. See the [per-model matrix](/docs/api-speed#3-reasoning-effort-by-model).

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

**Deepgram (direct)** — the full Deepgram speech-to-text line, billed per audio hour at the rates below:

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
| `deepgram-aura-1` | English | 12 (asteria default) | $0.15 / 10K chars |
| `melotts` | English + French | 1 per language | $0.05 / 10K chars |
| `gpt-4o-mini-tts` | Multilingual steerable | 6 OpenAI voices | $0.20 / 10K chars |

Aura 2 returns linear16 PCM streamed at 24 kHz for low-latency playback. MeloTTS returns base64 MP3. Output formats may vary as models are updated.

Deepgram Flux TTS is a voice-agent-first model and is **not** available on this `/v1/audio/speech` endpoint. It is offered only through the managed Voice Agent (see the Voice Sessions API), selectable with `tts_engine: "flux"`, where it is billed inside the per-minute voice rate.

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

Low-latency models routed directly through CallMissed — sub-2s end-to-end on small prompts and free-tier eligible per the [reasoning_effort matrix](/docs/api-speed#3-reasoning-effort-by-model).

| Model ID | Creator | Context |
|----------|---------|---------|
| `kimi-k2.5` | Moonshot AI | 256K |
| `kimi-k2.6` | Moonshot AI | 262K |
| `kimi-k2.7-code` | Moonshot AI | 262K |
| `gpt-oss-120b` | OpenAI (open-weights) | 128K |
| `gemma-4-26b-a4b-it` | Google | 128K |
| `glm-4.7-flash` | Zhipu | 128K |
| `glm-5.2` | Z.ai | 262K |
| `nemotron-3-super` | NVIDIA | 256K |
| `mistral-small-3.1` | Mistral | 128K |

## Models on Demand

`GET /api/v1/models` lists everything that is live today: **122** model IDs
callable right now with a `cm_` key.

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
| `gpt-4o` | LLM | Multimodal text + vision, 128K context |
| `gpt-4.1` | LLM | Long-context (1M) multimodal |
| `gpt-5-mini` | LLM | Fast reasoning, 400K context |
| `gpt-5.5` | LLM | GPT-5.5 reasoning flagship, 1M context, vision + tools |
| `gpt-5.6-sol` | LLM | GPT-5.6 flagship, 1.05M context, vision + tools |
| `gpt-5.6-terra` | LLM | GPT-5.6 balanced intelligence/cost, 1.05M context |
| `gpt-5.6-luna` | LLM | GPT-5.6 fast + affordable, 1.05M context |
| `grok-4.3` | LLM | xAI Grok, 200K context |
| `DeepSeek-V4-Pro` | LLM | Flagship DeepSeek reasoning, 1M context, vision + tools |
| `DeepSeek-V4-Flash` | LLM | Fast DeepSeek reasoning, 1M context, tools |
| `nova-sonic-2` | Realtime voice | Default speech-to-speech voice model — 16 voices, Hindi + en-IN, live |
| `nova-sonic` | Realtime voice | First-generation Amazon speech-to-speech voice model |
| `gpt-realtime` | Realtime voice | OpenAI flagship speech-to-speech (10 concurrent), live |
| `gpt-realtime-mini` | Realtime voice | Lowest-cost realtime, ~3× cheaper than gpt-realtime, live |
| `gpt-realtime-2` | Realtime voice | Newest realtime with stronger tool calling, live |
| `gpt-realtime-1.5` | Realtime voice | Pinned 1.5 snapshot of gpt-realtime, live |
| `gpt-realtime-2.1` | Realtime voice | Latest realtime — better recognition, silence/interrupt handling, configurable reasoning, live |
| `gpt-realtime-2.1-mini` | Realtime voice | Distilled low-cost 2.1 realtime, live |
| `whisper` | STT | OpenAI Whisper — 99 langs |
| `gpt-4o-transcribe` | STT | Streaming transcription |
| `gpt-4o-mini-transcribe` | STT | Low-cost streaming STT |
| `gpt-4o-transcribe-diarize` | STT | Speaker diarization |
| `gpt-4o-mini-tts` | TTS | Steerable OpenAI TTS, 6 voices |

See [Credits & Rate Limits](/docs/credits-rate-limits) for per-model USD pricing.

## Full Model Catalog

A curated, representative slice of the **123** models (57 LLM · 43 STT · 9 TTS · 12 image · 2 embedding) served by `GET /api/v1/models` as of the latest deploy — the per-domain variants of the direct Deepgram STT line and the `deepgram-voice-*` managed LLM ids are covered in their own sections above rather than repeated below. For live pricing and capability flags (`supports_vision`, `supports_tools`, `free`), query the API — it always reflects the current catalog.

### LLM (30 models)

| Model ID | Description | Context | Free | Pricing |
|----------|-------------|---------|------|---------|
| `sarvam-105b` | 105B MoE. Complex reasoning, agentic tasks, long documents. | 131K | Yes | $0.35 in / $0.35 out per 1M |
| `sarvam-105b-conversations` | 105B MoE tuned for conversation and voice. Tool calling. | 131K | Yes | $0.35 in / $0.35 out per 1M |
| `gpt-4o` | Multimodal text + vision. | 128K | No | $2.50 in / $10.00 out per 1M |
| `gpt-4.1` | Long-context multimodal. Strong instruction following. | 1M | No | $2.00 in / $8.00 out per 1M |
| `gpt-5-mini` | Fast, affordable reasoning. | 400K | No | $0.25 in / $2.00 out per 1M |
| `gpt-5.5` | Reasoning flagship. Vision, tools, prompt caching. | 1M | No | $5.00 in / $30.00 out per 1M |
| `gpt-5.6-sol` | Frontier model for complex professional work. Vision, reasoning, tools. | 1.05M | No | $5.00 in / $30.00 out per 1M |
| `gpt-5.6-terra` | Balances intelligence and cost. Vision, reasoning, tools. | 1.05M | No | $2.00 in / $12.00 out per 1M |
| `gpt-5.6-luna` | Cost-sensitive, high-volume workloads. Vision, reasoning, tools. | 1.05M | No | $0.20 in / $1.20 out per 1M |
| `grok-4.3` | xAI Grok 4.3. Reasoning + vision. | 200K | No | $3.50 in / $15.00 out per 1M |
| `DeepSeek-V4-Pro` | Flagship DeepSeek reasoning. Vision + tools. | 1M | No | $1.32 in / $3.96 out per 1M |
| `DeepSeek-V4-Flash` | Fast, affordable DeepSeek reasoning. Tools. | 1M | No | $0.44 in / $1.32 out per 1M |
| `kimi-k2.5` | Strong on coding and math. Vision. | 256K | Yes | $0.81 in / $4.05 out per 1M |
| `kimi-k2.5-fast` *(maintenance)* | Kimi K2.5 at ~414 tok/s for voice-agent latency. | 256K | No | $0.81 in / $4.05 out per 1M |
| `kimi-k2.6` | Improved reasoning and coding over K2.5. Vision. | 262K | Yes | $1.28 in / $5.40 out per 1M |
| `kimi-k2.7-code` | 1T-param agentic coding. Vision + tools. | 262K | Yes | $1.28 in / $5.40 out per 1M |
| `glm-4.7-flash` | Fast, cost-efficient bilingual model. Strong tool use. | 131K | Yes | $0.50 in / $2.00 out per 1M |
| `glm-5.2` | Flagship agentic coding. Tools + reasoning. | 262K | Yes | $1.89 in / $5.94 out per 1M |
| `gpt-oss-120b` | Open-weight 120B MoE. Reasoning-grade at lower cost. | 128K | Yes | $1.00 in / $4.00 out per 1M |
| `nemotron-3-super` | 120B MoE tuned for long-context reasoning. | 256K | Yes | $1.50 in / $6.00 out per 1M |
| `gemma-4-26b-a4b-it` | 26B MoE (4B active). Efficient instruct model. Vision. | 131K | Yes | $0.40 in / $1.60 out per 1M |
| `mistral-small-3.1` | 24B instruct. Strong tool use, fast. Vision. | 128K | Yes | $0.47 in / $0.76 out per 1M |
| `nova-sonic-2` | Amazon Nova 2 Sonic. Native speech-to-speech voice model — STT, reasoning, and TTS in one; 16 voices across 8 languages including Hindi + en-IN. | 32K | No | $4.00 in / $15.00 out per 1M • $0.064/min |
| `nova-sonic` | Amazon Nova Sonic 1.0. Native speech-to-speech voice model with 11 voices across English, Spanish, French, Italian, and German. | 32K | No | $4.50 in / $17.00 out per 1M • $0.071/min |
| `gpt-realtime` | OpenAI flagship realtime speech-to-speech model — STT + reasoning + function calling + TTS in one. 10 concurrent. | 32K | No | $4.00 in / $16.00 out per 1M • $0.375/min |
| `gpt-realtime-mini` | Lowest-cost realtime — same single-model shape as gpt-realtime, ~3× cheaper. 20 concurrent. | 32K | No | $0.60 in / $2.40 out per 1M • $0.118/min |
| `gpt-realtime-2` | Newest realtime with stronger tool calling. 128K text context. | 128K | No | $4.00 in / $24.00 out per 1M • $0.375/min |
| `gpt-realtime-1.5` | Pinned 1.5 snapshot of gpt-realtime. Use when you want version stability. | 32K | No | $4.00 in / $16.00 out per 1M • $0.375/min |
| `gpt-realtime-2.1` | Latest realtime speech-to-speech — better alphanumeric recognition, silence/noise + interruption handling, configurable reasoning effort. Voice-agent only. | 128K | No | $4.00 in / $24.00 out per 1M • $0.375/min |
| `gpt-realtime-2.1-mini` | Distilled, lower-cost realtime for faster voice interactions. Voice-agent only. | 128K | No | $0.60 in / $2.40 out per 1M • $0.117/min |

### Speech to Text (9 models)

| Model ID | Description | Context | Free | Pricing |
|----------|-------------|---------|------|---------|
| `saaras:v3` | 23 languages (22 Indic + English). Best on code-mixed speech. | — | Yes | $0.30 / hr |
| `saaras:v4` | 24 languages. Five output modes: transcribe, translate, verbatim, transliterate, code-mix. | — | Yes | $0.30 / hr |
| `gnani-prisma-v2.5` | India-first telephony STT. 10 Indian languages, code-switching. | — | No | $0.27 / hr |
| `whisper-large-v3-turbo` | 99 languages with auto-detect. Transcribe + translate. | — | Yes | $0.06 / hr |
| `nova-3` | Diarization, punctuation, smart-format. Streaming-capable. | — | Yes | $0.50 / hr |
| `whisper` | 99 languages. Transcription + translation to English. | — | No | $0.40 / hr |
| `gpt-4o-transcribe` | Higher accuracy than Whisper. Streaming. | — | No | $0.40 / hr |
| `gpt-4o-mini-transcribe` | Cheaper, faster streaming transcription. | — | No | $0.24 / hr |
| `gpt-4o-transcribe-diarize` | Streaming transcription with speaker labels. | — | No | $0.40 / hr |

### Text to Speech (7 models)

| Model ID | Description | Voices | Free | Pricing |
|----------|-------------|--------|------|---------|
| `bulbul:v3` | Indic TTS across 11 Indian languages. | 37 | Yes | $0.30 / 10K chars |
| `gnani-timbre-v2.0` | India-first neural TTS, English + Hindi + Indic. Context-aware tone. | 73 | No | $0.27 / 10K chars |
| `sonic-3.6` | Cartesia Sonic 3.6 — most natural conversational TTS. 44 languages, native-quality Hindi + Hinglish, sub-90ms first audio. | 16 | No | $0.50 / 10K chars |
| `aura-2-en` | Conversational English TTS, low-latency streaming. | 40 | Yes | $0.40 / 10K chars |
| `aura-2-es` | Spanish TTS, low-latency streaming. | 10 | Yes | $0.40 / 10K chars |
| `melotts` | Lightweight English + French TTS. Cheapest available. | 1 per language | Yes | $0.05 / 10K chars |
| `gpt-4o-mini-tts` | Steerable — takes an `instructions` field to direct tone. | 6 | No | $0.20 / 10K chars |

### Image Generation (12 models)

| Model ID | Description | Free | Pricing |
|----------|-------------|------|---------|
| `flux-2-klein-9b` | Flux 2 Klein. 1024×1024 default. | Yes | $0.10 / image |
| `flux-2-dev` | Flux 2 Dev. Higher fidelity, 50-step inference. | Yes | $0.12 / image |
| `flux-2-pro` | Flux 2 Pro. Flagship BFL fidelity. | No | $0.10 / image |
| `flux-1.1-pro` | Flux 1.1 Pro. Fast, production-grade. | No | $0.05 / image |
| `gpt-image-2` | Accurate on-image text rendering. | No | $0.25 / image |
| `gpt-image-1.5` | Precise image editing. Strong logo/face preservation. | No | $0.25 / image |
| `lucid-origin` | Vibrant, cinematic compositions. | Yes | $0.08 / image |
| `phoenix-1.0` | Strong prompt adherence, photorealistic portraits. | Yes | $0.10 / image |
| `sdxl-lightning` | 4-step inference. Fastest for iterative prompting. | Yes | $0.04 / image |
| `dreamshaper-8-lcm` | Stylised illustrations, fast generation. | Yes | $0.04 / image |
| `nano-banana-2` *(maintenance)* | Fast multimodal image generation. | No | $0.067 / image |
| `nano-banana-pro` *(maintenance)* | Flagship typography and fidelity. | No | $0.134 / image |

### Embeddings (2 models)

| Model ID | Description | Dimensions | Free | Pricing |
|----------|-------------|------------|------|---------|
| `text-embedding-3-small` | Fast, low-cost embeddings. Best price/performance for large corpora. | 1536 | Yes | $0.02 / 1M input tokens |
| `text-embedding-3-large` | Highest-accuracy embeddings. | 3072 | Yes | $0.13 / 1M input tokens |

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
    model="gpt-5.6-luna",
    messages=[{"role": "user", "content": "Hello"}]
)
```

The API automatically routes to the correct backend based on the model ID:
- Bare names (`kimi-k2.5`, `gpt-4o`, `DeepSeek-V4-Pro`, `mistral-small-3.1`, …) → direct-routed or first-party
- `sarvam-*` prefix → Indic LLMs
- `saaras:*` / `bulbul:*` / `deepgram-*` / image IDs → the matching speech or image backend

Every ID is a plain CallMissed ID with no vendor prefix — including models we
[deploy on demand](#models-on-demand).
