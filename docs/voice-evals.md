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
| Create a case from a call | `evals:write`, plus the key's `stt`, `tts` and `llm` permissions (the same ones reading a transcript needs) |

## Limits

| Thing | Limit |
| --- | --- |
| Cases executed per run | **50** |
| Turns per case | `1..20`, default `6` |
| Success criteria per case | 20, of which at most 5 `llm_judge` |
| Runs executing at once | 3 per account |
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
| `llm_judge` | plain-language criterion, at most 500 characters | A model reading the whole conversation decides the agent met it |

Optional per criterion: `role` (`agent` — the default, `caller`, or `any`) and `case_sensitive` for the text types. `llm_judge` ignores both: the judge reads the whole conversation, including which tools the agent used.

### Judged criteria

Use `llm_judge` for what a string match cannot check:

```json
{ "type": "llm_judge", "value": "The agent confirms the new date and time before booking it." }
```

Each result entry gets `passed` and a one-sentence `detail` explaining the verdict. All of a case's judged criteria are decided in **one** model call, charged with the run (see [Billing](#billing)). A judge that cannot be reached, or does not answer a criterion, leaves it **failed** — an unchecked criterion never passes.

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

### POST `/api/v1/voice/evals/{suite_id}/cases/from-session`

Turn a real voice session into a case. The caller's first line becomes the `opening`, their next lines shape the `persona` so the simulated caller pursues the same goal, and outcome checks are suggested: an `llm_judge` criterion on the caller's request, plus `ends_with_handoff` when the real call ended in a transfer. Free — no model runs.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `session_id` | `UUID` | Yes | A voice session in your account |
| `name` | `string` | No | 1–255 characters. Default `From call <first 8 characters of the id>` |
| `max_turns` | `integer` | No | `1..20`. Default: one more than the real call took |
| `dry_run` | `boolean` | No | `true` returns the draft without saving it |

```json
{
  "source_session_id": "vs42…",
  "name": "From call vs42ab12",
  "persona": "You are the caller from a real phone call, replaying it …",
  "opening": "Hi, can I move my appointment to Saturday?",
  "max_turns": 4,
  "success_criteria": [
    { "type": "llm_judge", "value": "The agent addresses the caller's request (\"Hi, can I move my appointment to Saturday?\") accurately and helpfully, without inventing facts." }
  ],
  "case": { "id": "bb21…", "suite_id": "aa10…", "…": "…" }
}
```

Returns `201`; `case` is `null` on a dry run. `404 Session not found` for a session outside your account, `422` when the call has no caller speech. The case copies what the caller said into your suite — review it before sharing the suite.

---

## Running a suite

### POST `/api/v1/voice/evals/{suite_id}/run`

No body. Query parameters:

