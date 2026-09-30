---
title: "Campaigns API"
description: "Build an outbound calling campaign over the API: create it, load the list, set the calling hours and dial mode, start it, watch it run, and read the per-contact result. Includes the account-wide do-not-call list."
slug: "voice-campaigns"
breadcrumb: "Numbers (PSTN)"
---

# Campaigns API

Build an outbound calling campaign over the API: create it, load the list, set the calling hours and dial mode, start it, watch it run, and read the per-contact result. Includes the account-wide do-not-call list.

## Overview

A **campaign** is a list of people, a voice agent, a number to call from, and the rules for when and how fast to dial. You build it as a draft, load the list, then start it — and a background dialer works through the list, respecting your calling hours, your retry limit and the account's do-not-call list.

**Base path:** `https://api.callmissed.com/api/v1/voice-campaigns`

:::flow
icon:app | Create | `POST /` makes a draft. Nobody is called yet
icon:gateway | Load the list | `POST /{id}/contacts` or a CSV upload
icon:done | Start | `POST /{id}/status` with `running` — the dialer takes it from here
:::

**Authentication.** Every endpoint accepts a **JWT** (`Authorization: Bearer <jwt>`) or an **API key** (`Authorization: Bearer cm_<key>`). API-key callers need `campaigns:read` for the list, get, contacts, live and do-not-call reads, and `campaigns:write` for everything that changes something — create, update, delete, add contacts, add a suppression, and every status change.

<Callout type="warn">
  Starting a campaign places **real calls to real people** and draws down your credit balance. `campaigns:write` is the scope that allows it; give it only to a key that is meant to run campaigns.
</Callout>

> **Availability.** Campaigns are enabled per account. Where the feature is off, every route below returns `404`.

## Campaign lifecycle

A campaign is always in exactly one status, and only certain moves are allowed. A move that is not allowed returns `409` naming both ends.

| From | May move to |
| --- | --- |
| `draft` | `scheduled`, `cancelled` |
| `scheduled` | `running`, `paused`, `draft`, `cancelled` |
| `running` | `paused`, `completed`, `cancelled` |
| `paused` | `running`, `scheduled`, `cancelled` |
| `insufficient_balance` | `running`, `paused`, `cancelled` |
| `completed` | — terminal |
| `cancelled` | — terminal |

`insufficient_balance` is set for you when the account runs out of credits mid-campaign; you cannot post it. Top up, then move back to `running`.

A brand-new campaign is a `draft`, and `draft` cannot jump straight to `running` — go `draft` → `scheduled` → `running`. Contacts can only be added while the campaign is `draft`, `scheduled` or `paused`.

## Create a campaign

