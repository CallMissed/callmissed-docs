---
title: "Prompt Management"
description: "Store prompts server-side, version them, label a release, pin model parameters as presets, and render a template without calling a model."
slug: "gateway-prompts"
breadcrumb: "API Reference"
---

# Prompt Management

Store prompts server-side, version them, label a release, pin model parameters as presets, and render a template without calling a model.

## Overview

Prompt management moves your system prompts out of your deployment and into the gateway, so you can change wording without shipping code.

- A **prompt** is a named container.
- A **version** is an immutable snapshot of a template, its declared variables, and optionally a model and parameter set. Versions are append-only — you never edit one, you add the next.
- A **label** (`production`, `staging`, …) points at exactly one version. Moving the label is the release.
- A **preset** pins a model plus a parameter block, optionally bound to a prompt version, so several call sites share one configuration.

Rendering happens server-side with `POST /{prompt_id}/render`. It substitutes variables and returns the text — it never calls a model and never costs credits. Feed the result into [`POST /v1/chat/completions`](/docs/chat-completion) yourself.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

| Operation | Scope |
| --- | --- |
| Every `GET`, plus `POST /{prompt_id}/render` | `prompts:read` |
| Every `POST`, `PATCH`, `DELETE` that changes stored data | `prompts:write` |

Render is a read: it writes nothing, so it only needs `prompts:read`.

## Limits

| Thing | Limit |
| --- | --- |
| Prompt name | 255 characters |
| Description | 500 characters |
| Template body | 65,536 characters |
| Declared variables | 100 per version |
| Variables sent to render | 100 keys, each string value at most 8,000 characters |
| Label | 64 characters |
| Pinned `params` | 32,000 characters when serialised |
| List pagination | `limit` `1..200` (default `50`), `offset` `0..100000` |

## Prompts

### GET `/api/v1/gateway/prompts`

Newest first. Accepts `limit` and `offset`.

```json
[
  {
    "id": "7c1e…",
    "tenant_id": "a0b1…",
    "name": "support-greeting",
    "description": "Opening turn for the WhatsApp support agent",
    "current_version": 4,
    "created_at": "2026-08-01T09:00:00Z",
    "updated_at": "2026-08-16T11:20:00Z"
  }
]
```

`current_version` is the version number the most recent `POST /versions` created — it is what `render` uses when you pass neither `version` nor `label`.

### POST `/api/v1/gateway/prompts`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, not blank. Unique per tenant |
| `description` | `string` | No | At most 500 characters |

Returns `201` with the prompt. A duplicate name returns `409 A prompt named '…' already exists`.

### GET / PATCH / DELETE `/api/v1/gateway/prompts/{prompt_id}`

`PATCH` accepts `name` and `description` only — `current_version` is server-managed. `DELETE` returns `204` and cascades the prompt's versions; presets that pointed at one of those versions keep their pinned model and params, with `prompt_version_id` set to `null`.

`404 Prompt not found` for an unknown or another tenant's id.

## Versions

### GET `/api/v1/gateway/prompts/{prompt_id}/versions`

Highest version first. Accepts `limit` (`1..200`, default `50`) and `offset`.

### POST `/api/v1/gateway/prompts/{prompt_id}/versions`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `template` | `string` | Yes | 1–65,536 characters, not blank |
| `variables` | `string[]` | No | At most 100. Each must match `^[A-Za-z_][A-Za-z0-9_.]*$`. Defaults to the names the template references |
| `model` | `string` | No | At most 128 characters |
| `params` | `object` | No | Pinned inference parameters (see below) |
| `label` | `string` | No | 1–64 characters |

```bash
curl -X POST https://api.callmissed.com/api/v1/gateway/prompts/7c1e…/versions \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "template": "You are {{brand}} support. Answer in {{language}}. Be brief.",
    "model": "kimi-k2.6",
    "params": { "temperature": 0.3, "max_tokens": 400 },
    "label": "production"
  }'
```

```json
{
  "id": "3f9a…",
  "prompt_id": "7c1e…",
  "version": 5,
  "template": "You are {{brand}} support. Answer in {{language}}. Be brief.",
  "variables": ["brand", "language"],
  "model": "kimi-k2.6",
  "params": { "temperature": 0.3, "max_tokens": 400 },
  "label": "production",
  "created_by_user_id": null,
  "created_at": "2026-08-17T10:04:00Z"
}
```

