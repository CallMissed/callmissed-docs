---
title: "Kimi K2.5 Fast (Maintenance)"
description: "High-throughput Kimi K2.5 inference tier — currently under maintenance. Use kimi-k2.5 in the meantime."
slug: "models-kimi-fast"
breadcrumb: "Getting Started > Models"
---

# Kimi K2.5 Fast (Maintenance)

High-throughput Kimi K2.5 inference tier — currently under maintenance. Use kimi-k2.5 in the meantime.

> **Under maintenance.** `kimi-k2.5-fast` is temporarily unavailable. Requests return HTTP 503 with `code: "model_under_maintenance"`. Use [`kimi-k2.5`](/docs/models) for production traffic; both ride on the same Kimi K2.5 model from Moonshot AI.

## Overview

The `kimi-k2.5-fast` tier targets ultra-low-latency voice-agent workloads via a high-throughput inference partner. While it's under maintenance, route the same workload through `kimi-k2.5` — the model and tokeniser are identical, only the inference latency differs.

## Kimi K2.5 Fast

| Field | Value |
|-------|-------|
| Model ID | `kimi-k2.5-fast` |
| Status | **Under maintenance** — returns 503 |
| Recommended fallback | `kimi-k2.5` |
| Architecture | MoE (Mixture of Experts) |
| Context window | 262,144 tokens |
| Supports streaming | Yes |
| Supports tools | Yes |

Kimi K2.5 (by Moonshot AI) is a 1T-parameter MoE model with 32B active parameters. It excels at reasoning, coding, and multilingual tasks.

## Usage

While `kimi-k2.5-fast` is in maintenance, point your code at `kimi-k2.5`:

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://api.callmissed.com/v1",
    api_key="cm_your_api_key",
)

response = client.chat.completions.create(
    model="kimi-k2.5",  # kimi-k2.5-fast is under maintenance
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain quantum computing briefly."},
    ],
    stream=True,
)

for chunk in response:
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

## Pricing

| Direction | Cost per 1M tokens |
|-----------|-------------------|
| Input | $0.81 |
| Output | $4.05 |

Credits: **1 credit = ₹1** (metered at roughly $0.01 of provider cost per credit — the USD figures above are the underlying rates). A typical voice agent turn (500 input + 200 output tokens) costs approximately **0.07 credits**. Pricing applies once `kimi-k2.5-fast` returns from maintenance; `kimi-k2.5` is priced separately on its [model page](/docs/models).