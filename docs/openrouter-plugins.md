---
title: "OpenRouter Plugins & Routing"
description: "Advanced request features on the gateway — file parsing, response healing, context compression, provider routing, and generation lookup."
slug: "openrouter-plugins"
breadcrumb: "Guides"
---

# OpenRouter Plugins & Routing

Advanced request features on the gateway — file parsing, response healing, context compression, provider routing, and generation lookup.

## Overview

When a request routes through the OpenRouter overlay (any model id with a slash, e.g. `openai/gpt-5.4`), you can opt into extra processing via the `plugins` array on a chat completion request.

## Plugins

```json
{
  "model": "anthropic/claude-sonnet-4.6",
  "messages": [{ "role": "user", "content": "Summarize this PDF" }],
  "plugins": [
    { "id": "web" },
    "file-parser",
    "response-healing",
    "context-compression"
  ]
}
```

| Plugin | What it does |
| --- | --- |
| `web` | Adds live [web search](/docs/web-search) results as grounding |
| `file-parser` | Extracts text from attached files before the model sees them |
| `response-healing` | Repairs malformed/truncated structured output |
| `context-compression` | Compresses long context to fit the model window |

You can also influence which upstream provider serves an OpenRouter model via provider-routing preferences in the request body.

## Generation Lookup

After a completion, fetch detailed generation metadata (token counts, upstream provider, native cost) by id:

```bash
curl https://api.callmissed.com/v1/generation?id=gen_abc123 \
  -H "Authorization: Bearer cm_your_key"
```