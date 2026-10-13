---
title: "Function Calling"
description: "Use tool calls and function calling with the chat completion API."
slug: "chat-function-calling"
breadcrumb: "LLM & AI"
---

# Function Calling

Use tool calls and function calling with the chat completion API.

## Overview

Function calling lets the model invoke your functions. Pass a `tools` array and the model returns structured `tool_calls` when it wants to call a function.

## Defining Tools

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get current weather for a city",
        "parameters": {
          "type": "object",
          "properties": {
            "city": {"type": "string", "description": "City name"},
            "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
          },
          "required": ["city"]
        }
      }
    }
  ]
}
```

## Tool Choice

| Value | Description |
|-------|-------------|
| `"auto"` | Model decides whether to call a tool (default) |
| `"none"` | Never call tools |
| `"required"` | Always call at least one tool |
| `{"type": "function", "function": {"name": "get_weather"}}` | Force a specific function |

Set `parallel_tool_calls: true` to allow the model to call multiple tools in one response.

Up to 128 tools per request. Every model with `supports_tools: true` in [`GET /v1/models`](/docs/models#models-api) accepts them. On the GPT-5.6 and GPT-6 models, a request that sends `tools` runs with reasoning switched off (`reasoning_effort` is sent as `"none"`), because those models cannot combine function tools with reasoning on this endpoint. A request with `tools` never uses the `models` fallback list.

### Claude models

`claude-opus-5-5`, `claude-sonnet-5-5` and `claude-haiku-5-5` (Pro plan and up) accept `tools` with reasoning on. Two things differ from the other models:

- `tool_choice: "required"` or a named function is honoured on `claude-haiku-5-5`. On `claude-opus-5-5` and `claude-sonnet-5-5` it runs as `"auto"`, because those models do not support forced tool use.
- When a Claude response contains tool calls on `/v1/chat/completions`, the first tool call carries an extra `anthropic_thinking` field holding the model's full turn, including its signed reasoning. Send the assistant message back **unchanged**, including that field, in your next request so the model keeps its reasoning across tool calls. On [`/v1/messages`](/docs/anthropic-api) the same reasoning comes back as standard `thinking` / `redacted_thinking` content blocks; pass them back unchanged.

## Handling Response

When the model calls a tool, `finish_reason` is `"tool_calls"` and `message.content` is `null`:

```json
{
  "choices": [{
    "finish_reason": "tool_calls",
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [{
        "id": "call_abc123",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"city\": \"Mumbai\"}"
        }
      }]
    }
  }]
}
```

Send the tool result back as a `tool` role message:

```json
{
  "role": "tool",
  "tool_call_id": "call_abc123",
  "content": "{\"temperature\": 32, \"condition\": \"sunny\"}"
}
```

## Counting Tool Calls

Every response includes a `usage.tool_call_count` field — the number of tool calls the model made in that response (`0` when none). It is present on both streaming and non-streaming responses, so you can track tool usage per request:

```json
{
  "choices": [{ "finish_reason": "tool_calls", "message": { "tool_calls": [/* ... */] } }],
  "usage": { "prompt_tokens": 18, "completion_tokens": 25, "total_tokens": 43, "tool_call_count": 2 }
}
```

When streaming, each chunk that carries a `delta.tool_calls` fragment also includes a top-level `tool_call_count` that increments as new tool calls begin — useful for showing a live "tools called" counter in your UI. The definitive total is always in the final `usage` chunk (requires `stream_options: {"include_usage": true}`).

## Full Example

```python
import json
from openai import OpenAI

client = OpenAI(api_key="cm_your_key", base_url="https://api.callmissed.com/v1")

# Step 1: Send initial request with tools
response = client.chat.completions.create(
    model="sarvam-105b",
    messages=[{"role": "user", "content": "What's the weather in Mumbai?"}],
    tools=[{
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get weather for a city",
            "parameters": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"]
            }
        }
    }],
    tool_choice="auto"
)

# Step 2: Check if model wants to call a tool
msg = response.choices[0].message
if msg.tool_calls:
    tool_call = msg.tool_calls[0]
    # Execute your function here...
    result = get_weather(json.loads(tool_call.function.arguments)["city"])

    # Step 3: Send result back
    final = client.chat.completions.create(
        model="sarvam-105b",
        messages=[
            {"role": "user", "content": "What's the weather in Mumbai?"},
            msg,  # assistant message with tool_calls
            {"role": "tool", "tool_call_id": tool_call.id, "content": json.dumps(result)}
        ]
    )
    print(final.choices[0].message.content)
```
