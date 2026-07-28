---
title: "Frontier Models"
description: "300+ frontier LLM models from OpenAI, Anthropic, Google, and more."
slug: "models-frontier"
breadcrumb: "Models"
---

# Frontier Models

300+ frontier LLM models from OpenAI, Anthropic, Google, and more.

## Overview

All frontier models are accessible via `POST /v1/chat/completions` using the model's full ID (e.g. `openai/gpt-5.4`).

| Model ID | Provider | Context |
|----------|----------|---------|
| `openai/gpt-5.4` | OpenAI | 1M |
| `openai/gpt-5.4-mini` | OpenAI | 400K |
| `anthropic/claude-sonnet-4.6` | Anthropic | 1M |
| `anthropic/claude-opus-4.6` | Anthropic | 1M |
| `google/gemini-3.1-pro-preview` *(maintenance)* | Google | 1M |
| `google/gemini-3-flash-preview` *(maintenance)* | Google | 1M |
| `google/gemini-3.5-flash` *(maintenance)* | Google | 1M |
| `google/gemini-3.1-flash-lite` *(maintenance)* | Google | 1M |
| `x-ai/grok-4.20` | xAI | 256K |
| `qwen/qwen3.5-plus` | Qwen | 262K |
| `moonshotai/kimi-k2` | Moonshot | 128K |
| `auto` | Auto Router | — |

## Auto Router

`auto` automatically selects the best model for your prompt:

```python
response = client.chat.completions.create(
    model="auto",
    messages=[{"role": "user", "content": "Your prompt here"}]
)
```

## Provider Routing

Control which upstream serves your request via the `provider` parameter (raw HTTP body):

```json
{
  "model": "openai/gpt-5.4",
  "messages": [...],
  "provider": {
    "sort": "price",
    "order": ["openai"],
    "only": ["OpenAI"],
    "ignore": [],
    "max_price": {"input": 5, "output": 15}
  }
}
```

> **OpenAI Python SDK** rejects unknown kwargs (`TypeError: Completions.create() got an unexpected keyword argument 'provider'`). Pass these fields under `extra_body` instead:
>
> ```python
> client.chat.completions.create(
>     model="openai/gpt-5.4",
>     messages=[...],
>     extra_body={"provider": {"sort": "throughput", "order": ["openai"]}},
> )
> ```

| Field | Values | Description |
|-------|--------|-------------|
| `sort` | `"price"` | `"latency"` | `"throughput"` | Optimization priority |
| `order` | array of provider names | Preferred provider order |
| `only` | array | Restrict to these providers only |
| `ignore` | array | Exclude these providers |
| `max_price` | `{input, output}` | Max price per 1M tokens |

## Plugins

Extend model capabilities with plugins:

```json
{
  "plugins": [
    {"id": "web"},
    {"id": "file-parser"},
    {"id": "response-healing"},
    {"id": "context-compression"}
  ]
}
```

| Plugin | Description |
|--------|-------------|
| `web` | Real-time web search |
| `file-parser` | Parse PDFs and documents |
| `response-healing` | Auto-repair malformed JSON responses |
| `context-compression` | Compress long contexts to fit within limits |

## Model Fallbacks

CallMissed never silently substitutes your model. If you send only `model` and its upstream is briefly unavailable, you get that model or a clean error (`429`/`503` with `Retry-After`) — never a different model (and a different price) you didn't choose.

Fallback is **opt-in and caller-controlled**: supply a `models` array and CallMissed tries them in order on a retryable failure (`429`/`5xx`/transport), billing the model that actually served. A non-retryable `4xx` (bad request, content filter) stops the chain immediately — it would fail identically on every model. Each fallback must be allowed by your key and plan; tool-calling and `response_format`/`structured_outputs` requests are never auto-switched (a swap would change the behavior you pinned).

```json
{
  "model": "gpt-5.6-sol",
  "models": ["gpt-5.6-sol", "gpt-5.6-terra", "kimi-k2.6"]
}
```

## Model Shortcuts

Append suffixes to any model ID for routing hints:

| Suffix | Description |
|--------|-------------|
| `:nitro` | Throughput priority — fastest response |
| `:floor` | Lowest price available |

Example: `openai/gpt-5.4:nitro`, `anthropic/claude-sonnet-4.6:floor`