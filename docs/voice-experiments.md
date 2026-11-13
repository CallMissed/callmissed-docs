---
title: "A/B Experiments"
description: "Split voice traffic across agent variants, assign callers deterministically, and read per-arm results against a chosen metric."
slug: "voice-experiments"
breadcrumb: "Voice Agents"
---

# A/B Experiments

Split voice traffic across agent variants, assign callers deterministically, and read per-arm results against a chosen metric.

## Overview

An **experiment** compares variants of one voice agent on a single metric. Each variant is an **arm**: a bot version, or a set of overrides (system prompt, voice, model, timing). Exactly one arm is the **control**.

`traffic_split` decides what share of callers each arm gets. While an experiment is `running`, every call to its agent is routed automatically — inbound and outbound phone calls, WhatsApp calls and web-widget calls — and served by the arm the caller is assigned to. `GET /results` reports the metric per arm. `POST /assign` previews or records an assignment yourself.

### How calls are routed

- **Sticky per person.** A caller is bucketed by their phone number (or widget visitor id), so the same person hears the same arm on every call while the split is unchanged.
- **One running experiment per agent.** Starting a second one on the same agent returns `409`.
- **What an arm changes on a live call:** its `bot_version_number` (that published version's prompt and settings), `system_prompt`, `model`, `voice` and `timing.allow_interruptions`. The arm's model and voice win over a phone number's own settings.
- **Only calls the agent answers itself count.** When a call flow, call menu or call queue answers the number first, the call is not routed. When an agent squad answers, the squad's entry agent is routed by its own experiment.
- **Not routed:** console test calls, sessions where you set `system_prompt`, `llm_model` or `voice` yourself, every call once the experiment is stopped or concluded, and any call where your plan no longer allows the arm's model. A call that is not routed is answered by the agent's live configuration and counted for no arm.

Nothing on this page consumes credits — the calls the experiment configures are billed as normal voice usage.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List/get experiments, read results | `experiments:read` |
| Create, edit arms, start/stop/conclude, **assign** | `experiments:write` |

`assign` needs the **write** scope — it records a durable assignment.

## Lifecycle

```
draft ──start──▶ running ──stop──▶ stopped ──conclude──▶ concluded
                     │                                      ▲
                     └──────────────conclude────────────────┘
```

| Status | What it means |
| --- | --- |
| `draft` | Being configured. The metric can still be changed |
| `running` | Routing calls to arms. Arms are frozen except for renaming |
| `stopped` | Not routing calls. Arms can be edited again, and it can restart |
| `concluded` | Terminal and **immutable** — a winner is recorded and nothing can change |

Stop before editing an arm; conclude only when you are done for good.

## Metrics

| `metric` | Label | Better |
| --- | --- | --- |
| `goal_completed` | Goal completion rate | Higher |
| `avg_score` | Average scorecard total | Higher |
| `handoff_rate` | Human-handoff rate | Lower |
| `completion_rate` | Call completion rate | Higher |
| `avg_duration_seconds` | Average call duration | Lower |

The deciding metric can only be changed while the experiment is a `draft` — picking the winner after seeing the data is exactly what that rule prevents.

## The experiment object

```json
{
  "id": "ex10…",
  "tenant_id": "a0b1…",
  "bot_id": "b1f2…",
  "name": "Shorter opening line",
  "hypothesis": "A one-sentence greeting raises goal completion.",
  "status": "running",
  "traffic_split": { "arm-a-id": 50, "arm-b-id": 50 },
  "metric": "goal_completed",
  "winner_arm_id": null,
  "started_at": "2026-08-14T09:00:00Z",
  "stopped_at": null,
  "created_at": "2026-08-13T09:00:00Z",
  "updated_at": "2026-08-14T09:00:00Z",
  "arms": []
}
```

`GET` and every mutating call return the detail shape, with `arms` populated oldest first.

## GET `/api/v1/voice/experiments`

Newest first. Filters: `bot_id`, `status`. `limit` `1..200` (default `50`), `offset` `0..100000`.

## POST `/api/v1/voice/experiments`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `bot_id` | `UUID` | Yes | Must be your agent |
| `name` | `string` | Yes | At most 255 characters, not blank, unique per agent |
| `metric` | `string` | Yes | One of the five metrics |
| `hypothesis` | `string` | No | At most 500 characters |

Created as `draft` with an empty `traffic_split`. Returns `201`.

## Arms

```json
{
  "id": "arm-b-id",
  "tenant_id": "a0b1…",
  "experiment_id": "ex10…",
  "name": "short-greeting",
  "bot_version_number": 12,
  "overrides": {
    "system_prompt": "Greet in one sentence, then ask how you can help.",
    "voice": "shubh",
    "timing": { "interrupt_sensitivity": 0.6, "silence_timeout_ms": 2000 }
  },
  "is_control": false,
  "created_at": "2026-08-13T09:05:00Z"
}
```

### POST `/api/v1/voice/experiments/{experiment_id}/arms`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | At most 64 characters, unique within the experiment |
| `bot_version_number` | `integer` | No | `1 <= n <= 1000000` |
| `overrides` | `object` | No | See below |
| `is_control` | `boolean` | No | Default `false`. Only one arm may be the control |

**At most 6 arms per experiment.**

### `overrides`

Unknown keys are rejected with `422` rather than ignored.

| Field | Type | Constraints |
| --- | --- | --- |
| `system_prompt` | `string` | At most 8,000 characters |
| `voice` | `string` | At most 64 characters |
| `model` | `string` | At most 100 characters |
| `timing` | `object` | See the timing table |
| `node_timing` | `object` | `{ node_id: timing }`, at most 100 entries, node ids at most 64 characters |

#### Timing fields

Only `allow_interruptions` is applied to live calls today. Start refuses an arm that sets any other timing field or `node_timing` (`422`), so an experiment never reports on a setting no caller heard.

| Field | Type | Range |
| --- | --- | --- |
| `allow_interruptions` | `boolean` | |
| `interrupt_sensitivity` | `number` | `0.0`–`1.0` |
| `resume_delay_ms` | `integer` | `0`–`5000` |
| `silence_timeout_ms` | `integer` | `500`–`30000` |
| `max_node_duration_ms` | `integer` | `1000`–`600000` |

### PATCH / DELETE `/api/v1/voice/experiments/arms/{arm_id}`

Arms are addressed directly, not under their experiment. While the experiment is `running` you may change only `name` — anything else returns `409 Stop the experiment before changing an arm's configuration`, because a mid-flight change would silently mix two configurations into one arm's numbers.

Deleting an arm also removes it from `traffic_split` in the same transaction.

## Traffic split

`traffic_split` maps every arm id to a whole-number percentage.

| Rule | Error when broken |
| --- | --- |
| Must name every arm, and only arms of this experiment | `traffic_split is missing arm(s): …` / `…names arm(s) that do not belong…` |
| Percentages are whole numbers in `0..100` | `traffic_split percentages must be whole numbers` |
| Must sum to exactly 100 | `traffic_split must sum to 100 (got 90)` |
| Exactly one arm is the control | `exactly one arm must be the control (found 0)` |

Set it with `PATCH /{experiment_id}`, or pass it on start.

## Start, stop, conclude

### POST `/api/v1/voice/experiments/{experiment_id}/start`

Optional body `{ "traffic_split": { … } }`; falls back to the stored split. Needs **at least two arms** — `422 An experiment needs at least two arms to compare`.

Every arm is checked before any call is routed to it: a `bot_version_number` must be a published version of the agent, its `model` and `voice` must be ones the agent's speech stack can run (and a `model` is refused on an agent with a `voice_tier`, which picks its own), and every arm's model — its override, else its pinned version's, else the agent's — must be allowed on your plan. A problem returns `422` (or `403` for the plan) naming the arm.

### POST `/api/v1/voice/experiments/{experiment_id}/stop`

No body. `409 This experiment is not running` if it was not.

### POST `/api/v1/voice/experiments/{experiment_id}/conclude`

| Field | Type | Required |
| --- | --- | --- |
| `winner_arm_id` | `UUID` | Yes — must be an arm of this experiment |

After this the experiment is immutable. A `draft` cannot be concluded — `409 Start the experiment before concluding it`.

## Assignment

### POST `/api/v1/voice/experiments/{experiment_id}/assign`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `voice_session_id` | `UUID` | Conditional | Must be your session |
| `key` | `string` | Conditional | At most 128 characters |

Send at least one. When both are present the session id is the bucketing key.

```bash
curl -X POST https://api.callmissed.com/api/v1/voice/experiments/ex10…/assign \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "voice_session_id": "vs99…" }'
```

```json
{
  "experiment_id": "ex10…",
  "arm_id": "arm-b-id",
  "arm_name": "short-greeting",
  "voice_session_id": "vs99…",
  "assignment_id": "as55…",
  "created": true,
  "assigned_at": "2026-08-17T08:20:00Z"
}
```

Bucketing is **deterministic** — the same key always lands in the same arm for the same split, so a returning caller keeps their variant.

Assignment by session is **idempotent**: a repeat call returns the existing row with `created: false`, and a concurrent double-call re-reads the winner rather than creating two.

> A **key-only** call writes nothing. `assignment_id` comes back `null` and the result is a preview of which arm that key maps to. Use it to plan; use `voice_session_id` to record.

`409 This experiment is not running; no traffic is assigned` outside the running state.

## Results

### GET `/api/v1/voice/experiments/{experiment_id}/results`

No parameters.

```json
{
  "experiment_id": "ex10…",
  "status": "running",
  "metric": "goal_completed",
  "metric_label": "Goal completion rate",
  "higher_is_better": true,
  "min_sample_per_arm": 30,
  "total_assignments": 412,
  "sufficient_data": true,
  "leader_arm_id": "arm-b-id",
  "winner_arm_id": null,
  "verdict": "short-greeting is ahead on goal completion rate",
  "arms": [
    {
      "arm_id": "arm-a-id", "name": "control", "is_control": true,
      "sample_size": 205, "metric_value": 0.61,
      "analysed_calls": 198, "goal_met_rate": 0.61,
      "positive_sentiment_rate": 0.44, "negative_sentiment_rate": 0.12,
      "avg_duration_seconds": 142.5, "avg_cost_credits": 3.8
    },
    {
      "arm_id": "arm-b-id", "name": "short-greeting", "is_control": false,
      "sample_size": 207, "metric_value": 0.68,
      "analysed_calls": 201, "goal_met_rate": 0.68,
      "positive_sentiment_rate": 0.51, "negative_sentiment_rate": 0.09,
      "avg_duration_seconds": 118.0, "avg_cost_credits": 3.1
    }
  ]
}
```

| Field | Notes |
| --- | --- |
| `sufficient_data` | `false` while any arm has fewer than **30** assignments, or fewer than two arms have a value |
| `leader_arm_id` | Currently ahead on the metric. Not a verdict |
| `winner_arm_id` | Only set once you conclude |
| `goal_met_rate`, `positive_sentiment_rate`, `negative_sentiment_rate` | From each call's post-call analysis, whatever the deciding metric. `null` until a call in the arm is analysed |
| `avg_duration_seconds`, `avg_cost_credits` | Per call. The cost is the call's AI usage in credits (phone carrier minutes are not included) |

> There is deliberately **no p-value or significance field**. `sufficient_data` is a floor, not a test — treat `leader_arm_id` as a signal to keep running, and decide the winner yourself.

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `experiments:read` / `experiments:write` |
| `404` | Experiment, arm, agent or voice session not in your tenant |
| `409` | Concluded and immutable, already running / not running, another experiment already running on the agent, or an arm edit while running |
| `422` | Over 6 arms, a second control, fewer than two arms on start, a `traffic_split` that does not add up, an arm that cannot be served on a live call, or a winner that is not an arm of the experiment |
