---
title: "Email API"
description: "Send and receive email from your own domain over the API: verified-domain onboarding, DKIM signing, delivery and suppression tracking."
slug: "email"
breadcrumb: "Email"
---

# Email API

Send and receive email from your own domain over the API: verified-domain onboarding, DKIM signing, delivery and suppression tracking.

## Overview

The Email API sends and receives email from a domain you own. You verify the domain once (we generate its DKIM key and the DNS records to publish), then send over the API and, optionally, receive mail at addresses on that domain.

**Base path:** `https://api.callmissed.com/api/v1/email`

Authentication uses your existing CallMissed API key, the same `cm_` key you use for every other API. The key needs the **email** permission enabled (toggle it on the [API keys](https://console.callmissed.com/developer/keys) page). No separate email key.

> **Read this before you write your first send.** Verification registers exactly one sender username on the domain, `donotreply`, so `donotreply@your-domain` always works. Any other local part on a verified domain has to be registered as a sender first: either up front with `POST /api/v1/email/domains/{domain_id}/senders`, or implicitly, because the send path registers the `from` local part on its first refusal and retries. Registration is eventually consistent, so a send from a brand-new sender can still come back as `503 sender_propagating`, meaning retry shortly and nothing else is needed. See [Sender Addresses](/docs/email-domains#sender-addresses).

:::flow
icon:app | Your app | Add a domain, publish the DNS records we generate
icon:gateway | CallMissed | Verify ownership, SPF and both DKIM records, then accept sends from that domain
icon:done | Recipients | Receive DKIM-signed mail from your own domain
:::

> **Billing:** Email is fully credit-based. Every send is charged to your credit balance at **30 credits (₹30) per 1,000 emails** (per recipient), the same wallet as every other API. There is no separate email invoice. Full detail: [Pricing](/docs/email-logs#pricing).

## The pages in this section

:::cards
/docs/email-domains | Domains & Senders | globe | Add a domain, publish DNS, verify, and the donotreply sender rule
/docs/email-send | Send Email | send | POST /api/v1/email/send with every field, header, response and the Brevo migration
/docs/email-templates | Templates | file-text | Reusable subject and body with per-send substitution values
/docs/email-scheduled | Scheduled & Batch Sending | calendar-clock | Send later with scheduledAt, or many recipient sets in one call
/docs/email-inbound | Receive Email | inbox | Claim addresses on a verified domain, read inbound mail, or have it forwarded to your app
/docs/email-logs | Delivery, Suppressions & Usage | chart-column | Send log, suppression list, engagement metrics, spend, and pricing
/docs/email-webhooks | Email Webhooks | webhook | Subscribe your endpoint to bounces and complaints, signed and logged per attempt
/docs/email-limits | Limits, Quotas & Errors | gauge | Send rate, monthly cap, daily quota, and every error shape and reason
:::

## End to end in three calls

Every request below uses the real base URL and the real auth header. Replace `cm_your_key` with your key and `acme.com` with your domain.

:::tabs
```bash [cURL]
# 1. Register the domain - the response carries the DNS records to publish
curl -X POST https://api.callmissed.com/api/v1/email/domains \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"domain": "acme.com"}'

# 2. After publishing every required record, verify it (repeat until verified is true)
curl -X POST https://api.callmissed.com/api/v1/email/domains/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e/verify \
  -H "Authorization: Bearer cm_your_key"

# 3. Send
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Acme Ops <donotreply@acme.com>",
    "to": ["Ada <customer@example.com>"],
    "subject": "Your receipt",
    "text": "Thanks for your order.",
    "html": "<p>Thanks for your order.</p>",
    "reply_to": "support@acme.com"
  }'
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/email"
h = {"Authorization": "Bearer cm_your_key"}

created = httpx.post(f"{BASE}/domains", headers=h, json={"domain": "acme.com"}).json()
for rec in created["dns_records"]:
    print(rec["type"], rec["host"], rec["value"])  # publish these at your DNS host

# once published (repeat until verified is true):
httpx.post(f"{BASE}/domains/{created['domain']['id']}/verify", headers=h)

httpx.post(f"{BASE}/send", headers=h, json={
    "from": "Acme Ops <donotreply@acme.com>",
    "to": ["Ada <customer@example.com>"],
    "subject": "Your receipt",
    "text": "Thanks for your order.",
    "html": "<p>Thanks for your order.</p>",
    "reply_to": "support@acme.com",
})
```
```javascript [JavaScript]
const BASE = "https://api.callmissed.com/api/v1/email";
const headers = {
  Authorization: "Bearer cm_your_key",
  "Content-Type": "application/json",
};

const created = await fetch(`${BASE}/domains`, {
  method: "POST",
  headers,
  body: JSON.stringify({ domain: "acme.com" }),
}).then((r) => r.json());

// publish created.dns_records at your DNS host, then (repeat until verified is true):
await fetch(`${BASE}/domains/${created.domain.id}/verify`, {
  method: "POST",
  headers: { Authorization: "Bearer cm_your_key" },
});

await fetch(`${BASE}/send`, {
  method: "POST",
  headers,
  body: JSON.stringify({
    from: "Acme Ops <donotreply@acme.com>",
    to: ["Ada <customer@example.com>"],
    subject: "Your receipt",
    text: "Thanks for your order.",
    html: "<p>Thanks for your order.</p>",
    reply_to: "support@acme.com",
  }),
});
```
:::

A successful send returns `202 Accepted`:

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

Field-by-field detail for that body is on [Send Email](/docs/email-send#response).

## When a call fails

Error bodies come in four shapes and they are not interchangeable, so branch on the HTTP status first, then check whether `detail` is an object, a string or an array before reaching for `reason`. The shapes, the full reason table, and the three sending ceilings are on [Limits, Quotas & Errors](/docs/email-limits).

Two failures dominate the first send. `403 domain_not_verified` means the domain has not passed all four DNS checks yet. `503 sender_propagating` means the `from` address was just registered as a sender and the mail service has not finished propagating it, so retry shortly. Both are covered on [Sender Addresses](/docs/email-domains#sender-addresses).
