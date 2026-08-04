---
title: "Knowledge Base & RAG"
description: "Store content your bots use to answer questions, plus a vector Knowledge API that chunks, embeds, and semantically retrieves your sources for RAG."
slug: "knowledge"
breadcrumb: "Knowledge"
---

# Knowledge Base & RAG

Store content your bots use to answer questions, plus a vector Knowledge API that chunks, embeds, and semantically retrieves your sources for RAG.

## Overview

There are two independent layers of knowledge, and they do not share storage:

1. **Bot knowledge base** — a flat document store scoped to a single bot. Add plain text or upload PDF/DOCX/TXT (max 20 MB); text is extracted on upload. This layer is **not** searched by RAG.
2. **Knowledge API (RAG)** — a vector store. Ingest text, URLs, or PDFs as **sources**; each is chunked and embedded, then retrieved by semantic search and passed as context to the model.

Every source you ingest is attached to a bot via a required `bot_id`. Retrieval is always tenant-scoped, and additionally bot-scoped whenever you pass a `bot_id`.

> Every endpoint on both layers accepts either a `cm_` API key or a dashboard session (JWT). API keys need the `knowledge:read` / `knowledge:write` [scopes](/docs/keys); scopes do not apply to JWT callers, whose access is role-based.

**Base paths:** `https://api.callmissed.com/api/v1/knowledge` for RAG, and `https://api.callmissed.com/api/v1/bots/{bot_id}/knowledge` for the flat store.

### Which layer to use

| You want to… | Use |
| --- | --- |
| Ground an LLM reply in your documents | **Knowledge API (RAG)** |
| Run semantic search over your content | **Knowledge API (RAG)** |
| Keep a plain list of documents attached to a bot | **Bot knowledge base** |
| Upload DOCX, or a file up to 20 MB | **Bot knowledge base** |

New integrations should use the Knowledge API. The flat store predates it and is kept for existing bots — it still counts toward the storage total in your [billing](/docs/billing) usage rollup.

## Bot Knowledge Base

A flat, non-vector list of entries on one bot. The bot must belong to your tenant or the request returns `404`.

`GET /api/v1/bots/{bot_id}/knowledge` · scope `knowledge:read`

Returns every entry for the bot, newest first.

`POST /api/v1/bots/{bot_id}/knowledge` · scope `knowledge:write`

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string (1–255) | Yes | Label for the entry |
| `content` | string (1–100000) | Yes | The raw text |
| `metadata` | object | No | Arbitrary JSON you can read back |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/$BOT_ID/knowledge \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"name":"Refund policy","content":"Refunds are issued within 7 days of purchase."}'
```

**Response (201 Created)** — a knowledge entry:

```json
{
  "id": "7c1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f",
  "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
  "name": "Refund policy",
  "content": "Refunds are issued within 7 days of purchase.",
  "metadata": null,
  "file_url": null,
  "file_size_bytes": null,
  "format": null,
  "status": "indexed",
  "error_message": null,
  "created_at": "2026-07-20T09:15:00Z"
}
```

### Upload a document

`POST /api/v1/bots/{bot_id}/knowledge/upload` · scope `knowledge:write`

A multipart upload of a single `file`. Accepted extensions are **PDF, DOCX, and TXT**, up to **20 MB**. Text is extracted server-side and stored in `content`.

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/$BOT_ID/knowledge/upload \
  -H "Authorization: Bearer cm_your_key" \
  -F 'file=@handbook.pdf'
```

The entry comes back with `format` set to the extension, `file_size_bytes` set, and `status` either `indexed` (text extracted) or `failed` with an `error_message`. A scanned PDF with no text layer yields `failed` — there is no OCR.

### Delete an entry

`DELETE /api/v1/bots/{bot_id}/knowledge/{entry_id}` · scope `knowledge:write` · returns `204 No Content`

## Knowledge API (RAG)

### How ingestion works

Ingestion is **synchronous** — the response returns only once the source is fully indexed, so there is no job to poll:

:::flow
icon:file-text | Extract | Text is taken as-is, fetched from a URL, or pulled out of a PDF
icon:scissors | Chunk | Split into ~600-token chunks with a 100-token overlap
icon:database | Embed & store | Each chunk is embedded to a 768-dimension vector and indexed for cosine similarity
:::

Embedding tokens are billed against your [credits](/docs/credits-rate-limits). If your balance cannot cover the embedding, the source is saved with `status: "failed"` and an `error_message` saying so — top up and re-ingest.

