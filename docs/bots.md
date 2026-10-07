---
title: "Bots"
description: "Create and manage AI communication agents, verify channel deployment, and load their knowledge base."
slug: "bots"
breadcrumb: "API Reference"
---

# Bots

Create and manage AI communication agents, verify channel deployment, and load their knowledge base.

## Overview

A **bot** is a configured AI agent bound to a channel. The `type` is fixed at creation and determines how the bot is reached:

| type | Channel |
| --- | --- |
| `whatsapp` | WhatsApp text conversations |
| `whatsapp_voice` | WhatsApp voice notes |
| `inbound_call` | Phone calls customers place to you |
| `outbound_call` | Calls the bot places (see [Voice](/docs/voice) for availability) |
| `ivr` | Smart IVR menu flows |

## Credential class: dashboard JWT **or** a `cm_` key with scopes

Both credentials work.

```
Authorization: Bearer <jwt_access_token>
# or
Authorization: Bearer cm_your_api_key
```

| Endpoints | Scope needed by a `cm_` key |
| --- | --- |
| List, get, get prompt, tool catalog, skill catalog, config schema, validate config, probe MCP servers, list/get versions | `bots:read` |
| Create, update, put prompt, patch config, delete, toggle, deploy verify, create/edit/commit/roll back versions | `bots:write` |
| Knowledge list | `knowledge:read` |
| Knowledge add, upload, delete | `knowledge:write` |

A dashboard JWT bypasses the scope check. One extra role rule applies to JWT callers: `POST /{bot_id}/deploy/verify` requires **owner or admin**.

Base URL for every example: `https://api.callmissed.com`. Errors are `{"detail": "..."}`; validation failures are `422`. A missing scope returns `403` naming the scope.

## Credential redaction in `config`

`config` is a free-form JSON object. On the way out, any key whose name looks like a credential (`access_token`, `api_key`, `auth_token`, `app_secret`, `secret`, `password`, `token`, `verify_token`, and similar, matched case-insensitively at any depth) is replaced with `"[REDACTED]"`. The stored value is unchanged; you simply cannot read it back.

`PATCH /{bot_id}/config` goes further and **refuses** a credential-looking key outright with `422 Credentials cannot be stored in bot config`. `POST` and `PUT` still accept one, so prefer `PATCH` when a machine is writing the config.

## What is validated at write time

Most of `config` is stored as-is, which is why [Build an agent programmatically](/docs/build-an-agent) exists: call `GET /bots/config-schema` to learn the keys, and `POST /bots/validate-config` to catch the rest before you write. These sub-keys are the exception — they are schema-checked on every write path (`POST`, `PUT`, `PATCH /config`) and return `422` when malformed:

| Sub-key | Checked |
| --- | --- |
| `analysis_variables` | Post-call extraction schema |
| `input_variables` | Per-call `{{token}}` declarations |
| `stt_keyterms` | List of at most 100 unique strings, each 1-100 chars |
| `pronunciations` | List of at most 100 `{ "word", "say_as" }` objects, each 1-64 chars, unique `word`. The voice says `say_as` instead of `word` (whole-word, case-insensitive); transcripts keep the original text |
| `web_widget_enabled` | Boolean |
| `web_widget` | Object; `title` ≤ 100 chars, `greeting` ≤ 500 chars, `color` a hex colour |
| `fallback_message` | String ≤ 4000 chars |
| `mcp_servers` | List of at most 10 objects, each with a string `id` and an `https://` `url` |
| `commands` | List of at most 30 `{name, instruction}`; names are lower-cased, must be unique, 1-32 chars of letters, digits, spaces, `-` or `_`; instructions 1-2000 chars |
| `disabled_tools` | List of at most 50 tool names, each 1-100 chars, de-duplicated |
| `followup_agent` | Object; `enabled` and `send_recap` booleans, `instructions` a string ≤ 4000 chars, `max_attempts` an integer 1-5 |
| `transcript_retention_days`, `recording_retention_days` | Whole number of days, 1-3650. Applies to calls made after you save this; older calls follow your account's retention setting. The server records when each was saved as `transcript_retention_set_at` / `recording_retention_set_at` ([Voice data retention](/docs/voice-data-retention)) |
| `data_retention_mode` | `"standard"` or `"zdr"` ([Voice data retention](/docs/voice-data-retention)) |

Two normalisations happen silently on every write: `commands` and `disabled_tools` come back normalised (lower-cased, de-duplicated) rather than as you sent them, and the retired `voice_fallbacks` key is dropped. Everything else in `config` is stored verbatim.

---

## GET /api/v1/bots

Lists your tenant's bots, newest first. No parameters.

```bash
curl https://api.callmissed.com/api/v1/bots \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
    "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
    "name": "Support Bot",
    "type": "whatsapp",
    "system_prompt": "You are a friendly support agent for Acme.",
    "config": { "phone_number_id": "109876543210987", "access_token": "[REDACTED]" },
    "is_active": true,
    "created_at": "2026-06-06T12:00:00Z",
    "updated_at": "2026-06-06T12:00:00Z",
    "conversation_count": 42,
    "config_warnings": []
  }
]
```

