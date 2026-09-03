---
title: "API Speed: Best Practices"
description: "How to make CallMissed API calls feel as fast as the playground — streaming, model choice, connection reuse, and Kimi instant mode."
slug: "api-speed"
breadcrumb: "Getting Started"
---

# API Speed: Best Practices

How to make CallMissed API calls feel as fast as the playground — streaming, model choice, connection reuse, and Kimi instant mode.

> **About the numbers on this page.** [Inference] Latency and throughput figures here (token rates, time-to-first-byte, per-call timings) are representative measurements taken under specific conditions — small prompts, warm connections, a given region and provider. They illustrate *relative* differences between settings; they are not SLAs or guarantees and will vary with prompt size, model, upstream load, and your network. AI behavior is not guaranteed and may vary.

## Why the playground feels faster

A single playground call and a typical API call hit the **exact same endpoint** at `https://api.callmissed.com/v1/chat/completions`. When the API feels slower, three compounding factors are usually at work:

| Setting | Playground default | Common API default | Effect |
| --- | --- | --- | --- |
| Streaming | `stream: true` | `stream: false` | Non-streaming waits for the *whole* generation before any byte returns |
| Model | `gpt-oss-120b` (fast free model) | `gpt-5.6-sol` | Bigger model → 2-3× wall-clock for the same prompt |
| Connection | One persistent HTTP/2 connection | New TCP+TLS per call | Adds ~150-300ms handshake to every request |

Same endpoint, very different perceived speed. Below: how to close the gap.

## 1. Use stream: true

With `stream: false` your client waits for the full generation. With `stream: true` the first byte typically arrives in well under a second — often around 100ms on fast models — and tokens flow as the model produces them instead of all at the end.

```python [Python]
from openai import OpenAI

client = OpenAI(
    api_key="cm_your_key",
    base_url="https://api.callmissed.com/v1",
)

# stream=True is the default in the Anthropic SDK; explicit here.
stream = client.chat.completions.create(
    model="gpt-oss-120b",
    messages=[{"role": "user", "content": "Explain quicksort in one paragraph."}],
    stream=True,
)
for chunk in stream:
    delta = chunk.choices[0].delta.content or ""
    print(delta, end="", flush=True)
```

```ts [TypeScript]
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "cm_your_key",
  baseURL: "https://api.callmissed.com/v1",
});

const stream = await client.chat.completions.create({
  model: "gpt-oss-120b",
  messages: [{ role: "user", content: "Explain quicksort in one paragraph." }],
  stream: true,
});

for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content ?? "");
}
```

## 2. Pick the right model

Same prompt, three different routes. The first-token and total times below are **indicative only**, not a dated benchmark: they were carried over unchanged when the model list was last revised, so they have not been measured against the models now named. Use them for rough shape and measure your own prompt size and region before designing around them.

| Model | Route | TTFB | Total |
| --- | --- | --- | --- |
| `gpt-oss-120b` | Direct-routed | ~50ms | ~1.5s |
| `kimi-k2.6` | Direct-routed | ~90ms | ~2.0s |
| `gpt-5.6-sol` | First-party flagship | ~70ms | ~4.8s |
| `gpt-5.6-luna` | First-party fast | ~80ms | ~2.0s |

For latency-sensitive integrations (autocomplete, agent tool loops), prefer **`gpt-oss-120b`** or **`kimi-k2.6`** — both direct-routed, both OpenAI-compatible, both sub-2s on small prompts.

## 3. Reasoning effort by model

Reasoning models can spend 100+ tokens "thinking" before producing visible content. On short answers, that's both a wall-clock and a credit cost the user never sees. Use `reasoning_effort` to dial it down — or off, where supported.

```json
{
  "model": "kimi-k2.6",
  "messages": [{ "role": "user", "content": "What is 2+2?" }],
  "reasoning_effort": "none"
}
```

Behaviour per model — verified live against each upstream on 2026-05-01:

| Model | `"none"` | `"low"` | `"medium"` | `"high"` | `"xhigh"` | `"minimal"` |
| --- | --- | --- | --- | --- | --- | --- |
| `gpt-5.6-sol` | ✅ off | ✅ | ✅ | ✅ | ✅ | ↓ `"low"` |
| `gpt-5.6-terra` | ✅ off | ✅ | ✅ | ✅ | ✅ | ↓ `"low"` |
| `gpt-5.6-luna` | ✅ off | ✅ | ✅ | ✅ | ✅ | ↓ `"low"` |
| `gpt-5.5` | ✅ off | ✅ | ✅ | ✅ | ✅ | ↓ `"low"` |
| `kimi-k2.5` | ✅ off | ✅ | ✅ | ✅ | — | ↓ `"none"` |
| `kimi-k2.6` | ✅ off | ✅ | ✅ | ✅ | — | ↓ `"none"` |
| `gpt-oss-120b` | ↓ `"low"` | ✅ | ✅ | ✅ | — | ↓ `"low"` |
| `nemotron-3-super` | ↓ `"low"` | ✅ | ✅ | ✅ | — | ↓ `"low"` |
| `glm-4.7-flash` | ✅ off | ⊘ | ⊘ | ⊘ | — | ✅ off |
| `glm-5.2` | ✅ off | ⊘ | ⊘ | ⊘ | — | ✅ off |
| `gemma-4-26b-a4b-it` | ✅ off | ⊘ | ⊘ | ⊘ | — | ✅ off |
| `sarvam-105b`, `sarvam-105b-conversations` | ↓ `"low"` | ✅ | ✅ | ✅ | — | ↓ `"low"` |
| `kimi-k2.7-code` | ⊘ | ⊘ | ⊘ | ⊘ | — | ⊘ |
| `mistral-small-3.1` | ⊘ | ⊘ | ⊘ | ⊘ | — | ⊘ |

Legend: ✅ = sent to upstream verbatim · ✅ off = thinking is switched off · ↓ = mapped to the listed value before forwarding · ⊘ = dropped from the request; the model runs at its default thinking behaviour · — = not accepted by that model.

Three notes worth reading before you rely on a value:

- `glm-4.7-flash`, `glm-5.2` and `gemma-4-26b-a4b-it` honour only the off switch. `"none"` and `"minimal"` turn thinking off. `"low"`, `"medium"` and `"high"` are dropped, so the model thinks at its own default — they are not an intensity dial.
- `kimi-k2.7-code` and `mistral-small-3.1` expose no reasoning control. Every value is dropped. Both still return 200.
- The GPT-5.5 / GPT-5.6 family accepts the full `none`/`low`/`medium`/`high`/`xhigh` ladder. `"minimal"` is rejected upstream, so we map it to `"low"`. Send `xhigh`, not `max` or `ultra`.

Concrete numbers on `kimi-k2.6` answering "What is 2+2?":

| Mode | Answer | Completion tokens |
| --- | --- | --- |
| Default (thinking on) | `2 + 2 = 4` | 101 |
| `reasoning_effort: "none"` | `2 + 2 = 4` | **9** |

Same answer, **11× fewer tokens** and a proportionally shorter wall-clock. Pass per-request based on whether you need the trace.

## 4. Reuse the HTTP connection

CallMissed serves HTTP/2. A persistent connection multiplexes many requests with no extra TLS handshake per call. Most modern HTTP clients do this *if you reuse the same client instance*:

```python [Python]
# Bad — new connection per call (~150-300ms TLS overhead each time)
def ask(prompt):
    return OpenAI(api_key=K, base_url=URL).chat.completions.create(...)

# Good — reuse one client (and its underlying connection pool)
client = OpenAI(api_key=K, base_url=URL)
def ask(prompt):
    return client.chat.completions.create(...)
```

```ts [TypeScript]
// Bad
async function ask(prompt: string) {
  const c = new OpenAI({ apiKey: K, baseURL: URL });
  return c.chat.completions.create(...);
}

// Good — module-scoped client
const client = new OpenAI({ apiKey: K, baseURL: URL });
async function ask(prompt: string) {
  return client.chat.completions.create(...);
}
```

Three back-to-back streaming calls on a reused HTTP/2 connection land in the region of **400-460ms each** for a small prompt, again indicative rather than a dated measurement; the first call without reuse pays an extra TLS handshake on top.

## Checklist

Before reporting "the API feels slow," verify:

- [ ] `stream: true` is set on every chat completion request
- [ ] Model is one of `gpt-oss-120b`, `kimi-k2.6`, `mistral-small-3.1`, or `gpt-5.6-luna` (avoid the 1M-context flagships for latency-bound paths)
- [ ] For Kimi: `reasoning_effort: "none"` is passed when you don't need the reasoning trace
- [ ] One `OpenAI()` / `Anthropic()` client instance is shared across requests
- [ ] HTTP client supports HTTP/2 (the official OpenAI / Anthropic SDKs do)

If all four are checked and you still see latency that doesn't match the playground, share the request and the upstream model — there are very few request shapes that beat playground default for the same model.
