---
title: "Agent Evals"
description: "Regression-test a voice agent against scripted personas with pass/fail assertions, and read the transcript of every case."
slug: "voice-evals"
breadcrumb: "Voice Agents"
---

# Agent Evals

Regression-test a voice agent against scripted personas with pass/fail assertions, and read the transcript of every case.

## Overview

An **eval suite** is a regression test for one voice agent. Each **case** in the suite gives a simulated caller a persona and an opening line, lets the conversation run for up to a fixed number of turns, and then checks the transcript against **success criteria**.

Running a suite produces a **run** — a pass count plus the full transcript and per-assertion result for every case. Use it before promoting a prompt change, exactly as you would a test suite.

> **Running a suite calls models and costs credits.** Everything else on this page is free. See [Billing](#billing).

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| List/get suites, list cases, list/get runs | `evals:read` |
| Create/update/delete suites and cases, **run a suite** | `evals:write` |

## Limits

| Thing | Limit |
| --- | --- |
| Cases executed per run | **50** |
| Turns per case | `1..20`, default `6` |
| Success criteria per case | 20 |
| Suite name | 255 characters |
| Persona | 4,000 characters |
| Opening line | 2,000 characters |

A suite may **store** more than 50 cases; the cap is on what one run executes.

---

## Suites

```json
{
  "id": "aa10…",
  "bot_id": "b1f2…",
  "name": "Booking flow — regression",
  "description": "Covers the happy path plus three refusals.",
  "scorecard_id": "sc33…",
  "is_active": true,
  "created_at": "2026-08-12T09:00:00Z",
  "updated_at": "2026-08-12T09:00:00Z"
}
```

Attaching a `scorecard_id` adds a graded score on top of the pass/fail assertions.

### GET `/api/v1/voice/evals`

Newest first. Filters: `bot_id`, `is_active`. `limit` `1..200` (default `50`), `offset` `0..100000`.

### POST `/api/v1/voice/evals`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `bot_id` | `UUID` | Yes | Must be your agent |
| `name` | `string` | Yes | 1–255 characters, unique per agent |
| `description` | `string` | No | At most 500 characters |
| `scorecard_id` | `UUID` | No | Must be your scorecard |
| `is_active` | `boolean` | No | Default `true` |

Returns `201`. `409 A suite named '…' already exists for this bot` on a duplicate.

### GET / PATCH / DELETE `/api/v1/voice/evals/{suite_id}`

`DELETE` returns `204` and cascades the suite's cases **and its run history**.

---

## Cases

```json
{
  "id": "bb20…",
  "suite_id": "aa10…",
  "name": "Caller wants a Saturday slot",
  "persona": "An impatient customer in Pune who only has Saturdays free and dislikes being put on hold.",
  "opening": "Hi, can I move my appointment to Saturday?",
  "max_turns": 6,
  "success_criteria": [
    { "type": "contains", "value": "Saturday", "role": "agent" },
    { "type": "tool_called", "value": "reschedule_appointment" },
    { "type": "max_turns_under", "value": 5 }
  ],
  "position": 0,
  "created_at": "2026-08-12T09:05:00Z",
  "updated_at": "2026-08-12T09:05:00Z"
}
```

### Success criteria

| `type` | `value` | Passes when |
| --- | --- | --- |
| `contains` | text, at most 500 characters | The transcript contains the text |
| `not_contains` | text | The transcript does not contain it |
| `regex` | pattern, at most 200 characters | The pattern matches |
| `tool_called` | tool name | The agent invoked that tool |
| `max_turns_under` | integer `1..20` | The conversation finished in fewer turns |
| `ends_with_handoff` | omitted | The call ended in a handoff to a human |

Optional per criterion: `role` (`agent` — the default, `caller`, or `any`) and `case_sensitive` for the text types.

Design the criteria as assertions about **outcomes**, not exact wording: `tool_called` and `ends_with_handoff` survive a prompt rewrite, `contains` on a whole sentence will not.

### GET `/api/v1/voice/evals/{suite_id}/cases`

Ordered by `position`, then oldest first. `limit` `1..200` (default `100`), `offset` `0..100000`.

### POST `/api/v1/voice/evals/{suite_id}/cases`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters |
| `persona` | `string` | Yes | 1–4,000 characters |
| `opening` | `string` | Yes | 1–2,000 characters |
| `max_turns` | `integer` | No | `1 <= n <= 20`, default `6` |
| `success_criteria` | `object[]` | No | At most 20 |
| `position` | `integer` | No | `0 <= position <= 10000`, default `0` |

### PATCH / DELETE `/api/v1/voice/evals/cases/{case_id}`

Note the path: cases are addressed directly, **not** under their suite.

---

## Running a suite

### POST `/api/v1/voice/evals/{suite_id}/run`

No body. Returns `201` with the run and every case result.

```bash
curl -X POST https://api.callmissed.com/api/v1/voice/evals/aa10…/run \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "id": "run77…",
  "suite_id": "aa10…",
  "status": "completed",
  "started_at": "2026-08-17T08:00:00Z",
  "finished_at": "2026-08-17T08:01:44Z",
  "total_cases": 12,
  "passed_cases": 11,
  "model": "kimi-k2.6",
  "cost_credits": 3.812,
  "created_at": "2026-08-17T08:00:00Z",
  "results": [
    {
      "id": "res01…",
      "run_id": "run77…",
      "case_id": "bb20…",
      "passed": true,
      "transcript": [
        { "role": "caller", "content": "Hi, can I move my appointment to Saturday?" },
        { "role": "agent", "content": "Of course — I can move it to Saturday." }
      ],
      "assertions": [
        { "type": "contains", "value": "Saturday", "passed": true }
      ],
      "score": 0.92,
      "error": null,
      "created_at": "2026-08-17T08:00:12Z"
    }
  ]
}
```

The call is **synchronous** — it returns when every case has finished, so allow a generous client timeout for a large suite.

### Nothing is charged before the work starts

Checks run in this order, and a failure at any step costs nothing and writes nothing:

1. Suite and agent loaded and confirmed yours.
2. Cases fetched and the 50-case cap checked.
3. Scorecard loaded, if attached.
4. Credit balance checked.
5. Only then does any model run.

| Status | Detail |
| --- | --- |
| `402` | `Insufficient credits to run an eval suite. Add credits to use this feature.` |
| `409` | `This suite has no cases to run.` |
| `422` | `A run executes at most 50 cases. Split this suite.` |
| `404` | `Eval suite not found` / `Bot not found` / `Scorecard not found` |

An over-cap suite is **rejected, not truncated** — a silently-shortened run would report a green result it did not earn.

## Runs

### GET `/api/v1/voice/evals/runs`

Newest first. Filter by `suite_id`. `limit` `1..100` (default `25`), `offset` `0..100000`.

### GET `/api/v1/voice/evals/runs/{run_id}`

The run plus its case results, oldest first, **capped at 50 results** — the same bound as a run.

## Billing

Only `POST /{suite_id}/run` charges. The cost is the agent model's usage across every case, plus the scoring model when a scorecard is attached, deducted after the run completes and visible in [usage logs](/docs/usage-api) as `service: "llm"`.

Cost scales with `cases × (max_turns × 2 + 1)` model calls, so trimming `max_turns` is the cheapest lever. A run that completes but whose deduction fails is still returned to you in full.

## Errors

| Status | When |
| --- | --- |
| `402` | Credit balance exhausted at the pre-run gate |
| `403` | Key is missing `evals:read` / `evals:write` |
| `404` | Suite, case, run, agent or scorecard not in your tenant |
| `409` | Duplicate suite name, or an empty suite |
| `422` | Blank name/persona/opening, over 20 criteria, an unknown criterion type, or over 50 cases in a run |
