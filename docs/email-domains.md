---
title: "Domains & Senders"
description: "Register a sending domain, publish its DNS records, verify ownership, SPF and both DKIM keys, and understand the donotreply sender rule."
slug: "email-domains"
breadcrumb: "Email"
---

# Domains & Senders

Register a sending domain, publish its DNS records, verify ownership, SPF and both DKIM keys, and understand the donotreply sender rule.

## Overview

Before you can send anything you register a domain you own, publish the DNS records we generate for it, and verify. Verification also registers the one sender username you may send from. Everything on this page uses the base path `https://api.callmissed.com/api/v1/email` and a `cm_` key with the **email** permission.

## Add & Verify a Domain

Register a domain, publish the returned DNS records at your DNS provider, then verify.

**`POST /api/v1/email/domains`** registers the domain and returns its records.
**`POST /api/v1/email/domains/{domain_id}/verify`** runs the checks.

| Create field | Type | Required | Notes |
|--------------|------|----------|-------|
| `domain` | string | Yes | The domain you own, 3–255 chars, for example `acme.com` |

:::tabs
```bash [cURL]
# 1. Add the domain - returns the DNS records to publish
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

for (const rec of created.dns_records) {
  console.log(rec.type, rec.host, rec.value); // publish these at your DNS host
}

// once published:
await fetch(`${BASE}/domains/${created.domain.id}/verify`, {
  method: "POST",
  headers: { Authorization: "Bearer cm_your_key" },
});
```
:::

The add response returns `domain` (a `DomainOut`), `dns_records` (the exact set to publish) and a `message` reading `"Publish these DNS records, then call verify."`. Use each record's `host` (already formatted the way DNS panels want it: `@` for the apex, a bare label for a subdomain) and its `value` verbatim.

| Record | Required | Purpose |
|--------|----------|---------|
| `TXT` (ownership) | Yes | A one-off token proving you control the domain |
| `TXT` (SPF) | Yes | Authorises our sending infrastructure to send for the domain |
| `CNAME` (DKIM) | Yes | Delegates the DKIM signing key for the domain to us |
| `CNAME` (DKIM2) | Yes | A second delegated DKIM key, used for key rotation |
| `MX` | No | Routes inbound mail to us (only needed to **receive**) |

DKIM is **delegated by `CNAME`**, not published as a `TXT` key. The exact host and value of every record are generated per domain, so read them from `dns_records` rather than hard-coding them. If your DNS panel refuses the `CNAME`, you almost certainly have a conflicting record at that name already.

**No DMARC record is generated.** DMARC is worth publishing and we recommend it, but you author `_dmarc.your-domain` yourself. It is not in the returned set and not part of verification.

Verification covers **four** checks (domain ownership, SPF, DKIM and DKIM2) and a domain may send only once **all four** report verified. `POST /domains/{id}/verify` returns the refreshed `domain`, `verified` (a single boolean over all four) and a `checks` array with one entry per check, so a partial pass tells you exactly which record has not propagated yet. Each check is a `CheckOut` carrying `name`, `status`, `detail`, and `found` (the values actually seen in DNS, empty when the check is served by the mail service rather than a DNS scan). Propagation is not instant; call verify again until `verified` is `true`.

A newly verified domain starts on a warm-up quota that rises automatically as it sends clean volume. See [Limits & Quotas](/docs/email-limits).

### Managing domains

| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/email/domains` | Register a domain (`201`) and get its DNS records. A domain already on your account → `400` |
| `GET /api/v1/email/domains` | List your domains, newest first, as `DomainOut` |
| `GET /api/v1/email/domains/{id}/records` | Re-fetch a domain's DNS records at any time, the same set the create call returned |
| `GET /api/v1/email/domains/{id}/provider` | Detect the domain's DNS host from its nameservers and return a deep link to the right DNS-management page |
| `POST /api/v1/email/domains/{id}/verify` | Run verification and return the per-check states |
| `PATCH /api/v1/email/domains/{id}` | Toggle [open/click tracking and sending](#tracking-and-sending-toggles) |
| `DELETE /api/v1/email/domains/{id}` | Remove a domain (`204`). Sending from it stops immediately |

```bash
# List your domains, re-fetch records, look up the DNS host, then remove a domain
curl https://api.callmissed.com/api/v1/email/domains \
  -H "Authorization: Bearer cm_your_key"

