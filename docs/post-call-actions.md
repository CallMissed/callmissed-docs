---
title: "Post-Call Actions"
description: "Decide what a voice agent does on its own after each call: call the caller back, update or tag their contact, create a task, ticket or deal, email or WhatsApp your team, send a WhatsApp template or an email to the caller, grade the call, or send it to a webhook, a Google Sheet, a CSV file or HubSpot, based on what happened on the call."
slug: "post-call-actions"
breadcrumb: "Voice Agents"
---

# Post-Call Actions

Decide what a voice agent does on its own after each call: call the caller back, update or tag their contact, create a task, ticket or deal, email or WhatsApp your team, send a WhatsApp template or an email to the caller, grade the call, or send it to a webhook, a Google Sheet, a CSV file or HubSpot, based on what happened on the call.

## Overview

After a call ends, [call analysis](/docs/call-analysis) writes a summary, the caller's sentiment, the outcome and the answers your agent's `analysis_variables` ask for (for example `star_rating`, `callback_requested`). **Post-call actions** act on those answers automatically.

Each agent has an ordered list of **rules**. A rule is a set of **conditions** and one or more **actions**. By default every condition must pass (`match: "all"`); set `match: "any"` to run the rule when at least one condition passes. A rule with no conditions runs after every analysed call, whichever `match` it has.

| Example | Condition | Actions |
| --- | --- | --- |
| Low rating follow-up | `star_rating` is at most 2 | Create a task for the manager, email the owner |
| Callback requested | `callback_requested` is yes | Create a task to call back within 4 hours |
| Ask for a review | `star_rating` is at least 4 | Send your approved WhatsApp template with your review link |
| Keep a log | none | Add a row to a Google Sheet |
| Keep a CSV | none | Add a row with only the columns you choose to a CSV file you download |
| Remember answers | none | Save the rating and comment on the caller's contact |
| Call back at the time asked | `callback_requested` is yes | Call the caller back at the time they gave |
| Voicemail retry | Answered by is voicemail | Call again the next day at 11:00 |
| Hot lead | `interest_level` is at least 4 | Create or move a deal, WhatsApp the sales lead |
| Send each call to your system | none | Call your own API with the call's details |
| AI follow-up | none | The AI writes an email from what was said, then it is sent to the caller |

Actions run in the background shortly after the analysis finishes, or after their own delay (`delay_minutes`). Each action runs at most once per call, its status is recorded per call, and temporary failures are retried with backoff.

A rule with a condition on **Answered by** (`call.answered_by`) also runs on outbound calls that nobody, or only voicemail, answered. Those calls have no analysis, so their analysis answers are empty. Other rules run only after analysed calls.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

Reading rules, options, recipes, test results and run history needs the `bots:read` scope. Creating, changing or deleting rules, and the on/off switches, need `bots:write`. Downloading a CSV file also needs `telephony:read`, because its rows hold what callers said.

All paths below are under `https://api.callmissed.com/api/v1/bots/{bot_id}/post-call`.

## Recipes

```bash
curl https://api.callmissed.com/api/v1/bots/{bot_id}/post-call/recipes \
  -H "Authorization: Bearer cm_your_api_key"
```

Ready-made rules, already filled in from this agent's own analysis variables. Each recipe has a `key`, a plain-language `title` and `description`, `available` (with `unavailable_reason` when not), `needs` (the blanks you still choose) and the `rule` it would save.

