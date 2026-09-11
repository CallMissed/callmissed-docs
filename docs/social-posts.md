---
title: "Social Post Studio"
description: "Generate a ready-to-publish social post — image, caption and hashtags — in one call."
slug: "social-posts"
breadcrumb: "Images & Search"
---

# Social Post Studio

Generate a ready-to-publish social post — image, caption and hashtags — in one call.

## Overview

One call turns a topic into a post you can publish: an AI-generated image, a
caption written for the platform, and a set of hashtags.

**Endpoint:** `POST /v1/social/posts/generate`

It composes the two metered surfaces it sits beside — image generation and the
LLM — and bills as exactly that: the caption at the drafting model's token rate,
the image at your chosen model's per-image rate. Both figures come back on every
response.

:::flow
icon:app | Your app | Send a `topic` (and optionally a `platform`, `tone` and `image_model`)
icon:gateway | CallMissed gateway | Draft the caption and hashtags, then render the image
icon:image | Image model | Produce the post image
icon:done | Your app | Publish the caption, hashtags and image
:::

## Basic Usage

:::tabs
```python [Python]
import requests

res = requests.post(
    "https://api.callmissed.com/v1/social/posts/generate",
    headers={"Authorization": "Bearer cm_your_key"},
    json={
        "topic": "our bakery's new sourdough, baked fresh at 6am",
        "platform": "instagram",
        "tone": "warm",
        "hashtag_count": 8,
        "image_model": "flux-2-klein-9b",
        "size": "1024x1024",
    },
).json()

print(res["caption"])
print(" ".join(res["hashtags"]))
# res["image"]["b64_json"] → base64 PNG
# res["credits"]["total"]  → what this post cost
```
```javascript [JavaScript]
const res = await fetch(
  "https://api.callmissed.com/v1/social/posts/generate",
  {
    method: "POST",
    headers: {
      Authorization: "Bearer cm_your_key",
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      topic: "our bakery's new sourdough, baked fresh at 6am",
      platform: "instagram",
      tone: "warm",
      hashtag_count: 8,
    }),
  },
).then((r) => r.json());

console.log(res.caption, res.hashtags.join(" "));
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/social/posts/generate \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "topic": "our bakery'"'"'s new sourdough, baked fresh at 6am",
    "platform": "instagram",
    "tone": "warm",
    "hashtag_count": 8
  }'
```
:::

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `topic` | string | — | What the post is about. 3–2000 characters. Required. |
| `platform` | string | `generic` | `facebook`, `instagram` or `generic`. Sets the caption length ceiling and the voice. |
| `tone` | string | — | Optional voice, e.g. `warm`, `professional`, `playful`. Max 40 characters. |
| `hashtag_count` | integer | `8` | How many hashtags to return, 0–30. Instagram allows at most 30. |
| `generate_image` | boolean | `true` | Set `false` for a caption-only draft — no image credits, and no `image` key permission needed. |
| `image_model` | string | `flux-2-klein-9b` | Any model from the [image generation](/docs/image-generation) catalogue. |
| `image_prompt` | string | — | An explicit prompt for the image. When omitted, the image prompt is written **for you** from the `topic` — see below. Max 4000 characters. |
| `size` | string | — | Width×height for the image, e.g. `1024x1024`, `1024x1280`. Defaults to a 4:5 portrait, the ratio the Instagram and Facebook feeds prioritise. |
| `quality` | string | — | For models that support it (GPT Image): `low`, `medium`, `high` or `auto`. Ignored by models without a quality tier. |
| `reference_images` | string[] | — | Up to **16** base64 PNG/JPEG images used as visual references, so the picture is built **from your own** logo or product shot. Requires a `gpt-image-*` model (`gpt-image-2.5-sunburst`, `gpt-image-2.5-flare`, `gpt-image-2`, `gpt-image-1.5`). |
| `reference_pdf` | string | — | One base64 PDF. Its extracted **text** becomes brand context for the image prompt. Requires a `gpt-image-*` model (`gpt-image-2.5-sunburst`, `gpt-image-2.5-flare`, `gpt-image-2`, `gpt-image-1.5`). |

Caption length follows the platform: 2,200 characters for `instagram`, 5,000 for
`facebook`. `generic` uses the smaller of the two, so one draft is publishable on
either.

### Reference images and a brand PDF

Pass `reference_images` to put your real brand assets in the generated picture —
a logo, a product shot, a packaging photo. The image is then produced as an
**edit** of those references rather than a text-only generation, so the mark in
the output is yours instead of an invented lookalike.

