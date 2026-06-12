---
title: "Changelog"
description: "Latest updates, new features, and improvements to the CallMissed API."
slug: "changelog"
breadcrumb: "Getting Started"
---

# Changelog

Latest updates, new features, and improvements to the CallMissed API.

## June 2026

### v1.6.0 — LiveKit Voice, Image Generation & Web Search

- **LiveKit voice sessions** — `POST /v1/voice/sessions` returns a LiveKit token + URL; the server-side agent runs the STT→LLM→TTS pipeline. List, fetch, fetch transcript (`json | txt | srt`), and end sessions under `/v1/voice/sessions`. A public, capped browser demo lives at `POST /v1/voice/demo`. The legacy `/ws/voice-agent` WebSocket still works for backward compatibility. See [Voice Session API](/docs/voice-sessions-api).
- **Image Generation API** — `POST /v1/images/generations` (OpenAI-compatible). Free models include `flux-2-klein-9b`, `flux-2-dev`, `lucid-origin`, `phoenix-1.0`, `sdxl-lightning`, `dreamshaper-8-lcm`; paid models include `flux-2-pro`, `flux-1.1-pro`, `nano-banana-2`, `nano-banana-pro`, and `openai-gpt-image-2`. See [Image Generation](/docs/image-generation).
- **Web Search API** — `POST /v1/search` with Serper or Exa providers; 1 credit per query (Exa adds 0.1 credit per result above 10). See [Web Search](/docs/web-search).
- **Knowledge RAG** — vector knowledge sources at `/api/v1/knowledge/sources` (ingest text, URL, or PDF; chunked + embedded) with semantic search at `POST /api/v1/knowledge/search`.

## May 2026

### v1.5.0 — Azure Models, Account Security & WhatsApp Platform

- **Azure-hosted models** — first-party deployments callable by bare ID: `gpt-4o`, `gpt-4.1`, `gpt-5-mini`, `grok-4.3`, `DeepSeek-V4-Pro`, `DeepSeek-V4-Flash`, plus Azure STT (`whisper`, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe`, `gpt-4o-transcribe-diarize`) and TTS (`gpt-4o-mini-tts`).
- **More STT/TTS** — `whisper-large-v3-turbo` (99 langs), Deepgram `nova-3` (diarization), Deepgram `aura-2-en` / `aura-2-es`, and `melotts` — all free-tier.
- **Kimi K2.6** — `kimi-k2.6` added to the direct-routed free tier alongside `kimi-k2.5`.
- **TOTP 2FA & passkeys** — `/api/v1/auth/2fa/*` (authenticator apps + backup codes) and WebAuthn passkeys at `/api/v1/auth/passkey/*`. Active sessions are listable and revocable at `/api/v1/auth/sessions`.
- **WhatsApp platform** — Embedded Signup onboarding, message templates, broadcast campaigns, and delivery analytics under `/api/v1/whatsapp/*`. See [WhatsApp API](/docs/whatsapp-api).
- **Billing surfaces** — coupon redemption (`POST /api/v1/coupons/redeem`), downloadable invoices (`/api/v1/invoices/:invoice_number/pdf`), and a credit ledger with `summary_by_type` at `/api/v1/credits/transactions`.
- **Audit log** — sensitive-action audit feed at `/api/v1/audit/events`.

## April 2026

### v1.4.0 — Anthropic API Compatibility & Audio Translation

- **Anthropic Messages API** — New `POST /v1/messages` endpoint. Use the Anthropic SDK with CallMissed by changing only the `base_url`. Full streaming support with Anthropic SSE lifecycle (`message_start`, `content_block_delta`, `message_stop`).
- **Dual auth headers** — Anthropic endpoint accepts both `x-api-key` and `Authorization: Bearer` headers
- **Model aliasing** — Send `claude-sonnet-4.6` on the Anthropic endpoint and it auto-routes to `anthropic/claude-sonnet-4.6` in the frontier catalog
- **Audio Translation** — New `POST /v1/audio/translations` endpoint. Translate audio in 24 languages to English text. OpenAI SDK compatible (`client.audio.translations.create()`)
- **Token counting** — `POST /v1/messages/count_tokens` for input token estimation
- **Anthropic rate limit headers** — `anthropic-ratelimit-requests-limit`, `anthropic-ratelimit-requests-remaining`, etc.

### v1.3.0 — Voice Agent & Ultra-Low-Latency Pipeline

- **Voice Agent WebSocket** — Real-time STT→LLM→TTS pipeline over `/ws/voice-agent`. PCM audio in, streaming MP3 out. LLM and TTS run concurrently for minimum latency.
- **PCM AudioWorklet capture** — Raw PCM s16le at 16kHz, no container overhead
- **Streaming MP3 playback** — MediaSource API appends and plays chunks as they arrive
- **Profile management** — Save and update user profile from the dashboard

### v1.2.0 — Security, Google OAuth & Plan Enforcement

- **Google OAuth** — Sign in with Google via `POST /api/v1/auth/google`. Auto-creates tenant/user, links to existing accounts by email.
- **OTP Authentication** — Email-based OTP for passwordless login and password reset
- **Plan limit enforcement** — Server-side usage caps per plan tier (free/starter/pro/enterprise). API returns `429 quota_exceeded` when limits reached. Usage headers (`X-RateLimit-*`, `X-Usage-Warning`) on every response.
- **Per-API-key rate limiting** — 60 req/min per key on top of IP-based limits
- **Security hardening** — SSRF prevention on webhooks, WebSocket auth enforcement, input sanitization, rate limiter IP cleanup, error message sanitization
- **Model catalog update** — OpenAI gpt-5.4 family, Anthropic Claude 4.6, Google Gemini 3.1, xAI Grok 4.20, Qwen 3.5, Mistral Small
- **Knowledge Base file upload** — Upload PDF, DOCX, TXT files (max 20 MB) with auto text extraction
- **Bot deployment verification** — Verify WhatsApp/Twilio channel connectivity from the dashboard
- **Settings verification** — Verify WhatsApp, Twilio, and Indic LLM API connectivity
- **Contact form** — Public `POST /api/v1/contact` endpoint with email notifications
- **Dual-domain support** — `.com` and `.in` TLDs for all apps

### v1.1.0 — Platform Playground & SEO

- **Playground rebuild** — LLM (streaming + non-streaming), STT (file upload + mic), TTS (39 voices across 11 Indian languages), Voice Agent demo
- **Call Analytics API** — Upload audio files for batch STT with diarization and LLM-powered analysis
- **SEO pages** — 6 product pages, legal pages, company pages on the landing site
- **Sitemaps** — All 4 apps have sitemap.ts for SEO

### v1.0.0 — Initial Release

- **Chat Completion API** — OpenAI-compatible endpoint with streaming, tool calls, and function calling
- **Speech to Text** — `saaras:v3` with 22 Indic language support
- **Text to Speech** — `bulbul:v3` (39 voices across 11 Indian languages)
- **WhatsApp Bot** — Full WhatsApp Business API integration
- **Voice Calling** — Twilio-based inbound voice with WebSocket streaming
- **Multi-tenant** — Complete tenant isolation with role-based access
- **API Keys** — Scoped API keys with usage tracking
- **Webhook Delivery** — Outbound webhooks with retry and HMAC signing
- **Analytics Dashboard** — Real-time conversation and usage analytics
- **300+ LLM Models** — Access frontier models from OpenAI, Anthropic, Google, xAI, Qwen, and more