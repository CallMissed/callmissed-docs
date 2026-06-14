---
title: "Model Access by Plan"
description: "See which models each plan tier can call. Free plan includes 23 models across LLM, STT, TTS, and Image."
slug: "models-access"
breadcrumb: "Getting Started > Models"
---

# Model Access by Plan

See which models each plan tier can call. Free plan includes 23 models across LLM, STT, TTS, and Image.

## Overview

Not all models are available on every plan. The free tier includes 23 models; paid plans (Starter, Pro, Enterprise) unlock all models.

Use the `/api/v1/models/access` endpoint to programmatically check which models your plan can call.

## Endpoint

`GET /api/v1/models/access` — No authentication required.

:::tabs
```bash [cURL]
curl https://api.callmissed.com/api/v1/models/access
```
```python [Python]
import requests

resp = requests.get("https://api.callmissed.com/api/v1/models/access")
data = resp.json()

# Check free plan models
free = data["plans"]["free"]
print(f"Free plan: {len(free['models'])} models")
for category, ids in free["by_category"].items():
    if ids:
        print(f"  {category}: {', '.join(ids)}")
```
```javascript [JavaScript]
const resp = await fetch("https://api.callmissed.com/api/v1/models/access");
const data = await resp.json();

// Check free plan models
const free = data.plans.free;
console.log(`Free plan: ${free.models.length} models`);
Object.entries(free.by_category).forEach(([cat, ids]) => {
  if (ids.length) console.log(`  ${cat}: ${ids.join(", ")}`);
});
```
:::

**Response shape:**

```json
{
  "plans": {
    "free": {
      "models": ["sarvam-30b", "sarvam-105b", "kimi-k2.5", "..."],
      "by_category": {
        "llm": ["auto", "sarvam-30b", "sarvam-105b", "kimi-k2.5", "kimi-k2.6", "kimi-k2.7-code", "glm-4.7-flash", "gpt-oss-120b", "nemotron-3-super", "gemma-4-26b-a4b-it", "mistral-small-3.1"],
        "stt": ["saaras:v3", "whisper-large-v3-turbo", "nova-3"],
        "tts": ["bulbul:v3", "aura-2-en", "aura-2-es", "melotts"],
        "image": ["flux-2-klein-9b", "flux-2-dev", "lucid-origin", "phoenix-1.0", "sdxl-lightning", "dreamshaper-8-lcm"]
      },
      "restriction": "Only the models listed here are callable on the free plan."
    },
    "starter": {
      "models": ["...all models..."],
      "by_category": { "llm": [...], "stt": [...], "tts": [...], "image": [...] },
      "restriction": "All models callable. Per-service monthly call limits apply."
    },
    "pro": { "..." : "..." },
    "enterprise": { "..." : "..." }
  }
}
```

## Free Plan Models

The free plan includes 23 models. Source of truth: `FREE_TIER_MODELS` in `backend/app/api/v1/services.py`.

**LLM (10):**

| Model ID | Description |
|----------|-------------|
| `auto` | Free auto-router — picks a free model per request |
| `sarvam-30b` | 30B MoE, 64K context — real-time chat, Indic languages |
| `sarvam-105b` | 105B MoE, 128K context — complex reasoning |
| `kimi-k2.5` | Moonshot 1T MoE, 256K context — coding, math |
| `kimi-k2.6` | Moonshot K2.6 — improved reasoning + coding, 256K context |
| `kimi-k2.7-code` | Moonshot K2.7 Code — frontier 1T-param agentic coding, 262K context |
| `glm-4.7-flash` | Z.ai bilingual, 128K context — fast, tool use |
| `gpt-oss-120b` | OpenAI open-weight 120B MoE — reasoning-grade |
| `nemotron-3-super` | NVIDIA 120B MoE — long-context enterprise |
| `gemma-4-26b-a4b-it` | Google 26B MoE — efficient chat + code |
| `mistral-small-3.1` | Mistral 24B Instruct, 128K context — fast, tool use |

**STT (3):**

| Model ID | Description |
|----------|-------------|
| `saaras:v3` | 23 langs (22 Indic + English), best for code-mixed |
| `whisper-large-v3-turbo` | Whisper — 99 langs with auto-detect, transcribe + translate |
| `nova-3` | Nova 3 — 11 langs, diarize, smart-format, streaming |

**TTS (4):**

| Model ID | Description |
|----------|-------------|
| `bulbul:v3` | 39 voices, 11 Indian languages |
| `aura-2-en` | Aura 2 — 39 English voices, low-latency streaming |
| `aura-2-es` | Aura 2 — 10 Spanish voices, low-latency streaming |
| `melotts` | MeloTTS — en + fr, cheapest TTS available |

**Image (6):**

| Model ID | Description |
|----------|-------------|
| `flux-2-klein-9b` | Flux 2 Klein 9B — high-quality text-to-image |
| `flux-2-dev` | Flux 2 Dev — higher fidelity, 50-step |
| `lucid-origin` | Lucid Origin — cinematic compositions |
| `phoenix-1.0` | Phoenix 1.0 — photorealistic portraits |
| `sdxl-lightning` | SDXL Lightning — fastest, 4-step inference |
| `dreamshaper-8-lcm` | DreamShaper 8 LCM — stylised illustrations |

## Paid Plans

Starter, Pro, and Enterprise plans unlock all models. The only difference between paid tiers is per-service monthly call limits (see [Credits & Rate Limits](/docs/credits-rate-limits)).

Paid-only models include: all Azure-hosted IDs (`gpt-4o`, `gpt-4.1`, `gpt-5-mini`, `grok-4.3`, `DeepSeek-V4-Pro`, `DeepSeek-V4-Flash`, `gpt-realtime`, `gpt-realtime-mini`, `whisper`, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe`, `gpt-4o-transcribe-diarize`, `gpt-4o-mini-tts`), `kimi-k2.5-fast` (under maintenance), frontier IDs (`openai/gpt-5.4*`, `anthropic/claude-*`, `google/gemini-*`, `x-ai/grok-4.20`, `qwen/qwen3.5-*`, `mistralai/mistral-small-2603`, `openrouter/auto`), and paid image models (`flux-2-pro`, `flux-1.1-pro`, `nano-banana-2`, `nano-banana-pro`). The free-tier auto-router is `auto` (not `openrouter/auto`) and IS available on the free plan. See [Full Model Catalog](/docs/models#full-catalog) for all 56 models.

## Error Handling

When a free-plan user calls a paid-only model, the API returns:

```json
{
  "error": {
    "message": "Model 'openai/gpt-5.4' requires a paid plan. See GET /api/v1/models/access for the full list of free-plan models. Upgrade at https://app.callmissed.com/pricing",
    "type": "invalid_request_error",
    "code": "model_not_available"
  }
}
```

HTTP status: `403`. Check the `code` field for `model_not_available` to handle this programmatically.