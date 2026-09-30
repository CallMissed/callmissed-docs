---
title: "Batch API"
description: "Run thousands of chat completions asynchronously from one JSONL file — OpenAI Batch API compatible."
slug: "batch"
breadcrumb: "LLM & AI"
---

# Batch API

Run thousands of chat completions asynchronously from one JSONL file — OpenAI Batch API compatible.

## Overview

The Batch API runs a file of chat-completion requests in the background and hands you the results as a file. Use it for work that does not need an answer right now: classifying a backlog, summarising transcripts overnight, generating evaluation sets.

It is compatible with the OpenAI Batch API. An OpenAI SDK pointed at `https://api.callmissed.com/v1` works unchanged.

| Endpoint | Purpose |
| --- | --- |
| `POST /v1/files` | Upload a JSONL input file (`purpose=batch`) |
| `POST /v1/batches` | Start a batch from an uploaded file |
| `GET /v1/batches/{batch_id}` | Poll status and progress |
| `GET /v1/files/{file_id}/content` | Download the input, output or error file |
| `POST /v1/batches/{batch_id}/cancel` | Stop a running batch |
| `GET /v1/batches` | List batches (newest first) |
| `GET /v1/files`, `GET /v1/files/{file_id}`, `DELETE /v1/files/{file_id}` | List, inspect and delete files |

Each line runs exactly like a live `POST /v1/chat/completions` call made with the same API key: the same models, free-plan model rules, `allowed_models`, key budget, plan limits, response cache and bring-your-own-key routing apply. Streaming is the only thing a batch line cannot do.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

The key needs the **LLM** service permission (the same one chat completions needs). Creating a batch requires an API key, because every line later runs as that key. If the key is revoked or expires while the batch runs, the remaining lines fail with `api_key_revoked` or `api_key_expired`.

## 1. Write the input file

One JSON object per line. `custom_id` is yours and must be unique within the file; `body` is a normal chat-completions request.

```json
{"custom_id": "ticket-1041", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "sarvam-105b", "messages": [{"role": "user", "content": "Classify: my order never arrived"}], "max_tokens": 20}}
{"custom_id": "ticket-1042", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "sarvam-105b", "messages": [{"role": "user", "content": "Classify: how do I change my plan?"}], "max_tokens": 20}}
```

| Field | Rule |
| --- | --- |
| `custom_id` | String, 1–512 characters, unique in the file |
| `method` | `"POST"` |
| `url` | `"/v1/chat/completions"` |
| `body` | A valid chat-completions request with `model` set. `stream` must be absent or `false` |

The whole file is validated on upload. An invalid line rejects the upload with `400 invalid_batch_file` and a message naming the line, so nothing is half-accepted.

## 2. Upload it

```bash
curl https://api.callmissed.com/v1/files \
  -H "Authorization: Bearer cm_your_api_key" \
  -F purpose=batch \
  -F file=@requests.jsonl
```

```json
{
  "id": "file-6f1c2e0a9b8d4c3e8f7a6b5c4d3e2f10",
  "object": "file",
  "bytes": 412,
  "created_at": 1790000000,
  "expires_at": 1792592000,
  "filename": "requests.jsonl",
  "purpose": "batch",
  "status": "processed"
}
```

## 3. Create the batch

```bash
curl https://api.callmissed.com/v1/batches \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "input_file_id": "file-6f1c2e0a9b8d4c3e8f7a6b5c4d3e2f10",
    "endpoint": "/v1/chat/completions",
    "completion_window": "24h",
    "metadata": {"job": "nightly-triage"}
  }'
```

| Parameter | Type | Notes |
| --- | --- | --- |
| `input_file_id` | `string` | A file uploaded with `purpose=batch` |
| `endpoint` | `string` | `/v1/chat/completions` (the only supported endpoint) |
| `completion_window` | `string` | `24h` (the only supported window) |
| `metadata` | `object` | Optional. Up to 16 string pairs; keys ≤ 64 characters, values ≤ 512 |

Before the batch starts, every model in the file is checked against your plan and key. A model the key can never call is rejected here (`403 model_not_available` / `model_not_allowed`) rather than failing line by line. Creating a batch also needs a positive credit balance (`402 insufficient_credits`).

## 4. Poll for completion

```python
import time
from openai import OpenAI

client = OpenAI(base_url="https://api.callmissed.com/v1", api_key="cm_your_api_key")

batch_file = client.files.create(file=open("requests.jsonl", "rb"), purpose="batch")
batch = client.batches.create(
    input_file_id=batch_file.id,
    endpoint="/v1/chat/completions",
    completion_window="24h",
)
while batch.status not in ("completed", "failed", "expired", "cancelled"):
    time.sleep(30)
    batch = client.batches.retrieve(batch.id)

print(batch.request_counts)
if batch.output_file_id:
    print(client.files.content(batch.output_file_id).text)
if batch.error_file_id:
    print(client.files.content(batch.error_file_id).text)
```

### Batch object

```json
{
  "id": "batch_0b9e1f3a5c7d4e2f8a6b4c2d0e8f6a4b",
  "object": "batch",
  "endpoint": "/v1/chat/completions",
  "errors": null,
  "input_file_id": "file-6f1c2e0a9b8d4c3e8f7a6b5c4d3e2f10",
  "completion_window": "24h",
  "status": "completed",
  "output_file_id": "file-1a2b3c4d5e6f47a8b9c0d1e2f3a4b5c6",
  "error_file_id": null,
  "created_at": 1790000000,
  "in_progress_at": 1790000000,
  "expires_at": 1790086400,
  "finalizing_at": 1790000420,
  "completed_at": 1790000421,
  "failed_at": null,
  "expired_at": null,
  "cancelling_at": null,
  "cancelled_at": null,
  "request_counts": {"total": 2, "completed": 2, "failed": 0},
  "metadata": {"job": "nightly-triage"}
}
```