### The source object

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | Source identifier |
| `tenant_id` | uuid | Owning tenant |
| `bot_id` | uuid | Bot the source is attached to |
| `kind` | string | `text`, `pdf`, or `url` |
| `title` | string \| null | Your label; defaults to the URL for URL sources |
| `uri` | string \| null | Source URL, or the uploaded filename for PDFs |
| `status` | string | `pending` → `ingesting` → `ready`, or `failed` |
| `error_message` | string \| null | Set when `status` is `failed` |
| `byte_size` | int \| null | Size of the extracted text in bytes |
| `token_count` | int \| null | Total tokens embedded |
| `chunk_count` | int | Number of retrievable chunks |
| `created_at` | datetime | |
| `ingested_at` | datetime \| null | Set when indexing completes |

All three ingest endpoints return the same envelope — the source plus a flat summary:

```json
{
  "source": { "id": "…", "kind": "text", "status": "ready", "chunk_count": 12, "…": "…" },
  "status": "ready",
  "chunk_count": 12,
  "token_count": 7043
}
```

### Ingest text

`POST /api/v1/knowledge/sources` · scope `knowledge:write` · returns `201`

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `bot_id` | uuid | Yes | Must be a bot in your tenant |
| `title` | string (1–512) | Yes | Label for the source |
| `content` | string | Yes | The text to index; max **5 MB** of UTF-8 |

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/knowledge/sources \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
    "title": "Shipping FAQ",
    "content": "We ship across India in 3-5 business days. Express delivery reaches metro cities next day."
  }'
```
```python [Python]
import httpx

resp = httpx.post(
    "https://api.callmissed.com/api/v1/knowledge/sources",
    headers={"Authorization": "Bearer cm_your_key"},
    json={
        "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
        "title": "Shipping FAQ",
        "content": "We ship across India in 3-5 business days.",
    },
)
result = resp.json()
print(result["status"], result["chunk_count"])  # "ready" 1
```
:::

### Ingest a URL

`POST /api/v1/knowledge/sources/url` · scope `knowledge:write` · returns `201`

The server fetches the page itself, strips HTML to text, and indexes the result.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `bot_id` | uuid | Yes | Must be a bot in your tenant |
| `url` | string | Yes | A bare domain works — `https://` is added when no scheme is present |
| `title` | string (≤512) | No | Defaults to the URL |

```bash
curl -X POST https://api.callmissed.com/api/v1/knowledge/sources/url \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"bot_id":"'$BOT_ID'","url":"acme.in/help/returns","title":"Returns policy"}'
```

> **Only public URLs.** Requests that resolve to private or internal address ranges are rejected with `400 URL rejected`. The fetch caps the download at **2 MB**, times out after **30 seconds**, and follows at most **5 redirects** — each hop is re-validated.

A page that yields no extractable text returns `400`, so JavaScript-rendered pages with no server-side HTML will not ingest.

### Ingest a PDF

`POST /api/v1/knowledge/sources/pdf` · scope `knowledge:write` · returns `201`

A multipart upload. Unlike the text and URL endpoints, the fields are **form fields**, not JSON.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `bot_id` | uuid (form) | Yes | Must be a bot in your tenant |
| `title` | string (form) | Yes | Label for the source |
| `file` | file | Yes | PDF only, max **5 MB** |

```bash
curl -X POST https://api.callmissed.com/api/v1/knowledge/sources/pdf \
  -H "Authorization: Bearer cm_your_key" \
  -F "bot_id=$BOT_ID" \
  -F 'title=Product catalogue' \
  -F 'file=@catalogue.pdf'
```

> A scanned PDF is an image, not text. With no text layer to extract, the upload returns `400` — there is no OCR in v1.

### List sources

`GET /api/v1/knowledge/sources` · scope `knowledge:read`

| Query | Type | Default | Notes |
|-------|------|---------|-------|
| `bot_id` | uuid | — | Filter to one bot; omit to list the whole tenant |
| `limit` | int (1–200) | `50` | Page size |
| `offset` | int (0–10000) | `0` | Page offset |

Returns `{ "items": [...], "total": 42 }`, newest first, where `total` is the count before paging.

`GET /api/v1/knowledge/sources/{source_id}` returns a single source, or `404` if it is not in your tenant.

### Delete a source

`DELETE /api/v1/knowledge/sources/{source_id}` · scope `knowledge:write` · returns `204 No Content`

