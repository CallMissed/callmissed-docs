---
title: "Call Notes & Scoring"
description: "Turn a finished call or chat into structured notes, push them to your CRM, and grade the conversation against a QA scorecard you write — with a transcript quote behind every score."
slug: "call-analysis"
breadcrumb: "Voice Agents"
---

# Call Notes & Scoring

Turn a finished call or chat into structured notes, push them to your CRM, and grade the conversation against a QA scorecard you write — with a transcript quote behind every score.

## Overview

Two post-conversation tools, both keyed by a **conversation id** (the `id` from [Conversations](/docs/conversations); every voice call and chat has one):

- **Call notes** — `POST /api/v1/calls/{conversation_id}/notes` reads the transcript and writes a structured note: a summary, action items, a disposition, a follow-up date and tags. Edit it, then push it to your CRM.
- **Call scoring** — write a **scorecard** (a rubric of weighted criteria), then `POST /api/v1/calls/{conversation_id}/score` grades the conversation against it. Every criterion comes back with a 0–100 score **and the transcript line that justifies it**.

Both run one language-model completion and are **billed in credits** at that model's per-token [LLM rate](/docs/pricing). Reading, editing and pushing notes, and managing scorecards, are free.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| Read notes, list/get scorecards, read a score | `conversations:read` |
| Generate/edit/push notes, create/update/delete scorecards, score a call | `conversations:write` |

Base URL: `https://api.callmissed.com`. Errors are `{"detail": "..."}`. A conversation or scorecard outside your workspace returns `404`, never `403`.

---

## Call notes

### The notes object

```json
{
  "conversation_id": "c0ffee00-1111-2222-3333-444455556666",
  "notes": {
    "summary": "Caller asked to move her Thursday appointment. Agent offered Friday 11:00, which she accepted.",
    "action_items": ["Send the Friday confirmation by WhatsApp"],
    "disposition": "rescheduled",
    "follow_up_date": "2026-10-09",
    "tags": ["appointment", "reschedule"],
    "generated_at": "2026-10-02T09:30:00+00:00",
    "edited_at": null,
    "edited_by": null,
    "pushed": null
  }
}
```

`notes` is `null` until notes have been generated.

| Field | Type | Notes |
| --- | --- | --- |
| `summary` | `string` | Up to 2,000 characters |
| `action_items` | `string[]` | Up to 10 items, 300 characters each |
| `disposition` | `string \| null` | Short outcome label, up to 64 characters |
| `follow_up_date` | `string \| null` | `YYYY-MM-DD`. A date the model could not express this way is dropped, not guessed |
| `tags` | `string[]` | Up to 10, 40 characters each, lower-cased |
| `generated_at` | `string` | ISO 8601 |
| `edited_at` | `string \| null` | Set by `PATCH` |
| `edited_by` | `string \| null` | The dashboard user who edited; `null` for an API-key edit |
| `pushed` | `object \| null` | After a CRM push: `{provider, at, note_id, contact_id}` |

### GET `/api/v1/calls/{conversation_id}/notes`

Returns the notes object — `notes: null` if none have been generated yet.

### POST `/api/v1/calls/{conversation_id}/notes`

Generates notes from the transcript and saves them.

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `force` | `boolean` | No | `false` | Regenerate even if notes exist. Without it, existing notes are returned unchanged and **nothing is billed** |

```bash
curl -X POST https://api.callmissed.com/api/v1/calls/c0ffee00-1111-2222-3333-444455556666/notes \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{}'
```

The note is written by the conversation agent's configured `model`, or `gpt-oss-120b` when the agent has none, and billed at that model's token rate.

| Status | Cause |
| --- | --- |
| `400` | The conversation has no messages yet |
| `402` | `Insufficient credits`, or `Monthly budget cap reached` |
| `404` | Conversation not found |
| `502` | The model failed or returned an unusable note — retry |

### PATCH `/api/v1/calls/{conversation_id}/notes`

Edit the stored notes. Every field is optional; only what you send changes.

| Field | Type | Notes |
| --- | --- | --- |
| `summary` | `string` | Up to 2,000 characters. Cannot be empty (`422`) |
| `action_items` | `string[]` | Replaces the list |
| `disposition` | `string` | Up to 64 characters |
| `follow_up_date` | `string` | `YYYY-MM-DD`; `""` clears it. Anything else is `422 follow_up_date must be YYYY-MM-DD` |
| `tags` | `string[]` | Replaces the list |

`404` when no notes have been generated yet.

### POST `/api/v1/calls/{conversation_id}/notes/push`

Pushes the stored notes to your CRM as a note, attached to the matching CRM contact when the conversation's contact has an email that exists there.

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `provider` | `string` | `"hubspot"` | `hubspot` is the only CRM available today |

