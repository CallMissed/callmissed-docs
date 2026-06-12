---
title: "WhatsApp Bot"
description: "Deploy AI-powered WhatsApp chatbots with knowledge base support."
slug: "whatsapp"
breadcrumb: "Channels"
---

# WhatsApp Bot

Deploy AI-powered WhatsApp chatbots with knowledge base support.

## Setup

1. Create a bot with `type: "whatsapp"`
2. Set your system prompt to define the bot's persona
3. Add knowledge base entries for your FAQ/product info
4. Configure your WhatsApp Business API credentials in Settings
5. Set the webhook URL in Meta Business Manager:

```
https://api.callmissed.com/api/v1/webhooks/whatsapp
```

## Message Flow

```
User sends WhatsApp message
  → Meta sends POST to /api/v1/webhooks/whatsapp
  → Signature verified
  → Conversation found or created
  → Message saved
  → Conversation history + system prompt → LLM
  → AI response saved
  → Response sent back via WhatsApp Cloud API
```

## Bot Config

When creating a WhatsApp bot, pass channel config:

```json
{
  "name": "Support Bot",
  "type": "whatsapp",
  "system_prompt": "You are a helpful agent for Acme Corp.",
  "config": {
    "phone_number_id": "your_meta_phone_number_id",
    "language": "en"
  }
}
```