`403` when a `cm_` key lacks `bots:read`.

## GET /api/v1/bots/tool-catalog

Agent tools a bot can enable. Write the chosen `name` values into `config.tools`; the runtime resolves them from the registry. No parameters. Requires `bots:read`.

```bash
curl https://api.callmissed.com/api/v1/bots/tool-catalog \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "name": "shopify_order_status",
    "description": "Look up a Shopify order's payment + fulfillment status and shipping tracking, by order number (e.g. 1001) or the customer's email.",
    "category": "general",
    "parameters": {
      "type": "object",
      "properties": {
        "order_number": { "type": "string", "description": "Order number, e.g. '1001' (with or without #)." },
        "email": { "type": "string", "description": "Customer email to find their recent orders." }
      }
    },
    "unavailable_on": []
  }
]
```

`parameters` is a JSON Schema object, empty (`{"type": "object", "properties": {}}`) for tools that take no arguments.

`unavailable_on` lists the channels a tool cannot run on, and is `[]` for most tools:

| Value | Meaning |
| --- | --- |
| `voice` | The call runtime never offers this tool on a phone or WhatsApp call. Enabling it on a calling agent does nothing; `POST /api/v1/bots/validate-config` returns an `incompatible` warning for it |
| `whatsapp_personal` | Not offered on a linked personal WhatsApp account |

Copy the `name` verbatim. A name that is not in this catalog is **dropped silently** when the agent starts — no error at write time, no error at call time, the tool simply is not there.

## GET /api/v1/bots/skill-catalog

Skills a bot can enable. Write the chosen `name` values into `config.skills`. No parameters. Requires `bots:read`.

A skill is a bundle, not a flag: switching one on adds the agent tools it names **and** injects that skill's own instructions into the agent's prompt when the model loads it mid-conversation. So it changes how the agent behaves, not only what it can do — which is why the catalog publishes both halves.

<Callout type="warn">
  **Skills apply to chat channels only.** A calling agent never reads `config.skills`: the key is resolved on the chat path, and nothing on a call looks at it. Put the tools a call needs directly in `config.tools`.
</Callout>

```bash
curl https://api.callmissed.com/api/v1/bots/skill-catalog \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "name": "order_support",
    "description": "Look up Shopify orders, products, and customers, and start a draft order/checkout.",
    "tool_names": [
      "shopify_order_status",
      "shopify_product_lookup",
      "shopify_customer_lookup",
      "shopify_create_draft_order"
    ],
    "channels": ["text"],
    "unavailable_on": ["voice"],
    "instructions_preview": "Order-support rules: - Use shopify_order_status to check an order...",
    "instructions_truncated": true,
    "note": "Chat channels only. A calling agent never reads config['skills'], so enabling this skill on one does nothing — put the tools it bundles in config['tools'] instead."
  }
]
```

| Field | Meaning |
| --- | --- |
| `tool_names` | The agent tools this skill adds when it loads. Each one is also in `GET /api/v1/bots/tool-catalog` |
| `channels` | Where the skill is resolved. Always `["text"]` |
| `unavailable_on` | Same vocabulary as the tool catalog. Always `["voice"]` |
| `instructions_preview` | The start of the instruction text the skill injects, capped at 400 characters |
| `instructions_truncated` | `true` when the real instructions are longer than the preview |

An unknown skill name in `config.skills` is **dropped silently** at conversation time, exactly like an unknown tool name. Setting `skills` on a calling agent returns an `incompatible` warning from `POST /api/v1/bots/validate-config`.

## POST /api/v1/bots

Returns `201`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1-255 chars |
| `type` | `string` | Yes | One of `whatsapp`, `inbound_call`, `outbound_call`, `ivr`, `whatsapp_voice`. Immutable after creation |
| `system_prompt` | `string` | No | Max 20000 chars, default `""`. The voice runtime adds its own call-handling instructions on top |
| `config` | `object \| null` | No | Free-form JSON. Only the sub-keys listed above are validated; everything else is stored as-is and reported in `config_warnings` |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Support Bot",
    "type": "whatsapp",
    "system_prompt": "You are a friendly support agent for Acme. Keep replies under 3 sentences.",
    "config": { "tools": ["shopify_order_status"] }
  }'
```

```json
{
  "id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "name": "Support Bot",
  "type": "whatsapp",
  "system_prompt": "You are a friendly support agent for Acme. Keep replies under 3 sentences.",
  "config": { "tools": ["shopify_order_status"] },
  "is_active": true,
  "created_at": "2026-08-04T11:00:00Z",
  "updated_at": "2026-08-04T11:00:00Z",
  "conversation_count": 0,
  "config_warnings": []
}
```

`403` without `bots:write`; `422` on an unknown `type`, a name outside 1-255 chars, a prompt over 20000 chars, or a malformed `analysis_variables` / `input_variables` block.

## GET `/api/v1/bots/{bot_id}`

Same object shape as one row of the list, with an accurate `conversation_count`. Requires `bots:read`.

```bash
curl https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab \
  -H "Authorization: Bearer cm_your_api_key"
