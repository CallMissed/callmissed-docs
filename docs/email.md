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

Provide **at least one** of `text` / `html` (or their Brevo aliases). Provide **exactly one** of `from` / `sender`.

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
| 422 | `invalid_attachment` | An attachment is malformed (not exactly one of `url`/`content`, or `content` without `name`), or a recipient address is malformed |
| 422 | `attachment_fetch_failed` | A `url` attachment could not be fetched, or exceeded the size cap |
| 422 | `attachment_url_forbidden` | A `url` attachment points at a blocked (internal/private) address |
| 429 | `rate_limited` | Per-minute send rate for your plan exceeded |
| 429 | `monthly_cap_exceeded` | Monthly send volume for your plan exceeded |
| 429 | `quota_exceeded` | The domain's daily warm-up quota is exhausted |
| 502 | `relay_failed` | The message could not be accepted for delivery; the `id` is returned so you can look it up |