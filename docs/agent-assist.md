---
title: "Agent Assist"
description: "Suggested replies for a human agent working a live conversation, grounded in the conversation's knowledge base."
slug: "agent-assist"
breadcrumb: "LLM & AI"
---

# Agent Assist

Suggested replies for a human agent working a live conversation, grounded in the conversation's knowledge base.

## Overview

`POST /api/v1/assist/suggest` returns up to three short candidate replies for the
next message in one of your [conversations](/docs/conversations), plus the
knowledge-base snippets they were grounded in. It is built for a human agent who
has taken over a thread: show the suggestions, let the agent pick or edit one,
then send it yourself.

Nothing is sent to the customer by this endpoint — it only reads the thread and
returns text.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

The key needs the `conversations:write` scope. A dashboard session also works.

## POST `/api/v1/assist/suggest`

| Field | Type | Required | Constraints |
| --- | --- | --- | --- |
| `conversation_id` | `UUID` | Yes | A conversation in your account |
| `turns` | `array` | No | Up to 12 recent `{role, content}` turns (`role` is `user`, `assistant` or `system`; `content` 1–4,000 characters). Omit it to use the conversation's stored messages |
| `instruction` | `string` | No | Extra steering for these suggestions only, at most 500 characters (e.g. "offer a callback") |
| `log_suggestions` | `boolean` | No | Default `false`. When `true`, the returned suggestions are saved on the conversation for audit |

Send `turns` when the agent can see a message that may not be stored yet; the
server otherwise reads the last 12 messages of the conversation.

```bash
curl -X POST https://api.callmissed.com/api/v1/assist/suggest \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "conversation_id": "0f5c2a8e-6b1d-4c3e-9a7f-2d4e6b8c0a12",
    "instruction": "keep it under two sentences"
  }'
```

```json
{
  "suggestions": [
    "Sorry about the delay! Could you share your order number so I can check it right away?",
    "Your order left our warehouse yesterday and should arrive within 2 business days.",
    "I'm escalating this to our delivery team now and will update you within the hour."
  ],
  "kb_snippets": [
    {
      "source_id": "b3e1…",
      "chunk_index": 4,
      "score": 0.81,
      "content": "Standard delivery takes 3–5 business days after dispatch…"
    }
  ],
  "model": "gpt-oss-120b",
  "remaining_in_window": 29
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `suggestions` | `string[]` | Up to 3 replies, each at most 320 characters, in the customer's language |
| `kb_snippets` | `array` | Up to 4 knowledge chunks (score at least 0.55) from the conversation's bot, each `content` at most 600 characters. Empty when nothing relevant was found |
| `model` | `string` | The model that wrote the suggestions — the conversation's bot model, or `gpt-oss-120b` when the bot sets none |
| `remaining_in_window` | `integer` | Calls left for this conversation in the current 5-minute window |

## Limits

Each conversation allows **30 calls per rolling 5-minute window**. The 31st call
returns `429` with the number of seconds until the window resets in the message.

## Billing

Each call is billed as one small completion at the model's normal
[per-token rate](/docs/models), plus the knowledge-base search. The call fails
with `402` before any model runs when your balance is empty or your monthly
budget cap is reached.

## Errors

Errors use the `{"detail": "..."}` shape.

| Status | When |
| --- | --- |
| `400` | The conversation has no messages yet |
| `402` | `Insufficient credits` or `Monthly budget cap reached` |
| `403` | Key is missing the `conversations:write` scope |
| `404` | `Conversation not found` (unknown id, or another account's) |
| `422` | Body failed validation (bad `role`, `turns` over 12, `instruction` over 500 characters) |
| `429` | The conversation's 30-per-5-minutes limit is spent |
| `502` | The model call failed — safe to retry |
