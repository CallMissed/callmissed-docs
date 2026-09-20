---
title: "Build an Agent"
description: "The ordered recipe for creating and configuring a voice or chat agent entirely over the API: fetch the config schema, validate the config, create the bot, attach knowledge, attach a number, place a test call, read the transcript."
slug: "build-an-agent"
breadcrumb: "API Reference"
---

# Build an Agent

The ordered recipe for creating and configuring a voice or chat agent entirely over the API: fetch the config schema, validate the config, create the bot, attach knowledge, attach a number, place a test call, read the transcript.

:::cards
/docs/bots | Bots API | workflow | Full reference for every route used here
/docs/telephony-api | Telephony API | hash | Buy a number and place calls
/docs/agent-tools-mcp | Account MCP Server | wrench | The same flow as MCP tools
:::

## Who this page is for

A program — your backend, a script, or an AI agent holding a `cm_` key — that has to build a working agent without a human clicking through the console.

The hard part is not the routes. It is `config`: a JSON object whose keys the call runtime reads at call time, where most keys are **not** validated when you write them. A misspelled key is stored and ignored. A voice the chosen speech engine cannot speak is stored and then quietly replaced. Neither shows up as an error — they show up as a call that sounds wrong.

So the recipe below is not "create a bot". It is **ask what the keys are, check the values, then write** — and every step has a route behind it.

Everything here works with `Authorization: Bearer cm_your_api_key`. The scope each route needs is named inline. Base URL for every example: `https://api.callmissed.com`.

## The recipe

::::steps

## Fetch the config schema

Never hand-write a `config` from memory. Ask for the key list first — it is generated from the same constants the call runtime reads, so it cannot drift from reality.

```bash
curl "https://api.callmissed.com/api/v1/bots/config-schema?bot_type=inbound_call" \
  -H "Authorization: Bearer cm_your_api_key"
```

Scope: `bots:read`. Pass the `bot_type` you intend to create and you get only the keys that channel reads.

Three fields in the response decide how you treat each key:

| Field | Use it for |
| --- | --- |
| `if_omitted` | What actually happens on a live call when you skip the key. This is the field that tells you whether a key is genuinely optional |
| `validated_on_write` | `false` means a wrong value is accepted by the write and fails later. Those are the keys worth validating |
| `values_depend_on` | The key whose value narrows this key's allowlist. `voice` depends on `tts_model`, so neither can be checked alone |

The response also carries `minimal_example`, a complete body that `POST /api/v1/bots` accepts and that produces an agent which answers and speaks. Start from that and change what you need, rather than building a config from scratch.

## Compose the config

Take `minimal_example`, then apply your own choices:

- Pick `tts_model` **before** `voice`. The voice allowlist is a property of the speech engine, not a global list.
- Pick `stt_model` and `language` together. A recognition model built for one language set will not silently learn another.
- Write a `greeting`. Without one the agent improvises its opening line, differently on every call.
- Put tool names in `tools`, copied verbatim from `GET /api/v1/bots/tool-catalog`. On a calling agent, skip any tool whose `unavailable_on` contains `voice` — the call runtime never offers it.
- `skills` is for chat agents only. It names bundles from `GET /api/v1/bots/skill-catalog`, each adding a group of tools plus its own instructions. A calling agent never reads the key.

```json
{
  "name": "Acme Support",
  "type": "inbound_call",
  "system_prompt": "You are a friendly support agent for Acme. Keep answers under three sentences and offer to transfer if you cannot help.",
  "config": {
    "voice_model": "gpt-oss-120b",
    "tts_model": "deepgram-aura-2",
    "stt_model": "deepgram-flux-general-en",
    "voice": "Thalia",
    "language": "en",
    "max_call_duration_seconds": 600,
    "greeting": "Hi, thanks for calling Acme. How can I help?",
    "tools": ["search_knowledge_base", "update_contact"]
  }
}
```

## Validate it before you write it

This is the step that separates a working agent from one that fails on its first real call. `POST /api/v1/bots/validate-config` writes nothing and changes nothing — it just tells you what is wrong.

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/validate-config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "bot_type": "inbound_call",
    "config": {
      "voice_model": "gpt-oss-120b",
      "tts_model": "deepgram-aura-2",
      "stt_model": "deepgram-flux-general-en",
      "voice": "Thalia",
      "language": "en",
      "max_call_duration_seconds": 600,
      "greeting": "Hi, thanks for calling Acme. How can I help?",
      "tools": ["search_knowledge_base", "update_contact"]
    }
  }'
