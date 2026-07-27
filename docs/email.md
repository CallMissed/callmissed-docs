---
title: "Email API"
description: "Send and receive email from your own domain over the API — verified-domain onboarding, DKIM signing, delivery and suppression tracking."
slug: "email"
breadcrumb: "API Guides & Tutorials"
---

# Email API

Send and receive email from your own domain over the API — verified-domain onboarding, DKIM signing, delivery and suppression tracking.

## Overview

The Email API sends and receives email from a domain you own. You verify the domain once (we generate its DKIM key and the DNS records to publish), then send over the API and, optionally, receive mail at addresses on that domain.

**Base path:** `https://api.callmissed.com/api/v1/email`

Authentication uses your existing CallMissed API key — the same `cm_` key you use for every other API. The key needs the **email** permission enabled (toggle it on the [API keys](https://app.callmissed.com/api-keys) page). No separate email key.

:::flow
icon:app | Your app | Add a domain, publish the DNS records we generate
icon:gateway | CallMissed | Verify SPF + DKIM, then accept sends from that domain
icon:done | Recipients | Receive DKIM-signed mail from your own domain
:::

> **Billing:** Email is fully credit-based. Every send is charged to your credit balance at **₹30 per 1,000 emails** (per recipient) — the same wallet as every other API. There is no separate email invoice.

## Add & Verify a Domain

Register a domain, publish the returned DNS records at your DNS provider, then verify.

:::tabs
```bash [cURL]
# 1. Add the domain — returns the DNS records to publish
curl -X POST https://api.callmissed.com/api/v1/email/domains \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"domain": "acme.com"}'

# 2. After publishing the records, verify
curl -X POST https://api.callmissed.com/api/v1/email/domains/{domain_id}/verify \
  -H "Authorization: Bearer cm_your_key"
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/email"
h = {"Authorization": "Bearer cm_your_key"}

created = httpx.post(f"{BASE}/domains", headers=h, json={"domain": "acme.com"}).json()
for rec in created["dns_records"]:
    print(rec["type"], rec["host"], rec["value"])  # publish these at your DNS host

# once published:
httpx.post(f"{BASE}/domains/{created['domain']['id']}/verify", headers=h)
```
:::

The add response returns the records you must publish:

| Record | Purpose |
|--------|---------|
| `TXT` (SPF) | Authorises our infrastructure to send for the domain |
| `TXT` (DKIM) | Lets receivers verify the signature on your mail |
| `TXT` (DMARC) | Reporting policy (recommended) |
| `MX` | Routes inbound mail to us (only needed to **receive**) |

Sending is allowed once SPF + DKIM verify. A new domain starts on a warm-up quota that rises automatically as it sends clean volume.

## Send Email

**Endpoint:** `POST /api/v1/email/send`

The send body is a **superset**: it accepts both the original CallMissed shapes (string `from`, string-list `to`, `text`/`html`) and Brevo-style shapes (`sender` object, recipient objects, `textContent`/`htmlContent`). A Brevo `sendTransacEmail` integration works here by changing only the base URL and the auth header — see [Switching from Brevo](#switching-from-brevo) below.

The example below uses `cc`, one URL attachment and one base64 attachment, `tags`, and the `Idempotency-Key` header:

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: order-1043-receipt" \
  -d '{
    "from": "Ops <ops@acme.com>",
    "to": ["Ada <customer@example.com>"],
    "cc": ["accounts@example.com"],
    "subject": "Your receipt",
    "text": "Thanks for your order. Your receipt is attached.",
    "html": "<p>Thanks for your order. Your receipt is attached.</p>",
    "reply_to": "support@acme.com",
    "tags": ["receipt", "order-1043"],
    "attachment": [
      { "url": "https://acme.com/receipts/1043.pdf" },
      { "name": "terms.txt", "content": "VGhhbmsgeW91IGZvciB5b3VyIG9yZGVyLg==" }
    ]
  }'
```
```python [Python]
import base64, httpx

pdf_b64 = base64.b64encode(b"...raw bytes...").decode()

httpx.post(
    "https://api.callmissed.com/api/v1/email/send",
    headers={
        "Authorization": "Bearer cm_your_key",
        "Idempotency-Key": "order-1043-receipt",
    },
    json={
        "from": "Ops <ops@acme.com>",
        "to": ["Ada <customer@example.com>"],
        "cc": ["accounts@example.com"],
        "subject": "Your receipt",
        "text": "Thanks for your order. Your receipt is attached.",
        "html": "<p>Thanks for your order. Your receipt is attached.</p>",
        "reply_to": "support@acme.com",
        "tags": ["receipt", "order-1043"],
        "attachment": [
            {"url": "https://acme.com/receipts/1043.pdf"},
            {"name": "terms.txt", "content": pdf_b64},
        ],
    },
)
```
```javascript [JavaScript]
const termsB64 = Buffer.from("Thank you for your order.").toString("base64");

await fetch("https://api.callmissed.com/api/v1/email/send", {
  method: "POST",
  headers: {
    Authorization: "Bearer cm_your_key",
    "Content-Type": "application/json",
    "Idempotency-Key": "order-1043-receipt",
  },
  body: JSON.stringify({
    from: "Ops <ops@acme.com>",
    to: ["Ada <customer@example.com>"],
    cc: ["accounts@example.com"],
    subject: "Your receipt",
    text: "Thanks for your order. Your receipt is attached.",
    html: "<p>Thanks for your order. Your receipt is attached.</p>",
    reply_to: "support@acme.com",
    tags: ["receipt", "order-1043"],
    attachment: [
      { url: "https://acme.com/receipts/1043.pdf" },
      { name: "terms.txt", content: termsB64 },
    ],
  }),
});
```
:::

### Fields

An **address** may be written three ways, and you can mix them within one array: a bare `"a@b.com"`, a display form `"Name <a@b.com>"`, or an object `{"email": "a@b.com", "name": "Name"}`.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `from` | string | one of `from` / `sender` | Sender as `"Name <ops@acme.com>"` or bare address; must be on one of your verified domains |
| `sender` | object | one of `from` / `sender` | Brevo-style sender `{"email", "name"}` — an alternative to `from` |
| `to` | array | Yes | One or more recipient addresses (min 1). Each delivered recipient is billed |
| `cc` | array | No | Carbon-copy recipients. Appear in the `Cc` header **and** are delivered |
| `bcc` | array | No | Blind-copy recipients. Delivered but **never** written to any header |
| `subject` | string | No | Up to 998 characters |
| `text` | string | body required | Plain-text body |
| `html` | string | body required | HTML body |
| `textContent` | string | body required | Brevo alias for `text` |
| `htmlContent` | string | body required | Brevo alias for `html` |
| `reply_to` | string | No | Reply-To address as a string |
| `replyTo` | string \| object | No | Brevo alias for `reply_to` — string or `{"email", "name"}` |
| `headers` | object | No | Extra headers as string→string. Reserved headers (`From`, `To`, `Cc`, `Bcc`, `Reply-To`, `Subject`, `Date`, `Message-ID`, `DKIM-Signature`, `Received`) are ignored |
| `attachment` | array | No | Attachments; also accepted as `attachments`. See below |
| `tags` | array | No | Up to 10 string tags for your own categorisation; trimmed, empties dropped |
| `templateId` | string (UUID) | No | Send from a saved [template](#templates) — its subject/body are the base; explicit send fields override. See [Templates](#templates) |
| `params` | object | No | Substitution values for `{{ params.KEY }}` placeholders in the template |
| `scheduledAt` | string | No | ISO-8601 UTC timestamp to send later (future, within 72h). See [Scheduled Sending](#scheduled-sending) |
| `batchId` | string (UUID) | No | Groups related scheduled sends; auto-generated if omitted |
| `messageVersions` | array | No | One call, many recipient sets — each overrides the base. See [Batch (messageVersions)](#batch-messageversions) |

Provide **at least one** of `text` / `html` (or their Brevo aliases), a `templateId`, or `messageVersions`. Provide **exactly one** of `from` / `sender`. A top-level `to` is not required when `messageVersions` is present.

**Attachments.** Each item in `attachment` is **either** a URL reference **or** inline base64 — exactly one of the two:

| Attachment field | Type | Required | Notes |
|------------------|------|----------|-------|
| `url` | string | one of `url` / `content` | `http`/`https` URL fetched at send time. Internal/private URLs are refused; the fetched file is size-capped |
| `content` | string | one of `url` / `content` | Base64-encoded file bytes |
| `name` | string | with `content` | Filename (≤255 chars). Required when `content` is set; optional with `url` |

**Headers.**

| Header | Notes |
|--------|-------|
| `Authorization` | `Bearer cm_...` — the key needs the **email** permission (required) |
| `Idempotency-Key` | Optional. A repeat with the same key returns the first send's result without sending or charging again |

### Semantics

- **cc vs bcc.** `cc` recipients are written to the `Cc` header and delivered; `bcc` recipients are delivered but never appear in any header.
- **De-duplication.** An address listed in both `to` and `cc` is delivered — and billed — once.
- **Suppression.** Recipients on your [suppression list](#suppressions) are dropped from `to`, `cc`, and `bcc` before sending, and returned in `suppressed`.
- **Validation.** Recipient addresses are validated first — a malformed address, or one containing control characters, is rejected with `422` before any charge.
- **Idempotency.** Send the same `Idempotency-Key` on a retry to guarantee the message is sent and charged at most once; the original response is replayed.

The message is DKIM-signed with the From domain's key and handed to delivery.

### Response

A successful call returns `202 Accepted`:

```json
{
  "id": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
  "message_id": "<1a2b3c4d@acme.com>",
  "messageId": "<1a2b3c4d@acme.com>",
  "messageIds": ["<1a2b3c4d@acme.com>"],
  "status": "sent",
  "suppressed": ["blocked@example.com"]
}
```

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | CallMissed send id — use it with `GET /api/v1/email/sends` |
| `message_id` | string | RFC 5322 `Message-ID` of the sent message |
| `messageId` | string | Brevo-compatible; equal to `message_id` |
| `messageIds` | string[] | Brevo-compatible; `[message_id]` |
| `status` | string | `sent` when accepted for delivery |
| `suppressed` | string[] | Recipients dropped by your suppression list |

View delivery history: `GET /api/v1/email/sends`. Spend so far: `GET /api/v1/email/usage`.

### Switching from Brevo

The endpoint accepts Brevo `sendTransacEmail` payloads unchanged — `sender`, recipient objects, `replyTo`, `htmlContent`/`textContent`, `attachment` (`url` or `content`), `tags`, and the `Idempotency-Key` header — and returns `messageId` / `messageIds` alongside our native fields. To migrate, point your client at `https://api.callmissed.com/api/v1/email/send` and send `Authorization: Bearer cm_...`.

## Templates

Save a reusable subject + body once, then send it with per-recipient values. Templates are tenant-scoped and managed with the same `cm_` key (email permission).

| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/email/templates` | Create a template (`201`). Duplicate `name` for the same account → `409` |
| `GET /api/v1/email/templates` | List your templates (supports `limit` / `offset`) |
| `GET /api/v1/email/templates/{id}` | Fetch one (`404` if not yours) |
| `PUT /api/v1/email/templates/{id}` | Partial update — only supplied fields change |
| `DELETE /api/v1/email/templates/{id}` | Delete a template (`204`) |

**Create body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | Yes | 1–255 chars; unique per account |
| `subject` | string | Yes | Base subject (overridable per send) |
| `html` | string | Yes | Base HTML body |
| `text` | string | No | Base plain-text body |
| `default_sender` | string | No | Used when the send omits `from` / `sender` |
| `default_reply_to` | string | No | Used when the send omits `reply_to` |
| `tags` | array | No | String tags for your own categorisation |
| `is_active` | boolean | No | Defaults to `true`; an inactive template can't be sent |

The template object returns `id`, `name`, `subject`, `html`, `text`, `default_sender`, `default_reply_to`, `tags`, `is_active`, `created_at`, `updated_at`. The `id` is a UUID.

**Using a template on send.** Add `templateId` and `params` to `POST /api/v1/email/send`:

:::tabs
```bash [cURL]
# Create a template
curl -X POST https://api.callmissed.com/api/v1/email/templates \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "receipt",
    "subject": "Your receipt, {{ params.name }}",
    "html": "<p>Hi {{ params.name }}, your order {{ params.order_id }} is confirmed.</p>",
    "default_sender": "Ops <ops@acme.com>"
  }'

# Send from it — send-call fields override the template
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "templateId": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
    "to": ["Ada <customer@example.com>"],
    "params": { "name": "Ada", "order_id": "1043" }
  }'
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/email"
h = {"Authorization": "Bearer cm_your_key"}