```

`404` `Bot not found` when the id is not in your tenant; `422` when it is not a valid UUID.

## PUT `/api/v1/bots/{bot_id}`

Partial update. Omitted fields are unchanged. `type` cannot be changed. Requires `bots:write`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string \| null` | No | 1-255 chars |
| `system_prompt` | `string \| null` | No | Max 20000 chars. Written as **one flat string** — see the warning below |
| `config` | `object \| null` | No | **Replaces** the whole object, not a deep merge |

<Callout type="warn">
  `system_prompt` here is the whole prompt as one block. A stored prompt is normally *section-structured* — the dashboard edits it as an objective, response guidelines and a conversation script — and writing a flat string through this route collapses those three boxes into one, which the owner then has to split apart by hand. Nothing reports it. Use `PUT /api/v1/bots/{bot_id}/prompt`, documented below, instead — unless you genuinely mean to replace the prompt with unstructured text.
</Callout>

```bash
curl -X PUT https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"system_prompt": "You are a concise support agent for Acme."}'
```

Returns the updated bot. Note that `conversation_count` is `0` in the update response; read the bot to get the real count.

`403` without `bots:write`; `404` when not found; `422` on validation failure.

## GET `/api/v1/bots/{bot_id}/prompt`

The agent's instructions split into the four boxes its owner edits in the dashboard. Requires `bots:read`.

`system_prompt` is stored as one string, but it is not written as one. The first three boxes are sections of that string, delimited by marker lines; the fourth, the first message, is `config.greeting`. This route is the published reader of that layout, so a program does not have to know the markers to edit one section.

```bash
curl https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/prompt \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "objective": "You are the voice of Acme. Understand what the caller needs before offering anything.",
  "response_guidelines": "- Friendly and consultative. Short sentences.\n- One question at a time.\n- Never guess a price or a date.",
  "conversation_script": "## 1. Open\n\nGreet the caller and ask how you can help.",
  "first_message": "Hi, thanks for calling Acme. How can I help?",
  "freeform": null
}
```

Exactly one side of the response is populated:

| Field | Meaning |
| --- | --- |
| `objective` | Who the agent is and what it is trying to achieve. `null` when the prompt has no sections |
| `response_guidelines` | How it should speak. `null` when the prompt has no sections |
| `conversation_script` | The steps it works through, in order. `null` when the prompt has no sections |
| `first_message` | The line it speaks when the call connects — `config.greeting`. `null` when there is none |
| `freeform` | The **whole** prompt, when it carries no section markers. `null` for a section-structured prompt |

A prompt written before the section editor existed — or one written flat through `PUT /api/v1/bots/{bot_id}` — comes back in `freeform` with the three blocks `null`. That is the honest reading, not an empty agent: the reader never invents sections that are not there. Reading is deliberately strict, so a prompt with text above the first marker, or with the same marker twice, is also reported as `freeform` rather than guessed at.

`404` `Bot not found` when the id is not in your tenant; `403` without `bots:read`.

## PUT `/api/v1/bots/{bot_id}/prompt`

Writes the prompt back in the same four boxes, with the markers the dashboard parses. Requires `bots:write`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `objective` | `string \| null` | No | Max 8000 chars. The three blocks together must fit in 20000 chars, or the call returns `422` |
| `response_guidelines` | `string \| null` | No | Max 8000 chars |
| `conversation_script` | `string \| null` | No | Max 8000 chars |
| `first_message` | `string \| null` | No | Max 500 chars. Stored as `config.greeting` |
| `freeform` | `string \| null` | No | Max 20000 chars — the same cap as `system_prompt`. The whole prompt as one block. **Cannot be combined** with the three fields above |

```bash
curl -X PUT https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/prompt \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "objective": "You are the voice of Acme. Understand what the caller needs before offering anything.",
    "response_guidelines": "- Friendly and consultative. Short sentences.\n- One question at a time.",
    "conversation_script": "## 1. Open\n\nGreet the caller and ask how you can help.",
    "first_message": "Hi, thanks for calling Acme. How can I help?"
  }'
```

Returns the whole bot, the same shape `PUT /api/v1/bots/{bot_id}` returns, including `config_warnings`.

Three rules decide what a request changes:

- **Omitted means unchanged.** Sending only `first_message` edits the greeting and leaves the prompt alone. Sending only prompt fields leaves the greeting alone.
- **Among the three blocks, omitted means empty.** They are written together, so a block you leave out is cleared. `GET` first, change the one block you mean, and send all three back.
- **`freeform` and the blocks are mutually exclusive.** They are two spellings of the same field; sending both is `422`. Use `freeform` only for a prompt `GET` already returned as `freeform` — round-tripping one that way stores it byte-identically, with no markers injected.

