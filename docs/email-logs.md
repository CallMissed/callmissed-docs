---
title: "Delivery, Suppressions & Usage"
description: "Read the send log, manage the suppression list, and check email spend and pricing."
slug: "email-logs"
breadcrumb: "Email"
---

# Delivery, Suppressions & Usage

Read the send log, manage the suppression list, and check email spend and pricing.

## Overview

Four read-mostly surfaces cover what happened after a send: the send log (`/sends`), the suppression list (`/suppressions`), engagement metrics (`/emails/metrics`), and spend (`/usage`). Everything except the suppression writes is a read, so a read-only key works.

To be told about a bounce or a complaint as it happens instead of polling, subscribe to [Email Webhooks](/docs/email-webhooks).

## Suppressions

A suppression list per account prevents sending to addresses that hard-bounced or complained. Entries are added automatically from delivery feedback, and you can manage them:

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/email/suppressions` | List suppressed addresses, newest first. `limit` (1–500, default 100) and `offset` (≥0, default 0) |
| `POST /api/v1/email/suppressions` | Suppress an address manually (`201`) |
| `POST /api/v1/email/suppressions/batch` | Suppress up to 100 addresses in one call (`201`) |
| `GET /api/v1/email/suppressions/{id}` | Retrieve one suppression by id |
| `DELETE /api/v1/email/suppressions/{id}` | Remove a suppression (`204`) |

| Create field | Type | Required | Notes |
|--------------|------|----------|-------|
| `address` | string | Yes | A valid email address. Stored lower-cased |
| `reason` | string | No | One of `hard_bounce`, `complaint`, `manual`, `unsubscribe`. Defaults to `manual`; anything else is a `422` |
| `detail` | string | No | Your own note about why |

Adding an address that is already suppressed is safe: the existing entry is returned unchanged rather than duplicated or rejected.

`SuppressionOut` returns `id`, `address`, `reason`, `detail`, and `created_at`. `GET /suppressions/{id}` returns the same object for a single entry; an id that is not yours is a `404`.

### Suppress in bulk

**`POST /api/v1/email/suppressions/batch`** applies one `reason` and `detail` to many addresses:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `addresses` | array | Yes | 1–100 valid email addresses. Stored lower-cased |
| `reason` | string | No | Same set as the single-add route. Defaults to `manual`; anything else is a `422` |
| `detail` | string | No | Your own note, applied to every address in the batch |

```bash
curl -X POST https://api.callmissed.com/api/v1/email/suppressions/batch \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "addresses": ["one@example.com", "two@example.com", "three@example.com"],
    "reason": "unsubscribe",
    "detail": "Imported from legacy list"
  }'
```

The `201` response is an array of `SuppressionOut` in the order you sent, so it lines up with your input. It is idempotent per address: one already on the list is returned unchanged rather than erroring, and duplicates within a single payload are collapsed. That means a partially-applied batch can simply be retried.

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

**`GET /api/v1/email/sends`** is your send log, newest first. Returns an array of `SendOut`:

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

### Filtering the send log

| Query param | Type | Notes |
|-------------|------|-------|
| `limit` | integer | 1–200, default 50 |
| `offset` | integer | ≥0, default 0 |
| `status` | string | Exact send status, e.g. `delivered`, `bounced`, `queued`. An unknown value is a `422` listing the valid set |
| `message_id` | string | Exact RFC 5322 `Message-ID` match, up to 255 chars |
| `recipient` | string | Match a `To:` address on the send, up to 320 chars. Case-insensitive and exact per address, so `bob@ex.com` will not match `notbob@ex.com`. A display form like `Alice <alice@x.com>` matches on the bare address. `cc` and `bcc` are deliberately not searched |
| `since` | string | ISO 8601. Only sends created at or after this timestamp |
| `until` | string | ISO 8601. Only sends created at or before this timestamp |

Filters combine. When you pass both `since` and `until`, the window may not exceed **90 days**, and `until` must not precede `since`; either violation is a `422`.

```bash
# Everything that bounced in a date window
curl -G https://api.callmissed.com/api/v1/email/sends \
  -H "Authorization: Bearer cm_your_key" \
  --data-urlencode "status=bounced" \
  --data-urlencode "since=2026-07-01T00:00:00Z" \
  --data-urlencode "until=2026-07-31T23:59:59Z"

# Every send to one recipient
curl -G https://api.callmissed.com/api/v1/email/sends \
  -H "Authorization: Bearer cm_your_key" \
  --data-urlencode "recipient=customer@example.com"
```

### Retrieve one send

**`GET /api/v1/email/sends/{send_id}`** returns a single `SendOut` for the `id` you got back from `POST /send`:

```bash
curl https://api.callmissed.com/api/v1/email/sends/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e \
  -H "Authorization: Bearer cm_your_key"
