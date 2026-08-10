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
| `forward_url` | string | No | Up to 2048 chars, and it must be a public `http`/`https` URL: an internal or private target is refused at write time with a `400`. Stored against the address for forwarding; **messages are not POSTed to it yet**, so read them from the messages endpoints below |

An inbound address is **globally unique**: one mailbox per address across the whole platform, so a `local_part` already claimed on that domain returns `409`.

`InboundAddressOut` returns `id`, `address` (the full `local_part@domain`), `domain_id`, `forward_url`, `is_active` and `created_at`.

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
