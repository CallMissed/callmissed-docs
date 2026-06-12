---
title: "Chat Completion"
description: "Generate text responses using our OpenAI-compatible chat completion API."
slug: "chat-completion"
breadcrumb: "API Guides & Tutorials"
---

# Chat Completion

Generate text responses using our OpenAI-compatible chat completion API.

:::cards
/docs/chat-streaming | Streaming | play | Server-sent events for real-time responses
/docs/chat-function-calling | Function Calling | settings | Tool use with structured outputs
/docs/models | Model Catalog | boxes | Pick the right model for your workload
/docs/anthropic-api | Anthropic API | file-text | Messages API compatible endpoint
:::

## Overview

The Chat Completion API generates AI responses given a list of messages. It's fully OpenAI-compatible — use the same SDK and request format.

**Endpoint:** `POST /v1/chat/completions`

### How a request flows

Every chat completion takes the same path through the platform — your app never talks to the underlying provider directly:

:::flow
icon:app | Your app | Send `POST /v1/chat/completions` with `model` + `messages`
icon:gateway | CallMissed gateway | Authenticate the `cm_` key, check credits, route by model id
icon:provider | Provider | Run inference on Clarifai, Sarvam, or OpenRouter — picked from the model id
icon:gateway | CallMissed gateway | Stream tokens back and deduct credits when the response completes
icon:done | Your app | Receive the completion (all at once, or token-by-token when streaming)
:::

> **Tip:** The model id decides the provider automatically — no slash → Sarvam, `clarifai/…` → Clarifai, `vendor/model` → OpenRouter. See [How CallMissed Works](/docs/how-it-works).

## Make your first request

:::steps
## Get an API key