```

The fields are identical to a row from the list, so the two surfaces cannot drift. An id that is not yours is a `404`, not a `403`.

Message bodies are **not** returned: we do not retain the rendered `html`/`text` after the message is handed off for delivery. Keep your own copy if you need to display what was sent.

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

## Engagement metrics

**`GET /api/v1/email/emails/metrics`** is one aggregate row for a time window: volume, engagement and the derived rates. A read, so a read-only key works.

The repeated `email/emails` in that path is correct, not a typo: the metrics route is named `/emails/metrics` and sits under the `/api/v1/email` prefix like every other endpoint here.

| Query param | Notes |
|-------------|-------|
| `start_date` | ISO 8601. Defaults to 6 days before `end_date`. A value with no offset is treated as UTC |
| `end_date` | ISO 8601. Defaults to now; a future value is clamped to now |

The window may not exceed **90 days**, and `start_date` must be on or before `end_date`; either violation is a `422`.

```bash
# Default window (the last 6 days)
curl https://api.callmissed.com/api/v1/email/emails/metrics \
  -H "Authorization: Bearer cm_your_key"

# An explicit window
curl -G https://api.callmissed.com/api/v1/email/emails/metrics \
  -H "Authorization: Bearer cm_your_key" \
  --data-urlencode "start_date=2026-07-01T00:00:00Z" \
  --data-urlencode "end_date=2026-07-31T23:59:59Z"
```

```json
{
  "start_date": "2026-07-01T00:00:00Z",
  "end_date": "2026-07-31T23:59:59Z",
  "sent": 4820,
  "delivered": 4731,
  "bounced": 61,
  "complained": 3,
  "opened": 5904,
  "unique_opened": 2140,
  "clicked": 812,
  "unique_clicked": 655,
  "tracked_opens": 4820,
  "tracked_clicks": 4820,
  "delivery_rate": 0.9815,
  "bounce_rate": 0.0127,
  "complaint_rate": 0.0006,
  "open_rate": 0.4439,
  "click_rate": 0.1359
}
```

| Field | Type | Notes |
|-------|------|-------|
| `start_date` | string | The window actually used, after defaults and clamping |
| `end_date` | string | As above |
| `sent` | integer | Sends accepted for delivery in the window. This is the denominator for the delivery, bounce and complaint rates |
| `delivered` | integer | Confirmed delivered |
| `bounced` | integer | Bounced |
| `complained` | integer | Marked as spam |
| `opened` | integer | **Total** open hits. A mail client refetching the pixel increments this |
| `unique_opened` | integer | Distinct sends that were opened at least once |
| `clicked` | integer | Total click hits |
| `unique_clicked` | integer | Distinct sends that were clicked at least once |
| `tracked_opens` | integer | Sends in the window that actually carried an open pixel |
| `tracked_clicks` | integer | Sends in the window that actually carried rewritten links |
| `delivery_rate` | number | `delivered / sent` |
| `bounce_rate` | number | `bounced / sent` |
| `complaint_rate` | number | `complained / sent` |
| `open_rate` | number | `unique_opened / tracked_opens` |
| `click_rate` | number | `unique_clicked / tracked_clicks` |

Every rate is a fraction in `[0, 1]` rounded to 4 decimal places. An empty window returns zeros rather than nulls or an error, so a graph always has a number to plot.

**Open and click rates divide by the tracked subset, not by `sent`.** A message that carried no pixel cannot be opened, so counting it in the denominator would understate your real open rate. Whether a send carried tracking is recorded at send time, which is what keeps a rate meaningful across a window where you flipped a domain toggle. If `tracked_opens` is `0`, tracking is off for the domains you sent from: see [Open and click tracking](/docs/email-send#open-and-click-tracking).

Only sends accepted for delivery are counted. A rejected or still-queued send never reached a mailbox, so it is not in any denominator.

This is the aggregate total for one window. There is no per-day or per-dimension breakdown; call it once per window you want to chart.

## Pricing

**30 credits (₹30) per 1,000 emails**, charged per recipient to your credit balance, the same credits as every other API (your signup bonus counts). Only accepted sends are billed; rejected or failed sends cost nothing. See [Credits & Pricing](/docs/credits-rate-limits).

## Common failures on these routes

| Status | Body | Meaning |
|--------|------|---------|
| 401 | string `detail` | Missing, malformed or unrecognised `Authorization` header |
| 403 | string `detail` | The API key is read-only and this route writes (suppression writes only) |
| 404 | string `detail` | The suppression id, or the send id, is not yours |
| 422 | string `detail` | An unknown `status` on `GET /sends`, a date window over 90 days, `until` before `since`, or an unknown `reason` on `POST /suppressions/batch` |
| 422 | schema array `detail` | An unknown `reason` on `POST /suppressions`, or a malformed address in a batch |

Every shape is spelled out on [Limits, Quotas & Errors](/docs/email-limits#response-shapes).