| Parameter | Default | Meaning |
| --- | --- | --- |
| `wait` | `true` | `true` returns `201` when every case has finished. `false` returns `202` at once with `status: "running"`; poll [the run](#runs) for the outcome |
| `min_pass_rate` | none | `0..1`. Adds a `gate` verdict to the response |

```bash
curl -X POST https://api.callmissed.com/api/v1/voice/evals/aa10…/run \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "id": "run77…",
  "suite_id": "aa10…",
  "status": "failed",
  "started_at": "2026-08-17T08:00:00Z",
  "finished_at": "2026-08-17T08:01:44Z",
  "total_cases": 12,
  "passed_cases": 11,
  "model": "kimi-k2.6",
  "cost_credits": 3.812,
  "created_at": "2026-08-17T08:00:00Z",
  "pass_rate": 0.9167,
  "min_pass_rate": 0.9,
  "gate": "pass",
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

`status` is `running` while cases execute, then `passed` (every case passed), `failed` (at least one did not) or `error` (the run could not execute). `pass_rate` is `passed_cases / total_cases` from `0` to `1`, and `null` while the run is still `running`.

With the default `wait=true` the call returns when every case has finished. A large suite can take minutes, longer than many HTTP clients and proxies wait, so for anything automated use `wait=false` and poll.

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
| `429` | `At most 3 eval runs can execute at once. Wait for one to finish.` |
| `409` | `This suite has no cases to run.` |
| `422` | `A run executes at most 50 cases. Split this suite.` |
| `404` | `Eval suite not found` / `Bot not found` / `Scorecard not found` |

An over-cap suite is **rejected, not truncated** — a silently-shortened run would report a green result it did not earn.

## Runs

### GET `/api/v1/voice/evals/runs`

Newest first. Filter by `suite_id`. `limit` `1..100` (default `25`), `offset` `0..100000`.

### GET `/api/v1/voice/evals/runs/{run_id}`

The run plus its case results, oldest first, **capped at 50 results** — the same bound as a run. Accepts `min_pass_rate` (`0..1`) like the run call, and returns the same `gate`.

A run still `running` three hours after it started is reported as `error`: it did not finish, and a poller should stop waiting.

## Gating CI on a suite

`gate` answers one question for a pipeline: did this run clear the bar?

| `gate` | Meaning |
| --- | --- |
| `pass` | The run finished and `pass_rate >= min_pass_rate` |
| `fail` | The run finished below the bar, **or** the run ended in `error` (whatever the bar) |
| `pending` | Still running — poll again |
| `null` | No `min_pass_rate` was sent |

```bash
RUN=$(curl -sf -X POST \
  "https://api.callmissed.com/api/v1/voice/evals/$SUITE_ID/run?wait=false&min_pass_rate=0.9" \
  -H "Authorization: Bearer $CALLMISSED_API_KEY" | jq -r .id)

while :; do
  GATE=$(curl -sf "https://api.callmissed.com/api/v1/voice/evals/runs/$RUN?min_pass_rate=0.9" \
    -H "Authorization: Bearer $CALLMISSED_API_KEY" | jq -r .gate)
  [ "$GATE" != "pending" ] && break
  sleep 15
done
[ "$GATE" = "pass" ]
```

### GitHub Actions

Save this composite action in your repository as `.github/actions/callmissed-eval/action.yml`:

```yaml
name: CallMissed eval gate
description: >-
  Run a CallMissed agent eval suite and fail the job when its pass rate is
  below a threshold. Starts the run with wait=false, then polls the run until
  it reports a gate verdict.

inputs:
  api-key:
    description: A CallMissed API key (cm_...) with the evals:write scope. Pass it from a secret.
    required: true
  suite-id:
    description: The eval suite to run.
    required: true
  min-pass-rate:
    description: Fraction of cases that must pass, 0 to 1.
    required: false
    default: "1"
  timeout-minutes:
    description: Stop waiting after this many minutes and fail the job.
    required: false
    default: "30"
  poll-seconds:
    description: Seconds between polls.
    required: false
    default: "15"
  api-base:
    description: API host.
    required: false
    default: https://api.callmissed.com

outputs:
  run-id:
    description: The eval run id.
    value: ${{ steps.gate.outputs.run-id }}
  status:
    description: passed, failed or error.
    value: ${{ steps.gate.outputs.status }}
  pass-rate:
    description: passed_cases / total_cases, 0 to 1.
    value: ${{ steps.gate.outputs.pass-rate }}
  gate:
    description: pass or fail.
    value: ${{ steps.gate.outputs.gate }}

runs:
  using: composite
  steps:
    - id: gate
      shell: bash
      # Inputs reach the script as environment variables, never interpolated
      # into it, so a crafted input cannot inject shell.
      env:
        CM_API_KEY: ${{ inputs.api-key }}
        CM_SUITE_ID: ${{ inputs.suite-id }}
        CM_MIN_PASS_RATE: ${{ inputs.min-pass-rate }}
        CM_TIMEOUT_MINUTES: ${{ inputs.timeout-minutes }}
        CM_POLL_SECONDS: ${{ inputs.poll-seconds }}
        CM_API_BASE: ${{ inputs.api-base }}
      run: |
        set -euo pipefail
        base="${CM_API_BASE%/}/api/v1/voice/evals"
        auth="Authorization: Bearer ${CM_API_KEY}"
        body="$(mktemp)"

        # Both values go into a URL, so they are checked, not trusted.
        if ! [[ "$CM_SUITE_ID" =~ ^[0-9a-fA-F-]{36}$ ]]; then
          echo "::error::suite-id must be a UUID."; exit 1
        fi
        if ! [[ "$CM_MIN_PASS_RATE" =~ ^(0(\.[0-9]+)?|1(\.0+)?|\.[0-9]+)$ ]]; then
          echo "::error::min-pass-rate must be a number from 0 to 1."; exit 1
        fi
        if ! [[ "$CM_TIMEOUT_MINUTES" =~ ^[0-9]+$ && "$CM_POLL_SECONDS" =~ ^[0-9]+$ ]]; then
          echo "::error::timeout-minutes and poll-seconds must be whole numbers."; exit 1
        fi

        code=$(curl -sS -o "$body" -w '%{http_code}' -X POST -H "$auth" \
          "${base}/${CM_SUITE_ID}/run?wait=false&min_pass_rate=${CM_MIN_PASS_RATE}")
        if [ "$code" != "202" ] && [ "$code" != "201" ]; then
          echo "::error::Could not start the eval run (HTTP $code): $(jq -r '.detail // .' "$body" 2>/dev/null || cat "$body")"
          exit 1
        fi
        run_id=$(jq -r '.id' "$body")
        echo "run-id=${run_id}" >> "$GITHUB_OUTPUT"
        echo "Started eval run ${run_id}"

        deadline=$(( $(date +%s) + CM_TIMEOUT_MINUTES * 60 ))
        gate=$(jq -r '.gate // "pending"' "$body")
        while [ "$gate" = "pending" ]; do
          if [ "$(date +%s)" -ge "$deadline" ]; then
            echo "::error::Eval run ${run_id} did not finish within ${CM_TIMEOUT_MINUTES} minutes."
            exit 1
          fi
          sleep "$CM_POLL_SECONDS"
          code=$(curl -sS -o "$body" -w '%{http_code}' -H "$auth" \
            "${base}/runs/${run_id}?min_pass_rate=${CM_MIN_PASS_RATE}")
          if [ "$code" != "200" ]; then
            echo "::warning::Polling run ${run_id} returned HTTP $code; retrying."
            continue
          fi
          gate=$(jq -r '.gate' "$body")
        done

        status=$(jq -r '.status' "$body")
        rate=$(jq -r '.pass_rate // "null"' "$body")
        passed=$(jq -r '.passed_cases' "$body")
        total=$(jq -r '.total_cases' "$body")
        {
          echo "status=${status}"
          echo "pass-rate=${rate}"
          echo "gate=${gate}"
        } >> "$GITHUB_OUTPUT"
        {
          echo "### CallMissed eval run \`${run_id}\`"
          echo ""
          echo "| Status | Passed | Pass rate | Required | Gate |"
          echo "| --- | --- | --- | --- | --- |"
          echo "| ${status} | ${passed}/${total} | ${rate} | ${CM_MIN_PASS_RATE} | ${gate} |"
          echo ""
          jq -r '.results[] | select(.passed == false) | "- case `\(.case_id)`: " +
            ([.assertions[]? | select(.passed == false) | "\(.type) (\(.detail // ""))"] | join("; "))
            + (if .error then " error: \(.error)" else "" end)' "$body" || true
        } >> "$GITHUB_STEP_SUMMARY"

        if [ "$gate" != "pass" ]; then
          echo "::error::Eval gate failed: ${passed}/${total} cases passed (pass rate ${rate}, required ${CM_MIN_PASS_RATE}, status ${status})."
          exit 1
        fi
        echo "Eval gate passed: ${passed}/${total} cases (pass rate ${rate})."
```

Then add a step to any workflow, with a key that has `evals:write` stored as a repository secret:

```yaml
- uses: actions/checkout@v4
- uses: ./.github/actions/callmissed-eval
  with:
    api-key: ${{ secrets.CALLMISSED_API_KEY }}
    suite-id: aa10…
    min-pass-rate: "0.9"
```

| Input | Default | Meaning |
| --- | --- | --- |
| `api-key` | — | A `cm_` key with `evals:write` |
| `suite-id` | — | The suite to run |
| `min-pass-rate` | `1` | `0..1` |
| `timeout-minutes` | `30` | Give up waiting after this long, failing the job |
| `poll-seconds` | `15` | Wait between polls |
| `api-base` | `https://api.callmissed.com` | API host |

The job summary lists every failed case with the assertions it missed. Outputs: `run-id`, `status`, `pass-rate`, `gate`. Every run is charged like any other, so run the gate on the events that need it — a pull request that touches the agent's prompt, not every push.

## Billing

Only `POST /{suite_id}/run` charges. The cost is the agent model's usage across every case, plus the scoring model for a scorecard grade and for `llm_judge` criteria (one call per case that has any), charged after the run completes and visible in [usage logs](/docs/usage-api) as `service: "llm"`.

Cost scales with `cases × (max_turns × 2 + 1)` model calls, so trimming `max_turns` is the cheapest lever. The model calls have already happened by the time the charge is made, so a run that finishes is always charged in full, even when that takes the balance below zero; the next run is then refused at the credit check.

## Errors

| Status | When |
| --- | --- |
| `402` | Credit balance exhausted at the pre-run gate |
| `403` | Key is missing `evals:read` / `evals:write`, or (for a case from a call) the `stt`/`tts`/`llm` permissions |
| `404` | Suite, case, run, agent or scorecard not in your tenant |
| `409` | Duplicate suite name, or an empty suite |
| `422` | Blank name/persona/opening, over 20 criteria, over 5 `llm_judge` criteria or one over 500 characters, an unknown criterion type, over 50 cases in a run, a `min_pass_rate` outside `0..1`, or a call with no caller speech |
| `429` | Three runs are already executing on the account |
