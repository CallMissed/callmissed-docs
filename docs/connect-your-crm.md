---
title: "Connect Your CRM"
description: "Two-way sync between your own CRM and CallMissed: create-or-update records by your own id, receive change webhooks, catch up after downtime with updated_after, attach call transcripts and recordings, and handle 409s and retries."
slug: "connect-your-crm"
breadcrumb: "Getting Started"
---

# Connect Your CRM

Two-way sync between your own CRM and CallMissed: create-or-update records by your own id, receive change webhooks, catch up after downtime with updated_after, attach call transcripts and recordings, and handle 409s and retries.

:::cards
/docs/crm-contacts | Contacts | boxes | The person object and create-or-update
/docs/webhooks | Webhooks | webhook | Event catalogue, signatures, delivery log
/docs/telephony-api | Telephony | phone | Calls, transcripts and recordings
:::

## Overview

This guide connects a CRM you already run (HubSpot, Salesforce, Zoho, or your own database) to CallMissed, in both directions, using only a `cm_` API key. Every step names the exact method, path, scope, request and response, so it can be followed by a person or by an AI coding agent without opening the console.

| Direction | What you use | Section |
| --- | --- | --- |
| Your CRM to CallMissed | `POST …/upsert` with your own record id | [1](#1-send-your-records-to-callmissed) |
| CallMissed to your CRM, live | Webhooks: `contact.*`, `company.*`, `lead.*`, `deal.*`, `task.*`, `note.*` | [2](#2-receive-changes-from-callmissed) |
| CallMissed to your CRM, after a gap | `GET …?updated_after=` | [3](#3-catch-up-after-downtime) |
| Calls onto your records | `call.completed`, then the transcript and recording endpoints | [4](#4-attach-calls-to-your-records) |

Base URL for every request: `https://api.callmissed.com`. Header on every request: `Authorization: Bearer cm_your_api_key`. Errors are `{"detail": …}`. Nothing in this guide consumes credits.

## Before you start

### Key scopes

Create one API key for the integration and give it only these scopes. A scope the key lacks returns `403 API key missing required scope: <scope>`.

| Scope | Needed for |
| --- | --- |
| `contacts:read`, `contacts:write` | Contact upsert (needs both), lookup, changed-since list, subscribing to `contact.*` |
| `companies:read`, `companies:write` | Company upsert (needs both), lookup, changed-since list, subscribing to `company.*` |
| `crm_leads:read`, `crm_leads:write` | Lead upsert (needs both), lookup, changed-since list, subscribing to `lead.*` (skip if you do not sync leads) |
| `crm_deals:read` | Deal changed-since list, subscribing to `deal.*` (add `crm_deals:write` to write deals) |
| `crm_tasks:read` | Task changed-since list, subscribing to `task.*` (add `crm_tasks:write` to write tasks) |
| `crm_notes:read` | Subscribing to `note.*` (add `crm_notes:write` to write a note onto a record, e.g. a call summary) |
| `webhooks:write` | Creating and inspecting the webhook subscription. Not needed at run time if you create the subscription once in the console |
| `telephony:read` | Reading calls, call transcripts and recordings |
| `crm_search:write` | Only if your integration merges duplicates itself after a `409 ambiguous_match` |

Receiving a webhook needs no scope: events are pushed to your URL and authenticated by their signature.

### Decide how records are linked

Each side stores the other side's id:

| Record | Where your id lives in CallMissed | Where to keep the CallMissed id |
| --- | --- | --- |
| Contact | `external_ids`, e.g. `{"hubspot": "123"}` | A field on your contact |
| Company | `external_ids`, e.g. `{"hubspot": "8841"}` | A field on your company |
| Lead | `external_ref`, e.g. `"form-88213"` | A field on your lead |
| Deal, task, note | No external id field | Keep a map of CallMissed `id` to your id on your side |

Pick one system name (for example `hubspot`) and use it everywhere. It is 1 to 64 characters with no colon. Ids are stored as strings.

## 1. Send your records to CallMissed

Use create-or-update for every write. It finds the existing record by your id (or by phone, email, domain or name), updates it, or creates it, so the same call is correct for a first import, a nightly sync and a retry.

| Record | Method and path | Scope | Reference |
| --- | --- | --- | --- |
| Contact | `POST /api/v1/contacts/upsert` | `contacts:write` + `contacts:read` | [Contacts](/docs/crm-contacts#post-apiv1contactsupsert) |
| Company | `POST /api/v1/companies/upsert` | `companies:write` + `companies:read` | [Companies](/docs/crm-companies#post-apiv1companiesupsert) |
| Lead | `POST /api/v1/crm/leads/upsert` | `crm_leads:write` + `crm_leads:read` | [Leads](/docs/crm-leads#post-apiv1crmleadsupsert) |

Send companies first, then contacts with the returned company `id` as `company_id`.

```bash
curl -X POST https://api.callmissed.com/api/v1/companies/upsert \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "external_ids": { "hubspot": "8841" },
    "name": "Acme Retail",
    "domain": "acme.com",
    "match_on": ["external_id", "domain"]
  }'
```

```json
{
  "id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "name": "Acme Retail",
  "domain": "acme.com",
  "phone": null,
  "website": null,
  "industry": null,
  "size": null,
  "notes": null,
  "external_ids": { "hubspot": "8841" },
  "tags": [],
  "metadata": null,
  "created_at": "2026-11-17T06:31:09.870000Z",
  "updated_at": "2026-11-17T06:31:09.870000Z",
  "created": true,
  "matched_on": null
}
```

```bash
curl -X POST https://api.callmissed.com/api/v1/contacts/upsert \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "external_ids": { "hubspot": "123" },
    "phone": "+919876543210",
    "email": "asha@acme.com",
    "name": "Asha Menon",
    "company_id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f"
  }'
```

```json
{
  "id": "44110c7e-2a51-4c0f-9d3b-6f1e2a7b8c90",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "company_id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f",
  "phone": "+919876543210",
  "email": "asha@acme.com",
  "name": "Asha Menon",
  "external_ids": { "hubspot": "123" },
  "whatsapp_opt_in": false,
  "email_opt_in": false,
  "sms_opt_in": false,
  "consent_source": null,
  "consent_updated_at": null,
  "whatsapp_bsuid": null,
  "whatsapp_parent_bsuid": null,
  "whatsapp_username": null,
  "lifecycle_stage": null,
  "lead_source": null,
  "owner_user_id": null,
  "tags": [],
  "metadata": null,
  "created_at": "2026-11-17T06:31:10.412000Z",
  "updated_at": "2026-11-17T06:31:10.412000Z",
  "created": true,
  "matched_on": null
}
```

What to do with the response:

| Response | Meaning | Next step |
| --- | --- | --- |
| `201`, `"created": true` | A new record was created | Store `id` on your record |
| `200`, `"created": false`, `"matched_on": "external_id"` | Your record was already linked and has been updated | Nothing |
| `200`, `"created": false`, `"matched_on": "phone"` (or `email`, `domain`, `name`) | An existing record was found by a natural key and is now linked to your id | Store `id` on your record |
| `409` | See [conflicts](#5-handle-conflicts-and-retries) | |

Rules worth knowing before you send real data:

- **Only the fields you send are written** on an update. Leave out a field you do not own (for example consent flags captured in CallMissed) and it is kept. `tags`, when sent, replaces the whole list.
- **`external_ids` are merged**, so a second system can add its own id without removing yours.
- **Phone matching** compares the last ten digits and, when both numbers carry a country code, the country codes too. Send numbers in E.164 (`+919876543210`) where you can.
- **Company names match by default.** `name` is required on a company, and the default `match_on` includes `name`. If two of your accounts can share a name, send `"match_on": ["external_id", "domain"]` as above.
- **A contact needs a way to be reached.** A new contact requires `phone`, `email` or `whatsapp_bsuid`; an id alone is `422`.

To check a link without writing, look the record up by your id. An empty array means it has not been sent yet:

```bash
curl "https://api.callmissed.com/api/v1/contacts?external_id=hubspot:123" \
  -H "Authorization: Bearer cm_your_api_key"

curl "https://api.callmissed.com/api/v1/companies?external_id=hubspot:8841" \
  -H "Authorization: Bearer cm_your_api_key"

curl "https://api.callmissed.com/api/v1/crm/leads?external_ref=form-88213" \
  -H "Authorization: Bearer cm_your_api_key"
```

Deleting on your side: look the record up as above, then `DELETE /api/v1/contacts/{contact_id}` (or the company or lead equivalent), which returns `204`.

## 2. Receive changes from CallMissed

Records change in CallMissed without you: an agent updates a contact during a call, a teammate edits a deal in the console, an import adds leads. Subscribe one HTTPS endpoint to the CRM events and you are told about each change.

### Create the subscription

`POST /api/v1/webhooks`, scope `webhooks:write` plus the read scope of every record type you name (`contacts:read` for `contact.*`, `companies:read`, `crm_leads:read`, `crm_deals:read`, `crm_tasks:read`, `crm_notes:read`). Name each CRM event you want: they are not included in an "every event" subscription. Leave `bot_id` out: CRM events are sent to workspace-wide subscriptions only. Add the call events here too if you follow [section 4](#4-attach-calls-to-your-records).

```bash
curl -X POST https://api.callmissed.com/api/v1/webhooks \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://crm.example.com/hooks/callmissed",
    "events": [
      "contact.created", "contact.updated", "contact.deleted",
      "company.created", "company.updated", "company.deleted",
      "lead.created", "lead.updated", "lead.deleted",
      "deal.created", "deal.updated", "deal.deleted",
      "task.created", "task.updated", "task.deleted",
      "note.created", "note.updated", "note.deleted",
      "call.completed", "call.failed", "voice_analysis.completed"
    ]
  }'
```

```json
{
  "id": "0f1e2d3c-4b5a-4697-8887-766554433221",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "url": "https://crm.example.com/hooks/callmissed",
  "secret": "Xk4pR7tYb2Lm8sNcVhJ3wQeZ1aFgD6uO9dQz",
  "events": ["contact.created", "contact.updated", "contact.deleted", "company.created", "company.updated", "company.deleted", "lead.created", "lead.updated", "lead.deleted", "deal.created", "deal.updated", "deal.deleted", "task.created", "task.updated", "task.deleted", "note.created", "note.updated", "note.deleted", "call.completed", "call.failed", "voice_analysis.completed"],
  "is_active": true,
  "bot_id": null,
  "created_at": "2026-11-17T06:20:00Z"
}
```

Store `secret` now. It is shown in full only in this response. `400` means the URL is not a public HTTPS address; `403` names the scope the key is missing (drop that record type's events or add the scope); `422` means an event name is not in the [catalogue](/docs/webhooks#event-types).

### What arrives

A `POST` with headers `X-CallMissed-Event`, `X-CallMissed-Delivery` and `X-CallMissed-Signature`, and this body. The record sits under a key named after its type (`contact`, `company`, `lead`, `deal`, `task` or `note`) and is the same JSON its `GET` returns:

```json
{
  "event": "contact.updated",
  "data": {
    "event": "contact.updated",
    "id": "44110c7e-2a51-4c0f-9d3b-6f1e2a7b8c90",
    "occurred_at": "2026-11-17T09:02:44.201330+00:00",
    "source": "agent",
    "contact": {
      "id": "44110c7e-2a51-4c0f-9d3b-6f1e2a7b8c90",
      "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
      "company_id": "5c6d7e8f-9a0b-4c1d-8e2f-3a4b5c6d7e8f",
      "phone": "+919876543210",
      "email": "asha@acme.com",
      "name": "Asha Menon",
      "external_ids": { "hubspot": "123" },
      "whatsapp_opt_in": true,
      "email_opt_in": false,
      "sms_opt_in": false,
      "consent_source": "voice_call",
      "consent_updated_at": "2026-11-17T09:02:44.120000Z",
      "whatsapp_bsuid": null,
      "whatsapp_parent_bsuid": null,
      "whatsapp_username": null,
      "lifecycle_stage": "qualified",
      "lead_source": null,
      "owner_user_id": null,
      "tags": [],
      "metadata": null,
      "created_at": "2026-11-17T06:31:10.412000Z",
      "updated_at": "2026-11-17T09:02:44.120000Z"
    }
  },
  "timestamp": "2026-11-17T09:02:44.301554+00:00"
}
```

A `deleted` event carries only `id` plus your own reference (`external_ids` for a contact or company, `external_ref` for a lead, nothing more for a deal, task or note), and `merged_into` when a contact or company was folded into another by a merge. Full payload reference: [CRM record events](/docs/webhooks#crm-record-events).

### Handle it

1. **Verify the signature** over the raw body, before parsing, exactly as on [Webhooks](/docs/webhooks#signing-and-verification). Reject with `401` when it does not match.
2. **De-duplicate** on the `X-CallMissed-Delivery` header. A retry or a replay delivers the same event again.
3. **Skip your own writes.** `data.source` is `api` for anything written with an API key, which includes your integration. If no other API client writes CRM records, drop every `source: "api"` event; otherwise compare the record with what you hold.
4. **Ignore stale events.** Events are not delivered in a fixed order. Keep the record's `updated_at` and skip an event whose `updated_at` is older than the one you stored.
5. **Apply it.** Find your record by `external_ids` (or `external_ref`); if there is none, by the CallMissed `id` you stored; if still none, create it on your side and send the link back with an upsert.
6. **Answer `2xx` within 10 seconds.** Do slow work after responding. Anything else is retried, up to 5 attempts in total.

```python
import hashlib, hmac, json

RECORD_KEYS = ("contact", "company", "lead", "deal", "task", "note")

def handle(raw_body: bytes, headers: dict, secret: str) -> int:
    """Returns the HTTP status to answer with. `headers` keys as sent; match them case-insensitively in your framework."""
    expected = "sha256=" + hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()
    if not any(
        hmac.compare_digest(expected, headers.get(name, ""))
        for name in ("X-CallMissed-Signature", "X-CallMissed-Signature-Previous")
    ):
        return 401
    if seen_before(headers["X-CallMissed-Delivery"]):       # your store of delivery ids
        return 200

    body = json.loads(raw_body)
    entity, _, action = body["event"].partition(".")
    data = body["data"]
    if entity not in RECORD_KEYS:
        return 200                                          # call.* and others: see section 4
    if data["source"] == "api":
        return 200                                          # our own write coming back
    record = data[entity]

    if action == "deleted":
        if record.get("merged_into"):
            repoint(entity, old_id=record["id"], new_id=record["merged_into"])
        else:
            delete_local(entity, callmissed_id=record["id"], external=record)
    elif not is_stale(entity, record["id"], record["updated_at"]):
        save_local(entity, record)                          # create or update by id
    return 200
```

`seen_before`, `repoint`, `delete_local`, `is_stale` and `save_local` are your own functions.

Check deliveries with `GET /api/v1/webhooks/{webhook_id}/deliveries`, and re-send one with `POST /api/v1/webhooks/{webhook_id}/deliveries/{delivery_id}/replay` ([reference](/docs/webhooks#get-apiv1webhookswebhook_iddeliveries)).

## 3. Catch up after downtime

A webhook delivery is attempted up to 5 times in quick succession and then stops. If your endpoint was down for longer, or you are connecting an account that already has data, read what changed with `updated_after`.

| Record | Method and path | Scope |
| --- | --- | --- |
| Contacts | `GET /api/v1/contacts?updated_after=…` | `contacts:read` |
| Companies | `GET /api/v1/companies?updated_after=…` | `companies:read` |
| Leads | `GET /api/v1/crm/leads?updated_after=…` | `crm_leads:read` |
| Deals | `GET /api/v1/crm/deals?updated_after=…` | `crm_deals:read` |
| Tasks | `GET /api/v1/crm/tasks?updated_after=…` | `crm_tasks:read` |

| Parameter | Type | Default | Meaning |
| --- | --- | --- | --- |
| `updated_after` | ISO 8601 date-time | none | Only records whose `updated_at` is **later** than this. A time with no offset is read as UTC. Not a date-time: `422` |
| `limit` | `integer` | `50` | `1` to `200` |
| `offset` | `integer` | `0` | `0` to `100000` |

With `updated_after` set, the result is ordered oldest change first (`updated_at` ascending, then `id`) and any `sort` parameter is ignored. The other filters of each list still apply. The response is a plain JSON array of the records.

```bash
curl "https://api.callmissed.com/api/v1/contacts?updated_after=2026-11-17T06:30:00Z&limit=200" \
  -H "Authorization: Bearer cm_your_api_key"
```

### Paging without missing a record

Many records can share one `updated_at` (a bulk edit or an import stamps them together), and `updated_after` is strictly "later than". So do not simply move the checkpoint to the last row of each page. Use this loop: it moves the checkpoint forward only to a timestamp it has fully read, and pages with `offset` inside a group of equal timestamps.

```python
import requests
from datetime import datetime, timedelta

API = "https://api.callmissed.com"
HEADERS = {"Authorization": "Bearer cm_your_api_key"}
PAGE = 200

def ts(row):
    return datetime.fromisoformat(row["updated_at"].replace("Z", "+00:00"))

def changed_since(path, checkpoint):
    """Yield every record changed after `checkpoint` (ISO 8601 string), oldest change first.
    A record can be yielded more than once, so write it on your side by id."""
    cursor, offset = checkpoint, 0
    while True:
        r = requests.get(API + path, headers=HEADERS, timeout=30,
                         params={"updated_after": cursor, "limit": PAGE, "offset": offset})
        r.raise_for_status()
        page = r.json()
        yield from page
        if len(page) < PAGE:
            return
        earlier = [row["updated_at"] for row in page if ts(row) < ts(page[-1])]
        if earlier:
            cursor, offset = earlier[-1], 0   # re-read the last timestamp's group from its start
        else:
            offset += PAGE                    # the whole page shares one timestamp

def sync(path, saved_checkpoint):
    # Start a little before the saved checkpoint: a change being saved while the
    # last run was reading can carry a slightly earlier timestamp.
    start = (datetime.fromisoformat(saved_checkpoint.replace("Z", "+00:00"))
             - timedelta(minutes=5)).isoformat()
    newest = saved_checkpoint
    for record in changed_since(path, start):
        save_local(record)                    # create or update by id; skip if not newer
        if ts(record) > datetime.fromisoformat(newest.replace("Z", "+00:00")):
            newest = record["updated_at"]
    return newest                             # store as the next checkpoint

# First run: sync("/api/v1/contacts", "1970-01-01T00:00:00Z")
```

Things this list cannot tell you:

- **Deletions.** A deleted record is simply absent. Deletes reach you only as `*.deleted` webhooks. If you may have missed some, compare ids: page the full list (no `updated_after`) and remove local records whose CallMissed id is no longer there, or check one with `GET /api/v1/contacts/{contact_id}` (`404` = gone).
- **Notes.** Notes are listed per record (`GET /api/v1/crm/notes?entity_type=…&entity_id=…`), so there is no account-wide changed-since list. Follow them with the `note.*` webhooks, and re-read the notes of any record that `updated_after` returned if you need them complete.

A simple schedule that works: handle webhooks as they arrive, and run `sync` for each record type every few minutes or hourly as a safety net. Both paths write by id, so overlap is harmless.

## 4. Attach calls to your records

When a phone call ends, put its outcome on the right record in your CRM.

### 4.1 Hear about the call

Subscribe to `call.completed` and `call.failed` (included in the subscription above). The body:

```json
{
  "event": "call.completed",
  "data": {
    "call_id": "5e6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c",
    "direction": "inbound",
    "status": "completed",
    "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
    "remote_e164": "+919876543210",
    "voice_session_id": "7f8a9b0c-1d2e-3f4a-5b6c-7d8e9f0a1b2c",
    "hangup_cause_code": null
  },
  "timestamp": "2026-11-17T06:45:31.774120+00:00"
}
```

| `data` field | Use |
| --- | --- |
| `call_id` | Key for the transcript, recording and call detail requests below |
| `remote_e164` | The other party's number: find your record by it |
| `voice_session_id` | Matches `session_id` on the later `voice_analysis.completed` event. `null` when no agent took the call |
| `status` | `completed`; on `call.failed` one of `failed`, `busy`, `no_answer` |

### 4.2 Find the contact

Match `remote_e164` against the phone numbers in your own CRM, or ask CallMissed (`q` is a substring match on phone, so the last ten digits avoid formatting differences):

```bash
curl "https://api.callmissed.com/api/v1/contacts?q=9876543210&limit=5" \
  -H "Authorization: Bearer cm_your_api_key"
```

The returned contact's `external_ids` holds your record id.

### 4.3 Fetch the transcript

`GET /api/v1/telephony/calls/{call_id}/transcript`, scope `telephony:read`. `format` is `json` (default), `txt` or `srt`.

```bash
curl "https://api.callmissed.com/api/v1/telephony/calls/5e6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c/transcript?format=txt" \
  -H "Authorization: Bearer cm_your_api_key"
```

```
Session: 7f8a9b0c-1d2e-3f4a-5b6c-7d8e9f0a1b2c
Started: 2026-11-17T06:40:02.031000+00:00
Ended:   2026-11-17T06:45:30.900000+00:00

User:  Hi, I wanted to check on my order.
Agent: Sure, can you tell me the order number?
```

`format=json` returns one object per turn (`turn_index`, `user_transcript`, `agent_response`, timings, `tool_calls`); every field is listed on [Telephony](/docs/telephony-api#transcripts). `404 No transcript available` means no agent answered the call, so there is nothing to attach; `404 Call not found` means the id is wrong.

### 4.4 Fetch the recording

`GET /api/v1/telephony/calls/{call_id}/recording`, scope `telephony:read`. Calls are recorded only when the agent has recording turned on.

```bash
curl https://api.callmissed.com/api/v1/telephony/calls/5e6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c/recording \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "url": "https://media.callmissed.com/recordings/....?token=..." }
```

The URL is time-limited. Download the audio into your CRM, or store the `call_id` and request a fresh URL when someone presses play; do not store the URL. `404` means the call has no recording. The recording is ready a few seconds after the call ends, so on a `404` right after `call.completed` try once more after a short wait. `503` means recording storage is temporarily unavailable: retry later.

### 4.5 Add the summary

`voice_analysis.completed` arrives after the call's post-call analysis is saved. Join it to the call through `session_id` = the call event's `voice_session_id`.

```json
{
  "event": "voice_analysis.completed",
  "data": {
    "analysis_id": "b7c8d9e0-f1a2-4b3c-8d4e-5f6a7b8c9d0e",
    "session_id": "7f8a9b0c-1d2e-3f4a-5b6c-7d8e9f0a1b2c",
    "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
    "summary": "Caller asked about order 4471. Agent confirmed it ships tomorrow.",
    "sentiment": "positive",
    "disposition": "resolved",
    "goal_met": true,
    "extracted": { "order_number": "4471" }
  },
  "timestamp": "2026-11-17T06:46:05.019842+00:00"
}
```

For duration, cost and the answering-machine verdict, read the call itself: `GET /api/v1/telephony/calls/{call_id}` ([reference](/docs/telephony-api#7-list--fetch-calls)).

To also show the call on the CallMissed contact, write a note (scope `crm_notes:write`):

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/notes \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "entity_type": "contact",
    "entity_id": "44110c7e-2a51-4c0f-9d3b-6f1e2a7b8c90",
    "body": "Call 5e6a7b8c: asked about order 4471, ships tomorrow."
  }'
```

## 5. Handle conflicts and retries

### 409 from an upsert

An upsert `409` carries an object. Read `detail.code`; `detail.ids` lists the CallMissed records involved.

```json
{
  "detail": {
    "code": "ambiguous_match",
    "message": "More than one contact matches this record. Merge them, or send match_on with only the key that identifies it.",
    "ids": ["44110c7e-2a51-4c0f-9d3b-6f1e2a7b8c90", "9922aa01-77b4-4e2c-8a10-0c5d6e7f8a9b"]
  }
}
```

| `detail` | Why | What to do |
| --- | --- | --- |
| `code: "ambiguous_match"` | The keys you sent point at more than one record, e.g. your id is on one contact and the phone is on another | If they are one person, merge them (below) and retry. If they are two people, retry with `"match_on": ["external_id"]`. Do not retry unchanged: the answer will be the same |
| `code: "external_id_conflict"` | The record found by phone, email, domain or name already carries a different id for your system, so it is linked to another of your records | Usually a duplicate on your side. Resolve it there, or retry with `"match_on": ["external_id"]` to write by id alone |
| A plain string, e.g. `A contact with this phone, email or WhatsApp BSUID already exists` | A unique field that is not in your `match_on` is already on another record | Put that key back in `match_on`, or leave the field out of the body |

Merging two contacts that are one person (scope `crm_search:write`; this deletes the duplicates and cannot be undone):

```bash
curl -X POST https://api.callmissed.com/api/v1/crm/search/merge \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "entity_type": "contact",
    "primary_id": "44110c7e-2a51-4c0f-9d3b-6f1e2a7b8c90",
    "duplicate_ids": ["9922aa01-77b4-4e2c-8a10-0c5d6e7f8a9b"]
  }'
```

Each duplicate then sends `contact.deleted` with `merged_into` set to the survivor. See [merge](/docs/crm-import-export#post-apiv1crmsearchmerge).

### Other statuses

| Status | Meaning | Retry? |
| --- | --- | --- |
| `401` | Missing, wrong or expired key | No. Fix the key |
| `403` | The key lacks the scope named in `detail` | No. Add the scope |
| `404` | The id is not in your account | No |
| `409` | See above | Only after changing the request or the data |
| `422` | The body or a parameter is invalid; `detail` says which field | No. Fix the request |
| `429` | Rate limited | Yes, after the number of seconds in the `Retry-After` header. See [Rate limits](/docs/rate-limits) |
| `500`, `502`, `503`, `504` or a timeout | Temporary | Yes, with increasing waits (for example 1, 2, 4, 8 seconds) |

### Retrying safely

- **Upserts can be repeated.** Sending the same upsert twice leaves one record; the second call returns `200` with `"created": false`. Two upserts of the same record sent at the same moment are handled one after the other.
- **Plain `POST` creates cannot.** `POST /api/v1/crm/deals`, `/crm/tasks` and `/crm/notes` create a new row each time. After a timeout, list first (deals by `contact_id`, tasks and notes by `entity_type` and `entity_id`) before sending again.
- **Webhook handling must tolerate repeats.** De-duplicate on `X-CallMissed-Delivery` and write by id.

## Endpoint summary

| Purpose | Method and path | Scope |
| --- | --- | --- |
| Create-or-update a contact | `POST /api/v1/contacts/upsert` | `contacts:write` + `contacts:read` |
| Create-or-update a company | `POST /api/v1/companies/upsert` | `companies:write` + `companies:read` |
| Create-or-update a lead | `POST /api/v1/crm/leads/upsert` | `crm_leads:write` + `crm_leads:read` |
| Contact by your id | `GET /api/v1/contacts?external_id=system:id` | `contacts:read` |
| Company by your id | `GET /api/v1/companies?external_id=system:id` | `companies:read` |
| Lead by your id | `GET /api/v1/crm/leads?external_ref=…` | `crm_leads:read` |
| Changed since | `GET` contacts, companies, `crm/leads`, `crm/deals`, `crm/tasks` with `?updated_after=` | the list's `:read` scope |
| Subscribe to changes | `POST /api/v1/webhooks` | `webhooks:write` + each record type's `:read` scope |
| Delivery log, replay | `GET /api/v1/webhooks/{webhook_id}/deliveries`, `POST …/deliveries/{delivery_id}/replay` | `webhooks:write` |
| Call detail | `GET /api/v1/telephony/calls/{call_id}` | `telephony:read` |
| Call transcript | `GET /api/v1/telephony/calls/{call_id}/transcript?format=json\|txt\|srt` | `telephony:read` |
| Call recording | `GET /api/v1/telephony/calls/{call_id}/recording` | `telephony:read` |
| Merge duplicates | `POST /api/v1/crm/search/merge` | `crm_search:write` |