Each item is base64, either bare or as a full `data:image/png;base64,…` URL (what
a browser's `FileReader.readAsDataURL` gives you, so no string surgery needed).
PNG and JPEG only — the format is checked from the file's own bytes, not from a
declared type. Up to 16 images, each at most 10 MB decoded.

`reference_pdf` is a different kind of reference: a brand guideline, a spec sheet
or a menu, whose **text** is extracted and folded into the image prompt as
context. The file itself never reaches the image model, and a PDF with no
extractable text (a scan) is ignored rather than failing the call. At most 10 MB
decoded.

**Both require an image model with an edit surface: `gpt-image-2.5-sunburst`,
`gpt-image-2.5-flare`, `gpt-image-2` or `gpt-image-1.5`.** Sent with any other `image_model`, the request fails `422`
before a single credit is reserved. That is deliberate — the alternative is
quietly dropping your references and charging full price for a picture carrying
none of your branding. For the same reason, a reference-carrying request will not
silently fall back to a model that cannot honour them.

**Pricing does not change.** A referenced image is billed at exactly the same
per-image rate as a plain generation of the same model, size and quality.

```bash [cURL]
curl -X POST https://api.callmissed.com/v1/social/posts/generate \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "topic": "our winter service package, booked till February",
    "platform": "instagram",
    "image_model": "gpt-image-2.5-sunburst",
    "reference_images": ["data:image/png;base64,iVBORw0KGgoAAAANS..."],
    "reference_pdf": "JVBERi0xLjQKJcfs..."
  }'
```

### How the image prompt is written

When you don't pass `image_prompt`, the endpoint does **not** send your `topic`
to the image model directly. A marketing topic ("winter tune-up, ₹1,200, booked
until February") sent to an image model renders the words literally — garbled
numbers, invented signage. Instead the caption drafter produces a separate,
purely-visual prompt: it keeps the facts (prices, dates, your business name) in
the **caption**, and describes a real, photographable scene for the **image**.
The result is a photo that represents the offer rather than a picture trying to
spell it out.

Pass your own `image_prompt` to override this and control the image directly.

### Response

```json
{
  "created": 1731234567,
  "topic": "our bakery's new sourdough, baked fresh at 6am",
  "platform": "instagram",
  "caption": "Pulled from the oven at 6am. Crackling crust, open crumb, still warm.",
  "hashtags": ["#sourdough", "#freshbread", "#bakery"],
  "image": {
    "b64_json": "<base64 PNG>",
    "revised_prompt": null,
    "url": "https://...signed-url..."
  },
  "model_used": "glm-4.7-flash",
  "image_model_used": "flux-2-klein-9b",
  "credits": {
    "image": 10.0,
    "llm": 0.065,
    "total": 10.065
  }
}
```

| Field | Description |
|-------|-------------|
| `caption` | The drafted caption, already trimmed to the platform's limit. |
| `hashtags` | Hashtags, each with its leading `#`. Empty when `hashtag_count` is 0. |
| `image` | `b64_json` plus a short-lived signed `url`. `null` when `generate_image` is `false`. |
| `model_used` | The model that drafted the caption. |
| `image_model_used` | The image model that actually served. `null` for a caption-only call. |
| `credits` | What you were charged, split by leg. `total` is the sum. |

`image.url` is a short-lived signed link — download or re-host it promptly. The
base64 payload in the same response has no expiry.

Generated images also appear in [`GET /v1/images/history`](/docs/image-generation#list-history)
alongside your other generations.

## Permissions

| Call | Required key permissions |
|------|--------------------------|
| `generate_image: true` (default) | `llm` **and** `image` |
| `generate_image: false` | `llm` only |

A key missing the permission for a leg gets `403 permission_denied` before
anything is generated or charged.

## Pricing

Billed as two separate line items on your usage, at the same rates as calling the
endpoints individually:

- **Caption + hashtags** — per token, at the drafting model's rate. A typical post
  is well under one credit.
- **Image** — the flat per-image price of `image_model`. See the
  [image pricing table](/docs/image-generation#pricing).

Every response carries the exact split in `credits`. A caption-only call is
charged for the caption alone.

Image credits are reserved before generation and **refunded in full** if the post
does not complete — and a request that fails is never charged for the caption
either. A failed post costs nothing.

## Errors

| HTTP | Code | Meaning |
|------|------|---------|
| 402 | `insufficient_credits` | Balance below the image cost. Top up in the dashboard. |
| 403 | `permission_denied` | Key lacks `llm` (or `image`, when generating one). |
| 403 | `model_not_available` | `image_model` needs a paid plan. |
| 404 | `model_not_found` | Unknown `image_model`. |
| 422 | `invalid_request_error` | Bad `topic`, `platform` or `hashtag_count` (0–30); a malformed or oversized `reference_images` / `reference_pdf`; or references sent with an `image_model` that has no edit surface. |
| 429 | `quota_exceeded` | Monthly plan cap hit for the LLM or image service. |
| 429 | `too_many_concurrent_requests` | Too many in-flight requests on this key. Retry shortly. |
| 502 | `upstream_error` | The caption or the image failed. No credits charged — safe to retry. |
| 503 | `service_unavailable` | Image service temporarily unavailable. No credits charged. |