| Status | Meaning |
| --- | --- |
| `in_progress` | Lines are running. `request_counts` updates live |
| `finalizing` | All lines are done; the result files are being written |
| `completed` | Results are ready in `output_file_id` / `error_file_id` |
| `cancelling` | Cancel requested; lines already running finish, the rest are skipped |
| `cancelled` | Stopped. Lines that finished before the cancel are in the output file |
| `expired` | The 24-hour window ran out. Finished lines are in the output file; the rest are in the error file with `batch_expired` |

`validating` and `failed` exist for OpenAI compatibility; CallMissed validates the file when you upload it, so a new batch starts directly in `in_progress`.

## 5. Read the results

The output file holds one line per successful request; the error file holds one line per failed request. Match lines to your input with `custom_id` — do not rely on line order.

```json
{"id": "batch_req_3c5e…", "custom_id": "ticket-1041", "response": {"status_code": 200, "request_id": "chatcmpl-…", "body": {"id": "chatcmpl-…", "object": "chat.completion", "model": "sarvam-105b", "choices": [ … ], "usage": { … }}}, "error": null}
```

An error-file line carries an `error` object. When the request reached the model and was refused (for example a context-length error), `response` holds that response too; when the line never ran, `response` is `null`.

```json
{"id": "batch_req_9a1b…", "custom_id": "ticket-1042", "response": null, "error": {"code": "batch_cancelled", "message": "This request was not executed because the batch was cancelled."}}
```

| `error.code` | Meaning |
| --- | --- |
| Any chat-completions error code | The line ran and got that error (e.g. `context_length_exceeded`, `model_not_allowed`) |
| `insufficient_credits` | Your balance ran out. Every line not yet run fails with this code |
| `budget_exceeded` | The key's budget cap was reached. Every line not yet run fails with this code |
| `api_key_revoked` / `api_key_expired` | The creating key stopped being usable |
| `account_inactive` / `account_terminated` | The account can no longer make requests |
| `batch_cancelled` | Skipped because you cancelled the batch |
| `batch_expired` | Not run before the 24-hour window ended |
| `server_error` | The line kept failing after retries |

Temporary failures (upstream errors, concurrency limits) are retried automatically, up to three attempts per line, before they are written to the error file.

## Cancel

```bash
curl -X POST https://api.callmissed.com/v1/batches/batch_0b9e…/cancel \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns the batch in `cancelling`; it moves to `cancelled` once lines already running finish. Cancelling a batch that has already finished returns `409 batch_not_cancellable`.

## List

`GET /v1/batches?limit=20&after=batch_…` returns `{object, data, first_id, last_id, has_more}`, newest first. `limit` is 1–100; pass the previous page's `last_id` as `after` for the next page.

## Files

| Call | Returns |
| --- | --- |
| `GET /v1/files?purpose=batch&limit=20` | Your files, newest first (`purpose` is `batch` for uploads, `batch_output` for results) |
| `GET /v1/files/{file_id}` | The file object |
| `GET /v1/files/{file_id}/content` | The raw JSONL |
| `DELETE /v1/files/{file_id}` | `{"id", "object": "file", "deleted": true}` — deletes it now instead of at expiry |

Every file — the ones you upload and the result files — is deleted automatically 30 days after it was created (`expires_at`). Download results you want to keep.

## Pricing

Batch lines are billed at the **same per-token rate as a live `/v1/chat/completions` call** for the model that served them, and each line appears in your usage logs with `batch_id` and `custom_id` in its metadata. Lines answered from the response cache or served on your own provider key cost what they would live. Lines that fail are not charged.

## Limits

| Limit | Value |
| --- | --- |
| Requests per file | 10,000 |
| File size | 20 MB |
| Running batches per account | 5 (`429 too_many_active_batches` above that) |
| Completion window | 24 hours |
| File retention | 30 days |

Lines of one account run a few at a time so a large batch does not crowd out your live traffic. Most batches finish well inside the window; the 24 hours is an upper bound, not a target.

## Errors

| Status | Code | When |
| --- | --- | --- |
| `400` | `invalid_batch_file` | The upload is not valid batch JSONL (the message names the line) |
| `400` | `invalid_purpose` | `purpose` is not `batch` |
| `400` | `unsupported_endpoint` / `invalid_completion_window` / `invalid_metadata` | Bad create parameters |
| `400` | `zdr_unavailable` | [Zero data retention](/docs/gateway-controls) is on for the account or key, or a line sends `provider.zdr: true`. A batch stores its input and output files, so it cannot run under ZDR |
| `402` | `insufficient_credits` | No credit balance at create time |
| `403` | `permission_denied` | The key lacks the LLM permission |
| `403` | `api_key_required` | Batch create called without an API key |
| `403` | `model_not_available` / `model_not_allowed` | A model in the file is not available to this key |
| `404` | `file_not_found` / `batch_not_found` | Unknown id, or it belongs to another account |
| `409` | `batch_not_cancellable` | The batch already finished |
| `413` | `file_too_large` | Upload over 20 MB |
| `429` | `too_many_active_batches` | Five batches already running |
| `503` | `storage_unavailable` | File storage is temporarily unavailable — retry |