`first_message` goes through the same merge as `PATCH /api/v1/bots/{bot_id}/config`, so the config validator still runs and the findings come back in `config_warnings`. An empty or whitespace-only `first_message` **removes** `greeting` rather than storing a blank one, which is the difference between "no greeting" and "a greeting that is nothing".

As with `PUT /api/v1/bots/{bot_id}`, rewriting the prompt on a non-WhatsApp-voice agent drops a leftover `config.voice_system_prompt`, so a retired spoken persona cannot keep answering calls.

| Status | Cause |
| --- | --- |
| `403` | Key missing `bots:write` |
| `404` | `Bot not found` |
| `422` | `freeform` sent together with a section block, a field over its length limit, or a `first_message` the config validator rejects |

## PATCH `/api/v1/bots/{bot_id}/config`

Changes individual `config` keys without resending the whole object — the safe write for an agent that only knows about the keys it cares about. `PUT` replaces `config` wholesale and will silently drop every key you left out; this does not. Requires `bots:write`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `values` | `object` | No | Keys to set or overwrite. At most 50 keys, each 1-100 characters. Defaults to `{}` |
| `unset` | `array of string` | No | Keys to remove. At most 50, each 1-100 characters. Defaults to `[]` |

The merge is: apply `values` over the stored config, drop every key named in `unset`, then validate the result. A key may not appear in both lists.

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "values": { "greeting": "Namaste! Acme support, how can I help?", "max_call_duration_seconds": 600 },
    "unset": ["fallback_message"]
  }'
