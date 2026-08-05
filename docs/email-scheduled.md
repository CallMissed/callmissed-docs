---
title: "Scheduled & Batch Sending"
description: "Send later with scheduledAt, cancel before it fires, or deliver many recipient sets in one call with messageVersions."
slug: "email-scheduled"
breadcrumb: "Email"
---

# Scheduled & Batch Sending

Send later with scheduledAt, cancel before it fires, or deliver many recipient sets in one call with messageVersions.

## Overview

Two variants of `POST /api/v1/email/send` live on this page: a send that fires later (`scheduledAt`) and a send that carries many recipient sets at once (`messageVersions`). A message cannot be both. Every other field behaves exactly as it does on [Send Email](/docs/email-send).

## Scheduled Sending

Send a message later by adding `scheduledAt` to `POST /api/v1/email/send`. The send is accepted and enqueued: **no charge and no delivery happen at enqueue time**; billing and delivery occur when it fires at `scheduledAt`.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `scheduledAt` | string | Yes (to schedule) | ISO-8601 UTC timestamp. Must be in the **future** and **within 72 hours**, else `422`. A timestamp with no offset is treated as UTC rather than rejected |
| `batchId` | string (UUID) | No | A UUID you supply to group related scheduled sends; auto-generated if omitted |

A message cannot be both scheduled **and** a `messageVersions` batch; sending both returns `422`.

A scheduled send returns `202`:

```json
{
  "id": "3f1c9a2d-4b5e-6a7f-8c9d-0e1f2a3b4c5d",
  "message_id": "",
  "messageId": "",
  "messageIds": [],
  "status": "scheduled",
  "suppressed": [],
  "batchId": "7a2b1c0d-9e8f-4a5b-8c7d-6e5f4a3b2c1d",
  "scheduledAt": "2026-07-28T09:00:00Z"
}
```

`message_id` is empty and `suppressed` is `[]` because nothing has been built or filtered yet: suppression, quota, billing and delivery all run when the send fires. `batchId` and `scheduledAt` appear only on a scheduled enqueue.

### Managing scheduled sends

Manage scheduled sends by id **or** batchId:

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/email/scheduled/{identifier}` | List the scheduled rows for a send `id` or a `batchId`, ordered by `scheduledAt`. A batch lists all its rows, including ones that already fired or were cancelled. A UUID you don't own returns an empty list |
| `DELETE /api/v1/email/scheduled/{identifier}` | Cancel the **pending** scheduled send(s) by id or batchId (`204`). `404` if there's nothing pending to cancel |

An `identifier` that is not a UUID at all is a `404` on both routes rather than a `422`.

Each `ScheduledSendOut` returns `id`, `batch_id` / `batchId`, `scheduled_at` / `scheduledAt`, `status` (`pending` / `sent` / `failed` / `cancelled`), `send_id` (the delivered send once it fires), and `created_at` / `createdAt`. The camelCase keys are Brevo-compatible aliases of the snake_case ones and carry the same values.

:::tabs
```bash [cURL]
# Schedule a send for later (within 72h)
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Acme Ops <donotreply@acme.com>",
    "to": ["Ada <customer@example.com>"],
    "subject": "Reminder",
    "text": "Your appointment is tomorrow.",
    "scheduledAt": "2026-07-28T09:00:00Z"
  }'

# Inspect the scheduled rows by send id or batchId
curl https://api.callmissed.com/api/v1/email/scheduled/3f1c9a2d-4b5e-6a7f-8c9d-0e1f2a3b4c5d \
  -H "Authorization: Bearer cm_your_key"

# Cancel it before it fires - never charged
curl -X DELETE https://api.callmissed.com/api/v1/email/scheduled/3f1c9a2d-4b5e-6a7f-8c9d-0e1f2a3b4c5d \
  -H "Authorization: Bearer cm_your_key"
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/email"
h = {"Authorization": "Bearer cm_your_key"}

scheduled = httpx.post(f"{BASE}/send", headers=h, json={
    "from": "Acme Ops <donotreply@acme.com>",
    "to": ["Ada <customer@example.com>"],
    "subject": "Reminder",
    "text": "Your appointment is tomorrow.",
    "scheduledAt": "2026-07-28T09:00:00Z",
}).json()

httpx.get(f"{BASE}/scheduled/{scheduled['id']}", headers=h).json()
httpx.delete(f"{BASE}/scheduled/{scheduled['id']}", headers=h)
```
```javascript [JavaScript]
const BASE = "https://api.callmissed.com/api/v1/email";
const headers = {
  Authorization: "Bearer cm_your_key",
  "Content-Type": "application/json",
};

const scheduled = await fetch(`${BASE}/send`, {
  method: "POST",
  headers,
  body: JSON.stringify({
    from: "Acme Ops <donotreply@acme.com>",
    to: ["Ada <customer@example.com>"],
    subject: "Reminder",
    text: "Your appointment is tomorrow.",
    scheduledAt: "2026-07-28T09:00:00Z",
  }),
}).then((r) => r.json());

await fetch(`${BASE}/scheduled/${scheduled.id}`, {
  headers: { Authorization: "Bearer cm_your_key" },
});

