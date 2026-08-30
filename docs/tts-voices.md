---
title: "Voices"
description: "Available voices for text-to-speech synthesis."
slug: "tts-voices"
breadcrumb: "Speech"
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

**gnani-timbre-v2.0** provides **73 voices** across English, Hindi and other Indian languages with context-aware tone for telephony-grade delivery. The default voice is `Nalini`; an unrecognized voice falls back to the default.

```text
Nalini · Bhavna · Yashvi · Urmila · Jwala · Chitra · Ambuja · Deepak · Roopesh · Vikrant
Hemraj · Jalaj · Omkar · Aarohi · Bhavini · Charvi · Eishani · Falguni · Gauri · Iravati
Janaki · Kamakshi · Madhuri · Radhika · Shweta · Tanvi · Vidya · Wamika · Yamini · Abhimanyu
Chirag · Deven · Farhan · Jatin · Kartik · Kaveri · Trupti · Devika · Pranav · Shlok
Girish · Asmita · Trisha · Brinda · Vedika · Noopur · Oviya · Parvati · Suhana · Lehara
Lavanya · Yukti · Varuni · Saanvi · Kavin · Hansika · Reshma · Riyaan · Zahira · Ishaan
Kirra · Dhruva · Damini · Urvashi · Falak · Veera · Lalita · Nayana · Gaurav · Harshit
Mehuli · Zayan · Poorvi
```

## Cartesia Voices

**sonic-3.6** (Cartesia Sonic 3.6) speaks 44 languages with native-quality Hindi and Hinglish. Pass a featured handle (`skylar`, default) or any public Cartesia voice UUID as `voice`, plus a base ISO `language` code (`en`, `hi`). An unrecognized non-UUID handle falls back to `skylar`.

The live public library is paginated — do not treat the 16 featured aliases as the full set:

```bash
curl "https://api.callmissed.com/api/v1/models/sonic-3.6/voices?q=hindi&limit=50"
curl "https://api.callmissed.com/v1/audio/voices?model=sonic-3.6&q=skylar" \
  -H "Authorization: Bearer cm_YOUR_KEY"
```

`GET /api/v1/models/sonic-3.6/voices/{id}/preview` streams Cartesia's own pre-recorded sample (no synthesis, no credit charge). Query params: `q`, `language`, `gender` (`masculine` / `feminine` / `gender_neutral`), `limit` (1–100), `starting_after`.

Featured aliases (stable handles, also valid UUIDs in the library):

```text
skylar · daniel · jacqueline · katie · cathy · caroline · ronald · carson · jameson
gemma · archie · riya · arushi · siya · parvati · kabir
```

`riya`, `arushi`, `siya`, `parvati`, and `kabir` are native Hindi speakers (pair with `"language": "hi"`). Browse the full library in the [console Voices page](https://console.callmissed.com/voice/voices), the [Playground](https://platform.callmissed.com/playground/tts), or [Talk](https://callmissed.com/talk).

> **Other TTS providers** also expose voices via the same `POST /v1/audio/speech` endpoint — **aura-2-en** (40 English voices, default `luna`), **aura-2-es** (10 Spanish voices), **deepgram-aura-2** (91 voices across English, Spanish, German, French, Dutch, Italian, and Japanese via the direct Deepgram API, default `thalia`), **deepgram-aura-1** (12 legacy English voices via the direct Deepgram API at half the Aura-2 rate, default `asteria`), and **gpt-4o-mini-tts** (`alloy`, `echo`, `fable`, `onyx`, `nova`, `shimmer`). See [Credits & Rate Limits](/docs/credits-rate-limits) for per-model pricing.

## Flux TTS Voices (managed Voice Agent only)

Deepgram Flux TTS is a voice-agent-first model. It is **not** available on the `POST /v1/audio/speech` endpoint — it is offered only through the managed Voice Agent (see the Voice Sessions API), selectable with `tts_engine: "flux"`, where synthesis is turn-based, prosody carries across turns, and it is billed inside the per-minute voice rate. It provides **11 English voices**; the default is `priya`, an Indian-accented English voice, and an unrecognized voice falls back to the default.

```text
alexis · bruce · cole · drew · haley · heather
jack · marcus · priya · rufus · sharon
```

English only — a multilingual voice set is planned for a later release. The model exposes no expressive/emotion/style controls and does not interpret SSML.
