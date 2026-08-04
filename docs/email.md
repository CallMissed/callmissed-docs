---
title: "Email API"
description: "Send and receive email from your own domain over the API — verified-domain onboarding, DKIM signing, delivery and suppression tracking."
slug: "email"
breadcrumb: "Email"
---

# Email API

Send and receive email from your own domain over the API — verified-domain onboarding, DKIM signing, delivery and suppression tracking.

## Overview

The Email API sends and receives email from a domain you own. You verify the domain once (we generate its DKIM key and the DNS records to publish), then send over the API and, optionally, receive mail at addresses on that domain.

**Base path:** `https://api.callmissed.com/api/v1/email`

Authentication uses your existing CallMissed API key — the same `cm_` key you use for every other API. The key needs the **email** permission enabled (toggle it on the [API keys](https://app.callmissed.com/api-keys) page). No separate email key.

> **Read this before you write your first send.** The only address you can send **from** is `donotreply@your-domain`. Verifying a domain registers exactly that one sender username on it, and our sending infrastructure refuses any other local part — a `from` of `ops@`, `hello@` or `support@` fails at submission and comes back to you as `502 relay_failed`. Put the address you want humans to answer in `reply_to` instead. See [Sender Addresses](#sender-addresses).

:::flow
icon:app | Your app | Add a domain, publish the DNS records we generate
icon:gateway | CallMissed | Verify ownership, SPF and both DKIM records, then accept sends from that domain
icon:done | Recipients | Receive DKIM-signed mail from your own domain
:::

> **Billing:** Email is fully credit-based. Every send is charged to your credit balance at **30 credits (₹30) per 1,000 emails** (per recipient) — the same wallet as every other API. There is no separate email invoice.

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

The add response returns `domain` plus `dns_records` — the exact set to publish. Use each record's `host` (already formatted the way DNS panels want it: `@` for the apex, a bare label for a subdomain) and its `value` verbatim.

| Record | Required | Purpose |
|--------|----------|---------|
| `TXT` (ownership) | Yes | A one-off token proving you control the domain |
| `TXT` (SPF) | Yes | Authorises our sending infrastructure to send for the domain |
| `CNAME` (DKIM) | Yes | Delegates the DKIM signing key for the domain to us |
| `CNAME` (DKIM2) | Yes | A second delegated DKIM key, used for key rotation |
| `MX` | No | Routes inbound mail to us (only needed to **receive**) |

DKIM is **delegated by `CNAME`**, not published as a `TXT` key. The exact host and value of every record are generated per domain — read them from `dns_records` rather than hard-coding them. If your DNS panel refuses the `CNAME`, you almost certainly have a conflicting record at that name already.

**No DMARC record is generated.** DMARC is worth publishing and we recommend it, but you author `_dmarc.your-domain` yourself — it is not in the returned set and not part of verification.

Verification covers **four** checks — domain ownership, SPF, DKIM and DKIM2 — and a domain may send only once **all four** report verified. `POST /domains/{id}/verify` returns `verified` (a single boolean over all four) plus a `checks` array with one entry per check, so a partial pass tells you exactly which record has not propagated yet. Propagation is not instant; call verify again until `verified` is `true`.

A newly verified domain starts on a warm-up quota that rises automatically as it sends clean volume — see [Limits & Quotas](#limits--quotas).

**Managing domains:**

| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/email/domains` | Register a domain (`201`) and get its DNS records. A domain already on your account → `400` |
| `GET /api/v1/email/domains` | List your domains, newest first, as `DomainOut` |
| `GET /api/v1/email/domains/{id}/records` | Re-fetch a domain's DNS records at any time — the same set the create call returned |
| `GET /api/v1/email/domains/{id}/provider` | Detect the domain's DNS host from its nameservers and return a deep link to the right DNS-management page |
| `POST /api/v1/email/domains/{id}/verify` | Run verification and return the per-check states |
| `DELETE /api/v1/email/domains/{id}` | Remove a domain (`204`). Sending from it stops immediately |

A domain id that isn't yours returns `404` with `{"detail": "Domain not found"}`.

`DomainOut` returns `id`, `domain`, `status` (`pending` / `verified` / `failed` / `paused`), `dkim_selector`, `daily_quota`, `verified_at`, `last_checked_at`, `paused_at`, `pause_reason`, `sent_count`, `bounce_count`, `complaint_count`, `created_at`.

`DnsRecordOut` returns `type`, `name` (the full FQDN), `host` (the panel-ready name), `value`, `purpose`, `required`, and `priority` (MX only; `null` otherwise).

`DnsProviderOut` returns `detected`, `provider_id`, `provider_name`, `manage_url`, `domain_specific`, and `nameservers`.

## Sender Addresses

**Every message is sent from `donotreply@your-domain`.** Verifying a domain registers exactly one sender username on it — `donotreply` — and our sending infrastructure only accepts a message whose sender local part is a registered username. Anything else is refused at submission and reaches you as `502 relay_failed`.

```json
{ "from": "Acme <donotreply@acme.com>" }
```

The local part is matched case-insensitively, so `DoNotReply@acme.com` works too. The **display name is entirely yours** — `"Acme Billing <donotreply@acme.com>"` is what a recipient's mail client shows first, so the mailbox name is rarely the part they read.

**To give people a real address to answer, set `reply_to`.** It is an ordinary header with no sender registration behind it, so it can be any address at all, including a mailbox on another provider:

```json
{
  "from": "Acme Support <donotreply@acme.com>",
  "reply_to": "support@acme.com"
}
```

If you want those replies to come back through the API, claim the same address as a [receiving address](#receive-email).

**Registering additional sender addresses is not self-serve today.** There is no API or dashboard control for it. If a second sender username is a hard requirement, contact [support@callmissed.com](mailto:support@callmissed.com) — it is handled manually.

What this constraint is **not**: the sending **domain** is entirely yours (any verified domain on your account), and `to`, `cc`, `bcc` and `reply_to` are unrestricted. Only the sender's local part is fixed.

## Send Email

**Endpoint:** `POST /api/v1/email/send`

The send body is a **superset**: it accepts both the original CallMissed shapes (string `from`, string-list `to`, `text`/`html`) and Brevo-style shapes (`sender` object, recipient objects, `textContent`/`htmlContent`). A Brevo `sendTransacEmail` integration works here by changing only the base URL and the auth header — see [Switching from Brevo](#switching-from-brevo) below.

Note the `from` in every example below: the sender local part is always `donotreply`, and the human address lives in `reply_to`. Any other local part is rejected by our sending infrastructure — see [Sender Addresses](#sender-addresses).

The example below uses `cc`, one URL attachment and one base64 attachment, `tags`, and the `Idempotency-Key` header:

:::tabs
```bash [cURL]
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: order-1043-receipt" \
  -d '{
    "from": "Acme Ops <donotreply@acme.com>",
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
        "from": "Acme Ops <donotreply@acme.com>",
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
    from: "Acme Ops <donotreply@acme.com>",
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
| `from` | string | one of `from` / `sender` | Sender as `"Name <donotreply@acme.com>"` or bare address. The domain must be one of your verified domains and the local part **must** be `donotreply` — see [Sender Addresses](#sender-addresses) |
| `sender` | object | one of `from` / `sender` | Brevo-style sender `{"email", "name"}` — an alternative to `from`. Same `donotreply` rule applies |
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
- **Recipient limit: 50 per message.** A single (non-batch) send accepts at most **50** recipients, counted across `to` + `cc` + `bcc` **after** de-duplication and suppression filtering — so 60 addresses of which 12 are suppressed and 3 are duplicates does pass. Over the limit is `422 too_many_recipients`. Batches have their own, larger limits; see [Batch (messageVersions)](#batch-messageversions).
- **Size limit: 25 MB per message.** The fully assembled message — headers, both bodies, and every attachment **after base64 encoding** — must stay under 25 MB, else `422 message_too_large`. Base64 inflates attachment bytes by roughly 1.37x, so the practical raw-attachment budget is nearer **18 MB**, less whatever the bodies take. The same 25 MB figure caps a `url` attachment while it is being fetched.
- **Validation.** Recipient addresses are validated first — a malformed address, or one containing control characters, is rejected with `422` before any charge. That rejection is a **schema** error, so its body is the validation-array shape, not `{"reason": …}` — see [Errors](#errors).
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

View delivery history and spend: see [Delivery Log & Usage](#delivery-log--usage).

### Switching from Brevo

The endpoint accepts Brevo `sendTransacEmail` payloads unchanged — `sender`, recipient objects, `replyTo`, `htmlContent`/`textContent`, `attachment` (`url` or `content`), `tags`, and the `Idempotency-Key` header — and returns `messageId` / `messageIds` alongside our native fields. To migrate, point your client at `https://api.callmissed.com/api/v1/email/send` and send `Authorization: Bearer cm_...`.

Two things do **not** carry over unchanged. Your Brevo `sender` must become `donotreply@your-verified-domain` (see [Sender Addresses](#sender-addresses)), and a single send here is capped at 50 recipients rather than Brevo's higher per-message limit — split a larger list across calls or use [messageVersions](#batch-messageversions).

## Limits & Quotas

Three separate ceilings govern sending, and they count three **different** things. A rejection always names which one you hit.

| Plan | Send rate | Recipients per calendar month | Daily-quota ceiling |
|------|-----------|-------------------------------|---------------------|
| Free | 10 requests / min | 2,000 | 200 |
| Starter | 60 requests / min | 50,000 | 5,000 |
| Pro | 300 requests / min | 1,000,000 | 50,000 |
| Enterprise | 1,000 requests / min | Unlimited | 500,000 |

- **Send rate** counts **requests** accepted in the trailing 60 seconds, across your whole account. One call is one request whether it carries 1 recipient or 50, and a `messageVersions` batch is still one request. Exceeding it is `429 rate_limited`.
- **Recipients per calendar month** counts **delivered recipients** in the current calendar month — it resets on the 1st, not on a rolling 30 days. A send is refused up front if it *would* push you past the cap. Exceeding it is `429 monthly_cap_exceeded`.
- **Daily quota** counts **sends** — messages, not recipients — for **one domain** over a rolling 24 hours. Exceeding it is `429 quota_exceeded`.

**Warm-up.** The daily quota is a property of each domain, not of your plan. Every newly verified domain starts at **200 sends / day** and climbs the ladder **200 → 1,000 → 5,000 → 20,000 → 50,000**, at most one step per day, and only after a day that carried real volume with a healthy bounce and complaint rate. The plan column above is the ceiling the plan permits; the figure actually enforced is the domain's current `daily_quota`, which `GET /api/v1/email/domains` returns.

**Reputation pause.** A domain whose bounce or complaint rate degrades is paused automatically, whatever the plan or remaining quota. Sends from it then return `403 domain_paused` carrying the reason, and `DomainOut` shows `paused_at` and `pause_reason`.

Per-message limits (recipients, size, tags, attachments) are under [Send Email](#send-email); batch limits under [Batch (messageVersions)](#batch-messageversions).

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
    "default_sender": "Acme Ops <donotreply@acme.com>"
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
    "default_sender": "Acme Ops <donotreply@acme.com>",
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
    "from": "Acme Ops <donotreply@acme.com>",
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
- **Limits** (exceeding any → `422`): ≤99 recipients per version, ≤2000 recipients across the batch (deduped), ≤1000 versions, ≤100 KB per-version `params`, ≤1000 KB `params` across the batch. The 50-recipient single-send cap does **not** apply here — the batch union cap replaces it.
- **A recipient is delivered by exactly one version — the first.** Versions are processed in array order, and each version delivers only the recipients no earlier version already claimed. If `ada@example.com` appears in version 1 **and** version 2, she receives **version 1's** subject, body and params, and version 2 simply does not send to her at all. This is a delivery outcome, not only a billing rule: repeating an address across versions silently drops the later content. Keep each version's recipient set disjoint.
- **Billing.** The **deduped union** of recipients across all versions is billed **once** at 30 credits (₹30) per 1,000 — a recipient in two versions is billed once, matching the delivery rule above. Suppression, quota, rate and monthly-cap checks are likewise evaluated once, over the union.
- **A partial failure still returns `202`.** The response `status` is `sent` when **any** version was accepted for delivery; only an all-versions-failed batch returns `502 relay_failed`. `messageIds` carries one id per version in array order **whether or not that version was accepted**, so the response alone cannot tell you which versions failed. To find out, list the sends and read each row's `status`.

A batch returns `202` with `messageIds` (one per version, in order) and a `batchId` grouping the batch's sends; `id` / `messageId` are the first version, and `suppressed` lists the union's suppressed addresses. Look up each send with `GET /api/v1/email/sends` — see [Delivery Log & Usage](#delivery-log--usage).

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

A suppression's `reason` is one of `hard_bounce`, `complaint`, `manual`, or `unsubscribe`; an unknown reason on `POST` is `422`. `SuppressionOut` returns `id`, `address`, `reason`, `detail`, and `created_at`.

## Delivery Log & Usage

**`GET /api/v1/email/sends`** — your send log, newest first. Query params: `limit` (1–200, default 50) and `offset` (≥0, default 0). Returns an array of `SendOut`:

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

One row per **message**, so a `messageVersions` batch writes one row per version — this is how you find out which versions of a batch failed.

**`GET /api/v1/email/usage`** — spend for the account. Cost is summed from the price stamped on each send at send time, so a later price change never rewrites history. Returns `UsageOut`:

| Field | Type | Notes |
|-------|------|-------|
| `currency` | string | `INR` |
| `price_per_1000` | number | Current price per 1,000 emails |
| `billed_sends` | integer | Sends that were actually charged, all time |
| `total_cost` | number | All-time spend in whole currency units |
| `sends_30d` | integer | Billed sends in the trailing 30 days |
| `cost_30d` | number | Spend in the trailing 30 days |

Both endpoints are reads, so a read-only key works.

## Pricing

**30 credits (₹30) per 1,000 emails**, charged per recipient to your credit balance — the same credits as every other API (your signup bonus counts). Only accepted sends are billed; rejected or failed sends cost nothing. See [Credits & Pricing](/docs/credits-rate-limits).

## Errors

### Response shapes

Error bodies come in **four** shapes. They are not interchangeable, and a client that always reads `reason` from the same place breaks — most obviously on `relay_failed`, where `reason` sits at the top level rather than under `detail`.

**1. Send rejection — `reason` nested under `detail`.** Everything the send pipeline refuses (every row in the table below except `relay_failed`), plus `403 email_not_enabled`, `422 scheduled_batch_unsupported`, `503 sending_unavailable`, and the `502` from the domain routes:

```json
{
  "detail": {
    "error": "acme.com has not completed DNS verification",
    "reason": "domain_not_verified"
  }
}
```

**2. `502 relay_failed` — flat, and it carries an `id`.** This one is **not** wrapped in `detail`. The send row already exists, so the `id` comes back for you to look up with `GET /api/v1/email/sends`:

```json
{
  "error": "The message could not be accepted for delivery",
  "reason": "relay_failed",
  "id": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e"
}
```

**3. Auth, not-found and conflict — `detail` is a plain string with no `reason` at all.** Covers `401`, the read-only-key `403`, every `404` (domain, template, inbound message, suppression, scheduled send), `409` (duplicate template name, address already claimed), the `400` on a duplicate domain, and the `503`s raised when email is not configured:

```json
{ "detail": "A valid API key is required" }
```

**4. Schema-validation `422` — `detail` is an array, with no `reason` key.** Anything rejected before the send pipeline runs: a malformed or control-character-bearing address, a subject or header carrying control characters, an attachment that is not exactly one of `url`/`content` (or `content` without `name`), over-limit `params`, a `scheduledAt` in the past or beyond the 72-hour horizon, and the batch cross-version rules:

```json
{
  "detail": [
    {
      "type": "value_error",
      "loc": ["body"],
      "msg": "Value error, invalid email address: 'not-an-email'",
      "input": { "...": "..." }
    }
  ]
}
```

When handling errors, branch on the HTTP status first, then check whether `detail` is an object, a string or an array before reaching for `reason`.

### Reasons

| Status | Reason | Meaning |
|--------|--------|---------|
| 401 | *(none — string `detail`)* | Missing, malformed or unrecognised `Authorization` header |
| 402 | `payment_required` | Not enough credit balance to cover the send |
| 403 | *(none — string `detail`)* | The API key is read-only and this route writes |
| 403 | `email_not_enabled` | The API key lacks the email permission |
| 403 | `domain_not_found` | The From domain is not registered on your account at all |
| 403 | `domain_not_verified` | The From domain is registered but hasn't passed verification |
| 403 | `domain_paused` | Sending from the domain is paused (reputation) |
| 403 | `all_recipients_suppressed` | Every recipient is on your suppression list — nothing to send |
| 404 | `template_not_found` | The `templateId` doesn't exist or isn't yours |
| 422 | `no_sender` | Neither `from`/`sender` nor a template `default_sender` supplied one |
| 422 | `invalid_from` | The resolved sender is not a usable email address |
| 422 | `no_recipients` | The resolved recipient list came out empty |
| 422 | `empty_body` | Neither `text` nor `html` (nor a template body) was present |
| 422 | `invalid_headers` | A rendered header value contains invalid characters — usually a template `param` with a newline in it |
| 422 | `message_too_large` | The assembled message exceeds 25 MB |
| 422 | `template_inactive` | The template exists but is not active (`is_active: false`) |
| 422 | `too_many_recipients` | Over 50 recipients on a single send, or over the union limit on a batch |
| 422 | `scheduled_batch_unsupported` | A send set both `scheduledAt` and `messageVersions` — pick one |
| 422 | `invalid_attachment` | An attachment's `content` is not valid base64 |
| 422 | `attachment_fetch_failed` | A `url` attachment could not be fetched, or exceeded the size cap |
| 422 | `attachment_url_forbidden` | A `url` attachment points at a blocked (internal/private) address, or uses a scheme other than http/https |
| 429 | `rate_limited` | Per-minute request rate for your plan exceeded |
| 429 | `monthly_cap_exceeded` | Monthly recipient volume for your plan exceeded |
| 429 | `quota_exceeded` | The domain's daily send quota is exhausted |
| 502 | `relay_failed` | The message could not be accepted for delivery. **The most common cause is a sender local part other than `donotreply`** — see [Sender Addresses](#sender-addresses). Uses the flat shape above |
| 502 | `acs_unavailable` | Domain provisioning or verification is temporarily unavailable — retry the domain call |
| 503 | `sending_unavailable` | Sending is temporarily unavailable |

A **malformed recipient address is not `invalid_attachment`** — it is a schema `422` (shape 4), rejected before the send pipeline runs and before any charge.
