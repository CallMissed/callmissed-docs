---
title: "Delivery, Suppressions & Usage"
description: "Read the send log, manage the suppression list, and check email spend and pricing."
slug: "email-logs"
breadcrumb: "Email"
---

# Delivery, Suppressions & Usage

Read the send log, manage the suppression list, and check email spend and pricing.

## Overview

Three read-mostly surfaces cover what happened after a send: the send log (`/sends`), the suppression list (`/suppressions`), and spend (`/usage`). The send log and usage are reads, so a read-only key works on both.

## Suppressions

A suppression list per account prevents sending to addresses that hard-bounced or complained. Entries are added automatically from delivery feedback, and you can manage them:

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/email/suppressions` | List suppressed addresses, newest first. `limit` (1–500, default 100) and `offset` (≥0, default 0) |
| `POST /api/v1/email/suppressions` | Suppress an address manually (`201`) |
| `DELETE /api/v1/email/suppressions/{id}` | Remove a suppression (`204`) |

| Create field | Type | Required | Notes |
|--------------|------|----------|-------|
| `address` | string | Yes | A valid email address. Stored lower-cased |
| `reason` | string | No | One of `hard_bounce`, `complaint`, `manual`, `unsubscribe`. Defaults to `manual`; anything else is a `422` |
| `detail` | string | No | Your own note about why |

Adding an address that is already suppressed is safe: the existing entry is returned unchanged rather than duplicated or rejected.

`SuppressionOut` returns `id`, `address`, `reason`, `detail`, and `created_at`.

:::tabs
```bash [cURL]
# List
curl https://api.callmissed.com/api/v1/email/suppressions \
  -H "Authorization: Bearer cm_your_key"

# Suppress an address manually
curl -X POST https://api.callmissed.com/api/v1/email/suppressions \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"address": "blocked@example.com", "reason": "manual"}'

# Remove a suppression
curl -X DELETE https://api.callmissed.com/api/v1/email/suppressions/3f1c9a2d-4b5e-6a7f-8c9d-0e1f2a3b4c5d \
  -H "Authorization: Bearer cm_your_key"
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/email"
h = {"Authorization": "Bearer cm_your_key"}

httpx.get(f"{BASE}/suppressions", headers=h).json()
httpx.post(f"{BASE}/suppressions", headers=h, json={
    "address": "blocked@example.com",
    "reason": "manual",
})
httpx.delete(f"{BASE}/suppressions/3f1c9a2d-4b5e-6a7f-8c9d-0e1f2a3b4c5d", headers=h)
```
```javascript [JavaScript]
const BASE = "https://api.callmissed.com/api/v1/email";
const headers = {
  Authorization: "Bearer cm_your_key",
  "Content-Type": "application/json",
};

await fetch(`${BASE}/suppressions`, { headers }).then((r) => r.json());

await fetch(`${BASE}/suppressions`, {
  method: "POST",
  headers,
  body: JSON.stringify({ address: "blocked@example.com", reason: "manual" }),
});

await fetch(`${BASE}/suppressions/3f1c9a2d-4b5e-6a7f-8c9d-0e1f2a3b4c5d`, {
  method: "DELETE",
  headers: { Authorization: "Bearer cm_your_key" },
});
```
:::

Suppressed recipients are dropped from `to`, `cc`, and `bcc` before sending and returned in the send response's `suppressed` array. If every recipient is suppressed the send is refused with `403 all_recipients_suppressed`. A `404` on a suppression id uses the plain-string `detail` shape.

## Delivery Log & Usage

**`GET /api/v1/email/sends`** is your send log, newest first. Query params: `limit` (1–200, default 50) and `offset` (≥0, default 0). Returns an array of `SendOut`:

| Field | Type | Notes |
|-------|------|-------|
| `id` | string (UUID) | The send id returned by `POST /send` |
| `message_id` | string \| null | RFC 5322 `Message-ID`; null if the message was never built |
| `from_address` | string | The sender the message went out with |
| `subject` | string | The rendered subject |
| `status` | string | `queued`, `sent` (accepted for delivery), `delivered`, `bounced`, `complained`, `rejected` (we refused it), or `failed` (delivery error) |
| `size_bytes` | integer | Assembled message size |
| `sent_at` | string \| null | When it was accepted for delivery |
| `delivered_at` | string \| null | Set from delivery feedback |
| `bounced_at` | string \| null | Set from bounce feedback |
| `complained_at` | string \| null | Set from a spam complaint |
| `created_at` | string \| null | When the row was written |

One row per **message**, so a `messageVersions` batch writes one row per version. This is how you find out which versions of a batch failed.

**`GET /api/v1/email/usage`** is spend for the account. Cost is summed from the price stamped on each send at send time, so a later price change never rewrites history. Returns `UsageOut`:

| Field | Type | Notes |
|-------|------|-------|
| `currency` | string | `INR` |
| `price_per_1000` | number | Current price per 1,000 emails |
| `billed_sends` | integer | Sends that were actually charged, all time |
| `total_cost` | number | All-time spend in whole currency units |
| `sends_30d` | integer | Billed sends in the trailing 30 days |
| `cost_30d` | number | Spend in the trailing 30 days |

Both endpoints are reads, so a read-only key works.

:::tabs
```bash [cURL]
# Newest 50 sends
curl "https://api.callmissed.com/api/v1/email/sends?limit=50&offset=0" \
  -H "Authorization: Bearer cm_your_key"

# Spend
curl https://api.callmissed.com/api/v1/email/usage \
  -H "Authorization: Bearer cm_your_key"
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/email"
h = {"Authorization": "Bearer cm_your_key"}

sends = httpx.get(f"{BASE}/sends", headers=h, params={"limit": 50, "offset": 0}).json()
usage = httpx.get(f"{BASE}/usage", headers=h).json()

for row in sends:
    print(row["id"], row["status"], row["subject"])
```
```javascript [JavaScript]
const BASE = "https://api.callmissed.com/api/v1/email";
const headers = { Authorization: "Bearer cm_your_key" };

const sends = await fetch(`${BASE}/sends?limit=50&offset=0`, { headers }).then((r) =>
  r.json(),
);
const usage = await fetch(`${BASE}/usage`, { headers }).then((r) => r.json());

for (const row of sends) console.log(row.id, row.status, row.subject);
```
:::

## Pricing

**30 credits (₹30) per 1,000 emails**, charged per recipient to your credit balance, the same credits as every other API (your signup bonus counts). Only accepted sends are billed; rejected or failed sends cost nothing. See [Credits & Pricing](/docs/credits-rate-limits).

## Common failures on these routes

| Status | Body | Meaning |
|--------|------|---------|
| 401 | string `detail` | Missing, malformed or unrecognised `Authorization` header |
| 403 | string `detail` | The API key is read-only and this route writes (suppression writes only) |
| 404 | string `detail` | The suppression id is not yours |
| 422 | schema array `detail` | An unknown `reason` on `POST /suppressions` |

Every shape is spelled out on [Limits, Quotas & Errors](/docs/email-limits#response-shapes).