```

Returns the whole bot with the merged `config`:

```json
{
  "id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
  "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
  "name": "Support Bot",
  "type": "inbound_call",
  "system_prompt": "You are a friendly support agent for Acme.",
  "config": {
    "voice_model": "gpt-oss-120b",
    "tts_model": "deepgram-aura-2",
    "stt_model": "deepgram-flux-general-en",
    "voice": "Thalia",
    "language": "en",
    "greeting": "Namaste! Acme support, how can I help?",
    "max_call_duration_seconds": 600
  },
  "is_active": true,
  "created_at": "2026-08-04T11:00:00Z",
  "updated_at": "2026-08-04T11:31:12Z",
  "conversation_count": 0,
  "config_warnings": []
}
```

As with `PUT`, `conversation_count` is `0` in this response; read the bot to get the real count.

| Status | Cause |
| --- | --- |
| `403` | Key missing `bots:write` |
| `404` | `Bot not found` |
| `422` | `A config key cannot be set and removed together`, `Config keys must be unique`, `Credentials cannot be stored in bot config`, or a malformed value for one of the validated sub-keys above |

Removing a key is not the same as setting it to `null`: `unset` deletes it, and the runtime then falls back to that key's documented default. Setting it to `null` stores a null and is read back as one.

## GET /api/v1/bots/config-schema

The machine-readable description of `config`: every key the runtime reads, its type, its default, what happens on a real call when you leave it out, and — where the allowlist is closed — the values it accepts. Fetch this **before** composing a config rather than guessing key names. Requires `bots:read`.

| Param | Type | Required | Notes |
| --- | --- | --- | --- |
| `bot_type` | `string` | No | One of `whatsapp`, `inbound_call`, `outbound_call`, `ivr`, `whatsapp_voice`. Narrows the result to the keys that apply to that channel, and builds `minimal_example` for it. Omit it to get every key. Any other value is `422` |

```bash
curl "https://api.callmissed.com/api/v1/bots/config-schema?bot_type=inbound_call" \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "bot_type": "inbound_call",
  "keys": [
    {
      "key": "tts_model",
      "type": "string",
      "channels": ["voice"],
      "summary": "The model that speaks the agent's replies.",
      "default": "bulbul:v3",
      "if_omitted": "Falls back to the Indic-tuned default voice, which sounds wrong on an English line.",
      "allowed_values": ["aura-2-en", "aura-2-es", "bulbul:v2", "bulbul:v3", "deepgram-aura-1", "deepgram-aura-2", "gnani-timbre-v2.0", "gpt-4o-mini-tts", "melotts", "sonic-3.6"],
      "values_depend_on": null,
      "bounds": null,
      "validated_on_write": false
    },
    {
      "key": "voice",
      "type": "string",
      "channels": ["voice"],
      "summary": "Which speaker the chosen tts_model uses.",
      "default": null,
      "if_omitted": "The chosen tts_model's own default speaker is used.",
      "allowed_values": null,
      "values_depend_on": "tts_model",
      "bounds": null,
      "validated_on_write": false
    },
    {
      "key": "max_call_duration_seconds",
      "type": "integer",
      "channels": ["voice"],
      "summary": "Hard cap on one call, in seconds.",
      "default": 600,
      "if_omitted": "The call runs to the platform ceiling.",
      "allowed_values": null,
      "values_depend_on": null,
      "bounds": [30.0, 14400.0],
      "validated_on_write": false
    }
  ],
  "minimal_example": {
    "name": "Support line",
    "type": "inbound_call",
    "system_prompt": "## Objective\n\nYou are the voice of [your business]…",
    "config": {
      "voice_model": "gemma-4-31b",
      "tts_model": "deepgram-aura-2",
      "stt_model": "deepgram-flux-general-en",
      "voice": "Thalia",
      "language": "en",
      "max_call_duration_seconds": 600,
      "greeting": "Hi {{callee_name}}, thanks for calling. How can I help you today?"
    }
  },
  "notes": [
    "Every key is optional. minimal_example is the smallest body that creates an agent that answers a call properly.",
    "Nothing here is enforced on write: a config with errors is still saved, and the findings come back in config_warnings.",
    "voice and language are only valid in combination with tts_model — send the stack together and validate it before you save it.",
    "GET /api/v1/models lists every model id; the voice list for a speech model is on its own model entry.",
    "system_prompt is not one block: the dashboard edits it as an objective, response guidelines and a conversation script… use GET/PUT /api/v1/bots/{bot_id}/prompt instead of writing a flat prompt."
  ],
  "voice_tiers": [ { "id": "t1", "label": "Standard", "price_credits_per_minute": 4.0, "billing": "flat", "minimum_seconds": 30.0, "…": "…" } ],
  "stt_languages": { "deepgram-flux-general-en": ["en"], "…": ["…"] },
  "multilingual": { "stt": ["deepgram-flux-general-multi", "saaras:v3", "…"], "tts": ["bulbul:v3", "sonic-3.6", "…"] }
}
```

`keys` is the full list for the requested channel — 41 keys for a call agent, 9 for a `whatsapp` agent, 46 with no `bot_type` — trimmed above to three. `notes` carries the five standing caveats, preceded, for any key whose allowlist runs past 60 values, by a line saying `allowed_values` was truncated and that `POST /validate-config` still checks against the whole list.

For an agent that speaks (every type except `whatsapp`, and the unfiltered schema), the response carries three more top-level fields:

| Field | Meaning |
| --- | --- |
| `voice_tiers` | The priced [voice tiers](/docs/voice-tiers) an agent can run on via `config.voice_tier` instead of its own STT/LLM/TTS stack — `id`, `label`, `blurb`, `price_credits_per_minute`, `speech_inputs`, `voices`, `voices_searchable`, `languages`, `multilingual`, `billing` (`flat`) and `minimum_seconds` |
| `stt_languages` | For each speech-recognition model, the language codes it can transcribe, in that model's own code format (`hi-IN` or `hi`). Fold to the base subtag before comparing with a TTS language list |
| `multilingual` | `{stt: [...], tts: [...]}` — the speech-recognition and speech models that can follow the caller's language when `language` is `"multi"` |

Per-key fields:

| Field | Meaning |
| --- | --- |
| `key` | The `config` key name, spelled exactly as the runtime reads it |
| `type` | `string`, `integer`, `number`, `boolean`, `array` or `object` |
| `channels` | Which channels read this key: `voice`, `text`, or both |
| `summary` | One line on what the key does |
| `default` | The effective value when the key is absent, or `null` when there is no single default |
| `if_omitted` | What actually happens on a live call when you leave the key out — read this before deciding a key is optional |
| `allowed_values` | The closed allowlist, or `null` when the value is free-form |
| `values_depend_on` | The other key that decides this key's allowlist — `voice` depends on `tts_model`, for example. When this is set, `allowed_values` alone is not enough; validate the pair |
| `bounds` | `[min, max]` for a numeric key, otherwise `null` |
| `validated_on_write` | `true` when a bad value is rejected with `422` at write time. **When this is `false`, a bad value is accepted and fails at call time** — that is exactly what `POST /validate-config` is for |

`minimal_example` is a complete, working body for `POST /api/v1/bots`. Post it as-is and you get an agent that answers a call and speaks.

`403` when a `cm_` key lacks `bots:read`.

## POST /api/v1/bots/validate-config

Checks a `config` object and tells you what is wrong with it. It writes nothing, touches no bot, and never returns an error for a bad config — the findings are the answer. Requires `bots:read`.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `config` | `object` | Yes | The config you are about to write. At most 200 keys |
| `bot_type` | `string \| null` | No | Narrows the check to one channel's keys. Omit it to check against every key. Any value outside the five bot types is `422` |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/validate-config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "bot_type": "inbound_call",
    "config": {
      "tts_model": "deepgram-aura-2",
      "voice": "anushka",
      "max_duration_seconds": 600,
      "greeting": "Hi, thanks for calling."
    }
  }'
```