| Recipe `key` | What it does | `needs` |
| --- | --- | --- |
| `low_rating_followup` | Low rating: create a follow-up task and email you | `task_assignee`, `notify_user` |
| `callback_task` | Caller asked for a callback: create a callback task | `task_assignee` |
| `happy_review_whatsapp` | High rating: send an approved WhatsApp template | `whatsapp_template` |
| `save_to_sheet` | Add a row for every call to a Google Sheet | `sheet` |
| `save_on_contact` | Save answers on the caller's contact | `contact_fields` |
| `issue_ticket` | Caller reported a problem: open a high-priority ticket | none |
| `callback_at_time` | Caller asked for a callback: call them back at the time they said | none |
| `not_interested_stop` | Not interested: add to the do-not-call list (calls) and tag the contact | none |
| `hot_lead_deal` | Strong interest: create or move a deal, WhatsApp the sales lead | `pipeline`, `whatsapp_team_member`, `whatsapp_template` |
| `complaint_ticket_alert` | Complaint or upset caller: open a ticket and email a manager | `task_assignee`, `notify_user` |
| `voicemail_retry` | Reached voicemail: call again the next day at 11:00 | none |
| `no_answer_campaign` | Not answered: add the number to a calling campaign | `campaign` |
| `booked_whatsapp` | Booked: send a WhatsApp confirmation template | `whatsapp_template` |
| `quote_email_task` | Asked for a quote: email the caller and create a task | `from_address`, `task_assignee` |
| `payment_promise_task` | Promised to pay: a task due on the promised date | `task_assignee` |
| `scorecard_alert_qa` | Grade every call, email QA below 60 | `scorecard`, `notify_user` |
| `transfer_note` | Transferred to a person: add a note to the contact | none |
| `webhook_every_call` | Send every call to a webhook | `webhook_url` |
| `hubspot_note` | Add every call's summary as a note in HubSpot | none |
| `csv_every_call` | Log every call to a CSV: date, time, number, contact, outcome, every answer and the summary | none |
| `api_every_call` | Send every call to your own API as JSON, with the sign-in you choose | `api_url` |
| `ai_followup_email` | The AI writes a follow-up email from the call and it is sent to the caller (`text` is `{{ai.result}}`) | `from_address` |
| `unhappy_any_alert` | Negative mood or a low rating (either one, `match: "any"`): email you the summary | `notify_user` |

Each recipe also has a `group` (`Follow up`, `Sales`, `Support`, `Compliance` or `Data`).

Save one with its blanks filled:

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/{bot_id}/post-call/rules/from-recipe \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "recipe": "low_rating_followup",
        "fills": { "assignee_user_id": "6f1c…", "notify_user_id": "6f1c…" } }'
```

| `fills` field | Used by |
| --- | --- |
| `assignee_user_id` | The team member who gets the task or ticket |
| `notify_user_id` | The team member who gets the email. Omit for the workspace owner |
| `template_id`, `template_variables` | The approved WhatsApp template, and the text for each of its inputs |
| `add_consent_line` | `true` adds the recipe's `consent.prompt_line` (the WhatsApp permission question) to the agent's prompt, unless it is already there. Default `false` |
| `on_no` | What a caller's "no" to WhatsApp messages on the call does: `"skip_message"` (default) only skips this message; `"pause_marketing"` also stops all WhatsApp marketing to them |
| `integration_id`, `spreadsheet_id`, `sheet` | The Google Sheets connection, spreadsheet and tab |
| `contact_field_ids` | `{answer name: contact field id}`. Answers left out get a new contact field |
| `voice_bot_id`, `from_number_id` | The voice agent that makes a callback (default: this agent) and the number it calls from (default: the agent's own) |
| `pipeline_id`, `stage_id` | The deal pipeline and stage |
| `campaign_id` | The calling campaign |
| `scorecard_id` | The scorecard to grade with |
| `whatsapp_user_id` | The team member to WhatsApp (needs a verified WhatsApp number on their profile) |
| `webhook_url` | The webhook address |
| `api_url` | The address of your API, for `api_every_call` (http or https, public) |
| `from_address` | The verified sending address for emails to the caller |
| `name` | Optional new rule name |

## Building blocks

```bash
curl https://api.callmissed.com/api/v1/bots/{bot_id}/post-call/options \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns everything a rule can use for this agent: `variables` (its analysis variables with their types), `call_fields` (with `option_labels` for choices, and `operators` where a field takes fewer than its type), `operators` per type, `tokens` for text, `team` (with `whatsapp_ready`), `contact_fields`, sendable `whatsapp_templates` with their `inputs`, shared Google `sheets`, `voice_agents`, `phone_numbers`, `pipelines` with their `stages`, `campaigns` (with `accepts_numbers`), `scorecards`, `hubspot_connected`, `tools_used` (tool names seen on this agent's recent calls), `recipe_groups`, `custom_tools` (the agent's and workspace-wide custom tools a `run_custom_tool` action can run, each with its `id`, `name`, `description`, `enabled` and `inputs`), `ai_models` (the model ids this workspace and agent may pick for an AI step, as `{id, name}`), `default_ai_models` (the model used for `ask_ai`, `score_call` and the draft when you pick none; currently `gpt-6-luna`) and `limits` (`rules`, `conditions`, `actions`, `http_headers`, `http_query`, `ai_output`). Use these ids; a rule that names anything else is refused.

## Rules

