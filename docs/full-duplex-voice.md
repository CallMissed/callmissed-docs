---
title: "Full-Duplex Voice"
description: "gpt-live-1 listens and speaks at the same time, with reasoning and tool work running on a separate backend while the conversation continues."
slug: "full-duplex-voice"
breadcrumb: "Voice Agents"
---

# Full-Duplex Voice

gpt-live-1 listens and speaks at the same time, with reasoning and tool work running on a separate backend while the conversation continues.

## Overview

`gpt-live-1` is a **full-duplex** speech-to-speech model. It listens and speaks
at the same time, so the caller can interject mid-sentence instead of waiting
for a pause. Select it per session; it is not a default.

:::cards
/docs/voice-sessions-api | Voice Session API | key | Create a session and pick the model
/docs/managed-voice-agent | Managed Voice Agent | radio | The WebSocket path, same model selection
/docs/models | Model Catalogue | boxes | Every model and its price
:::

## Full-duplex vs turn-based

Our other speech-to-speech models are **turn-based**: the caller speaks, the
model listens, the model replies. Interruption is handled as an event — the
caller starts talking, playback stops, a new turn begins.

`gpt-live-1` does not work in turns. Input audio and output audio are two
streams on one session timeline, both live for the whole call:

| | Turn-based speech-to-speech | `gpt-live-1` |
| --- | --- | --- |
| Listening while speaking | Interruption ends the turn | Continuous, both directions |
| Turn boundary | An explicit event you can act on | Not reported — you group fragments yourself |
| Interjecting | Cuts the reply short | Layers over the reply |

## The two-part split

A `gpt-live-1` session is two pieces of work running at once.

:::flow
icon:mic | The live voice model | Holds the conversation — hears the caller, speaks, handles overlap and backchannel
icon:bot | The backend | Reasoning, lookups and tool work, running concurrently with speech
icon:volume2 | Back into the conversation | The backend's result is injected into the live session and surfaces as speech
:::

The voice model keeps talking while the backend is still working. That is the
point of the design — the caller is not left in silence — but it changes what a
few familiar signals mean.

## What this changes about how you build

<Callout type="warn">
Interrupting speech does **not** cancel backend work. "Stop talking" and
"cancel my order" are different intents. If the caller talks over the agent
while a tool call is in flight, that call keeps running unless your application
decides otherwise.
</Callout>

- **A completed backend response does not mean the caller heard it.** Completion
  says the work finished, not that the audio reached the other end. Treat
  confirmation as something the caller says back to you.
- **Transcript fragments follow audio cadence, not turns.** Caller and agent
  fragments can overlap, and there is no authoritative turn-completed event.
  Group fragments into turns in your own code, using the timings on each
  fragment.
- **Context is compacted as the call grows.** Older detail may be dropped to
  keep a long session workable. Keep anything authoritative — the order, the
  booking, the verified identity — in your own application state rather than
  relying on the session to still remember it an hour in.

## Not supported

| Not supported | Detail |
| --- | --- |
| Image or video input | Audio and text only |
| Switching language mid-call | The session's configuration is fixed once the session starts |
| Structured outputs | Not available on this model |

Function tools on a `gpt-live-1` session belong to the backend half of the
split, not to the voice model. For the tool surface available on our other
voice agents, see [Voice Agent Tools](/docs/voice-agent-tools).

## Concurrency

`gpt-live-1` capacity is measured in **concurrent sessions**, not requests per
minute, and the number available to you depends on your plan. A session that is
never closed holds its slot until it expires, so close sessions when the call
ends rather than letting them lapse — an abandoned session is a slot another
call cannot use.

This is why `gpt-live-1` is opt-in per session rather than a workspace default.

## Voices

Fourteen voices, default **`marin`**:

```
alloy    ash      ballad   coral    echo     sage     shimmer
verse    marin    cedar    juniper  breeze   vale     ember
```

A voice outside this list falls back to `marin` rather than failing the call.
The voice is fixed for the session — pick it when you create the session.

## Pricing

| What | Rate |
| --- | --- |
| Voice session | **$0.05 / minute**, billed per second |

The per-minute rate covers the whole session, including silence and the time
the backend spends working. There is no separate per-token rate for the voice
half.

Any reasoning model doing backend work is billed **separately**, at its own
published rate — see the [model catalogue](/docs/models). Budget for both.

## Selecting it

```bash
curl -X POST https://api.callmissed.com/v1/voice/sessions \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "system_prompt": "You are a concise support agent.",
    "llm_model": "gpt-live-1",
    "voice": "marin"
  }'
```

The rest of the session lifecycle is unchanged — see the
[Voice Session API](/docs/voice-sessions-api). Because `gpt-live-1` is one model
doing speech in and speech out, no separate speech-recognition or
text-to-speech model is selected for the call.
