---
title: "Batch STT"
description: "Batch speech-to-text transcription with speaker diarization for call analytics."
slug: "stt-batch"
breadcrumb: "Speech to Text"
---

# Batch STT

Batch speech-to-text transcription with speaker diarization for call analytics.

## Overview

Batch STT with **speaker diarization** is available through the Call Analytics API. Upload audio files and receive:

- Diarized transcript (SPEAKER_1, SPEAKER_2, etc.)
- Per-speaker timing breakdown
- LLM-powered analysis across 9 dimensions
- Structured summaries

**Supported formats:** WAV, MP3, MP4, M4A, OGG, FLAC, WebM, AAC, AMR (max 100 MB)

## Analyze a Recording

```
POST /api/v1/analytics/calls/analyze
Authorization: Bearer <access_token>
Content-Type: multipart/form-data
```

:::tabs
```python [Python]
import requests

response = requests.post(
    "https://api.callmissed.com/api/v1/analytics/calls/analyze",
    headers={"Authorization": "Bearer YOUR_ACCESS_TOKEN"},
    files={"file": open("call_recording.wav", "rb")},
    data={"language": "hi-IN"},
)

result = response.json()
print(result["transcript"])
print("Speaker timing:", result["speaker_timing"])
print("Analysis:", result["analysis"])
```
```javascript [JavaScript]
const formData = new FormData();
formData.append("file", fs.createReadStream("call_recording.wav"));
formData.append("language", "hi-IN");

const response = await fetch(
  "https://api.callmissed.com/api/v1/analytics/calls/analyze",
  {
    method: "POST",
    headers: { Authorization: "Bearer YOUR_ACCESS_TOKEN" },
    body: formData,
  }
);

const result = await response.json();
console.log(result.transcript);
console.log("Speaker timing:", result.speaker_timing);
```
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/analytics/calls/analyze \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -F file=@call_recording.wav \
  -F language=hi-IN
```
:::

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | file | Yes | Audio file (max 100 MB) |
| `language` | string | No | BCP-47 language code (default: `unknown` for auto-detect) |

## Ask Questions

After analyzing a call, ask follow-up questions about the transcript:

:::tabs
```python [Python]
response = requests.post(
    "https://api.callmissed.com/api/v1/analytics/calls/question",
    headers={
        "Authorization": "Bearer YOUR_ACCESS_TOKEN",
        "Content-Type": "application/json",
    },
    json={
        "transcript": result["transcript"],
        "question": "Did the customer agree to the terms?"
    },
)

print(response.json()["answer"])
```
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/analytics/calls/question \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "transcript": "SPEAKER_1: Hello...\nSPEAKER_2: Hi...",
    "question": "Did the customer agree to the terms?"
  }'
```
:::

## Response Format

```json
{
  "transcript": "SPEAKER_1: Namaste, main aapki kaise madad kar sakta hoon?\nSPEAKER_2: Mujhe apne order ke baare mein jaanna hai.",
  "speaker_timing": {
    "SPEAKER_1": 45.2,
    "SPEAKER_2": 38.7
  },
  "analysis": "Customer Sentiment: Neutral\nResolution: Resolved\nAgent Performance: Good\n...",
  "summary": "Order inquiry, resolved successfully",
  "generated_at": "2026-04-12T10:00:00Z"
}
```

The analysis covers 9 dimensions: sentiment, intent, resolution, agent performance, escalation risk, key topics, action items, compliance, and satisfaction score.

See the [Call Analytics Cookbook](/docs/call-analytics) for a full walkthrough with real examples.