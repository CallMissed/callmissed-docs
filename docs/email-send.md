---
title: "Send Email"
description: "POST /api/v1/email/send: every field, attachment rule, header, semantic and response, plus the drop-in Brevo migration."
slug: "email-send"
breadcrumb: "Email"
---

# Send Email

POST /api/v1/email/send: every field, attachment rule, header, semantic and response, plus the drop-in Brevo migration.

## Send Email

**Endpoint:** `POST /api/v1/email/send`

The send body is a **superset**: it accepts both the original CallMissed shapes (string `from`, string-list `to`, `text`/`html`) and Brevo-style shapes (`sender` object, recipient objects, `textContent`/`htmlContent`). A Brevo `sendTransacEmail` integration works here by changing only the base URL and the auth header. See [Switching from Brevo](#switching-from-brevo) below.

Note the `from` in every example below. `donotreply@your-verified-domain` is registered as a sender by verification, so it always works, and the address you want humans to answer goes in `reply_to`. Any other local part must be a registered sender on the domain; the send path registers it on first use and retries, which can surface as `503 sender_propagating` until the registration is live. See [Sender Addresses](/docs/email-domains#sender-addresses).

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

## Fields

An **address** may be written three ways, and you can mix them within one array: a bare `"a@b.com"`, a display form `"Name <a@b.com>"`, or an object `{"email": "a@b.com", "name": "Name"}`.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `from` | string | one of `from` / `sender`, unless a template supplies `default_sender` | Sender as `"Name <donotreply@acme.com>"` or bare address. The domain must be one of your verified domains and the local part must be a registered sender on it. See [Sender Addresses](/docs/email-domains#sender-addresses) |
| `sender` | object | one of `from` / `sender`, unless a template supplies `default_sender` | Brevo-style sender `{"email", "name"}`, an alternative to `from`. Same registered-sender rule applies |
| `to` | array | Yes | One or more recipient addresses (min 1). Each delivered recipient is billed |
| `cc` | array | No | Carbon-copy recipients. Appear in the `Cc` header **and** are delivered |
| `bcc` | array | No | Blind-copy recipients. Delivered but **never** written to any header |
| `subject` | string | No | Up to 998 characters |
| `text` | string | body required | Plain-text body. Omit it on an HTML send and one is [generated for you](#automatic-plain-text); send `""` to opt out and ship HTML only |
| `html` | string | body required | HTML body |
| `textContent` | string | body required | Brevo alias for `text` |
| `htmlContent` | string | body required | Brevo alias for `html` |
| `reply_to` | string | No | Reply-To address as a string |
| `replyTo` | string \| object | No | Brevo alias for `reply_to`, string or `{"email", "name"}` |
| `headers` | object | No | Extra headers as string→string. Reserved headers (`From`, `To`, `Cc`, `Bcc`, `Reply-To`, `Subject`, `Date`, `Message-ID`, `DKIM-Signature`, `Received`) are ignored |
| `attachment` | array | No | Attachments, and [inline images](#inline-images); also accepted as `attachments`. See below |
| `tags` | array | No | Up to 10 tags for your own categorisation. Each is either a plain string or a [`{name, value}` object](#tags); trimmed, empties dropped |
| `templateId` | string (UUID) | No | Send from a saved template: its subject/body are the base; explicit send fields override. See [Templates](/docs/email-templates) |
| `params` | object | No | Substitution values for `{{ params.KEY }}` placeholders. They are applied to the template's subject and bodies **and** to any inline `subject` / `text` / `html` you send, so `params` works with no `templateId` at all. Capped at 100 KB of JSON |
| `scheduledAt` | string | No | ISO-8601 UTC timestamp to send later (future, within 72h). See [Scheduled Sending](/docs/email-scheduled) |
| `batchId` | string (UUID) | No | Groups related scheduled sends; auto-generated if omitted |
| `messageVersions` | array | No | One call, many recipient sets, each overrides the base. See [Batch (messageVersions)](/docs/email-scheduled#batch-messageversions) |

Provide **at least one** of `text` / `html` (or their Brevo aliases), a `templateId`, or `messageVersions`. Provide **exactly one** of `from` / `sender`. A top-level `to` is not required when `messageVersions` is present.

**Attachments.** Each item in `attachment` is **either** a URL reference **or** inline base64, exactly one of the two:

| Attachment field | Type | Required | Notes |
|------------------|------|----------|-------|
| `url` | string | one of `url` / `content` | `http`/`https` URL fetched at send time. Internal/private URLs are refused; the fetched file is size-capped |
| `content` | string | one of `url` / `content` | Base64-encoded file bytes |
| `name` | string | with `content` | Filename (≤255 chars). Required when `content` is set; optional with `url` |
| `content_id` | string | No | Makes the part an [inline image](#inline-images) your HTML references as `<img src="cid:THE_ID">`. Also accepted as `contentId`. Up to 255 chars, and only letters, numbers, `.`, `-`, `_` and `@`; anything else is a `422` |
| `disposition` | string | No | `inline` or `attachment`, to be explicit. Omitted, a part with a `content_id` is inline and everything else is an attachment |

### Inline images

Give an attachment a `content_id` and reference that id from the HTML body as `cid:`. The part is embedded where you placed it instead of arriving as a download:

```bash
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Acme Ops <donotreply@acme.com>",
    "to": ["customer@example.com"],
    "subject": "Your receipt",
    "html": "<p>Thanks for your order.</p><img src=\"cid:logo@acme\" alt=\"Acme\" width=\"120\">",
    "attachment": [
      {
        "name": "logo.png",
        "content": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8z8DwHwAFAAH/q842iQAAAABJRU5ErkJggg==",
        "content_id": "logo@acme"
      }
    ]
  }'
```

A `url` attachment can be inline too: set `content_id` on it and the fetched file is embedded the same way.

The one case `content_id` alone cannot express is a part that has a content id **and** should still appear as a normal downloadable attachment. Set `"disposition": "attachment"` for that; an explicit `disposition` always wins.

Inline parts count toward the 25 MB message ceiling like any other attachment, and `cid:` references in your HTML are never rewritten by [click tracking](#open-and-click-tracking).

### Tags

A tag is either a plain string, as before, or a `{name, value}` object when you need to carry a value:

```json
{
  "tags": [
    "receipt",
    { "name": "order_id", "value": "1043" },
    { "name": "campaign", "value": "spring-sale" }
  ]
}
```

Both shapes can be mixed in the same array. A structured tag's `name` and `value` are required, may contain only ASCII letters, numbers, underscores and dashes, and are capped at 256 characters each; anything else is a `422`. Up to 10 tags per send.

On the wire, tag **names** are joined into an `X-Tags` header exactly as before, and each structured tag additionally gets its own `X-Tag-<name>: <value>` header so the value survives onto the message. The authoritative copy is the one stored against the send.

Plain-string tags behave exactly as they did, so existing calls need no change.

### Automatic plain text

Send `html` with no `text` and a plain-text alternative is generated from your HTML. A message with no text part reads badly in text-only clients and scores worse with spam filters, so this is the default.

Link destinations are kept alongside their label as `label (https://url)`, and block-level markup becomes line breaks so the text keeps the shape of the document. Scripts and styles are dropped entirely.

It is a best-effort reading of your HTML, never an exact rendering. Two ways to take control:

- **Supply `text` yourself** for exact copy. Anything you send is used as-is.
- **Send `"text": ""`** to opt out and ship an HTML-only message. An empty string is treated as a deliberate choice, not an omission.

### Open and click tracking

Tracking is **per sending domain and off by default**. Turn it on with [`PATCH /api/v1/email/domains/{id}`](/docs/email-domains#tracking-and-sending-toggles):

```bash
curl -X PATCH https://api.callmissed.com/api/v1/email/domains/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"open_tracking": true, "click_tracking": true}'
```

Once a domain opts in, every HTML send from it is rewritten as it goes out:

- **Opens** append a 1x1 pixel at the end of the HTML body. It is marked `aria-hidden` with an empty `alt`, so a screen reader does not announce it, and it never displaces visible content.
- **Clicks** rewrite `http`/`https` links to a signed redirect that forwards the recipient to the original destination. `mailto:`, `tel:` and `cid:` links are left alone, as are unquoted `href` attributes, so nothing in your markup is mangled.

Only the HTML part is tracked; the plain-text part always keeps the real destinations. Both toggles are independent, and whether a send carried tracking is recorded at send time, so [metrics](/docs/email-logs#engagement-metrics) stay meaningful across a window where you flipped a toggle.

Read the results from [engagement metrics](/docs/email-logs#engagement-metrics).

**Headers.**

| Header | Notes |
|--------|-------|
| `Authorization` | `Bearer cm_...`, the key needs the **email** permission (required) |
| `Idempotency-Key` | Optional. A repeat with the same key returns the first send's result without sending or charging again |

## Semantics

- **cc vs bcc.** `cc` recipients are written to the `Cc` header and delivered; `bcc` recipients are delivered but never appear in any header.
- **De-duplication.** Each of `to`, `cc` and `bcc` is de-duplicated case-insensitively, then the three are merged into one recipient set. An address listed in both `to` and `cc` is dropped from the `Cc` header and delivered, and billed, once; a `bcc` address already covered by `to` or `cc` is likewise dropped.
- **Suppression.** Recipients on your [suppression list](/docs/email-logs#suppressions) are dropped from `to`, `cc`, and `bcc` before sending, and returned in `suppressed`.
- **Recipient limit: 50 per message.** A single (non-batch) send accepts at most **50** recipients, counted across `to` + `cc` + `bcc` **after** de-duplication and suppression filtering, so 60 addresses of which 12 are suppressed and 3 are duplicates does pass. Over the limit is `422 too_many_recipients`. Batches have their own, larger limits; see [Batch (messageVersions)](/docs/email-scheduled#batch-messageversions).
- **Size limit: 25 MB per message.** The fully assembled message (headers, both bodies, and every attachment **after base64 encoding**) must stay under 25 MB, else `422 message_too_large`. Base64 inflates attachment bytes by roughly 1.37x, so the practical raw-attachment budget is nearer **18 MB**, less whatever the bodies take. The same 25 MB figure caps a `url` attachment while it is being fetched.
- **Validation.** Recipient addresses are validated first: a malformed address, or one containing control characters, is rejected with `422` before any charge. That rejection is a **schema** error, so its body is the validation-array shape, not `{"reason": …}`. See [Errors](/docs/email-limits#response-shapes).
- **Idempotency.** Send the same `Idempotency-Key` on a retry to guarantee the message is sent and charged at most once; the original response is replayed.

The message is DKIM-signed with the From domain's key and handed to delivery.

## Response

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
| `id` | string | CallMissed send id, use it with `GET /api/v1/email/sends` |
| `message_id` | string | RFC 5322 `Message-ID` of the sent message |
| `messageId` | string | Brevo-compatible; equal to `message_id` |
| `messageIds` | string[] | Brevo-compatible; `[message_id]` |
| `status` | string | `sent` when accepted for delivery |
| `suppressed` | string[] | Recipients dropped by your suppression list |

View delivery history and spend: see [Delivery Log & Usage](/docs/email-logs).

## Common send failures

`relay_failed` is flat (no `detail` wrapper) and carries the `id` of the send row; every other reason here is nested under `detail`; schema rejections are an array under `detail`. Full shapes and the complete table: [Limits, Quotas & Errors](/docs/email-limits).

| Status | Reason | Meaning |
|--------|--------|---------|
| 402 | `payment_required` | Not enough credit balance to cover the send |
| 403 | `email_not_enabled` | The API key lacks the email permission |
| 403 | `domain_not_verified` | The From domain is registered but hasn't passed verification |
| 403 | `all_recipients_suppressed` | Every recipient is on your suppression list |
| 422 | `empty_body` | Neither `text` nor `html` (nor a template body) was present |
| 422 | `too_many_recipients` | Over 50 recipients on a single send |
| 422 | `message_too_large` | The assembled message exceeds 25 MB |
| 422 | `unresolvable_template_vars` | The subject or body references `{{ contact.something }}`, which nothing can populate. Pass the value in `params` instead |
| 429 | `rate_limited` / `monthly_cap_exceeded` / `quota_exceeded` | A plan or domain ceiling was hit |
| 502 | `relay_failed` | The message could not be accepted for delivery |
| 503 | `sender_propagating` | The `from` address was just registered as a sender and is not live yet. Retry shortly; no further setup is needed |

## Switching from Brevo

The endpoint accepts Brevo `sendTransacEmail` payloads unchanged (`sender`, recipient objects, `replyTo`, `htmlContent`/`textContent`, `attachment` with `url` or `content`, `tags`, and the `Idempotency-Key` header) and returns `messageId` / `messageIds` alongside our native fields. To migrate, point your client at `https://api.callmissed.com/api/v1/email/send` and send `Authorization: Bearer cm_...`.

```bash
curl -X POST https://api.callmissed.com/api/v1/email/send \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "sender": { "email": "donotreply@acme.com", "name": "Acme Ops" },
    "to": [{ "email": "customer@example.com", "name": "Ada" }],
    "subject": "Your receipt",
    "htmlContent": "<p>Thanks for your order.</p>",
    "textContent": "Thanks for your order.",
    "replyTo": { "email": "support@acme.com", "name": "Acme Support" }
  }'
```

Two things do **not** carry over unchanged. Your Brevo `sender` must be an address on a verified domain of yours, and its local part must be a registered sender (see [Sender Addresses](/docs/email-domains#sender-addresses)). And a single send here is capped at 50 recipients rather than Brevo's higher per-message limit, so split a larger list across calls or use [messageVersions](/docs/email-scheduled#batch-messageversions).
