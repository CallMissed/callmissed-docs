---
title: "Limits, Quotas & Errors"
description: "The three sending ceilings, domain warm-up and reputation pauses, the four error body shapes, and every error reason."
slug: "email-limits"
breadcrumb: "Email"
---

# Limits, Quotas & Errors

The three sending ceilings, domain warm-up and reputation pauses, the four error body shapes, and every error reason.

## Overview

Everything that can stop a send lives here: the three ceilings that govern volume, and the four error shapes plus the full reason table a client has to branch on. Per-message limits (recipients, size, tags, attachments) are under [Send Email](/docs/email-send); batch limits under [Batch (messageVersions)](/docs/email-scheduled#batch-messageversions).

## Limits & Quotas

Three separate ceilings govern sending, and they count three **different** things. A rejection always names which one you hit.

| Plan | Send rate | Recipients per calendar month | Daily-quota ceiling |
|------|-----------|-------------------------------|---------------------|
| Free | 10 requests / min | 2,000 | 200 |
| Starter | 60 requests / min | 50,000 | 5,000 |
| Pro | 300 requests / min | 1,000,000 | 50,000 |
| Enterprise | 1,000 requests / min | Unlimited | 500,000 |

- **Send rate** counts **requests** accepted in the trailing 60 seconds, across your whole account. One call is one request whether it carries 1 recipient or 50, and a `messageVersions` batch is still one request. Exceeding it is `429 rate_limited`.
- **Recipients per calendar month** counts **delivered recipients** in the current calendar month. It resets on the 1st, not on a rolling 30 days. A send is refused up front if it *would* push you past the cap. Exceeding it is `429 monthly_cap_exceeded`.
- **Daily quota** counts **sends**, meaning messages and not recipients, for **one domain** over a rolling 24 hours. Exceeding it is `429 quota_exceeded`.

**Warm-up.** The daily quota is a property of each domain, not of your plan. Every newly verified domain starts at **200 sends / day** and climbs the ladder **200 → 1,000 → 5,000 → 20,000 → 50,000**, at most one step per day, and only after a day that carried real volume with a healthy bounce and complaint rate. Past the top of the ladder the quota keeps **doubling** on each clean day rather than jumping straight to the plan ceiling, so a plan whose ceiling is higher than 50,000 is reached over several more clean days. The plan column above is the ceiling the plan permits; the figure actually enforced is the domain's current `daily_quota`, which `GET /api/v1/email/domains` returns.

**Reputation pause.** A domain whose bounce or complaint rate degrades is paused automatically, whatever the plan or remaining quota. Sends from it then return `403 domain_paused` carrying the reason, and `DomainOut` shows `paused_at` and `pause_reason`.

```bash
# Read the quota actually enforced for each domain
curl https://api.callmissed.com/api/v1/email/domains \
  -H "Authorization: Bearer cm_your_key"
```

## Errors

### Response shapes

Error bodies come in **four** shapes. They are not interchangeable, and a client that always reads `reason` from the same place breaks, most obviously on `relay_failed`, where `reason` sits at the top level rather than under `detail`.

**1. Send rejection: `reason` nested under `detail`.** Everything the send pipeline refuses (every row in the table below except `relay_failed`), plus `403 email_not_enabled`, `422 scheduled_batch_unsupported`, `503 sending_unavailable`, and the `502` from the domain routes:

```json
{
  "detail": {
    "error": "acme.com has not completed DNS verification",
    "reason": "domain_not_verified"
  }
}
```

**2. `502 relay_failed`: flat, and it carries an `id`.** This one is **not** wrapped in `detail`. The send row already exists, so the `id` comes back for you to look up with `GET /api/v1/email/sends`:

```json
{
  "error": "The message could not be accepted for delivery",
  "reason": "relay_failed",
  "id": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e"
}
```

**3. Auth, not-found and conflict: `detail` is a plain string with no `reason` at all.** Covers `401`, the read-only-key `403`, every `404` (domain, template, inbound message, suppression, scheduled send), `409` (duplicate template name, address already claimed), the `400` on a duplicate domain, the `400`s and `422`s on the sender and inbound-address routes, the `502`s from the sender routes, and the `503`s raised when email is not configured:

```json
{ "detail": "A valid API key is required" }
```

**4. Schema-validation `422`: `detail` is an array, with no `reason` key.** Anything rejected before the send pipeline runs: a malformed or control-character-bearing address, a subject or header carrying control characters, an attachment that is not exactly one of `url`/`content` (or `content` without `name`), over-limit `params`, a `scheduledAt` in the past or beyond the 72-hour horizon, and the batch cross-version rules:

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

Handling all four shapes:

:::tabs
```python [Python]
import httpx

r = httpx.post(
    "https://api.callmissed.com/api/v1/email/send",
    headers={"Authorization": "Bearer cm_your_key"},
    json={
        "from": "Acme Ops <donotreply@acme.com>",
        "to": ["Ada <customer@example.com>"],
        "subject": "Your receipt",
        "text": "Thanks for your order.",
    },
)

if r.status_code != 202:
    body = r.json()
    if "reason" in body:                       # shape 2 - relay_failed, flat, has id
        reason, send_id = body["reason"], body.get("id")
    else:
        detail = body.get("detail")
        if isinstance(detail, dict):           # shape 1 - send rejection
            reason, send_id = detail.get("reason"), None
        elif isinstance(detail, list):         # shape 4 - schema validation
            reason, send_id = "validation_error", None
        else:                                  # shape 3 - plain string detail
            reason, send_id = None, None
    print(r.status_code, reason, send_id)
```
```javascript [JavaScript]
const r = await fetch("https://api.callmissed.com/api/v1/email/send", {
  method: "POST",
  headers: {
    Authorization: "Bearer cm_your_key",
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    from: "Acme Ops <donotreply@acme.com>",
    to: ["Ada <customer@example.com>"],
    subject: "Your receipt",
    text: "Thanks for your order.",
  }),
});

if (r.status !== 202) {
  const body = await r.json();
  let reason = null;
  let sendId = null;
  if (body.reason) {
    reason = body.reason;          // shape 2 - relay_failed, flat, has id
    sendId = body.id ?? null;
  } else if (body.detail && typeof body.detail === "object" && !Array.isArray(body.detail)) {
    reason = body.detail.reason;   // shape 1 - send rejection
  } else if (Array.isArray(body.detail)) {
    reason = "validation_error";   // shape 4 - schema validation
  }                                // else shape 3 - plain string detail
  console.log(r.status, reason, sendId);
}
```
:::

### Reasons

| Status | Reason | Meaning |
|--------|--------|---------|
| 401 | *(none, string `detail`)* | Missing, malformed or unrecognised `Authorization` header |
| 402 | `payment_required` | Not enough credit balance to cover the send |
| 403 | *(none, string `detail`)* | The API key is read-only and this route writes |
| 403 | `email_not_enabled` | The API key lacks the email permission |
| 403 | `domain_not_found` | The From domain is not registered on your account at all |
| 403 | `domain_not_verified` | The From domain is registered but hasn't passed verification |
| 403 | `domain_paused` | Sending from the domain is paused (reputation) |
| 403 | `all_recipients_suppressed` | Every recipient is on your suppression list, nothing to send |
| 404 | `template_not_found` | The `templateId` doesn't exist or isn't yours |
| 422 | `no_sender` | Neither `from`/`sender` nor a template `default_sender` supplied one |
| 422 | `invalid_from` | The resolved sender is not a usable email address |
| 422 | `no_recipients` | The resolved recipient list came out empty |
| 422 | `empty_body` | Neither `text` nor `html` (nor a template body) was present |
| 422 | `invalid_headers` | A rendered header value contains invalid characters, usually a template `param` with a newline in it |
| 422 | `unresolvable_template_vars` | The subject or body references `{{ contact.something }}`, a namespace nothing can populate, so it would render empty. Pass the value in `params` instead |
| 422 | `message_too_large` | The assembled message exceeds 25 MB |
| 422 | `template_inactive` | The template exists but is not active (`is_active: false`) |
| 422 | `too_many_recipients` | Over 50 recipients on a single send, or over the union limit on a batch |
| 422 | `scheduled_batch_unsupported` | A send set both `scheduledAt` and `messageVersions`, pick one |
| 422 | `invalid_attachment` | An attachment's `content` is not valid base64 |
| 422 | `attachment_fetch_failed` | A `url` attachment could not be fetched, or exceeded the size cap |
| 422 | `attachment_url_forbidden` | A `url` attachment points at a blocked (internal/private) address, or uses a scheme other than http/https |
| 429 | `rate_limited` | Per-minute request rate for your plan exceeded |
| 429 | `monthly_cap_exceeded` | Monthly recipient volume for your plan exceeded |
| 429 | `quota_exceeded` | The domain's daily send quota is exhausted |
| 502 | `relay_failed` | The message could not be accepted for delivery. Uses the flat shape above |
| 502 | `acs_unavailable` | Domain provisioning or verification is temporarily unavailable, retry the domain call |
| 503 | `sender_propagating` | The `from` address has now been registered as a sender for the domain, but the mail service has not finished propagating it. Retry the send shortly; no further setup is needed. See [Sender Addresses](/docs/email-domains#sender-addresses) |
| 503 | `sending_unavailable` | Sending is temporarily unavailable |

A **malformed recipient address is not `invalid_attachment`**. It is a schema `422` (shape 4), rejected before the send pipeline runs and before any charge.
