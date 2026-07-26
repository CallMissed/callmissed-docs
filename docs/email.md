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

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Ops <ops@acme.com>",
    "to": ["customer@example.com"],
    "subject": "Welcome aboard",
    "text": "Thanks for signing up!",
    "html": "<p>Thanks for signing up!</p>"
  }'
```
```python [Python]
import httpx

httpx.post(
    "https://api.callmissed.com/api/v1/email/send",
    headers={"Authorization": "Bearer cm_your_key"},
    json={
        "from": "Ops <ops@acme.com>",
        "to": ["customer@example.com"],
        "subject": "Welcome aboard",
        "text": "Thanks for signing up!",
        "html": "<p>Thanks for signing up!</p>",
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
    from: "Ops <ops@acme.com>",
    to: ["customer@example.com"],
    subject: "Welcome aboard",
    text: "Thanks for signing up!",
    html: "<p>Thanks for signing up!</p>",
  }),
});
```
:::

| Field | Type | Notes |
|-------|------|-------|
| `from` | string | Must be an address on one of your verified domains |
| `to` | string[] | One or more recipients (each is billed) |
| `subject` | string | |
| `text` | string | Plain-text body (provide `text`, `html`, or both) |
| `html` | string | HTML body |
| `reply_to` | string | Optional |
| `headers` | object | Optional extra headers (reserved headers are ignored) |

The message is DKIM-signed with the From domain's key and handed to delivery. Suppressed recipients are dropped automatically. The response returns the message id, status, and any suppressed recipients.

View delivery history: `GET /api/v1/email/sends`. Spend so far: `GET /api/v1/email/usage`.

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
| 429 | `rate_limited` | Per-minute send rate for your plan exceeded |
| 429 | `monthly_cap_exceeded` | Monthly send volume for your plan exceeded |
| 429 | `quota_exceeded` | The domain's daily warm-up quota is exhausted |