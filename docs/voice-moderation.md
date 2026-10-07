---
title: "Voice Moderation and AI Disclosure"
description: "Tell callers they are speaking with an AI, screen what callers and the agent say for harmful content, keep the agent from saying blocked words, and list every finding through the API or a webhook."
slug: "voice-moderation"
breadcrumb: "Voice Agents"
---

# Voice Moderation and AI Disclosure

Tell callers they are speaking with an AI, screen what callers and the agent say for harmful content, keep the agent from saying blocked words, and list every finding through the API or a webhook.

## Overview

Two per-agent settings, both keys in the agent's `config`, set with `PATCH /api/v1/bots/{bot_id}/config` like any other agent setting:

- **`ai_disclosure`** speaks a short sentence before the greeting so callers know they are talking to an AI.
- **`moderation`** screens the call: what the caller says, what the agent says, and a list of words the agent must never speak.

Every finding is stored, listed by [`GET /api/v1/voice/moderation-events`](#list-findings) and sent to your [webhooks](/docs/webhooks) as `call.moderation_flagged`.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

Changing an agent's config needs the `bots:write` scope. Listing findings needs `bots:read`.

## AI disclosure

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/{bot_id}/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "values": { "ai_disclosure": { "enabled": true, "text": "Heads up, you are speaking with an AI assistant." } } }'
```

| Field | Type | Meaning |
| --- | --- | --- |
| `enabled` | boolean | Speak the disclosure before the greeting |
| `text` | string, 1-300 characters | The sentence to speak. Omit it for a short neutral default |

- The disclosure is spoken once per call, ahead of the [recording notice](/docs/voice-agent) when the call is recorded, whether the call opens on the greeting, a call menu or a queue hold.
- It is spoken as written, before anything else the agent says.
- **New agents** created from the starter template have it turned on. Existing agents are not changed; turn it off with `{ "unset": ["ai_disclosure"] }` or `"enabled": false`.

## Moderation

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/{bot_id}/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "values": { "moderation": {
        "enabled": true,
        "categories": ["hate", "violence"],
        "action": "warn",
        "severity_threshold": "medium",
        "blocklist": ["darn"]
      } } }'
```

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `enabled` | boolean | `false` | Nothing is screened unless this is `true` |
| `categories` | array | all four | Any of `hate`, `self_harm`, `sexual`, `violence` |
| `action` | string | `log` | What to do when the **caller** says something flagged: `log`, `warn`, `end_call` or `transfer` |
| `severity_threshold` | string | `medium` | Lowest severity that counts: `low`, `medium` or `high` |
| `blocklist` | array | none | Up to 50 words or phrases (1-64 characters) the agent must not speak |
| `blocklist_replacement` | string | `...` | What is spoken in place of a blocked term |

A malformed setting (an unknown category or action, an oversized blocklist) is rejected with `422` and nothing is saved.

### What the caller says

Each finished caller sentence is classified as the call goes on. When a category in `categories` is found at or above `severity_threshold`, the finding is stored and `action` runs:

| Action | What happens on the call |
| --- | --- |
| `log` | Nothing. The finding is stored and sent to your webhook |
| `warn` | The agent says, calmly, that it can't continue that way and steers back to helping. At most three times per call |
| `end_call` | The agent says a short goodbye and ends the call |
| `transfer` | The caller is handed to a person: a live transfer to your first [transfer target](/docs/voice-agent-tools) on a phone call (the agent tells the caller it is connecting them, or that the team will call back if nobody can be reached), otherwise a callback request in your handoffs queue. If neither is possible the call ends politely |

On a call handed between agents ([squads](/docs/voice-squads), call menus, call queues), each sentence is screened with the settings of the agent answering at that moment; the blocklist stays the one of the agent that answered the call.

Screening never slows the call down. If the classifier is slow, unavailable or not enabled for your workspace, the sentence is treated as not flagged and the call carries on. Moderation is a safeguard, not a guarantee that every harmful sentence is caught.

### What the agent says

- **Blocklist.** A blocked word is replaced in the speech with `blocklist_replacement` before the caller hears it. Matching is whole-word and case-insensitive. The saved transcript keeps the agent's original words, and each rewrite is stored as a finding with category `blocklist`.
- **Classification.** Each finished agent message is also classified afterwards. Findings about the agent are **log only**: the call is never acted on for what the agent said.

## List findings

```bash
curl "https://api.callmissed.com/api/v1/voice/moderation-events?direction=caller&limit=20" \
  -H "Authorization: Bearer cm_your_api_key"
```

| Query | Meaning |
| --- | --- |
| `bot_id`, `session_id` | Only this agent / this call |
| `direction` | `caller` or `agent` |
| `category` | `hate`, `self_harm`, `sexual`, `violence` or `blocklist` |
| `action` | `log`, `warn`, `end_call` or `transfer` |
| `since` | Only findings at or after this ISO 8601 time |
| `limit` | 1-200, default 50 |
| `offset` | Default 0 |

```json
[
  {
    "id": "7f0b3c1e-0d5a-4f29-9a5e-5b1c2f0d9e11",
    "session_id": "c3a1c2a0-61f4-4c8e-8a39-0f6b1f6c8a52",
    "bot_id": "2b7e6a53-27f0-4d2c-8d3e-1c0a4f9a7b10",
    "direction": "caller",
    "category": "hate",
    "severity": "high",
    "action": "warn",
    "excerpt": "...",
    "created_at": "2026-10-08T09:41:07.512Z"
  }
]
```

Newest first. `severity` is `low`, `medium` or `high`.

### Excerpts and data retention

`excerpt` holds at most the first 200 characters of the flagged speech. When the agent, or your whole workspace, runs with [zero data retention](/docs/voice-data-retention), the finding is kept but `excerpt` is `null`: you learn that a category was hit, never the words.

## Webhook

Subscribe to `call.moderation_flagged` ([webhooks](/docs/webhooks)) to be told as it happens. The payload carries the same fields as a listed finding, plus `event_id`:

```json
{
  "event_id": "7f0b3c1e-0d5a-4f29-9a5e-5b1c2f0d9e11",
  "session_id": "c3a1c2a0-61f4-4c8e-8a39-0f6b1f6c8a52",
  "bot_id": "2b7e6a53-27f0-4d2c-8d3e-1c0a4f9a7b10",
  "direction": "caller",
  "category": "hate",
  "severity": "high",
  "action": "warn",
  "excerpt": null,
  "created_at": "2026-10-08T09:41:07.512Z"
}
```

## Limits

- Moderation and AI disclosure apply to calls your agents take by phone, on the web and on WhatsApp. They do not apply to a [managed voice agent](/docs/managed-voice-agent) WebSocket connection, where you set the opening line yourself.
- Content screening is off for your workspace until it is switched on for you; until then `blocklist` and `ai_disclosure` work and the other settings are stored without effect.