Create a key in the [dashboard](https://app.callmissed.com) (**Profile → API Keys**). It looks like `cm_xxxx…` and is shown once.

## Point your SDK at CallMissed

Set the base URL to `https://api.callmissed.com/v1` and pass your `cm_` key. No other change to your OpenAI code.

## Send messages and read the reply

Call `chat.completions.create` with a `model` and a `messages` array. Read `response.choices[0].message.content`.
:::

:::capabilities
### Basic completion
Send a single-turn or multi-turn conversation and receive a complete response. Use any OpenAI SDK — set `base_url` to `https://api.callmissed.com/v1` and `api_key` to your `cm_` key.

### Streaming
Set `stream: true` to receive tokens as they're generated. See [Streaming](/docs/chat-streaming) for full examples.

### Function calling
Pass a `tools` array to let the model call your functions. See [Function Calling](/docs/chat-function-calling).
:::

## Basic Usage

:::tabs
```python [Python]
from openai import OpenAI

client = OpenAI(
    api_key="cm_your_key",
    base_url="https://api.callmissed.com/v1"
)

response = client.chat.completions.create(
    model="sarvam-30b",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is the capital of India?"}
    ]
)

print(response.choices[0].message.content)
```
```javascript [JavaScript]
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "cm_your_key",
  baseURL: "https://api.callmissed.com/v1",
});

const response = await client.chat.completions.create({
  model: "sarvam-30b",
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "What is the capital of India?" },
  ],
});

console.log(response.choices[0].message.content);
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/chat/completions \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sarvam-30b",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is the capital of India?"}
    ]
  }'
```
:::

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | Model ID (e.g. `sarvam-30b`, `openai/gpt-5.4-mini`) |
| `messages` | array | List of `{role, content}` objects. System prompt goes here as `{"role": "system", "content": "..."}` |
| `stream` | boolean | Enable streaming SSE responses |
| `temperature` | number | Sampling temperature (0–2) |
| `max_tokens` | integer | Maximum tokens to generate |
| `n` | integer | Number of completions to generate (default 1) |
| `top_p` | float | Nucleus sampling (0–1) |
| `top_k` | integer | Top-K sampling |
| `frequency_penalty` | float | Penalize repeated tokens (−2 to 2) |
| `presence_penalty` | float | Penalize new topics (−2 to 2) |
| `repetition_penalty` | float | Reduce repetition (0–2) |
| `seed` | integer | Deterministic sampling |
| `stop` | array | Stop sequences |
| `logit_bias` | object | Token probability adjustments |
| `logprobs` | boolean | Return log probabilities |
| `top_logprobs` | integer | Top N log probs per token |
| `tools` | array | Tool/function definitions for function calling |
| `parallel_tool_calls` | boolean | Allow parallel function calls |
| `response_format` | object | `{"type": "json_object"}` or `{"type": "json_schema", "json_schema": {...}}` |
| `structured_outputs` | boolean | Enforce strict JSON schema |
| `stream_options` | object | `{"include_usage": true}` to get token counts in stream |
| `reasoning_effort` | string | `"none"` / `"minimal"` / `"low"` / `"medium"` / `"high"` — see the per-model matrix below |

## Frontier Parameters

When using slash-prefixed frontier models, these additional parameters are supported:

| Parameter | Type | Description |
|-----------|------|-------------|
| `provider` | object | Provider routing preferences (`sort`, `order`, `only`, `ignore`, `max_price`) |
| `models` | array | Fallback model list |
| `route` | string | `"fallback"` |
| `plugins` | array | `[{"id": "web"}]` for web search, `"file-parser"`, `"response-healing"`, `"context-compression"` |
| `reasoning` | object | `{"effort": "high", "max_tokens": 5000}` |
| `transforms` | array | `["middle-out"]` for context compression |

> **OpenAI Python SDK note** — The OpenAI client validates kwargs against its known parameters and rejects `provider=...` (and the others above) with `TypeError: Completions.create() got an unexpected keyword argument 'provider'`. Pass them via `extra_body` instead:
>
> ```python
> client.chat.completions.create(
>     model="openai/gpt-5.4",
>     messages=[...],
>     extra_body={
>         "provider": {"sort": "throughput", "order": ["OpenAI", "Azure"]},
>         "models": ["openai/gpt-5.4-mini"],
>     },
> )
> ```
>
> Raw HTTP / curl users can keep `provider` at the top level — only the OpenAI SDK gates kwargs.

## Vision (Image Input)

Multimodal content (text + image parts) is accepted on any model whose
`supports_vision` flag is `true` in `GET /v1/models`. Models without vision
support reject image content with `400 unsupported_image_input` **before** the
upstream call, so you're not charged.

```python
from openai import OpenAI

client = OpenAI(api_key="cm_your_key", base_url="https://api.callmissed.com/v1")

resp = client.chat.completions.create(
    model="anthropic/claude-sonnet-4.6",   # supports_vision: true
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "What is in this image?"},
            {"type": "image_url", "image_url": {"url": "https://example.com/cat.png"}},
        ],
    }],
)
```

Current vision-capable models: `openai/gpt-5.4-pro`, `openai/gpt-5.4`,
`openai/gpt-5.4-mini`, `openai/gpt-5.4-nano`, `anthropic/claude-opus-4.6`,
`anthropic/claude-sonnet-4.6`, `anthropic/claude-haiku-4.5`,
`google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`, `google/gemini-3.1-flash-lite`,
`x-ai/grok-4.20`, `qwen/qwen3.5-plus`, `qwen/qwen3.5-flash`, `kimi-k2.5`,
`kimi-k2.6`, `gemma-4-26b-a4b-it`, `mistral-small-3.1`,
`mistralai/mistral-small-2603`, `auto` (free plan), `openrouter/auto`.
Check the live `GET /v1/models` response for the authoritative list — it's
computed from the same set the runtime guard uses.

## Context Window

Every model in the catalog advertises a `context_window` (token count for the
combined prompt + completion). The `GET /v1/models` response exposes it under
two keys for cross-client compatibility:

- `context_window` (OpenAI/CallMissed canonical name)
- `context_length` (OpenAI SDK convention — same value)

```python
from openai import OpenAI

client = OpenAI(api_key="cm_your_key", base_url="https://api.callmissed.com/v1")

for m in client.models.list():
    extra = m.model_extra or {}
    print(m.id, extra.get("context_window"), extra.get("supports_vision"))
```

Representative context windows (treat `GET /v1/models` as the authoritative
source — the table below is a snapshot):

| Model | context_window |
|-------|----------------|
| `openai/gpt-5.4`, `openai/gpt-5.4-pro`, `openai/gpt-5.4-mini`, `openai/gpt-5.4-nano` | 1,048,576 |
| `anthropic/claude-opus-4.6`, `anthropic/claude-sonnet-4.6` | 1,048,576 |
| `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`, `google/gemini-3.1-flash-lite` | 1,048,576 |
| `nemotron-3-super` | 1,048,576 |
| `x-ai/grok-4.20` | 262,144 |
| `qwen/qwen3.5-plus`, `qwen/qwen3.5-flash` | 262,144 |
| `kimi-k2.5`, `kimi-k2.5-fast`, `kimi-k2.6` | 262,144 |
| `sarvam-105b`, `gpt-oss-120b`, `glm-4.7-flash`, `gemma-4-26b-a4b-it`, `mistralai/mistral-small-2603` | 131,072 |
| `sarvam-30b` | 65,536 |

## Error Format

All errors return OpenAI-compatible format:

```json
{
  "error": {
    "message": "Invalid API key",
    "type": "invalid_request_error",
    "code": "invalid_api_key"
  }
}
```