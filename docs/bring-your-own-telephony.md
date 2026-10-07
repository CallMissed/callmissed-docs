---
title: "Bring Your Own Telephony"
description: "Connect a telephony account you already own, import your existing numbers, and let a CallMissed AI voice agent answer the calls."
slug: "bring-your-own-telephony"
breadcrumb: "Numbers (PSTN)"
---

# Bring Your Own Telephony

Connect a telephony account you already own, import your existing numbers, and let a CallMissed AI voice agent answer the calls.

## Overview

**Bring your own telephony (BYO)** lets you keep the phone numbers and the carrier contract you already have, and use them with a CallMissed AI voice agent. Nothing is ported, and you do not rent a number from us.

A connection has two halves, and calls only work once **both** are done:

1. **You give CallMissed your provider credentials.** We verify them against your provider, store them encrypted, and provision the trunk that carries audio between your provider and our voice agents.
2. **You point the number's call routing at CallMissed.** Inbound calls have to arrive at us, so the routing on the number (or on the trunk it belongs to) is switched over to the SIP endpoint we show you after connecting.

:::flow
icon:app | Connect the provider | Paste your provider credentials into CallMissed; we verify them before storing
icon:gateway | Provision the trunk | CallMissed creates the SIP trunk on both sides and hands you the inbound SIP endpoint
icon:phone | Import numbers | Your voice-capable numbers are read from your account and imported
icon:done | Route and answer | Point the number's call routing at CallMissed and assign a voice-agent bot
:::

**Prerequisites**

- An **active account** with a supported telephony provider, holding at least one **voice-capable** number.
- **Admin access to that provider's dashboard**, enough to create or edit a trunk and change a number's call routing.
- A CallMissed account with the **owner** or **admin** role.

> **Note:** Numbers you bring keep their existing contract and billing with your provider. CallMissed does not charge you a monthly number rental for them, unlike numbers you rent from us through the [Telephony API](/docs/telephony-api).

## Rent from us, or bring your own

| | Rent from CallMissed | Bring your own |
|---|---|---|
| **Time to first call** | Fastest. Search, buy, assign a bot. | Depends on your provider's trunk setup. |
| **KYC and compliance** | We handle it. Submit one [KYC application](/docs/telephony-api) and we do the rest. | You already hold the number, so its compliance stays with your provider. |
| **Numbers** | New numbers, issued by us. | Your existing numbers, unchanged. |
| **Carrier billing** | One bill: rental plus usage, in credits. | Your provider keeps billing carriage; CallMissed bills the AI voice agent. |
| **Best for** | Starting from zero, or adding a line quickly. | Keeping published numbers, existing rates, or an existing carrier relationship. |

Both paths converge: once a number is in CallMissed, whether rented or imported, you assign a bot and configure the call the same way.

## Supported providers

| Provider | How it connects | Numbers auto-imported | Status |
|---|---|---|---|
| **Twilio** | SIP trunk (Elastic SIP Trunking) | Yes | Self-serve |
| **Plivo** (your own account) | SIP trunk (Zentrunk) | Yes | Self-serve |
| **Custom SIP** | SIP trunk (any provider) | Manual entry | Self-serve |
| **Exotel** | SIP trunk (vSIP) | n/a | Set up by our team |
| **Smartflo** (Tata) | Media streaming (WebSocket) | n/a | Set up by our team |
| **Pulse** | Contact us | n/a | Contact us |
| **InTalk** | Contact us | n/a | Contact us |

What the **Status** column means:

- **Self-serve**: you can connect it yourself from the dashboard, start to finish. The three self-serve providers are documented below.
- **Set up by our team**: the integration exists, but part of the trunk mapping has to be arranged with the provider on your behalf. Write to `support@callmissed.com` with your account details and we complete the connection with you.
- **Contact us**: not wired yet. Tell us which provider you are on at `support@callmissed.com` and we will scope it.

Over the API, a provider that is not self-serve returns `501` when you try to connect it.

> **Do not follow the self-serve steps for a "Set up by our team" provider.** Their trunk mapping is done by the provider's own support team, not from your console, and a half-configured trunk silently drops inbound calls.