```

```json
{ "ok": true, "findings": [] }
```

Scope: `bots:read`. Treat any finding with `severity: "error"` as a stop — fix it and validate again. Treat `warning` as "read the message"; it is usually a typo the runtime will ignore rather than complain about.

## Create the bot

```bash
curl -X POST https://api.callmissed.com/api/v1/bots \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d @agent.json
```

Scope: `bots:write`. Returns `201` and the created bot; keep its `id`.

The write does **not** reject a config that failed validation — it reports it. Read `config_warnings` on the response and confirm it is empty. If you skipped the validate step, this is your last chance to notice.

## Attach knowledge

Give the agent the facts it needs to answer. Each entry belongs to one bot.

```bash
curl -X POST https://api.callmissed.com/api/v1/bots/{bot_id}/knowledge \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Refund policy",
    "content": "Refunds are issued within 7 business days of the return being received."
  }'
```

Scope: `knowledge:write`. For a document, `POST /api/v1/bots/{bot_id}/knowledge/upload` takes a PDF, DOCX or TXT as `multipart/form-data` and extracts the text synchronously.

**Check `status` on the response.** An entry whose text could not be extracted is still created, with `status: "failed"` and an `error_message` — it will not answer anything. See [Knowledge Base](/docs/knowledge) for how entries are retrieved at reply time.

## Attach a phone number (voice agents)

A voice agent with no number cannot be called. Link one of your active numbers to the bot:

```bash
curl -X PATCH https://api.callmissed.com/api/v1/telephony/numbers/{number_id} \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"alias": "Support line", "bot_id": "{bot_id}"}'
```

Scope: `telephony:write`. `GET /api/v1/telephony/numbers` lists what you already own; the [Telephony API](/docs/telephony-api) page covers buying one, including the KYC step.

<Callout type="info">
  The number's own optional `config` object holds per-number overrides that **win over the bot's config on that number's calls**. Two numbers can share one agent and still greet differently. It is also a place a mistake hides: an override you forgot beats the config you just fixed.
</Callout>

## Place a test call

```bash
curl -X POST https://api.callmissed.com/api/v1/telephony/calls \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "from_number_id": "9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d",
    "to_e164": "+919000000000",
    "bot_id": "{bot_id}"
  }'
```

Scope: `telephony:write`. Call yourself, not a customer. The response carries the call `id` and a `voice_session_id` — that second id is how you read what was said.

## Read the transcript

The transcript is the only check that tells you the agent actually worked: the right voice, the right language, the greeting spoken, the tools called.

```bash
curl "https://api.callmissed.com/v1/voice/sessions/{voice_session_id}/transcript?format=json" \
  -H "Authorization: Bearer cm_your_api_key"
```

Each turn carries `user_transcript`, `agent_response`, `interrupted`, the per-stage latencies (`stt_ms`, `first_token_ms`, `first_audio_ms`, `total_ms`) and the `llm_model` that answered. `format=txt` gives readable alternating lines if a human is reading it.

If the `voice_session_id` on the call is still `null`, the call has not connected yet — poll `GET /api/v1/telephony/calls/{call_id}` until it is set.

## Iterate without clobbering

Change one key with `PATCH /api/v1/bots/{bot_id}/config`, which merges:

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/{bot_id}/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"values": {"voice": "Asteria"}}'
```

Scope: `bots:write`. Use `PUT /api/v1/bots/{bot_id}` only when you genuinely mean to replace the whole `config` — it drops every key you leave out.

::::

## Common mistakes

Five ways a `config` is accepted, stored, and returned to you unchanged — and then behaves wrongly. **None of them raises an error at write time.** Most raise nothing at call time either: the call just is not what you configured, and you find out from a customer. `POST /api/v1/bots/validate-config` reports all five before that happens.

### 1. A `voice` the chosen `tts_model` cannot speak

The speaker list belongs to the speech model, not to the platform. `anushka` is a `bulbul` speaker; `thalia` is a `deepgram-aura-2` speaker. Send one with the other and the write succeeds, then the call does not: on the standard speech pipeline the engine rejects the speaker and the session **cannot start**, and on a managed voice model the agent's own default speaker is substituted — a different voice, possibly a different gender, with no warning anywhere.

```json
{ "tts_model": "deepgram-aura-2", "voice": "anushka" }
```

```json
{
  "key": "voice",
  "severity": "error",
  "code": "incompatible",
  "message": "'anushka' is not a speaker on deepgram-aura-2 — it belongs to bulbul:v2. On a standard voice call the session refuses to start; where a softer path applies, the caller hears a different speaker than you chose.",
  "did_you_mean": ["janus"],
  "allowed": ["agathe", "agustina", "alvaro", "ama", "amalthea", "andromeda"]
}
```

**Fix:** set `tts_model` first, then pick `voice` from that model's list. The schema marks this with `"values_depend_on": "tts_model"` — a key flagged that way cannot be checked on its own, so validate the pair together. Matching is case-insensitive, so `Thalia` and `thalia` are the same speaker.

### 2. `language: "en"` on an Indic stack

On a `bulbul` or `saaras` stack a bare `"en"` is not rejected — it is expanded to `"en-IN"`. Indian English, not the neutral English you asked for. The same rewrite on a language the stack does not serve at all is worse: you get a different language, silently.

