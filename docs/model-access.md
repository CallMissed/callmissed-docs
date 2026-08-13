---
title: "Model Access by Plan"
description: "See which models each plan tier (Free, Starter, Pro, Enterprise) can access, with cURL, Python, and JavaScript examples."
slug: "model-access"
breadcrumb: "LLM & AI"
---

# Model Access by Plan

See which models each plan tier (Free, Starter, Pro, Enterprise) can access, with cURL, Python, and JavaScript examples.

## Overview

The Model Access endpoint returns which model IDs each plan tier can call, bucketed by category (LLM, STT, TTS, Image). This is useful for:

- Showing users which models they can access on their current plan
- Building model selectors that grey out unavailable models
- Checking whether a specific model requires an upgrade

**No authentication required.**

## Endpoint

```
GET /api/v1/models/access
```

## Response Shape

```json
{
  "plans": {
    "free": {
      "models": ["sarvam-105b", "sarvam-105b-conversations", "kimi-k2.5", "kimi-k2.6", "kimi-k2.7-code", "glm-4.7-flash", "glm-5.2", "gpt-oss-120b", "nemotron-3-super", "gemma-4-26b-a4b-it", "mistral-small-3.1", "saaras:v3", "saaras:v4", "whisper-large-v3-turbo", "nova-3", "bulbul:v3", "aura-2-en", "aura-2-es", "melotts", "flux-2-klein-9b", "flux-2-dev", "lucid-origin", "phoenix-1.0", "sdxl-lightning", "dreamshaper-8-lcm"],
      "by_category": {
        "llm": ["sarvam-105b", "sarvam-105b-conversations", "kimi-k2.5", "kimi-k2.6", "kimi-k2.7-code", "glm-4.7-flash", "glm-5.2", "gpt-oss-120b", "nemotron-3-super", "gemma-4-26b-a4b-it", "mistral-small-3.1"],
        "stt": ["saaras:v3", "saaras:v4", "whisper-large-v3-turbo", "nova-3"],
        "tts": ["bulbul:v3", "aura-2-en", "aura-2-es", "melotts"],
        "image": ["flux-2-klein-9b", "flux-2-dev", "lucid-origin", "phoenix-1.0", "sdxl-lightning", "dreamshaper-8-lcm"]
      },
      "restriction": "25 models across 4 categories"
    },
    "starter": {
      "models": ["...every model in the catalog"],
      "by_category": { "llm": ["..."], "stt": ["..."], "tts": ["..."], "image": ["..."] },
      "restriction": "All models"
    },
    "pro": {
      "models": ["..."],
      "by_category": { "llm": ["..."], "stt": ["..."], "tts": ["..."], "image": ["..."] },
      "restriction": "All models"
    },
    "enterprise": {
      "models": ["..."],
      "by_category": { "llm": ["..."], "stt": ["..."], "tts": ["..."], "image": ["..."] },
      "restriction": "All models + models deployed on demand"
    }
  }
}
```

## Code Examples

:::tabs
```bash [cURL]
curl https://api.callmissed.com/api/v1/models/access
```
```python [Python]
import requests

resp = requests.get("https://api.callmissed.com/api/v1/models/access")
data = resp.json()

# List free LLM models
free_llms = data["plans"]["free"]["by_category"]["llm"]
print("Free LLM models:", free_llms)

# Check if a model is available on free plan
model = "gpt-5.6-luna"
is_free = model in data["plans"]["free"]["models"]
print(f"{model} on free plan: {is_free}")  # False
```
```typescript [JavaScript / TypeScript]
const resp = await fetch("https://api.callmissed.com/api/v1/models/access");
const data = await resp.json();

// List free LLM models
const freeLLMs = data.plans.free.by_category.llm;
console.log("Free LLM models:", freeLLMs);

// Check if a model is available on free plan
const model = "gpt-5.6-luna";
const isFree = data.plans.free.models.includes(model);
console.log(model, "on free plan:", isFree); // false
```
:::

## Free Plan Models

The free tier includes **25 models**:

| Category | Models |
|----------|--------|
| LLM (11) | `sarvam-105b`, `sarvam-105b-conversations`, `kimi-k2.5`, `kimi-k2.6`, `kimi-k2.7-code`, `glm-4.7-flash`, `glm-5.2`, `gpt-oss-120b`, `nemotron-3-super`, `gemma-4-26b-a4b-it`, `mistral-small-3.1` |
| STT (4) | `saaras:v3`, `saaras:v4`, `whisper-large-v3-turbo`, `nova-3` |
| TTS (4) | `bulbul:v3`, `aura-2-en`, `aura-2-es`, `melotts` |
| Image (6) | `flux-2-klein-9b`, `flux-2-dev`, `lucid-origin`, `phoenix-1.0`, `sdxl-lightning`, `dreamshaper-8-lcm` |

Every other model — `kimi-k2.5-fast`, the first-party OpenAI / xAI / DeepSeek IDs, the realtime voice models, the Deepgram direct line, and the paid image models — requires Starter, Pro, or Enterprise.

## Error Handling

When a free-plan user calls a paid model, the API returns:

```json
{
  "error": {
    "message": "Model 'gpt-5.6-luna' requires a paid plan. See GET /api/v1/models/access for the full list of free-plan models. Upgrade at https://app.callmissed.com/pricing",
    "type": "invalid_request_error",
    "code": "model_not_available"
  }
}
```

HTTP status: `403`
