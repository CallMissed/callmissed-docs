---
title: "Chat Completion"
description: "Generate text responses using our OpenAI-compatible chat completion API."
slug: "chat-completion"
breadcrumb: "LLM & AI"
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
icon:provider | Provider | Run inference on the best-fit backend — picked from the model id
icon:gateway | CallMissed gateway | Stream tokens back and deduct credits when the response completes
icon:done | Your app | Receive the completion (all at once, or token-by-token when streaming)
:::

> **Tip:** The model id decides routing automatically — you never pick a backend. See [How CallMissed Works](/docs/how-it-works).

## Make your first request

:::steps
## Get an API key

Create a key in the [dashboard](https://console.callmissed.com/developer/keys) (**Developer → API keys**). It looks like `cm_xxxx…` and is shown once.

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
    model="sarvam-105b",
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
  model: "sarvam-105b",
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
    "model": "sarvam-105b",
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
| `model` | string | Model ID (e.g. `sarvam-105b`, `gpt-5.6-luna`). Defaults to `sarvam-105b` when omitted |
| `messages` | array | **Required.** 1–2,000 `{role, content}` objects. System prompt goes here as `{"role": "system", "content": "..."}`. `content` may be a string or an array of text / image parts (see [Vision](#vision-image-input)) |
| `stream` | boolean | Enable streaming SSE responses (default `false`) |
| `stream_options` | object | `{"include_usage": true}` to get token counts in the stream |
| `temperature` | number | Sampling temperature, `0`–`2` |
| `max_tokens` | integer | Maximum tokens to generate, `1`–`1,048,576` |
| `max_completion_tokens` | integer | Same as `max_tokens` (the newer OpenAI name); used when `max_tokens` is absent |
| `n` | integer | Number of completions, `1`–`5` (default `1`) |
| `top_p` | number | Nucleus sampling, `0`–`1` |
| `top_k` | integer | Top-K sampling, `1`–`200` |
| `frequency_penalty` | number | Penalize repeated tokens, `-2`–`2` |
| `presence_penalty` | number | Penalize new topics, `-2`–`2` |
| `repetition_penalty` | number | Reduce repetition, `0`–`2` |
| `seed` | integer | Deterministic sampling, `0`–`2^63-1` |
| `stop` | string or array | Up to 16 stop sequences |
| `logit_bias` | object | Token id → bias between `-100` and `100`; at most 1,024 entries |
| `logprobs` | boolean | Return log probabilities |
| `top_logprobs` | integer | Top N log probs per token, `0`–`20` |
| `tools` | array | Up to 128 function definitions — see [Function Calling](/docs/chat-function-calling) |
| `tool_choice` | string or object | `"auto"`, `"none"`, `"required"`, or `{"type": "function", "function": {"name": "..."}}` |
| `parallel_tool_calls` | boolean | Allow parallel function calls |
| `response_format` | object | `{"type": "json_object"}` or `{"type": "json_schema", "json_schema": {...}}` |
| `structured_outputs` | boolean | Enforce strict JSON schema |
| `reasoning_effort` | string | `"none"` / `"minimal"` / `"low"` / `"medium"` / `"high"` / `"xhigh"` — each model accepts a different subset and the gateway maps the rest; see the [per-model matrix](/docs/api-speed#3-reasoning-effort-by-model) |
| `models` | array | Up to 32 fallback model ids, tried in order if `model` fails with a retryable error — see [Model substitution](#model-substitution) |
| `user` | string | Your end user's id. Enforces that user's [monthly budget](/docs/gateway-controls) when one is set (caps apply to ids of at most 256 characters) |
| `provider` | object | `{"zdr": true}` serves the request only on a [zero-data-retention route](/docs/gateway-controls), or fails with `zdr_unavailable` |
| `prompt_cache_key` | string | Up to 1,024 characters. Reuse the same key for requests that share a long prompt prefix to raise the cache hit rate — see [Prompt caching](#prompt-caching) |
| `bot_id` | string | ID of one of your [bots](/docs/bots). Its knowledge base is searched with the latest user message and the top matches are added to the prompt — see [Knowledge](/docs/knowledge) |
| `knowledge_top_k` | integer | With `bot_id`: how many knowledge chunks to add, `1`–`50` (default `6`) |
| `knowledge_min_score` | number | With `bot_id`: minimum similarity score, `0`–`1` (default `0`) |

On GPT-5 and GPT-6 family models, `temperature`, `top_p` and `logit_bias` are not supported by the model and are left out of the request; the model runs at its default sampling.

> **OpenAI Python SDK note** — The OpenAI client validates kwargs against its
> known parameters, so a CallMissed-specific field such as `reasoning_effort`
> raises `TypeError: Completions.create() got an unexpected keyword argument`.
> Pass it via `extra_body` instead:
>
> ```python
> client.chat.completions.create(
>     model="kimi-k2.6",
>     messages=[...],
>     extra_body={"reasoning_effort": "none"},
> )
> ```
>
> Raw HTTP / curl users can keep it at the top level — only the OpenAI SDK gates kwargs.

## Model Substitution

CallMissed never substitutes your model on its own. Send a `model` and you get
that model, or a clean error (`429`/`503` with `Retry-After`).

To opt in to failover, list fallbacks yourself in `models`:

```json
{
  "model": "kimi-k2.6",
  "models": ["kimi-k2.5", "gpt-oss-120b"],
  "messages": [{"role": "user", "content": "Hello"}]
}
```

If `model` fails with a retryable error (an upstream outage or rate limit), the
next id in `models` is tried. Each fallback must pass the same checks as
`model` — your plan, the key's `allowed_models`, maintenance status, vision
support and context window — and ids that fail them are skipped. The response's
`model` field names the model that actually answered, and you are billed at
that model's rate. Requests that send `tools`, `tool_choice`,
`response_format` or `structured_outputs` never fall back, because a different
model could change the result shape.

Need a model that is not in the catalog? See
[Models on demand](/docs/models#models-on-demand).

## Vision (Image Input)

Multimodal content (text + image parts) is accepted on any model whose
`supports_vision` flag is `true` in `GET /v1/models`. Models without vision
support reject image content with `400 unsupported_image_input` **before** the
upstream call, so you're not charged.

```python
from openai import OpenAI

client = OpenAI(api_key="cm_your_key", base_url="https://api.callmissed.com/v1")

resp = client.chat.completions.create(
    model="gpt-5.6-sol",   # supports_vision: true
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "What is in this image?"},
            {"type": "image_url", "image_url": {"url": "https://example.com/cat.png"}},
        ],
    }],
)
```

Vision-capable models: `gpt-6.1-sol`, `gpt-6-sol`, `gpt-6-luna`, `gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`,
`gpt-5.5`, `gpt-4o`, `gpt-4.1`, `gpt-5-mini`, `grok-4.3`, `gemini-3.8-flash`,
`gemini-3.7-flash`, `gemini-3.6-flash`, `gemini-3.5-flash`, `gemini-3.5-flash-lite`,
`gemini-3.1-pro-preview`, `gemini-3.1-flash-lite`, `gemma-4-31b`, `gemma-4-26b-a4b-it`,
`kimi-k2.5`, `kimi-k2.5-fast`, `kimi-k2.6`, `kimi-k2.7-code`, `mistral-small-3.1`.

`GET /v1/models` is authoritative. Read `supports_vision` there rather than
hard-coding this list.

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

Snapshot — `GET /v1/models` is authoritative:

| Model | context_window |
|-------|----------------|
| `gpt-5.5`, `gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`, `gpt-6-sol`, `gpt-6-luna`, `gpt-6.1-sol` | 1,050,000 |
| `DeepSeek-V4-Pro`, `DeepSeek-V4-Flash`, `glm-5.3` | 1,048,576 |
| `gpt-4.1` | 1,047,576 |
| `gpt-5-mini` | 400,000 |
| `kimi-k2.6`, `kimi-k2.7-code`, `glm-5.2` | 262,144 |
| `kimi-k2.5`, `kimi-k2.5-fast`, `nemotron-3-super`, `gemma-4-26b-a4b-it` | 256,000 |
| `grok-4.3` | 200,000 |
| `sarvam-105b`, `glm-4.7-flash`, `gemma-4-31b` | 131,072 |
| `gpt-4o`, `gpt-oss-120b`, `mistral-small-3.1` | 128,000 |
| `sarvam-105b-conversations` | 32,768 |

## Prompt caching

Models that support prompt caching reuse repeated prompt prefixes
automatically — there is nothing to turn on. Cached prompt tokens are billed at
the model's cached-input rate where one is published (see [Models](/docs/models));
a model with no cached rate bills them at its normal input rate. Tokens written
to the cache are billed at the input rate, except on models that publish a
separate cache-write rate (such as `gpt-6.1-sol`).

Every response reports the cached share of the prompt:

```json
"usage": {
  "prompt_tokens": 3120,
  "completion_tokens": 42,
  "total_tokens": 3162,
  "prompt_tokens_details": { "cached_tokens": 2944 }
}
```

- `prompt_tokens` is the whole prompt, cached part included.
- `prompt_tokens_details.cached_tokens` is always present (`0` on a miss or the first request).
- `prompt_tokens_details.cache_write_tokens` appears when the model reported tokens written to the cache (GPT-5.6 and later bill these at their own rate).
- Streaming: the same block is in the final usage chunk when you send `stream_options: {"include_usage": true}`.
- `/v1/responses` reports the same numbers as `usage.input_tokens_details.cached_tokens` and `usage.input_tokens_details.cache_write_tokens`.

To improve hit rates:

- Put stable content first (system prompt, tool definitions, reference documents) and the changing part last.
- Send a `prompt_cache_key` (up to 1,024 characters, on `/v1/chat/completions` and `/v1/responses`) and reuse it for requests that share a prefix. It is passed on as a cache-routing hint for `kimi-k2.5`, `kimi-k2.6`, `kimi-k2.7-code`, `glm-4.7-flash`, `glm-5.2`, `gpt-oss-120b`, `nemotron-3-super`, `gemma-4-26b-a4b-it`, `mistral-small-3.1`, `DeepSeek-V4-Pro` and `DeepSeek-V4-Flash`; other models ignore it.
- Explicit per-block cache breakpoints (`cache_control` / `prompt_cache_breakpoint` on a content part) are not applied today — caching works from the prompt prefix automatically.

```json
{
  "model": "kimi-k2.6",
  "prompt_cache_key": "support-bot:policy-v3",
  "messages": [
    {"role": "system", "content": "<long, stable instructions>"},
    {"role": "user", "content": "Where is my order?"}
  ]
}
```

A prefix usually needs to be at least ~1,024 tokens before it is cached, and an
idle cache expires after a few minutes. Hits are not guaranteed.

## Responses API

For clients built on OpenAI's newer **Responses API**, CallMissed exposes a compatible `POST /v1/responses` endpoint. It accepts a Responses-shaped body and translates to the same chat engine under the hood — so you can point an OpenAI Responses client at `https://api.callmissed.com/v1` without changes.

**Endpoint:** `POST /v1/responses`

:::tabs
```python [Python]
from openai import OpenAI

client = OpenAI(api_key="cm_your_key", base_url="https://api.callmissed.com/v1")

resp = client.responses.create(
    model="gpt-4.1",
    input="Write a haiku about databases.",
)
print(resp.output_text)
```
```bash [cURL]
curl https://api.callmissed.com/v1/responses \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4.1",
    "input": "Write a haiku about databases."
  }'
```
:::

| Field | Type | Notes |
|-------|------|-------|
| `model` | string | Any chat model id. Defaults to `kimi-k2.5` when omitted |
| `input` | string or array | A plain string, or the Responses item array (messages, `function_call` and `function_call_output` items) |
| `instructions` | string | System prompt |
| `max_output_tokens` | integer | `1`–`1,048,576` |
| `temperature` / `top_p` | number | `0`–`2` / `0`–`1` |
| `tools` | array | Flat function tools (`{"type": "function", "name", "description", "parameters"}`). Any other tool type returns `400 unsupported_tool_type` |
| `tool_choice`, `parallel_tool_calls` | | As in the Responses API |
| `reasoning` | object | `{"effort": "..."}` — mapped like `reasoning_effort` on chat completions |
| `text` | object | `{"format": {"type": "json_object" \| "json_schema", ...}}` for structured output |
| `stream` | boolean | Emits Responses-style SSE events (`response.output_text.delta`, …) |
| `user`, `provider`, `prompt_cache_key` | | Same meaning as on `/v1/chat/completions` |

- The same models, pricing, plan rules, vision and tool-calling support as `/v1/chat/completions` apply — this is a request/response-shape adapter, not a different model set.
- Responses are not stored: `store` and `previous_response_id` are accepted for compatibility but have no effect, so send the full conversation in `input` each turn.

If you're starting fresh, `/v1/chat/completions` is the most widely-supported surface; use `/v1/responses` when porting an existing Responses-API integration.

## Errors

All errors return the OpenAI-compatible envelope:

```json
{
  "error": {
    "message": "Invalid API key",
    "type": "invalid_request_error",
    "code": "invalid_api_key"
  }
}
```

| Status | `code` | When |
|--------|--------|------|
| `400` | `unsupported_image_input` | Image content sent to a model without vision support |
| `400` | `voice_agent_only_model` | A realtime or managed-voice model id — use `POST /v1/voice/sessions` |
| `400` | `zdr_unavailable` | Zero data retention is on and the model has no zero-retention route |
| `400` | `context_length_exceeded` | The prompt is longer than the model's context window |
| `400` | `max_tokens_too_small` | `max_tokens` below 3 on `gpt-5.6-luna`, `gpt-6-sol`, `gpt-6-luna` or `gpt-6.1-sol` |
| `401` | `invalid_api_key` / `api_key_expired` | Missing, malformed or revoked key / expired key |
| `402` | `insufficient_credits` | Balance exhausted (`X-Credits-Balance` carries the balance) |
| `402` | `budget_exceeded` | The key's own budget cap was reached |
| `402` | `end_user_budget_exceeded` | The `user` id has used its monthly budget |
| `403` | `permission_denied` | The key lacks the `llm` permission |
| `403` | `account_inactive` | The account is suspended or closed |
| `403` | `model_not_available` | A free-plan key calling a paid model |
| `403` | `model_not_allowed` | The key's `allowed_models` list excludes the model |
| `404` | `model_not_found` | Unknown model id |
| `422` | — | Request body failed validation (a field out of range, a missing `messages`). Body is `{"detail": [{loc, msg, type}]}` |
| `429` | `quota_exceeded` | Monthly plan call cap reached |
| `429` | `rate_limit_exceeded` | Per-key requests-per-minute exceeded |
| `429` | `too_many_concurrent_requests` | Too many requests in flight on this key. Honour `Retry-After` |
| `503` | `model_under_maintenance` | The model is temporarily unavailable; the message names an alternative |
| `503` | `provider_error` | The model is temporarily unavailable upstream. Safe to retry |

Every response carries an `X-Request-ID` header — quote it when you contact support.
