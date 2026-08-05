---
title: "Migrate from Twilio"
description: "Point an existing Twilio Programmable Voice integration at CallMissed by changing only the base URL and the credentials. Same path, same HTTP Basic scheme, same form parameters, same Call object and error envelope."
slug: "migrate-from-twilio"
breadcrumb: "Numbers (PSTN)"
---

# Migrate from Twilio

Point an existing Twilio Programmable Voice integration at CallMissed by changing only the base URL and the credentials. Same path, same HTTP Basic scheme, same form parameters, same Call object and error envelope.

## Overview

If you already place calls through Twilio Programmable Voice, you can move to CallMissed by changing **two things**: the **base URL** and the **credentials**. The path shape, the HTTP Basic scheme, the `application/x-www-form-urlencoded` body with TitleCase parameters, the JSON Call object, the status enum, the list envelope and the error envelope are all Twilio's — your existing SDK or HTTP client keeps working.

```diff
- https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Calls.json
+ https://api.callmissed.com/2010-04-01/Accounts/{AccountSid}/Calls.json

- -u "ACxxxxxxxx:your_auth_token"          # Twilio SID : auth token
+ -u "any:cm_your_api_key"                 # CallMissed key as the password
```

> **`{AccountSid}` is accepted, ignored, and never trusted.** Keep your old `AC…` sid in the path — every response echoes your *authenticated* CallMissed account sid, and every query is scoped to your tenant, so an arbitrary path sid can never reach another tenant's data. The only credential is your CallMissed API key.

**Authentication.** Both forms work:

- `Authorization: Basic base64(<anything>:cm_your_api_key)` — what a Twilio client sends (`AccountSid:AuthToken`). The **password** carries the key; a key in the username slot (`-u "cm_...:"`) is accepted too.
- `Authorization: Bearer cm_your_api_key` — the native form.

API-key callers need the `telephony:read` scope for fetch/list and `telephony:write` to place a call; placing a call additionally requires an owner/admin role when called with a JWT.

> **Availability.** Telephony is India-only and enabled per tenant. When it is not enabled, these routes are unmounted and return `404`. See [Telephony API](/docs/telephony-api) for the native lifecycle (KYC, buying a number, linking an agent).

## Place a call

```
POST /2010-04-01/Accounts/{AccountSid}/Calls.json
Content-Type: application/x-www-form-urlencoded
```

`201 Created` on success, matching Twilio.

| Parameter | Maps to | Notes |
|-----------|---------|-------|
| `To` | destination | E.164, e.g. `+919876543210` |
| `From` | one of your active numbers | Must be an **active** number on your account (matched with or without a leading `+`) |
| `ApplicationSid` | the agent that handles the call | A CallMissed **agent** is the analogue of a Twilio Application. Accepts an `AP<hex>` sid or a bare agent UUID. Omitted → the from-number's bound agent |
| `CallMissedReason` | spoken context | *(added — not a Twilio param)* a reason the agent can reference |
| `CallMissedVariables` | template values | *(added)* a JSON object of `{{token}}` values rendered into the agent's greeting/prompt |

:::tabs
```bash [cURL]
curl -X POST \
  https://api.callmissed.com/2010-04-01/Accounts/ACxxxxxxxx/Calls.json \
  -u "any:cm_your_api_key" \
  --data-urlencode "To=+919876543210" \
  --data-urlencode "From=+911140000000" \
  --data-urlencode "ApplicationSid=AP0123456789abcdef0123456789abcdef"
```
```python [Python]
import httpx

httpx.post(
    "https://api.callmissed.com/2010-04-01/Accounts/ACxxxxxxxx/Calls.json",
    auth=("any", "cm_your_api_key"),
    data={
        "To": "+919876543210",
        "From": "+911140000000",
        "ApplicationSid": "AP0123456789abcdef0123456789abcdef",
    },
)
```
```javascript [Node.js]
const body = new URLSearchParams({
  To: "+919876543210",
  From: "+911140000000",
  ApplicationSid: "AP0123456789abcdef0123456789abcdef",
});

await fetch(
  "https://api.callmissed.com/2010-04-01/Accounts/ACxxxxxxxx/Calls.json",
  {
    method: "POST",
    headers: {
      Authorization: "Basic " + btoa("any:cm_your_api_key"),
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body,
  },
);
```
:::