tpl = httpx.post(f"{BASE}/templates", headers=h, json={
    "name": "receipt",
    "subject": "Your receipt, {{ params.name }}",
    "html": "<p>Hi {{ params.name }}, your order {{ params.order_id }} is confirmed.</p>",
    "default_sender": "Ops <ops@acme.com>",
}).json()

httpx.post(f"{BASE}/send", headers=h, json={
    "templateId": tpl["id"],
    "to": ["Ada <customer@example.com>"],
    "params": {"name": "Ada", "order_id": "1043"},
})
```
:::

- **Override rule.** The template's `subject` / `html` / `text` are the base; an explicit `subject` / `html` / `text` / `from` / `reply_to` on the send **wins**. `default_sender` / `default_reply_to` fill in only when the send omits them.
- **Substitution.** `{{ params.KEY }}` (and nested `{{ params.a.b }}`) are replaced from `params`; a missing key renders empty. Values placed into the HTML body are HTML-escaped. This is plain variable substitution — **not** a programming language: no logic, loops, or expressions, and it can only read the `params` you pass.

> **Note:** unlike Brevo's integer template id, a CallMissed `templateId` is a UUID.

## Scheduled Sending

Send a message later by adding `scheduledAt` to `POST /api/v1/email/send`. The send is accepted and enqueued — **no charge and no delivery happen at enqueue time**; billing and delivery occur when it fires at `scheduledAt`.

- `scheduledAt` — ISO-8601 UTC timestamp. Must be in the **future** and **within 72 hours**, else `422`.
- `batchId` — optional UUID you supply to group related scheduled sends; auto-generated if omitted.
- A message cannot be both scheduled **and** a `messageVersions` batch — sending both returns `422`.

A scheduled send returns `202`:

```json
{
  "id": "3f1c9a2d-4b5e-6a7f-8c9d-0e1f2a3b4c5d",
  "status": "scheduled",
  "batchId": "7a2b1c0d-9e8f-4a5b-8c7d-6e5f4a3b2c1d",
  "scheduledAt": "2026-07-28T09:00:00Z"
}
```

Manage scheduled sends by id **or** batchId:

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/email/scheduled/{identifier}` | List the scheduled rows for a send `id` or a `batchId`. An identifier you don't own returns an empty list |
| `DELETE /api/v1/email/scheduled/{identifier}` | Cancel the **pending** scheduled send(s) by id or batchId (`204`). `404` if there's nothing pending to cancel |

