---
title: "Call Analytics Pipeline"
description: "A production-ready call analytics pipeline on the CallMissed API — batch STT with diarization, speaker-wise parsing, and LLM-powered analysis."
slug: "call-analytics"
breadcrumb: "Cookbooks"
---

# Call Analytics Pipeline

A production-ready call analytics pipeline on the CallMissed API — batch STT with diarization, speaker-wise parsing, and LLM-powered analysis.

## Overview

This cookbook demonstrates a robust, production-ready call analytics pipeline on the CallMissed API. It uses the **Call Analytics endpoint** for batch speech-to-text with diarization, parses speaker-wise transcripts, and runs LLM-powered analysis across 9 dimensions — all behind a single `cm_` API key.

## Business Value

- Improve agent effectiveness
- Understand customer sentiment
- Detect operational issues early
- Spot upsell/cross-sell opportunities

**Where is it useful:**

- **E-commerce / D2C** — Understand refund requests, delivery concerns, or dissatisfaction with product quality.
- **Contact Centers / BPOs** — Automate call reviews to improve training and ensure compliance at scale.
- **Healthcare & Insurance** — Analyze patient queries, support delays, and sentiment in sensitive service calls.

## 1. Get an API Key

1. Create a key in the [dashboard](https://app.callmissed.com/api-keys) — it looks like `cm_live_...`.
2. Set it as an environment variable so it never lands in source control:

```bash
export CALLMISSED_API_KEY="cm_live_..."
```

## 2. Analyze a Recording

A single `POST` to `/api/v1/analytics/calls/analyze` uploads the audio, runs diarized STT, and returns the transcript, per-speaker timing, and analysis in one response.

```python
import os
import requests

API_KEY = os.environ["CALLMISSED_API_KEY"]
BASE_URL = "https://api.callmissed.com/api/v1"

with open("call_recording.mp3", "rb") as audio:
    resp = requests.post(
        f"{BASE_URL}/analytics/calls/analyze",
        headers={"Authorization": f"Bearer {API_KEY}"},
        files={"file": audio},
        data={"language": "hi-IN"},  # omit for auto-detect
        timeout=300,
    )
resp.raise_for_status()
result = resp.json()
```

**Supported formats:** WAV, MP3, MP4, M4A, OGG, FLAC, WebM, AAC, AMR (max 100 MB).

## 3. CallAnalytics Class

Wrap the two endpoints (`analyze` + `question`) in a small reusable class:

```python
class CallAnalytics:
    def __init__(self, api_key: str, base_url: str = "https://api.callmissed.com/api/v1"):
        self.base_url = base_url
        self.headers = {"Authorization": f"Bearer {api_key}"}
        self.result = None

    def analyze(self, audio_path: str, language: str | None = None) -> dict:
        data = {"language": language} if language else {}
        with open(audio_path, "rb") as audio:
            resp = requests.post(
                f"{self.base_url}/analytics/calls/analyze",
                headers=self.headers,
                files={"file": audio},
                data=data,
                timeout=300,
            )
        resp.raise_for_status()
        self.result = resp.json()
        return self.result

    def answer_question(self, question: str) -> str:
        resp = requests.post(
            f"{self.base_url}/analytics/calls/question",
            headers={**self.headers, "Content-Type": "application/json"},
            json={"transcript": self.result["transcript"], "question": question},
        )
        resp.raise_for_status()
        return resp.json()["answer"]

    def get_summary(self) -> str:
        return self.result["summary"]
```

## 4. Full Workflow

```python
import os

analytics = CallAnalytics(api_key=os.environ["CALLMISSED_API_KEY"])

analytics.analyze("/path/to/your/audio/file.mp3", language="hi-IN")
print(analytics.answer_question("Was the customer satisfied with the resolution?"))
print(analytics.get_summary())
```

## 5. Sample Output

```
### Speaker Identification
- Customer: SPEAKER_00 (Adam Wilson)
- Agent: SPEAKER_01 (Sam from Coaching Downs)

### Customer Type
- Existing customer — has a previous order

### Resolution
- Escalated to corporate office. Email update promised in 2-4 business days.
```

The analysis covers 9 dimensions: sentiment, intent, resolution, agent performance, escalation risk, key topics, action items, compliance, and satisfaction score.

## 6. Resources

- **Batch STT reference**: [Batch STT](/docs/stt-batch)
- **Speech to Text overview**: [Speech to Text](/docs/speech-to-text)
- **Talk to us**: [Support &amp; Enterprise](/docs/talk-to-us)