curl https://api.callmissed.com/api/v1/email/domains/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e/records \
  -H "Authorization: Bearer cm_your_key"

curl https://api.callmissed.com/api/v1/email/domains/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e/provider \
  -H "Authorization: Bearer cm_your_key"

curl -X DELETE https://api.callmissed.com/api/v1/email/domains/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e \
  -H "Authorization: Bearer cm_your_key"
```

A domain id that isn't yours returns `404` with `{"detail": "Domain not found"}`.

### Tracking and sending toggles

**`PATCH /api/v1/email/domains/{id}`** sets per-domain behaviour. Needs a write key.

| Field | Type | Notes |
|-------|------|-------|
| `open_tracking` | boolean | Track opens on HTML mail from this domain. Also accepted as `track_opens` |
| `click_tracking` | boolean | Track link clicks on HTML mail from this domain. Also accepted as `track_clicks` |
| `sending` | boolean | `false` pauses sending from this domain; `true` resumes it |

Only the fields you send are applied, so an omitted toggle is left untouched. An unsupported field is a `422` rather than a silent no-op, so you always know whether a setting took effect.

```bash
# Turn on open + click tracking
curl -X PATCH https://api.callmissed.com/api/v1/email/domains/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"open_tracking": true, "click_tracking": true}'

# Pause sending from this domain
curl -X PATCH https://api.callmissed.com/api/v1/email/domains/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"sending": false}'
```

The response is `DomainSettingsOut`: every `DomainOut` field plus `track_opens` and `track_clicks`, so you can see what the call just set.

**Tracking.** Both toggles are off by default and independent. Once on, HTML sends from the domain get an open pixel and/or signed click-redirect links. Details of what gets rewritten are on [Send Email](/docs/email-send#open-and-click-tracking); read the numbers back from [engagement metrics](/docs/email-logs#engagement-metrics).

**Sending.** `sending: false` takes effect immediately: sends from the domain stop, and its `status` becomes `paused`. `sending: true` resumes it.

One important limit: **`sending: true` only resumes a domain you paused yourself.** A domain paused automatically for deliverability reasons (too many bounces or complaints for its volume) stays paused and returns `409`. That pause is a circuit breaker, and a breaker a caller can clear is only advice. Fix the underlying list quality, then contact support. `pause_reason` on `DomainOut` tells the two cases apart.

### Response objects

`DomainOut` returns `id`, `domain`, `status` (`pending` / `verified` / `failed` / `paused`), `dkim_selector`, `daily_quota`, `verified_at`, `last_checked_at`, `paused_at`, `pause_reason`, `sent_count`, `bounce_count`, `complaint_count`, `created_at`.

`DomainSettingsOut`, returned by `PATCH /domains/{id}`, is `DomainOut` plus `track_opens` and `track_clicks`.

`DnsRecordOut` returns `type`, `name` (the full FQDN), `host` (the panel-ready name), `value`, `purpose`, `required`, and `priority` (MX only; `null` otherwise).

`DnsProviderOut` returns `detected`, `provider_id`, `provider_name`, `manage_url`, `domain_specific`, and `nameservers`.

### Errors on the domain routes

| Status | Body | Meaning |
|--------|------|---------|
| 400 | string `detail` | The domain is already registered on your account |
| 401 | string `detail` | Missing, malformed or unrecognised `Authorization` header |
| 403 | string `detail` | The API key is read-only and this route writes |
| 404 | `{"detail": "Domain not found"}` | The domain id is not yours |
| 409 | string `detail` | `PATCH sending=true` on a domain paused automatically for deliverability reasons |
| 422 | schema array `detail` | An unsupported field in a `PATCH` body |
| 502 | `reason` nested under `detail` | `acs_unavailable`: domain provisioning or verification is temporarily unavailable, retry the domain call |
| 503 | string `detail` | `"Domain onboarding is not configured"` |

Every shape is spelled out on [Limits, Quotas & Errors](/docs/email-limits#response-shapes).

## Sender Addresses

A message may only be sent from an address whose local part is a **registered sender** on the verified domain. Verification registers exactly one, `donotreply`, so `donotreply@your-domain` works the moment the domain reports verified:

```json
{ "from": "Acme <donotreply@acme.com>" }
```

The local part is matched case-insensitively, so `DoNotReply@acme.com` works too. The **display name is entirely yours**: `"Acme Billing <donotreply@acme.com>"` is what a recipient's mail client shows first, so the mailbox name is rarely the part they read.

Any other local part on the domain has to be registered, and there are two ways to do it.

**1. Register it up front** with the senders endpoints below. This is the explicit route and the one to use when you know your sending addresses.

**2. Let the send path register it.** When a send is refused because the `from` local part is not a registered sender, the service registers that local part and retries the send once. Registration is eventually consistent across the mail service, so if the retries still land before it is live you get `503 sender_propagating`, which means the address is now registered and the send should simply be retried in a minute or two. Nothing else is required.

### Sender endpoints

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/email/domains/{domain_id}/senders` | List the addresses this domain may send from. Read from the mail service, which is the source of truth |
| `POST /api/v1/email/domains/{domain_id}/senders` | Register an address as a permitted sender (`201`) |
| `DELETE /api/v1/email/domains/{domain_id}/senders/{username}` | Remove a sender (`204`) |