`POST /` · scope `campaigns:write`

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string (1–255) | Yes | Your own label for the campaign |
| `bot_id` | uuid | No | The voice agent that answers for you on each call |
| `from_number_id` | uuid | No | Which of your numbers to call from |
| `scheduled_at` | timestamp | No | When the campaign should begin |
| `timezone` | string | No | IANA zone the calling hours are read in. Default `Asia/Kolkata` |
| `quiet_hours_start` | integer 0–23 | No | Earliest local hour a call may be placed. Default `9` |
| `quiet_hours_end` | integer 1–24 | No | Latest local hour. Must be after the start. Default `21` |
| `max_attempts` | integer 1–5 | No | Retries per contact. Default `3` |
| `dial_mode` | `preview` \| `power` \| `parallel` | No | Default `power` |
| `pace_seconds` | integer 5–3600 | No | Seconds between calls in `power` mode. Default `30` |
| `parallel_lines` | integer 1–5 | No | Calls at once in `parallel` mode. Default `1` |
| `amd_enabled` | boolean | No | Detect an answering machine. Off by default |
| `voicemail_template_id` | uuid | No | Voicemail to leave on a machine. Omit to detect and hang up |
| `dlt_principal_entity_id` | string (≤32, letters/digits) | No | Your India DLT principal entity ID. See [TRAI readiness](#trai-readiness-for-ai-calls-india) |
| `dlt_header` | string (≤32) | No | The DLT header / caller ID these calls go out under |
| `dlt_template_id` | string (≤64) | No | DLT content template ID, if you registered one |
| `dlt_consent_ref` | string (≤128) | No | Your reference to the recorded consent these calls rely on |

A `bot_id`, `from_number_id` or `voicemail_template_id` your account does not own returns `404`. A calling window that is empty or wraps midnight returns `422`, and so does `parallel_lines` above 1 in the same request as a non-parallel `dial_mode`.

```bash
curl -X POST https://api.callmissed.com/api/v1/voice-campaigns \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Q3 renewal outreach",
    "bot_id": "0f2c...",
    "from_number_id": "9a71...",
    "timezone": "Asia/Kolkata",
    "quiet_hours_start": 10,
    "quiet_hours_end": 19,
    "dial_mode": "power",
    "pace_seconds": 45
  }'
```

```json
{
  "id": "6d1e...",
  "tenant_id": "1b0a...",
  "name": "Q3 renewal outreach",
  "status": "draft",
  "total_contacts": 0,
  "completed_contacts": 0,
  "failed_contacts": 0,
  "timezone": "Asia/Kolkata",
  "quiet_hours_start": 10,
  "quiet_hours_end": 19,
  "max_attempts": 3,
  "dial_mode": "power",
  "pace_seconds": 45,
  "parallel_lines": 1,
  "amd_enabled": false,
  "created_at": "2026-09-20T09:00:00Z",
  "updated_at": "2026-09-20T09:00:00Z"
}
```

## Dial modes

| Mode | What it does | Use it when |
| --- | --- | --- |
| `preview` | Nothing is dialled until a person approves each contact, one at a time | A human should see who is next before the phone rings |
| `power` | One call at a time, `pace_seconds` apart | The default. Steady, predictable pacing |
| `parallel` | Up to `parallel_lines` calls at once; the first answer wins | Long lists where most calls go unanswered |

`parallel_lines` above 1 requires `dial_mode: "parallel"`. Switching a parallel campaign to another mode silently resets it to 1 line, so a `power` campaign never carries stale concurrency.

## List and read

`GET /` · scope `campaigns:read` · `limit` 1–100 (default 50), `offset` 0–100000

`GET /{campaign_id}` · scope `campaigns:read`

A campaign that is not yours returns `404`, never another account's row.

## Update

`PATCH /{campaign_id}` · scope `campaigns:write`

Accepts `name`, `scheduled_at`, `from_number_id` and every rail and DLT field from the create table. Anything you leave out is kept; an empty string clears a DLT field. **Status is not settable here** — use the transition endpoint below, so the allowed-move table always applies.

## Change status

`POST /{campaign_id}/status` · scope `campaigns:write`

```bash
curl -X POST https://api.callmissed.com/api/v1/voice-campaigns/6d1e.../status \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"status": "running"}'
```

Returns the updated campaign, or `409` if the move is not allowed. Posting the status it is already in is a no-op that returns `200`.

Moving to `scheduled` or `running` runs the [TRAI readiness check](#trai-readiness-for-ai-calls-india). If your account enforces it and the campaign is not ready, the move returns `409` naming what is missing.

Moving to `running` fires the `campaign.started` webhook; reaching `completed` fires `campaign.completed`. Pausing, cancelling and scheduling fire nothing — they are operator moves, not lifecycle beats a subscriber acts on.

## Add people to the list

`POST /{campaign_id}/contacts` · scope `campaigns:write` · up to **1000** per request

| Field | Type | Required |
|-------|------|----------|
| `contacts[].phone` | string (1–64) | Yes |
| `contacts[].name` | string (0–255) | No |

```bash
curl -X POST https://api.callmissed.com/api/v1/voice-campaigns/6d1e.../contacts \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"contacts": [
        {"phone": "+919000000001", "name": "Ada"},
        {"phone": "+919000000002"}
      ]}'
```

```json
{ "inserted": 2, "skipped_invalid": 0, "total_now": 2 }
```

A row with no usable digits is counted in `skipped_invalid` rather than failing the whole batch. This endpoint does **not** deduplicate — posting the same number twice queues two calls. Adding to a campaign that is `running`, `completed` or `cancelled` returns `409`.

### Upload a CSV

`POST /{campaign_id}/contacts/import` · scope `campaigns:write` · `multipart/form-data`

Each row is upserted into your CRM contacts **and** attached to the campaign, deduplicated against the numbers already on it — so re-uploading the same file never queues a second call to the same person.

| Form field | Type | Notes |
|-------|------|-------|
| `file` | file | UTF-8 CSV, first row is the header |
| `column_map` | string | JSON object mapping your headers to our fields, e.g. `{"Mobile No.":"phone"}` |
| `dry_run` | boolean | **Default `true`** — previews both effects and writes nothing |
| `opted_in` | boolean | Asserts messaging consent for new contacts with no opt-in column of their own |

```bash
curl -X POST https://api.callmissed.com/api/v1/voice-campaigns/6d1e.../contacts/import \
  -H "Authorization: Bearer cm_your_api_key" \
  -F 'file=@leads.csv' \
  -F 'column_map={"Mobile No.":"phone","Full Name":"name"}' \
  -F 'dry_run=false'
```

The response is the CRM import report plus `attached`, `skipped_invalid`, `skipped_duplicate` and `total_now`. Rows the CRM import rejected are never attached, so a row the report told you was skipped is not dialled.

## Read the results

`GET /{campaign_id}/contacts` · scope `campaigns:read` · `limit` 1–500 (default 100), `offset` 0–1000000

Each row carries what happened to that person:

| Field | Notes |
|-------|-------|
| `status` | `pending`, `approved`, `calling`, `done` or `failed` |
| `attempts` | How many times it has been tried |
| `next_attempt_at` | When the next retry is due, if any |
| `last_error` | Why the last attempt did not complete |
| `call_id` | The call this contact produced, for the Telephony API |

The campaign itself carries the totals: `total_contacts`, `completed_contacts`, `failed_contacts`.

## Watch it run

`GET /{campaign_id}/live` · scope `campaigns:read`

Returns the calls this campaign has on the wire right now — each with the contact, the status, and in `parallel` mode which leg won the race — plus, in `preview` mode, the contacts waiting for a go-ahead and how many are queued behind them.

```json
{
  "campaign_id": "6d1e...",
  "status": "running",
  "dial_mode": "parallel",
  "pace_seconds": 30,
  "parallel_lines": 2,
  "active_calls": [
    {
      "call_id": "b41f...",
      "contact_id": "77aa...",
      "phone": "+919000000001",
      "name": "Ada",
      "status": "in_progress",
      "burst_id": "3f2d...",
      "is_winner": true,
      "started_at": "2026-09-20T10:14:02Z"
    }
  ],
  "awaiting_approval": [],
  "awaiting_approval_total": 0
}
```

### Approve one contact in preview mode

`POST /{campaign_id}/contacts/{contact_id}/approve` · scope `campaigns:write`

A `preview` campaign dials only approved contacts, so this is what makes the phone ring. `409` on a campaign that is not in `preview` mode, or on a contact that is no longer waiting; `404` on a contact that is not on this campaign.

## TRAI readiness for AI calls (India)

TRAI's Third Amendment to the TCCCPR (announced 18 September 2026, [press release 119/2026](https://www.trai.gov.in/sites/default/files/2026-09/PR_No119of2026.pdf)) treats calls placed by an application or automated platform, including AI and artificial-voice calls, as A2P calls. Every business making them must declare that use, and the caller IDs it uses, to its telecom provider beforehand. An undeclared A2P call is treated as unsolicited commercial communication. Every campaign call is an A2P call.

You make the declaration with your telecom provider; CallMissed cannot file it for you. What CallMissed does:

- Records, per number, that you have declared it ([`PUT /numbers/{number_id}/a2p-declaration`](/docs/telephony-api#declare-a-number-for-ai-calls-india)).
- Stores the DLT fields above on each campaign.
- Checks both before a campaign starts, and again before each dial.

A campaign is ready when all of these are true:

- The number it calls from is declared.
- `dlt_principal_entity_id` and `dlt_header` are set.
- Either `dlt_template_id` or `dlt_consent_ref` is set.

Enforcement is **off by default**. With it off, a campaign that is not ready still starts and the warning is written to your audit log. With it on, a campaign that is not ready cannot be scheduled or started. A running campaign that stops being ready, for example because its number's declaration is withdrawn, is paused before its next call. An account owner or admin switches enforcement on or off in the dashboard, under **Campaigns**. API keys cannot change it.

`GET /{campaign_id}/trai-readiness` · scope `campaigns:read`

```bash
curl https://api.callmissed.com/api/v1/voice-campaigns/6d1e.../trai-readiness \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "enforced": true,
  "ready": false,
  "issues": [
    { "code": "number_not_declared", "message": "The calling number is not declared for automated/AI calls. ..." }
  ]
}
```

| `code` | Meaning |
|--------|---------|
| `no_from_number` | No active number to call from |
| `number_not_declared` | The calling number has no A2P declaration recorded |
| `dlt_principal_entity_missing` | `dlt_principal_entity_id` is empty |
| `dlt_header_missing` | `dlt_header` is empty |
| `dlt_consent_or_template_missing` | Neither `dlt_template_id` nor `dlt_consent_ref` is set |

`GET /trai-settings` · scope `campaigns:read` returns `{"enforce": false, "updated_at": null}`.

## Delete a campaign

`DELETE /{campaign_id}` · scope `campaigns:write` · `204`

Deletes the campaign and its contact list. Calls it already placed are unaffected and stay in the Telephony API.

## Do-not-call list

The do-not-call list is **account-wide**: no call is placed to a suppressed number from anywhere — a campaign, the Telephony API, an agent tool. Numbers are matched on their trailing digits, so the same person is suppressed whichever way the number was written.

`GET /dnc` · scope `campaigns:read` · `limit` 1–500 (default 100), `offset` 0–1000000

`POST /dnc` · scope `campaigns:write` · up to **1000** entries per request

```bash
curl -X POST https://api.callmissed.com/api/v1/voice-campaigns/dnc \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"entries": [
        {"phone": "+919000000001", "reason": "asked not to be called"}
      ]}'
```

```json
{ "added": 1, "skipped": 0 }
```

Idempotent: a number already on the list is counted in `skipped`, not rejected.

`DELETE /dnc/{entry_id}` · scope `campaigns:write` · `204`

Takes a number back off the list. Do this only when the person has asked to hear from you again.

## Errors

| Status | When |
|--------|------|
| `403` | The API key is missing `campaigns:read` or `campaigns:write` |
| `404` | The campaign, contact, agent, number or voicemail template is not yours |
| `409` | The status move is not allowed, the campaign fails an enforced TRAI readiness check, or you are adding contacts to a running campaign |
| `422` | A DLT field with characters an identifier cannot have, a calling window that wraps midnight, a batch over the cap, a malformed `column_map`, or a CSV that is empty or too large |

## Related

:::cards
/docs/telephony-api | Telephony API | phone | Numbers, KYC and single outbound calls
/docs/agent-tools-mcp | Account MCP Server | wrench | Run campaigns from a connected assistant
/docs/webhooks | Webhooks | webhook | Subscribe to `campaign.started` and `campaign.completed`
:::