Deleting a source also deletes all of its chunks, which removes it from retrieval immediately.

## Semantic search

`POST /api/v1/knowledge/search` · scope `knowledge:read`

Runs the retrieval step on its own. Use it to tune `k` and `min_score`, or to build your own RAG prompt.

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `query` | string (1–4096) | — | Required. The text to match against |
| `bot_id` | uuid | `null` | Restrict to one bot; omit to search every source in the tenant |
| `k` | int (1–50) | `6` | How many chunks to return |
| `min_score` | float (0–1) | `0.0` | Drop chunks below this score |

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/knowledge/search \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
    "query": "how long does delivery take?",
    "k": 4,
    "min_score": 0.6
  }'
```
```python [Python]
import httpx

resp = httpx.post(
    "https://api.callmissed.com/api/v1/knowledge/search",
    headers={"Authorization": "Bearer cm_your_key"},
    json={"query": "how long does delivery take?", "k": 4, "min_score": 0.6},
)
for chunk in resp.json()["chunks"]:
    print(round(chunk["score"], 3), chunk["content"][:80])
```
:::

**Response (200 OK)**

```json
{
  "query": "how long does delivery take?",
  "chunks": [
    {
      "id": "e4a1b2c3-d4e5-6f70-8192-a3b4c5d6e7f8",
      "source_id": "7c1d2e3f-4a5b-6c7d-8e9f-0a1b2c3d4e5f",
      "chunk_index": 0,
      "content": "We ship across India in 3-5 business days...",
      "token_count": 412,
      "score": 0.8317
    }
  ]
}
```

### Reading the score

`score` is cosine similarity — `1.0` is an exact semantic match and `0.0` is unrelated. Two details matter when tuning:

- `min_score` is applied **after** ranking, not before. You can get back fewer than `k` chunks, including zero.
- A default `min_score` of `0` returns the nearest chunks whether or not they are relevant. For question-answering, start around **`0.6`** to drop off-topic matches.

Searching a bot with no ingested chunks returns an empty list without spending credits. Otherwise each search embeds the query, which is billed as a small number of embedding tokens and appears in your [usage logs](/docs/audit-logs).

## Grounding a chat completion

You do not have to call `/search` and assemble a prompt yourself. Pass `bot_id` to [chat completions](/docs/chat-completion) and the server retrieves for you, prepending the matched chunks to the system message before the model runs.

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `bot_id` | string | — | Enables retrieval against that bot's sources |
| `knowledge_top_k` | int (1–50) | `6` | Chunks to retrieve |
| `knowledge_min_score` | float (0–1) | `0.0` | Minimum score to include |

```bash
curl -X POST https://api.callmissed.com/v1/chat/completions \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
    "knowledge_top_k": 4,
    "knowledge_min_score": 0.6,
    "messages": [{"role": "user", "content": "Do you deliver to Pune?"}]
  }'
```

Behaviour worth knowing:

- Retrieval runs against the **latest user message** in `messages`, not the system prompt or conversation history.
- Retrieved context is merged into your existing system message, or inserted as one if you did not send any.
- If retrieval fails, the completion still runs — just without context, rather than returning an error.
- A `bot_id` belonging to another tenant is treated as "no knowledge" instead of returning `404`.

Cost is one embedding call for the query plus the added context tokens, billed at your model's normal input rate.

## Limits

| | Bot knowledge base | Knowledge API (RAG) |
| --- | --- | --- |
| Storage | Raw documents | Chunks + vectors |
| Formats | PDF, DOCX, TXT, plain text | Plain text, PDF, URL |
| Max upload | 20 MB | 5 MB (2 MB for a URL fetch) |
| Max text per entry | 100,000 characters | 5 MB of UTF-8 |
| Chunking | None | ~600 tokens, 100-token overlap |
| Semantic search | No | Yes |
| Scoping | One bot | Ingest per bot; search per bot or tenant-wide |

## Status codes

| Code | Meaning |
|------|---------|
| `201` | Source ingested |
| `204` | Source or entry deleted |
| `400` | No extractable text, a non-PDF upload, an unsupported format, or a rejected URL |
| `403` | Key is missing the `knowledge:read` / `knowledge:write` scope |
| `404` | Bot or source not found in your tenant |
| `413` | Content or upload exceeds the size cap |
| `502` | Indexing failed after the text was extracted — safe to retry |

A `502` means extraction succeeded but embedding did not, so nothing was stored. Retry the same request. See [Errors](/docs/errors) for the standard error shape.
