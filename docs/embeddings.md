---
title: "Embeddings"
description: "Turn text into vectors with the OpenAI-compatible embeddings endpoint — batching, dimensions, base64 output, and per-token pricing."
slug: "embeddings"
breadcrumb: "LLM & AI"
---

# Embeddings

Turn text into vectors with the OpenAI-compatible embeddings endpoint — batching, dimensions, base64 output, and per-token pricing.

## Overview

`POST /v1/embeddings` converts text into a dense float vector you can store in your own vector database and search with cosine similarity. It is the primitive behind retrieval, semantic search, clustering, deduplication and classification.

The request and response are **OpenAI-compatible**, so the official OpenAI SDKs work unchanged once you point them at `https://api.callmissed.com/v1` with a `cm_` key.

> If you want retrieval without running your own vector store, use [Knowledge & RAG](/docs/knowledge) instead — it ingests, chunks, embeds and searches for you.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

This endpoint is gated by the key's **service permission**, not by a resource scope. The key needs `llm` (or `*`). There is no separate `embedding` permission — a key that can call `/v1/chat/completions` can call `/v1/embeddings`.

A key without it returns `403`:

```json
{
  "error": {
    "message": "This API key does not have permission for embeddings (requires LLM permission). Update key permissions in your dashboard.",
    "type": "invalid_request_error",
    "code": "permission_denied"
  }
}
```

## Models

Both embedding models are **free-plan callable** — they are metered per input token, so your credit balance is the only governor.

| Model | Dimensions | Max input | Price (per 1M input tokens) |
| --- | --- | --- | --- |
| `text-embedding-3-small` | 1536 | 8,192 tokens | $0.02 |
| `text-embedding-3-large` | 3072 | 8,192 tokens | $0.13 |

Start with `text-embedding-3-small`: it is the better price/performance choice for large corpora. Move to `-large` only when you have measured that retrieval quality is the bottleneck.

Both appear in `GET /v1/models` with `"owned_by": "openai"`.

## Quickstart

```bash
curl https://api.callmissed.com/v1/embeddings \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "text-embedding-3-small",
    "input": "Where is my order?"
  }'
```

```json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "index": 0,
      "embedding": [0.0023064255, -0.009327292, 0.015797347]
    }
  ],
  "model": "text-embedding-3-small",
  "usage": {
    "prompt_tokens": 5,
    "total_tokens": 5
  }
}
```

```python
from openai import OpenAI

client = OpenAI(
    api_key="cm_your_api_key",
    base_url="https://api.callmissed.com/v1",
)

resp = client.embeddings.create(
    model="text-embedding-3-small",
    input=["Where is my order?", "How do I return this?"],
)
vectors = [row.embedding for row in resp.data]
```

## Request

| Field | Type | Required | Default | Constraints |
| --- | --- | --- | --- | --- |
| `model` | `string` | Yes | — | `text-embedding-3-small` or `text-embedding-3-large` |
| `input` | `string` or `string[]` | Yes | — | Up to **128 items** per request; each item non-empty, at most **100,000 characters** and within the model's 8,192-token limit. Pre-tokenised integer arrays are **not** accepted |
| `encoding_format` | `string` | No | `float` | `float` or `base64` |
| `dimensions` | `integer` | No | model native | `1 <= dimensions <= 3072` (large) or `1536` (small) |
| `user` | `string` | No | — | At most 256 characters. An opaque end-user identifier for your own abuse tracing |

### Batching

Send an array to embed up to 128 strings in one round trip. The `index` on each row matches the position in your `input` array, so you can zip the results back onto your records without re-ordering.

```json
{
  "model": "text-embedding-3-small",
  "input": ["first chunk", "second chunk", "third chunk"]
}
```

A batch of 129 or more returns `422`:

```json
{ "error": { "message": "`input` array too long: 200 items (maximum 128). Split the batch across multiple requests.", "type": "invalid_request_error", "code": "invalid_request_error" } }
```

