---
title: "Voice Agent Plans"
description: "Run a voice agent on a flat per-minute plan — ₹4, ₹5 or ₹6 a minute for listening, the language model and the voice — instead of choosing and paying for each model yourself."
slug: "voice-tiers"
breadcrumb: "Voice Agents"
---

# Voice Agent Plans

Run a voice agent on a flat per-minute plan — ₹4, ₹5 or ₹6 a minute for listening, the language model and the voice — instead of choosing and paying for each model yourself.

## Overview

A **voice agent plan** (a *tier*) is a priced product: you choose the plan and a voice, and the platform chooses the speech recognition, the language model and the voice model. The whole AI layer of the call bills at **one flat rate a minute**.

The alternative is a **custom stack**: you set `stt_model`, `voice_model` and `tts_model` yourself, and each one bills at its own published rate, by the second, with no minimum.

| Plan | `voice_tier` | Per minute | Listening options | Best for |
| --- | --- | --- | --- | --- |
| Standard | `t1` | ₹4 (4 credits) | `multilingual` | Natural Indian-language voices at the lowest price |
| Expressive | `t2` | ₹5 (5 credits) | `multilingual` | Richer, more expressive voices across Indian and world languages |
| Best latency | `t3` | ₹6 (6 credits) | `multilingual`, `english` | The fastest replies and the best-sounding voices |

1 credit = ₹1 = $0.01. Always read the live list from [`GET /api/v1/bots/config-schema`](/docs/bots) (the `voice_tiers` array) rather than hard-coding it.

## How a plan call is billed

- **One flat rate a minute** for the call's duration, charged once when the call ends.
- **30-second minimum per call.** A 10-second call bills 30 seconds; a 2-minute call on Standard bills ₹8.
- **A call that never connects costs nothing.** A zero-length call does not reach the minimum.
- **No per-component charges.** A plan call is not also billed per speech-recognition second, per token or per character.
- **Phone carriage is separate.** A rented number, or Meta's fee on a business-initiated WhatsApp call, is billed the same way as on a custom stack.
- **The call ends when your credits run out.** The running cost is checked against your balance during the call, and the call is stopped once your balance can no longer cover the minutes already spoken.

## Put an agent on a plan

Set `voice_tier` (and optionally `voice` and `voice_tier_speech_input`) with a config patch. Requires `bots:write`.

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "values": { "voice_tier": "t3", "voice_tier_speech_input": "english", "language": "en" } }'
```

| Key | Values | If omitted |
| --- | --- | --- |
| `voice_tier` | `t1`, `t2`, `t3` | The agent runs its own models and bills per component |
| `voice_tier_speech_input` | A listening option the plan offers (see the table above) | The plan's first listening option |
| `voice` | One of the plan's voices, listed per plan in `voice_tiers[].voices` | The plan's voice model uses its own default speaker |

On a plan, the agent's own `stt_model`, `voice_model`, `tts_model` and `tts_provider` are **ignored**, not merged — a half-overridden plan would no longer match its price. `voice` and `language` stay yours. [`POST /api/v1/bots/validate-config`](/docs/bots) reports a finding when a config sets both.

To move an agent back to a custom stack, remove the key:

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/b1f2c3d4-5678-90ab-cdef-1234567890ab/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "unset": ["voice_tier", "voice_tier_speech_input"] }'
```

## Languages

Standard and Expressive listen in 22 Indian languages plus English, including code-mixed speech such as Hinglish.

Best latency hears ten languages: English, Spanish, French, German, Hindi, Russian, Portuguese, Japanese, Italian and Dutch. For any other language, use Standard or Expressive.

## Read the plans

`GET /api/v1/bots/config-schema` returns the plans beside the per-model keys. Requires `bots:read`.

```json
{
  "voice_tiers": [
    {
      "id": "t1",
      "label": "Standard",
      "blurb": "Natural Indian-language voices at the lowest price.",
      "price_credits_per_minute": 4.0,
      "speech_inputs": [{ "id": "multilingual", "label": "Multilingual" }],
      "voices": ["…"],
      "billing": "flat",
      "minimum_seconds": 30.0
    }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `id` | The value to set as `voice_tier` |
| `price_credits_per_minute` | The flat rate. 1 credit = ₹1 |
| `speech_inputs` | Listening options; the first is the default |
| `voices` | The voices this plan offers |
| `billing` | Always `flat` |
| `minimum_seconds` | The billed floor per connected call |

## Where to see what a call cost

[`GET /v1/voice/sessions/{id}/cost`](/docs/voice-sessions-api) returns the cost of a finished session, and your usage history lists plan calls as one line per call.
