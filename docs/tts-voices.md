---
title: "Voices"
description: "Available voices for text-to-speech synthesis."
slug: "tts-voices"
breadcrumb: "Speech"
---

# Voices

Available voices for text-to-speech synthesis.

## Indic Voices

**bulbul:v3** provides **37 voices** spanning 11 Indian languages (`bn-IN`, `en-IN`, `gu-IN`, `hi-IN`, `kn-IN`, `ml-IN`, `mr-IN`, `od-IN`, `pa-IN`, `ta-IN`, `te-IN`). Pass the voice ID as the `voice` parameter and the target `language`. The default voice is `shubh`. An unrecognized voice falls back to the recommended voice for the request's `language` (for example `ratan` for `en-IN`, `ta-IN`, `mr-IN` and `gu-IN`, `shubh` for the rest).

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

`GET /api/v1/models/sonic-3.6/voices/{id}/preview` streams Cartesia's own pre-recorded sample (no synthesis, no credit charge). Neither listing endpoint synthesizes or bills.

Query params (both listing endpoints): `q` (search, max 200 chars), `language` (base code such as `hi`), `gender` (`masculine` / `feminine` / `gender_neutral`; `male` / `female` also accepted), `limit` (1–100, default 50), `starting_after` (a voice `id` from the previous page). `GET /v1/audio/voices` takes `model=sonic-3.6` (the default and the only accepted value; any other returns `400`).

```json
{
  "object": "list",
  "model": "sonic-3.6",
  "data": [
    {
      "id": "voice-uuid",
      "handle": "skylar",
      "name": "Skylar",
      "gender": "feminine",
      "language": "en",
      "locale": "en-US",
      "accent": "",
      "tagline": "",
      "description": "",
      "has_preview": true,
      "preview": "/api/v1/models/sonic-3.6/voices/voice-uuid/preview"
    }
  ],
  "has_more": true,
  "next_page": "…"
}
```

`handle` is set only for the featured aliases below (otherwise `null`); pass either the `id` or the handle as `voice` on `POST /v1/audio/speech`. `preview` is `null` when a voice has no sample. When `has_more` is `true`, pass the last voice's `id` as `starting_after` to fetch the next page.

If the upstream library is briefly unreachable, the first page answers `200` with the featured aliases below and an extra `"degraded": true` field instead of failing, so a voice picker still renders and every returned voice remains usable for synthesis. The field is **absent** on a normal response — treat its presence as "this is the short list, retry later for the full library". A request carrying `starting_after` is not degraded: pagination returns `502` so you keep the page you already have.

Featured aliases (stable handles, also valid UUIDs in the library):

```text
skylar · daniel · jacqueline · katie · cathy · caroline · ronald · carson · jameson
gemma · archie · riya · arushi · siya · parvati · kabir
```

`riya`, `arushi`, `siya`, `parvati`, and `kabir` are native Hindi speakers (pair with `"language": "hi"`). Browse the full library in the [console Voices page](https://console.callmissed.com/voice/voices), the [Playground](https://platform.callmissed.com/playground/tts), or [Talk](https://callmissed.com/talk).

> **Other TTS providers** also expose voices via the same `POST /v1/audio/speech` endpoint — **aura-2-en** (40 English voices, default `luna`), **aura-2-es** (10 Spanish voices, default `aquila`), **deepgram-aura-2** (90 voices across English, Spanish, German, French, Dutch, Italian, and Japanese, default `thalia`), **deepgram-aura-1** (12 legacy English voices at half the Aura-2 rate, default `asteria`), and **gpt-4o-mini-tts** (`alloy`, `echo`, `fable`, `onyx`, `nova`, `shimmer`). See [Credits & Rate Limits](/docs/credits-rate-limits) for per-model pricing.

## Flux TTS Voices (managed Voice Agent only)

Deepgram Flux TTS is a voice-agent-first model. It is **not** available on the `POST /v1/audio/speech` endpoint — it is offered only through the managed Voice Agent (see the Voice Sessions API), selectable with `tts_engine: "flux"`, where synthesis is turn-based, prosody carries across turns, and it is billed inside the per-minute voice rate. It provides **36 English voices** (American, British, Indian, Irish, Australian, Singaporean and Filipino accents); the default is `priya`, an Indian-accented English voice, and an unrecognized voice falls back to the default.

```text
alexis · bree · brittany · brooke · bruce · cliff · cole · colin · conor
donovan · drew · elise · gemma · haley · hannah · heather · jack · kai
kelsey · kit · maeve · marcelo · marcus · meena · meghan · miles · naveen
paige · priya · rufus · sean · sharon · sienna · tanner · wade · wes
```

English only — a multilingual voice set is planned for a later release. The model exposes no expressive/emotion/style controls and does not interpret SSML.
