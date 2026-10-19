---
title: "Model Access by Plan"
description: "See which models each plan tier (Free, Starter, Pro, Enterprise) can access, with cURL, Python, and JavaScript examples."
slug: "model-access"
breadcrumb: "LLM & AI"
---

# Model Access by Plan

See which models each plan tier (Free, Starter, Pro, Enterprise) can access, with cURL, Python, and JavaScript examples.

## Overview

The Model Access endpoint returns which model IDs each plan tier can call, bucketed by category (LLM, STT, TTS, Image, Embedding). This is useful for:

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
      "models": ["sarvam-105b", "sarvam-105b-conversations", "kimi-k2.5", "kimi-k2.6", "kimi-k2.7-code", "gpt-oss-120b", "saaras:v4", "bulbul:v3", "bulbul:v4-flash", "text-embedding-3-small", "text-embedding-3-large"],
      "by_category": {
        "llm": ["sarvam-105b", "sarvam-105b-conversations", "kimi-k2.5", "kimi-k2.6", "kimi-k2.7-code", "gpt-oss-120b"],
        "stt": ["saaras:v4"],
        "tts": ["bulbul:v3", "bulbul:v4-flash"],
        "image": [],
        "embedding": ["text-embedding-3-small", "text-embedding-3-large"]
      },
      "restriction": "Only the models listed here, plus the retired LLM and speech IDs that still answer on the free plan (see /docs/models#retired-and-deprecated-models), are callable on the free plan. Other models require a paid plan; Claude models require Pro or higher."
    },
    "starter": {
      "models": ["...every model in the catalog"],
      "by_category": { "llm": ["..."], "stt": ["..."], "tts": ["..."], "image": ["..."], "embedding": ["..."] },
      "restriction": "All models callable except Claude models, which need Pro or higher. Per-service monthly call limits apply…"
    },
    "pro": {
      "models": ["..."],
      "by_category": { "llm": ["..."], "stt": ["..."], "tts": ["..."], "image": ["..."], "embedding": ["..."] },
      "restriction": "All models callable. Higher per-service monthly call limits than Starter."
    },
    "enterprise": {
      "models": ["..."],
      "by_category": { "llm": ["..."], "stt": ["..."], "tts": ["..."], "image": ["..."], "embedding": ["..."] },
      "restriction": "All models callable. Custom per-service call limits."
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
model = "gpt-6-luna"
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
const model = "gpt-6-luna";
const isFree = data.plans.free.models.includes(model);
console.log(model, "on free plan:", isFree); // false
```
:::

## Free Plan Models

The free tier includes **11 models**:

| Category | Models |
|----------|--------|
| LLM (6) | `sarvam-105b`, `sarvam-105b-conversations`, `kimi-k2.5`, `kimi-k2.6`, `kimi-k2.7-code`, `gpt-oss-120b` |
| STT (1) | `saaras:v4` |
| TTS (2) | `bulbul:v3`, `bulbul:v4-flash` |
| Image (0) | None. Image generation needs a paid plan. |
| Embedding (2) | `text-embedding-3-small`, `text-embedding-3-large` |

The retired LLM, speech-to-text and text-to-speech IDs (`glm-4.7-flash`, `glm-5.2`, `nemotron-3-super`, `gemma-4-26b-a4b-it`, `mistral-small-3.1`, `nova-3`, `whisper-large-v3-turbo`, `aura-2-en`, `aura-2-es`, `melotts`) still answer on the free plan. They are not listed. See [Retired and deprecated models](/docs/models#retired-and-deprecated-models).

`restriction` is a human-readable note — branch on `models` / `by_category`, not on this text.

Every other model — `kimi-k2.5-fast`, `glm-5.3`, `deepseek-v4.1-flash`, `gemma-4-31b`, the `gemini-*` chat models, the first-party OpenAI / xAI / DeepSeek IDs, the realtime voice models, the Deepgram direct line, the other paid speech models, and the paid image models — requires Starter, Pro, or Enterprise, except the Claude models, which need Pro or higher.

## Error Handling

When a free-plan user calls a paid model, the API returns:

```json
{
  "error": {
    "message": "Model 'gpt-5.6-luna' requires a paid plan. See GET /api/v1/models/access for the full list of free-plan models. Upgrade at https://console.callmissed.com/org/plans",
    "type": "invalid_request_error",
    "code": "model_not_available"
  }
}
```

HTTP status: `403`. The request is refused before any model runs, so it is not billed.

A key whose `allowed_models` list excludes the model gets `403` with `code: "model_not_allowed"` instead, on any plan.
