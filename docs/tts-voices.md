---
title: "Voices"
description: "Available voices for text-to-speech synthesis."
slug: "tts-voices"
breadcrumb: "Text to Speech"
---

# Voices

Available voices for text-to-speech synthesis.

## Indic Voices

**bulbul:v3** provides **37 voices** spanning 11 Indian languages (`bn-IN`, `en-IN`, `gu-IN`, `hi-IN`, `kn-IN`, `ml-IN`, `mr-IN`, `od-IN`, `pa-IN`, `ta-IN`, `te-IN`). Pass the voice ID as the `voice` parameter and the target `language`. The default voice is `shubh`; an unrecognized voice falls back to `shubh`.

```text
shubh · aditya · ritu · priya · neha · rahul · pooja · rohan · simran · kavya
amit · dev · ishita · shreya · ratan · varun · manan · sumit · roopa · kabir
aayan · ashutosh · advait · anand · tanya · tarun · sunny · mani · gokul · vijay
shruti · suhani · mohit · kavitha · rehan · soham · rupali
```

Preview every voice in the [Playground](https://platform.callmissed.com/playground/tts).

**gnani-timbre-v2.0** provides **24 voices** across English and Hindi with context-aware tone for telephony-grade delivery. The default voice is `Karan`; an unrecognized voice falls back to the default.

```text
Karan · Pranav · Deepak · Raju · Kaveri · Simran · Shubhra · Nara · Riya · Trupti
Vikrant · Viraj · Shlok · Omkar · Tanmay · Girish · Roopesh · Devika · Poorvi · Nalini
Bhavna · Yashvi · Urmila · Chitra
```

> **Other TTS providers** also expose voices via the same `POST /v1/audio/speech` endpoint — **aura-2-en** (40 English voices, default `luna`), **aura-2-es** (10 Spanish voices), **deepgram-aura-2** (91 voices across English, Spanish, German, French, Dutch, Italian, and Japanese via the direct Deepgram API, default `thalia`), and **gpt-4o-mini-tts** (`alloy`, `echo`, `fable`, `onyx`, `nova`, `shimmer`). See [Credits & Rate Limits](/docs/credits-rate-limits) for per-model pricing.