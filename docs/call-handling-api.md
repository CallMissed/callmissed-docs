---
title: "Call Handling API"
description: "Decide what happens when someone calls you: build call flows, press-1 menus and call queues, set the message left on an answering machine, and check whether a number has been flagged as spam — all with your cm_ key."
slug: "call-handling-api"
breadcrumb: "Numbers (PSTN)"
---

# Call Handling API

Decide what happens when someone calls you: build call flows, press-1 menus and call queues, set the message left on an answering machine, and check whether a number has been flagged as spam — all with your cm_ key.

> An inbound call has to be answered by *something*. These five surfaces are that something: a **flow** (a step-by-step graph), a **menu** ("press 1 for sales"), a **queue** (hold until someone is free), a **voicemail message** for outbound calls that reach a machine, and a **reputation** check on the number itself.

:::cards
/docs/telephony-api | Telephony API | hash | Rent a number, link it to an agent, place calls
/docs/agent-tools-mcp | Account MCP Server | wrench | The same setup as tools an AI agent can call
:::

## Credential class: dashboard JWT **or** a `cm_` key

```
Authorization: Bearer cm_your_api_key
```

Every endpoint on this page takes either. API-key callers need:

| Action | Scope |
| --- | --- |
| Listing, reading, simulating, live state, reputation check | `telephony:read` |
| Creating, updating, publishing, attaching a number, roster changes, deleting | `telephony:write` |

Call handling is an attribute of telephony, so it reuses those two scopes and adds none. Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`. Every create (`POST` on a collection) returns `201 Created`, and every `DELETE` of a flow, menu, queue, member or voicemail message returns `204 No Content`.

<Callout type="info">
  Flows, menus and queues can be built **before** calling is switched on for your account — they only read and write your own configuration. The voicemail-message and reputation endpoints need calling enabled and return `404` until it is.
</Callout>

## How a number picks one

A number's binding is stored on the number itself, and each surface has its own attach call:

| Surface | Attach | Detach |
| --- | --- | --- |
| Flow | `POST /api/v1/call-flows/{id}/numbers/{number_id}` | `DELETE` the same path |
| Menu | `POST /api/v1/call-menus/{id}/numbers/{number_id}` | `DELETE` the same path |
| Queue | `POST /api/v1/call-queues/{id}/numbers/{number_id}` | `DELETE` the same path |

Attaching takes effect from the next inbound call on that number. A flow answers only once it is **published**, and a menu only while it is **enabled**; until then the number answers with its agent as usual.

If one number carries more than one binding, the first that can answer wins: a published **flow**, then an enabled **menu**, then the **queue**. A number answered by a flow or menu reaches its queues through a key or a `route` step instead of putting every caller on hold first.

Detach is idempotent: unbinding a number that is not bound is a success, not an error. Deleting the flow, menu or queue a number points at is also safe — the call simply answers normally, because the binding is re-checked against your own rows on every call.

Both ids are checked against your account on every one of these calls, so a number can never be pointed at another account's flow, menu or queue.

## What the caller experiences

A flow or menu runs before any agent speaks. The caller hears the prompt (your `prompt_audio_url` recording if you set one, otherwise the `prompt` text read aloud) and presses a key. A key pressed while the prompt is still playing is accepted straight away. Pressing nothing within `timeout_seconds`, or a key that maps nowhere, replays the prompt until `max_retries` is spent. If the call is recorded, the recording notice plays before the first prompt.

Where each destination takes the call:

| Destination | What happens |
| --- | --- |
| `bot` | That agent takes the call and greets the caller with its own greeting, prompt, voice and tools. The call, its transcript and its analytics move to that agent |
| `queue` | The caller joins that queue and holds until a member is free |
| `phone` | The number is dialled and the caller is connected as soon as someone answers. If nobody answers, the number's own agent picks the call back up |
| `hangup` | The call ends |
| a `voicemail` node | After the prompt the caller leaves a message. It ends after a few seconds of quiet or at `max_duration_seconds`, and lands in your [handoffs](/docs/handoffs) queue as a callback request with the caller's words and number, filed under the agent linked to the number |

If a step cannot complete — a menu deleted mid-call, an agent that is switched off, or an agent that runs on a different voice tier or a different kind of voice model from the number's agent (a speech-to-speech model in place of a standard one, or the reverse) — the number's own agent answers the call instead, so a caller is never left in silence. Use agents on the same voice tier and model kind as the number's agent for `bot` destinations.

## Call flows

A flow is a directed graph of steps. Each step is a **node** with a `type` and a `config`; **edges** connect them.

### The flow object

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "name": "Front desk",
  "description": "Business hours, then a menu",
  "graph": { "entry": "hello", "nodes": [], "edges": [] },
  "published_graph": { "entry": "hello", "nodes": [], "edges": [] },
  "version": 3,
  "published": true,
  "published_at": "2026-09-08T12:00:00Z",
  "created_at": "2026-09-01T09:00:00Z",
  "updated_at": "2026-09-08T11:59:00Z"
}
```

