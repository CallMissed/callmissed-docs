---
title: "Libraries & SDKs"
description: "Use the OpenAI or Anthropic SDK to integrate CallMissed — point the client at our base URL and use your cm_ API key."
slug: "sdks"
breadcrumb: "Getting Started"
---

# Libraries & SDKs

Use the OpenAI or Anthropic SDK to integrate CallMissed — point the client at our base URL and use your cm_ API key.

## Supported Libraries

CallMissed does not ship its own client library for the inference API. Our endpoints are compatible with **two existing SDK families**, so use whichever you prefer: change the base URL and use your `cm_` API key. The platform APIs under `/api/v1/` (agents, CRM, support, webhooks and the rest) are plain JSON over HTTPS; call them with any HTTP client.

### OpenAI SDK (recommended for most use cases)

| Language | Package | Manager |
|----------|---------|---------|
| Python | `openai` | PyPI |
| JavaScript / TypeScript | `openai` | npm |
| Go | `github.com/openai/openai-go/v3` | go modules |
| PHP | `openai-php/client` | Composer |
| Ruby | `ruby-openai` | RubyGems |
| Java / Kotlin | HTTP client | Maven / Gradle |

### Anthropic SDK (for /v1/messages endpoint)

| Language | Package | Manager |
|----------|---------|---------|
| Python | `anthropic` | PyPI |
| JavaScript / TypeScript | `@anthropic-ai/sdk` | npm |

> **Tip:** The Anthropic SDK talks to `/v1/messages`. See the [Anthropic API docs](/docs/anthropic-api) for full details.

## Installation

:::tabs
```bash [Python]
pip install openai
```
```bash [JavaScript / TypeScript]
npm install openai
```
```bash [Go]
go get github.com/openai/openai-go/v3
```
```bash [PHP]
composer require openai-php/client
```
```bash [Ruby]
gem install ruby-openai
```
```bash [Python (Anthropic)]
pip install anthropic
```
```bash [JS/TS (Anthropic)]
npm install @anthropic-ai/sdk
```
:::

Upgrade to the latest version:

:::tabs
```bash [Python]
pip install --upgrade openai
```
```bash [JavaScript / TypeScript]
npm install openai@latest
```
```bash [Go]
go get -u github.com/openai/openai-go/v3
```
```bash [PHP]
composer update openai-php/client
```
```bash [Ruby]
gem update ruby-openai
```
:::

## Configuration

:::tabs
```python [Python]
from openai import OpenAI

client = OpenAI(
    api_key="cm_your_api_key",
    base_url="https://api.callmissed.com/v1"
)

response = client.chat.completions.create(
    model="sarvam-105b",
    messages=[{"role": "user", "content": "Hello"}]
)
print(response.choices[0].message.content)
```
```typescript [TypeScript]
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "cm_your_api_key",
  baseURL: "https://api.callmissed.com/v1",
});

const response = await client.chat.completions.create({
  model: "sarvam-105b",
  messages: [{ role: "user", content: "Hello" }],
});
console.log(response.choices[0].message.content);
```
```javascript [JavaScript]
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "cm_your_api_key",
  baseURL: "https://api.callmissed.com/v1",
});

const response = await client.chat.completions.create({
  model: "sarvam-105b",
  messages: [{ role: "user", content: "Hello" }],
});
console.log(response.choices[0].message.content);
```
```go [Go]
package main

import (
    "context"
    "fmt"
    "github.com/openai/openai-go/v3"
    "github.com/openai/openai-go/v3/option"
)

func main() {
    client := openai.NewClient(
        option.WithAPIKey("cm_your_api_key"),
        option.WithBaseURL("https://api.callmissed.com/v1"),
    )

    resp, err := client.Chat.Completions.New(context.Background(),
        openai.ChatCompletionNewParams{
            Model: "sarvam-105b",
            Messages: []openai.ChatCompletionMessageParamUnion{
                openai.UserMessage("Hello"),
            },
        },
    )
    if err != nil {
        panic(err)
    }
    fmt.Println(resp.Choices[0].Message.Content)
}
```
```php [PHP]
<?php
$client = OpenAI::factory()
    ->withApiKey('cm_your_api_key')
    ->withBaseUri('https://api.callmissed.com/v1')
    ->make();

$response = $client->chat()->create([
    'model'    => 'sarvam-105b',
    'messages' => [['role' => 'user', 'content' => 'Hello']],
]);

echo $response->choices[0]->message->content;
```
```ruby [Ruby]
require "openai"

client = OpenAI::Client.new(
  access_token: "cm_your_api_key",
  uri_base: "https://api.callmissed.com/v1"
)

response = client.chat(
  parameters: {
    model: "sarvam-105b",
    messages: [{ role: "user", content: "Hello" }]
  }
)
puts response.dig("choices", 0, "message", "content")
```
```bash [cURL]
curl https://api.callmissed.com/v1/chat/completions \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "sarvam-105b", "messages": [{"role": "user", "content": "Hello"}]}'
```
:::

## Resources

| Resource | Link |
|----------|------|
| API Reference | [docs.callmissed.com](https://docs.callmissed.com) |
| Docs in your coding agent | [`callmissed-docs-mcp`](/docs/mcp-server) on npm |
| Dashboard | [console.callmissed.com](https://console.callmissed.com) |
| LinkedIn | [linkedin.com/company/callmissed](https://www.linkedin.com/company/callmissed) |