Each `ScheduledSendOut` returns `id`, `batchId`, `scheduledAt`, `status` (`pending` / `sent` / `failed` / `cancelled`), `send_id` (the delivered send once it fires), and `createdAt`.

:::tabs
```bash [cURL]
# Schedule a send for later (within 72h)
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Ops <ops@acme.com>",
    "to": ["Ada <customer@example.com>"],
    "subject": "Reminder",
    "text": "Your appointment is tomorrow.",
    "scheduledAt": "2026-07-28T09:00:00Z"
  }'

# Cancel it before it fires — never charged
curl -X DELETE https://api.callmissed.com/api/v1/email/scheduled/3f1c9a2d-4b5e-6a7f-8c9d-0e1f2a3b4c5d \
  -H "Authorization: Bearer cm_your_key"
```
:::

A cancelled send never fires and is never charged.

## Batch (messageVersions)

Send to many recipient sets in one call. Add `messageVersions` — an array where each version is its own recipient set that may override the global subject, body, params, or template.

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
| `params` | object | No | Substitution values for this version |
| `templateId` | string (UUID) | No | Overrides the global template |

- **Base / override.** The top-level `subject` / `html` / `text` / `templateId` / `params` / `from` are the **base** each version overrides. A per-version body override requires a global body to be present; a per-version `templateId` requires a global `templateId`. Global attachments and tags apply to all versions — there are no per-version attachments.
- **Limits** (exceeding any → `422`): ≤99 recipients per version, ≤2000 recipients across the batch (deduped), ≤1000 versions, ≤100 KB per-version `params`, ≤1000 KB `params` across the batch.
- **Billing.** The **deduped union** of recipients across all versions is billed **once** at ₹30/1,000 — a recipient in two versions is billed once. Suppression, quota, and caps are evaluated once over the union.