```json
{
  "ok": false,
  "findings": [
    {
      "key": "max_duration_seconds",
      "severity": "warning",
      "code": "unknown_key",
      "message": "max_duration_seconds is not read from an agent's config — it is the name the call runtime receives. Set max_call_duration_seconds instead, or this setting does nothing.",
      "did_you_mean": ["max_call_duration_seconds"],
      "allowed": null
    },
    {
      "key": "voice",
      "severity": "error",
      "code": "incompatible",
      "message": "'anushka' is not a speaker on deepgram-aura-2 — it belongs to bulbul:v2; set tts_model to it. On a standard voice call the session refuses to start; where a softer path applies, the caller hears a different speaker than you chose.",
      "did_you_mean": ["janus"],
      "allowed": ["agathe", "agustina", "alvaro", "ama", "amalthea", "andromeda"]
    }
  ]
}
```

`ok` is `true` only when no finding has severity `error`. An empty `findings` array means the config is clean. A clean config returns:

```json
{ "ok": true, "findings": [] }
```

| Severity | Meaning |
| --- | --- |
| `error` | The call will not work, or will be audibly wrong — a voice the chosen engine cannot speak, a model id that does not exist, a value outside its bounds |
| `warning` | Accepted, but probably not what you meant — an unknown key, a typo'd tool name, a missing `greeting` |

| Code | Raised when |
| --- | --- |
| `unknown_key` | The key is not one the runtime reads. It is stored and ignored |
| `unknown_value` | The value is not in that key's allowlist |
| `wrong_type` | The value is the wrong JSON type for the key |
| `out_of_range` | A numeric value is outside the key's `bounds`. It is clamped at call time |
| `incompatible` | The value is invalid **given another key** — most often a `voice` that the chosen `tts_model` cannot speak |
| `missing_recommended` | A key you almost certainly want is absent — no `greeting`, for instance, which leaves the caller with silence |

`did_you_mean` is a list — the closest valid spellings, at most three, and `[]` when nothing is close enough. `allowed` is that key's allowlist when the list is closed, capped at 40 values, and `null` otherwise.

<Callout type="warn">
  `POST /bots`, `PUT /bots/{id}` and `PATCH /bots/{id}/config` do **not** reject a config that fails this check. They report it: their response carries a `config_warnings` array holding the same findings. Read it after every write — an empty array is the only confirmation that what you wrote will actually run. Read paths (`GET /bots`, `GET /bots/{id}`) always return `config_warnings: []`; ask `POST /validate-config` to check a stored config.
</Callout>

`403` when a `cm_` key lacks `bots:read`.

## DELETE `/api/v1/bots/{bot_id}`

Returns `204` with an empty body. Requires `bots:write`.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab \
  -H "Authorization: Bearer cm_your_api_key"
```

`403` without `bots:write`; `404` when not found.

## POST `/api/v1/bots/{bot_id}/toggle`

Flips `is_active`. No request body. Requires `bots:write`.

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/toggle \
  -H "Authorization: Bearer cm_your_api_key"
```

Returns the bot with the new `is_active`. `403` without `bots:write`; `404` when not found.

## POST `/api/v1/bots/{bot_id}/deploy/verify`

Live connectivity check against the bot's channel, using credentials stored in the bot's own `config`. No request body. Requires `bots:write`; a JWT caller must also be **owner or admin**.

Which credentials are read depends on the bot `type`:

| Bot type | Reads from `config` | Checked against |
| --- | --- | --- |
| `whatsapp`, `whatsapp_voice` | `phone_number_id`, `access_token` | Meta Graph API |
| `inbound_call`, `outbound_call`, `ivr` | `account_sid`, `auth_token` | Twilio REST API |

The result is also persisted back into `config` as `channel_verified`, `last_verified_at`, and `error_message`.

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/deploy/verify \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{
  "channel_verified": true,
  "last_verified_at": "2026-08-04T11:05:33.129004+00:00",
  "error_message": null
}
```

A failed check is still `200`, with `channel_verified: false` and a reason such as `Missing phone_number_id or access_token`, `WhatsApp API returned 401`, `Twilio API returned 401`, or `Provider API timeout — try again`.

| Status | Cause |
| --- | --- |
| `403` | Key missing `bots:write`, or `Only owners/admins can verify deployments` for a JWT caller |
| `404` | `Bot not found` |

## POST `/api/v1/bots/{bot_id}/mcp-servers/probe`

Connects to each MCP server saved in the bot's `config.mcp_servers`, lists its tools, and reports whether it answered. Read-only: nothing is stored. Requires `bots:read`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `ids` | `array of string \| null` | No | Probe only the servers with these `id`s. At most 10. Omit the body (or send `null`) to probe every saved server |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/mcp-servers/probe \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"ids": ["crm"]}'
```