### Shortening vectors with `dimensions`

Both models support Matryoshka-style truncation. Passing `dimensions` returns a shorter, renormalised vector — smaller index, faster search, slightly lower recall.

```json
{ "model": "text-embedding-3-large", "input": "hello", "dimensions": 256 }
```

`dimensions` must be between `1` and the model's native size. Anything else returns `422`.

> Vectors of different lengths are not comparable. Pick one model **and** one `dimensions` value per index and keep it fixed — re-embed the whole corpus if you change either.

### `encoding_format: "base64"`

`base64` returns each vector as a base64 string of little-endian `float32` values instead of a JSON array. It is roughly a third of the payload size, which matters when you are embedding thousands of chunks.

```python
import base64, struct

raw = base64.b64decode(resp.data[0].embedding)
vector = list(struct.unpack(f"<{len(raw) // 4}f", raw))
```

## Response

| Field | Type | Notes |
| --- | --- | --- |
| `object` | `string` | Always `list` |
| `data[].object` | `string` | Always `embedding` |
| `data[].index` | `integer` | Position in your `input` array |
| `data[].embedding` | `number[]` or `string` | Float array, or a base64 string when `encoding_format` is `base64` |
| `model` | `string` | The model that served the request |
| `usage.prompt_tokens` | `integer` | Input tokens billed |
| `usage.total_tokens` | `integer` | Same as `prompt_tokens` — embeddings have no output tokens |

## Billing

Embeddings are metered on **input tokens only**. Credits are deducted as `tokens / 1,000,000 x rate`, where 1 credit = $0.01.

- `text-embedding-3-small` — 2 credits per 1M input tokens
- `text-embedding-3-large` — 13 credits per 1M input tokens

A request that fails with a `4xx` or `5xx` is recorded in your usage log but **not charged**. Track spend with [`GET /v1/usage/summary`](/docs/usage-api).

## Errors

| Status | `code` | When |
| --- | --- | --- |
| `400` | `invalid_request_error` | Body is not a JSON object |
| `400` | `context_length_exceeded` | An input item exceeds the model's 8,192-token limit |
| `401` | `invalid_api_key` / `api_key_expired` | Missing, malformed, or expired key |
| `402` | `insufficient_credits` | Balance is exhausted. The `X-Credits-Balance` header carries the current balance |
| `402` | `budget_exceeded` | The key's own budget cap was hit |
| `403` | `permission_denied` | Key lacks the `llm` permission |
| `403` | `model_not_allowed` | The key's `allowed_models` list excludes this model |
| `404` | `model_not_found` | Unknown embedding model id |
| `413` | `invalid_request_error` | An input item is over 100,000 characters |
| `422` | `invalid_request_error` | Batch too long, empty input, bad `dimensions`, unsupported `encoding_format`, or a token array instead of a string |
| `429` | `rate_limit_exceeded` | Per-key request rate exceeded. Retry with backoff |
| `429` | `quota_exceeded` | Plan or monthly budget cap reached. Honour `Retry-After` |
| `502` | `upstream_error` | Embedding generation failed. Safe to retry |
| `503` | `service_unavailable` | Temporary capacity problem. Retry with backoff |

Every error uses the standard envelope:

```json
{ "error": { "message": "…", "type": "invalid_request_error", "code": "model_not_found", "request_id": "req_…" } }
```

## Building a search index

1. Chunk your documents to roughly 200–500 tokens with a little overlap.
2. Embed chunks in batches of 128 with `text-embedding-3-small`.
3. Store `{id, text, vector, metadata}` in your vector database.
4. At query time, embed the query with the **same model and `dimensions`**, then retrieve by cosine similarity.
5. Pass the top chunks to [`POST /v1/chat/completions`](/docs/chat-completion) as context.

```python
query = client.embeddings.create(
    model="text-embedding-3-small",
    input="refund policy",
).data[0].embedding
# hits = your_vector_db.search(query, top_k=5)
```
