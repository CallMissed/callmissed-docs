---
title: "Developer Quickstart"
description: "Get started with CallMissed APIs in under a minute using just a few lines of code"
slug: "quickstart"
breadcrumb: "Getting Started"
---

# Developer Quickstart

Get started with CallMissed APIs in under a minute using just a few lines of code

> **Note:** CallMissed APIs are OpenAI-compatible — use the official OpenAI SDK and change only the base URL and API key prefix (`cm_`).

:::tabs
```python [Python]
from openai import OpenAI

client = OpenAI(api_key="cm_your_api_key", base_url="https://api.callmissed.com/v1")
response = client.chat.completions.create(
    model="sarvam-30b",
    messages=[{"role": "user", "content": "Hello in Hindi"}],
)
print(response.choices[0].message.content)
```
```typescript [TypeScript]
import OpenAI from "openai";
const client = new OpenAI({ apiKey: "cm_your_api_key", baseURL: "https://api.callmissed.com/v1" });
const response = await client.chat.completions.create({
  model: "sarvam-30b",
  messages: [{ role: "user", content: "Hello in Hindi" }],
});
console.log(response.choices[0].message.content);
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/chat/completions \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"model":"sarvam-30b","messages":[{"role":"user","content":"Hello in Hindi"}]}'
```
:::

:::cards
/docs/sdks | SDKs & Libraries | package | Install the OpenAI SDK in your language
/docs/models | Model Catalog | boxes | 63 models — LLM, STT, TTS, and image
:::

## Get started

:::steps
## Create an API Key

Visit the [CallMissed Dashboard](https://app.callmissed.com) and create a new API key. Keep this key secure — you'll need it to authenticate your requests.

## Set up your environment

Export your API key as an environment variable:

:::tabs
```bash [macOS / Linux]
export CALLMISSED_API_KEY="your_api_key_here"
```
```powershell [Windows (PowerShell)]
$env:CALLMISSED_API_KEY = "your_api_key_here"
```
```cmd [Windows (CMD)]
set CALLMISSED_API_KEY=your_api_key_here
```
:::

## Install the SDK

Choose your preferred language and install the OpenAI SDK:

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
```bash [Java]
# Add to pom.xml or build.gradle — use OkHttp or any HTTP client
```
:::

## Make your first API call

Use the same pattern as the hero example above — pass your `cm_` API key and pick any [free-tier model](/docs/model-access).

> **Tip:** Store your API key in environment variables. Never commit keys to version control.

:::

**Ready to explore?** See [Chat Completion](/docs/chat-completion) for streaming and tool calling, or browse the [Model Catalog](/docs/models).

## CallMissed APIs

### Chat Completion

:::tabs
```python [Python]
response = client.chat.completions.create(
    model="sarvam-30b",
    messages=[{"role": "user", "content": "Explain quantum computing"}],
    stream=True
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```
```javascript [JavaScript]
const stream = await client.chat.completions.create({
  model: "sarvam-30b",
  messages: [{ role: "user", content: "Explain quantum computing" }],
  stream: true,
});

for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content || "");
}
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/chat/completions \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sarvam-30b",
    "messages": [{"role": "user", "content": "Explain quantum computing"}],
    "stream": true
  }'
```
:::

### Speech to Text

:::tabs
```python [Python]
with open("audio.wav", "rb") as f:
    response = client.audio.transcriptions.create(
        model="saaras:v3",
        file=f
    )
print(response.text)
```
```javascript [JavaScript]
import fs from "fs";

const response = await client.audio.transcriptions.create({
  model: "saaras:v3",
  file: fs.createReadStream("audio.wav"),
});

console.log(response.text);
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/audio/transcriptions \
  -H "Authorization: Bearer cm_your_key" \
  -F file=@audio.wav \
  -F model=saaras:v3
```
:::

### Text to Speech

:::tabs
```python [Python]
response = client.audio.speech.create(
    model="bulbul:v3",
    voice="shubh",
    input="Namaste, kaise hain aap?"
)

response.stream_to_file("speech.mp3")
```
```javascript [JavaScript]
const response = await client.audio.speech.create({
  model: "bulbul:v3",
  voice: "shubh",
  input: "Namaste, kaise hain aap?",
});

const buffer = Buffer.from(await response.arrayBuffer());
fs.writeFileSync("speech.mp3", buffer);
```
```bash [cURL]
curl -X POST https://api.callmissed.com/v1/audio/speech \
  -H "Authorization: Bearer cm_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "bulbul:v3", "input": "Namaste, kaise hain aap?", "voice": "shubh"}' \
  --output speech.mp3
```
:::

## Next steps

:::cards
/docs/models | Models | boxes | LLM, STT, TTS, and image models across every major provider
/docs/chat-completion | Chat Completion | message-square | Streaming, tools, and frontier model routing
/docs/voice-agent | Voice Agent | phone | LiveKit WebRTC voice agents with Indic STT/TTS
/docs/call-analytics | Cookbooks | bar-chart | Step-by-step tutorials for production workflows
:::