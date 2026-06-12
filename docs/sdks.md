---
title: "Libraries & SDKs"
description: "Use the OpenAI SDK to integrate CallMissed APIs — our endpoints are fully OpenAI-compatible."
slug: "sdks"
breadcrumb: "Getting Started"
---

# Libraries & SDKs

Use the OpenAI SDK to integrate CallMissed APIs — our endpoints are fully OpenAI-compatible.

## Official Libraries

CallMissed supports **two SDK families** — use whichever you prefer. Just change the base URL and use your `cm_` API key.

### OpenAI SDK (recommended for most use cases)

| Language | Package | Manager |
|----------|---------|---------|
| Python | `openai` | PyPI |
| JavaScript / TypeScript | `openai` | npm |
| Go | `openai-go` | go modules |
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
go get github.com/openai/openai-go
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
go get -u github.com/openai/openai-go
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
    model="sarvam-30b",
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
  model: "sarvam-30b",
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
  model: "sarvam-30b",
  messages: [{ role: "user", content: "Hello" }],
});
console.log(response.choices[0].message.content);
```
```go [Go]
package main

import (
    "context"
    "fmt"
    "github.com/openai/openai-go"
    "github.com/openai/openai-go/option"
)

func main() {
    client := openai.NewClient(
        option.WithAPIKey("cm_your_api_key"),
        option.WithBaseURL("https://api.callmissed.com/v1"),
    )

    resp, _ := client.Chat.Completions.New(context.Background(),
        openai.ChatCompletionNewParams{
            Model: openai.F("sarvam-30b"),
            Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
                openai.UserMessage("Hello"),
            }),
        },
    )
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
    'model'    => 'sarvam-30b',
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
    model: "sarvam-30b",
    messages: [{ role: "user", content: "Hello" }]
  }
)
puts response.dig("choices", 0, "message", "content")
```
```bash [cURL]
curl https://api.callmissed.com/v1/chat/completions \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "sarvam-30b", "messages": [{"role": "user", "content": "Hello"}]}'
```
:::

## Resources

| Resource | Link |
|----------|------|
| API Reference | [docs.callmissed.com](https://docs.callmissed.com) |
| Dashboard | [app.callmissed.com](https://app.callmissed.com) |
| LinkedIn | [linkedin.com/company/callmissed](https://www.linkedin.com/company/callmissed) |