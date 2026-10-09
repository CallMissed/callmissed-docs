---
title: "Voice Data Retention"
description: "Delete an agent's call transcripts and recordings after a set number of days, or run it with zero data retention so no call content is kept at all."
slug: "voice-data-retention"
breadcrumb: "Voice Agents"
---

# Voice Data Retention

Delete an agent's call transcripts and recordings after a set number of days, or run it with zero data retention so no call content is kept at all.

## Overview

By default CallMissed keeps a call's transcript and recording until you delete them. Two per-agent controls change that:

- **A retention window** — `transcript_retention_days` and `recording_retention_days` delete an agent's call transcripts and recordings once they are older than the window. A window applies to calls made after you save it; older calls follow your account's retention setting.
- **Zero data retention (ZDR)** — `data_retention_mode: "zdr"` stops the agent's calls from keeping any content in the first place.

Both are keys in the agent's `config`, set with `PATCH /api/v1/bots/{bot_id}/config` like any other agent setting. An account-wide window and an account-wide ZDR switch are also available to owners and admins in the console (**Settings → Data retention**).

## Authentication

```
Authorization: Bearer cm_your_api_key
```

The key needs the `bots:write` scope to change an agent's config.

## Retention windows

| Key | Type | Effect |
| --- | --- | --- |
| `transcript_retention_days` | integer, 1-3650 | Erase the words of this agent's calls once they are older than this many days. Applies to calls made after you save it |
| `recording_retention_days` | integer, 1-3650 | Delete this agent's call recordings once they are older than this many days. Applies to calls made after you save it |

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/{bot_id}/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "values": { "transcript_retention_days": 30, "recording_retention_days": 7 } }'
```

- **Applies to calls made after you save this.** Older calls follow your account's retention setting. Changing a window starts it again from that change, so a shorter window never reaches calls made before it. The agent's `config` shows when each window was saved as `transcript_retention_set_at` and `recording_retention_set_at`; the server sets these, and any value you send for them is ignored.
- **An agent's window can only shorten the account-wide one.** The window applied to a call is the shorter of the two. With no account-wide window, the agent's window applies on its own.
- **Unset means keep.** Remove a window with `{ "unset": ["transcript_retention_days"] }`; the account-wide setting (or no deletion at all) applies again.
- Deletion runs in the background, so content is removed within a few hours of passing its window, not to the second.
- A malformed value (`0`, `3651`, a string, a decimal) is rejected with `422` and nothing is saved.

### What is erased and what is kept

When a call passes its **transcript** window, its turn text, its [post-call analysis](/docs/call-analysis) summary and extracted fields are erased. When it passes its **recording** window, the recording is deleted and [Get Recording](/docs/voice-sessions-api) returns `404` for it.

Everything you need for billing and reporting stays: the session itself, its duration, status, cost, turn count, per-turn latency and token counts, and the analysis sentiment, disposition and goal flag. A transcript request for an erased call returns its turns with empty text.

Erasure cannot be undone. Export anything you need to keep before you shorten the account-wide window.

## Zero data retention

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/{bot_id}/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "values": { "data_retention_mode": "zdr" } }'
```

| `data_retention_mode` | Behaviour |
| --- | --- |
| `standard` (default) | Call content is stored, and deleted only by a retention window |
| `zdr` | No call content is stored |

Under `zdr`, for every call the agent takes:

- **No recording.** The call is not recorded even if `record_calls` is `true`, and no recording notice is spoken.
- **No transcript.** Each turn keeps its timing, latency and token counts, but not the words.
- **No post-call analysis** and no automatic call memory.

The call is billed exactly as before. When the account-wide ZDR switch is on, every agent behaves as if it were set to `zdr`.
