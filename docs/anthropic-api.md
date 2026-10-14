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
- `POST /v1/messages/count_tokens` — token count estimation (real BPE, not char-based); also at `/anthropic/v1/messages/count_tokens`
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
    "output_tokens": 12,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 0
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
| `model` | string | Yes | Model ID (e.g. `gpt-5.6-sol`, `sarvam-105b`, `kimi-k2.6`) |
| `max_tokens` | integer | Yes | Maximum tokens to generate, `1`–`1,048,576` |
| `messages` | array | Yes | 1–2,000 `{role, content}` objects. `content` is a string or an array of `text`, `image` (`base64` or `url` source), `document` (`text` source), `tool_use` and `tool_result` blocks. A `tool_result` may hold text, images, text documents and search results, and `is_error`. OpenAI-style `image_url`, `input_audio` and `file` parts are also accepted. Any other block or source type returns `400` naming it |
| `system` | string or array | No | System prompt (top-level, not in messages); a string or a list of text blocks |
| `stream` | boolean | No | Enable streaming (default: false) |
| `temperature` | number | No | Sampling temperature, `0`–`1` |
| `top_p` | number | No | Nucleus sampling, `0`–`1` |
| `top_k` | integer | No | Top-K sampling, `1`–`500` |
| `stop_sequences` | array | No | Up to 16 stop sequences |
| `tools` | array | No | Up to 128 `{name, description, input_schema}` tools. Server tools (for example `web_search_20250305`) return `400` |
| `thinking` | object | No | `{"type": "disabled"}` turns reasoning off (`reasoning_effort: "none"`). `enabled` and `adaptive` keep the model's default reasoning; `budget_tokens` is not applied |
| `output_config` | object | No | `effort` (`low`, `medium`, `high`, `xhigh`, `max`) sets the model's `reasoning_effort` and takes precedence over `thinking`. Each model keeps only the levels it supports. `format` (`{"type": "json_schema", "schema": {...}}`) requests structured JSON output |
| `tool_choice` | object | No | `{"type": "auto"}`, `{"type": "any"}`, `{"type": "none"}` or `{"type": "tool", "name": "..."}` |
| `metadata` | object | No | Up to 16 string, number or boolean values. `user_id` is your end user's id and enforces that user's [monthly budget](/docs/gateway-controls). `trace_id` and `session_id` are recorded on the usage row so you can filter [usage logs](/docs/usage-api) by them |
| `provider` | object | No | CallMissed extension: `{"zdr": true}` requires a [zero-data-retention route](/docs/gateway-controls) |

> **Note:** Unlike the OpenAI API, `max_tokens` is **required** and `system` is a **top-level parameter** (not a message with `role: "system"`).

## Model Selection

Send any model ID from the [Models](/docs/models) catalog — not just Anthropic-shaped names. The `model` field takes the same values as `/v1/chat/completions`, including legacy aliases of a catalog model.

```json
{ "model": "gpt-5.6-sol", "max_tokens": 1024, "messages": [...] }
```

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

Image blocks and PDF `document` blocks contribute a fixed estimate (~1,120
tokens each) rather than a fetch-and-resize pass, never their base64 size; text
`document` blocks count their text. Tool definitions are counted against the
total by serializing each to JSON and tokenizing the schema.

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
      "pricing": {"input": 5.208, "output": 31.25, "unit": "per_million_tokens", "currency": "USD"},
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

Send images on any model whose `supports_vision` flag is `true` in the model
listing. That is the authoritative source; see the
[vision list](/docs/chat-completion#vision-image-input) for the current set.
Models without vision reject image content with `400 invalid_request_error`
before the upstream call — you are not charged.

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
| `invalid_request_error` | 400 / 422 | Bad request, image sent to a text-only model, a voice-only model id, a prompt longer than the model's context window, or `max_tokens` below the model's minimum |
| `authentication_error` | 401 | Bad, missing, revoked or expired API key |
| `billing_error` | 402 | Insufficient credits, key budget or end-user budget exhausted |
| `permission_error` | 403 | Account inactive, key lacks the `llm` permission, or a free-plan key calling a paid model |
| `not_found_error` | 404 | Model not found |
| `request_too_large` | 413 | Request body too large |
| `rate_limit_error` | 429 | Plan limit or API key rate limit exceeded |
| `api_error` | 500 / 503 | Upstream model failure, or the model is under maintenance (the message names an alternative) |
| `overloaded_error` | 503 / 529 | Model temporarily unavailable. Retry with backoff |
| `timeout_error` | 503 | Upstream model timed out |

**Rate limit headers** are returned in Anthropic format:

```
anthropic-ratelimit-requests-limit: 60
anthropic-ratelimit-requests-remaining: 45
anthropic-ratelimit-requests-reset: 2026-05-01T00:00:00+00:00
```

## Prompt caching

Models that support prompt caching reuse repeated prompt prefixes automatically. Usage uses
Anthropic's field names, with the same meaning:

- `input_tokens` — prompt tokens that were not read from or written to the cache
- `cache_read_input_tokens` — prompt tokens served from the cache (billed at the model's cached-input rate)
- `cache_creation_input_tokens` — prompt tokens written to the cache

Total prompt = `input_tokens + cache_read_input_tokens + cache_creation_input_tokens`.
When streaming, the final counts arrive in the `message_delta` event.

`cache_control` blocks (on `system`, message content or `tools`) are accepted
so Anthropic SDK code runs unchanged, but explicit breakpoints and `ttl` are not
applied today — caching works from the prompt prefix automatically.

## Differences from Anthropic

This endpoint is designed to work with the Anthropic SDK out of the box. Key differences from the official Anthropic API:

- **`anthropic-version` header** is accepted but not required
- **Model routing** — requests can target any model in the CallMissed catalogue, not just Anthropic-shaped names.
- **Token counting** uses a BPE tokenizer approximation (tiktoken `cl100k_base`). Expect ~5-10% variance from Anthropic's native counts on English prompts; larger on CJK and heavy-punctuation text.
- **Tools** are supported — `tools` and `tool_choice` work as documented, and `tool_use`/`tool_result` content blocks are preserved. Send each `tool_use.id` back exactly as you received it: on some models it carries state the next turn needs.
- **Vision** is supported on models whose `supports_vision` flag is `true`. Image content sent to text-only models is rejected with a `400 invalid_request_error` before the upstream call, so your credits are safe.
- **Message Batches API** (`/v1/messages/batches`) is not implemented — use the regular `/v1/messages` endpoint.
- **Claude models** (`claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`, Pro plan and up) return their reasoning as standard `thinking` and `redacted_thinking` content blocks. Pass those blocks back unchanged in later turns, especially alongside `tool_result` blocks. `temperature`, `top_p` and `top_k` are ignored for these models, `tool_choice` forcing runs as `auto` on `claude-opus-5-5` and `claude-sonnet-5-5`, and the conversation must end with a `user` message (a trailing assistant prefill turn is dropped).
- **Billing** uses CallMissed credits, not Anthropic billing.