| Method | Path | What it does |
| --- | --- | --- |
| `GET` | `/rules` | The agent's rules, in order |
| `POST` | `/rules` | Add a rule |
| `PATCH` | `/rules/{rule_id}` | Change a rule; only the fields sent change. `{"enabled": false}` pauses it |
| `DELETE` | `/rules/{rule_id}` | Delete a rule. What it already did stays in the history |
| `POST` | `/rules/{rule_id}/run` | Run a saved rule on one past call now (see below) |
| `GET` | `/rules/{rule_id}/actions/{action_id}/secret` | The signing secret of a `send_webhook` action (`bots:write`) |
| `GET` | `/rules/{rule_id}/actions/{action_id}/csv` | Download the CSV file of an `append_csv_row` action (see [CSV files](#csv-files)) |
| `GET` | `/csv-files` | Each CSV action of the agent with rows: `rule_id`, `action_id`, `row_count`, `last_row_at` |
| `GET` / `PUT` | `/settings` | `{"enabled": true}`: the agent-wide switch for all its rules |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/{bot_id}/post-call/rules \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Low rating follow-up",
    "conditions": [ { "field": "star_rating", "op": "lte", "value": 2 } ],
    "actions": [
      { "type": "create_task",
        "config": { "title": "Low rating ({{star_rating}}) from {{caller_number}}",
                    "description": "{{summary}}", "due_in_hours": 24 } },
      { "type": "notify",
        "config": { "subject": "Low rating from {{caller_number}}", "message": "{{feedback_comment}}" } }
    ]
  }'
```

A rule is checked when it is saved: every condition must name one of the agent's analysis variables (or a call field) with an operator that suits its type, every `{{token}}` must exist, and every team member, contact field, template and spreadsheet must belong to your workspace. A problem returns `422` with a message saying what to fix.

| Limit | Value |
| --- | --- |
| Rules per agent | 100 |
| Conditions per rule | 25 |
| Actions per rule | 1 to 20 |
| `ask_ai` actions per rule | 3 |
| `http_request` and `run_custom_tool` actions per rule, together | 5 |
| `ask_ai` runs per call, across all the agent's rules | 10 |
| `http_request` and `run_custom_tool` runs per call, across all the agent's rules | 25 |

A call that matches more `ask_ai`, `http_request` or `run_custom_tool` actions than the per-call limits allows runs them in rule order up to the limit; the rest are recorded as `skipped` with the reason.

### Conditions

`match` on the rule is `"all"` (default: every condition must pass) or `"any"` (one passing condition is enough). Rules, rule updates, drafts and dry-run results all carry `match`.

`{ "field": "...", "op": "...", "value": ... }`. `field` is one of the agent's analysis variable names, or a call field:

| Call field | Type | Values |
| --- | --- | --- |
| `call.sentiment` | `enum` | `positive`, `neutral`, `negative`, `unknown` |
| `call.disposition` | `string` | The call outcome from the analysis |
| `call.goal_met` | `boolean` | |
| `call.direction` | `enum` | `inbound`, `outbound`, `web` |
| `call.duration_seconds` | `number` | |
| `call.answered_by` | `enum` | `human`, `voicemail`, `no_answer`, `unknown` |
| `call.end_reason` | `enum` | `participant_disconnect`, `agent_closed`, `no_input`, `no_answer`, `transferred`, `handoff`, `opted_out`, `max_duration`, `budget_exhausted`, `agent_error`, `timeout` |
| `call.tools_used` | list | The names of the tools the agent used on the call |
| `call.transferred` | `boolean` | The call was transferred or handed to a person |
| `call.transcript` | `string` | What the caller said (`contains`, `is_empty`, `is_not_empty` only) |
| `call.channel` | `enum` | `pstn` (phone), `whatsapp_call`, `web` |

| Field type | Operators |
| --- | --- |
| `number` | `eq`, `ne`, `lt`, `lte`, `gt`, `gte`, `between` (`value: [low, high]`), `is_empty`, `is_not_empty` |
| `boolean` | `is_true`, `is_false`, `is_empty` |
| `enum` | `is`, `in` (`value` is a list of options), `is_empty`, `is_not_empty` |
| `string` | `contains`, `is`, `is_empty`, `is_not_empty` |
| list (`call.tools_used`) | `contains` (one name), `in` (a list: any of them), `is_empty`, `is_not_empty` |

An unanswered variable never matches a comparison, only `is_empty`. If a variable is later removed from the agent, conditions on it stop matching.

### Actions

`{ "type": "...", "config": { ... }, "delay_minutes": 0 }`. `delay_minutes` (optional, 0-10080) waits that long after the call before the action runs; while it waits, its run is `pending` with `run_after` set. A waiting action runs as the rule has it when it is due: an edit made meanwhile applies (a later time or a longer delay is waited for), an action removed from the rule is `skipped`, and so is one whose rule's conditions were changed and no longer match the call (unless it was started with `force`). The run records the action as it ran. Text fields take `{{tokens}}`: any analysis variable, plus `summary`, `sentiment`, `disposition`, `goal_met`, `caller_number`, `contact_name`, `agent_name`, `call_date`, `call_time`, `call_duration`, `call_direction`, `call_link` (the call's page in the console, for phone, WhatsApp and web calls), `answered_by`, `end_reason`, `tools_used` and `call_channel`. In actions that come after an `ask_ai` action in the same rule, `{{ai.result}}` (or `{{ai.<output_name>}}`) is the AI's answer; see [Ask the AI](#ask-the-ai).

| `type` | `config` | Notes |
| --- | --- | --- |
| `set_contact_fields` | `fields: [{field_def_id or create_field: {label, field_type}, value}]`, `create_contact` (default `true`) | Writes [custom fields](/docs/crm-custom-fields) on the contact with the caller's number, creating the contact if needed. Only when the number is verified: outbound calls and WhatsApp calls always are; inbound phone calls only when the agent trusts caller ID |
| `create_task` | `title`, `description`, `assignee_user_id`, `due_in_hours` (0-720) | Linked to the caller's contact when there is one |
| `create_ticket` | `subject`, `description`, `priority` (`low`, `normal`, `high`, `urgent`), `assignee_user_id` | Your ticket routing rules apply |
| `notify` | `user_id` (omit for the workspace owner), `subject`, `message` | An email to an active member of your workspace |
| `send_whatsapp_template` | `template_id`, `variables: {input key: text}`, optional `consent_field` and `on_no` | An approved template without a media header, from a connected number on its WhatsApp account |
| `append_sheet_row` | `integration_id`, `spreadsheet_id`, `sheet`, `columns: [{header, value}]` (1-30) | Missing header cells are added to the tab first |
| `schedule_callback` | `voice_bot_id`, `from_number_id` (optional), `at_field` (optional text answer holding the time asked for), `at_time` (`HH:MM`) and `days_after` (0-7), `requested_field` (optional yes/no answer recording that the caller asked for a callback), `hours_start` (7-21) / `hours_end` (8-22), default 10 and 21, `reason` | Places one outbound call with that voice agent: at the time the caller gave, else at `at_time`, else after `delay_minutes`. When the caller gave a time that can't be read, or is more than a week after the call, the run is `skipped` with that reason and nobody is called. Calling hours are the callee's local time when their number is tied to one place, else the agent's (toll-free and other non-geographic numbers, and numbers of countries that span several time zones); calls to Indian numbers also stay inside 10:00 to 21:00 IST. Outside the hours the run waits. Your do-not-call list and credit balance apply. A dial that may have rung is never retried. The agent is told why it is calling: that the person asked to be called back only when `requested_field` was yes on the call; a voicemail or no-answer retry, or a follow-up, is described as such. A call this action placed is never called back again. One per rule. Verified numbers only |
| `send_webhook` | `url`, `include_transcript` (default `false`) | A `POST` of signed JSON: `event` (`post_call.action`), `delivery_id`, `call`, `analysis` and, when asked, `transcript`. Headers `X-CallMissed-Event`, `X-CallMissed-Delivery` (the same on a retry) and `X-CallMissed-Signature: sha256=<HMAC of the body>` with the action's secret, made when the rule is saved and read from the secret endpoint. The address must be public. 2xx is delivered; other 4xx fail; 5xx, 408, 429 and no answer are retried |
| `upsert_deal` | `pipeline_id`, `stage_id`, `title`, `value_field` (optional number answer), `owner_user_id` | Moves the contact's open deal in that pipeline to the stage, or creates one. Linked to the contact only when the number is verified |
| `add_to_campaign` | `campaign_id` | Adds the number to a calling campaign that is a draft, scheduled or paused. A number already on it, or on your do-not-call list, is not added. Verified numbers only |
| `add_to_dnc` | `channel` (`all` = calls and messages, the default; `voice` = calls), `reason` | Adds the number to your do-not-call list. Any number, verified or not: the list only ever means less contact |
| `tag_contact` | `tags` (1-10), `create_contact` (default `true`) | Adds [tags](/docs/crm-contacts) to the caller's contact. Verified numbers only |
| `add_note` | `body`, `create_contact` (default `true`) | A note on the caller's contact. Verified numbers only |
| `send_email_to_caller` | `from_address` (a sending address on one of your verified [email domains](/docs/email-domains)), `subject`, `text`, `reply_to` | To the email on the caller's contact. Skipped when the contact has no email. Sent with an `Idempotency-Key` per run, so a retry never sends it twice |
| `notify_whatsapp` | `user_id`, `template_id`, `variables` | An approved template from your WhatsApp number to a team member's verified WhatsApp number |
| `score_call` | `scorecard_id`, `alert_below` (1-100, optional), `alert_user_id`, `model` (optional, see [Choosing the AI model](#choosing-the-ai-model)) | Grades the call's transcript with a [scorecard](/docs/call-analysis#scorecards); billed per token like the [Chat Completions API](/docs/chat-completion). Below `alert_below`, emails `alert_user_id` (default: the owner). The result has `total` and `breakdown` |
| `http_request` | `url`, `method`, `auth`, `headers`, `query`, `body_type`, `body` or `form` | Calls your own API; see [Call an API](#call-an-api) |
| `run_custom_tool` | `tool_id`, `arguments: {input name: template}` | Runs one of the agent's [custom tools](/docs/agent-tools); see [Run a custom tool](#run-a-custom-tool) |
| `ask_ai` | `instruction`, `output_name`, `model` (optional) | Asks a language model about the call; see [Ask the AI](#ask-the-ai) |
| `crm_sync` | `provider: "hubspot"`, `text` | Adds a note in HubSpot, on the HubSpot contact with the caller's email when the number is verified |
| `append_csv_row` | `columns: [{header, value}]` (1-30), `file_name` (optional; letters, digits, dots, dashes and underscores, up to 60) | Adds one row per call to a CSV file CallMissed keeps for this action; nothing is sent anywhere. Pick only the columns you need. Download it from `/rules/{rule_id}/actions/{action_id}/csv`. Skipped for a call with zero data retention, and for a call whose details your retention settings already removed |

### Call an API

`http_request` sends one HTTP request to an address you choose after each matching call.

| `config` field | What it holds |
| --- | --- |
| `url` | A public `http` or `https` address, without a user name or password in it (use `auth`). Redirects are not followed |
| `method` | `GET`, `POST`, `PUT`, `PATCH` or `DELETE` (default `POST`) |
| `auth` | `{ "type": "none" }`, `{ "type": "bearer", "token": "…" }`, `{ "type": "basic", "username": "…", "password": "…" }` or `{ "type": "api_key", "header": "X-API-Key", "value": "…" }` (`header` defaults to `X-API-Key`) |
| `headers` | Up to 20 of `{ "name", "value", "secret": false }`. Set `secret: true` to keep a value write-only. Typed text is plain ASCII with no control characters (tab included); a `{{token}}`'s text that is not ASCII, such as a name in Hindi, is percent-encoded when sent |
| `query` | Up to 20 of `{ "name", "value" }`; values take `{{tokens}}`. Query values are not secret: they are shown on every read and in the run history, so put keys in `auth` or a secret header |
| `body_type` | `none`, `json`, `form` or `text`. Default `json` for `POST`, `PUT` and `PATCH`, `none` for `GET` and `DELETE`; `GET` must be `none` |
| `body` | A template of up to 4000 characters for `json` and `text`; `json` must be valid JSON |
| `form` | For `form`: 1 to 30 of `{ "name", "value" }` |

These headers are refused: `Host`, `Content-Length`, `Transfer-Encoding`, `Connection`, `Keep-Alive`, `TE`, `Trailer`, `Upgrade`, `Expect`, anything starting with `Proxy-` or `X-CallMissed-`, and `Authorization` when `auth` is not `none` (use `auth` instead).

In a `json` body, each `{{token}}` is JSON-escaped when it is filled in, so put text answers inside quotes: `"summary": "{{summary}}"`. A request whose body is not valid JSON for a particular call is skipped with the reason, and nothing is sent.

The request waits up to 10 seconds. Every request carries an `X-CallMissed-Delivery` header holding the run id, the same on every retry, so your server can drop a repeat. A `2xx` answer succeeds, and the run keeps the first 500 characters of the response (`status_code`, `response_preview`, `truncated`). A `5xx`, `408`, `429` or no answer is retried with backoff. Other `4xx` answers and any `3xx` redirect fail the run.

Credentials are write-only. The bearer token, the basic password, the API key value and any header value with `secret: true` are returned as `••••` by every read. Send `••••` (or an empty string) back for the same action id to keep the saved value; a new action needs the real one. A saved credential is only kept while the address stays on the same server: changing the scheme, host or port returns `422` until you enter it again. A sign-in or a secret header needs an `https://` address.

A failed request keeps the status code and the start of the answer on the run's result (`status_code`, `response_preview`), not in its error. `response_preview` reads as hidden without `telephony:read`, because an answer can repeat what was sent about the call, and it is removed with the call's other content under your data retention settings.

```json
{
  "name": "Send to my API",
  "conditions": [],
  "actions": [
    { "type": "http_request",
      "config": {
        "url": "https://example.com/hooks/calls",
        "method": "POST",
        "auth": { "type": "bearer", "token": "YOUR_TOKEN" },
        "headers": [],
        "query": [],
        "body_type": "json",
        "body": "{\"caller\": \"{{caller_number}}\", \"outcome\": \"{{disposition}}\", \"summary\": \"{{summary}}\"}"
      } }
  ]
}
```

### Run a custom tool

`run_custom_tool` runs one of the agent's own [custom tools](/docs/agent-tools), or a workspace-wide one, as listed in `custom_tools` in the options. `tool_id` picks it and `arguments` maps each of its inputs to a template that takes `{{tokens}}`; every required input must be filled. The caller's number, name and email are passed to the tool as context only when the number is verified. The tool is never retried, because it may already have reached the service. The run result holds `tool_name`, `status_code`, `response_preview` and `truncated`; `response_preview` is hidden and removed as for [Call an API](#call-an-api).

### Ask the AI

`ask_ai` asks a language model a question about the call and keeps its answer for the actions that follow.

| `config` field | What it holds |
| --- | --- |
| `instruction` | What to ask, up to 4000 characters. Takes `{{tokens}}`. The model also sees the call's summary, answers and transcript. A token is not filled into the instruction itself: the model sees `[name]` there and the value with the call's content, so a caller's words can't act as your instruction |
| `output_name` | Lowercase letters, digits and `_`, starting with a letter, up to 32 characters (default `result`). Unique within the rule |
| `model` | Optional model id, see [Choosing the AI model](#choosing-the-ai-model) |

The answer (up to 4000 characters) is available as `{{ai.result}}`, or `{{ai.<output_name>}}`, in actions placed **after** the `ask_ai` action in the same rule. Using it without an earlier step that names it returns `422` when the rule is saved. At run time those later actions wait for the AI step, and each is `skipped` with the reason if the AI step failed. The step is also skipped on a call with zero data retention, because the call's content is not sent to a model. The run result holds `output`, `output_name`, `model`, `input_tokens`, `output_tokens` and `credits`; `output` reads as hidden without `telephony:read` and is removed with the call's other content under your data retention settings.

Treat `{{ai.*}}` as untrusted text. The answer is written from what the caller said, so a caller can steer it: check it before it reaches a place where its words act, such as a URL, a tag or a field a later system trusts.

```json
{
  "name": "AI follow-up email",
  "match": "any",
  "conditions": [
    { "field": "call.sentiment", "op": "is", "value": "negative" },
    { "field": "star_rating", "op": "lte", "value": 3 }
  ],
  "actions": [
    { "type": "ask_ai",
      "config": {
        "instruction": "Write a short, kind follow-up email that apologises and says the next step.",
        "output_name": "result"
      } },
    { "type": "send_email_to_caller",
      "config": {
        "from_address": "support@yourdomain.com",
        "subject": "About your call",
        "text": "{{ai.result}}"
      } }
  ]
}
```

### Choosing the AI model

`ask_ai`, `score_call` and the [draft endpoint](#describe-it) each take an optional `model`. `GET options` lists the ids you may use in `ai_models` and the one used when you pick none in `default_ai_models`. Only public model ids are accepted. Some models need a paid plan, an agent that keeps call data in India needs a model processed in India, and an API key limited to some models can only pick those. An id that is unknown, under maintenance or not allowed is refused with `422` and a plain message. With no pick, the default is `gpt-6-luna`; when that is not allowed for your plan, agent or key, the first allowed of `kimi-k2.6` and `sarvam-105b` is used instead (on the free plan, `kimi-k2.6`, or `sarvam-105b` for an agent that keeps call data in India), and `default_ai_models` shows which. Prices per million tokens are on the public `GET /api/v1/models?category=llm` listing.

**Billing.** These steps are billed per token at the chosen model's price, the same as [`/v1/chat/completions`](/docs/chat-completion), and each call adds a usage-log row. The balance and your budget cap are checked before the model is called, and the estimated cost is held from your balance until the step finishes, so steps running at the same time can't spend the same credits. The hold is then settled to the real tokens. A call that fails is not billed and its hold is returned. Dry runs never call the AI.

**WhatsApp consent.** A message is never sent to a number that opted out of your WhatsApp messages or is on your do-not-contact list. A **marketing** template is sent only to contacts who agreed to receive WhatsApp messages. The action is recorded as skipped, with the reason, when either applies. WhatsApp messages are billed as usual.

**Agreement on the call.** Set `consent_field` on the action to an analysis variable with the answers `yes`, `no` and `not_asked` (the `happy_review_whatsapp` recipe uses `whatsapp_ok`; saving the rule adds that variable to the agent when it is missing, and upgrades an older yes/no `whatsapp_ok` to the three answers). A yes/no variable of your own is also accepted: `true` counts as yes, `false` as not asked. The message is then sent when the caller clearly said **yes** on this call, or when the answer is **not_asked** and the contact had already agreed to WhatsApp messages. A yes is saved on the caller's contact first: `whatsapp_opt_in: true`, `consent_source: "voice call <session id>"` and `consent_updated_at`; the number must be verified, as for `set_contact_fields`. A **no** is a request to stop: this message is not sent, even to a contact who had agreed before. What else a no does is set per action with `on_no`. With `"skip_message"` (the default, and how a rule saved without `on_no` behaves) nothing is written and the run is `skipped` with `Not sent: the caller said no to WhatsApp messages on this call`. With `"pause_marketing"` the no also stops all WhatsApp marketing to that number (utility messages such as order and appointment updates still go through): when the number is verified, the no is recorded: the contact's `whatsapp_opt_in` is set to `false` with `consent_source: "voice call <session id> declined"`, the number is added to your do-not-contact list for WhatsApp marketing (`channel: whatsapp_marketing`, so campaigns and agent tools skip marketing templates to it as well), and the run is `skipped` with `Not sent: the caller said no to WhatsApp messages on this call; their WhatsApp marketing is now paused`; from an unverified caller ID nothing is written and the run is `skipped` without the suffix. Not asked and not already agreed is `skipped` with `Not sent: this caller hasn't agreed to WhatsApp messages`. An opt-out always wins over a yes.

The agent has to ask. The recipe's `consent` object in `GET recipes` carries `prompt_line` (a question that names your business and WhatsApp and says a yes covers messages such as a review link), `prompt_has_line` (whether that line is already in the agent's prompt), `prompt_may_ask` (a guess that the prompt asks permission to message the caller on WhatsApp in its own words) and `field_exists`. Pass `add_consent_line: true` when saving to add the question to the prompt; it goes into the prompt the agent speaks on calls and is added unless already present.

## Describe it

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/{bot_id}/post-call/rules/draft \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "description": "When a caller is unhappy, open a high-priority ticket and WhatsApp the manager", "session_id": "…" }'
```

Write what should happen after a call in plain words (`description`, 10-2000 characters) and get a draft rule back. It needs `bots:write` and **costs credits**: it runs a language model once. The balance is checked first, so an empty balance returns `402` and costs nothing. Every answer the model gives is billed once, per token like the Chat Completions API, even one that could not be turned into a rule. `model` (optional, see [Choosing the AI model](#choosing-the-ai-model)) picks the model, and the response's `model` is the one that drafted. `session_id` (optional) is a past call of this agent to start from: its outcome, mood and answers are used, not its number or transcript. A call of another agent returns `404`, a call without a finished analysis `409`.

**Nothing is saved or run.** The draft uses only this agent's answers, the call fields, and the team members, WhatsApp templates, spreadsheets, agents, numbers, pipelines, campaigns and scorecards your workspace has. Each part goes through the same checks as saving a rule. The response:

| Field | What it holds |
| --- | --- |
| `rule` | A rule in the shape `POST /rules` takes, always with `enabled: false` |
| `questions` | What you still have to choose, such as which template or which team member |
| `notes` | What the draft assumed, for example that "unhappy" means the caller's mood is negative |
| `errors` | Parts that would not save, in plain words. A condition that does not fit is left out; an action missing a choice is kept with that choice empty |
| `valid` | `true` when `rule` can be saved as it is |

Anything that is not in your workspace is removed from the draft. Answer the questions, fix what `errors` lists, test it with `POST /rules/preview`, then save it with `POST /rules`. The rule is saved switched off unless you set `enabled`.

## Test a rule

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/{bot_id}/post-call/dry-run \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "session_id": "…" }'
```

Runs the agent's saved rules (or one `rule_id`, or an unsaved `rule`) against a past call of this agent. `POST /rules/preview` with `{ "session_id": "…", "rule": {…} }` does the same for one unsaved rule. For each rule: `matched`, each condition with the call's `actual` value and `passed`, and for each action a plain-language `would` and a `blocked_reason` when it would be held back. Nothing is sent, saved or created. What the caller said (`call.transcript`'s `actual`, and a webhook preview's `transcript`) is shown only to a key with `telephony:read` (or the key that made the call) and to console users; otherwise it reads `(hidden: …)`. `GET /recent-calls?limit=20` (1-50) lists the agent's latest analysed calls to pick from. A call without a finished analysis returns `409`.

## Run a rule on one call

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/{bot_id}/post-call/rules/{rule_id}/run \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "session_id": "…", "force": false }'
```

Queues the rule's actions for one past call of this agent (`bots:write`). Its conditions must pass unless `force` is `true`. Returns `matched`, `queued` and `runs`: this rule's runs for the call. An action this rule already ran or queued for the call is returned, not run again. A switched-off rule or agent returns `409`.

## CSV files

```bash
curl -o calls.csv "https://api.callmissed.com/api/v1/bots/{bot_id}/post-call/rules/{rule_id}/actions/{action_id}/csv?from=2026-11-01T00:00:00Z" \
  -H "Authorization: Bearer cm_your_api_key"
```

The rows an `append_csv_row` action added, one per call, oldest first, with the action's columns as they are now: a column you add later is blank on older rows, and a column you remove is left out. `from` (inclusive) and `to` (exclusive, default now) are ISO 8601 and may be at most 366 days apart; without `from`, the last 366 days. The file is UTF-8 with a byte-order mark, so Excel shows every script correctly, and a cell that starts with `=`, `+`, `-`, `@`, a tab or a carriage return is prefixed with `'` so a spreadsheet never runs it as a formula. A plain number made only of digits, spaces and `+ - ( ) .`, such as a phone number like `+919820012345`, cannot run a function and is written as is. It is named from the action's `file_name` and the date, for example `call-log-20261114.csv`. Needs `bots:read` and `telephony:read`.

Each call adds at most one row per action, even when a rule is run on the call again. Rows are deleted with their call, with the rule, and when your data retention settings remove a call's details.

## Run history

```bash
curl "https://api.callmissed.com/api/v1/bots/{bot_id}/post-call/runs?session_id=…" \
  -H "Authorization: Bearer cm_your_api_key"
```

One entry per action per call, newest first (`limit` 1-100, default 50; `offset` 0-10000): `rule_name`, `action_type`, `action_label`, `status` (`pending`, `running`, `succeeded`, `failed`, `skipped`), `run_after` (when a waiting action runs), `attempts`, `error` (why it failed or was skipped) and `result` (for example `task_id`, `ticket_id`, `contact_id`, `send_id`, `deal_id`, `telephony_call_id`, `total`, or a `note` when a succeeded action had nothing to write, such as "No answers to save on this call").

For one call, `GET https://api.callmissed.com/api/v1/voice/sessions/{session_id}/post-call/runs` (`bots:read`) lists every rule's runs, oldest first. A `score_call` result's `evidence_quote` lines quote the call, so they are left out without `telephony:read`.

## From an AI assistant

The [Account MCP server](/docs/agent-tools-mcp) can set all of this up: `list_post_call_recipes`, `get_post_call_options`, `create_post_call_rule_from_recipe`, `create_post_call_rule`, `update_post_call_rule`, `toggle_post_call_rule`, `set_post_call_actions_enabled`, `delete_post_call_rule`, `list_post_call_recent_calls`, `dry_run_post_call_rule`, `draft_post_call_rule` (a draft rule from a plain-words description; costs credits, saves nothing), `run_post_call_rule_on_call` (run a rule on one call for real), `list_post_call_action_runs` and `get_post_call_csv` (read a CSV file as text; also needs `telephony:read`). The `post_call_actions` playbook (`get_playbook`) walks an assistant through interviewing you first.
