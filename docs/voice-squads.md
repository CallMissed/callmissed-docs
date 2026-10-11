---
title: "Agent Squads"
description: "Group specialist voice agents behind one entry point, hand a live call between them, control handoffs with a policy, dry-run the routing decision, and draft a new agent from a description."
slug: "voice-squads"
breadcrumb: "Voice Agents"
---

# Agent Squads

Group specialist voice agents behind one entry point, hand a live call between them, control handoffs with a policy, dry-run the routing decision, and draft a new agent from a description.

## Overview

A **squad** is several specialist voice agents behind one entry point. The entry agent answers, and hands the live call to a member when the caller's need matches that member's role. The member gets the whole conversation so far, introduces itself and carries on, so the caller never repeats themselves. A **handoff policy** bounds how far that can go, so a call cannot bounce between agents forever.

A call runs as a squad when you [bind the squad to a phone number](#answering-a-phone-number), pass [`squad_id` when you create a voice session](#web-and-api-sessions), or set [`squad_id` on an agent](#an-agents-own-squad). See [Live handoff](#live-handoff-on-a-call) for what happens on the call.

Two extras sit alongside the roster:

- `POST /{squad_id}/simulate-handoff` — a pure dry run that shows which member *would* be picked and why.
- `POST /author/draft` — describe an agent in prose and get a complete configuration proposal back. **This one costs credits.**

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List/get squads, list members, **simulate a handoff** | `squads:read` |
| Create/update/delete squads and members, **draft an agent** | `squads:write` |
| Bind or unbind a phone number | `squads:write` **and** `telephony:write` |

## Limits

| Thing | Limit |
| --- | --- |
| Members per squad | **12**. Past that, use a call flow |
| Handoffs per call | `0..5`, default `3` |
| Role name | 64 characters |
| Member description | 2,000 characters |

---

## Squads

```json
{
  "id": "sq10…",
  "tenant_id": "a0b1…",
  "name": "Support desk",
  "description": "Front line plus billing and technical specialists.",
  "entry_bot_id": "b1f2…",
  "handoff_policy": {
    "max_handoffs": 3,
    "allow_return_to_previous": false,
    "min_score": 1,
    "fallback_role": "generalist"
  },
  "is_active": true,
  "created_at": "2026-08-11T09:00:00Z",
  "updated_at": "2026-08-11T09:00:00Z",
  "members": []
}
```

### Handoff policy

| Field | Type | Default | Constraints |
| --- | --- | --- | --- |
| `max_handoffs` | `integer` | `3` | `0 <= n <= 5`. `0` disables handoffs entirely |
| `allow_return_to_previous` | `boolean` | `false` | Leaving this `false` is what stops two agents ping-ponging a caller |
| `min_score` | `integer` | `1` | `1 <= n <= 20`. The match strength a member must reach to be handed to |
| `fallback_role` | `string \| null` | `null` | Role to use when nothing scores high enough |

Unknown keys inside the policy are rejected with `422`.

### GET `/api/v1/voice/squads`

Newest first. Filter by `is_active`. `limit` `1..200` (default `50`), `offset` `0..100000`.

### POST `/api/v1/voice/squads`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, unique per tenant |
| `description` | `string` | No | At most 500 characters |
| `entry_bot_id` | `UUID` | Yes | The agent that answers the call |
| `entry_role` | `string` | No | 1–64 characters, default `entry` |
| `entry_description` | `string` | No | At most 2,000 characters |
| `handoff_policy` | `object` | No | Omit for the defaults above |
| `is_active` | `boolean` | No | Default `true` |

Creating a squad **automatically enrols the entry agent as the first member** at `position: 0` — you do not add it yourself.

Returns `201` with the squad and its roster.

### GET / PATCH / DELETE `/api/v1/voice/squads/{squad_id}`

`PATCH` takes `name`, `description`, `entry_bot_id`, `handoff_policy` (an explicit `null` clears it back to defaults) and `is_active`.

`entry_bot_id` must point at an agent **already in the squad** — `422 entry_bot_id must be a bot that is already a member of this squad`. Add the member first, then promote it.

`DELETE` returns `204` and cascades the roster.

---

## Members

```json
{
  "id": "mb20…",
  "tenant_id": "a0b1…",
  "squad_id": "sq10…",
  "bot_id": "b7c8…",
  "role": "billing",
  "description": "Handles invoices, refunds and payment failures.",
  "position": 1,
  "created_at": "2026-08-11T09:02:00Z"
}
```

`role` and `description` are what the routing engine matches a caller's utterance against — write the description as the things this agent handles, in the caller's words.

### GET `/api/v1/voice/squads/{squad_id}/members`

Ordered by `position`, then oldest first. **No pagination** — the 12-member cap bounds it.

### POST `/api/v1/voice/squads/{squad_id}/members`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `bot_id` | `UUID` | Yes | Must be your agent, not already in this squad |
| `role` | `string` | Yes | 1–64 characters, not blank |
| `description` | `string` | No | At most 2,000 characters |
| `position` | `integer` | No | `0 <= position <= 1000`, default `0` |

`422 A squad holds at most 12 agents. Past that, use a call flow.` · `409 That agent is already in this squad`.

Members share one live call, so every member must run **the same kind of voice pipeline** as the squad's entry agent (all speech-to-text + language model + text-to-speech, or all speech-to-speech models such as `gpt-realtime`) and be on **the same [voice tier](/docs/voice-tiers)**, because a call is billed at one tier. An agent on a managed voice-agent model (`deepgram-voice-*`, or the tier built on one) cannot take over a call from another agent. Each case is a `422` with the reason.

### PATCH / DELETE `/api/v1/voice/squads/members/{member_id}`

`PATCH` takes `role`, `description` (explicit `null` clears it) and `position`. `bot_id` is not editable — remove the member and add the other agent.

Removing the entry agent returns `409 This agent answers the call for the squad. Point entry_bot_id at another member before removing it.`

---

## Live handoff on a call

While a call runs as a squad, the agent answering it can hand the caller to another member. It sees every other member's `role` and `description` and picks one when the caller's need is clearly that member's job. Then:

1. The handoff is checked against the squad's [handoff policy](#handoff-policy): the per-call cap, no handing straight back to the member that just handed over (unless `allow_return_to_previous`), no handing to itself, and only roles on the roster. The agent can propose a member; it cannot get around these rules.
2. The new member takes the call in the same session: same caller, same recording, same transcript. It receives the conversation so far, introduces itself in one sentence and continues. It does not repeat the greeting or the recording notice.
3. The handoff is recorded on the session (`metadata.squad_handoffs` on [`GET /v1/voice/sessions/{id}`](/docs/voice-sessions-api#get-session): from and to agent, roles, reason, time), the session's `bot_id` moves to the new member, and a [`call.handoff`](/docs/webhooks#squad-handoff-event) webhook is sent.

From the handoff on, the new member's prompt, voice (including a [cloned voice](/docs/voice-cloning)), language, model, tools, [fallback transfer](/docs/voice-agent-tools#transfer_to_human) and [moderation](/docs/voice-moderation) settings apply, and its usage is billed at its own model rates (or the squad's tier). Settings fixed when the call started stay with the call: turn-taking, the pronunciation dictionary, the moderation blocklist, call recording and its notice, and the AI disclosure (spoken once, at the start of the call).

An agent that is inactive is never handed a call. An agent in [zero data retention](/docs/voice-data-retention) mode can only hand a call to another agent in that mode (the handoff is refused with `data_retention` otherwise), so content from the first part of the call is never stored by the next agent. If a handoff cannot be completed, the agent keeps the caller and carries on.

---

## Answering a phone number

### POST `/api/v1/voice/squads/{squad_id}/numbers/{number_id}`

Answer inbound calls on one of your numbers with this squad. The squad's **entry agent** answers, with the number's own call settings (greeting, voice and so on) applied as they would be for the number's agent. Needs `squads:write` and `telephony:write`.

```bash
curl -X POST https://api.callmissed.com/api/v1/voice/squads/sq10…/numbers/pn40… \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "status": "bound", "squad_id": "sq10…", "number_id": "pn40…" }
```

A number answers with one squad; binding another squad replaces it. A [call flow, call menu or call queue](/docs/call-handling-api#how-a-number-picks-one) on the same number answers first, and the squad does not apply to those calls. An inactive squad, or one whose entry agent is inactive, leaves the number answering with its own agent.

### DELETE `/api/v1/voice/squads/{squad_id}/numbers/{number_id}`

Stop answering the number with this squad. Returns `{"status": "unbound", …}`. Idempotent: a number bound to a different squad, or to none, is left as it is.

`404 Squad not found` · `404 Phone number not found` (not one of your numbers).

---

## Web and API sessions

Pass `squad_id` to [`POST /v1/voice/sessions`](/docs/voice-sessions-api#request-body). The squad's entry agent answers, or `bot_id` when you name another active member. The answering agent uses its own prompt, greeting and voice; any field you send in the request still wins. `404` for an unknown or inactive squad, `422` for a `bot_id` that is not an active member.

---

## An agent's own squad

Set `squad_id` in an agent's configuration and the agent hands calls within that squad on any call it answers or places, as long as it is a member. A value that is not a squad id is reported by config validation; remove it with `{ "unset": ["squad_id"] }`.

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/{bot_id}/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "values": { "squad_id": "sq10…" } }'
```

---

## Simulating a handoff

### POST `/api/v1/voice/squads/{squad_id}/simulate-handoff`

Requires `squads:read`. Pure and read-only: no model call, no credits, nothing written.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `utterance` | `string` | Yes | 1–2,000 characters |
| `handoffs_used` | `integer` | No | `0 <= n <= 100`, default `0` |
| `current_member_id` | `UUID` | No | Who is handling the call now |
| `previous_member_id` | `UUID` | No | Who handled it before — used for the ping-pong check |

```bash
curl -X POST https://api.callmissed.com/api/v1/voice/squads/sq10…/simulate-handoff \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "utterance": "my card was charged twice", "handoffs_used": 1, "current_member_id": "mb01…" }'
```

```json
{
  "squad_id": "sq10…",
  "handoff": true,
  "target_member_id": "mb20…",
  "target_bot_id": "b7c8…",
  "target_role": "billing",
  "reason": "matched billing on 'charged'",
  "blocked_by": null,
  "score": 3,
  "handoffs_used": 1,
  "max_handoffs": 3,
  "scores": [
    { "member_id": "mb20…", "role": "billing", "score": 3, "eligible": true },
    { "member_id": "mb30…", "role": "technical", "score": 0, "eligible": true }
  ]
}
```

`scores` shows every member's match strength, so a wrong route is debuggable: if the right agent scored `0`, its description is missing the words callers actually use.

### Why a handoff was blocked

| `blocked_by` | Meaning |
| --- | --- |
| `no_members` | The squad has no one to hand to |
| `max_handoffs` | The policy's handoff budget is spent |
| `ping_pong` | The target is the previous member and returns are disallowed |
| `already_current` | The best match is already handling the call |
| `no_match` | Nothing reached `min_score` |
| `unknown_role` | A routing model named a role that is not on the roster. Only produced by model-based routing on a live call; this keyword dry run never returns it |

---

## Drafting an agent

### POST `/api/v1/voice/squads/author/draft`

Requires `squads:write`. **Charges credits.** It creates nothing — you get a proposal to review and then submit yourself via the bots API.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `description` | `string` | Yes | 20–4,000 characters, not blank |
| `bot_type` | `string` | No | `inbound_call` (default), `outbound_call`, `ivr`, `whatsapp` or `whatsapp_voice` |

```bash
curl -X POST https://api.callmissed.com/api/v1/voice/squads/author/draft \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "An agent for a Pune dental clinic that books, moves and cancels appointments in Hindi and English, and escalates anything about pain to a human.",
    "bot_type": "inbound_call"
  }'
```

```json
{
  "draft": {
    "name": "Clinic Reception",
    "bot_type": "inbound_call",
    "system_prompt": "You are the receptionist for a dental clinic in Pune…",
    "objective": "Book, reschedule or cancel appointments; escalate pain reports.",
    "response_guidelines": "Keep replies under two sentences…",
    "conversation_script": "",
    "first_message": "Namaste, thanks for calling. How can I help?",
    "tools": ["calcom_list_slots", "calcom_book", "escalate_to_human"],
    "voice_model": "…",
    "tts_model": "…",
    "stt_model": "…",
    "voice": "anushka",
    "language": "hi-IN"
  },
  "dropped_tools": ["send_invoice"],
  "model": "…"
}
```

`dropped_tools` lists tools the draft asked for that are not in your tool registry — they were removed so the configuration is valid on submission. Check this list: a dropped tool usually means the capability you described is not wired up yet.

At most 12 tools are proposed, de-duplicated and validated against the registry.

| Status | Detail | Note |
| --- | --- | --- |
| `402` | `Not enough credits to draft an agent. Top up to continue.` | Checked **before** any model runs — costs nothing |
| `402` | `Monthly budget cap reached` | Your account's monthly budget cap is spent. Also checked before any model runs |
| `422` | `Could not draft an agent: …` | The model returned an unusable draft. **This attempt is still billed** — the work was done |
| `503` | `The agent drafting service is unavailable.` | Retry |

Cost appears in [usage logs](/docs/usage-api) as `service: "llm"`.

## Errors

| Status | When |
| --- | --- |
| `402` | Credit balance exhausted, or the monthly budget cap reached, before drafting |
| `403` | Key is missing `squads:read` / `squads:write` |
| `404` | Squad, member, agent or phone number not in your tenant |
| `409` | Duplicate squad name, agent already a member, or removing the entry agent |
| `422` | Over 12 members, a blank name/role, an unknown key in `handoff_policy`, an `entry_bot_id` that is not a member, or a member on a different voice pipeline or tier |