A batch returns `202` with `messageIds` (one per version, in order) and a `batchId` grouping the batch's sends; `id` / `messageId` are the first version, and `suppressed` lists the union's suppressed addresses. Look up each send with `GET /api/v1/email/sends`.

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Ops <ops@acme.com>",
    "templateId": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
    "subject": "Your receipt, {{ params.name }}",
    "messageVersions": [
      { "to": ["Ada <ada@example.com>"], "params": { "name": "Ada", "order_id": "1043" } },
      { "to": ["Bo <bo@example.com>"], "params": { "name": "Bo", "order_id": "1044" } }
    ]
  }'
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

## Receive Email

Receiving is optional. Publish the domain's **MX** record (returned with its DNS records), then claim receiving addresses. Mail to an unknown address is refused — there is no catch-all.

:::tabs
```bash [cURL]
# Claim support@acme.com on a verified domain
curl -X POST https://api.callmissed.com/api/v1/email/inbound/addresses \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"local_part": "support", "domain_id": "{domain_id}", "forward_url": "https://your-app.com/inbound"}'

# List received messages
curl https://api.callmissed.com/api/v1/email/inbound/messages \
  -H "Authorization: Bearer cm_your_key"
```
:::

| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/email/inbound/addresses` | Claim an address on a verified domain |
| `GET /api/v1/email/inbound/addresses` | List your receiving addresses |
| `DELETE /api/v1/email/inbound/addresses/{id}` | Stop receiving at an address |
| `GET /api/v1/email/inbound/messages` | List received messages |
| `GET /api/v1/email/inbound/messages/{id}` | Read one message (parsed text/HTML) |

`forward_url` is optional — set it to have parsed messages POSTed to your webhook; leave it out to store only and poll the messages endpoints.

## Suppressions

A suppression list per account prevents sending to addresses that hard-bounced or complained. Entries are added automatically from delivery feedback, and you can manage them:

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/email/suppressions` | List suppressed addresses |
| `POST /api/v1/email/suppressions` | Suppress an address manually |
| `DELETE /api/v1/email/suppressions/{id}` | Remove a suppression |