### The Call object

Twilio's snake_case Call object, with a few honest CallMissed additions:

```json
{
  "sid": "CA0f1e2d...",
  "account_sid": "AC9a8b7c...",
  "to": "+919876543210",
  "from": "+911140000000",
  "status": "queued",
  "start_time": "Thu, 24 Aug 2023 05:01:45 +0000",
  "end_time": null,
  "duration": null,
  "price": null,
  "price_unit": "credits",
  "direction": "outbound-api",
  "date_created": "Thu, 24 Aug 2023 05:01:45 +0000",
  "date_updated": "Thu, 24 Aug 2023 05:01:45 +0000",
  "uri": "/2010-04-01/Accounts/AC9a8b7c.../Calls/CA0f1e2d....json",
  "api_version": "2010-04-01",
  "callmissed_call_id": "…",
  "callmissed_agent_sid": "AP…"
}
```

Fidelity details that match Twilio exactly:

- **`duration` and `price` are strings**, not numbers, and stay `null` until the call is billed. `price` is negative (an amount debited).
- **Dates are RFC 2822** (`"Thu, 24 Aug 2023 05:01:45 +0000"`), not ISO 8601.
- **`price_unit` is `"credits"`** — CallMissed bills in credits, not a currency, so the field says so rather than pretending to be `"USD"`. Your upstream carrier cost is never exposed.
- **`status`** is exactly `queued`, `ringing`, `in-progress`, `canceled`, `completed`, `busy`, `failed`, `no-answer`.

## Fetch and list

```bash
# One call
curl https://api.callmissed.com/2010-04-01/Accounts/ACxxxxxxxx/Calls/CA0f1e2d....json \
  -u "any:cm_your_api_key"

# List (filters: To, From, Status; paging: Page, PageSize)
curl "https://api.callmissed.com/2010-04-01/Accounts/ACxxxxxxxx/Calls.json?Status=completed&PageSize=50" \
  -u "any:cm_your_api_key"
```

The list envelope is Twilio's, and the array key is the lower-cased resource name — **`calls`** — with `page`, `page_size`, `uri`, `first_page_uri`, `next_page_uri` and `previous_page_uri`.

## Parameters that are rejected, not ignored

Silently ignoring a parameter would connect the call and then behave differently from what you asked — worse than refusing it. These return `400` with a Twilio-shaped error that names the parameter:

- **`Url` / `Twiml` / `Method` / `Fallback*`** — there is no TwiML interpreter. CallMissed calls are agent-driven; the behaviour comes from `ApplicationSid` (the agent), not a markup document.
- **`Record` / `RecordingStatusCallback*`** — per-call recording is not controllable through this API.
- **`MachineDetection*` / `AsyncAmd*`** — no answering-machine detection.
- **`SendDigits`** — no post-answer DTMF injection.
- **`Timeout` / `TimeLimit`** — no per-call ring/duration override (the agent's configured max duration applies).
- **`StatusCallback` / `StatusCallbackEvent` / `StatusCallbackMethod`** — per-call status callbacks are not delivered. Subscribe instead to the `call.started` / `call.completed` / `call.failed` [webhook events](/docs/webhooks) at `/api/v1/webhooks`.

## Error envelope

Twilio's shape, unchanged:

```json
{
  "status": 400,
  "message": "The 'From' number +911140000000 is not an active phone number on this account.",
  "code": 21210,
  "more_info": "https://www.twilio.com/docs/errors/21210"
}
```

Where a real Twilio error code fits, it is used (`21201`, `21211`, `21212`, `21213`, `21210`, `21217`, `20003`, `20404`, `20429`), and `more_info` points at `twilio.com`. Where no Twilio code fits, a CallMissed code in the `61000–61999` range is used, and its `more_info` points at `docs.callmissed.com` — never at a `twilio.com` page that would describe something unrelated.

## When to use the native API instead

For new builds, the [Telephony API](/docs/telephony-api) exposes CallMissed's full lifecycle — India KYC, buying numbers, linking agents, richer call records — with your `cm_` key directly. This Twilio-compat surface exists to make an *existing* Programmable Voice integration a two-line migration.