await fetch(`${BASE}/scheduled/${scheduled.id}`, {
  method: "DELETE",
  headers: { Authorization: "Bearer cm_your_key" },
});
```
:::

A cancelled send never fires and is never charged.

## Batch (messageVersions)

Send to many recipient sets in one call. Add `messageVersions`, an array where each version is its own recipient set that may override the global subject, body, params, or template.

**Per-version fields:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `to` | array | Yes | Recipient addresses (same forms as the top-level send) |
| `cc` | array | No | Carbon-copy recipients |
| `bcc` | array | No | Blind-copy recipients |
| `subject` | string | No | Overrides the global subject |
| `htmlContent` | string | No | Overrides the global HTML body |
| `textContent` | string | No | Overrides the global text body |
| `replyTo` | string \| object | No | Overrides the global reply-to |
| `params` | object | No | Substitution values for this version, shallow-merged over the global `params` |
| `templateId` | string (UUID) | No | Overrides the global template |

- **Base / override.** The top-level `subject` / `html` / `text` / `templateId` / `params` / `from` are the **base** each version overrides. A per-version body override requires a global body to be present; a per-version `templateId` requires a global `templateId`. Global attachments and tags apply to all versions; there are no per-version attachments.
- **Limits** (exceeding any → `422`): ≤99 recipients per version, ≤2000 recipients across the batch (deduped), ≤1000 versions, ≤100 KB per-version `params`, ≤1000 KB `params` across the batch. The 50-recipient single-send cap does **not** apply here; the batch union cap replaces it.
- **A recipient is delivered by exactly one version, the first.** Versions are processed in array order, and each version delivers only the recipients no earlier version already claimed. If `ada@example.com` appears in version 1 **and** version 2, she receives **version 1's** subject, body and params, and version 2 simply does not send to her at all. This is a delivery outcome, not only a billing rule: repeating an address across versions silently drops the later content. Keep each version's recipient set disjoint.
- **Billing.** The **deduped union** of recipients across all versions is billed **once** at 30 credits (₹30) per 1,000; a recipient in two versions is billed once, matching the delivery rule above. Suppression, quota, rate and monthly-cap checks are likewise evaluated once, over the union.
- **A partial failure still returns `202`.** The response `status` is `sent` when **any** version was accepted for delivery; only an all-versions-failed batch returns `502 relay_failed`. `messageIds` carries one id per version in array order **whether or not that version was accepted**, so the response alone cannot tell you which versions failed. To find out, list the sends and read each row's `status`.

A batch returns `202` with `messageIds` (one per version, in order) and a `batchId` grouping the batch's sends; `id` / `messageId` are the first version, and `suppressed` lists the union's suppressed addresses. Look up each send with `GET /api/v1/email/sends`. See [Delivery Log & Usage](/docs/email-logs).

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Acme Ops <donotreply@acme.com>",
    "templateId": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
    "subject": "Your receipt, {{ params.name }}",
    "messageVersions": [
      { "to": ["Ada <ada@example.com>"], "params": { "name": "Ada", "order_id": "1043" } },
      { "to": ["Bo <bo@example.com>"], "params": { "name": "Bo", "order_id": "1044" } }
    ]
  }'
```
```python [Python]
import httpx

httpx.post(
    "https://api.callmissed.com/api/v1/email/send",
    headers={"Authorization": "Bearer cm_your_key"},
    json={
        "from": "Acme Ops <donotreply@acme.com>",
        "templateId": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
        "subject": "Your receipt, {{ params.name }}",
        "messageVersions": [
            {"to": ["Ada <ada@example.com>"], "params": {"name": "Ada", "order_id": "1043"}},
            {"to": ["Bo <bo@example.com>"], "params": {"name": "Bo", "order_id": "1044"}},
        ],
    },
)
```
```javascript [JavaScript]
await fetch("https://api.callmissed.com/api/v1/email/send", {
  method: "POST",
  headers: {
    Authorization: "Bearer cm_your_key",
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    from: "Acme Ops <donotreply@acme.com>",
    templateId: "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
    subject: "Your receipt, {{ params.name }}",
    messageVersions: [
      { to: ["Ada <ada@example.com>"], params: { name: "Ada", order_id: "1043" } },
      { to: ["Bo <bo@example.com>"], params: { name: "Bo", order_id: "1044" } },
    ],
  }),
});
```
:::

```json
{
  "id": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
  "messageId": "<1a2b3c4d@acme.com>",
  "messageIds": ["<1a2b3c4d@acme.com>", "<5e6f7g8h@acme.com>"],
  "batchId": "7a2b1c0d-9e8f-4a5b-8c7d-6e5f4a3b2c1d",
  "status": "sent",
  "suppressed": []
}
```

## Common failures

| Status | Reason | Meaning |
|--------|--------|---------|
| 422 | `scheduled_batch_unsupported` | A send set both `scheduledAt` and `messageVersions`, pick one |
| 422 | *(schema array `detail`)* | A `scheduledAt` in the past or beyond the 72-hour horizon, over-limit `params`, or a broken batch cross-version rule |
| 422 | `too_many_recipients` | Over the union limit on a batch |
| 404 | *(string `detail`)* | `DELETE /scheduled/{identifier}` found nothing pending to cancel |
| 502 | `relay_failed` | Every version of a batch failed. Flat shape, carries `id` |

Shapes and the full reason table: [Limits, Quotas & Errors](/docs/email-limits).