```json
{
  "servers": [
    {
      "id": "crm",
      "host": "mcp.example.com",
      "ok": true,
      "tool_count": 3,
      "tools": ["find_contact", "create_lead", "log_call"],
      "error": null
    }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `id` | The server's `id` from `config.mcp_servers` |
| `host` | Hostname only. The full URL and any header value you stored are never returned |
| `ok` | `true` when the server answered `tools/list` |
| `tool_count` / `tools` | How many tools it advertised, and their names (the list is capped at 40) |
| `error` | On failure, the class of error only (for example `ConnectError` or `TimeoutException`), otherwise `null` |

An unreachable server is still `200`, with `ok: false`. An `id` that is not saved on the bot is simply absent from `servers`.

| Status | Cause |
| --- | --- |
| `403` | Key missing `bots:read` |
| `404` | `Bot not found` |
| `422` | More than 10 `ids` |

---

# Versions

Stage edits to an agent's `system_prompt` and `config` as a **draft**, publish them atomically, and roll back to any earlier published version. The bot row is always what serves live traffic: a draft changes nothing about what the agent says until you commit it.

- **Version 1 is created lazily.** The first time you create a draft on a bot, a committed `Baseline` version holding the current live prompt and config is created first, so your first commit has something to roll back to. A bot that has never been versioned lists `[]`.
- **One open draft per bot.** A second `POST` returns `409` naming the existing draft.
- **Committed versions are immutable.** Rollback never rewrites history: it appends a **new** committed version carrying the old snapshot and publishes it.
- **History is capped at 50 committed versions per bot.** After each commit or rollback the oldest committed versions beyond that are deleted; a draft is never pruned.
- Writes made outside versioning — `PUT /api/v1/bots/{bot_id}`, `PATCH .../config`, `PUT .../prompt` — still change the live agent directly and do not create a version.

The version object:

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | `uuid` | Version row id |
| `tenant_id`, `bot_id` | `uuid` | Owner and agent |
| `version_number` | `integer` | 1, 2, 3… per bot. Every path below addresses a version by this number |
| `label` | `string \| null` | Your name for the version |
| `system_prompt` | `string` | The prompt snapshot |
| `config` | `object \| null` | The config snapshot, credential-redacted exactly like the bot's own `config` |
| `status` | `string` | `draft` or `committed` |
| `created_by`, `committed_by` | `uuid \| null` | Dashboard user who created / committed it; `null` when done with an API key |
| `commit_message` | `string \| null` | Set on commit; `Rollback to vN` on a rollback version |
| `created_at`, `updated_at`, `committed_at` | `datetime` | `committed_at` is `null` on a draft |

## GET `/api/v1/bots/{bot_id}/versions`

Every version of the bot, newest first. Requires `bots:read`.

```bash
curl https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/versions \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "id": "0f6c2a51-3d1e-4b7a-9c55-1f2e3d4c5b6a",
    "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
    "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
    "version_number": 2,
    "label": "Shorter greeting",
    "system_prompt": "You are a concise support agent for Acme.",
    "config": { "greeting": "Hi, Acme here." },
    "status": "draft",
    "created_by": null,
    "committed_by": null,
    "commit_message": null,
    "created_at": "2026-09-20T10:00:00Z",
    "updated_at": "2026-09-20T10:02:00Z",
    "committed_at": null
  },
  {
    "id": "8b1d7e40-6a2c-4f13-a8e9-2c3d4e5f6a7b",
    "tenant_id": "a0b1c2d3-4455-6677-8899-aabbccddeeff",
    "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
    "version_number": 1,
    "label": "Baseline",
    "system_prompt": "You are a friendly support agent for Acme.",
    "config": { "greeting": "Hi, thanks for calling Acme. How can I help?" },
    "status": "committed",
    "created_by": null,
    "committed_by": null,
    "commit_message": "Baseline snapshot of the live agent",
    "created_at": "2026-09-20T10:00:00Z",
    "updated_at": "2026-09-20T10:00:00Z",
    "committed_at": "2026-09-20T10:00:00Z"
  }
]
```

`404` `Bot not found`.

## GET `/api/v1/bots/{bot_id}/versions/{version_number}`

One version. Requires `bots:read`. `404` `Bot not found` or `Version not found`.

## POST `/api/v1/bots/{bot_id}/versions`

Stages a draft. Every field defaults to the bot's **current live** value, so an empty body `{}` snapshots what is live and lets you edit it. Returns `201` with the draft. Requires `bots:write`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `label` | `string \| null` | No | 1-255 chars |
| `system_prompt` | `string \| null` | No | Max 20000 chars. Defaults to the live prompt |
| `config` | `object \| null` | No | The **whole** config for this version (not a merge). Defaults to the live config. The same sub-keys as `POST /api/v1/bots` are validated |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/versions \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"label": "Shorter greeting", "system_prompt": "You are a concise support agent for Acme."}'
```

<Callout type="warn">
  A `config` you send replaces the live config wholesale **when committed**, and the live config's credential fields (for example `access_token`) are not readable back through the API. Omit `config` to inherit the stored config, credentials included, rather than rebuilding it from a redacted read.
</Callout>

| Status | Cause |
| --- | --- |
| `404` | `Bot not found` |
| `409` | `A draft already exists (vN) — edit or commit it first` |
| `422` | A field outside its bounds, or a malformed validated `config` sub-key |

