---
title: "Changelog"
description: "Latest updates, new features, and improvements to the CallMissed API."
slug: "changelog"
breadcrumb: "Resources"
---

# Changelog

Latest updates, new features, and improvements to the CallMissed API.

## October 2026

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

### Email domains — neutral `502` reason

- The `502` returned by the email domain routes when provisioning or verification is temporarily unavailable now carries `reason: "provider_unavailable"`. Deployments still being upgraded may return the older `acs_unavailable` for the same condition; status, body shape and retry advice are identical, so match both strings. See [Email limits](/docs/email-limits).

### Voice session webhooks

- `voice_session.ended` and `voice_session.failed` now fire exactly once per session, on every way a session can end (the agent hanging up, `DELETE`, the session timing out), and only after the end has been saved. If two end signals arrive together, the first one decides `end_reason` and `duration_seconds`.
- A session that ends because the agent hit an error (`end_reason: "agent_error"`) now has status `failed` and fires `voice_session.failed` on every path; it previously fired `voice_session.ended` for some sessions. See [Webhooks](/docs/webhooks#voice-session-end-reasons).

### Speech-to-text error codes

- `/v1/audio/transcriptions` and `/v1/audio/translations` now return `400 invalid_request` for a problem with the request (unsupported audio, a streaming-only or unknown model), `429 rate_limit_exceeded` when rate limited, and `502 service_unavailable` when transcription is temporarily unavailable, the same code `/v1/audio/speech` uses. See [Speech-to-Text](/docs/speech-to-text#errors).

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

- **`sarvam-105b-conversations`** — 105B MoE tuned for conversation and voice. 128K context, tool calling, streaming, hybrid thinking. Free-tier, same $0.3646 in / $0.3646 out per 1M as `sarvam-105b`. See [Indic Models](/docs/models-indic).
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
