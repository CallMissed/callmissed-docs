---
title: "Anthropic-Compatible API"
description: "Use the Anthropic SDK with CallMissed — just change the base URL. Full Messages API compatibility."
slug: "anthropic-api"
breadcrumb: "LLM & AI"
---

# Anthropic-Compatible API

Use the Anthropic SDK with CallMissed — just change the base URL. Full Messages API compatibility.

## Overview

CallMissed provides an **Anthropic Messages API-compatible endpoint** alongside the OpenAI-compatible API. If you're already using the Anthropic SDK, you can switch to CallMissed by changing only the `base_url`.

**Endpoints:**
- `POST /v1/messages` — chat completions (streaming + non-streaming)
- `POST /v1/messages/count_tokens` — token count estimation (real BPE, not char-based)
- `GET  /anthropic/v1/models` — list models in Anthropic shape with capability metadata
- `GET  /anthropic/v1/models/{model_id}` — single model detail
- `POST /anthropic/v1/messages` — alternate path for the chat endpoint

**Authentication:** Use either header style:
- `x-api-key: cm_your_key` (Anthropic SDK default)
- `Authorization: Bearer cm_your_key` (OpenAI style)

## Basic Usage

:::tabs
```python [Python]
import anthropic

client = anthropic.Anthropic(
    api_key="cm_your_key",
    base_url="https://api.callmissed.com"
)

message = client.messages.create(
    model="gpt-5.6-sol",
    max_tokens=1024,
    system="You are a helpful assistant.",
    messages=[
        {"role": "user", "content": "What is the capital of India?"}
    ]
)

print(message.content[0].text)
```
```javascript [JavaScript]
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({
  apiKey: "cm_your_key",
  baseURL: "https://api.callmissed.com",
});

const message = await client.messages.create({
  model: "gpt-5.6-sol",
  max_tokens: 1024,
  system: "You are a helpful assistant.",
  messages: [
    { role: "user", content: "What is the capital of India?" },
  ],
});

console.log(message.content[0].text);
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/messages \
  -H "x-api-key: cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.6-sol",
    "max_tokens": 1024,
    "system": "You are a helpful assistant.",
    "messages": [
      {"role": "user", "content": "What is the capital of India?"}
    ]
  }'
```
:::

**Response:**

```json
{
  "id": "msg-abc123def456",
  "type": "message",
  "role": "assistant",
  "content": [
    {"type": "text", "text": "The capital of India is New Delhi."}
  ],
  "model": "gpt-5.6-sol",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 25,
    "output_tokens": 12
  }
}
```

## Streaming

Set `stream: true` to receive Server-Sent Events with the full Anthropic streaming lifecycle:

:::tabs
```python [Python]
import anthropic

client = anthropic.Anthropic(
    api_key="cm_your_key",
    base_url="https://api.callmissed.com"
)

with client.messages.stream(
    model="gpt-5.6-sol",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Tell me a short story."}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```
```javascript [JavaScript]
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({
  apiKey: "cm_your_key",
  baseURL: "https://api.callmissed.com",
});

const stream = client.messages.stream({
  model: "gpt-5.6-sol",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Tell me a short story." }],
});

for await (const event of stream) {
  if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
    process.stdout.write(event.delta.text);
  }
}
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/messages \
  -H "x-api-key: cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.6-sol",
    "max_tokens": 1024,
    "stream": true,
    "messages": [
      {"role": "user", "content": "Tell me a short story."}
    ]
  }'
```
:::

**SSE event lifecycle:**

```
event: message_start        → message metadata + input token count
event: content_block_start  → new content block begins
event: content_block_delta  → text chunks (repeats)
event: content_block_stop   → content block complete
event: message_delta        → stop_reason + output token count
event: message_stop          → stream complete
```

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `model` | string | Yes | Model ID (e.g. `gpt-5.6-sol`, `sarvam-105b`, `google/gemini-3.5-flash`) |
| `max_tokens` | integer | Yes | Maximum tokens to generate |
| `messages` | array | Yes | List of `{role, content}` objects |
| `system` | string | No | System prompt (top-level, not in messages) |
| `stream` | boolean | No | Enable streaming (default: false) |
| `temperature` | number | No | Sampling temperature (0–1) |
| `top_p` | float | No | Nucleus sampling (0–1) |
| `top_k` | integer | No | Top-K sampling |
| `stop_sequences` | array | No | Stop sequences |
| `metadata` | object | No | Request metadata (e.g. `{"user_id": "u123"}`) |

> **Note:** Unlike the OpenAI API, `max_tokens` is **required** and `system` is a **top-level parameter** (not a message with `role: "system"`).

## Model Aliasing

On the Anthropic endpoint, a bare model name resolves against the CallMissed catalog; an ID that already carries a provider prefix is routed as sent:

| You send | Routed as |
|----------|-----------|
| `gpt-5.6-sol` | `gpt-5.6-sol` (first-party model, no prefix) |
| `google/gemini-3.5-flash` | `google/gemini-3.5-flash` (already has prefix) |

You can use **any model** from our [Models](/docs/models) catalog — not just Anthropic models.

## Token Counting

Estimate input token count before sending a request. The endpoint uses a BPE
tokenizer (tiktoken `cl100k_base`) — close to Claude's real tokenizer on
typical English prompts, and noticeably more accurate than char-length
heuristics.

```bash
curl -X POST https://api.callmissed.com/v1/messages/count_tokens \
  -H "x-api-key: cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.6-sol",
    "messages": [{"role": "user", "content": "Hello, how are you?"}],
    "system": "You are a helpful assistant."
  }'
```

**Response:**

```json
{"input_tokens": 15}
```

Image content blocks contribute a fixed estimate (~258 tokens per image) rather
than a fetch-and-resize pass. Tool definitions are counted against the total by
serializing each to JSON and tokenizing the schema.

## Listing Models

List all available models via the Anthropic-shape endpoint:

```bash
curl https://api.callmissed.com/anthropic/v1/models \
  -H "x-api-key: cm_your_key"
```

**Response:**

```json
{
  "data": [
    {
      "type": "model",
      "id": "gpt-5.6-sol",
      "display_name": "GPT-5.6 Sol",
      "created_at": "2023-11-14T22:13:20+00:00",
      "description": "Frontier model for complex professional work. Multimodal, reasoning + tools.",
      "category": "llm",
      "context_window": 1050000,
      "context_length": 1050000,
      "pricing": {"input": 5.00, "output": 30.00, "unit": "per_million_tokens", "currency": "USD"},
      "supports_streaming": true,
      "supports_tools": true,
      "supports_reasoning": true,
      "supports_vision": true
    }
  ],
  "has_more": false,
  "first_id": "...",
  "last_id": "..."
}
```

Fetch a single model at `GET /anthropic/v1/models/{model_id}`.

## Vision (Image Input)

You can send images on any model whose `supports_vision` flag is `true` in
the model listing. Vision-capable models currently include the OpenAI GPT-5.6
family (Sol, Terra, Luna), GPT-5.5, GPT-4o / GPT-4.1, Google Gemini 3 / 3.1,
Moonshot Kimi K2.5 / K2.6 / K2.7 Code, Google Gemma 4 26B, and Mistral Small 3.1.
Models without vision support reject image content with a
`400 invalid_request_error` before the upstream call — you won't be charged.

```bash
curl -X POST https://api.callmissed.com/v1/messages \
  -H "x-api-key: cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.6-sol",
    "max_tokens": 1024,
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "What is in this image?"},
        {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": "<base64>"}}
      ]
    }]
  }'
```

## Error Format

Errors return the Anthropic format (different from the OpenAI endpoints):

```json
{
  "type": "error",
  "error": {
    "type": "authentication_error",
    "message": "Invalid API key"
  }
}
```

| Error type | HTTP Status | When |
|------------|-------------|------|
| `authentication_error` | 401 | Bad or missing API key |
| `permission_error` | 403 | Account inactive, domain blocked, or free tier model restriction |
| `invalid_request_error` | 400/402 | Bad request or insufficient credits |
| `rate_limit_error` | 429 | Plan limit or API key rate limit exceeded |
| `not_found_error` | 404 | Model not found |
| `api_error` | 502 | Provider failure |

**Rate limit headers** are returned in Anthropic format:

```
anthropic-ratelimit-requests-limit: 60
anthropic-ratelimit-requests-remaining: 45
anthropic-ratelimit-requests-reset: 2026-05-01T00:00:00+00:00
```

## Differences from Anthropic

This endpoint is designed to work with the Anthropic SDK out of the box. Key differences from the official Anthropic API:

- **`anthropic-version` header** is accepted but not required
- **Model routing** — requests can target *any* model in the CallMissed catalogue, using either a bare first-party ID or a slash-prefixed frontier ID.
- **Token counting** uses a BPE tokenizer approximation (tiktoken `cl100k_base`). Expect ~5-10% variance from Anthropic's native counts on English prompts; larger on CJK and heavy-punctuation text.
- **Tools** are supported — `tools` and `tool_choice` work as documented, and `tool_use`/`tool_result` content blocks are preserved.
- **Vision** is supported on models whose `supports_vision` flag is `true`. Image content sent to text-only models is rejected with a `400 invalid_request_error` before the upstream call, so your credits are safe.
- **Message Batches API** (`/v1/messages/batches`) is not implemented — use the regular `/v1/messages` endpoint.
- **Billing** uses CallMissed credits, not Anthropic billing.