## Twilio

Connects as an **Elastic SIP Trunk** in your own Twilio account.

:::steps
## Get your Twilio credentials

From the [Twilio Console](https://console.twilio.com/) home page, under **Account Info**, copy:

- **Account SID**, starts with `AC…`.
- **Auth Token**, click to reveal.

Prefer a scoped credential? Alongside the Account SID and Auth Token (both still required), you can supply a Twilio **API Key SID** (starts with `SK…`) and its **API Key Secret**; when present, the API key is what CallMissed uses to call Twilio. Give both or neither: an API Key SID without its secret is rejected.

The credentials must belong to an account (or subaccount) allowed to manage **Elastic SIP Trunking** and to list incoming phone numbers.

## Enter the details in CallMissed

Open **Phone numbers → Bring your own telephony → Connect**, choose **Twilio**, and paste the Account SID and Auth Token. CallMissed makes a live call to Twilio to verify the pair before anything is stored. Bad credentials fail here, not later on a live call.

## We provision the trunk

CallMissed creates the Elastic SIP Trunk in your Twilio account and wires both directions: an **origination URI** pointing at our SIP endpoint for inbound calls, and a **termination URI** for outbound.

The termination domain Twilio issues always ends in `pstn.twilio.com`.

## Import your numbers

Your voice-capable Twilio numbers are read from the account and listed for import. Numbers must be in **`+E.164`** form, with the leading `+` and the country code, for example `+14155550123`. A number that Twilio reports in any other format is skipped.

Pick the numbers you want CallMissed to answer. Each imported number is pointed at the trunk we created, which is what takes it off its old voice webhook and routes it to your agent.
:::

> **Inbound calls on a Twilio trunk are matched on the called number**, not on a SIP password. A number that is not imported, or that is still routed by its own voice webhook in the Twilio Console, will not reach your agent even though the credentials are valid.

## Plivo (your own account)

Connects as a **Zentrunk** SIP trunk in your own Plivo account. This is the BYO path. It is separate from numbers you rent from CallMissed, which are billed as rentals.

:::steps
## Get your Plivo credentials

In the [Plivo Console](https://console.plivo.com/), open **Account → Keys & Credentials** and copy your **Auth ID** and **Auth Token**. The account must be allowed to manage Zentrunk trunks and to list phone numbers.

## Enter the details in CallMissed

Open **Phone numbers → Bring your own telephony → Connect**, choose **Plivo**, and paste the Auth ID and Auth Token. They are verified against Plivo before they are stored.

## We provision the trunks

Zentrunk splits the two directions, so CallMissed creates **both** an inbound trunk and an outbound trunk on your Plivo account, and points the inbound trunk's destination at our SIP endpoint.

## Import your numbers

Your Plivo voice numbers are listed for import. Note the format difference: **Plivo returns numbers in E.164 without a leading `+`** (for example `918080247309`, not `+918080247309`). CallMissed normalises them to `+E.164` on import, so they appear the same way as every other number in the dashboard.

Finally, attach each imported number to the inbound trunk in the Plivo Console. Plivo binds a number to a trunk on their side, so this last step is done in their console, not ours.
:::

## Custom SIP

Use this for any provider not listed above that can terminate a SIP trunk. You supply the trunk details yourself, and you enter the numbers manually because there is no account API for us to read them from.

**Fields you supply**

| Field | Example | Notes |
|---|---|---|
| **Termination host** | `sip.example.com` | The outbound SIP host, as a bare hostname. No `sip:` or `sips:` prefix, no path, no `;transport=` parameter. An explicit `:port` is accepted if your provider needs one. |
| **Transport** | `tcp` | One of `auto`, `udp`, `tcp`, `tls`. Defaults to `tcp`. Match what the provider's trunk actually accepts. |
| **SIP username** | `acme-outbound` | The digest username we authenticate outbound calls with. |
| **SIP password** | your trunk password | Stored encrypted, never shown again. |
| **Phone numbers** | `+911140848000` | The `+E.164` numbers to accept inbound calls on. Every entry must carry the leading `+`. |
| **Inbound source IPs** | `1.2.3.4`, `1.2.3.0/24` | The public IPs or CIDR ranges your provider sends inbound calls from. A bare IP is stored as a single-address range. Required unless you set inbound SIP credentials. |
| **Inbound SIP username / password** | `carrier-in` | Optional. Digest credentials your provider sends with inbound calls. Give both or neither; the password must be at least 12 characters. Stored encrypted, never shown again. |

> **Outbound calls authenticate with the SIP username and password, not with an IP allowlist.** We cannot guarantee a static egress IP for allowlisting, so an IP-only trunk cannot be authorised. Ask your provider to enable digest (username and password) authentication on the trunk. If they cannot, write to `support@callmissed.com` before you start.

**Inbound.** Set CallMissed's SIP host as the inbound destination on your provider's trunk, then confirm each number is attached to that trunk on their side. The connection response does not include that host for Custom SIP, so ask `support@callmissed.com` for it. Only the numbers you entered are accepted.

> **Inbound calls must be restricted to your provider.** Give the source IPs your provider sends calls from, inbound SIP credentials, or both. A Custom SIP connection with neither is refused, because anyone who knows one of your numbers could otherwise send a call straight to our SIP host and start an AI call billed to your account. Source ranges must be public and no wider than `/16` (IPv4) or `/48` (IPv6); `0.0.0.0/0`, private, loopback and link-local ranges are rejected.

**Connected before inbound protection was required?** Lock the connection to your provider's IPs with [`PUT /provider-connections/{connection_id}/inbound-allowed-addresses`](#restrict-inbound-calls-on-a-custom-sip-connection). Until you do, the connection accepts inbound calls for its numbers from any source.

## Connecting in the dashboard

The wizard is the same for every self-serve provider.

:::steps
## Open the Phone numbers page

In the [Dashboard](https://console.callmissed.com), go to **Phone numbers**. The **Bring your own telephony** card sits next to the rent-a-number card. Choose **Connect**.

## Pick a provider

Select your provider from the list. Providers marked *Set up by our team* show a contact panel instead of a credential form.

## Step 1. Get credentials

The panel tells you exactly where the credentials live in that provider's console, with the field names they use. Fetch them in another tab.

## Step 2. Enter details

Paste the credentials, and for Custom SIP the termination host, transport, and number list. Optionally give the connection a **label** (for example "Twilio, prod account") so two accounts on the same provider are easy to tell apart.

Submitting verifies the credentials with your provider. If verification fails, you see the reason and no connection is saved. Nothing partial is left behind.

## Step 3. Configure and import numbers

CallMissed provisions the trunk, then lists the numbers it found on your account. Select the ones to import. For Custom SIP, this step confirms the numbers you typed in.
:::

A connection is saved only once verification and provisioning have both succeeded, so a new connection is **active** straight away. If either step fails, the request returns the reason and nothing is saved; fix the cause at your provider and connect again. A connection you no longer want is disconnected (deleted).

## After connecting

- **Imported numbers appear alongside rented ones.** They show up on the Phone numbers page and in the number list of the [Telephony API](/docs/telephony-api), tagged with the provider they came from.
- **Assign an agent the same way.** Link a voice-agent bot to the number exactly as you would for a rented number. See [Voice Calling](/docs/voice) for building the bot.
- **Per-number call settings still apply.** Greeting, language, voice, STT and TTS models, system prompt, tools, and maximum call duration are all set per number and override the linked bot on that number's calls. The full list is in [Per-number call overrides](/docs/telephony-api).
- **Disconnecting only affects CallMissed.** Removing a provider connection removes the trunk CallMissed set up for it, so calls on its numbers stop reaching your agent. The imported numbers stay in your number list, detached from any connection. Nothing is released or cancelled at your provider, and your contract with them is unchanged. Point the number's routing back at your own application before you disconnect, or inbound calls will go nowhere.

## API reference

Everything the dashboard wizard does is also available with your `cm_` key. Base path: `https://api.callmissed.com/api/v1/telephony`. Reads need `telephony:read`; connecting, disconnecting and importing need `telephony:write` (and an owner/admin role when called with a JWT).

| Endpoint | Scope | Purpose |
|---|---|---|
| `GET /provider-catalog` | `telephony:read` | The providers and the credential fields each one needs |
| `GET /provider-connections` | `telephony:read` | Your connections, newest first. `limit` 1–200 (default 50), `offset` 0–100000 |
| `POST /provider-connections` | `telephony:write` | Verify credentials, provision the trunk, and save the connection. `201` |
| `GET /provider-connections/{connection_id}/numbers` | `telephony:read` | Numbers on your provider account that can be imported. `limit` 1–200 (default 100) |
| `POST /provider-connections/{connection_id}/import` | `telephony:write` | Import numbers. `201` |
| `PUT /provider-connections/{connection_id}/inbound-allowed-addresses` | `telephony:write` | Custom SIP only: replace the inbound source IPs. `200` |
| `DELETE /provider-connections/{connection_id}` | `telephony:write` | Disconnect. `204` |

### The provider catalogue

`GET /provider-catalog` returns one entry per provider: `id`, `label`, `description`, `integration` (`sip`, `wss` or `manual`), `can_auto_provision`, `can_list_numbers`, `contact_us` (`true` means it cannot be connected over the API), `credentials` (the fields to send — each with `name`, `label`, `type`, `required`, `help`) and `docs_url`.

| `provider` id | `credentials` fields |
|---|---|
| `twilio` | `account_sid` (required, `AC` + 32 hex), `auth_token` (required), `api_key_sid` and `api_key_secret` (optional, together) |
| `plivo_byo` | `auth_id` (required), `auth_token` (required) |
| `custom_sip` | `address` (required, bare host, optional `:port`), `transport` (`auto`/`udp`/`tcp`/`tls`, default `tcp`), `auth_username` (required), `auth_password` (required), `numbers` (list of `+E.164`, up to 200), `inbound_allowed_addresses` (list of public IPs/CIDRs, up to 200), `inbound_auth_username` and `inbound_auth_password` (optional, together, password at least 12 characters). At least one of `inbound_allowed_addresses` or the inbound credential pair is required |

### Connect a provider

```bash
curl -X POST https://api.callmissed.com/api/v1/telephony/provider-connections \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "twilio",
    "label": "Twilio, prod account",
    "credentials": {"account_sid": "ACxxxxxxxx...", "auth_token": "your_twilio_auth_token"}
  }'
```

| Field | Type | Notes |
|---|---|---|
| `provider` | string (1–64) | **Required.** An `id` from the catalogue |
| `label` | string (≤ 128) | Your own name for the connection |
| `credentials` | object | The fields listed for that provider. Unknown keys are dropped |

**Response (201 Created)** — the connection:

```json
{
  "id": "4b8d2f10-6c3e-4a1b-9f2d-7e8a9b0c1d2e",
  "tenant_id": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
  "provider": "twilio",
  "label": "Twilio, prod account",
  "status": "active",
  "external_account_id": "ACxxxxxxxx...",
  "provider_trunk_id": "TKxxxxxxxx...",
  "config": {"outbound_address": "example.pstn.twilio.com", "transport": "tcp"},
  "last_error": null,
  "has_credentials": true,
  "created_at": "2026-09-20T09:00:00Z",
  "updated_at": "2026-09-20T09:00:00Z"
}
```

The response also carries internal routing identifiers for the trunk; treat any field not shown here as opaque. Credentials are never returned.

| Code | Meaning |
|---|---|
| `409` | This provider account is already connected |
| `422` | Unknown `provider`, a missing or malformed credential field, or (Custom SIP) an `address` that is not a public, fully-qualified host, or no inbound restriction (neither `inbound_allowed_addresses` nor inbound credentials) |
| `501` | The provider is *Set up by our team* or *Contact us* |
| `4xx` / `5xx` | Your provider rejected the credentials or the trunk setup. The message says which |

### List and import numbers

`GET /provider-connections/{connection_id}/numbers` returns the numbers on your provider account (for Custom SIP, the ones you entered): `e164`, `provider_number_id`, `number_type`, `voice_enabled`, `sms_enabled`, `label`, `already_attached` (the provider already routes it somewhere else — importing takes it over) and `already_imported` (it is already on your CallMissed account).

```bash
curl -X POST https://api.callmissed.com/api/v1/telephony/provider-connections/{connection_id}/import \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"numbers": ["+14155550123", "+14155550124"]}'
```

`numbers` takes 1–50 `+E.164` entries; duplicates are collapsed. The response is the imported numbers, in the same shape as `GET /numbers` on the [Telephony API](/docs/telephony-api), with no rental charge. `409` if the connection is not active or a number is already in use (your own clashes are named; a number held elsewhere is only counted). A number that cannot be attached to the trunk at your provider is still imported and works for outbound calls; attach it on the provider's side for inbound.

### Restrict inbound calls on a Custom SIP connection

Replaces the inbound source IPs of an existing Custom SIP connection. Use it to lock down a connection made before inbound protection was required, or when your provider's IPs change.

```bash
curl -X PUT https://api.callmissed.com/api/v1/telephony/provider-connections/{connection_id}/inbound-allowed-addresses \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"addresses": ["1.2.3.4", "1.2.3.0/24"]}'
```

`addresses` takes 1–200 public IPs or CIDR ranges, validated as above; the list replaces the previous one and cannot be emptied. The response is the connection, with the applied list in `config.inbound_allowed_addresses`. `404` if the connection is not yours, `409` for a Twilio or Plivo connection (their provider's ranges are applied automatically at connect time), `422` for an invalid, private or too-broad entry.

### Disconnect

```bash
curl -X DELETE https://api.callmissed.com/api/v1/telephony/provider-connections/{connection_id} \
  -H "Authorization: Bearer cm_your_api_key"
```

`204 No Content`. See [After connecting](#after-connecting) for what this does and does not change.

## Security

- **Credentials are encrypted at rest** before they reach the database. They are decrypted only in memory, only when a call to your provider needs them.
- **They are never returned by the API.** No response body, log line, or webhook payload contains a provider secret. A connection exposes only whether credentials are present (`has_credentials`), your provider account id (Account SID or Auth ID — an identifier, not a secret), and the non-secret connection facts (trunk ids, SIP address, transport).
- **Scoped per tenant.** A connection belongs to one tenant and is only ever readable inside it.
- **To rotate a credential**, change it at your provider, then connect the provider again in CallMissed with the new pair. Verification runs against the new credential before it replaces the old one.

If you believe a credential has leaked, revoke it at your provider first, then reconnect. Revoking at the provider takes effect immediately, whatever is stored on our side.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Inbound calls ring, then drop or hit voicemail. The agent never picks up. | The number's call routing still points at your provider's old app, IVR, or trunk. | Point the number (or its trunk) at the SIP endpoint shown on the connection, and confirm the number is attached to that trunk on the provider's side. |
| One number fails while others on the same account work. | That number was never imported, or it is attached to a different trunk. | Import it, then attach it to the trunk CallMissed provisioned. |
| Outbound calls fail immediately with an authentication error. | Wrong SIP username or password, or the provider's trunk expects IP-based authentication. | Re-enter the SIP credentials. If the trunk is IP-authenticated, ask your provider to enable digest authentication, since outbound uses username and password. |
| Outbound calls time out instead of failing fast. | Wrong termination host, or the wrong transport (for example `tls` on a trunk that only accepts `udp`). | Check the host is a bare hostname with no `sip:` prefix and no parameters, and set the transport to what your provider documents for the trunk. |
| A number is missing from the import list. | The API credentials cannot list numbers, the number is not voice-capable, or it lives in a subaccount the credentials do not cover. | Use credentials for the account that actually holds the number, and confirm the number has the voice capability. |
| A number imported but shows a different format than expected. | Plivo returns numbers without a leading `+`. | Nothing to do. CallMissed normalises to `+E.164` on import. If you enter numbers manually, always include the `+` and the country code. |
| Connecting fails with an error. | Credential verification or trunk provisioning failed at your provider. | Read the reason returned, fix it at the provider, and connect again. Credentials are re-verified on every connect. |

Still stuck, or on a provider marked *Set up by our team*? Write to `support@callmissed.com` with your provider, the connection label, and the number you are testing.
