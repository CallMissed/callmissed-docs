---
title: "WhatsApp Bot"
description: "Run an AI agent on your WhatsApp Business number: connect a WABA, auto-reply to inbound messages, and send, template, campaign and call through one API."
slug: "whatsapp"
breadcrumb: "WhatsApp"
---

# WhatsApp Bot

Run an AI agent on your WhatsApp Business number: connect a WABA, auto-reply to inbound messages, and send, template, campaign and call through one API.

CallMissed connects your **WhatsApp Business Account (WABA)** to an AI agent. Inbound messages land on a webhook, get stored as a conversation, and are answered by your bot's system prompt plus knowledge base. Everything the agent can do by itself you can also do programmatically over the REST API: send any WhatsApp message type, manage approved templates, run bulk template campaigns, place voice calls on WhatsApp, and read delivery analytics.

## What you get

| Capability | Where |
|---|---|
| AI auto-reply to inbound WhatsApp messages | Automatic once a bot is linked to a number |
| Send text, template, media, interactive, location, reaction, contact cards | [Sending Messages](/docs/whatsapp-messages) |
| Create, list, delete and sync message templates | [Message Templates](/docs/whatsapp-templates) |
| Bulk template sends with per-recipient variables | [Campaigns](/docs/whatsapp-campaigns) |
| Voice calls over WhatsApp, answered by the same agent | [Calling](/docs/whatsapp-calling) |
| Connected accounts, numbers, delivery funnel, cost | [WhatsApp API](/docs/whatsapp-api) |

## Two ways in

**Dashboard.** Connect a number under **Settings → Integrations → WhatsApp**, create a bot, link the two, and the agent starts replying. Nothing to build.

**API.** Everything the dashboard does is an endpoint under `https://api.callmissed.com/api/v1/whatsapp`, authenticated with a `cm_` API key. Use it to embed WhatsApp into your own product, run campaigns from your backend, or ship a custom inbox.

The two share one data model. A number connected in the dashboard is immediately sendable from the API, and a message sent over the API appears in the dashboard conversation thread.

## Message flow

Meta posts every inbound event to a single CallMissed endpoint. You never configure that endpoint yourself: connecting a number subscribes the CallMissed app to your WABA's webhooks.

:::flow
icon:user | Customer | Sends a WhatsApp message to your business number
icon:gateway | Meta | POSTs the event to `/api/v1/webhooks/whatsapp` with an `X-Hub-Signature-256` header
icon:gateway | CallMissed | Verifies the signature, archives the raw event, and acknowledges with `200` immediately
icon:llm | Agent | Routes the number to its linked bot, stores the message, marks it read, and runs the LLM with the conversation history plus knowledge base
icon:done | Customer | Receives the reply through the WhatsApp Cloud API
:::

The acknowledgement is sent before the LLM runs, so a slow model never causes Meta to retry the event.

### When the bot replies

An inbound message is **always stored**. The agent only answers when all of these hold:

1. The number is **explicitly linked** to a bot (`POST /phone_numbers/{phone_id}/link-bot`). An unlinked number stores messages and stays silent.
2. The bot has a non-empty `system_prompt`.
3. AI auto-reply is on for the number (`ai_autoreply_enabled`, togglable per number).
4. AI auto-reply is on for that conversation (an agent can take over a single thread from the inbox without pausing the whole number).
5. The message is **text**, or an **image** when the bot's model supports vision. Audio, stickers, reactions, interactive replies and other types are stored but not auto-answered.

## Quickstart

Send your first message and get an AI reply in five calls. You need an API key with the `whatsapp:read`, `whatsapp:write` and `whatsapp:send` scopes, plus a connected number ([Business Setup](/docs/whatsapp-setup)).

:::steps
## Create the bot

```bash
curl -X POST https://api.callmissed.com/api/v1/bots \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme Support",
    "type": "whatsapp",
    "system_prompt": "You are the support agent for Acme, an Indian D2C coffee brand. Answer in under 60 words. If asked about an order, ask for the order id first. Never invent a delivery date."
  }'
```

