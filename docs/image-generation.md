---
title: "Image Generation"
description: "Generate images from a text prompt. OpenAI-compatible endpoint."
slug: "image-generation"
breadcrumb: "Images & Search"
---

# Image Generation

Generate images from a text prompt. OpenAI-compatible endpoint.

## Overview

Generate images from a text prompt. The request and response shape match OpenAI's `images.generate`, so any existing OpenAI SDK works by pointing `base_url` at `https://api.callmissed.com/v1`.

**Endpoint:** `POST /v1/images/generations`

Images come back as base64-encoded PNG (or JPEG, depending on the model) in the `data[].b64_json` field, with a short-lived signed `data[].url` alongside it. Send `response_format: "url"` to get only the link.

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
| `model` | string | `flux-2-klein-9b` | Model ID (see below). |
| `prompt` | string | — | Text description. 1–4000 characters. Required. |
| `n` | integer | 1 | Number of images. 1–4. Billed per image. |
| `size` | string | model default | Width×height, e.g. `1024x1024`, `768x768`, `1024x1536`. |
| `response_format` | string | `b64_json` | `b64_json` returns the image inline plus a signed `url`. `url` returns only the signed link (smaller response); if no link can be issued you get `b64_json` instead. |
| `negative_prompt` | string | — | Concepts to avoid (e.g. "lowres, blurry"). At most 4000 characters. |
| `seed` | integer | random | Reproducibility, `0`–`2147483647`. Same seed + prompt + model → same image. |
| `steps` | integer | auto | Denoising steps, 1–50. Higher = slower + more detail. |
| `reference_images` | string[] | — | Up to 16 base64 PNG/JPEG images (bare base64 or a `data:` URL), each at most 10 MB decoded. The image is produced as an **edit** of these references — e.g. to place your real logo. Requires a `gpt-image-*` model. |
| `reference_pdf` | string | — | One base64 PDF, at most 10 MB decoded. Its extracted text is added to the prompt as brand context. Requires a `gpt-image-*` model. |
| `user` | string | — | Your end user's id, for your own records. |

### Response

```json
{
  "created": 1731234567,
  "data": [
    { "b64_json": "<base64 PNG>", "url": "https://...signed-url...", "revised_prompt": null }
  ]
}
```

`url` is a short-lived signed link — download or re-host it promptly. It is omitted when the image could not be stored, so always fall back to `b64_json`.

`reference_images` and `reference_pdf` work only with `gpt-image-2.5-sunburst`, `gpt-image-2.5-flare`, `gpt-image-2` and `gpt-image-1.5`. With any other model the request fails `422` before any credits are reserved, rather than quietly ignoring your references.

## Models

| ID | Creator | Plan | Speed | Best for |
|----|---------|------|-------|----------|
| `flux-2-klein-9b` | Black Forest Labs | Free | Slow | Final output, print, marketing |
| `flux-2-dev` | Black Forest Labs | Free | Slow | Maximum fidelity, hero imagery |
| `lucid-origin` | Leonardo | Free | Medium | Cinematic, concept art |
| `phoenix-1.0` | Leonardo | Free | Medium | Photorealistic portraits |
| `sdxl-lightning` | ByteDance | Free | Fast | Prototyping, iteration |
| `dreamshaper-8-lcm` | Lykon | Free | Fast | Stylised illustrations |
| `flux-2-pro` | Black Forest Labs | Paid | Medium | Flagship FLUX fidelity |
| `flux-1.1-pro` | Black Forest Labs | Paid | Fast | Production-grade at lower cost |
| `gpt-image-2.5-sunburst` | OpenAI | Paid | Medium | Most capable generation + editing, inpainting |
| `gemini-3.1-flash-lite-image` | Google | Paid | Fast | Low-cost generation with reference edits |
| `gpt-image-2.5-flare` | OpenAI | Paid | Fast | Fast everyday generation, on-image text |
| `gpt-image-2` | OpenAI | Paid | Medium | Accurate on-image text, marketing visuals |
| `gpt-image-1.5` | OpenAI | Paid | Medium | Precise editing, logo/face preservation |
| `nano-banana-pro` | Google | Paid | Medium | Infographics, accurate typography |
| `nano-banana-2` | Google | Paid | Fast | Multimodal (text + reference images) |

Free-plan keys can call the six **Free** rows. Every **Paid** row needs Starter
or above; a free key gets `403 model_not_available`. `flux-1.1-pro` is currently
under maintenance and returns `503 model_under_maintenance` — use `flux-2-pro` or
`gpt-image-2`.

### Availability fallback