## PATCH `/api/v1/bots/{bot_id}/versions/{version_number}`

Edits the open draft. Omitted fields are unchanged; `config`, when sent, replaces the draft's config. Same field table as create. Requires `bots:write`.

| Status | Cause |
| --- | --- |
| `404` | `Bot not found` or `Version not found` |
| `409` | `Committed versions are immutable — roll back to it instead` |

## POST `/api/v1/bots/{bot_id}/versions/{version_number}/commit`

Publishes the draft: copies its `system_prompt` and `config` onto the live agent and marks the version `committed`, in one transaction — a call in progress reads either the whole old agent or the whole new one. Requires `bots:write`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `commit_message` | `string \| null` | No | Max 2000 chars |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/versions/2/commit \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"commit_message": "Tighter greeting after call review"}'
```

Returns the committed version.

| Status | Cause |
| --- | --- |
| `404` | `Bot not found` or `Version not found` |
| `409` | `Version is already committed` |

## POST `/api/v1/bots/{bot_id}/versions/{version_number}/rollback`

Rolls back by moving forward: appends a new committed version carrying version `version_number`'s snapshot, with `commit_message` `Rollback to vN`, and publishes it. The target version is untouched. No request body. Returns `201` with the new version. Requires `bots:write`.

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/versions/1/rollback \
  -H "Authorization: Bearer cm_your_api_key"
```

| Status | Cause |
| --- | --- |
| `404` | `Bot not found` or `Version not found` |
| `409` | `Only a committed version can be rolled back to — commit the draft instead` |

---

# Knowledge base

Entries attached to one bot. See [Knowledge Base](/docs/knowledge) for retrieval behaviour.

## GET `/api/v1/bots/{bot_id}/knowledge`

Lists entries for the bot, newest first. Requires `knowledge:read`.

```bash
curl https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/knowledge \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
[
  {
    "id": "k9e8d7c6-b5a4-4321-9876-543210fedcba",
    "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
    "name": "Refund policy",
    "content": "Refunds are issued within 7 business days...",
    "metadata": { "source": "handbook" },
    "file_url": null,
    "file_size_bytes": null,
    "format": null,
    "status": "indexed",
    "error_message": null,
    "created_at": "2026-07-11T09:00:00Z"
  }
]
```

`403` without `knowledge:read`; `404` `Bot not found`.

## POST `/api/v1/bots/{bot_id}/knowledge`

Adds a text entry. Returns `201`. Requires `knowledge:write`.

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | `string` | Yes | 1-255 chars |
| `content` | `string` | Yes | 1-100000 chars |
| `metadata` | `object \| null` | No | Free-form JSON |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/knowledge \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Refund policy",
    "content": "Refunds are issued within 7 business days of the return being received.",
    "metadata": { "source": "handbook" }
  }'
```

Returns the created entry. `403` without `knowledge:write`; `404` `Bot not found`; `422` when `content` is empty or over 100000 chars.

## POST `/api/v1/bots/{bot_id}/knowledge/upload`

Uploads a document as `multipart/form-data` and extracts its text synchronously. Returns `201`. Requires `knowledge:write`.

| Part | Type | Required | Constraints |
| --- | --- | --- | --- |
| `file` | file | Yes | Extension must be `pdf`, `docx`, or `txt`. Max **20 MB** (20,971,520 bytes) |

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/knowledge/upload \
  -H "Authorization: Bearer cm_your_api_key" \
  -F "file=@handbook.pdf"
```

```json
{
  "id": "k1a2b3c4-d5e6-4f70-8192-a3b4c5d6e7f8",
  "bot_id": "b1f2c3d4-5678-90ab-cdef-1234567890ab",
  "name": "handbook.pdf",
  "content": "Acme employee handbook\n\n1. Refunds...",
  "metadata": null,
  "file_url": null,
  "file_size_bytes": 184203,
  "format": "pdf",
  "status": "indexed",
  "error_message": null,
  "created_at": "2026-08-04T11:12:00Z"
}
```

Check `status` on the response. When extraction yields nothing usable the entry is still created with `status: "failed"` and an `error_message` such as `No text content could be extracted`. The `name` is the uploaded filename.

| Status | Cause |
| --- | --- |
| `400` | `File exceeds 20MB limit`, or `Unsupported format: <ext>. Use PDF, DOCX, or TXT.` |
| `403` | Key missing `knowledge:write` |
| `404` | `Bot not found` |

## DELETE `/api/v1/bots/{bot_id}/knowledge/{entry_id}`

Returns `204` with an empty body. Requires `knowledge:write`.

```bash
curl -X DELETE https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/knowledge/k9e8d7c6-b5a4-4321-9876-543210fedcba \
  -H "Authorization: Bearer cm_your_api_key"
```

`403` without `knowledge:write`; `404` `Knowledge entry not found` when the entry is not on that bot in your tenant.