| Field | Notes |
| --- | --- |
| `graph` | The **draft**. What you are editing |
| `published_graph` | The **snapshot** live calls actually walk. A copy taken at publish time, so editing the draft afterwards cannot change what callers hear |
| `version` | Incremented on every publish |
| `published` | `false` means the flow answers nothing, even while `published_graph` is kept |

### Node types

| `type` | What it does | `config` | Branches it may declare in `edges` |
| --- | --- | --- | --- |
| `greeting` | Says something and moves on | `prompt` (≤ 2000 chars), `prompt_audio_url` | `next` |
| `dtmf_menu` | Asks for a keypress and routes on it | `prompt`, `prompt_audio_url`, `options`, `timeout_seconds` (1–60, default 5), `max_retries` (0–5, default 2), `invalid_action` (`reprompt` or `hangup`) | none — its branches live in `options` |
| `business_hours` | Splits on whether you are open | `timezone` (an IANA name), `hours`, `holidays` (≤ 60 `YYYY-MM-DD` dates) | `open`, `closed` |
| `route` | Terminal: hands the call over | `destination` | none |
| `voicemail` | Takes a message (see [What the caller experiences](#what-the-caller-experiences)) | `prompt`, `prompt_audio_url`, `max_duration_seconds` (5–600, default 120) | none |
| `webhook` | Calls your endpoint mid-call | `url` (required), `method` (`GET` or `POST`), `timeout_seconds` (1–10), `headers` (≤ 10) | `ok`, `error` |

A destination — used by `route.destination` and by each key in a `dtmf_menu`'s `options` — is `{"type": ..., "value": ..., "label": ...}`:

| `type` | `value` |
| --- | --- |
| `bot` | One of your agents' ids |
| `queue` | One of your call queues' ids |
| `phone` | A number in E.164 form, e.g. `+919812345678`. Premium-rate and other blocked ranges are refused with `422` when you save, and checked again when a call is forwarded |
| `menu` | **Inside a flow this means another NODE in the same flow.** The id of that node, so a key hop stays in the graph |
| `hangup` | None |

`hours` is keyed by `mon` `tue` `wed` `thu` `fri` `sat` `sun`, each a list of up to 6 `{"start": "09:00", "end": "18:00"}` windows. A day with no windows is closed all day. An unknown timezone, a malformed `HH:MM`, or a holiday that is not a real date is rejected at write time rather than silently misrouting a call later.

<Callout type="warn">
  A `webhook` node's URL is checked when you save it **and** re-checked when the call fires it. Internal, loopback and link-local addresses are refused both times, so a flow cannot be used to reach something that is not on the public internet.
</Callout>

### GET /api/v1/call-flows

Your flows, newest first. Requires `telephony:read`.

| Parameter | Type | Notes |
| --- | --- | --- |
| `limit` | `int` | 1–100, default 50 |
| `offset` | `int` | 0–100000, default 0 |

```bash
curl https://api.callmissed.com/api/v1/call-flows \
  -H "Authorization: Bearer cm_your_api_key"
```

### POST /api/v1/call-flows

Create a flow. Requires `telephony:write`. It answers nothing until it is published and a number is attached, so a half-built graph is safe to save.

| Field | Type | Notes |
| --- | --- | --- |
| `name` | `string` | **Required.** 1–255 characters |
| `description` | `string` | ≤ 1024 characters |
| `graph` | `object` | `entry`, `nodes` (≤ 200), `edges` (≤ 600). Defaults to an empty graph |

```bash
curl -X POST https://api.callmissed.com/api/v1/call-flows \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Front desk",
    "graph": {
      "entry": "hello",
      "nodes": [
        {"id": "hello", "type": "greeting", "config": {"prompt": "Thanks for calling Acme."}},
        {"id": "pick", "type": "dtmf_menu", "config": {
          "prompt": "Press 1 for sales, 2 for support.",
          "options": {
            "1": {"type": "bot", "value": "8f14e45f-ceea-467a-9f6b-0f9a7bd1c111", "label": "Sales"},
            "2": {"type": "queue", "value": "3d9a1b22-0f4c-4a61-9d33-3f2a51b0c222", "label": "Support"}
          }
        }}
      ],
      "edges": [{"from": "hello", "to": "pick", "on": "next"}]
    }
  }'
```

Structural problems are `422` with a message naming the node: a duplicate node id, an edge pointing at a node that does not exist, a branch the node type cannot take, or two branches of the same name leaving one node. A destination id that is not yours is `404`.

`entry` may be omitted — the first node becomes the entry point — but an explicit one is clearer.

### GET /api/v1/call-flows/\{flow_id\}

One flow, with both the draft and the published snapshot. Requires `telephony:read`.

### PATCH /api/v1/call-flows/\{flow_id\}

Update the **draft**. Requires `telephony:write`. Live calls keep hearing the published snapshot until you publish again.

`name`, `description` and `graph` are all optional; an omitted field keeps its stored value and `"description": ""` clears it. **`graph` is replaced wholesale**, because a canvas is edited as one unit and a per-node merge would make deleting a node impossible. Send the whole graph.

### POST /api/v1/call-flows/\{flow_id\}/publish

Snapshot the draft into `published_graph`, bump `version`, and start answering real calls with it. Requires `telephony:write`.

The draft is re-validated first — including re-checking that every destination still belongs to you — so a queue deleted since the flow was drawn is caught here instead of on a live call. A flow with no nodes, or whose entry node is missing, is refused with `422`.

```bash
curl -X POST https://api.callmissed.com/api/v1/call-flows/{flow_id}/publish \
  -H "Authorization: Bearer cm_your_api_key"
```

### POST /api/v1/call-flows/\{flow_id\}/unpublish

Stop answering with this flow. Requires `telephony:write`. The snapshot and the version are kept, so re-publishing is one call. Idempotent, and attached numbers answer normally in the meantime.

### DELETE /api/v1/call-flows/\{flow_id\}

Delete the flow. Requires `telephony:write`. Numbers still carrying it answer normally from the next call.

### POST /api/v1/call-flows/\{flow_id\}/simulate

Dry-run one step and get back the instruction a real call would receive. Requires `telephony:read`. Nothing is dialled and nothing is charged.

| Field | Type | Notes |
| --- | --- | --- |
| `node_id` | `string` | Where the call is. Omit to start at the entry node |
| `event` | `string` | `start`, `done` or `dtmf`. Default `start` |
| `pressed_key` | `string` | The key pressed. Omit with `dtmf` to test a no-press timeout |
| `attempt` | `int` | 0–6. `0` is the first ask, `1` the first reprompt |
| `use_draft` | `bool` | Default `true`. `false` walks the published snapshot |
| `at` | `string` | An ISO timestamp to pretend it is — the way to test `business_hours` without waiting for 2am |

Webhook nodes are **not** called during a simulation. Such a node takes its `ok` branch and the response lists it under `simulated_webhook_nodes`, so you never mistake a dry run for a real delivery.

```bash
curl -X POST https://api.callmissed.com/api/v1/call-flows/{flow_id}/simulate \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"event": "dtmf", "node_id": "pick", "pressed_key": "1"}'
```

## Call menus

A standalone press-1-for-sales step, for when a whole flow is more than you need. Same idea as a flow's `dtmf_menu` node, but it is its own object and attaches to a number directly.

```json
{
  "id": "3d9a1b22-0f4c-4a61-9d33-3f2a51b0c222",
  "name": "Main menu",
  "prompt": "Press 1 for sales, 2 for support.",
  "prompt_audio_url": null,
  "options": {
    "1": { "type": "bot", "value": "8f14e45f-ceea-467a-9f6b-0f9a7bd1c111", "label": "Sales" }
  },
  "timeout_seconds": 5,
  "invalid_action": "reprompt",
  "max_retries": 2,
  "enabled": true,
  "created_at": "2026-09-01T09:00:00Z",
  "updated_at": "2026-09-01T09:00:00Z"
}
```

| Endpoint | Scope | Notes |
| --- | --- | --- |
| `GET /api/v1/call-menus` | `telephony:read` | `limit` 1–100 (default 50), `offset` 0–100000 |
| `POST /api/v1/call-menus` | `telephony:write` | `name` required |
| `GET /api/v1/call-menus/{menu_id}` | `telephony:read` | |
| `PATCH /api/v1/call-menus/{menu_id}` | `telephony:write` | Omitted fields keep their value |
| `DELETE /api/v1/call-menus/{menu_id}` | `telephony:write` | |

| Field | Type | Notes |
| --- | --- | --- |
| `name` | `string` | 1–255 characters |
| `prompt` | `string` | ≤ 2000 characters. What the caller hears |
| `prompt_audio_url` | `string` | A public `https` recording to play instead of speaking the prompt. `""` clears it |
| `options` | `object` | Keys `"1"` to `"9"`, each `{"type", "value", "label"}` — the same destination shape as a flow, except that here `menu` means **another call menu's id** |
| `timeout_seconds` | `int` | 1–60, default 5 |
| `invalid_action` | `string` | `reprompt` or `hangup`. Default `reprompt` |
| `max_retries` | `int` | 0–5, default 2 |
| `enabled` | `bool` | `false` keeps the menu but stops it answering |

`options` is **replaced wholesale** on `PATCH`, so send every key you want to keep. A menu pointing at itself is refused (`422`) — it would loop the caller forever. A key outside `1`–`9`, or a destination missing its `value`, is also `422`; a destination id that is not yours is `404`.

```bash
curl -X POST https://api.callmissed.com/api/v1/call-menus \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Main menu",
    "prompt": "Press 1 for sales, 2 for support.",
    "options": {
      "1": {"type": "bot", "value": "8f14e45f-ceea-467a-9f6b-0f9a7bd1c111", "label": "Sales"},
      "2": {"type": "queue", "value": "3d9a1b22-0f4c-4a61-9d33-3f2a51b0c222", "label": "Support"}
    }
  }'
```

## Call queues

A queue holds callers until someone on its roster is free.

```json
{
  "id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "name": "Support",
  "strategy": "round_robin",
  "hold_audio_url": null,
  "position_announce_seconds": null,
  "max_wait_seconds": 300,
  "overflow_action": "voicemail",
  "enabled": true,
  "created_at": "2026-09-01T09:00:00Z",
  "updated_at": "2026-09-01T09:00:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `name` | `string` | 1–255 characters |
| `strategy` | `string` | `round_robin`, `longest_idle` or `priority`. Default `round_robin` |
| `hold_audio_url` | `string` | A public `https` audio clip (up to 5 MB) looped while the caller holds. `""` clears it, and callers then hear built-in hold music |
| `position_announce_seconds` | `int` | `0` (or unset) never announces; otherwise 10–600. The caller hears their place in line this often |
| `max_wait_seconds` | `int` | 0–3600, default 300. `0` means hold indefinitely |
| `overflow_action` | `string` | `voicemail`, `hangup` or `callback`. What happens after `max_wait_seconds` |
| `enabled` | `bool` | `false` keeps the queue but stops it taking calls |

While a caller holds, the agent does not listen to them and the AI per-minute fee does not run; it starts when the caller is routed to an agent. A caller routed to a person is promised a callback, and the request appears in your handoffs.

| Endpoint | Scope | Notes |
| --- | --- | --- |
| `GET /api/v1/call-queues` | `telephony:read` | `limit` 1–100 (default 50), `offset` 0–100000 |
| `POST /api/v1/call-queues` | `telephony:write` | `name` required |
| `GET /api/v1/call-queues/{queue_id}` | `telephony:read` | |
| `PATCH /api/v1/call-queues/{queue_id}` | `telephony:write` | Omitted fields keep their value |
| `DELETE /api/v1/call-queues/{queue_id}` | `telephony:write` | Roster and anyone waiting go with it |

### The roster

A member is either an agent (`kind: "ai"`, `bot_id`) or a person in your workspace (`kind: "human"`, `user_id`). Either way the id must belong to your account, which is what stops a queue routing calls to somewhere else.

| Endpoint | Scope | Notes |
| --- | --- | --- |
| `GET /api/v1/call-queues/{queue_id}/members` | `telephony:read` | `limit` 1–500 (default 100), `offset` 0–100000 |
| `POST /api/v1/call-queues/{queue_id}/members` | `telephony:write` | |
| `PATCH /api/v1/call-queues/{queue_id}/members/{member_id}` | `telephony:write` | |
| `DELETE /api/v1/call-queues/{queue_id}/members/{member_id}` | `telephony:write` | |

| Field | Type | Notes |
| --- | --- | --- |
| `kind` | `string` | `ai` (default) or `human` |
| `bot_id` | `uuid` | Required when `kind` is `ai` |
| `user_id` | `uuid` | Required when `kind` is `human`, and the user must be active |
| `priority` | `int` | 0–1000, default 100. Lower is served sooner |
| `max_concurrent` | `int` | 1–50, default 1. How many calls at once |
| `enabled` | `bool` | `false` keeps them on the roster and sends them nothing |

`PATCH` changes `priority`, `max_concurrent` and `enabled` only — `kind`, `bot_id` and `user_id` are immutable, so point a queue at a different answerer by removing the member and adding a new one. Raising `max_concurrent` or re-enabling a member pulls waiting callers in straight away rather than at the next release.

```bash
curl -X POST https://api.callmissed.com/api/v1/call-queues/{queue_id}/members \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"kind": "ai", "bot_id": "8f14e45f-ceea-467a-9f6b-0f9a7bd1c111", "max_concurrent": 3}'
```

### GET /api/v1/call-queues/\{queue_id\}/live

Who is holding right now. Requires `telephony:read`. `limit` is 1–200, default 50.

```json
{
  "queue_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "strategy": "round_robin",
  "waiting": 2,
  "members_total": 4,
  "members_free": 1,
  "longest_wait_seconds": 96,
  "callers": [
    {
      "entry_id": "1f0c2e77-1b2a-4a55-8f0e-9b1c2d3e4f50",
      "caller_suffix": "…4321",
      "position": 1,
      "waiting_seconds": 96,
      "priority": 100,
      "enqueued_at": "2026-09-08T12:00:00Z"
    }
  ]
}
```

A caller's full number is never returned here — only the last four digits, which is enough to match someone on screen. This is a point-in-time read; poll it no faster than you need.

## Voicemail messages

What an **outbound** call leaves when an answering machine picks up. Needs calling enabled on your account.

```json
{
  "id": "2f8a7c10-5b6d-4e3f-8a1b-7c9d0e1f2a3b",
  "tenant_id": "0a1b2c3d-4e5f-6071-8293-a4b5c6d7e8f9",
  "name": "Missed you — sales",
  "text": "Hi, this is Acme returning your call. We'll try again tomorrow.",
  "voice": null,
  "language": "en-IN",
  "audio_state": "pending",
  "created_at": "2026-09-01T09:00:00Z",
  "updated_at": "2026-09-01T09:00:00Z"
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `name` | `string` | 1–255 characters |
| `text` | `string` | ≤ 2500 characters. The words to leave |
| `voice` | `string` | ≤ 64 characters. Which voice reads it; omit for the account default |
| `language` | `string` | ≤ 16 characters, e.g. `en-IN` |
| `audio_state` | `string` | `ready` (a clip exists), `pending` (text only — it is turned into speech, and billed, the first time it is used) or `empty` (nothing to leave) |

| Endpoint | Scope | Notes |
| --- | --- | --- |
| `GET /api/v1/voicemail-templates` | `telephony:read` | `limit` 1–100 (default 50), `offset` 0–100000 |
| `POST /api/v1/voicemail-templates` | `telephony:write` | `name` required |
| `GET /api/v1/voicemail-templates/{template_id}` | `telephony:read` | |
| `PATCH /api/v1/voicemail-templates/{template_id}` | `telephony:write` | Omitted fields keep their value |
| `DELETE /api/v1/voicemail-templates/{template_id}` | `telephony:write` | Also removes the cached clip |
| `GET /api/v1/voicemail-templates/{template_id}/audio` | `telephony:read` | `{"url": "..."}` — a short-lived link. A text-only message is synthesised, cached and billed **once** here, so a preview costs the same as the first real drop |
| `POST /api/v1/voicemail-templates/{template_id}/audio` | `telephony:write` | `multipart/form-data` with `file`: a WAV or MP3 up to 5 MB. Bypasses synthesis entirely, so a professionally recorded greeting is used as-is and nothing is billed for speech |

Changing `text` throws away the clip made from the old words, and the next use re-synthesises once. Without that, a call would keep playing the old message forever.

## Number reputation

Whether one of your numbers has been reported as spam, and what is still open against it. Needs calling enabled on your account.

### GET /api/v1/reputation/\{number\}

Requires `telephony:read`. `{number}` is one of your numbers in E.164, with or without the leading `+`. A number you do not own returns `404` — never `403` — so nobody can probe for other accounts' numbers.

```json
{
  "phone_number": "919000000000",
  "status": "pending",
  "complaints": [
    {
      "reference_id": "PUCC-000123",
      "status": "pending",
      "to_number": "919111111111",
      "initiation_date": "2026-09-05",
      "complaint_category": "promotional",
      "opt_in_proof": null,
      "rejection_reason": null,
      "actionable": true
    }
  ],
  "actionable_count": 1,
  "last_checked": "2026-09-08T11:30:00Z",
  "stale": false
}
```

| Field | Notes |
| --- | --- |
| `status` | The worst open standing: `clean`, `pending`, `in_review`, `rejected` or `accepted` |
| `actionable` / `actionable_count` | Complaints that still need proof from you |
| `to_number` | The complainant's own number. You need it on the proof document, and you must stop calling them — continuing to call a complainant is itself a violation |
| `stale` | `true` when the standing could not be refreshed recently, or has never been checked. **Treat a `clean` answer with `stale: true` as unknown, not clean** |

<Callout type="warn">
  An open complaint has a **5-business-day** window to answer in. Missing it risks the number being blocked across India for a year, so check `actionable_count` on a schedule rather than after a problem.
</Callout>

### POST /api/v1/reputation/\{number\}/remediate

Upload opt-in proof for one complaint. Requires `telephony:write`. `multipart/form-data`:

| Part | Notes |
| --- | --- |
| `reference_id` | The complaint's `reference_id`, ≤ 64 characters |
| `file` | PDF, PNG or JPEG, up to 10 MB |

```bash
curl -X POST https://api.callmissed.com/api/v1/reputation/919000000000/remediate \
  -H "Authorization: Bearer cm_your_api_key" \
  -F 'reference_id=PUCC-000123' \
  -F 'file=@opt-in-proof.pdf'
```

```json
{
  "phone_number": "919000000000",
  "reference_id": "PUCC-000123",
  "status": "in_review",
  "message": "Opt-in proof uploaded and the complaint is under review. ..."
}
```

A file over 10 MB returns `413`. Only the file's type and size are checked at upload; the document itself is reviewed afterwards, so a `200` means "under review", not "cleared". The proof must show your business logo, the opt-in date (within the six months before the complaint) and the complainant's exact number, or the review declines it.

## Do this from an AI assistant instead

Every read and write on this page except the two file uploads is also a tool on the [Account MCP Server](/docs/agent-tools-mcp), so an assistant you have connected can build a flow, wire a menu, staff a queue and publish the lot without you writing a single request.
