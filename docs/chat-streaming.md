---
title: "Streaming"
description: "Stream chat completion responses in real-time using Server-Sent Events."
slug: "chat-streaming"
breadcrumb: "Chat Completion"
---

# Streaming

Stream chat completion responses in real-time using Server-Sent Events.

## Overview

Enable streaming by setting `"stream": true`. The response is a Server-Sent Events (SSE) stream with `Content-Type: text/event-stream`.

## SSE Format

Each event is a line starting with `data: ` followed by a JSON chunk:

```
data: {"id":"...","choices":[{"delta":{"role":"assistant","content":""},"finish_reason":null}]}

data: {"id":"...","choices":[{"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"...","choices":[{"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

- **First chunk** always includes `{"delta": {"role": "assistant", "content": ""}}`
- **Content chunks** carry `{"delta": {"content": "token"}}`
- **Final chunk** has `{"delta": {}, "finish_reason": "stop"}`
- **End marker** is `data: [DONE]`

## Usage in Stream

To get token usage in the stream, set `stream_options: {"include_usage": true}`. A final chunk with a `usage` field is sent before `[DONE]`:

```json
data: {"id":"...","choices":[],"usage":{"prompt_tokens":12,"completion_tokens":34,"total_tokens":46}}

data: [DONE]
```

## Code Example

:::tabs
```python [Python]
from openai import OpenAI

client = OpenAI(
    api_key="cm_your_key",
    base_url="https://api.callmissed.com/v1"
)

stream = client.chat.completions.create(
    model="sarvam-30b",
    messages=[{"role": "user", "content": "Hello"}],
    stream=True,
    stream_options={"include_usage": True}
)

for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```
```javascript [JavaScript]
const stream = await client.chat.completions.create({
  model: "sarvam-30b",
  messages: [{ role: "user", content: "Hello" }],
  stream: true,
  stream_options: { include_usage: true },
});

for await (const chunk of stream) {
  const content = chunk.choices[0]?.delta?.content;
  if (content) process.stdout.write(content);
}
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/chat/completions \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model":"sarvam-30b","messages":[{"role":"user","content":"Hello"}],"stream":true}'
```
:::