The response carries the bot `id`. Keep it.

```json
{
  "id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
  "tenant_id": "7f2e1d0c-9b8a-4756-b3c2-1a0f9e8d7c6b",
  "name": "Acme Support",
  "type": "whatsapp",
  "system_prompt": "You are the support agent for Acme...",
  "is_active": true
}
```

## Find your connected number

```bash
curl https://api.callmissed.com/api/v1/whatsapp/phone_numbers \
  -H "Authorization: Bearer cm_your_api_key"
```

Take `id` (CallMissed's `phone_id`) and `phone_number_id` (Meta's id) from the number you want to use.

## Link the bot to the number

Without this the bot never auto-replies.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/phone_numbers/9c2b7e30-1d8a-4c5f-9b3d-2f4a6e8b1c2d/link-bot \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "bot_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d" }'
```

## Subscribe to inbound messages

Register your own HTTPS endpoint so every customer message is pushed to you.

```bash
curl -X POST https://api.callmissed.com/api/v1/webhooks \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-app.example.com/hooks/callmissed",
    "events": ["message.received"]
  }'
```

See [Inbound events](/docs/whatsapp-api#inbound-events-you-receive) for the exact payload and the signature header.

## Send a message

Free-form sends need an open 24-hour window (the customer messaged you within the last 24 hours). Outside it, send a template instead.

```bash
curl -X POST https://api.callmissed.com/api/v1/whatsapp/messages \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "1234567890",
    "to": "+919000000000",
    "text": "Thanks for reaching out. How can we help?"
  }'
```

```json
{
  "wamid": "wamid.HBgMOTE5MDAwMDAwMDAwFQIAERgSMkE5N0Y4RDcxMkYzQTJEMQA=",
  "contacts": [{ "input": "+919000000000", "wa_id": "919000000000" }]
}
```
:::

Now message your business number from a personal WhatsApp account. The message appears in the dashboard conversation, hits your webhook, and the agent answers.

## The 24-hour window

WhatsApp only allows free-form messages (text, media, interactive, location) inside the 24-hour customer service window that opens each time the user messages you. Outside it you must send an **approved template**.

A closed-window send returns `422` with an actionable message:

```json
{
  "detail": "The 24-hour customer service window is closed. Send a template message instead, or wait for the user to message you."
}
```

Templates are never window-limited, which is why order updates, reminders and one-time codes are all template sends. See [Message Templates](/docs/whatsapp-templates).

## Bot configuration

A bot is channel-agnostic. What makes it a WhatsApp agent is the `link-bot` binding to a connected number, not its `config`. Credentials live on the connected number (encrypted at rest), so a linked bot needs no WhatsApp keys of its own.

```json
{
  "name": "Acme Support",
  "type": "whatsapp",
  "system_prompt": "You are the support agent for Acme Coffee.",
  "config": {
    "model": "clarifai/kimi-k2.5",
    "language": "en"
  }
}
```

Add product facts, FAQs and policies as [knowledge base](/docs/knowledge) entries. The agent calls a `search_knowledge_base` tool on demand and grounds its answer in what it retrieves.

## Where to next

:::cards
/docs/whatsapp-setup | Business Setup | Settings | Connect a WABA and register a number, with or without Embedded Signup.
/docs/whatsapp-api | WhatsApp API | Webhook | Auth, scopes, error shapes, accounts, numbers, analytics and inbound events.
/docs/whatsapp-messages | Sending Messages | Send | Every send endpoint plus media upload and download.
/docs/whatsapp-templates | Message Templates | FileText | Create, list, delete and sync approved templates.
/docs/whatsapp-campaigns | Campaigns | Megaphone | Bulk template sends with per-recipient variables.
/docs/whatsapp-calling | Calling | Phone | Voice calls over WhatsApp, answered by the same agent.
:::
