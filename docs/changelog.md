---
title: "Changelog"
description: "Latest updates, new features, and improvements to the CallMissed API."
slug: "changelog"
breadcrumb: "Resources"
---

# Changelog

Latest updates, new features, and improvements to the CallMissed API.

## October 2026

### Deepgram Think-Fast and phoneLLM mini chat models (2026-10-31)

- **`deepgram-think-fast`** (Deepgram) and **`deepgram-phonellm-mini`** (Pipecat) are early-access chat models for voice agents, on `POST /v1/chat/completions` and as the `llm_model` of a voice agent. 32,768-token context, tool calling and streaming, text only. `deepgram-think-fast` supports `reasoning_effort` (`none` default, `minimal`, `low`, `medium`, `high`); `deepgram-phonellm-mini` does not reason. Both are $0.3646 per 1M input tokens and $0.9375 per 1M output tokens, paid plans only, and not a default anywhere.
- **Training on your data.** Conversations on `deepgram-think-fast` are used by the provider to improve its models; the opt-out does not apply to it. `deepgram-phonellm-mini` is excluded from the provider's model training.
- **Processed outside India.** India-only keys and accounts get `region_unavailable`, and voice agents with `hosting: "india"` cannot select them. `tool_choice` takes only `"auto"` or `"none"`, and `response_format` returns `400 unsupported_parameter`. See [Models](/docs/models#deepgram-think-fast-and-phonellm-mini-early-access).
- **Languages.** In our tests `deepgram-think-fast` was reliable in English, Hindi, Marathi, Telugu and Kannada, and `deepgram-phonellm-mini` only in English and Hindi. Validating a voice agent's config now flags other languages (an error for `deepgram-phonellm-mini`, a warning for `deepgram-think-fast`).
- The catalogue is now 144 models: 70 LLMs.

### Gnani Prisma v3, and Prisma v2.5 retired (2026-10-30)

- **`gnani-prisma-v3.0`** is Gnani's newest India-first telephony speech-to-text model: English plus 9 Indian languages (`hi-IN`, `bn-IN`, `ta-IN`, `te-IN`, `kn-IN`, `ml-IN`, `mr-IN`, `gu-IN`, `pa-IN`, `en-IN`), automatic language detection with `language: "indic-auto"`, mixed-language speech and keyterm boosting. It runs real-time and batch. Price is $0.2813 / hour, the same as v2.5. Paid plans only.
- **`POST /v1/audio/transcriptions`.** `language` takes one of the 10 codes, a comma-separated list of up to 4 of them, or `indic-auto`; a base code such as `hi` is read as `hi-IN`. Audio is limited to 30 seconds per request (v2.5 allowed 60); longer audio returns `400 audio_too_long`.
- **Voice agents and sessions.** On `gnani-prisma-v3.0`, `language: "multi"` follows a multilingual caller (it detects the language, so the voice follows), and per-agent `stt_keyterms` are applied. It is processed in India, so it stays available in India mode.
- **`gnani-prisma-v2.5` is retired.** The ID keeps working on the API, in voice agents and in saved agents. It is answered by `gnani-prisma-v3.0`, the response keeps the ID you sent, it keeps its price ($0.2813 / hour) and it is no longer listed by `GET /api/v1/models`. The same 30-second per-request limit now applies. Agents that set the old code-mixed codes `en-hi-in-cm` or `en-hi-IN-latn` keep working. See [Retired and deprecated models](/docs/models#retired-and-deprecated-models).
- The catalogue size is unchanged: one model added, one retired.

### Deepgram updates (2026-10-30)

- **Managed voice-agent models.** `deepgram-voice-anthropic-claude-sonnet-5-5` (Advanced, $0.1698/min), `deepgram-voice-anthropic-claude-haiku-5-5` (Standard, $0.07813/min), and the early-access `deepgram-voice-deepgram-think-fast` and `deepgram-voice-deepgram-phonellm-mini-alpha-2-fp8` (Standard). Conversations on `deepgram-voice-deepgram-think-fast` are used by the provider to improve its models.
- **Speech to text.** New `deepgram-nova-3-pharma` (English only) for drug names. `deepgram-nova-3-medical` now accepts `language=multi` (10 languages). `POST /v1/audio/transcriptions` takes a repeatable `language_hint` field for `deepgram-nova-3` with `language=multi`.
- **Pricing by mode.** `deepgram-nova-3`, `-medical` and `-pharma` are $0.325 / hour for files and $0.4792 / hour on live voice sessions, or $0.5729 / hour when the session's language is `multi` (`-pharma` is English only, so always $0.4792) (previously $0.3021 / hour for both). `deepgram-flux-general-en` is $0.4792 / hour (previously $0.4063).
- **Flux TTS on `/v1/audio/speech`.** `deepgram-flux-tts` ($0.4688 / 10K characters) now has 266 voices in 42 languages, including Hindi, Tamil, Telugu, Kannada, Marathi, Gujarati, Bengali and Urdu (32 languages are early access), and supports `stream: true`. Input is capped at 2000 characters, and at 400 without streaming.
- The catalogue is now 142 models: 47 speech-to-text and 16 text-to-speech.

### Preview: Microsoft AI speech models (2026-10-30)

- **Text to speech.** `mai-voice-2.1` ($0.2292 / 10K characters) is Microsoft AI's most expressive voice model, for narration and long-form audio. `mai-voice-2.1-flash` ($0.1563 / 10K characters) is built for low latency and real-time voice agents. Both have 97 voices across 23 languages, including Hindi (`hi-in-arjun`, `hi-in-dhruv`, `hi-in-grant`, `hi-in-harper`, `hi-in-kavya`, `hi-in-priya`) and Indian English (`en-in-dhruv`, `en-in-priya`). Plain text only, speed 0.5 to 2.0. They work on `POST /v1/audio/speech` and in voice agents and voice sessions (`tts_model`). See [Voices](/docs/tts-voices#mai-voice-preview).
- **Speech to text.** `mai-transcribe-2` ($0.375 / hour) transcribes files in 60 languages with automatic language detection and Hinglish code-switching, up to 2 hours per file, on `POST /v1/audio/transcriptions`. It is not available in voice sessions or translation. See [Speech to Text](/docs/speech-to-text#mai-transcribe-2-preview).
- **All three are preview.** Microsoft ships them with no SLA and may change or retire them at short notice. If one is retired, its ID keeps working and is answered by a named successor, as in [Retired and deprecated models](/docs/models#retired-and-deprecated-models), so an agent that uses it keeps working but may sound different. Use generally available models for production-critical agents. They are paid-plan only, are never a default, are not processed in India, and the voices cannot be used in `tts_fallbacks`. `GET /api/v1/models` marks them with `preview: true`.
- The catalogue is now 137 models: 46 speech-to-text and 16 text-to-speech.

### API behaviour fixes

- **Ending a voice session made by another key now needs `telephony:read`.** `DELETE /v1/voice/sessions/{id}` follows the same rule as reading a session: a key can always end the sessions it created, including calls it placed with `POST /api/v1/telephony/calls`, and needs `telephony:read` for sessions the console or another key created. Without it the call returns `403`.
- **Key budget refusals say so.** When an API key's own spending budget, not the account balance, cannot cover a request, the response is now `budget_exceeded` instead of `insufficient_credits`.
- **`n` > 1 near the time limit.** A non-streaming chat completion that reaches the request time limit after some of its `n` choices are finished now returns those choices, billed as delivered, instead of an error.
- **Audio of unknown length.** Speech-to-text uploads whose duration cannot be read from the file are no longer refused up front for a worst-case estimate; they are billed on the duration actually processed.

### Hosted in India voice agents

- **New `hosting` setting.** Set `hosting: "india"` in a voice agent's config, or **Where calls run → Hosted in India** in the console's Voice card. Speech recognition, the model, the voice, their backups, knowledge-base and memory search, and post-call analysis, notes, scoring and contact memory then all run in India.
- **Models you can choose in India mode.** Speech to text: `saaras:v4`, `gnani-prisma-v2.5`, `microsoft-speech-stt` and the Deepgram Nova and Flux models (Whisper excluded). Models: `gemma-4-31b`, `sarvam-105b`, `sarvam-105b-conversations`, `glm-5.3` and `deepseek-v4.1-flash`. Voices: `bulbul:v3`, `bulbul:v4-flash`, `gnani-timbre-v3.0`, `microsoft-neural-tts` and `deepgram-aura-2` / `deepgram-aura-1`. Of the voice plans, only `t1` qualifies. Speech-to-speech models are not available.
- **Saving is checked.** Saving an India-hosted agent with a model outside this list returns `422` naming the model. `GET /api/v1/bots/config-schema` lists the allowed models under `india_hosting`, and each plan carries `hosted_in_india`.
- **`hosting: "global"`** runs `gemma-4-31b` on a global host first, with India as its fallback; nothing else changes. Without `hosting`, agents behave as before. Prices are the same in every mode.
- **`hosted_in_india` in the model catalogue now means processed in India on every request.** Models served from a global pool are no longer flagged, even when their endpoint is in India.

### New voice-agent models: Microsoft Speech and Microsoft Neural voices

- **Speech to text.** `microsoft-speech-stt` ($1.0417 / hour) is Microsoft's real-time streaming recognizer, processed in India. It hears 81 languages (148 locales), including Hindi, Bengali, Tamil, Telugu, Marathi, Gujarati, Kannada, Malayalam, Punjabi, Odia, Assamese, Urdu, Bhojpuri and Indian English, one language per call.
- **Text to speech.** `microsoft-neural-tts` ($0.1563 / 10K characters), processed in India, has 57 voices across 13 languages, with plain-text input and a speaking-speed control (`tts_speed`, 0.5 to 2.0).
- **Voice sessions only.** Both run inside voice agents and voice sessions. `POST /v1/audio/transcriptions` and `POST /v1/audio/speech` return `400` for them. See [Voices](/docs/tts-voices#microsoft-neural-tts--microsoft-neural-voices) and [Speech to Text](/docs/speech-to-text#voice-session-only-models).
- The catalogue is now 134 models: 45 speech-to-text and 14 text-to-speech.

### New voice-agent models: AssemblyAI, Inworld, Rime, Fish Audio, Gradium and Deepgram Flux TTS

- **Speech to text.** `assemblyai-universal-3.6-pro` ($0.4688 / hour) is AssemblyAI's newest streaming model with built-in turn detection and Hindi, Marathi, Urdu and Indian English among its languages. `inworld-stt-1` ($0.1563 / hour) is a low-cost English-only model.
- **Text to speech.** `deepgram-flux-tts` ($0.4688 / 10K characters, 16 English voices), `rime-coda` ($0.5208, 8 languages including Hindi), `inworld-tts-2` ($0.2604) and `inworld-tts-2-flash` ($0.1563) with 94 languages including 11 Indian languages, `fish-audio-drama-3` ($0.5208, 13 languages) and `gradium-tts-beta` ($0.50, 5 languages).
- **Voice sessions only.** All eight run inside voice agents and voice sessions. `POST /v1/audio/transcriptions` and `POST /v1/audio/speech` return `400` for them. See [Voices](/docs/tts-voices#voice-session-voices) and [Speech to Text](/docs/speech-to-text#voice-session-only-models).
- The catalogue is now 132 models: 44 speech-to-text and 13 text-to-speech.

### Models moved, retired and deprecated

- **Same IDs, new hosting.** `kimi-k2.5`, `kimi-k2.6`, `kimi-k2.7-code`, `gpt-oss-120b`, `DeepSeek-V4-Pro` and `DeepSeek-V4-Flash` keep their IDs and prices. `kimi-k2.5` now has a 262,144-token context window and `gpt-oss-120b` 131,072. `DeepSeek-V4-Pro` and `DeepSeek-V4-Flash` take 1,000,000 input tokens.
- **Price updated 2026-10-17.** `DeepSeek-V4-Pro` input is now $1.813 per 1M tokens (was $1.375) and cached input $0.151 (was $0.04583); output is unchanged at $4.125. `DeepSeek-V4-Flash` cached input is now $0.02917 per 1M (was $0.01458); input and output are unchanged.
- **Retired IDs keep working.** `glm-4.7-flash`, `glm-5.2`, `nemotron-3-super`, `gemma-4-26b-a4b-it`, `mistral-small-3.1`, `flux-2-klein-9b`, `lucid-origin`, `phoenix-1.0`, `sdxl-lightning`, `dreamshaper-8-lcm`, `aura-2-en`, `aura-2-es`, `melotts`, `nova-3` and `whisper-large-v3-turbo` are no longer listed. Requests that name them are answered by a successor model and the response keeps the ID you sent. Most keep their price; `gemma-4-26b-a4b-it`, `mistral-small-3.1`, `sdxl-lightning`, `dreamshaper-8-lcm`, `melotts` and `whisper-large-v3-turbo` now bill at their successor's price. See [Retired and deprecated models](/docs/models#retired-and-deprecated-models) for each successor and price.
- **Free plan.** The retired LLM, speech-to-text and text-to-speech IDs stay free. The free plan no longer includes an image model: a free key that requests an image gets `403 model_not_available`. The default image model for `POST /v1/images/generations` and social posts is now `flux-2-pro`. The free tier is now 11 listed models and the catalogue is 124.
- **Deprecated, sunset 2026-11-16:** `gpt-4o`, `gpt-4.1`, `gpt-5.5`, `gpt-5.6-sol` and `gpt-5.6-terra` (successor `gpt-6-sol`); `gpt-5-mini` and `gpt-5.6-luna` (`gpt-6-luna`); `gemini-3.5-flash`, `gemini-3.6-flash` and `gemini-3.7-flash` (`gemini-3.8-flash`); `gemini-3.1-flash-lite` (`gemini-3.5-flash-lite`); `gpt-realtime`, `gpt-realtime-1.5` and `gpt-realtime-2` (`gpt-realtime-2.1`); `gpt-realtime-mini` (`gpt-realtime-2.1-mini`); `gpt-image-1.5` and `gpt-image-2` (`gpt-image-2.5-flare`); `flux-1.1-pro` (`flux-2-pro`); and the older Deepgram speech-to-text IDs (`deepgram-nova`, `deepgram-nova-2`, `deepgram-enhanced`, `deepgram-base` to `deepgram-nova-3`; their medical variants to `deepgram-nova-3-medical`; their phonecall, meeting, finance, conversationalai, voicemail and video variants to the matching `deepgram-nova-2-*` model). The full table is in [Retired and deprecated models](/docs/models#retired-and-deprecated-models). Until then they are served as themselves; responses carry `Deprecation`, `Sunset` and `Link` headers, and the catalogue marks them `deprecated: true` with `sunset_date` and `successor`. From 2026-11-16 a request that names one is answered by and billed as its successor.
- `aura-2-en` and `aura-2-es` keep their voices but now accept up to 2,000 characters per request, the limit of the model that answers them.

### Gnani Timbre v3, and older speech models retired

- **`gnani-timbre-v3.0`** replaces `gnani-timbre-v2.0`. It has 12 voices (default `Shivani`), every voice speaks all 10 Indian languages plus Hinglish, it streams, and the text accepts expressive audio tags such as `<laugh>` and `<pauses>`. Price is unchanged at $0.27 per 10K characters. See [Text to Speech](/docs/text-to-speech#gnani-timbre-v3-audio-tags). Requests naming `gnani-timbre-v2.0` now return the unknown-model error.
- **`bulbul:v2` and `saaras:v3` are retired.** Use `bulbul:v3` or `bulbul:v4-flash` for speech, and `saaras:v4` for transcription (same languages, modes and price). `saaras:v4` is now the default when `model` is omitted. Voice sessions that still name `saaras:v3` or `saarika:v2.5` are served as `saaras:v4`.
- The catalogue is now 139 models and the free tier is 26.

### New models: DeepSeek V4.1 Flash and Bulbul V4 Flash

- **`deepseek-v4.1-flash`** — a 1M-token reasoning model with tool calling and text-only input. It always reasons: `reasoning_effort` takes `"low"`, `"high"` or `"max"` (the default). Pricing per 1M tokens is $0.427 in / $1.771 out ($0.0104 cached input). Paid plans only.
- **`bulbul:v4-flash`** — low-latency Indic text to speech with 222 persona voices (`<name>_<language>_<style>`, default `shubh_enhi_ads`) across 9 Indian languages and Hinglish, at $0.2083 per 10K characters. It is on the free tier. Input is capped at 2,500 characters. `ml-IN` and `od-IN` are not available on it; use `bulbul:v3`.
- The catalogue is now 140 models and the free tier is 27.

### flux-2-dev removed

- `flux-2-dev` has been removed: every request to it was failing upstream. `POST /v1/images/generations` with `"model": "flux-2-dev"` now returns `404` with code `model_not_found`. Use `flux-2-klein-9b`, which is also free. `nano-banana-2` and `flux-1.1-pro` now fall back to `flux-2-klein-9b`. The free tier is 26 models (5 image). Past images and usage records are unchanged.

### Nova Sonic models removed

- `nova-sonic-2` and `nova-sonic` have been removed from the catalog. `GET /api/v1/models` no longer lists them, and a voice session that names either returns `422`. Use `gpt-realtime-2.1-mini` for speech-to-speech. An agent still configured with either model keeps answering calls on the standard STT, LLM and TTS pipeline; past usage and session cost records are unchanged.

### Upstream failures now return 503 with Retry-After

- Failures of a model or channel behind the API, and gateway timeouts, now return `503` (with a `Retry-After` header) instead of `502` or `504`. Our CDN replaces an origin `502` or `504` with its own HTML error page, so your client never saw the JSON error body or its reason code. Reason codes and retry advice are unchanged: only the status moved. A timeout that may have applied the action (for example a social action or a Facebook publish) is still ambiguous, so check before retrying. See [Errors](/docs/errors).

### New models: Claude Opus 5.5, Sonnet 5.5 and Haiku 5.5

- **`claude-opus-5-5`**, **`claude-sonnet-5-5`** and **`claude-haiku-5-5`** — Anthropic's Claude 5.5 family, available on the Pro plan and higher (free and Starter keys get `403 model_not_available`). 1M context, 128K max output, text + image input, reasoning and tool calling.
- **Pricing per 1M tokens** — Opus $4.167 in / $20.83 out ($0.2083 cached input); Sonnet $2.083 in / $10.42 out ($0.1042 cached); Haiku $0.1042 in / $0.5208 out ($0.01042 cached), rising to $0.5208 / $2.604 ($0.05208 cached) for prompts over 100K tokens.
- **`reasoning_effort`** — accepts `low`, `medium`, `high`, `xhigh` and `max`; `none` and `minimal` run as `low`. `temperature`, `top_p` and `top_k` are ignored. See [API speed](/docs/api-speed) and [function calling](/docs/chat-function-calling) for the `anthropic_thinking` field to send back with tool calls.

### Nova Sonic models under maintenance

- `nova-sonic-2` and `nova-sonic` are under maintenance: `POST /v1/voice/sessions` returns `503` with a message naming the alternative, and `GET /api/v1/models` shows `status: "maintenance"`. Use `gpt-realtime-2.1-mini` for speech-to-speech. An agent still configured with either model keeps answering calls on the standard STT, LLM and TTS pipeline.

### Usage API — costs in US$, prompt-cache totals

- `cost_usd` on [`/v1/usage/summary`](/docs/usage-api), `/v1/usage/logs` and `/v1/usage/logs.csv` (and the `get_usage_summary` / `list_usage_logs` MCP tools) is now the US$ you were billed at the published rate (1 credit = ₹1, US$1 = ₹96). It was previously reported in internal billing units (`0.01` per credit), about 4% below the US$ amount; credits = US$ × 96. Past rows are converted the same way, so a window spanning the change stays consistent.
- `/v1/usage/summary` reports prompt-cache usage: `totals.total_cache_read_tokens`, `totals.total_cache_creation_tokens`, and `cache_read_tokens` / `cache_creation_tokens` per service. `input_tokens` keeps its meaning (the uncached part of the prompt).

### Per-model parameters follow each model's documentation

- **Output limits** — `GET /api/v1/models` now lists `max_output_tokens` where a model documents one, and `max_tokens` above it returns `400 max_tokens_too_large` on `/v1/chat/completions`, `/v1/responses` and `/v1/messages`. The `gemini-*` models now allow up to 65,536 output tokens (previously capped at 8,192), and an omitted `max_tokens` leaves the model's own default.
- **`reasoning_effort`** — `xhigh` and `max` now reach every model instead of being cut to `high` first; each model maps them to its own top level. `DeepSeek-V4-Pro`/`-Flash` and `glm-5.2` now honour their levels, `nemotron-3-super` switches to its low-effort mode for `low`/`none`, `sarvam-105b` switches thinking fully off for `none`, and `gpt-5-mini` uses `minimal`. See [API speed](/docs/api-speed).
- **Sarvam** — when you omit `max_tokens`, we send 4,096 (16,384 for `glm-5.3`) so thinking no longer cuts answers short at 2,048. A `glm-5.3` or `gemma-4-31b` request over 10 MB returns `413`.
- **Models** — `gpt-4.1` serves a 300,000-token context window. The realtime `gpt-realtime-2*` models take 32,000 input and 4,096 output tokens. `nova-sonic` was retired upstream on 2026-09-14 and returns `503`; use `nova-sonic-2`.
- **Images and speech** — image sizes outside a model's range, and `negative_prompt` on `lucid-origin`, return `400`. `gpt-4o-mini-tts` has 13 voices and rejects an unknown voice with `400`. `gpt-4o-transcribe-diarize` returns speaker `segments` with `response_format` `verbose_json` or `diarized_json`; it is no longer accepted as a voice-session `stt_model` (`422`), because live transcription cannot label speakers. grok-4.3 reasoning tokens now count as output tokens, and its images must have at least 512 pixels.

### Email domains — neutral `503` reason

- The `503` returned by the email domain routes when provisioning or verification is temporarily unavailable now carries `reason: "provider_unavailable"`. Deployments still being upgraded may return the older `acs_unavailable` for the same condition; status, body shape and retry advice are identical, so match both strings. See [Email limits](/docs/email-limits).

### Voice session webhooks

- `voice_session.ended` and `voice_session.failed` now fire exactly once per session, on every way a session can end (the agent hanging up, `DELETE`, the session timing out), and only after the end has been saved. If two end signals arrive together, the first one decides `end_reason` and `duration_seconds`.
- A session that ends because the agent hit an error (`end_reason: "agent_error"`) now has status `failed` and fires `voice_session.failed` on every path; it previously fired `voice_session.ended` for some sessions. See [Webhooks](/docs/webhooks#voice-session-end-reasons).

### Speech-to-text error codes

- `/v1/audio/transcriptions` and `/v1/audio/translations` now return `400 invalid_request` for a problem with the request (unsupported audio, a streaming-only or unknown model), `429 rate_limit_exceeded` when rate limited, and `503 service_unavailable` when transcription is temporarily unavailable, the same code `/v1/audio/speech` uses. See [Speech-to-Text](/docs/speech-to-text#errors).

### Batch API, zero data retention, per-end-user budgets and prompt caching

- **Batch API** — OpenAI-compatible `POST /v1/files` and `POST /v1/batches`: upload a JSONL file of requests, run it asynchronously, and download the results. Gated by the key's `llm` permission. See [Batch API](/docs/batch).
- **Zero data retention** — send `"provider": {"zdr": true}` to route a request only to models that keep no data, or enforce it for a whole key or account. See [Gateway controls](/docs/gateway-controls#zero-data-retention).
- **Per-end-user budgets** — monthly credit caps per end user of your app, managed under `/api/v1/gateway/end-user-budgets` (scopes `end_user_budgets:read` / `end_user_budgets:write`). See [Gateway controls](/docs/gateway-controls#per-end-user-budgets).
- **Prompt caching** — `prompt_cache_key` on `/v1/chat/completions` and `/v1/responses` passes through as a cache-routing hint to the models that support it (and is dropped elsewhere); `cache_control` blocks on `/v1/messages` are accepted so Anthropic SDK code runs unchanged, but explicit breakpoints are not applied yet — caching works from the prompt prefix. `usage.prompt_tokens_details.cached_tokens` is now always present, and the [Usage API](/docs/usage-api) reports cache reads and writes.

### Payments, evals, contacts and telephony

- **Customer payment links** — agents can send payment links on calls and WhatsApp that are paid into your own connected Razorpay account. Track them under [`/api/v1/payment-requests`](/docs/payment-requests) and subscribe to the `payment_request.*` [webhook events](/docs/webhooks).
- **Evals** — an `llm_judge` assertion, a pass-rate gate for CI, and creating an eval case from a real call. See [Voice evals](/docs/voice-evals).
- **WhatsApp identities on contacts** — contacts carry `whatsapp_bsuid`, `whatsapp_parent_bsuid` and `whatsapp_username`, for WhatsApp users who hide their phone number. A BSUID is unique per account like a phone number or email. See [Contacts](/docs/crm-contacts).
- **TRAI readiness for outbound campaigns** — a readiness check for AI outbound calling under the TCCCPR amendment. See [Voice campaigns](/docs/voice-campaigns#trai-readiness-for-ai-calls-india).

### Newly documented

- **[Contacts](/docs/crm-contacts)** (`/api/v1/contacts`, plus each contact's memory), the **[handoff queue](/docs/handoffs)** (`/api/v1/handoffs`), **[integrations and sheet automations](/docs/integrations)** (`/api/v1/integrations`, `/api/v1/sheet-automations`), **[payment requests](/docs/payment-requests)**, and `POST /api/v1/crm/csv/inspect` were already callable with an API key and now have reference pages.
- **[Webhooks](/docs/webhooks)** — the delivery body, the `X-CallMissed-Event` / `X-CallMissed-Delivery` headers, the retry schedule, and the `call.amd_detected` event.
- **[Idempotency](/docs/idempotency)** and **[Errors](/docs/errors)** — rewritten to match exactly what the API does.

## September 2026

### Account MCP server — run your account from an AI assistant

- **Account MCP server** — connect Claude, ChatGPT or any MCP client to `https://api.callmissed.com/api/v1/mcp`, sign in with your CallMissed account or an API key, and act on your account through 331 tools: voice agents, calls and campaigns, the inbox, CRM, support desk, WhatsApp, email and more. Every tool is gated by the key's scopes, spending tools are marked, and destructive ones ask first. See [Account MCP](/docs/agent-tools-mcp).
- **Multilingual voice agents** — voice tiers and custom agents accept an auto-detect language, so one agent can answer in the caller's language. See [Voice tiers](/docs/voice-tiers).

### New model — GPT-6.1 Sol

- **`gpt-6.1-sol`** — OpenAI's GPT-6.1 Sol: near-Astra performance for complex coding, computer use and professional work at a lower cost. 1.05M context, 128K output, text + image input, reasoning and tool calling. $2.083 in / $10.42 out per 1M tokens ($0.1042 cached input).
- **Long prompts** — requests with more than 272K input tokens are billed at 2× input and cached-input rates and 1.5× output for the whole request ($4.167 in / $0.2083 cached / $15.63 out per 1M).
- **`reasoning_effort`** — accepts `low`, `medium` (default), `high` and `xhigh`. The model always reasons, so `none` and `minimal` are sent as `low`, and `max` is sent as `xhigh`. Only the default temperature is supported, and `max_tokens` must be at least 3. See [API speed](/docs/api-speed).
- Paid plans only. See [Models](/docs/models).

### New models — GPT-6 Sol and GPT-6 Luna

- **`gpt-6-sol`** — OpenAI's GPT-6 frontier reasoning model for enterprise agents, coding and complex knowledge work; succeeds `gpt-5.6-sol`. 1.05M context, 128K output, text + image input, reasoning and tool calling. $2.083 in / $10.42 out per 1M tokens ($0.2083 cached input).
- **`gpt-6-luna`** — the efficient GPT-6 model for high-volume, cost-sensitive workloads; succeeds `gpt-5.6-luna`. Same 1.05M context, vision, reasoning and tools. $0.1042 in / $0.5208 out per 1M tokens ($0.01042 cached input).
- **Long prompts** — requests with more than 272K input tokens are billed at 2× input and cached-input rates and 1.5× output for the whole request.
- **`reasoning_effort`** — both accept `none`, `low`, `medium`, `high` and `xhigh` (`minimal` is sent as `low`). When a request includes `tools`, reasoning is set to `none` automatically. `max_tokens` must be at least 3. See [API speed](/docs/api-speed).
- Paid plans only. See [Models](/docs/models).

## August 2026

### Managed Voice Agent — speech-to-speech over one WebSocket

- **Managed Voice Agent** — a full speech-to-speech pipeline behind a single WebSocket. Stream microphone audio in, get synthesized speech and conversation events back; speech recognition, the language model, text-to-speech, turn-taking and interruption handling are all run and tuned for you. No WebRTC and no client SDK. Two protocols on the same host and the same engine: `wss://api.callmissed.com/v2/voice/agent` (CallMissed-native) and `wss://api.callmissed.com/v1/agent/converse` (**Deepgram Voice Agent compatible** — an existing Deepgram integration can repoint its URL and work unchanged). Supports in-call tool calling, live model/prompt/voice updates, and sessions up to 2 hours. See [Managed Voice Agent](/docs/managed-voice-agent).
- **Voice model catalogue** — `GET /api/v1/voice/models` lists every selectable speech-to-text, language and text-to-speech model with a **measured** latency verdict (`eligible`, `too_slow`, `unsupported`, `unmeasured`) plus its p50, sample count and budget. Any eligible combination is valid. Models too slow to hold a conversation are not offered — they are listed with the reason, rather than silently disappearing or being offered with a caveat. Verdicts come from real production traffic, not vendor claims, and a model with too few samples reads `unmeasured` rather than being assumed fast.

### Embeddings, usage API, CRM, support desk and voice-agent operations

- **Embeddings** — `POST /v1/embeddings`, OpenAI-compatible. `text-embedding-3-small` (1536 dims, $0.02083 / 1M input tokens) and `text-embedding-3-large` (3072 dims, $0.1354 / 1M). Batches of up to 128 inputs, optional `dimensions` shortening and `base64` output. Both are free-plan callable, taking the **free tier to 27 models across five categories**. Gated by the key's `llm` permission. See [Embeddings](/docs/embeddings).
- **Usage API** — `GET /v1/usage/summary`, `/logs` and `/logs.csv` return your own metering rows for the last 90 days, filterable by service, model, key, `session_id` and `trace_id`. Scope `usage:read`. See [Usage API](/docs/usage-api).
- **Gateway tooling** — server-side [prompt management](/docs/gateway-prompts) with versions, labels, presets and free rendering; [response cache](/docs/gateway-cache) stats and purge; and [bring your own provider key](/docs/provider-keys) with liveness verification and a write-only secret.
- **CRM** — [companies](/docs/crm-companies), [notes and tasks](/docs/crm-notes-tasks), [deals and pipelines](/docs/crm-deals), [custom fields and saved views](/docs/crm-custom-fields), [search, bulk and CSV](/docs/crm-import-export), and [lead scoring with a unified timeline](/docs/crm-lead-scores).
- **Support desk** — [tickets](/docs/support-tickets) with server-managed lifecycle stamps, [SLA policies](/docs/support-sla) with business hours and live breach reporting, [macros, tags and routing rules](/docs/support-ops) with a dry-run evaluator, and [CSAT/NPS surveys](/docs/csat) and a hosted page where customers answer.
- **Voice-agent operations** — [eval suites](/docs/voice-evals) (up to 50 cases per run, credit-charged), [A/B experiments](/docs/voice-experiments) with deterministic assignment, and [agent squads](/docs/voice-squads) with handoff simulation and credit-charged agent drafting.
- **WhatsApp** — [Flows](/docs/whatsapp-flows) (create, publish, read submissions) and [catalog orders](/docs/whatsapp-orders).

### New models — Cartesia Ink STT

- **`ink-whisper`** — Cartesia's fastest and most affordable STT at $0.1875 / hour, across **100 languages** including Hindi, Urdu and Tamil. Better accuracy than baseline Whisper, and dynamic chunking that cuts hallucination during pauses and silence. Works for both file transcription and voice sessions. See [Speech to Text](/docs/speech-to-text#cartesia-ink-models).
- **`ink-2`** — Cartesia's top-ranked STT for voice agents at $0.5625 / hour: 8% WER on AppTek's 14-accent call-centre benchmark, against 10% for Deepgram Flux and 12% for ElevenLabs. Self-detects turns, so no separate turn detector is needed. Two limits: it is **English only**, and it is **voice-session only** — the file transcription endpoint returns `400` and points you to `ink-whisper`. See [Speech to Text](/docs/speech-to-text#cartesia-ink-models).

### New models — conversational Indic LLM, Saaras V4 STT, Flux TTS

- **`sarvam-105b-conversations`** — 105B MoE tuned for conversation and voice. 128K context, tool calling, streaming, hybrid thinking. Free-tier, same $0.1563 in / $0.625 out per 1M as `sarvam-105b`. See [Indic Models](/docs/models-indic).
- **`saaras:v4`** — Sarvam STT with five output modes (transcribe, translate, verbatim, transliterate, code-mix) across 24 languages. Free-tier at $0.3125 / hour. See [Speech to Text](/docs/speech-to-text).
- **Deepgram Flux TTS** — streaming-first TTS built for voice agents: turn-based synthesis with prosody carried across turns. 36 English voices including `priya` (Indian-accented English, the default). English only, no expressive controls. Available only through the managed Voice Agent (`tts_engine: "flux"`), billed inside the per-minute voice rate. See [Voices](/docs/tts-voices).
- **Free tier** — now 27 models (11 LLM, 4 STT, 4 TTS, 6 image, 2 embedding).

### Model catalog update — retired models

- **Retired LLM IDs** — the following model IDs are no longer served: `openai/gpt-5.4-pro`, `openai/gpt-5.4`, `openai/gpt-5.4-mini`, `openai/gpt-5.4-nano`, `anthropic/claude-opus-4.6`, `anthropic/claude-sonnet-4.6`, `anthropic/claude-haiku-4.5`, `x-ai/grok-4.20`, `qwen/qwen3.5-plus`, `qwen/qwen3.5-flash`, `mistralai/mistral-small-2603`, and the `auto` auto-router.
- **Migration** — use the first-party flagships (`gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`, `gpt-5.5`, `grok-4.3`) or the direct-routed free tier (`kimi-k2.6`, `kimi-k2.7-code`, `glm-5.2`, `gpt-oss-120b`, `mistral-small-3.1`). See [Models](/docs/models).
- **Free tier** — now 24 models (11 LLM). The `auto` free auto-router is retired; pick a free model explicitly.
- **Endpoints unchanged** — `POST /v1/chat/completions` and the Anthropic-compatible `POST /v1/messages` both continue to work and accept every current catalog ID.

## June 2026

### v1.6.0 — WebRTC Voice, Image Generation & Web Search

- **WebRTC voice sessions** — `POST /v1/voice/sessions` returns a connection token + URL; CallMissed handles the STT→LLM→TTS pipeline. List, fetch, fetch transcript (`json | txt | srt`), and end sessions under `/v1/voice/sessions`. The legacy `/ws/voice-agent` WebSocket still works for backward compatibility. See [Voice Session API](/docs/voice-sessions-api).
- **Image Generation API** — `POST /v1/images/generations` (OpenAI-compatible). Free models include `flux-2-klein-9b`, `flux-2-dev`, `lucid-origin`, `phoenix-1.0`, `sdxl-lightning`, `dreamshaper-8-lcm`; paid models include `flux-2-pro`, `flux-1.1-pro`, `nano-banana-2`, and `nano-banana-pro`. See [Image Generation](/docs/image-generation).
- **Web Search API** — `POST /v1/search` defaults to Serper web search (Exa, Firecrawl also available); flat 1 credit per query. See [Web Search](/docs/web-search).
- **Knowledge RAG** — vector knowledge sources at `/api/v1/knowledge/sources` (ingest text, URL, or PDF; chunked + embedded) with semantic search at `POST /api/v1/knowledge/search`.

## May 2026

### v1.5.0 — First-Party Models, Account Security & WhatsApp Platform

- **First-party models** — deployments callable by bare ID: `gpt-4o`, `gpt-4.1`, `gpt-5-mini`, `grok-4.3`, `DeepSeek-V4-Pro`, `DeepSeek-V4-Flash`, plus first-party STT (`whisper`, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe`, `gpt-4o-transcribe-diarize`) and TTS (`gpt-4o-mini-tts`).
- **More STT/TTS** — `whisper-large-v3-turbo` (99 langs),  `nova-3` (diarization), `aura-2-en` / `aura-2-es`, and `melotts` — all free-tier.
- **Kimi K2.6** — `kimi-k2.6` added to the direct-routed free tier alongside `kimi-k2.5`.
- **TOTP 2FA & passkeys** — two-factor auth (authenticator apps + backup codes) and passkeys for dashboard sign-in. Active sessions can be reviewed and revoked from the dashboard.
- **WhatsApp platform** — Embedded Signup onboarding, message templates, broadcast campaigns, and delivery analytics under `/api/v1/whatsapp/*`. See [WhatsApp API](/docs/whatsapp-api).
- **Billing surfaces** — coupon redemption, downloadable PDF invoices, and a credit ledger broken down by transaction type, all in the dashboard.
- **Audit log** — a sensitive-action audit feed in the dashboard.

## April 2026

### v1.4.0 — Anthropic API Compatibility & Audio Translation

- **Anthropic Messages API** — New `POST /v1/messages` endpoint. Use the Anthropic SDK with CallMissed by changing only the `base_url`. Full streaming support with Anthropic SSE lifecycle (`message_start`, `content_block_delta`, `message_stop`).
- **Dual auth headers** — Anthropic endpoint accepts both `x-api-key` and `Authorization: Bearer` headers
- **Model aliasing** — A bare model name on the Anthropic endpoint resolves against the CallMissed catalog
- **Audio Translation** — New `POST /v1/audio/translations` endpoint. Translate audio in 24 languages to English text. OpenAI SDK compatible (`client.audio.translations.create()`)
- **Token counting** — `POST /v1/messages/count_tokens` for input token estimation
- **Anthropic rate limit headers** — `anthropic-ratelimit-requests-limit`, `anthropic-ratelimit-requests-remaining`, etc.

### v1.3.0 — Voice Agent & Ultra-Low-Latency Pipeline

- **Voice Agent WebSocket** — Real-time STT→LLM→TTS pipeline over `/ws/voice-agent`. PCM audio in, streaming MP3 out. LLM and TTS run concurrently for minimum latency.
- **PCM AudioWorklet capture** — Raw PCM s16le at 16kHz, no container overhead
- **Streaming MP3 playback** — MediaSource API appends and plays chunks as they arrive
- **Profile management** — Save and update user profile from the dashboard

### v1.2.0 — Security, Google OAuth & Plan Enforcement

- **Sign in with Google** — Google sign-in for the dashboard. Auto-creates the organisation and user, and links to an existing account by email.
- **OTP Authentication** — Email-based OTP for passwordless login and password reset
- **Plan limit enforcement** — Server-side usage caps per plan tier (free/starter/pro/enterprise). API returns `429 quota_exceeded` when limits reached. Usage headers (`X-RateLimit-*`, `X-Usage-Warning`) on every response.
- **Per-API-key rate limiting** — a requests-per-minute limit on every key (today set by plan; see [Rate limits](/docs/rate-limits))
- **Model catalog update** — OpenAI gpt-5.4 family, Anthropic Claude 4.6, Google Gemini 3.1, xAI Grok 4.20, Qwen 3.5, Mistral Small
- **Knowledge Base file upload** — Upload PDF, DOCX, TXT files (max 20 MB) with auto text extraction
- **Bot deployment verification** — Verify WhatsApp/Twilio channel connectivity from the dashboard
- **Settings verification** — Verify WhatsApp, Twilio, and Indic LLM API connectivity

### v1.1.0 — Platform Playground

- **Playground rebuild** — LLM (streaming + non-streaming), STT (file upload + mic), TTS (37 voices across 11 Indian languages), Voice Agent demo

### v1.0.0 — Initial Release

- **Chat Completion API** — OpenAI-compatible endpoint with streaming, tool calls, and function calling
- **Speech to Text** — `saaras:v3` with 22 Indic language support
- **Text to Speech** — `bulbul:v3` (37 voices across 11 Indian languages)
- **WhatsApp Bot** — Full WhatsApp Business API integration
- **Voice Calling** — Twilio-based inbound voice with WebSocket streaming
- **Multi-tenant** — Complete tenant isolation with role-based access
- **API Keys** — Scoped API keys with usage tracking
- **Webhook Delivery** — Outbound webhooks with retry and HMAC signing
- **Analytics Dashboard** — Real-time conversation and usage analytics
- **Model catalog** — LLM, STT, TTS and image models from one OpenAI-compatible endpoint