## Pricing

**₹30 per 1,000 emails**, charged per recipient to your credit balance — the same credits as every other API (your signup bonus counts). Only accepted sends are billed; rejected or failed sends cost nothing. See [Credits & Pricing](/docs/credits-rate-limits).

## Errors

| Status | Reason | Meaning |
|--------|--------|---------|
| 402 | `payment_required` | Not enough credit balance to cover the send |
| 403 | `domain_not_verified` | The From domain hasn't passed DNS verification |
| 403 | `domain_paused` | Sending from the domain is paused (reputation breaker) |
| 403 | `email_not_enabled` | The API key lacks the email permission |
| 403 | `all_recipients_suppressed` | Every recipient is on your suppression list — nothing to send |
| 404 | `template_not_found` | The `templateId` doesn't exist or isn't yours |
| 422 | `template_inactive` | The template exists but is not active (`is_active: false`) |
| 422 | `too_many_recipients` | A message or batch union exceeds the recipient limit |
| 422 | `scheduled_batch_unsupported` | A send set both `scheduledAt` and `messageVersions` — pick one |
| 422 | `invalid_attachment` | An attachment is malformed (not exactly one of `url`/`content`, or `content` without `name`), or a recipient address is malformed |
| 422 | `attachment_fetch_failed` | A `url` attachment could not be fetched, or exceeded the size cap |
| 422 | `attachment_url_forbidden` | A `url` attachment points at a blocked (internal/private) address |
| 429 | `rate_limited` | Per-minute send rate for your plan exceeded |
| 429 | `monthly_cap_exceeded` | Monthly send volume for your plan exceeded |
| 429 | `quota_exceeded` | The domain's daily warm-up quota is exhausted |
| 502 | `relay_failed` | The message could not be accepted for delivery; the `id` is returned so you can look it up |