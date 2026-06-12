---
title: "Glossary"
description: "Definitions for the core CallMissed concepts and terminology used throughout these docs."
slug: "glossary"
breadcrumb: "Resources"
---

# Glossary

Definitions for the core CallMissed concepts and terminology used throughout these docs.

## Platform

| Term | Definition |
| --- | --- |
| **Tenant** | Your organization. All users, bots, keys, and data are isolated per tenant. |
| **Bot** | A configured AI agent (WhatsApp, inbound/outbound call, IVR) with a system prompt and optional knowledge base. |
| **Channel** | The surface a bot runs on — WhatsApp or voice (Twilio / LiveKit). |
| **Conversation** | A thread of messages between an end user and a bot on a channel. |
| **API Key** | A secret prefixed `cm_` used for server-to-server auth, with scopes, domain locks, and per-key limits. |
| **Permission** | A service an API key may call — `llm`, `stt`, `tts`, `search`, `image`, or `*`. Enforced on the inference endpoints; default `*`. |
| **Scope** | A platform resource an API key may access — e.g. `bots:read`, `conversations:write`, `knowledge:read`, `webhooks:write`, `whatsapp:write`. Defaults to empty (no resource access). |
| **Webhook** | An HTTPS endpoint CallMissed calls on events; payloads are HMAC-SHA256 signed. |
| **Idempotency-Key** | A header that makes a mutating request safely retryable — replays return the original result. |

## Billing

| Term | Definition |
| --- | --- |
| **Credit** | The universal billing unit. **1 credit = ₹1.** Every API call deducts credits based on usage. |
| **Plan** | Your subscription tier — free, starter, pro, or enterprise — which sets limits and model access. |
| **Budget cap** | An optional monthly credit limit; requests over the cap are rejected with `budget_exceeded`. |
| **Credit pack** | A purchasable bundle of credits for top-ups. |

## AI Services

| Term | Definition |
| --- | --- |
| **LLM** | Large Language Model — powers chat completions and the Anthropic Messages API. |
| **STT** | Speech-to-Text — transcription, translation, real-time, and batch. |
| **TTS** | Text-to-Speech — voice synthesis. |
| **RAG** | Retrieval-Augmented Generation — semantic search over ingested knowledge passed as context. |
| **Diarization** | Labeling who spoke when in a transcript (speaker separation). |
| **Voice Agent** | A real-time STT→LLM→TTS pipeline over WebRTC (LiveKit). |
| **OpenAI-compatible** | Our `/v1` endpoints accept the same request shapes as the OpenAI API — change only the base URL and key. |