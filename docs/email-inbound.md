---
title: "Receive Email"
description: "Publish the MX record, claim receiving addresses on a verified domain, and read or forward inbound mail."
slug: "email-inbound"
breadcrumb: "Email"
---

# Receive Email

Publish the MX record, claim receiving addresses on a verified domain, and read or forward inbound mail.

## Receive Email

Receiving is optional. Publish the domain's **MX** record (returned with its [DNS records](/docs/email-domains#add--verify-a-domain)), then claim receiving addresses. Mail to an unknown address is refused; there is no catch-all.

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
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/email"
h = {"Authorization": "Bearer cm_your_key"}

httpx.post(f"{BASE}/inbound/addresses", headers=h, json={
    "local_part": "support",
    "domain_id": "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
    "forward_url": "https://your-app.com/inbound",
})

messages = httpx.get(f"{BASE}/inbound/messages", headers=h).json()
```
```javascript [JavaScript]
const BASE = "https://api.callmissed.com/api/v1/email";
const headers = {
  Authorization: "Bearer cm_your_key",
  "Content-Type": "application/json",
};

await fetch(`${BASE}/inbound/addresses`, {
  method: "POST",
  headers,
  body: JSON.stringify({
    local_part: "support",
    domain_id: "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e",
    forward_url: "https://your-app.com/inbound",
  }),
});

const messages = await fetch(`${BASE}/inbound/messages`, {
  headers: { Authorization: "Bearer cm_your_key" },
}).then((r) => r.json());
```
:::

## Endpoints

| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/email/inbound/addresses` | Claim an address on a verified domain |
| `GET /api/v1/email/inbound/addresses` | List your receiving addresses |
| `DELETE /api/v1/email/inbound/addresses/{id}` | Stop receiving at an address |
| `GET /api/v1/email/inbound/messages` | List received messages, newest first (`limit` / `offset`) |
| `GET /api/v1/email/inbound/messages/{id}` | Read one message (parsed text/HTML) |

Mail arriving for an address you have not claimed is refused at the door, so nothing is stored for it.

### Claim an address

**`POST /api/v1/email/inbound/addresses`**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `local_part` | string | Yes | The mailbox name to claim, 1–64 chars, for example `support` for `support@acme.com`. A bare name only: `@` or `/` in it is a `422`. Lower-cased before the address is built |
| `domain_id` | string (UUID) | Yes | The id of one of your **verified** domains. An unverified domain is a `400` |
| `forward_url` | string | No | Up to 2048 chars, and it must be a public `http`/`https` URL: an internal or private target is refused at write time with a `400`. Every message that arrives at this address is [POSTed to it](#forwarding-to-your-app) |

An inbound address is **globally unique**: one mailbox per address across the whole platform, so a `local_part` already claimed on that domain returns `409`.

`InboundAddressOut` returns `id`, `address` (the full `local_part@domain`), `domain_id`, `forward_url`, `is_active` and `created_at`.

### Forwarding to your app

Set `forward_url` on an address and every message that arrives there is POSTed to it as JSON, so you do not have to poll the messages endpoints.

```json
{
  "type": "email.received",
  "data": {
    "id": "3f1c9a2d-4b5e-6a7f-8c9d-0e1f2a3b4c5d",
    "message_id": "<CAF9x8@mail.example.com>",
    "from": "customer@example.com",
    "to": "support@acme.com",
    "subject": "Where is my order?",
    "text": "Hi, checking on order 1043.",
    "html": "<p>Hi, checking on order 1043.</p>",
    "auth_results": null,
    "size_bytes": 4821,
    "received_at": "2026-08-13T09:41:02.118431+00:00"
  }
}
```

The `data` object is the message as we stored it. Raw MIME is not included; it is not retained after parsing. `auth_results` carries the sender-authentication verdicts computed when the message was received, and is `null` when none were recorded.

The forward payload uses shorter key names than the messages API, so a handler written against one does not read the other unchanged:

| Forward payload | `GET /inbound/messages/{id}` |
|-----------------|------------------------------|
| `data.id` | `id` |
| `data.from` | `from_address` |
| `data.to` | `to_address` |
| `data.text` | `text_body` |
| `data.html` | `html_body` |
| `data.auth_results` | not returned |

`message_id`, `subject`, `size_bytes` and `received_at` are spelled the same on both.

How it behaves:

- **The message is stored first, then forwarded.** It is always readable through the API even if your endpoint is down, and the `id` in the payload is the one you can fetch.
- **The message's `status` records the outcome:** `forwarded` when your endpoint answered 2xx, `failed` when it did not. Poll `GET /inbound/messages` filtered by nothing and check `status` to find what your endpoint missed.
- **A failed forward never loses the message.** Delivery is best-effort on top of a message we have already persisted.
- **Redirects are not followed**, and the URL is re-validated at delivery time: one that resolves to a private or internal address is refused even though it passed validation when you claimed the address.
- **Forwards are not signed.** Unlike [email webhooks](/docs/email-webhooks), a `forward_url` carries no HMAC signature, because there is no per-subscription secret behind it. Treat the payload as unauthenticated: use an unguessable URL, and confirm anything you act on by re-reading the message with `GET /inbound/messages/{id}` using your API key.

If you want signed, retried, per-event delivery instead, subscribe to `email.received` on [Email Webhooks](/docs/email-webhooks) — note that event is accepted but not emitted yet, so `forward_url` is the live path for inbound mail today.

### Read what arrived

```bash
# The addresses you currently receive at
curl https://api.callmissed.com/api/v1/email/inbound/addresses \
  -H "Authorization: Bearer cm_your_key"

# One message, parsed to text/HTML
curl https://api.callmissed.com/api/v1/email/inbound/messages/3f1c9a2d-4b5e-6a7f-8c9d-0e1f2a3b4c5d \
  -H "Authorization: Bearer cm_your_key"

# Stop receiving at an address
curl -X DELETE https://api.callmissed.com/api/v1/email/inbound/addresses/7a2b1c0d-9e8f-4a5b-8c7d-6e5f4a3b2c1d \
  -H "Authorization: Bearer cm_your_key"
```

`GET /inbound/messages` takes `limit` (1–200, default 50) and `offset` (≥0, default 0) and returns the newest first. Each `InboundMessageOut` carries `id`, `address_id`, `message_id`, `from_address`, `to_address`, `subject`, `text_body`, `html_body`, `size_bytes`, `status` (`received`, `forwarded` or `failed`) and `received_at`.

## Common inbound failures

| Status | Body | Meaning |
|--------|------|---------|
| 400 | string `detail` | The domain is not verified yet, or the `forward_url` is not a permitted public URL |
| 401 | string `detail` | Missing, malformed or unrecognised `Authorization` header |
| 403 | string `detail` | The API key is read-only and this route writes |
| 404 | string `detail` | The inbound message, or the receiving address, is not yours |
| 409 | string `detail` | The address is already claimed |
| 422 | string `detail` | `local_part` is not a bare mailbox name |

These use the plain-string `detail` shape with no `reason` key. See [Limits, Quotas & Errors](/docs/email-limits#response-shapes).

## Related

- [Domains & Senders](/docs/email-domains) for the `MX` record and verification.
- [Sender Addresses](/docs/email-domains#sender-addresses): claim the same address you put in `reply_to` if you want replies to come back through the API.