Creating a version bumps the prompt's `current_version`. Version numbers are allocated as `max + 1`; under heavy concurrency the API retries and, if it still cannot allocate, returns `409 Could not allocate a version number — retry`.

Reusing a label moves it: the previous holder loses it in the same transaction.

### GET `/api/v1/gateway/prompts/{prompt_id}/versions/{version}`

One version by its integer number. `404 Prompt version not found`.

### POST `/api/v1/gateway/prompts/{prompt_id}/versions/{version}/label`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `label` | `string` or `null` | Yes | 1–64 characters. `null` clears the label |

This is the release switch: promote a tested version by pointing `production` at it. `409 Label '…' is already in use` only fires when the label is held elsewhere and cannot be moved.

## Rendering

### POST `/api/v1/gateway/prompts/{prompt_id}/render`

Requires `prompts:read`. **No model is called and no credits are charged.**

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `variables` | `object` | No | At most 100 keys. Values may be string, number, boolean or `null`; strings at most 8,000 characters |
| `version` | `integer` | No | `1..1000000` |
| `label` | `string` | No | 1–64 characters |

Resolution order: `version`, then `label`, then the prompt's `current_version`.

```bash
curl -X POST https://api.callmissed.com/api/v1/gateway/prompts/7c1e…/render \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "label": "production", "variables": { "brand": "Acme", "language": "Hindi" } }'
```

```json
{
  "prompt_id": "7c1e…",
  "version": 5,
  "rendered": "You are Acme support. Answer in Hindi. Be brief.",
  "missing_variables": [],
  "model": "kimi-k2.6",
  "params": { "temperature": 0.3, "max_tokens": 400 }
}
```

`missing_variables` lists placeholders the template referenced that you did not supply. Rendering does **not** fail on a missing variable — check the array and decide whether to proceed.

Errors: `404 Prompt not found`, `404 Prompt version not found`, `404 No version labelled '…'`, `404 Prompt has no versions yet`.

## Presets

A preset is a named model + parameter block. It is the reusable half of a version, useful when several prompts should share one decoding configuration.

`PresetOut`: `id`, `tenant_id`, `name`, `model`, `params`, `prompt_version_id`, `created_at`, `updated_at`.

| Method | Path | Scope |
| --- | --- | --- |
| `GET` | `/api/v1/gateway/prompts/presets` | `prompts:read` |
| `POST` | `/api/v1/gateway/prompts/presets` | `prompts:write` |
| `PATCH` | `/api/v1/gateway/prompts/presets/{preset_id}` | `prompts:write` |
| `DELETE` | `/api/v1/gateway/prompts/presets/{preset_id}` | `prompts:write` |

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1–255 characters, unique per tenant |
| `model` | `string` | Yes | 1–128 characters |
| `params` | `object` | No | Defaults to `{}`. Allowlisted keys only |
| `prompt_version_id` | `UUID` | No | Must be a version in your tenant |

### Pinnable parameters

`temperature`, `max_tokens`, `top_p`, `top_k`, `frequency_penalty`, `presence_penalty`, `repetition_penalty`, `seed`, `stop`, `logit_bias`, `logprobs`, `top_logprobs`, `n`, `tools`, `tool_choice`, `parallel_tool_calls`, `response_format`, `structured_outputs`, `reasoning`, `reasoning_effort`, `provider`, `models`, `route`.

Bounds match the chat completion request (for example `temperature` `0..2`, `n` `1..5`). Anything outside the list returns `422 Unsupported parameter(s): …`; an out-of-range value returns `422 Invalid parameter value: …`.

## Errors

| Status | When |
| --- | --- |
| `403` | Key is missing `prompts:read` / `prompts:write` |
| `404` | Prompt, version, label or preset not found in your tenant |
| `409` | Duplicate prompt or preset name, label already in use, version allocation lost a race |
| `422` | Blank name/template, bad variable name, unsupported or out-of-range pinned parameter, oversized `params` |

Nothing on this page consumes credits.
