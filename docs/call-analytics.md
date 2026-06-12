---
title: "Call Analytics Pipeline"
description: "A production-ready call analytics pipeline using Sarvam's SDK — batch STT with diarization, speaker-wise parsing, and LLM-powered analysis."
slug: "call-analytics"
breadcrumb: "Cookbooks"
---

# Call Analytics Pipeline

A production-ready call analytics pipeline using Sarvam's SDK — batch STT with diarization, speaker-wise parsing, and LLM-powered analysis.

## Overview

This cookbook demonstrates a robust, production-ready call analytics pipeline using Sarvam's SDK. It leverages Sarvam's Speech-to-Text Translate Batch API with diarization, parses speaker-wise transcripts, and uses Sarvam's LLM for deep analysis.

## Business Value

- Improve agent effectiveness
- Understand customer sentiment
- Detect operational issues early
- Spot upsell/cross-sell opportunities

**Where is it useful:**

- **E-commerce / D2C** — Understand refund requests, delivery concerns, or dissatisfaction with product quality.
- **Contact Centers / BPOs** — Automate call reviews to improve training and ensure compliance at scale.
- **Healthcare & Insurance** — Analyze patient queries, support delays, and sentiment in sensitive service calls.

## 1. Install SDK

```bash
pip install sarvamai
```

## 2. Authentication

1. Obtain your API key from the [Sarvam AI Dashboard](https://dashboard.sarvam.ai/)
2. Replace `"YOUR_SARVAM_AI_API_KEY"` with your actual key

## 3. Setup Modules

```python
import os
import json
import hashlib
from pathlib import Path
from datetime import datetime
from typing import List, Dict, Optional
from pydub import AudioSegment
from sarvamai import SarvamAI
import textwrap

OUTPUT_DIR = "outputs"
Path(OUTPUT_DIR).mkdir(exist_ok=True)
```

## 4. CallAnalytics Class

```python
class CallAnalytics:
    def __init__(self, client):
        self.client = client
        self.transcriptions = {}

    def process_audio_files(self, audio_paths: List[str]) -> Dict[str, str]:
        job = client.speech_to_text_translate_job.create_job(
            model="saaras:v3",
            mode="translate",
            with_diarization=True,
        )
        job.upload_files(file_paths=audio_paths, timeout=300)
        job.start()
        job.wait_until_complete()
        # ... process results
```

## 5. Full Workflow

```python
client = SarvamAI(api_subscription_key="YOUR_SARVAM_AI_API_KEY")
analytics = CallAnalytics(client=client)

audio_path = "/path/to/your/audio/file.mp3"
analytics.process_audio_files([audio_path])
analytics.answer_question("Was the customer satisfied with the resolution?")
analytics.get_summary()
```

## 6. Sample Output

```
### Speaker Identification
- Customer: SPEAKER_00 (Adam Wilson)
- Agent: SPEAKER_01 (Sam from Coaching Downs)

### Customer Type
- Existing customer — has a previous order

### Resolution
- Escalated to corporate office. Email update promised in 2-4 business days.
```

## 7. Resources

- **Sarvam AI Docs**: [docs.sarvam.ai](https://docs.sarvam.ai)
- **Sample Audio Files**: [GitHub Cookbook](https://github.com/sarvamai/sarvam-ai-cookbook)
- **Community**: [Discord](https://discord.com/invite/5rAsykttcs)