```json
{ "tts_model": "bulbul:v3", "stt_model": "saaras:v3", "language": "en" }
```

```json
{
  "key": "language",
  "severity": "warning",
  "code": "incompatible",
  "message": "'en' is not a language 'saaras:v3' accepts; it is read as 'en-IN'. Send the full code so it is unambiguous.",
  "did_you_mean": ["en-IN"]
}
```

**Fix:** always send the full code (`en-IN`, `hi-IN`) rather than a bare one. The finding is a `warning` when the rewrite is just the region being filled in, and an **`error`** when the resolved language is not the one you asked for.

### 3. `max_duration_seconds` instead of `max_call_duration_seconds`

`max_duration_seconds` is a real field — on the [Voice Sessions API](/docs/voice-sessions-api) request body, which is a different surface. In `bot.config` the key is `max_call_duration_seconds`. Copy the wrong one across and the agent's config contains a duration cap that caps nothing.

```json
{
  "key": "max_duration_seconds",
  "severity": "warning",
  "code": "unknown_key",
  "message": "max_duration_seconds is not read from an agent's config — it is the name the call runtime receives. Set max_call_duration_seconds instead, or this setting does nothing.",
  "did_you_mean": ["max_call_duration_seconds"]
}
```

**Fix:** use `max_call_duration_seconds`, bounds `30`–`14400`. This is the general case, not a special one: **any** key the platform does not read is stored and ignored, and comes back as `unknown_key` with the nearest real key in `did_you_mean`. `variables` is the other name that looks right and does nothing — the key is `input_variables`.

### 4. A typo in `tools` or `skills`

Tool and skill names are resolved against a registry when the agent starts. A name that is not in the registry is dropped — not logged to you, not rejected on write. The agent runs with one fewer capability than you think it has, and the only symptom is that it never uses the tool.

```json
{ "tools": ["get_order_statuss"], "skills": ["bookng"] }
```

```json
{
  "key": "tools",
  "severity": "warning",
  "code": "unknown_value",
  "message": "tools contains 'get_order_statuss', which is not a known name. It is ignored at call time.",
  "did_you_mean": ["shopify_order_status", "woocommerce_order_status"],
  "allowed": ["add_internal_note", "calcom_book", "calculator", "escalate_to_human", "…"]
}
```

**Fix:** copy names from `GET /api/v1/bots/tool-catalog` and `GET /api/v1/bots/skill-catalog` rather than typing them. `allowed` on the finding is the live registry, so you can correct it without a second request.

A name can also be real and still do nothing on this agent. Two findings say so, both `warning` / `incompatible`:

- a tool whose `unavailable_on` contains `voice`, listed in a calling agent's `tools` — the call runtime drops its category, so the agent never sees it;
- `skills` set on a calling agent at all — skills are resolved on chat channels only, so the named bundles are neither offered nor loaded on a call.

### 5. An unknown `tts_provider`

`tts_provider` is the legacy speech switch and takes exactly three values. A speech **model** id is not one of them — `deepgram` looks plausible and is not accepted. The value is discarded, and the engine is then whatever `tts_model` says; with no `tts_model` either, that is the Indic-tuned default, which sounds wrong on an English line.

```json
{
  "key": "tts_provider",
  "severity": "error",
  "code": "unknown_value",
  "message": "'deepgram' is not an accepted value for tts_provider. It is ignored and the platform default is used instead.",
  "did_you_mean": [],
  "allowed": ["cartesia", "elevenlabs", "sarvam"]
}
```

**Fix:** prefer `tts_model`, which picks the provider for you, and leave `tts_provider` out entirely. Set it only when you are maintaining an older config.

### And the one that is not a typo: no `greeting`

Not a wrong value — an absent one. With no `greeting` the agent still speaks first, but not the line you meant: on the standard pipeline the model improvises an opener from the system prompt, and on a managed voice model a generic platform line is used. Neither is the brand sentence you would have written, and neither is stable between calls.

```json
{
  "key": "greeting",
  "severity": "warning",
  "code": "missing_recommended",
  "message": "No greeting: the agent improvises its own opening line, or uses a generic one. Set a greeting to control the first thing a caller hears."
}
```

This is why `if_omitted` is in the schema at all. Read it for every key you decide to skip.

## Doing this from an MCP client

Every step above is also an [Account MCP server](/docs/agent-tools-mcp) tool, so an agent that speaks MCP does not need to write HTTP calls at all.

## What this page does not cover

- **Buying a number and KYC** — see [Telephony API](/docs/telephony-api).
- **Custom tools that call your own endpoints** — see [Agent Tools](/docs/agent-tools).
- **How knowledge is retrieved at reply time** — see [Knowledge Base](/docs/knowledge).
- **Browser and mobile voice, without a phone number** — see [Voice Sessions API](/docs/voice-sessions-api).