Connect HubSpot under **Integrations** in the console first.

```json
{
  "provider": "hubspot",
  "note_id": "81234567890",
  "contact_id": "5123",
  "associated": true
}
```

`associated` is `false` when no matching contact was found — the note is still created, just not attached.

| Status | Cause |
| --- | --- |
| `400` | Unknown CRM provider |
| `404` | Conversation not found, or no notes generated yet |
| `409` | `salesforce`, `zoho` or `pipedrive` (not available yet), or HubSpot is not connected / needs reconnecting |
| `502` | HubSpot rejected the push |

---

## Scorecards

A scorecard is your QA rubric.

```json
{
  "id": "5c0a1b2c-3d4e-4f50-8a6b-7c8d9e0f1a2b",
  "name": "Support QA",
  "criteria": [
    { "label": "Verified caller identity", "weight": 2, "guidance": "Asked for name and booking reference before discussing the booking." },
    { "label": "Resolved or escalated", "weight": 3, "guidance": null },
    { "label": "Polite close", "weight": 1, "guidance": null }
  ],
  "created_at": "2026-10-01T08:00:00Z",
  "updated_at": "2026-10-01T08:00:00Z"
}
```

| Criterion field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `label` | `string` | Yes | 1–200 characters, unique within the scorecard (case-insensitive) |
| `weight` | `number` | No (default `1`) | `0 < weight <= 100`. **Relative** importance — weights need not sum to anything |
| `guidance` | `string` | No | Up to 1,000 characters. What a good answer looks like |

A scorecard has 1–20 criteria. A rule violation returns `400` naming the criterion.

### GET `/api/v1/scorecards`

Newest first. `limit` `1..500` (default `100`), `offset` `0..100000`.

### POST `/api/v1/scorecards`

Body: `name` (1–200 characters) and `criteria`. Returns `201` with the scorecard.

```bash
curl -X POST https://api.callmissed.com/api/v1/scorecards \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Support QA",
    "criteria": [
      {"label": "Verified caller identity", "weight": 2},
      {"label": "Resolved or escalated", "weight": 3},
      {"label": "Polite close", "weight": 1}
    ]
  }'
```

### GET `/api/v1/scorecards/{scorecard_id}`

Returns one scorecard.

### PUT `/api/v1/scorecards/{scorecard_id}`

Replaces `name` and `criteria` (both required, same rules as create). Scores already taken keep their own breakdown, so past grades are unchanged.

### DELETE `/api/v1/scorecards/{scorecard_id}`

Returns `204`. Scores taken with it survive with `scorecard_id: null`.

---

## Scoring a call

### POST `/api/v1/calls/{conversation_id}/score`

| Field | Type | Required |
| --- | --- | --- |
| `scorecard_id` | `UUID` | Yes |

```bash
curl -X POST https://api.callmissed.com/api/v1/calls/c0ffee00-1111-2222-3333-444455556666/score \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"scorecard_id": "5c0a1b2c-3d4e-4f50-8a6b-7c8d9e0f1a2b"}'
```

```json
{
  "conversation_id": "c0ffee00-1111-2222-3333-444455556666",
  "scorecard_id": "5c0a1b2c-3d4e-4f50-8a6b-7c8d9e0f1a2b",
  "total": 76.67,
  "breakdown": [
    { "criterion": "Verified caller identity", "score": 90, "evidence_quote": "Agent: Could I have your booking reference, please?" },
    { "criterion": "Resolved or escalated", "score": 70, "evidence_quote": "Agent: I've moved you to Friday at eleven." },
    { "criterion": "Polite close", "score": 60, "evidence_quote": null }
  ],
  "model": "gpt-5-mini",
  "created_at": "2026-10-02T09:31:00Z"
}
```

- `total` is the **weight-weighted mean** of the criterion scores, on a 0–100 scale.
- `breakdown` always follows your rubric, in order. A criterion the model skipped scores `0` rather than disappearing, so a missing line can never inflate the total.
- `evidence_quote` is a verbatim transcript line, or `null` when nothing in the transcript supports the criterion.
- Re-scoring the same conversation with the same scorecard **replaces** the previous score.
- Billed at the `model`'s LLM token rate. Long transcripts are trimmed from the middle before grading, keeping the opening and the close.

| Status | Cause |
| --- | --- |
| `402` | Credit balance is zero |
| `404` | Conversation or scorecard not found |
| `409` | The conversation has no transcript yet |
| `502` | The model returned an unreadable response (still billed — retry) |
| `503` | Scoring is temporarily unavailable — retry |

### GET `/api/v1/calls/{conversation_id}/score`

The **newest** stored score for the conversation, across all scorecards. `404 This call has not been scored yet` if there is none.
