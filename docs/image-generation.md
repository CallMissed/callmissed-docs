---
title: "Image Generation"
description: "Generate images from a text prompt. OpenAI-compatible endpoint."
slug: "image-generation"
breadcrumb: "API Guides & Tutorials"
---

# Image Generation

Generate images from a text prompt. OpenAI-compatible endpoint.

## Overview

Generate images from a text prompt. The request and response shape match OpenAI's `images.generate`, so any existing OpenAI SDK works by pointing `base_url` at `https://api.callmissed.com/v1`.

**Endpoint:** `POST /v1/images/generations`

Images come back as base64-encoded PNG (or JPEG, depending on the model) in the `data[].b64_json` field.

:::flow
icon:app | Your app | Send a `prompt`, `model`, and `size` to `POST /v1/images/generations`
icon:gateway | CallMissed gateway | Route to the image provider and deduct per-image credits
icon:image | Image model | Render the image from your prompt
icon:done | Your app | Decode `data[].b64_json` (base64 PNG/JPEG) and save or display it
:::

## Basic Usage

:::tabs
```python [Python]
from openai import OpenAI

client = OpenAI(
    api_key="cm_your_key",
    base_url="https://api.callmissed.com/v1",
)

res = client.images.generate(
    model="flux-2-klein-9b",
    prompt="A golden retriever in a sunlit library, cinematic bokeh",
    n=1,
    size="1024x1024",
)
# res.data[0].b64_json → base64 image
```
```javascript [JavaScript]
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "cm_your_key",
  baseURL: "https://api.callmissed.com/v1",
});

const res = await client.images.generate({
  model: "flux-2-klein-9b",
  prompt: "A golden retriever in a sunlit library, cinematic bokeh",
  n: 1,
  size: "1024x1024",
});
// res.data[0].b64_json → base64 image
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/images/generations \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "flux-2-klein-9b",
    "prompt": "A golden retriever in a sunlit library",
    "n": 1,
    "size": "1024x1024"
  }'
```
:::

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | — | Model ID (see below). Required. |
| `prompt` | string | — | Text description. 1–4000 characters. Required. |
| `n` | integer | 1 | Number of images. 1–4. Billed per image. |
| `size` | string | 1024x1024 | Width×height, e.g. `1024x1024`, `768x768`, `1024x1536`. |
| `response_format` | string | `b64_json` | Only `b64_json` supported today. |
| `negative_prompt` | string | — | Concepts to avoid (e.g. "lowres, blurry"). |
| `seed` | integer | random | Reproducibility. Same seed + prompt + model → same image. |
| `steps` | integer | auto | Denoising steps, 1–50. Higher = slower + more detail. |

### Response

```json
{
  "created": 1731234567,
  "data": [
    { "b64_json": "<base64 PNG>", "revised_prompt": null }
  ]
}
```

## Models

| ID | Provider | Quality | Speed | Best for |
|----|----------|---------|-------|----------|
| `nano-banana-pro` | Google (direct) | Flagship | Medium | Marketing infographics, accurate typography, brand-true visuals |
| `nano-banana-2` | Google (direct) | High | Fast | Multimodal (text + reference images), highest LM-Arena Elo |
| `flux-2-dev` | Black Forest Labs (Cloudflare) | Flagship | Slow | Maximum fidelity, hero imagery |
| `flux-2-klein-9b` | Black Forest Labs (Cloudflare) | Highest | Slow | Final output, print, marketing |
| `lucid-origin` | Leonardo (Cloudflare) | High | Medium | Cinematic, concept art |
| `phoenix-1.0` | Leonardo (Cloudflare) | High | Medium | Photorealistic portraits |
| `sdxl-lightning` | ByteDance (Cloudflare) | Medium | Fast | Prototyping, iteration |
| `dreamshaper-8-lcm` | Lykon (Cloudflare) | Medium | Fast | Stylised illustrations |

The `nano-banana-*` models are routed direct to Google AI Studio (no
markup). They are paid-only — the Cloudflare-hosted rows above stay on
the free plan. See [Pricing](#pricing) below for exact per-image rates.

## Sizes

Common presets: `512x512`, `768x768`, `1024x1024`, `1024x1536`, `1536x1024`.

Any width/height from 64 to 4096 is accepted, but providers may clamp or round down to their supported values.

## Pricing

Flat per-image price, converted to credits at 1 credit = ₹1.

| Model | USD per image | Credits per image |
|-------|---------------|-------------------|
| `nano-banana-pro` | $0.134 | 13.4 |
| `flux-2-dev` | $0.12 | 12 |
| `flux-2-klein-9b` | $0.10 | 10 |
| `phoenix-1.0` | $0.10 | 10 |
| `lucid-origin` | $0.08 | 8 |
| `nano-banana-2` | $0.067 | 6.7 |
| `sdxl-lightning` | $0.04 | 4 |
| `dreamshaper-8-lcm` | $0.04 | 4 |

Both `nano-banana-*` models are routed direct to Google AI Studio. Pricing is pass-through — exactly what Google charges for a standard-resolution (1K) image.

Credits are deducted **after** the upstream call returns successfully. A failed generation does not cost credits.

## Errors

| HTTP | Code | Meaning |
|------|------|---------|
| 400 | `invalid_request_error` | Bad prompt / size / n. Check the parameter table. |
| 402 | `insufficient_credits` | Balance below the request's cost. Top up in the dashboard. |
| 403 | `permission_denied` | API key lacks `image` permission. Edit the key in the dashboard. |
| 404 | `model_not_found` | Unknown model ID. |
| 429 | `quota_exceeded` | Monthly plan cap hit. Upgrade tier. |
| 400 | `invalid_request` | Invalid parameters (e.g. unsupported size). Upstream validation error passed through. |
| 502 | `upstream_error` | Network failure reaching the image provider. No credits debited — safe to retry. |