The domain must be **verified** before any of the three work; on a `pending`, `failed` or `paused` domain they return `400` with `"Verify the domain before managing its senders"`.

| Create field | Type | Required | Notes |
|--------------|------|----------|-------|
| `username` | string | Yes | A bare local part, 1–64 chars, for example `hello`. It must not contain `@` or `/`, else `422`. Lower-cased before registration |

`SenderOut` returns `username` and `address` (the full `username@domain`).

:::tabs
```bash [cURL]
# What can this domain send from today?
curl https://api.callmissed.com/api/v1/email/domains/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e/senders \
  -H "Authorization: Bearer cm_your_key"

# Register hello@acme.com as a sender
curl -X POST https://api.callmissed.com/api/v1/email/domains/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e/senders \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"username": "hello"}'

# Remove it again
curl -X DELETE https://api.callmissed.com/api/v1/email/domains/9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e/senders/hello \
  -H "Authorization: Bearer cm_your_key"
```
```python [Python]
import httpx

BASE = "https://api.callmissed.com/api/v1/email"
h = {"Authorization": "Bearer cm_your_key"}
domain_id = "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e"

httpx.get(f"{BASE}/domains/{domain_id}/senders", headers=h).json()
httpx.post(f"{BASE}/domains/{domain_id}/senders", headers=h, json={"username": "hello"}).json()
httpx.delete(f"{BASE}/domains/{domain_id}/senders/hello", headers=h)
```
```javascript [JavaScript]
const BASE = "https://api.callmissed.com/api/v1/email";
const domainId = "9d0f8b3a-1c2e-4a5b-8f7d-6e2a1b0c9d4e";
const headers = {
  Authorization: "Bearer cm_your_key",
  "Content-Type": "application/json",
};

await fetch(`${BASE}/domains/${domainId}/senders`, { headers }).then((r) => r.json());

await fetch(`${BASE}/domains/${domainId}/senders`, {
  method: "POST",
  headers,
  body: JSON.stringify({ username: "hello" }),
}).then((r) => r.json());

await fetch(`${BASE}/domains/${domainId}/senders/hello`, {
  method: "DELETE",
  headers: { Authorization: "Bearer cm_your_key" },
});
```
:::

A registered sender responds:

```json
{ "username": "hello", "address": "hello@acme.com" }
```

Registering is **idempotent**: the upstream call is a create-or-update, so re-registering an existing username succeeds rather than returning `409`. It is also eventually consistent, so a send from a just-registered address can still be refused for up to a couple of minutes.

| Status | Body | Meaning |
|--------|------|---------|
| 400 | string `detail` | The domain is not verified yet |
| 422 | string `detail` | `username` is not a bare local part (it contains `@` or `/`, or is empty) |
| 502 | string `detail` | The sender list could not be read, or the sender could not be registered or removed. Note this `502` is a plain string, not the `{error, reason}` shape the domain create/verify routes use |

### Reply-To

**To give people a real address to answer, set `reply_to`.** It is an ordinary header with no sender registration behind it, so it can be any address at all, including a mailbox on another provider:

```json
{
  "from": "Acme Support <donotreply@acme.com>",
  "reply_to": "support@acme.com"
}
```

If you want those replies to come back through the API, claim the same address as a [receiving address](/docs/email-inbound).

The sending **domain** is entirely yours (any verified domain on your account), and `to`, `cc`, `bcc` and `reply_to` are unrestricted.

## Next

- [Send Email](/docs/email-send) once a domain reports verified.
- [Receive Email](/docs/email-inbound) if you also published the `MX` record.
