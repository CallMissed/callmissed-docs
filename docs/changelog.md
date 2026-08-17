---
title: "Changelog"
description: "Latest updates, new features, and improvements to the CallMissed API."
slug: "changelog"
breadcrumb: "Resources"
---

# Changelog

Latest updates, new features, and improvements to the CallMissed API.

## August 2026

### Embeddings, usage API, CRM, support desk and voice-agent operations

- **Embeddings** — `POST /v1/embeddings`, OpenAI-compatible. `text-embedding-3-small` (1536 dims, $0.02 / 1M input tokens) and `text-embedding-3-large` (3072 dims, $0.13 / 1M). Batches of up to 128 inputs, optional `dimensions` shortening and `base64` output. Both are free-plan callable, taking the **free tier to 27 models across five categories**. Gated by the key's `llm` permission. See [Embeddings](/docs/embeddings).
- **Usage API** — `GET /v1/usage/summary`, `/logs` and `/logs.csv` return your own metering rows for the last 90 days, filterable by service, model, key, `session_id` and `trace_id`. Scope `usage:read`. See [Usage API](/docs/usage-api).
- **Gateway tooling** — server-side [prompt management](/docs/gateway-prompts) with versions, labels, presets and free rendering; [response cache](/docs/gateway-cache) stats and purge; and [bring your own provider key](/docs/provider-keys) with liveness verification and a write-only secret.
- **CRM** — [companies](/docs/crm-companies), [notes and tasks](/docs/crm-notes-tasks), [deals and pipelines](/docs/crm-deals), [custom fields and saved views](/docs/crm-custom-fields), [search, bulk and CSV](/docs/crm-import-export), and [lead scoring with a unified timeline](/docs/crm-lead-scores).
- **Support desk** — [tickets](/docs/support-tickets) with server-managed lifecycle stamps, [SLA policies](/docs/support-sla) with business hours and live breach reporting, [macros, tags and routing rules](/docs/support-ops) with a dry-run evaluator, and [CSAT/NPS surveys](/docs/csat) with a public, token-authenticated response surface.
- **Voice-agent operations** — [eval suites](/docs/voice-evals) (up to 50 cases per run, credit-charged), [A/B experiments](/docs/voice-experiments) with deterministic assignment, and [agent squads](/docs/voice-squads) with handoff simulation and credit-charged agent drafting.
- **WhatsApp** — [Flows](/docs/whatsapp-flows) (create, publish, read submissions) and [catalog orders](/docs/whatsapp-orders).

### New models — conversational Indic LLM, Saaras V4 STT, Flux TTS

- **`sarvam-105b-conversations`** — 105B MoE tuned for conversation and voice. 128K context, tool calling, streaming, hybrid thinking. Free-tier, same $0.35 in / $0.35 out per 1M as `sarvam-105b`. See [Indic Models](/docs/models-indic).
- **`saaras:v4`** — Sarvam STT with five output modes (transcribe, translate, verbatim, transliterate, code-mix) across 24 languages. Free-tier at $0.30 / hour. See [Speech to Text](/docs/speech-to-text).
- **Deepgram Flux TTS** — streaming-first TTS built for voice agents: turn-based synthesis with prosody carried across turns. 11 English voices including `priya` (Indian-accented English, the default). English only, no expressive controls. Available only through the managed Voice Agent (`tts_engine: "flux"`), billed inside the per-minute voice rate. See [Voices](/docs/tts-voices).
- **Free tier** — now 27 models (11 LLM, 4 STT, 4 TTS, 6 image, 2 embedding).

### Model catalog update — retired models

- **Retired LLM IDs** — the following model IDs are no longer served: `openai/gpt-5.4-pro`, `openai/gpt-5.4`, `openai/gpt-5.4-mini`, `openai/gpt-5.4-nano`, `anthropic/claude-opus-4.6`, `anthropic/claude-sonnet-4.6`, `anthropic/claude-haiku-4.5`, `x-ai/grok-4.20`, `qwen/qwen3.5-plus`, `qwen/qwen3.5-flash`, `mistralai/mistral-small-2603`, and the `auto` auto-router.
- **Migration** — use the first-party flagships (`gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`, `gpt-5.5`, `grok-4.3`) or the direct-routed free tier (`kimi-k2.6`, `kimi-k2.7-code`, `glm-5.2`, `gpt-oss-120b`, `mistral-small-3.1`). See [Models](/docs/models).
- **Free tier** — now 24 models (11 LLM). The `auto` free auto-router is retired; pick a free model explicitly.
- **Endpoints unchanged** — `POST /v1/chat/completions` and the Anthropic-compatible `POST /v1/messages` both continue to work and accept every current catalog ID.

## June 2026

### v1.6.0 — LiveKit Voice, Image Generation & Web Search

- **LiveKit voice sessions** — `POST /v1/voice/sessions` returns a LiveKit token + URL; CallMissed handles the STT→LLM→TTS pipeline. List, fetch, fetch transcript (`json | txt | srt`), and end sessions under `/v1/voice/sessions`. A public, capped browser demo lives at `POST /v1/voice/demo`. The legacy `/ws/voice-agent` WebSocket still works for backward compatibility. See [Voice Session API](/docs/voice-sessions-api).
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
- **Per-API-key rate limiting** — 60 req/min per key
- **Security hardening** — across the API surface
- **Model catalog update** — OpenAI gpt-5.4 family, Anthropic Claude 4.6, Google Gemini 3.1, xAI Grok 4.20, Qwen 3.5, Mistral Small
- **Knowledge Base file upload** — Upload PDF, DOCX, TXT files (max 20 MB) with auto text extraction
- **Bot deployment verification** — Verify WhatsApp/Twilio channel connectivity from the dashboard
- **Settings verification** — Verify WhatsApp, Twilio, and Indic LLM API connectivity
- **Contact form** — Public `POST /api/v1/contact` endpoint with email notifications
- **Dual-domain support** — `.com` and `.in` TLDs for all apps

### v1.1.0 — Platform Playground & SEO

- **Playground rebuild** — LLM (streaming + non-streaming), STT (file upload + mic), TTS (37 voices across 11 Indian languages), Voice Agent demo
- **Call Analytics API** — Upload audio files for batch STT with diarization and LLM-powered analysis
- **SEO pages** — 6 product pages, legal pages, company pages on the landing site
- **Sitemaps** — All 4 apps have sitemap.ts for SEO

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
