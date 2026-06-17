---
title: "How CallMissed Works"
description: "The architecture behind CallMissed — one OpenAI-compatible gateway that routes to the best provider for each model, billed in a single credit currency."
slug: "how-it-works"
breadcrumb: "Getting Started"
---

# How CallMissed Works

The architecture behind CallMissed — one OpenAI-compatible gateway that routes to the best provider for each model, billed in a single credit currency.

## The Gateway

CallMissed is a single, OpenAI-compatible gateway in front of many AI providers. You point the official OpenAI (or Anthropic) SDK at `https://api.callmissed.com/v1`, authenticate with a `cm_` key, and call chat, speech, image, and search — without integrating each provider yourself.

The same endpoint powers the dashboard playground and your production code, so behavior is identical everywhere.

:::flow
icon:app | Your app | One SDK, one base URL, one `cm_` key for every capability
icon:gateway | CallMissed gateway | Authenticate, enforce tenant isolation + rate limits, route by model id
icon:provider | Best provider | the best-fit backend for each model — chosen automatically
icon:db | Credits & logs | Deduct from one credit balance and record usage for every request
:::

## Provider Routing

We pick the upstream provider from the model id, so you never manage multiple SDKs or keys:

| Model id shape | Routed to |
| --- | --- |
| default fast tier | Kimi K2.5 Fast (~414 tok/s) |
| no slash (e.g. `sarvam-30b`, `saaras:v3`) | Indic LLM/STT/TTS |
| has a slash (e.g. `openai/gpt-5.4`, `anthropic/claude-sonnet-4.6`) | Frontier catalog |
| audio / image models | Audio/Image backends |

You get one API surface, one key, and one bill regardless of which provider ultimately serves the request.

## One Credit Currency

Every call — LLM tokens, STT minutes, TTS characters, an image, a web search — deducts **credits**. **1 credit = ₹1.** This removes per-provider pricing math: top up once, spend across every capability. See [Credits & Rate Limits](/docs/credits-rate-limits) for the per-service rates.

## Tenancy & Isolation

Your organization is a **tenant**. Every user, bot, API key, conversation, and log belongs to exactly one tenant, and every database query is scoped to it — data is never shared across tenants. Roles (owner / admin / agent) gate sensitive actions. See [Team & Tenants](/docs/team).

## Channels vs APIs

There are two ways to use CallMissed:

- **Direct APIs** — call `/v1/*` from your own app (chat, speech, image, search, voice sessions).
- **Channels** — let CallMissed run a bot end-to-end on [WhatsApp](/docs/whatsapp) or [voice calls](/docs/voice), handling the inbound webhook, the AI turn, and the reply.