If the model you asked for fails with a temporary upstream error, the request
may be served by a similar fallback model instead of failing (for example
`gpt-image-2` → `gpt-image-1.5`). A fallback is only used when your plan and the
key's `allowed_models` permit it, never for a model under maintenance, and only
with models that can honour `reference_images` when you sent them. You are never
charged more than the price of the model you asked for: if a cheaper model
served, the difference is refunded. The served model is recorded in your
[usage logs](/docs/usage-api). Errors caused by the request itself (a content
policy refusal, a bad parameter) never fall back.

## Sizes

Common presets: `512x512`, `768x768`, `1024x1024`, `1024x1536`, `1536x1024`.

Any width/height from 64 to 4096 is accepted, but providers may clamp or round down to their supported values.

## Pricing

Flat per-image price, converted to credits at 1 credit = ₹1 ≈ US$0.0104 (US$1 = ₹96).

| Model | USD per image | Credits per image |
|-------|---------------|-------------------|
| `gpt-image-2.5-sunburst` | $0.2604 | 25 |
| `gpt-image-2.5-flare` | $0.2604 | 25 |
| `gpt-image-2` | $0.2604 | 25 |
| `gpt-image-1.5` | $0.2604 | 25 |
| `nano-banana-pro` | $0.1396 | 13.4 |
| `flux-2-dev` | $0.125 | 12 |
| `flux-2-klein-9b` | $0.1042 | 10 |
| `flux-2-pro` | $0.1042 | 10 |
| `phoenix-1.0` | $0.1042 | 10 |
| `lucid-origin` | $0.08333 | 8 |
| `nano-banana-2` | $0.06979 | 6.7 |
| `flux-1.1-pro` | $0.05208 | 5 |
| `sdxl-lightning` | $0.04167 | 4 |
| `dreamshaper-8-lcm` | $0.04167 | 4 |
| `gemini-3.1-flash-lite-image` | $0.035 | 3.36 |

Prices are for a standard-resolution (1K) image.

Credits for the request are reserved before generation and refunded in full if it fails, so a failed generation costs nothing.

## List History

Retrieve images previously generated with your API key, newest first. Useful for galleries and audit trails.

**Endpoint:** `GET /v1/images/history`

:::tabs
```python [Python]
import requests

resp = requests.get(
    "https://api.callmissed.com/v1/images/history",
    headers={"Authorization": "Bearer cm_your_key"},
    params={"limit": 20},
)
data = resp.json()
for item in data["data"]:
    print(item["id"], item["model"], item["url"])

# Paginate with the returned cursor
if data["next_cursor"]:
    next_page = requests.get(
        "https://api.callmissed.com/v1/images/history",
        headers={"Authorization": "Bearer cm_your_key"},
        params={"limit": 20, "before": data["next_cursor"]},
    ).json()
```
```bash [cURL]
curl "https://api.callmissed.com/v1/images/history?limit=20" \
  -H "Authorization: Bearer cm_your_key"
```
:::

| Query param | Type | Default | Description |
|-------------|------|---------|-------------|
| `limit` | integer | `20` | Rows to return (1–100). |
| `before` | string | — | ISO-8601 cursor — return rows created before this timestamp. Use `next_cursor` from the previous page. |

**Response:**

```json
{
  "data": [
    {
      "id": "a1b2c3d4-...",
      "url": "https://...signed-url...",
      "model": "nano-banana-pro",
      "size": "1024x1024",
      "prompt": "a red bicycle on a beach",
      "negative_prompt": null,
      "revised_prompt": null,
      "seed": 42,
      "steps": 28,
      "created": 1760000000
    }
  ],
  "next_cursor": "2026-04-12T10:00:00+00:00"
}
```

`url` is a short-lived signed link — download or re-host promptly. The API key must have `image` permission (otherwise `403 permission_denied`). `next_cursor` is `null` on the last page.

## Errors

| HTTP | Code | Meaning |
|------|------|---------|
| 400 | `invalid_request` | The model rejected the parameters (e.g. an unsupported size). |
| 400 | `content_policy_violation` | The prompt or references were refused by the model's safety filter. Not billed. |
| 402 | `insufficient_credits` | Balance below the request's cost. Top up in the dashboard. |
| 403 | `permission_denied` | API key lacks `image` permission. Edit the key in the dashboard. |
| 403 | `model_not_available` | A free-plan key calling a paid model. |
| 403 | `model_not_allowed` | The key's `allowed_models` list excludes the model. |
| 404 | `model_not_found` | Unknown model ID. |
| 422 | — | Validation failed: prompt length, `n`, `seed`, a malformed or oversized reference, or references sent to a model without an edit surface. |
| 429 | `quota_exceeded` | Monthly plan cap hit. Upgrade tier. |
| 429 | `rate_limit_exceeded` / `too_many_concurrent_requests` | Too many requests on this key. Retry with backoff. |
| 502 | `upstream_error` | The image provider failed. No credits debited — safe to retry. |
| 503 | `model_under_maintenance` | The model is temporarily unavailable; the message names an alternative. |
