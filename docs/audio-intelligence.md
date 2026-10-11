---
title: "Audio Intelligence"
description: "Summarize an audio file and detect its topics, sentiment and intents in one call."
slug: "audio-intelligence"
breadcrumb: "Speech"
---

# Audio Intelligence

Summarize an audio file and detect its topics, sentiment and intents in one call.

## Overview

Upload an English audio file and get back its transcript plus any combination of four analyses: a summary, topics, sentiment and intents.

**Endpoint:** `POST /v1/audio/intelligence`

| Feature | `features` value | Adds to the response |
|---------|------------------|----------------------|
| Summarization | `deepgram-summarize` | `summary` |
| Topic detection | `deepgram-topics` | `topics` |
| Sentiment analysis | `deepgram-sentiment` | `sentiments` |
| Intent recognition | `deepgram-intents` | `intents` |

English only. Audio Intelligence is available on the Starter, Pro and Enterprise plans, and the key needs the `stt` permission. To run the same analyses on text instead of audio, use `POST /v1/text/intelligence` — see [Models](/docs/models).

## Basic Usage

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/audio/intelligence \
  -H "Authorization: Bearer cm_your_key" \
  -F file=@call.wav \
  -F features=deepgram-summarize,deepgram-sentiment
```
```python [Python]
import requests

with open("call.wav", "rb") as f:
    resp = requests.post(
        "https://api.callmissed.com/v1/audio/intelligence",
        headers={"Authorization": "Bearer cm_your_key"},
        files={"file": f},
        data={"features": "deepgram-summarize,deepgram-sentiment"},
    )

result = resp.json()
print(result["summary"])
```
```javascript [JavaScript]
import fs from "fs";

const form = new FormData();
form.append("file", new Blob([fs.readFileSync("call.wav")]), "call.wav");
form.append("features", "deepgram-summarize,deepgram-sentiment");

const resp = await fetch("https://api.callmissed.com/v1/audio/intelligence", {
  method: "POST",
  headers: { Authorization: "Bearer cm_your_key" },
  body: form,
});
console.log(await resp.json());
```
:::

## Parameters

Send as `multipart/form-data`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file` | file | Yes | Audio file (WAV, MP3, etc.) |
| `features` | string | No | Comma-separated list of the `features` values above. Default `deepgram-summarize` |
| `model` | string | No | Transcription model used for the analysis. Default `nova-3`. Also accepts other Deepgram model names such as `nova-2`, `nova-2-phonecall`, `nova-2-meeting` or `enhanced` (the `deepgram-*` STT IDs without the `deepgram-` prefix) |

## Response

```json
{
  "request_id": "dgai-3f9c1a2b4d5e6f70",
  "features": ["deepgram-summarize", "deepgram-sentiment"],
  "transcript": "Hi, I'm calling about my order...",
  "summary": {
    "result": "success",
    "short": "The customer called to ask about a delayed order."
  },
  "sentiments": {
    "segments": [
      {"text": "Hi, I'm calling about my order...", "start_word": 0, "end_word": 7, "sentiment": "neutral", "sentiment_score": 0.1}
    ],
    "average": {"sentiment": "neutral", "sentiment_score": 0.05}
  },
  "metadata": {
    "summary_info": {"input_tokens": 120, "output_tokens": 28},
    "sentiment_info": {"input_tokens": 120, "output_tokens": 0}
  }
}
```

| Field | Description |
|-------|-------------|
| `request_id` | ID for this request |
| `features` | The features that ran |
| `transcript` | Transcript of the audio |
| `summary` / `topics` / `sentiments` / `intents` | Present only for the features you requested |
| `metadata` | Per-feature token counts (`*_info`), the basis of the charge |

## Pricing

Billed per token, summed across the features you request: $0.0003125 per 1K input tokens plus $0.000625 per 1K output tokens. See [Models](/docs/models).

## Errors

Errors use the OpenAI envelope: `{"error": {"message", "type", "code"}}`.

| Status | `code` | When |
|--------|--------|------|
| `400` | `invalid_request_error` | `features` is empty or contains an unknown value. The message lists the valid values |
| `402` | `insufficient_credits` | Credit balance is exhausted |
| `403` | `permission_denied` | The key does not have the `stt` permission, or a feature is not available on your plan |
| `404` | `model_not_found` | `model` is not an accepted model name |
| `429` | `quota_exceeded` | Plan usage limit reached |
| `503` | `provider_error`, `upstream_timeout`, `upstream_unavailable` | The analysis failed. The response includes a `request_id` |
