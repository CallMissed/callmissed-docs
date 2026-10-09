---
title: "Caller ID pools"
description: "Let one outbound campaign dial from several of your own numbers. Create a pool, add numbers with optional daily limits, and attach it to a campaign."
slug: "caller-id-pools"
breadcrumb: "Numbers (PSTN)"
---

# Caller ID pools

Let one outbound campaign dial from several of your own numbers. Create a pool, add numbers with optional daily limits, and attach it to a campaign.

## Overview

A **caller ID pool** is a set of your own phone numbers that an outbound [campaign](/docs/voice-campaigns) can dial from, instead of a single number. Each call picks one number from the pool, so volume is spread across numbers you hold and no single number carries all of it.

**Base path:** `https://api.callmissed.com/api/v1/caller-id-pools`

**Authentication.** Every endpoint accepts a **JWT** (`Authorization: Bearer <jwt>`) or an **API key** (`Authorization: Bearer cm_<key>`). API-key callers need `telephony:read` to list and get pools and `telephony:write` to create, change or delete them and to add or remove numbers. Everything is scoped to your account; another account's pool or number returns `404`.

## Which number is used

For each call the dialer looks at the pool's numbers and uses the best eligible one. A number is **eligible** only when it:

- is one of your numbers and is active,
- has been declared for automated calls, when TRAI readiness is switched on for your account,
- has no open spam complaint against it, and
- has not reached its daily limit.

Among eligible numbers, the pool's `strategy` decides:

| Strategy | Behaviour |
| --- | --- |
| `round_robin` | The number used least recently goes next |
| `local_presence` | Prefers a number in the same country as the person being called and, for US and Canadian (+1) numbers, the same area code. If none matches it falls back to `round_robin` |

A pool is a way to spread calls across numbers you legitimately hold. It does not make an ineligible number usable, and it is not a way around carrier or TRAI rules: numbers that fail the checks above are skipped, and if none is eligible the campaign pauses with a message saying so. If every number has hit its daily limit the campaign simply waits, and the limits reset at the start of the next day in the campaign's timezone.

## Create a pool

`POST /` · scope `telephony:write` · `201 Created`

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `name` | string (1–120) | Yes | Your own label |
| `strategy` | `round_robin` \| `local_presence` | No | Default `round_robin` |
| `enabled` | boolean | No | Default `true`. A disabled pool is ignored and the campaign uses its own number |

An account can hold up to 20 pools.

```bash
curl -X POST https://api.callmissed.com/api/v1/caller-id-pools \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"name": "Mumbai outreach", "strategy": "local_presence"}'
```

## List, get, update, delete

| Method and path | Scope | Notes |
| --- | --- | --- |
| `GET /?limit=&offset=` | `telephony:read` | `limit` 1–100. Each pool includes `number_count` |
| `GET /{pool_id}` | `telephony:read` | Includes the pool's `numbers` with `daily_cap`, `uses_today` and `last_used_at` |
| `PATCH /{pool_id}` | `telephony:write` | Any of `name`, `strategy`, `enabled` |
| `DELETE /{pool_id}` | `telephony:write` | `204`. Campaigns that used it go back to their own number |

## Add and remove numbers

`POST /{pool_id}/numbers` · scope `telephony:write`

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `phone_number_id` | uuid | Yes | One of your active numbers (see [Numbers](/docs/telephony-api)) |
| `daily_cap` | integer 1–100000 | No | Most calls this number places per day through the pool. Omit for no limit |

A number you do not own returns `404`, a number that is not active returns `422`, and a number already in the pool returns `409`. A pool holds up to 50 numbers.

`PATCH /{pool_id}/numbers/{phone_number_id}` changes `daily_cap` (send `null` to remove the limit). `DELETE /{pool_id}/numbers/{phone_number_id}` removes the number from the pool and returns `204`; the number itself is untouched.

## Use a pool in a campaign

Set `caller_id_pool_id` when you create a campaign, or on `PATCH /api/v1/voice-campaigns/{id}`. Send `null` on a patch to detach the pool. A pool your account does not own returns `404`. While the pool is enabled it takes the place of the campaign's `from_number_id`.

```bash
curl -X PATCH https://api.callmissed.com/api/v1/voice-campaigns/CAMPAIGN_ID \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"caller_id_pool_id": "POOL_ID"}'
```
