---
title: "Welcome to CallMissed API"
description: "CallMissed provides AI-powered communication APIs to deploy WhatsApp chatbots and voice call agents for your business."
slug: "introduction"
breadcrumb: "Getting Started"
---

# Welcome to CallMissed API

CallMissed provides AI-powered communication APIs to deploy WhatsApp chatbots and voice call agents for your business.

:::cards
/docs/quickstart | Quickstart | play | Make your first API call in under a minute
/docs/models | Models | boxes | 56 models — Indic STT/TTS, frontier LLMs, image gen
/docs/voice-agent | Voice Agent | phone | LiveKit WebRTC agents with Sarvam pipeline
:::

## Overview

CallMissed is an AI Communication Infrastructure platform. Use our APIs to:

- Deploy **WhatsApp chatbots** with custom knowledge bases
- Build **AI voice call agents** for inbound calls
- Create **Smart IVR** flows with AI escalation
- Access **300+ LLM models** from every major creator (OpenAI, Anthropic, Google, xAI, and more) — a curated catalog plus full OpenRouter passthrough
- Use **OpenAI-compatible APIs** — same SDK, just change the base URL
- Manage **multi-tenant** deployments for your customers

## Base URL

All API requests go to:

```
https://api.callmissed.com
```

For local development:

```
http://localhost:8000
```

## Key Features

- **Indic Models** — STT, TTS, and LLM purpose-built for 22 Indic languages
- **Multi-tenant** — full data isolation between tenants
- **Real-time** — WebSocket voice streaming with ultra-low-latency STT→LLM→TTS pipeline
- **Webhooks** — WhatsApp Business API and Twilio integration
- **OpenAI-compatible** — use the OpenAI SDK with your `cm_` API key
- **Request Logging** — per-key request logs with latency, model, cost, and error tracking