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
| **Exotel** | SIP trunk (vSIP) | Yes | Set up by our team |
| **Smartflo** (Tata) | Media streaming (WebSocket) | Yes | Set up by our team |
| **Pulse** | Contact us | n/a | Contact us |
| **InTalk** | Contact us | n/a | Contact us |
| **Vobiz** | Contact us | n/a | Contact us |

What the **Status** column means:

- **Self-serve**: you can connect it yourself from the dashboard, start to finish. The three self-serve providers are documented below.
- **Set up by our team**: the integration exists, but part of the trunk mapping has to be arranged with the provider on your behalf. Write to `support@callmissed.com` with your account details and we complete the connection with you.
- **Contact us**: not wired yet. Tell us which provider you are on at `support@callmissed.com` and we will scope it.

> **Do not follow the self-serve steps for a "Set up by our team" provider.** Their trunk mapping is done by the provider's own support team, not from your console, and a half-configured trunk silently drops inbound calls.

## Twilio

Connects as an **Elastic SIP Trunk** in your own Twilio account.

:::steps
## Get your Twilio credentials

From the [Twilio Console](https://console.twilio.com/) home page, under **Account Info**, copy:

- **Account SID**, starts with `AC…`.
- **Auth Token**, click to reveal.

Prefer a scoped credential? Instead of the Auth Token you can supply a Twilio **API Key SID** (starts with `SK…`) and its **API Key Secret**. Give both or neither: an API Key SID without its secret is rejected.

The credentials must belong to an account (or subaccount) allowed to manage **Elastic SIP Trunking** and to list incoming phone numbers.

## Enter the details in CallMissed

Open **Phone numbers → Bring your own telephony → Connect**, choose **Twilio**, and paste the Account SID and Auth Token. CallMissed makes a live call to Twilio to verify the pair before anything is stored. Bad credentials fail here, not later on a live call.

## We provision the trunk

CallMissed creates the Elastic SIP Trunk in your Twilio account and wires both directions: an **origination URI** pointing at our SIP endpoint for inbound calls, and a **termination URI** for outbound.

The termination domain Twilio issues always ends in `pstn.twilio.com`. If you are supplying an existing trunk instead of letting us create one, its termination domain must end in `pstn.twilio.com` or the connection is rejected.

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

> **Outbound calls authenticate with the SIP username and password, not with an IP allowlist.** We cannot guarantee a static egress IP for allowlisting, so an IP-only trunk cannot be authorised. Ask your provider to enable digest (username and password) authentication on the trunk. If they cannot, write to `support@callmissed.com` before you start.

**Inbound.** After the connection is created, CallMissed shows you the SIP endpoint to route to. Set that as the inbound destination on your provider's trunk, then confirm each number is attached to that trunk on their side. Only the numbers you entered are accepted.

## Connecting in the dashboard

The wizard is the same for every self-serve provider.

:::steps
## Open the Phone numbers page

In the [Dashboard](https://app.callmissed.com), go to **Phone numbers**. The **Bring your own telephony** card sits next to the rent-a-number card. Choose **Connect**.

## Pick a provider

Select your provider from the list. Providers marked *Set up by our team* show a contact panel instead of a credential form.

## Step 1. Get credentials

The panel tells you exactly where the credentials live in that provider's console, with the field names they use. Fetch them in another tab.

## Step 2. Enter details

Paste the credentials, and for Custom SIP the termination host, transport, and number list. Optionally give the connection a **label** (for example "Twilio, prod account") so two accounts on the same provider are easy to tell apart.

Submitting verifies the credentials with your provider. If verification fails, the connection stays unconnected and shows the reason. Nothing partial is left behind.

## Step 3. Configure and import numbers

CallMissed provisions the trunk, then lists the numbers it found on your account. Select the ones to import. For Custom SIP, this step confirms the numbers you typed in.
:::

A connection moves through **pending → provisioning → active**. If provisioning fails it lands in **error** with a short, readable reason on the connection card. Fix the cause at your provider and reconnect. A connection you no longer want can be **disabled**.

## After connecting

- **Imported numbers appear alongside rented ones.** They show up on the Phone numbers page and in the number list of the [Telephony API](/docs/telephony-api), tagged with the provider they came from.
- **Assign an agent the same way.** Link a voice-agent bot to the number exactly as you would for a rented number. See [Voice Calling](/docs/voice) for building the bot.
- **Per-number call settings still apply.** Greeting, language, voice, STT and TTS models, system prompt, tools, and maximum call duration are all set per number and override the linked bot on that number's calls. The full list is in [Per-number call overrides](/docs/telephony-api).
- **Disconnecting only affects CallMissed.** Removing a provider connection removes its numbers from CallMissed. It does **not** release or cancel anything at your provider, and it does not change your contract with them. Point the number's routing back at your own application before you disconnect, or inbound calls will go nowhere.

## Security

- **Credentials are encrypted at rest** before they reach the database. They are decrypted only in memory, only when a call to your provider needs them.
- **They are never returned by the API.** No response body, log line, or webhook payload contains a provider secret. A connection exposes only whether credentials are present, a masked account identifier, and the non-secret connection facts (trunk ids, SIP address, transport).
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
| The connection sits in **error**. | Credential verification or trunk provisioning failed upstream. | Read the reason on the connection card, fix it at the provider, and reconnect. Credentials are re-verified on every connect. |

Still stuck, or on a provider marked *Set up by our team*? Write to `support@callmissed.com` with your provider, the connection label, and the number you are testing.
